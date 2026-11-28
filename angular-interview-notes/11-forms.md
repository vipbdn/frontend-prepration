# 11 — Forms 🔥🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 11
>
> Goal: Master all three form approaches — template-driven, typed reactive forms, and **Signal Forms (stable in v22)** — plus validation (sync, async, cross-field), dynamic forms with `FormArray`, and custom form controls (`ControlValueAccessor` / `FormValueControl`).

---

# 1. Three Approaches ⭐

| | Template-driven | Reactive (typed) | Signal Forms (v22 stable) |
| --- | --- | --- | --- |
| Module/API | `FormsModule`, `ngModel` | `ReactiveFormsModule`, `FormGroup`/`FormControl` | `@angular/forms/signals`: `form()`, `[formField]` |
| Model lives in | Template (directives create controls) | Component class (explicit control tree) | **A signal** of your data model |
| Data flow | Async, two-way | Sync, observable `valueChanges` | Signal-based, fine-grained |
| Validation | Directives (`required`, custom directive) | Validator functions | Schema functions (`required(p.email)`) |
| Typing | Weak | Strong (typed since v14) | Strong, inferred from model |
| Testing | Harder (needs rendering) | Easy (pure class) | Easy |
| Best for | Simple forms | Complex/dynamic forms, existing apps | New apps, signal-first code |

---

# 2. Template-Driven Forms

```ts
@Component({
  selector: "app-newsletter",
  imports: [FormsModule],
  template: `
    <form #f="ngForm" (ngSubmit)="submit(f)">
      <input name="email" [(ngModel)]="model.email" required email #email="ngModel" />
      @if (email.invalid && email.touched) {
        @if (email.errors?.["required"]) { <small>Email is required</small> }
        @if (email.errors?.["email"]) { <small>Invalid email</small> }
      }
      <button [disabled]="f.invalid">Subscribe</button>
    </form>
  `,
})
export class Newsletter {
  model = { email: "" };
  submit(f: NgForm) {
    console.log(f.value);
  }
}
```

Each `ngModel` with a `name` registers a `FormControl` in the `NgForm`. CSS state classes: `ng-valid/invalid`, `ng-touched/untouched`, `ng-dirty/pristine`, `ng-pending`.

---

# 3. Reactive Forms ⭐⭐⭐

```ts
import { FormBuilder, ReactiveFormsModule, Validators } from "@angular/forms";

@Component({
  selector: "app-register",
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <input formControlName="name" />
      <input formControlName="email" type="email" />
      @if (form.controls.email.hasError("email") && form.controls.email.touched) {
        <small>Invalid email</small>
      }

      <div formGroupName="passwords">
        <input formControlName="password" type="password" />
        <input formControlName="confirm" type="password" />
      </div>
      @if (form.controls.passwords.hasError("mismatch")) { <small>Passwords don't match</small> }

      <button [disabled]="form.invalid || form.pending">Register</button>
    </form>
  `,
})
export class Register {
  private readonly fb = inject(FormBuilder).nonNullable;    // controls reset to initial value, not null

  protected readonly form = this.fb.group({
    name: ["", [Validators.required, Validators.minLength(2)]],
    email: ["", [Validators.required, Validators.email], [uniqueEmailValidator()]],   // 3rd arg: async validators
    passwords: this.fb.group(
      { password: ["", [Validators.required, Validators.minLength(8)]], confirm: [""] },
      { validators: matchFields("password", "confirm") },
    ),
    newsletter: [false],
  });

  submit() {
    if (this.form.invalid) {
      this.form.markAllAsTouched();          // reveal errors
      return;
    }
    const value = this.form.getRawValue();   // typed; includes disabled controls (form.value excludes them)
    console.log(value.email);
  }
}
```

### Building blocks

| Class | Represents |
| --- | --- |
| `FormControl<T>` | Single value |
| `FormGroup<{...}>` | Fixed set of named controls (object) |
| `FormArray<T>` | Dynamic list of controls |
| `FormRecord<T>` | Dynamic keys, same control type |

### Key API

```ts
control.value; control.valid; control.errors; control.touched; control.dirty; control.status;
control.setValue(v);          // full value (must match structure for groups)
control.patchValue(partial);  // partial update
control.reset();
control.disable(); control.enable();
control.setValidators([...]); control.updateValueAndValidity();
control.valueChanges.pipe(...)   // Observable
control.events                   // unified events Observable (v18+): value, status, touched, pristine, submit, reset
{ updateOn: "blur" }             // validate/update on blur or submit instead of change
```

---

# 4. Validation ⭐⭐

## Built-in

`Validators.required`, `requiredTrue`, `email`, `min`, `max`, `minLength`, `maxLength`, `pattern`, `Validators.compose`.

## Custom sync validator

```ts
import { AbstractControl, ValidationErrors, ValidatorFn } from "@angular/forms";

