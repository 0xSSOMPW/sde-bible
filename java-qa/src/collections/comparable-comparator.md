# Q: What is the difference between Comparable and Comparator?

**Answer:**

Both are used for sorting objects, but they serve different purposes.

### Comparable — Natural Ordering (Built-In)
The class itself implements `Comparable<T>` and defines its **natural ordering** via `compareTo()`. There is only **one** natural ordering per class.

```java
public class Employee implements Comparable<Employee> {
    private int id;
    private String name;

    @Override
    public int compareTo(Employee other) {
        return Integer.compare(this.id, other.id); // Natural order: by ID
    }
}

List<Employee> employees = new ArrayList<>();
Collections.sort(employees); // Uses compareTo() — sorts by ID
```

### Comparator — Custom Ordering (External)
A separate functional interface that defines a custom ordering **without modifying the class**. You can have **multiple** comparators for different sort criteria.

```java
// Sort by name
Comparator<Employee> byName = Comparator.comparing(Employee::getName);

// Sort by salary descending, then by name ascending
Comparator<Employee> bySalaryDesc = Comparator.comparing(Employee::getSalary).reversed()
                                              .thenComparing(Employee::getName);

employees.sort(byName);
employees.sort(bySalaryDesc);
```

### Key Differences

| Feature | Comparable | Comparator |
|---|---|---|
| **Package** | `java.lang` | `java.util` |
| **Method** | `compareTo(T other)` | `compare(T a, T b)` |
| **Modifies class?** | ✅ Yes (class implements it) | ❌ No (external) |
| **# of orderings** | 1 (natural order) | Unlimited |
| **Usage** | `Collections.sort(list)` | `Collections.sort(list, comparator)` |
| **Functional interface?** | ❌ | ✅ (can use lambdas) |

### Modern Java Comparator Utilities

```java
// Null-safe comparisons
Comparator.nullsFirst(Comparator.comparing(Employee::getName))

// Chaining
Comparator.comparing(Employee::getDepartment)
          .thenComparing(Employee::getSalary)
          .reversed()

// Lambda shorthand
employees.sort((a, b) -> a.getName().compareTo(b.getName()));
```

> [!TIP]
> Rule of thumb: Implement `Comparable` for the **one obvious natural ordering** (e.g., alphabetical for String, numeric for Integer). Use `Comparator` for any **alternative orderings** (e.g., sort employees by salary, by department, etc.).
