# 🧠 Memory Leaks in JavaScript & React

A practical guide to understanding, detecting, debugging, and fixing **memory leaks** in JavaScript and React applications.

---

## 📌 What is a Memory Leak?

A **memory leak** happens when an application keeps references to objects that are no longer needed, preventing JavaScript's **Garbage Collector (GC)** from reclaiming that memory.

In simple terms:

```text
Memory is no longer needed
        ↓
Reference still exists
        ↓
Garbage Collector cannot clean it
        ↓
Memory remains occupied
        ↓
Repeated leaks
        ↓
Application becomes slower
```

Over time, this can cause:

- Increased memory usage
- Sluggish UI
- Poor application performance
- Browser tab becoming unresponsive
- In extreme cases, browser crashes

---

# 🧠 Garbage Collection & Reachability

JavaScript uses **Garbage Collection** to automatically manage memory.

The Garbage Collector determines whether an object is still **reachable** from a GC Root.

A simplified model:

```text
GC Root
   │
   ↓
Window
   │
   ↓
React Tree
   │
   ├── Dashboard
   │
   └── LiveTracker
```

If an object is reachable from the GC Root, it is considered active and cannot be garbage collected.

If an object is no longer reachable, its memory can eventually be reclaimed.

### Important Concept

> **Reachability is one of the key concepts behind JavaScript memory management.**

---

# ⚛️ React Components & Memory

Consider:

```text
Window / GC Root
       │
       ↓
   React Tree
       │
       ├── Dashboard
       │
       └── LiveTracker
```

When `LiveTracker` is mounted:

```text
GC Root
   ↓
React Tree
   ↓
LiveTracker
```

It is reachable.

When it is unmounted:

```text
GC Root
   ↓
React Tree

LiveTracker ❌
```

Normally, its memory can now be reclaimed.

However, problems occur when something outside the React tree still holds a reference to the component or its closure.

---

# 🚨 Common React Memory Leak

One common example is `setInterval`.

## ❌ Incorrect Implementation

```jsx
function LiveTracker() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    setInterval(() => {
      setCount((prev) => prev + 1);
    }, 1000);
  }, []);

  return <div>{count}</div>;
}
```

The problem is that the interval continues running even after the component is unmounted.

Conceptually:

```text
Component
    ↓
setInterval()
    ↓
Callback
    ↓
Closure
    ↓
Component state/functions
```

The component may be removed from the React tree, but the browser timer can still hold the callback.

Therefore, memory can remain retained.

---

# ✅ Fix: Cleanup the Interval

```jsx
function LiveTracker() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const intervalId = setInterval(() => {
      setCount((prev) => prev + 1);
    }, 1000);

    return () => {
      clearInterval(intervalId);
    };
  }, []);

  return <div>{count}</div>;
}
```

### Lifecycle

```text
Component mounts
      ↓
setInterval starts
      ↓
Component runs
      ↓
Component unmounts
      ↓
useEffect cleanup runs
      ↓
clearInterval()
      ↓
Reference released
      ↓
GC can reclaim memory
```

---

# 🎧 Event Listener Memory Leak

Another common source of memory leaks is an event listener.

## ❌ Incorrect

```jsx
useEffect(() => {
  const handleResize = () => {
    console.log(window.innerWidth);
  };

  window.addEventListener("resize", handleResize);
}, []);
```

The listener is registered but never removed.

If the component is mounted and unmounted repeatedly, unnecessary listeners may remain.

---

## ✅ Correct

```jsx
useEffect(() => {
  const handleResize = () => {
    console.log(window.innerWidth);
  };

  window.addEventListener("resize", handleResize);

  return () => {
    window.removeEventListener("resize", handleResize);
  };
}, []);
```

### Important

The same function reference should be used:

```js
window.addEventListener("resize", handleResize);

window.removeEventListener("resize", handleResize);
```

Not:

```js
window.removeEventListener("resize", () => {});
```

because that creates a different function reference.

---

# 🧹 `useEffect` Cleanup Pattern

One of the most important React patterns for preventing resource leaks:

```jsx
useEffect(() => {
  // Setup

  return () => {
    // Cleanup
  };
}, []);
```

Think of it as:

```text
Setup
  ↓
Component is active
  ↓
Unmount / Effect re-run
  ↓
Cleanup
```

---

# 🔥 Common Sources of Memory Leaks

## 1. Timers

```js
setInterval();
setTimeout();
```

Cleanup:

