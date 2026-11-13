# 30 — Class Components & Legacy React

> **Interview + Understanding Guide** — Roadmap Section 30
>
> Goal: Read and maintain class-component codebases, map lifecycle methods to hooks, and answer legacy interview questions (`setState` behaviour, `PureComponent`, `shouldComponentUpdate`, `getDerivedStateFromProps`, HOCs with `connect`).

---

# 1. Class Component Anatomy

```jsx
import { Component } from "react";

class Counter extends Component {
  state = { count: 0 };           // class field (or this.state in constructor)

  increment = () => {             // arrow class field → `this` bound automatically
    this.setState((prev) => ({ count: prev.count + 1 }));
  };

  render() {
    return (
      <button onClick={this.increment}>
        {this.props.label}: {this.state.count}
      </button>
    );
  }
}
```

### Constructor style

```jsx
class Counter extends Component {
  constructor(props) {
    super(props);                          // must call before using `this`
    this.state = { count: 0 };
    this.increment = this.increment.bind(this); // bind or `this` is undefined in handler
  }
  increment() {
    this.setState({ count: this.state.count + 1 });
  }
}
```

---

# 2. `setState` Behaviour ⭐

```jsx
this.setState({ count: 1 });                             // shallow MERGE into state (unlike useState)
this.setState((prev, props) => ({ count: prev.count + props.step })); // updater
this.setState({ open: true }, () => console.log("updated", this.state)); // callback after commit
```

- **Asynchronous / batched**: `this.state` isn't updated immediately after calling `setState`.
- **Merges** top-level keys (useState **replaces**).
- Calling with the same value **still re-renders** (unless `PureComponent`/`shouldComponentUpdate`) — `useState` bails out.

```jsx
handleClick = () => {
  this.setState({ count: this.state.count + 1 });
  this.setState({ count: this.state.count + 1 });
  // → +1 (batched, same stale this.state)
};
```

---

# 3. Lifecycle Methods ⭐

```text
MOUNTING      constructor → static getDerivedStateFromProps → render → (DOM) → componentDidMount
UPDATING      static getDerivedStateFromProps → shouldComponentUpdate → render
              → getSnapshotBeforeUpdate → (DOM) → componentDidUpdate
UNMOUNTING    componentWillUnmount
ERRORS        static getDerivedStateFromError → componentDidCatch
```

| Method | Purpose |
| --- | --- |
| `constructor` | Initialize state, bind methods |
| `static getDerivedStateFromProps(props, state)` | Rare: derive state from props before every render (return partial state or null) |
| `render` | Pure: return JSX |
| `componentDidMount` | Side effects after first render: fetch, subscribe, DOM measurement |
| `shouldComponentUpdate(nextProps, nextState)` | Return `false` to skip re-render (performance) |
| `getSnapshotBeforeUpdate(prevProps, prevState)` | Read DOM before it changes (e.g. scroll position); value passed to didUpdate |
| `componentDidUpdate(prevProps, prevState, snapshot)` | Side effects after updates; compare with prev values |
| `componentWillUnmount` | Cleanup: timers, listeners, subscriptions |
| `getDerivedStateFromError` / `componentDidCatch` | Error boundaries |

### Removed "unsafe" lifecycles

`componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate` → renamed `UNSAFE_*` in 16.3 because they ran in the render phase and broke with async/concurrent rendering. Replacements: constructor/didMount, `getDerivedStateFromProps`/didUpdate, `getSnapshotBeforeUpdate`.

```jsx
class ChatRoom extends Component {
  componentDidMount() {
    this.connection = createConnection(this.props.roomId);
    this.connection.connect();
  }
  componentDidUpdate(prevProps) {
    if (prevProps.roomId !== this.props.roomId) {   // must compare manually!
      this.connection.disconnect();
      this.connection = createConnection(this.props.roomId);
      this.connection.connect();
    }
  }
  componentWillUnmount() {
    this.connection.disconnect();
  }
  render() { return <h1>Room {this.props.roomId}</h1>; }
}
```

Same logic with hooks — one effect, no manual comparison:

```jsx
useEffect(() => {
  const c = createConnection(roomId);
  c.connect();
  return () => c.disconnect();
}, [roomId]);
```

---

# 4. Lifecycle → Hooks Mapping ⭐⭐⭐

| Class | Hooks |
| --- | --- |
| `constructor` / `this.state` | `useState` / `useReducer` (lazy init) |
| `componentDidMount` | `useEffect(() => {…}, [])` |
| `componentDidUpdate` (on specific changes) | `useEffect(() => {…}, [dep])` |
| `componentWillUnmount` | Cleanup returned from `useEffect` |
| `shouldComponentUpdate` / `PureComponent` | `React.memo` (+ `useMemo`/`useCallback`) |
| `getDerivedStateFromProps` | Compute during render / `key` reset / set state during render |
| `getSnapshotBeforeUpdate` | `useLayoutEffect` + refs (no exact equivalent) |
| `componentDidCatch` / `getDerivedStateFromError` | **No hook** — still need a class (or `react-error-boundary`) |
| Instance variables (`this.timer`) | `useRef` |
| `this.forceUpdate()` | `const [, force] = useReducer((x) => x + 1, 0)` |
| `contextType` / `<Consumer>` | `useContext` |
| `createRef` | `useRef` |

