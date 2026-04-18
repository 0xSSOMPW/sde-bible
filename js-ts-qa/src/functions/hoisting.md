# Q: What is Hoisting in JavaScript?

**Answer:**

**Hoisting** is JavaScript's default behavior of moving **declarations** to the top of the current scope (script or function) prior to execution. 

It's crucial to understand that **only declarations are hoisted, not initializations (assignments).** The JavaScript engine executes code in a two-pass process: the Creation Phase (where hoisting happens) and the Execution Phase.

### 1. Function Hoisting
**Function Declarations** are completely hoisted—both the function name and the function body. This means you can invoke a function before it appears in your code.

```javascript
// ✅ Works perfectly!
sayHello();

function sayHello() { // Function Declaration
    console.log("Hello there!");
}
```

However, **Function Expressions** (including arrow functions) are treated like variables. If they use `var`, `let`, or `const`, they follow those respective hoisting rules.

```javascript
// ❌ TypeError: sayGoodbye is not a function (it is `undefined` right now)
sayGoodbye();

var sayGoodbye = function() { // Function Expression
    console.log("Goodbye!");
};
```

### 2. Variable Hoisting (`var`)
Variables declared with `var` are also hoisted to the top of their function/global scope. However, they are initialized with the default value of `undefined`.

```javascript
console.log(count); // Output: undefined
var count = 5;      // Declaration is hoisted, but assignment (= 5) stays here
```
*How the JS Engine sees it:*
```javascript
var count; // Hoisted and set to undefined
console.log(count);
count = 5;
```

### 3. The Temporal Dead Zone (`let` and `const`)
Variables declared with `let` and `const` **are** technically hoisted, but with a major catch: they are **NOT initialized**. 

Because they aren't initialized with `undefined`, trying to access them before the exact line they are declared results in a strict `ReferenceError`. The space between the top of the scope and the line where they are defined is known as the **Temporal Dead Zone (TDZ)**.

```javascript
// entering TDZ for `username`
console.log("Doing some work..."); 

// console.log(username); // ❌ ReferenceError: Cannot access 'username' before initialization

let username = "Abhay"; // TDZ ends here
console.log(username); // ✅ Output: Abhay
```

### Summary
*   **Function Declarations**: Fully hoisted (safe to call early).
*   **`var`**: Hoisted, but initialized to `undefined`.
*   **`let` / `const`**: Hoisted, but uninitialized. Placed in the Temporal Dead Zone (causes an error if accessed early).
*   **Function Expressions / Arrow Functions**: Handled based on the variable keyword (`var/let/const`) they are attached to.
