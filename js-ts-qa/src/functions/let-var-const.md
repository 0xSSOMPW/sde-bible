# Q: What are the differences between `var`, `let`, and `const`?

**Answer:**

The primary differences between `var`, `let`, and `const` revolve around **scope**, **hoisting behavior**, and **reassignment**. This is a fundamental concept in modern JavaScript.

### 1. Scope
*   **`var` is Function Scoped:** If declared inside a function, it is scoped to that function. If declared block (like an `if` statement or `for` loop), it "leaks" out to the surrounding function scope.
*   **`let` and `const` are Block Scoped:** They are constrained strictly to the block they are defined in (any code wrapped in `{}`).

```javascript
function scopeTest() {
    if (true) {
        var functionScoped = "I leak out!";
        let blockScoped = "I stay inside.";
    }
    console.log(functionScoped); // ✅ "I leak out!"
    // console.log(blockScoped); // ❌ ReferenceError
}
```

### 2. Hoisting
All three are hoisted to the top of their respective scopes, but they initialize differently:
*   **`var`**: Initialized with `undefined`. You can access a `var` variable before you write the declaration, but its value will be `undefined`.
*   **`let` and `const`**: They are hoisted, but they are NOT initialized. Accessing them before their declaration line results in a `ReferenceError`. The space before their declaration is known as the **Temporal Dead Zone (TDZ)**.

```javascript
console.log(a); // Output: undefined
var a = 10;

// console.log(b); // ❌ ReferenceError: Cannot access 'b' before initialization
let b = 20;
```

### 3. Reassignment & Redeclaration
*   **`var`**: Can be randomly redeclared in the same scope without throwing an error (which is highly prone to bugs), and can be reassigned.
*   **`let`**: Cannot be redeclared in the same scope, but its value *can* be reassigned.
*   **`const`**: Cannot be redeclared *and* its binding cannot be reassigned. It must be initialized at the time of declaration.

> [!CAUTION]
> While `const` prevents the reassignment of the variable itself, it **does not** make the assigned object or array immutable. You can still mutate the internal properties of a `const` object.

```javascript
var x = 1;
var x = 2; // ✅ Perfectly fine

let y = 1;
// let y = 2; // ❌ SyntaxError: Identifier 'y' has already been declared
y = 2; // ✅ Reassignment is fine

const person = { name: "Abhay" };
// person = { name: "John" }; // ❌ TypeError: Assignment to constant variable.
person.name = "John"; // ✅ Allowed! (Mutation, not reassignment)
```

### Summary Rule of Thumb
1.  **Always use `const` by default.** It clarifies your intent that the variable should not be reassigned.
2.  If you know you will need to re-assign the variable (like a counter in a loop), use **`let`**.
3.  **Stop using `var` entirely** in modern codebase environments unless maintaining legacy code.
