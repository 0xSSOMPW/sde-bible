# Q: What are Type Guards in TypeScript?

**Answer:**

A **Type Guard** is a technique in TypeScript that allows you to narrow down the type of a variable within a conditional block. By performing a runtime check, you give TypeScript the guarantee it needs to let you safely access properties that belong only to a specific type.

TypeScript supports several built-in type guards, and also allows you to define your own.

### 1. `typeof`
Used to check basic, standard Javascript primitive types (`string`, `number`, `boolean`, `symbol`).

```typescript
function printId(id: number | string) {
    if (typeof id === "string") {
        // In this block, TypeScript knows `id` is a string
        console.log(id.toUpperCase());
    } else {
        // Here, TypeScript knows it's a number
        console.log(id.toFixed(2));
    }
}
```

### 2. `instanceof`
Used to check if an object was constructed from a specific class.

```typescript
class Car { drive() {} }
class Plane { fly() {} }

function moveVehicle(vehicle: Car | Plane) {
    if (vehicle instanceof Car) {
        vehicle.drive(); 
    } else {
        vehicle.fly(); 
    }
}
```

### 3. The `in` Operator
Often used to narrow down structural types (like interfaces or generic objects) by checking if a specific property exists on the object.

```typescript
interface Bird { fly(): void; }
interface Fish { swim(): void; }

function moveAnimal(animal: Bird | Fish) {
    if ("fly" in animal) {
        animal.fly(); // TypeScript narrowed `animal` down to `Bird`
    } else {
        animal.swim(); // Narrowed to `Fish`
    }
}
```

### 4. User-Defined Type Guards (Type Predicates)
Sometimes standard checks aren't descriptive enough. You can define a custom validation function that returns a **type predicate** (`parameterName is Type`).

```typescript
interface Admin { role: string; privileges: string[]; }
interface User { role: string; lastLogin: Date; }

// The `person is Admin` tells the TS compiler the type if this returns true
function isAdmin(person: Admin | User): person is Admin {
    // We are doing a manual check
    return (person as Admin).privileges !== undefined;
}

function processDashboard(person: Admin | User) {
    if (isAdmin(person)) {
        // TypeScript knows `person` is an Admin here!
        console.log("Admin Privileges:", person.privileges); 
    } else {
        console.log("User Last Login:", person.lastLogin);
    }
}
```
