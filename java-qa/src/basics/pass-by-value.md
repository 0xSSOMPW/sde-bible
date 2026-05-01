# Q: Is Java pass-by-value or pass-by-reference?

**Answer:**

**Java is strictly pass-by-value.** Always. Even for objects.

The confusion: for objects, the **value being passed is the reference** (the address). The reference is copied — the object is not.

### Primitives — Copy of Value
```java
void increment(int x) { x++; }

int a = 5;
increment(a);
System.out.println(a);  // 5 — caller unaffected
```

### Objects — Copy of Reference
```java
class Box { int val; }

void mutate(Box b) { b.val = 99; }       // mutate via the copied reference
void reassign(Box b) { b = new Box(); }  // reassign the local copy

Box box = new Box();
box.val = 1;

mutate(box);
System.out.println(box.val);   // 99 — same object, mutated

reassign(box);
System.out.println(box.val);   // 99 — local b reassigned, caller's reference unchanged
```

### Mental Model
- Variable stores a value (primitive) or a reference (object handle).
- Method call copies that value/reference into a new local variable.
- Mutating object **state** through the copied reference is visible (same object).
- Reassigning the parameter to a new object is **not** visible (caller still holds original reference).

### Why "pass-by-reference" Would Look Different
True pass-by-reference (C++ `&`, C# `ref`): reassigning the parameter would change the caller's variable.

```java
void swap(Integer a, Integer b) {
    Integer tmp = a; a = b; b = tmp;
}
Integer x = 1, y = 2;
swap(x, y);
// x=1, y=2 — Java can't swap. Pass-by-reference languages can.
```

### String Trap
```java
void change(String s) { s = "world"; }

String s = "hello";
change(s);
System.out.println(s);  // hello — String is immutable + reference reassigned locally
```

### Interview-Killer Phrasing
> "Java is pass-by-value. For object types, the value of the reference is passed by value — a copy of the reference, not the object."
