# Q: How does Serialization and Deserialization work in Java?

**Answer:**

**Serialization** converts a Java object into a byte stream (for storage or network transfer). **Deserialization** reconstructs the object from that byte stream.

### Basic Serialization
A class must implement `java.io.Serializable` (a marker interface with no methods):

```java
public class Employee implements Serializable {
    private static final long serialVersionUID = 1L; // Version control
    private String name;
    private int salary;
    private transient String password; // NOT serialized
}

// Serialize
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("emp.ser"))) {
    oos.writeObject(new Employee("Alice", 90000, "secret"));
}

// Deserialize
try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("emp.ser"))) {
    Employee emp = (Employee) ois.readObject();
    // emp.password is null (was transient)
}
```

### Key Concepts

**`serialVersionUID`**
A version identifier. If the class changes (add/remove fields) and the UID doesn't match the serialized data, deserialization throws `InvalidClassException`. Always declare it explicitly.

**`transient`**
Fields marked `transient` are excluded from serialization. Used for sensitive data, derived fields, or non-serializable references.

**`static` fields**
Static fields belong to the class, not the instance — they are NOT serialized.

### Custom Serialization

```java
public class Employee implements Serializable {
    private String name;
    private transient String encryptedPassword;

    private void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject();
        oos.writeObject(encrypt(encryptedPassword)); // Custom logic
    }

    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject();
        encryptedPassword = decrypt((String) ois.readObject());
    }
}
```

### Serialization Problems

| Problem | Issue |
|---|---|
| **Security** | Deserialization of untrusted data can execute arbitrary code (deserialization attacks) |
| **Versioning** | Any class change can break existing serialized data |
| **Performance** | Java serialization is slow and produces verbose output |
| **Inheritance** | Superclass must also be Serializable, or have a no-arg constructor |

### Modern Alternatives

| Alternative | Format | Speed | Use Case |
|---|---|---|---|
| **Jackson** | JSON | Fast | REST APIs, config |
| **Gson** | JSON | Fast | Simple JSON mapping |
| **Protocol Buffers** | Binary | Very fast | gRPC, microservices |
| **Avro** | Binary | Fast | Kafka, data pipelines |
| **Java Records** | N/A | N/A | Immutable data carriers (Java 16+) |

> [!CAUTION]
> Java's built-in serialization (`ObjectOutputStream`) is considered a **security risk** by Oracle itself. It has been the source of many critical CVEs. For new projects, use JSON (Jackson) or Protocol Buffers instead. Java serialization is mainly relevant for understanding legacy systems and the `Serializable` contract.