export function forbiddenWord(word: string): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null =>
    String(control.value ?? "").toLowerCase().includes(word) ? { forbiddenWord: { word } } : null;
}
```

Return `null` when valid, an error object when invalid.

## Cross-field validator (on the group)

```ts
export function matchFields(a: string, b: string): ValidatorFn {
  return (group: AbstractControl) =>
    group.get(a)?.value === group.get(b)?.value ? null : { mismatch: true };
}
```

## Async validator

```ts
export function uniqueEmailValidator(): AsyncValidatorFn {
  const api = inject(UserApi);          // called in the component's injection context
  return (control) =>
    timer(400).pipe(                    // debounce: re-validation unsubscribes the previous timer
      switchMap(() => api.emailExists(control.value)),
      map((exists) => (exists ? { emailTaken: true } : null)),
      catchError(() => of(null)),
    );
}
```

Async validators run **only after sync validators pass**; status is `PENDING` meanwhile.

---

# 5. Dynamic Forms with `FormArray` ⭐

```ts
@Component({
  selector: "app-invoice",
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form">
      <div formArrayName="items">
        @for (item of items.controls; track item; let i = $index) {
          <div [formGroupName]="i">
            <input formControlName="description" />
            <input formControlName="qty" type="number" />
            <button type="button" (click)="items.removeAt(i)">✕</button>
          </div>
        }
      </div>
      <button type="button" (click)="addItem()">Add item</button>
    </form>
  `,
})
export class Invoice {
  private readonly fb = inject(FormBuilder).nonNullable;
  protected readonly form = this.fb.group({ items: this.fb.array([this.createItem()]) });

  get items() {
    return this.form.controls.items;
  }

  createItem() {
    return this.fb.group({ description: ["", Validators.required], qty: [1, Validators.min(1)] });
  }

  addItem() {
    this.items.push(this.createItem());
  }
}
```

---

# 6. Signal Forms ⭐⭐⭐ (stable in v22) 🆕

The model is a **signal you own**; `form()` creates a **field tree** mirroring it, with validation defined in a **schema function**.

```ts
import { Component, signal } from "@angular/core";
import { form, FormField, required, email, minLength, validate, submit, debounce } from "@angular/forms/signals";

interface Signup { name: string; email: string; password: string; confirm: string; age: number }

@Component({
  selector: "app-signup",
  imports: [FormField],
  template: `
    <form (submit)="onSubmit($event)" novalidate>
      <input [formField]="f.name" />
      @if (f.name().touched() && f.name().invalid()) {
        @for (e of f.name().errors(); track e.kind) { <small>{{ e.message }}</small> }
      }

      <input type="email" [formField]="f.email" />
      <input type="password" [formField]="f.password" />
      <input type="password" [formField]="f.confirm" />
      <input type="number" [formField]="f.age" />

      <button [disabled]="f().invalid() || f().submitting()">Sign up</button>
    </form>
    <pre>{{ model() | json }}</pre>
  `,
})
export class SignupForm {
  protected readonly model = signal<Signup>({ name: "", email: "", password: "", confirm: "", age: 18 });

  protected readonly f = form(this.model, (p) => {
    required(p.name, { message: "Name is required" });
    required(p.email);
    email(p.email, { message: "Enter a valid email" });
    debounce(p.email, "blur");                     // v22: update model on blur (or ms)
    minLength(p.password, 8);
    validate(p.confirm, ({ value, valueOf }) =>    // cross-field
      value() === valueOf(p.password) ? undefined : { kind: "mismatch", message: "Passwords don't match" },
    );
  });

  protected async onSubmit(event: Event) {
    event.preventDefault();
    await submit(this.f, async (field) => {
      const result = await this.api.signup(field().value());
      return result.ok ? undefined : [{ kind: "server", message: result.error, fieldTree: field.email }];  // server errors map to fields
    });
  }

  private readonly api = inject(AuthApi);
}
```

### Concepts

- `f.name` is a **field**; `f.name()` returns its **state**: `value`, `valid`, `invalid`, `errors`, `touched`, `dirty`, `disabled`, `hidden`, `readonly`, `pending`.
- `[formField]` (the `FormField` directive) binds native inputs and custom controls two-way to the field.
- The **model signal stays the source of truth** — `model()` always holds current data.
- Custom validators return a `{ kind, message? }` error (or an array) when invalid and **`undefined`** when valid (classic reactive validators return `null`). Server errors from `submit()` target a field with `fieldTree`.
- Schema rules: `required`, `email`, `min`, `max`, `minLength`, `maxLength`, `pattern`, `validate`, `validateTree`, `disabled(p.x, cond)`, `hidden`, `readonly`, `applyEach` (arrays), `applyWhen` (conditional), reusable `schema()`.
- 🆕 v22 additions: `minDate`/`maxDate`, `debounce(field, 'blur' | ms)`, `validateAsync` / `validateHttp` with debounce, `reloadValidation()`, `getError()`, `touched` input + `touch()` output on custom controls.

```ts
// async/HTTP validation
validateHttp(p.username, {
  request: ({ value }) => `/api/users/check?name=${value()}`,
  onSuccess: (res: { taken: boolean }) => (res.taken ? { kind: "taken", message: "Username taken" } : undefined),
  onError: () => undefined,
  debounce: 300,
});
```

### Interop

- `FormValueControl<T>` — interface for custom controls (`value = model<T>()`) that work with **both** Signal Forms and legacy reactive forms in v22.
- Existing `ControlValueAccessor` controls and their validators are compatible with `[formField]`.

---

# 7. Custom Form Controls ⭐⭐

## 7.1 Signal Forms style: `FormValueControl`

```ts
import { FormValueControl } from "@angular/forms/signals";

@Component({
  selector: "app-star-input",
  template: `
    @for (s of [1, 2, 3, 4, 5]; track s) {
      <button type="button" (click)="value.set(s); touch.emit()" [disabled]="disabled()">{{ s <= value() ? "★" : "☆" }}</button>
    }
  `,
})
export class StarInput implements FormValueControl<number> {
  readonly value = model(0);
  readonly disabled = input(false);
  readonly touch = output<void>();
}
// <app-star-input [formField]="f.rating" />
```

## 7.2 Classic: `ControlValueAccessor` (interview staple)

```ts
@Component({
  selector: "app-toggle-switch",
  template: `<button type="button" [class.on]="value" [disabled]="disabled" (click)="toggle()">{{ value ? "On" : "Off" }}</button>`,
  providers: [{ provide: NG_VALUE_ACCESSOR, useExisting: forwardRef(() => ToggleSwitch), multi: true }],
})
export class ToggleSwitch implements ControlValueAccessor {
  value = false;
  disabled = false;
  private onChange: (v: boolean) => void = () => {};
  private onTouched: () => void = () => {};

  writeValue(v: boolean) { this.value = !!v; }                       // model → view
  registerOnChange(fn: (v: boolean) => void) { this.onChange = fn; }  // view → model
  registerOnTouched(fn: () => void) { this.onTouched = fn; }
  setDisabledState(d: boolean) { this.disabled = d; }

  toggle() {
    this.value = !this.value;
    this.onChange(this.value);
    this.onTouched();
  }
}
// <app-toggle-switch formControlName="notifications" />
```

(In an OnPush component, call `markForCheck()` in `writeValue`/`setDisabledState`, or store state in signals.)

---

# 8. Best Practices

- Reactive or Signal Forms for anything non-trivial; typed, `nonNullable`.
- Show errors after `touched` or submit; `markAllAsTouched()` on submit.
- Debounce async validators; handle their errors.
- Disable submit while `pending`/`submitting`; prevent double submit.
- Use `getRawValue()` if disabled controls matter.
- Accessible errors: `aria-invalid`, `aria-describedby`, labels.
- Unsaved-changes guard with `form.dirty`.

---

# 9. Interview Answers

**Template-driven vs reactive forms?**

> "Template-driven forms define the model in the template with `ngModel` and directives — quick for simple forms but asynchronous and harder to test. Reactive forms build an explicit, typed control tree in the class with `FormGroup`, `FormControl` and `FormArray`, giving synchronous access, observable value changes, easy dynamic forms and unit testing. In Angular 22 there's a third, stable option — Signal Forms — where your data model is a signal and validation lives in a schema."

**What are Signal Forms?**

> "The signal-based forms API, stable in v22. You keep your model in a signal, call `form(model, schema)` to get a field tree, bind inputs with `[formField]`, and declare validation with functions like `required`, `email` or `validate` for custom and cross-field rules. Each field exposes state — value, errors, touched, dirty, pending — as signals, so the UI updates fine-grained, the model stays the single source of truth, and typing is inferred. It also supports async/HTTP validation, debouncing and `submit()` with server error mapping."

**How do you create a custom form control?**

> "Classically by implementing `ControlValueAccessor` — `writeValue`, `registerOnChange`, `registerOnTouched` and optionally `setDisabledState` — and registering it with the `NG_VALUE_ACCESSOR` multi provider using `useExisting` and `forwardRef`. With Signal Forms, implementing `FormValueControl` with a `value = model()` is enough, and in v22 such controls also work with reactive forms."

**How do you implement cross-field validation?**

> "Put a validator on the parent `FormGroup` that reads both child controls and returns an error like `{ mismatch: true }`, then display it from the group's errors. In Signal Forms I use `validate` on one field and read the other with `valueOf`."

**How do async validators work?**

> "They return an Observable or Promise of errors or null, run only after sync validators pass, and set status to `PENDING` while running. I debounce them with a timer plus `switchMap`, and handle errors so a failed call doesn't block the form. Signal Forms have `validateAsync` and `validateHttp` with a built-in debounce."

---

# 10. Key Takeaways

- Three approaches; reactive is the most-asked; Signal Forms is the v22 headline.
- Validators return `null` or an error object; group validators for cross-field; async after sync.
- `FormArray` for dynamic lists.
- CVA (classic) / `FormValueControl` (signals) for custom controls.

## Readiness Checklist

- [ ] Build a typed reactive registration form with async + cross-field validation
- [ ] Build a dynamic `FormArray`
- [ ] Rebuild the form with Signal Forms
- [ ] Implement a CVA and a `FormValueControl`
