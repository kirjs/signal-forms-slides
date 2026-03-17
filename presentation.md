<div class="title-slide">

# What's New in Signal Forms

<div class="subtitle">Angular Framework · 2026</div>

<div class="author">
<img src="kirjs.jpg" alt="kirjs" />
<div class="author-info">
<span class="author-name">Kirill Cherkashin</span>
<span class="author-handle">@kirjs</span>
</div>
</div>

</div>
---

## 🏴‍☠️🏴‍☠️🏴‍☠️🏴‍☠️
## Warning, the following presentation might include some actual code samples.
---

## Why Signal Forms?

---

### Template Forms

Simple and declarative. Great for getting started.

- Logic bleeds into templates
- Hard to test
- No type safety
- Doesn't scale

---

### Reactive Forms

Powerful and explicit. Logic lives in the component.

- Verbose
- Form model diverges from your data model
- Updating arrays is a ceremony
- Reacting to changes means `valueChanges` subscriptions everywhere

---

### ControlValueAccessor

Building a custom form control? Meet your new friend.

```typescript

@Component({
  selector: 'star-ratings',
  standalone: true,
  template: `...`,
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => StarRatings),
    multi: true
  }]
})
export class StarRatings implements ControlValueAccessor {
  
  writeValue(value: number) { ...
  }

  registerOnChange(fn: (v: number) => void) { ...
  }

  registerOnTouched(fn: () => void) { ...
  }

  setDisabledState(isDisabled: boolean) { ...
  }
}
```

---
## Meet Signal Forms 🔥
### ⚠️ Experimental, but stable soon!
---

### You own the data

```typescript
const feedback = signal({ name: '', email: '', rating: 5 });

const f = form(this.feedback);
```

The signal is yours. Signal forms wraps it — it doesn't own it.

---

### Type safety end-to-end

```typescript
const f = form(this.feedback);
// f.email        — FieldTree<string>
// f.email()      — FieldState<string>
// f.typo         — compile error ✗
```

```html
<form [formRoot]="f">
  <input [formField]="f.name" />
  <input [formField]="f.email" />
</form>
```

---

### Logic lives in one place

```typescript
const f = form(this.feedback, (p) => {
  required(p.email);
  email(p.email);
});
```

---

###  All the states updates reactively

```typescript
const f = form(this.feedback, (p) => {
  required(p.email);
  email(p.email);
  debounce(p.email, 300);
  disabled(p.feedback, ({ valueOf }) => valueOf(p.rating) < 3);
  validate(p.confirmPassword, ({ value, valueOf }) =>
    value() !== valueOf(p.password)
      ? customError({ message: 'Must match' })
      : undefined
  );
});
```

---

### Custom controls are just components

```typescript
@Component({ selector: 'star-ratings' })
export class StarRatings implements FormValueControl {
  readonly value = model(0);
  readonly disabled = input(false);
}
```

```html
<star-ratings [formField]="f.rating" />
```

---
## What's new in Signal Forms
---

### `[field]` → `[formField]`

`[field]` was too generic — likely to collide with existing components.

```html
<!-- Before -->
<input [field]="f.email"/>

<!-- After -->
<input [formField]="f.email"/>
```

---

### `formRoot` directive

Wires your `<form>` element to the signal forms model automatically.

```html

<form [formRoot]="f">
  ...
</form>
```

- Sets `novalidate` — disables browser validation
- Intercepts `submit`, calls `submit(f)` for you
- No manual event wiring needed

---

### Form submission — declarative

```typescript
f = form(this.model, (p) => {
  required(p.email);
}, {
  submission: {
    action: async () => { saveData(this.model()); }
  }
});
```

```html
<form [formRoot]="f">
  <button type="submit" [disabled]="f().invalid()">Submit</button>
</form>
```

`formRoot` handles `preventDefault` and triggers the action automatically.

---

### Form submission — imperative

```typescript
onSubmit() {
  if (this.f().valid()) {
    saveData(this.model());
  }
}
```

```html
<form (submit)="onSubmit()">
  <button type="submit" [disabled]="f().invalid()">Submit</button>
</form>
```

---

### Async Validation > The problem

```typescript
validateAsync(p.email, {
  params: ({ value }) => value(),
  factory: (email) => checkEmailResource(email),
  onSuccess: (isTaken) => isTaken
    ? customError({ message: 'Email already taken' })
    : undefined,
  onError: () => customError({ message: 'Check failed' }),
});
// ↑ fires a new request on every keystroke
```

---

### Async Validation > `debounce()`

```typescript
const f = form(this.model, (p) => {
  debounce(p.email, 300);

  validateAsync(p.email, {
    params: ({ value }) => value(),
    factory: (email) => checkEmailResource(email),
    onSuccess: (isTaken) => isTaken
      ? customError({ message: 'Email already taken' })
      : undefined,
    onError: () => customError({ message: 'Check failed' }),
  });
});
```

Also fires immediately on blur.

---

### Async Validation > `controlValue` vs `value`

`debounce()` splits the field value into two:

```typescript
f.email().controlValue  // WritableSignal — what the user is typing right now
f.email().value         // WritableSignal — debounced, what validators see
```

Setting `value` directly cancels any pending debounce.

---

### Advanced Controls > The problem

A custom date picker may be in a state that can't produce a valid value yet — e.g. the user typed `"31/02"`.

The control can't set a `Date` on the model. Previously there was no way to signal this.

---

### Advanced Controls > `parseErrors`

```typescript

@Component({selector: 'date-picker'})
export class DatePicker implements FormValueControl {
  readonly value = model<Date | null>(null);

  readonly parseErrors = signal<ValidationError[]>([]);

  onInput(raw: string) {
    const parsed = parseDate(raw);
    if (!parsed) {
      this.parseErrors.set([customError({message: 'Invalid date'})]);
    } else {
      this.parseErrors.set([]);
      this.value.set(parsed);
    }
  }
}
```

Parse errors surface in `f.date().errors()` like any other validation error.

---

### Advanced Controls > Native input parsing

Number inputs now parse properly — no more string/number confusion.

```typescript
const f = form(signal({count: 0}));
// <input type="number" [formField]="f.count" />

f.count().value() // number, not string

// Empty input → null, not ""
f.count().value() // null
```

---

### `SignalFormControl`

Use signal forms rules inside a reactive `FormGroup`.

```typescript
const form = new FormGroup({
  name: new SignalFormControl('', (p) => {
    required(p);
    minLength(p, 2);
  }),
  age: new FormControl(25), // regular reactive forms control
});
```

```html

<form [formGroup]="form">
  <input [formField]="nameControl.fieldTree"/>
  <input formControlName="age"/>
</form>
```

---

### `extractValue`

Unwraps a `FieldTree` into a plain value, even when it wraps `AbstractControl` instances.

```typescript
const f = compatForm(signal({
  name: new FormControl('Alice'),
  age: 30,
}));

extractValue(f); // { name: 'Alice', age: 30 }
```

Filter by state:

```typescript
extractValue(f, {dirty: true});   // only dirty fields
extractValue(f, {touched: true}); // only touched fields
```

  
---

### Focus management

Programmatically focus the native control bound to a field.

```typescript
f.email().focusBoundControl();

// With options
f.email().focusBoundControl({preventScroll: true});
```

Useful for directing user attention after submit or on error.

---
<div class="title-slide">

# Questions?

<div class="subtitle">Angular Framework · 2026</div>

<div class="author">
<img src="kirjs.jpg" alt="kirjs" />
<div class="author-info">
<span class="author-name">Kirill Cherkashin</span>
<span class="author-handle">@kirjs</span>
</div>
</div>

</div>
