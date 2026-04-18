# Q: What is Type Coercion in JavaScript?

**Answer:**

**Type Coercion** is the process of converting a value from one data type to another (such as a string to a number, or an object to a boolean). In JavaScript, type coercion can be either **explicit** or **implicit**.

Understanding this concept is crucial because JavaScript is a loosely-typed language, meaning it will aggressively try to coerce types silently in the background to execute operations, which often leads to bizarre bugs.

### 1. Explicit Type Coercion (Type Casting)
This happens when a developer intentionally writes code to convert one type to another using built-in global functions.

```javascript
let val = "123";

// Explicitly converting a String to a Number
const num = Number(val); 
const num2 = parseInt(val, 10); 

// Explicitly converting a Number to a String
const str = String(456); 
const str2 = (456).toString();

// Explicitly converting to a Boolean
const bool = Boolean(1); // true
const bool2 = !!0; // false
```

### 2. Implicit Type Coercion
This happens silently by the JavaScript engine when you apply operators to values of different types. The engine automatically decides what type the value *should* be to make the operation work.

**The String Concatenation Trap (`+`)**
If any operand of the `+` operator is a string, JavaScript coerces the other operands to strings and concatenates them.
```javascript
console.log(1 + "2");     // "12" (Number 1 is coerced to String "1")
console.log("5" + true);  // "5true"
```

**The Numeric Conversion Trap (`-`, `*`, `/`)**
Unlike `+`, math operators like `-`, `*`, and `/` strictly expect numbers. JavaScript will attempt to coerce strings into numbers to perform the math.
```javascript
console.log("5" - "2");   // 3 (Strings are coerced to Numbers)
console.log("5" * "2");   // 10
console.log("10" - "a");  // NaN (Not-a-Number, because "a" cannot be cleanly converted)
```

### 3. Loose Equality (`==`) vs Strict Equality (`===`)
This is the most common interview question related to coercion.

*   `===` (**Strict Equality**): Checks if both the value AND the data type are identical. **No type coercion is performed.**
*   `==` (**Loose Equality**): Checks if the values are equal *after* performing implicit type coercion if the types are different.

```javascript
console.log(1 === "1"); // false (Number vs String)
console.log(1 == "1");  // true (The string "1" is coerced into a Number before comparing)

console.log(0 == false); // true
console.log("" == false); // true
console.log(null == undefined); // true
```

> [!CAUTION]
> Always use strict equality (`===`) in modern JavaScript/TypeScript to avoid catastrophic bugs caused by unpredictable implicit coercion rules!

### 4. Truthy and Falsy Values
When variables are used in a boolean context (like an `if` statement), JavaScript coerces them into booleans. 

Every value in JS is considered "Truthy" (coerces to `true`) **except for exactly 6 "Falsy" values:**
1.  `false`
2.  `0` (and `-0`)
3.  `""` (empty string)
4.  `null`
5.  `undefined`
6.  `NaN`
