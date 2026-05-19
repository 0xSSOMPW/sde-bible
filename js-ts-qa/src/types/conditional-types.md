# Q: Explain conditional types in TypeScript. What is distribution and how does `infer` work?

**Answer:**

A **conditional type** has the form `T extends U ? X : Y`. It chooses between two type branches based on assignability — the type-level analogue of a ternary. Combined with `infer`, conditional types let you take types apart and rebuild them.

### Basic Shape

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<'hi'>;  // true
type B = IsString<42>;    // false
```

### Pattern Matching with `infer`

`infer X` introduces a new type variable inside the `extends` clause, captured from the structure being matched:

```typescript
type ReturnType<F>   = F extends (...args: any[]) => infer R ? R : never;
type Awaited<T>      = T extends Promise<infer U> ? Awaited<U> : T;
type FirstArg<F>     = F extends (a: infer A, ...rest: any[]) => any ? A : never;
type ElementOf<A>    = A extends (infer E)[] ? E : never;
```

`infer` only lives inside the true branch.

### Distribution Over Unions

If the *checked* type (left of `extends`) is a **naked** type parameter and is a union, the conditional **distributes** over each member:

```typescript
type ToArray<T> = T extends any ? T[] : never;
type R = ToArray<string | number>;  // string[] | number[]   (NOT (string | number)[])
```

To turn distribution off, wrap both sides in a tuple:

```typescript
type ToArrayNoDist<T> = [T] extends [any] ? T[] : never;
type R2 = ToArrayNoDist<string | number>;  // (string | number)[]
```

> [!NOTE]
> Distribution is incredibly useful — it's how `Exclude`, `Extract`, `NonNullable` filter unions:
> ```typescript
> type Exclude<T, U> = T extends U ? never : T;
> type Extract<T, U> = T extends U ? T : never;
> ```

### Useful Built-Ins Reframed

```typescript
type NonNullable<T> = T extends null | undefined ? never : T;
type Parameters<F>  = F extends (...args: infer P) => any ? P : never;
type InstanceType<C> = C extends new (...a: any[]) => infer I ? I : any;
```

### Real-World Example: API Endpoints

```typescript
type Endpoints = {
  '/users':   { method: 'GET';  res: User[] };
  '/users/:id': { method: 'GET'; res: User };
  '/login':   { method: 'POST'; body: { email: string; pwd: string }; res: { token: string } };
};

type ResponseOf<P extends keyof Endpoints> =
  Endpoints[P] extends { res: infer R } ? R : never;

type BodyOf<P extends keyof Endpoints> =
  Endpoints[P] extends { body: infer B } ? B : never;

type R = ResponseOf<'/users'>;     // User[]
type B = BodyOf<'/login'>;         // { email: string; pwd: string }
```

You can now write a `fetch` wrapper whose response type is keyed on the URL literal.

### Recursive Conditional Types

```typescript
type Flatten<T> = T extends (infer U)[]
  ? Flatten<U>
  : T;

type X = Flatten<number[][][]>;  // number
```

TypeScript 4.1+ allows recursion in conditional types up to an instantiation depth limit.

### Distributing Object Properties

Combine conditional types with mapped types to filter keys by value type:

```typescript
type KeysOfType<T, V> = {
  [K in keyof T]-?: T[K] extends V ? K : never;
}[keyof T];

interface User { id: number; name: string; isAdmin: boolean; age: number; }
type NumericKeys = KeysOfType<User, number>;  // "id" | "age"
```

### Gotchas

- `any extends X ? A : B` resolves to `A | B` (not `A`). `any` is contagious.
- `never extends X ? A : B` is `never` because distribution over an empty union gives nothing.
- Order matters in chained conditionals — they're checked top-down.
