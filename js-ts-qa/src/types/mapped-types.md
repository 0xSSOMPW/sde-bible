# Q: What are mapped types and template literal types in TypeScript? Show practical examples.

**Answer:**

A **mapped type** iterates over the keys of an existing type and produces a new type whose properties are derived from them. **Template literal types** let you compose string literal types like template strings at the type level. Together they enable advanced "type DSLs".

### Mapped Type Basics

```typescript
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

type Optional<T> = {
  [K in keyof T]?: T[K];
};

type Required<T> = {
  [K in keyof T]-?: T[K];
};
```

The `-` modifier *removes* a property attribute (`readonly` or `?`), the `+` (default) adds it.

### Built-Ins Derived from Mapped Types

```typescript
type Partial<T>   = { [K in keyof T]?: T[K] };
type Readonly<T>  = { readonly [K in keyof T]: T[K] };
type Pick<T, K extends keyof T> = { [P in K]: T[P] };
type Record<K extends PropertyKey, V> = { [P in K]: V };
```

### Key Remapping with `as`

TypeScript 4.1 added `as` clauses for renaming keys:

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface User { id: number; name: string; }
type UG = Getters<User>;
// { getId: () => number; getName: () => string; }
```

`as never` filters keys out entirely:

```typescript
type OmitByValue<T, V> = {
  [K in keyof T as T[K] extends V ? never : K]: T[K];
};
```

### Template Literal Types

```typescript
type Hello<W extends string> = `hello, ${W}`;
type H = Hello<'world'>; // "hello, world"

type EventName<T extends string> = `on${Capitalize<T>}`;
type E = EventName<'click'>; // "onClick"
```

Built-in string manipulation utilities: `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize`.

### Distributing Over Unions

Template literals distribute across union components:

```typescript
type Size = 'sm' | 'md' | 'lg';
type Color = 'red' | 'blue';
type Class = `${Color}-${Size}`;
// "red-sm" | "red-md" | "red-lg" | "blue-sm" | "blue-md" | "blue-lg"
```

### Real Example: CSS-Style Variant Props

```typescript
type Spacing = 0 | 1 | 2 | 4 | 8;
type Side    = 't' | 'r' | 'b' | 'l';
type MarginClass = `m${Side | ''}-${Spacing}`;
// "m-0" | "m-1" | ... | "mt-0" | "mr-0" | ...
```

You can now constrain a `className` prop to a finite, type-safe set.

### Path Strings

```typescript
type DeepKeys<T, P extends string = ''> = {
  [K in keyof T & string]:
    T[K] extends object
      ? `${P}${K}` | DeepKeys<T[K], `${P}${K}.`>
      : `${P}${K}`;
}[keyof T & string];

interface Cfg { db: { host: string; port: number }; app: { name: string } }
type Paths = DeepKeys<Cfg>;
// "db" | "app" | "db.host" | "db.port" | "app.name"
```

This is the foundation behind type-safe `get(obj, 'a.b.c')` helpers in libraries like react-hook-form.

> [!NOTE]
> Mapped + template literals + conditional types form a small, Turing-incomplete (but practical) compile-time language. Use them sparingly — every clever helper costs readability.

### Common Patterns

**1. "DeepReadonly":**

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
```

**2. "Snake to camel":**

```typescript
type Camel<S extends string> =
  S extends `${infer A}_${infer B}${infer C}`
    ? `${A}${Uppercase<B>}${Camel<C>}`
    : S;
type X = Camel<'first_name'>; // "firstName"
```

**3. Filter keys by value:**

```typescript
type StringKeys<T> = { [K in keyof T]: T[K] extends string ? K : never }[keyof T];
```

### When To Stop

If a mapped type takes more than two reads to understand, alias intermediate steps with descriptive names, or push the complexity into a code generator rather than the type system.
