# Q: How does Spring Boot auto-configuration work? How do you debug it?

**Answer:**

Auto-configuration = Boot inspects the classpath + your config + existing beans, then **conditionally** registers default beans.

### The Entry Point
```java
@SpringBootApplication
public class App { public static void main(String[] a) { SpringApplication.run(App.class, a); } }
```

`@SpringBootApplication` = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`.

### `@EnableAutoConfiguration`
Triggers `AutoConfigurationImportSelector`, which loads class names from:
```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```
(Pre-2.7 used `META-INF/spring.factories`.)

Each class is `@AutoConfiguration` annotated — a `@Configuration` evaluated only if its conditions hold.

### Conditions
```java
@AutoConfiguration
@ConditionalOnClass({DataSource.class, EmbeddedDatabaseType.class})
@ConditionalOnMissingBean(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    @Bean @ConditionalOnProperty(name = "spring.datasource.url")
    DataSource dataSource(DataSourceProperties p) { ... }
}
```

Common conditions:
| Annotation | Fires when |
|---|---|
| `@ConditionalOnClass` | class present on classpath |
| `@ConditionalOnMissingClass` | class absent |
| `@ConditionalOnBean` | bean of type already in context |
| `@ConditionalOnMissingBean` | bean of type **not** yet in context |
| `@ConditionalOnProperty` | config property matches |
| `@ConditionalOnWebApplication` | servlet/reactive web app |
| `@ConditionalOnExpression` | SpEL evaluates true |
| `@ConditionalOnResource` | resource exists |

### Order Matters
- `@AutoConfigureBefore` / `@AutoConfigureAfter` / `@AutoConfigureOrder`.
- User `@Configuration` classes process **before** auto-config → user beans win via `@ConditionalOnMissingBean`.

### Debugging — `--debug` Mode
Run with `--debug` or `debug=true`:
```
=========================
AUTO-CONFIGURATION REPORT
=========================

Positive matches:
-----------------
   DataSourceAutoConfiguration matched:
      - @ConditionalOnClass found required class 'javax.sql.DataSource'
      ...

Negative matches:
-----------------
   GsonAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'com.google.gson.Gson'

Exclusions:
-----------
   org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration

Unconditional classes:
----------------------
   ...
```

### Actuator `/actuator/conditions`
Same data as a live JSON endpoint when actuator is enabled.

### Override / Disable

**Disable specific auto-configs**
```java
@SpringBootApplication(exclude = { SecurityAutoConfiguration.class })
// or via property
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration
```

**Override a bean**
```java
@Bean
public DataSource dataSource() { return myCustomDataSource(); }
// Boot's @ConditionalOnMissingBean → its DataSource doesn't register
```

### `@ConfigurationProperties` Binding
Tunables for auto-configs come from `application.yml`:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://db/app
    username: app
    hikari:
      maximum-pool-size: 20
```

```java
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties { ... }
```

### Common Gotchas
- **Bean conflicts**: forgot `@ConditionalOnMissingBean` in a custom starter → user can't override.
- **Component scan misses package**: `@SpringBootApplication` scans from its own package down. Move it up the package tree if needed.
- **Test slices** (`@WebMvcTest`, `@DataJpaTest`) load **subset** of auto-config. Some beans missing in tests but present in prod.
- **Order surprises**: two auto-configs both register a bean of same type → first wins, others skip due to `@ConditionalOnMissingBean`.

### Interview Soundbite
> "Auto-configuration = conditional `@Bean` definitions, gated on classpath + properties + existing beans. Boot looks under `META-INF/spring/...AutoConfiguration.imports`, evaluates each class's conditions, registers what fits. User beans always win because they're processed first and conditions check `@ConditionalOnMissingBean`."