```js
clearInterval();
clearTimeout();
```

---

## 2. Event Listeners

```js
window.addEventListener();
document.addEventListener();
```

Cleanup:

```js
window.removeEventListener();
document.removeEventListener();
```

---

## 3. Subscriptions

For example:

```js
socket.on();
```

Cleanup:

```js
socket.off();
```

The exact cleanup API depends on the library.

---

## 4. WebSockets

If a component creates a WebSocket connection, clean it up when the component is no longer needed.

```js
socket.close();
```

---

## 5. Observers

Examples:

```text
ResizeObserver
MutationObserver
IntersectionObserver
```

Cleanup:

```js
observer.disconnect();
```

---

## 6. Async Operations

For long-running requests or operations, cancellation can be useful.

Example with `fetch`:

```jsx
useEffect(() => {
  const controller = new AbortController();

  fetch("/api/products", {
    signal: controller.signal,
  });

  return () => {
    controller.abort();
  };
}, []);
```

---

# 🛠️ How to Debug Memory Leaks

Chrome DevTools provides a **Memory** tab for investigating memory usage.

A typical debugging process:

```text
1. Reproduce the problem
        ↓
2. Take a baseline heap snapshot
        ↓
3. Perform the suspected interaction repeatedly
        ↓
4. Unmount the component
        ↓
5. Force GC where appropriate
        ↓
6. Take another heap snapshot
        ↓
7. Compare snapshots
        ↓
8. Investigate retained objects
        ↓
9. Identify the retainer
        ↓
10. Fix the cleanup
        ↓
11. Profile again
```

---

# 📸 Heap Snapshot

A heap snapshot provides information about JavaScript objects and related DOM nodes in memory.

The useful approach is to compare snapshots.

For example:

```text
Snapshot 1
   ↓
Baseline

User interacts with application
   ↓
Components mount/unmount repeatedly

Snapshot 2
   ↓
Compare with Snapshot 1
```

Look for objects whose:

- Count keeps increasing
- Retained size keeps increasing
- References remain after unmounting

---

# 📊 Allocation Timeline

Allocation Timeline helps you understand memory allocation over a period of time.

Conceptually:

```text
Blue → Memory allocation

Gray → Garbage collection / memory being freed
```

A healthy application should generally show memory being allocated and subsequently reclaimed.

If allocation keeps increasing while memory isn't being reclaimed, investigate further.

---

# 🔎 Allocation Sampling

Allocation Sampling helps identify which functions are responsible for memory allocation.

It can be useful when dealing with:

- Heavy JavaScript functions
- Large calculations
- Repeated operations
- Recursive functions
- Functions creating large amounts of data

---

# 🌐 Detached Elements

Detached elements are DOM elements that have been removed from the active DOM but may still be retained by references.

This can happen when performing direct DOM manipulation and failing to release references properly.

Chrome DevTools can help identify these retained DOM elements.

---

# 📏 Shallow Size vs Retained Size

## Shallow Size

The amount of memory directly occupied by an object.

## Retained Size

The amount of memory that could potentially be released if that object were removed.

When investigating memory leaks, **retained size is especially useful** because it helps identify objects that are keeping other memory alive.

---

# 🔗 Retainers

A **retainer** is something that keeps an object reachable.

For example:

```text
Window
  ↓
Event Listener
  ↓
Callback
  ↓
Closure
  ↓
Object
```

If an object should have been released but is still present, inspect its **retainers** to determine what is keeping it alive.

---

# ⚠️ Does Every Memory Increase Mean a Memory Leak?

**No.**

Applications naturally consume memory.

The goal is not:

```text
Memory usage = 0
```

The goal is:

```text
Necessary memory → Keep

Unnecessary memory → Release
```

Browser, V8, React, compiled JavaScript, application state, and other internal processes naturally require memory.

The concern is **unnecessary or continuously increasing memory retention** caused by application code.

---

# 🚀 Debouncing & Throttling

Debouncing and throttling can help control excessive operations.

For example, resize events can fire many times:

```text
resize
resize
resize
resize
resize
resize
```

### Debouncing

Wait until the events stop before executing.

```text
resize resize resize resize
              ↓
         wait
              ↓
          execute
```

### Throttling

Limit execution to a specific interval.

```text
resize resize resize resize resize
   ↓
execute
        ↓
     execute
              ↓
           execute
```

> Debouncing and throttling help control excessive execution, but they do **not replace cleanup**.

