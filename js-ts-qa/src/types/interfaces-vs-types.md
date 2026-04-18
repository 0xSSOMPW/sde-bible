# Q: What is the difference between `interface` and `type` in TypeScript?

**Answer:**

In modern TypeScript, both `interface` and `type` (type aliases) are often used interchangeably to define object shapes, but they have a few crucial differences under the hood. 

Here are the key distinctions to remember for your interviews:

### 1. Primitive and Union Types
**`type`** can represent *any* kind of type. This includes primitives, unions, and tuples.
**`interface`** can *only* represent the shape of an object (including functions and arrays).

```typescript
// ✅ Only possible with `type`
type ID = string | number; // Union
type Coordinates = [number, number]; // Tuple
type Callback = (data: string) => void;

// ❌ Cannot be done with `interface`
// interface ID = string | number; // Error!
```

### 2. Extending / Inheritance 
Both support extension, but their syntax and under-the-hood behavior differ.
- **`interface`** uses the `extends` keyword.
- **`type`** uses intersections (`&`).

```typescript
// Interface extension
interface Animal { name: string; }
interface Bear extends Animal { honey: boolean; }

// Type intersection
type AnimalType = { name: string; }
type BearType = AnimalType & { honey: boolean; }
```
> [!NOTE] 
> Performance tip: TypeScript's compiler caches interface resolution much more efficiently than type intersections. If you have deep, complex hierarchies, `interface extends` will compile faster than massive `type & type` intersections.

### 3. Declaration Merging
**`interface`** supports *declaration merging*. If you declare the same interface twice, TypeScript will automatically merge them into one. This is extremely useful for extending third-party libraries (like adding custom properties to the `Window` object).

**`type`** does not support declaration merging. Declaring a type twice throws an error.

```typescript
// ✅ Interfaces Merge
interface User { name: string; }
interface User { age: number; }
// Result: { name: string; age: number }

// ❌ Types Conflict
type Person = { name: string; }
type Person = { age: number; } // Duplicate identifier 'Person'
```

### 4. Implementation in Classes
A class can `implements` both an `interface` and a `type` alias (as long as the type alias resolves to an object shape). There is no difference here.

```typescript
type Flyable = { fly(): void };
interface Swimmable { swim(): void };

class Duck implements Flyable, Swimmable {
    fly() {}
    swim() {}
}
```

### Summary Rule of Thumb
Use **`interface`** for public-facing API contracts, object shapes, and when you need declaration merging. Use **`type`** when you need a union, intersection, tuple, or are aliasing a primitive type.
