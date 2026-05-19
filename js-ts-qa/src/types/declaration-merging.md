# Q: What is declaration merging in TypeScript? When is it useful (and dangerous)?

**Answer:**

**Declaration merging** is TypeScript's ability to combine multiple separate declarations that share the same name into a single definition. The compiler unifies them in well-defined ways depending on what kind of declarations they are.

### What Can Merge

| Combination                          | Result                                                            |
|----------------------------------------|-------------------------------------------------------------------|
| `interface` + `interface`             | Members unioned                                                   |
| `namespace` + `namespace`             | Inner declarations merged                                          |
| `namespace` + `class`                 | Static side augmented with namespace members                       |
| `namespace` + `function`              | Function gains properties                                          |
| `namespace` + `enum`                  | Enum gains extra members or static helpers                         |
| `interface` + `class`                 | Class's instance side gains extra members (declared but not impl) |
| Module augmentation `declare module`  | Extends an external module's types                                |

**Cannot merge:** `type` aliases with anything else, two `class` declarations, two `enum` value-defining declarations.

### Interface Merging

```typescript
interface User { id: number; }
interface User { name: string; }

const u: User = { id: 1, name: 'Ada' }; // both members required
```

This is the basis for declaration files where many libraries augment a shared type (e.g., adding fields to `Express.Request`).

### Module Augmentation

The most common real-world use: extend types in a third-party library.

```typescript
// express-augmentation.d.ts
import 'express';

declare module 'express' {
  interface Request {
    user?: { id: string; roles: string[] };
  }
}

// later, in middleware
app.use((req, _res, next) => {
  req.user = { id: '1', roles: ['admin'] }; // now type-checks
  next();
});
```

> [!NOTE]
> Module augmentation only works on the module's existing **interfaces**, not on its `type` aliases. Library authors who export shapes as `type` cannot be augmented — a useful authoring distinction.

### Global Augmentation

```typescript
// globals.d.ts
declare global {
  interface Window {
    analytics?: { track(event: string): void };
  }
}
export {}; // turns this file into a module so `declare global` is allowed
```

After this, `window.analytics?.track('login')` is typed throughout the codebase.

### Function + Namespace (Static Properties)

```typescript
function makeCounter() { return 0; }
namespace makeCounter {
  export const version = '1.0';
}

makeCounter();          // function
makeCounter.version;    // "1.0"
```

### Class + Interface (Mixin-like)

```typescript
class Greeter { hi() { return 'hi'; } }
interface Greeter { bye(): string }   // declares an instance method
Greeter.prototype.bye = function () { return 'bye'; }; // implement at runtime
```

Useful when you mix runtime augmentation (plugins) with the type system.

### Dangerous Patterns

**1. Silent shadowing.** Declaration merging is *invisible* at call sites — readers don't see which file added a property. Reviewers may miss security-critical changes (e.g., adding a `headers` field to a request).

**2. Augmentation drift.** When a library upgrades and renames `Request` to a `type`, every augmentation across your codebase breaks silently.

**3. Polluting global types.** Stick to module-scoped augmentation when you can; reserve `declare global` for true app-wide invariants like custom `Window` fields.

**4. Order sensitivity.** Most merges are order-independent, but `namespace` + `class` requires the class to be declared first. Mistakes produce confusing "duplicate identifier" errors.

### Practical Checklist

- Keep augmentations in dedicated `*.d.ts` files near the consumers.
- Always `import` the library before `declare module` — TypeScript needs to know the module exists.
- Re-export `{}` from a `declare global` file to ensure it's parsed as a module.
- Document why each augmentation exists; future maintainers won't otherwise know.