If you add an event listener, you still need to remove it.

---

# 🧪 Example: Complete Cleanup

A component may use multiple external resources:

```jsx
useEffect(() => {
  const intervalId = setInterval(() => {
    // work
  }, 1000);

  const handleResize = () => {
    // work
  };

  window.addEventListener("resize", handleResize);

  return () => {
    clearInterval(intervalId);
    window.removeEventListener("resize", handleResize);
  };
}, []);
```

This gives the component a clear lifecycle:

```text
Mount
 ↓
Create resources
 ↓
Use resources
 ↓
Unmount
 ↓
Cleanup resources
 ↓
GC can reclaim unused memory
```

---

# 🎯 Interview Questions

## 1. What is a memory leak?

> A memory leak occurs when memory that is no longer needed remains retained because something still holds a reference to it, preventing the Garbage Collector from reclaiming it.

---

## 2. Can React automatically prevent all memory leaks?

> No. React manages the component lifecycle, but external resources such as timers, event listeners, subscriptions, WebSockets, and observers need appropriate cleanup.

---

## 3. How do you clean up a `setInterval`?

```js
const id = setInterval(callback, 1000);

return () => {
  clearInterval(id);
};
```

---

## 4. How do you clean up an event listener?

```js
window.addEventListener("resize", handleResize);

return () => {
  window.removeEventListener("resize", handleResize);
};
```

---

## 5. What is reachability?

> Reachability describes whether an object can still be reached through references from a Garbage Collector root. If an object is no longer reachable, it can eventually be garbage collected.

---

## 6. What is a GC Root?

> A GC Root is a starting point used by the garbage collector to determine which objects are still reachable.

For browser JavaScript, `window` is an important root in this model.

---

## 7. How would you debug a memory leak?

> I would reproduce the issue, take a baseline heap snapshot, perform the problematic interaction multiple times, force garbage collection where appropriate, take another snapshot, compare them, and inspect objects with increasing counts or retained sizes. I would then inspect their retainers to find what is keeping them alive.

---

## 8. What is the difference between shallow size and retained size?

> Shallow size is the memory directly occupied by an object, while retained size represents the memory that could become reclaimable if that object were removed.

---

## 9. What are common causes of memory leaks in React?

```text
Timers
Event listeners
Subscriptions
WebSockets
Observers
Uncancelled async operations
Unnecessary references
Detached DOM elements
```

---

# 🎤 60-Second Interview Answer

> **“A memory leak occurs when an application retains memory that is no longer needed, preventing JavaScript's Garbage Collector from reclaiming it.**
>
> **In React, a common example is creating a timer or event listener inside `useEffect` and not cleaning it up. When the component unmounts, React removes it from the React tree, but the timer or listener can continue holding references to the component's closure.**
>
> **I prevent this by returning a cleanup function from `useEffect`. For example, I use `clearInterval()` for timers, `removeEventListener()` for event listeners, unsubscribe from subscriptions, close WebSockets, disconnect observers, and cancel requests where appropriate.**
>
> **For debugging, I use Chrome DevTools' Memory tab. I can take heap snapshots before and after reproducing the issue, compare them, inspect retained objects and retainers, and use Allocation Timeline or Allocation Sampling to identify where memory is being allocated and whether it is being released.”**

---

# 🧠 Quick Revision

Remember this:

```text
MEMORY LEAK
    ↓
Unnecessary memory remains retained
    ↓
GC cannot reclaim it
    ↓
Memory usage increases
    ↓
Performance can degrade
```

### React Rule

```jsx
useEffect(() => {
  // Setup

  return () => {
    // Cleanup
  };
}, []);
```

### Common Cleanup

| Resource       | Cleanup                   |
| -------------- | ------------------------- |
| `setInterval`  | `clearInterval`           |
| `setTimeout`   | `clearTimeout`            |
| Event listener | `removeEventListener`     |
| Subscription   | `unsubscribe` / `off`     |
| WebSocket      | `close`                   |
| Observer       | `disconnect`              |
| Fetch          | `AbortController.abort()` |

### Debugging Tools

```text
Chrome DevTools
       │
       └── Memory
            ├── Heap Snapshot
            ├── Allocation Timeline
            ├── Allocation Sampling
            └── Detached Elements
```

---

## ⭐ One Sentence to Remember

> **A memory leak occurs when something that should no longer be needed is still reachable through a reference, preventing the Garbage Collector from reclaiming its memory.**