> Mental shift: classes think in **lifecycle moments**; hooks think in **synchronization** with deps (`06-useeffect-deep-dive.md`).

---

# 5. `PureComponent` & `shouldComponentUpdate`

```jsx
class Row extends PureComponent {   // shallow compares props & state → skips render if equal
  render() { return <li>{this.props.item.name}</li>; }
}

class Chart extends Component {
  shouldComponentUpdate(nextProps) {
    return nextProps.data !== this.props.data; // custom check
  }
}
```

Same pitfalls as `React.memo`: inline functions/objects break shallow equality; mutations go undetected.

---

# 6. `getDerivedStateFromProps` (and why it's rare)

```jsx
class EmailInput extends Component {
  state = { email: this.props.defaultEmail, prevUserId: this.props.userId };

  static getDerivedStateFromProps(props, state) {
    if (props.userId !== state.prevUserId) {
      return { email: props.defaultEmail, prevUserId: props.userId }; // reset when user changes
    }
    return null;
  }
}
```

Usually better: **fully controlled** component, or **`key={userId}`** to reset.

---

# 7. Legacy Patterns You'll See

### Redux `connect` HOC

```jsx
const mapStateToProps = (state) => ({ todos: state.todos });
const mapDispatchToProps = { addTodo };
export default connect(mapStateToProps, mapDispatchToProps)(TodoList);
// Modern: useSelector / useDispatch
```

### Render props / HOCs for logic sharing

`withRouter(Component)`, `<Query>{({ data }) => …}</Query>` → replaced by hooks (`useNavigate`, `useQuery`).

### Other legacy APIs (removed in React 19)

- String refs: `ref="input"` → `this.refs.input`
- Legacy context: `childContextTypes`, `getChildContext`
- `ReactDOM.render`, `findDOMNode`
- `propTypes` runtime checks (replaced by TypeScript); `defaultProps` on function components

### Mixins (React.createClass era)

Removed long ago — replaced by HOCs, then hooks.

---

# 8. Migrating Classes to Hooks

1. Don't rewrite working classes just for style — migrate when touching code.
2. Convert state fields to `useState`/`useReducer`.
3. Merge `didMount`/`didUpdate`/`willUnmount` pieces **per concern** into separate effects.
4. Replace instance variables with `useRef`.
5. Replace `connect`/HOCs with hooks.
6. Keep error boundaries as classes.
7. Add tests before refactoring.

---

# 9. Output Questions 🧪

### Q1

```jsx
class A extends Component {
  state = { n: 0 };
  click = () => {
    this.setState({ n: this.state.n + 1 });
    console.log(this.state.n);
  };
  render() { return <button onClick={this.click}>{this.state.n}</button>; }
}
```
First click logs?

<details><summary>Answer</summary>`0` — `setState` is batched; `this.state` not yet updated.</details>

### Q2

```jsx
class B extends Component {
  state = { a: 1, b: 2 };
  componentDidMount() { this.setState({ a: 10 }); }
  render() { return <p>{this.state.a} {this.state.b}</p>; }
}
```

<details><summary>Answer</summary>`10 2` — class `setState` merges.</details>

### Q3

```jsx
class C extends Component {
  handleClick() { console.log(this); }
  render() { return <button onClick={this.handleClick}>x</button>; }
}
```

<details><summary>Answer</summary>`undefined` — method not bound; class bodies are strict mode.</details>

---

# 10. Interview Answers

**Map class lifecycles to hooks.**

> "State maps to `useState` or `useReducer`. `componentDidMount`, `componentDidUpdate` and `componentWillUnmount` collapse into `useEffect` with dependencies and a cleanup function — grouped by concern rather than lifecycle. `shouldComponentUpdate` and `PureComponent` map to `React.memo`. Instance variables become `useRef`. Error boundaries have no hook equivalent, so they remain classes."

**Why were hooks introduced?**

> "Classes made it hard to reuse stateful logic — HOCs and render props led to wrapper hell — and spread related logic across lifecycle methods while mixing unrelated logic in the same method. `this` binding was confusing and classes were harder to optimize. Hooks let you compose logic in custom hooks and organize code by concern."

---

# 11. Key Takeaways

- Class `setState` merges, batches, has a callback; no same-value bail-out.
- Know mount/update/unmount/error lifecycle order and purposes.
- `UNSAFE_` lifecycles removed for concurrent safety.
- Mapping to hooks; error boundaries still classes.
- `PureComponent` ≈ `memo`; `getDerivedStateFromProps` is rarely needed.
- Recognize `connect`, HOCs, render props, string refs, legacy context.

## Readiness Checklist

- [ ] Write a class with constructor binding vs arrow fields
- [ ] Recite lifecycle order for mount and update
- [ ] Convert a class with subscription logic to hooks
- [ ] Explain why `componentWillReceiveProps` was deprecated
