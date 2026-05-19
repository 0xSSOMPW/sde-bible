# Q: Explain JavaScript's prototype chain. How does prototypal inheritance differ from classical inheritance?

**Answer:**

Every JavaScript object has an internal slot `[[Prototype]]` (exposed via `Object.getPrototypeOf(obj)` or the legacy `__proto__`). When you access a property, the engine walks **up** the chain until it finds the property or reaches `null`. There are no classes at runtime — `class` syntax is sugar over functions and prototypes.

### The Chain

```
   instance ----> Constructor.prototype ----> Object.prototype ----> null
   { name }       { greet, constructor }     { toString, hasOwn... }
```

Property lookup is dynamic: changing `Constructor.prototype.greet` is visible from every existing instance instantly.

### Three Ways to Build the Chain

```javascript
// 1. Constructor function (pre-ES6)
function Animal(name) { this.name = name; }
Animal.prototype.speak = function () { return `${this.name} makes a sound`; };
const a = new Animal('Rex');

// 2. Object.create — direct prototype linking
const proto = { speak() { return `${this.name} makes a sound`; } };
const b = Object.create(proto);
b.name = 'Rex';

// 3. class — sugar over option 1
class Animal2 {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
}
```

All three produce equivalent prototype chains.

### Inheritance Hierarchy

```javascript
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
}
class Dog extends Animal {
  speak() { return `${super.speak()} — bark!`; }
}
const d = new Dog('Rex');
//
// d --> Dog.prototype --> Animal.prototype --> Object.prototype --> null
```

`super` resolves through the prototype chain, not through any class-internal table.

### Prototypal vs Classical

| Aspect           | Classical (Java/C++)             | Prototypal (JS)                        |
|------------------|----------------------------------|----------------------------------------|
| Blueprint        | Class defined at compile time    | Objects link to other objects at runtime |
| Inheritance unit | Class extends class              | Object delegates to object             |
| Mutation         | Class hierarchy is fixed         | Add/replace methods anytime            |
| Multiple parents | Often forbidden or via interfaces | Single chain, but mixins are easy     |

In a prototypal system, every object **is** the data. There's no separate class entity; you can clone, link, or rewire chains at runtime.

### `Object.create` and Pure Prototypal Style

```javascript
const vehicle = {
  start() { return `${this.kind} starting`; },
};
const car = Object.create(vehicle);
car.kind = 'Car';
car.start(); // "Car starting"
```

No constructors, no `new`, no classes — pure delegation.

> [!NOTE]
> Avoid `__proto__`. Use `Object.getPrototypeOf` / `Object.setPrototypeOf`. Better still, set the prototype at creation time via `Object.create` — mutating an object's prototype after creation is a major performance deoptimization in V8.

### Common Interview Questions

**1. Difference between `Object.create(null)` and `{}`?**
`Object.create(null)` has no prototype — no `toString`, `hasOwnProperty`, etc. Useful for safe maps so `obj['toString']` returns `undefined` rather than the inherited method.

**2. `instanceof` vs duck typing?**
`a instanceof B` walks `a`'s prototype chain looking for `B.prototype`. It is brittle across realms (iframes, workers). Prefer feature checks where possible.

**3. Why is shared state on the prototype dangerous?**
Storing a mutable array on `Animal.prototype.tags` means every instance shares the same array. Put per-instance state inside the constructor with `this.tags = []`.
