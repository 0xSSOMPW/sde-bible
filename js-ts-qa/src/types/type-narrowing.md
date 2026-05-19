# Q: How does TypeScript narrow types? Explain discriminated unions, exhaustiveness, and assertion functions.

**Answer:**

**Narrowing** is the process by which TypeScript refines a value's type as it flows through control-flow constructs. Every `if`, `switch`, `typeof`, `instanceof`, equality check, and user-defined predicate participates in the *control-flow analysis* the compiler performs.

### The Built-In Narrowers

| Construct                         | Narrows                                  |
|------------------------------------|------------------------------------------|
| `typeof x === 'string'`           | primitives                                |
| `x instanceof Foo`                | classes                                   |
| `'prop' in x`                     | object shapes                             |
| `x == null` / `x === undefined`   | nullability                               |
| Literal equality `x === 'admin'`  | string/number literal members             |
| `Array.isArray(x)`                | tuple/array vs object                     |
| Truthiness `if (x)`               | strips `0 | '' | null | undefined | NaN`  |

```typescript
function format(x: string | number | null) {
  if (x == null) return '-';
  // x: string | number
  if (typeof x === 'string') return x.trim(); // x: string
  return x.toFixed(2);                        // x: number
}
```

### Discriminated Unions

The canonical pattern: every member has a literal-typed *discriminant* property.

```typescript
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'square'; side: number }
  | { kind: 'rect'; w: number; h: number };

function area(s: Shape): number {
  switch (s.kind) {
    case 'circle': return Math.PI * s.radius ** 2;
    case 'square': return s.side ** 2;
    case 'rect':   return s.w * s.h;
  }
}
```

The compiler can prove this `switch` is exhaustive.

### Exhaustiveness with `never`

```typescript
function area(s: Shape): number {
  switch (s.kind) {
    case 'circle': return ...;
    case 'square': return ...;
    case 'rect':   return ...;
    default:
      const _exhaustive: never = s; // compile error if a new variant is added
      throw new Error(`Unhandled: ${(_exhaustive as any).kind}`);
  }
}
```

When you add `{ kind: 'triangle'; ... }` to `Shape`, the `default` branch's `s` is no longer `never`, so the assignment fails — the compiler points you at every place to update.

### User-Defined Type Guards

A function with return type `arg is Type` is a *predicate* that narrows on truthy return:

```typescript
function isString(x: unknown): x is string {
  return typeof x === 'string';
}

function example(v: unknown) {
  if (isString(v)) v.toUpperCase(); // v: string
}
```

For runtime-validated data (parsed JSON, API responses), pair a predicate with a schema validator (Zod, Valibot) and use it as the type-guard return.

### Assertion Functions

`asserts` declarations narrow without returning a boolean — they throw on failure:

```typescript
function assertString(v: unknown): asserts v is string {
  if (typeof v !== 'string') throw new TypeError('not a string');
}

function use(v: unknown) {
  assertString(v);
  v.toUpperCase(); // v: string from here onward
}
```

Useful with frameworks that perform invariant checks (`assert(x, msg)`).

### `in` Narrowing for Object Shapes

```typescript
type Cat = { meow(): void };
type Dog = { bark(): void };

function speak(a: Cat | Dog) {
  if ('meow' in a) a.meow();
  else             a.bark();
}
```

> [!NOTE]
> Prefer discriminated unions over `in`-based duck typing. Discriminants survive serialization, refactoring, and tooling far better.

### Control-Flow Pitfalls

**1. Aliasing breaks narrowing.**

```typescript
function f(o: { v?: string }) {
  if (o.v !== undefined) {
    const cb = () => o.v.trim(); // error: o.v could be undefined
  }
}
```

The async callback runs later; the compiler conservatively widens. Capture the narrowed value first:

```typescript
const v = o.v;
if (v !== undefined) const cb = () => v.trim();
```

**2. `let` widening.**

```typescript
let s: 'a' | 'b' = 'a';
function set(x: string) { s = x; } // error — but the compiler may widen elsewhere
```

Use `as const` and `readonly` to keep literal types narrow.

**3. `unknown` is the gate.**

Treat external input as `unknown` and narrow explicitly. Using `any` defeats every checker in the codebase.
