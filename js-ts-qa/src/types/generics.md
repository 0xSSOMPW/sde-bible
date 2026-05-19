# Q: How do generics work in TypeScript? Explain constraints, defaults, and inference.

**Answer:**

A **generic** is a type parameter — a placeholder for a type that the caller (or the compiler via inference) fills in. Generics let you write reusable code that *preserves* type information across boundaries instead of erasing it to `any`.

### The Basic Shape

```typescript
function identity<T>(value: T): T {
  return value;
}

const n = identity(42);       // T inferred as number, n: number
const s = identity('hi');     // T inferred as string, s: string
const x = identity<boolean>(true); // explicit
```

Without generics you'd return `any`, losing all downstream type safety.

### Generic Interfaces and Types

```typescript
interface Box<T> { value: T; }
type Pair<K, V> = { key: K; value: V };

const b: Box<number> = { value: 1 };
```

### Constraints with `extends`

A constraint narrows what types are acceptable:

```typescript
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}

longest('abc', 'de');          // ok — strings have length
longest([1, 2], [3, 4, 5]);    // ok — arrays have length
longest(10, 20);               // error — number has no `length`
```

### Default Type Parameters

```typescript
interface ApiResponse<T = unknown> {
  ok: boolean;
  data: T;
}

const r: ApiResponse = await fetch(...).then(r => r.json()); // data: unknown
```

Use `unknown` rather than `any` as the default — it forces consumers to narrow before using `data`.

### Inference From Usage

```typescript
function pluck<T, K extends keyof T>(obj: T, keys: K[]): T[K][] {
  return keys.map(k => obj[k]);
}

const user = { id: 1, name: 'Ada', email: 'a@b.co' };
const fields = pluck(user, ['id', 'name']);
// fields: (number | string)[]
```

`keyof T` and indexed access `T[K]` are the bread and butter of generic library code (think Lodash's `_.pick`).

### Generic Constraints That Chain

```typescript
function copy<T, U extends Partial<T>>(target: T, patch: U): T & U {
  return { ...target, ...patch };
}
```

Here `U` is constrained to be a subset of `T`'s shape; the return type intersects both.

### `infer` — Conditional Inference

Inside a conditional type, `infer` introduces a placeholder for the compiler to solve:

```typescript
type ReturnOf<F> = F extends (...args: any[]) => infer R ? R : never;
type ElementOf<T> = T extends (infer U)[] ? U : never;

type R = ReturnOf<() => Promise<string>>; // Promise<string>
type E = ElementOf<number[]>;             // number
```

### Variance Pitfalls

TypeScript is mostly bivariant for function parameters (a famous historical decision), but strict mode (`strictFunctionTypes`) makes parameters contravariant. Watch out:

```typescript
type Cmp<T> = (a: T, b: T) => number;
const animalCmp: Cmp<Animal> = ...;
const dogCmp: Cmp<Dog> = animalCmp;   // ok in strict mode (contravariant)
// const animalCmp2: Cmp<Animal> = dogCmp; // error — dogCmp can't handle every Animal
```

> [!NOTE]
> When generics get hairy, draw the contravariant/covariant arrows. Inputs flip, outputs preserve.

### Common Patterns

**Generic factory with branded return:**

```typescript
function tagged<Tag extends string>(tag: Tag) {
  return <T>(value: T) => ({ tag, value } as const);
}
const userOf = tagged('user');
const u = userOf({ id: 1 }); // { readonly tag: 'user'; readonly value: {...} }
```

**Higher-order generics for HOCs:**

```typescript
function withLogger<P>(Comp: React.ComponentType<P>): React.ComponentType<P> {
  return (props: P) => {
    console.log(Comp.name, props);
    return React.createElement(Comp, props);
  };
}
```

### Anti-Patterns

- `<T extends any>` — just write `<T>`.
- `<T>` you never actually use in the signature — drop it. The compiler will warn at the call site that nothing constrains it.
- Forcing explicit type arguments where inference works fine — clutter.
