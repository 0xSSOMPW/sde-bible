# Q: Comparison of `Object.freeze()` and `Object.seal()`

**Answer:**

Both `Object.freeze()` and `Object.seal()` are methods used to make an object immutable to a certain degree, but they have different levels of strictness.

### 1. `Object.seal()`
Sealing an object prevents new properties from being added and existing properties from being removed. However, **you can still modify the values of existing properties** (as long as they are writable).

*   **Add properties?** No
*   **Delete properties?** No
*   **Modify existing properties?** Yes
*   **Reconfigure properties?** No (cannot change enumerability/writability)

**Example:**
```javascript
const user = { name: "Abhay", role: "admin" };
Object.seal(user);

user.name = "John"; // ✅ Allowed! (Value is modified)
user.age = 25;      // ❌ Not allowed! (Silently fails in non-strict mode, throws error in strict mode)
delete user.role;   // ❌ Not allowed!
```

### 2. `Object.freeze()`
Freezing an object is the strictest level of immutability. It does exactly what `seal` does, but **it also prevents modifying the values of existing properties**. The object becomes completely read-only.

*   **Add properties?** No
*   **Delete properties?** No
*   **Modify existing properties?** No
*   **Reconfigure properties?** No

**Example:**
```javascript
const user = { name: "Abhay", role: "admin" };
Object.freeze(user);

user.name = "John"; // ❌ Not allowed!
user.age = 25;      // ❌ Not allowed!
delete user.role;   // ❌ Not allowed!
```

### Shallow vs Deep
> [!IMPORTANT]
> Both methods are **shallow**. This means if the object contains a nested object, the nested object's properties can still be modified, added, or deleted! To freeze or seal a deeply nested object, you have to recursively call the method on all child objects.

```javascript
const company = {
    name: "Tech Corp",
    details: { employees: 50 }
};

Object.freeze(company);
// company.name = "New Tech"; // ❌ Blocked
company.details.employees = 100; // ✅ Allowed! Nested objects are unprotected.
```
