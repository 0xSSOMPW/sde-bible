# Q: Can you explain what a closure is in JavaScript? And as a follow-up, what would this code log and why? How would you fix it to log 0, 1, 2?
```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 1000);
}
```

**Answer:**

A **Closure** is the combination of a function bundled together (enclosed) with references to its surrounding state (the **lexical environment**). 

In simpler terms, a closure gives a function access to its outer scope, **even after the outer function has returned.** In JavaScript, closures are created every time a function is created, at function creation time.

### How it Works
When a function executes, it uses a scope chain to look up variables. If a function returns another function inside of it, that inner function maintains a "backpack" (a hidden `[[Environment]]` reference) containing all the variables it needs from its parent's scope. 

```javascript
function makeGreeting(greeting) {
    // `greeting` is in the outer lexical scope
    return function(name) {
        // This inner function forms a closure 
        console.log(`${greeting}, ${name}!`);
    }
}

const sayHello = makeGreeting("Hello");
const sayHowdy = makeGreeting("Howdy");

// `makeGreeting` has already finished executing, but `sayHello` 
// still remembers the `greeting` variable ("Hello")!
sayHello("Abhay"); // Output: Hello, Abhay!
sayHowdy("Abhay"); // Output: Howdy, Abhay!
```

### Why are Closures Useful?

1. **Data Privacy / Encapsulation:**
   JavaScript did not historically have access modifiers like `private`. Closures allow you to create private variables that cannot be accessed from the outside.
   ```javascript
   function createCounter() {
       let count = 0; // Private variable
       return {
           increment: () => ++count,
           getCount: () => count
       };
   }
   
   const counter = createCounter();
   counter.increment();
   console.log(counter.getCount()); // 1
   console.log(counter.count); // undefined (cannot access directly!)
   ```

2. **Function Factories:**
   Creating partially applied functions (like `makeGreeting` above).

3. **Memoization / Caching:**
   Keeping a private cache object to store expensive calculation results.
   
4. **Callbacks & Event Handlers:**
   When attaching an event listener, you often use variables from the outer scope inside the callback. The event listener forms a closure to remember those variables when the event actually fires.

### Classic Interview "Gotcha"
A common interview question involves closures inside a `for` loop using `var` vs `let`:

```javascript
// Using var (Function Scoped)
for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 1000); 
}
// Output: 3, 3, 3 
// Because `var` is function-scoped, there is only one `i` variable shared 
// by all closures. By the time the timeout runs, the loop has finished and `i` is 3.

// Using let (Block Scoped)
for (let j = 0; j < 3; j++) {
    setTimeout(() => console.log(j), 1000);
}
// Output: 0, 1, 2
// Because `let` is block-scoped, a new `j` is created for every single iteration. 
// Each closure gets its own independent copy.
```
