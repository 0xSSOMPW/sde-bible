# Q: What is the difference between Structural Typing and Nominal Typing?

**Answer:**

This is one of the most fundamental design concepts in language architecture. The key difference dictates how the compiler determines if one type is "compatible" with another. **TypeScript is a Structurally Typed language**, whereas languages like Java, C#, and C++ are **Nominally Typed**.

### 1. Structural Typing (TypeScript)
Often referred to as **"Duck Typing"** (If it walks like a duck and quacks like a duck, it's a duck). 

In a structurally typed language, two types are completely compatible **if their internal structure (their shape) matches**. The compiler doesn't care what the types are literally named or if they explicitly inherit from one another.

```typescript
interface Ball { diameter: number; }
interface Earth { diameter: number; }

let myBall: Ball = { diameter: 10 };
let myEarth: Earth = { diameter: 12742 };

// 🤯 PERFECTLY VALID IN TYPESCRIPT!
myBall = myEarth; 
myEarth = myBall; 
```
*Why?* Because TypeScript only checks the structure. Both objects require a `diameter` property of type `number`. Since they both have it, they are structurally interchangeable.

### 2. Nominal Typing (Java, C#)
"Nominal" comes from the word for "name". In a nominally typed language, two types are **only compatible if they share the exact same name or explicit inheritance path**.

Even if two classes have the exact same shape, the compiler will refuse to mix them.

```java
// Java Example (Nominal Typing)
class Ball { public int diameter; }
class Earth { public int diameter; }

Ball myBall = new Ball();
Earth myEarth = new Earth();

// ❌ COMPILE ERROR IN JAVA! 
// "Incompatible types: Earth cannot be converted to Ball"
myBall = myEarth; 
```
*Why?* Because Java strictly looks at the *name* of the types. An `Earth` is not literally a `Ball`, nor does `class Earth implements Ball`, so the Java compiler violently rejects it despite their identical shapes.

### Which is better?
Neither is strictly better, they serve different paradigms:
*   **Structural Typing (TS)** makes mocking, testing, and merging JSON data incredibly fast and flexible. You don't need massive inheritance trees.
*   **Nominal Typing (Java)** provides tighter safety guarantees. You can't accidentally pass a `UserId` string into a function expecting a `Password` string if they are defined as distinct nominal classes, which prevents logical mix-ups.
