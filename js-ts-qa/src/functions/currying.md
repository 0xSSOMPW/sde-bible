# Q: What is currying? Implement a curry function that supports partial application.

**Answer:**

**Currying** transforms a function that takes `n` arguments into a sequence of `n` unary functions:

```
f(a, b, c)   --currying-->   f(a)(b)(c)
```

**Partial application** is related but slightly different: it pre-fills *some* arguments and returns a function expecting the rest. A practical `curry` usually supports both — you can pass arguments one at a time, in groups, or all at once.

### Why It Matters

- Build specialized functions from generic ones (`const add5 = add(5)`).
- Compose pipelines without anonymous lambdas everywhere.
- Match the signature expected by point-free functional utilities.

### Manual Currying

```javascript
// Original
function add(a, b, c) { return a + b + c; }

// Curried by hand
const addCurried = a => b => c => a + b + c;
addCurried(1)(2)(3); // 6
```

### Generic `curry` Implementation

```javascript
function curry(fn, arity = fn.length) {
  return function curried(...args) {
    if (args.length >= arity) {
      return fn.apply(this, args);
    }
    return function (...more) {
      return curried.apply(this, args.concat(more));
    };
  };
}

const sum = (a, b, c, d) => a + b + c + d;
const cSum = curry(sum);

cSum(1, 2, 3, 4);   // 10
cSum(1)(2)(3)(4);   // 10
cSum(1, 2)(3, 4);   // 10
cSum(1)(2, 3)(4);   // 10
```

The trick: each invocation either has *enough* arguments to call the original or returns a closure that remembers what we have so far and keeps collecting.

### Partial Application with Placeholders

Real-world libraries (Ramda, Lodash) support a placeholder so you can skip an earlier argument:

```javascript
const _ = Symbol('placeholder');

function curryP(fn) {
  return function curried(...args) {
    const filled = args.slice(0, fn.length);
    if (
      filled.length >= fn.length &&
      filled.every(a => a !== _)
    ) {
      return fn.apply(this, filled);
    }
    return (...more) => {
      const merged = [...filled];
      let i = 0;
      for (const m of more) {
        const idx = merged.indexOf(_, i);
        if (idx === -1) merged.push(m);
        else { merged[idx] = m; i = idx + 1; }
      }
      return curried.apply(this, merged);
    };
  };
}

const greet = (g, name, punct) => `${g}, ${name}${punct}`;
const cGreet = curryP(greet);
const yell = cGreet(_, _, '!');
yell('Hi', 'Ada'); // "Hi, Ada!"
```

### Practical Uses

```javascript
const fetchFrom = curry((baseUrl, path) => fetch(`${baseUrl}${path}`));
const fromApi   = fetchFrom('https://api.example.com');
fromApi('/users');
fromApi('/orders');

const map = curry((fn, arr) => arr.map(fn));
const doubleAll = map(x => x * 2);
doubleAll([1, 2, 3]); // [2, 4, 6]
```

> [!NOTE]
> Currying composes especially well with pipelines: `pipe(map(double), filter(isEven), reduce(sum))(list)`.

### Trade-offs

- Currying adds function-call overhead. In hot loops that matters.
- Stack traces become deeper and harder to read.
- Variadic functions (`fn.length === 0`) need an explicit arity argument.
