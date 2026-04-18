# Q: What is the difference between `any`, `unknown`, and `never` in TypeScript?

**Answer:**

These three special types form the extreme boundaries of TypeScript's type system. They dictate how strictly the compiler treats unknown or impossible data.

### 1. `any` (The Escape Hatch)
`any` completely disables the TypeScript compiler for that specific variable. You can assign anything to it, and you can perform *any* operation on it without the compiler complaining.

*   **When to use it:** Almost never. It's an escape hatch. It's useful during migrations from legacy JS to TS, or when interacting with poorly typed third-party libraries.

```typescript
let myVar: any = 5;
myVar = "Hello";  // Valid
myVar.doSomething(); // Valid (Compiler allows it, but it crushes at runtime!)
```

### 2. `unknown` (The Safe `any`)
`unknown` is the type-safe counterpart to `any`. Just like `any`, you can assign absolutely any value to `unknown`. **However**, you cannot access properties, call methods, or assign an `unknown` value to a strictly-typed variable *until you prove what it is* using a type guard.

*   **When to use it:** When you are receiving data from an API, parsing JSON, or accepting external user input where you truly don't know the shape yet.

```typescript
let safeVar: unknown = 5;
safeVar = { name: "Abhay" }; // Valid assignment

// ❌ COMPILE ERROR: Object is of type 'unknown'.
// safeVar.name = "John"; 

// ✅ We must prove what it is first (Type Narrowing)
if (typeof safeVar === "object" && safeVar !== null && "name" in safeVar) {
    console.log(safeVar.name); // Now the compiler is happy
}
```

### 3. `never` (The Impossible Type)
`never` represents a state that should **never** physically occur in your code. It's the bottom-most type in TypeScript.

*   **When does it happen naturally?** 
    1. A function that always throws an error.
    2. A function with an infinite loop.

```typescript
function throwCrashError(): never {
    throw new Error("System Crash");
}

function infiniteLoop(): never {
    while (true) {}
}
```

*   **When to use it purposefully? (Exhaustive Checking)**
    The most advanced and powerful use case for `never` is ensuring that `switch` statements have handled every single possible case.

```typescript
type Status = "pending" | "approved" | "rejected";

function handleStatus(status: Status) {
    switch (status) {
        case "pending": return "Wait";
        case "approved": return "Good";
        case "rejected": return "Bad";
        default:
            // If we ever add a new status to the type union but forget to 
            // add a case here, the compiler will try to assign it to this `never` 
            // variable and throw a compile error alerting us to the bug!
            const exhaustiveCheck: never = status;
            return exhaustiveCheck;
    }
}
```

### Summary
*   **`any`**: "I don't care about types. Turn off the compiler."
*   **`unknown`**: "I don't know what this is yet, but force me to check before I touch it."
*   **`never`**: "This code path represents an impossible state."
