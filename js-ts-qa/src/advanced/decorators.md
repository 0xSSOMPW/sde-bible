# Q: What are Decorators in TypeScript/JavaScript?

**Answer:**

A **Decorator** is a special kind of declaration that can be attached to a class declaration, method, accessor, property, or parameter. Decorators use the form `@expression`, where `expression` must evaluate to a function that will be called at runtime with information about the decorated declaration.

Essentially, decorators are a way to use **higher-order functions** to wrap or modify the behavior of classes and their members in a declarative way. They are heavily used in frameworks like Angular and NestJS.

### 1. Requirements
To use decorators in TypeScript, you typically need to enable the `experimentalDecorators` compiler option in your `tsconfig.json`.

```json
{
  "compilerOptions": {
    "target": "ES6",
    "experimentalDecorators": true
  }
}
```

### 2. Class Decorators
A class decorator is applied to the constructor of the class and can be used to observe, modify, or replace a class definition.

```typescript
function Logger(constructor: Function) {
    console.log(`Logging creation of class: ${constructor.name}`);
}

@Logger
class Person {
    constructor(public name: string) {
        console.log("Person initialized");
    }
}
// Outputs: 
// "Logging creation of class: Person" (At definition time)
```

### 3. Method Decorators
Method decorators are applied to methods, allowing you to observe, modify, or replace a method definition. They are extremely useful for tasks like logging, error handling, or binding `this` context.

It takes 3 arguments:
1. Target: The prototype of the class.
2. PropertyKey: The name of the method.
3. Descriptor: The `PropertyDescriptor` of the method.

```typescript
function ReadOnly(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    descriptor.writable = false; // Prevents the method from being overridden
}

class MathOperations {
    @ReadOnly
    multiply(a: number, b: number) {
        return a * b;
    }
}
```

### 4. Decorator Factories
If you want to customize how a decorator is applied to a declaration by passing arguments to it, you can write a decorator factory. A decorator factory is simply a function that returns the actual decorator wrapper function.

```typescript
function LogWithPrefix(prefix: string) {
    return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
        const originalMethod = descriptor.value;
        descriptor.value = function (...args: any[]) {
            console.log(`[${prefix}] Calling ${propertyKey}`);
            return originalMethod.apply(this, args);
        };
    };
}

class Service {
    @LogWithPrefix("DEBUG")
    fetchData() {
        // ...
    }
}
```

> [!NOTE]
> Decorators run when the class is *defined*, not when it is instantiated. The decorator functions execute sequentially during the file's initialization step.
