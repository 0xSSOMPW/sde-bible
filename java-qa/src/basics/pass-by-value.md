# Q: Is Java pass-by-value or pass-by-reference?

**Answer:**

**Java is strictly pass-by-value.** Always — even for objects.

In Java, everything is pass-by-value. No exceptions. What changes is **what the value represents**:

* For primitives (`int`, `double`, etc.), the value is the actual data.
* For objects, the value is a **reference to the object**.

So when you pass an object to a method, Java **copies the reference** (not the object itself). That means:

* You can mutate the object’s internal state inside the method.
* You cannot change which object the caller’s variable refers to.

### Example

```java
class Test {
    int value;
}

void modify(Test obj) {
    obj.value = 10; // affects original object
}

void reassign(Test obj) {
    obj = new Test(); // does NOT affect original reference
}
```

### Key takeaway

> Java is pass-by-value; for objects, the value passed is a copy of the reference.

---

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
