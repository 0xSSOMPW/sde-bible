# Q: What are Spring Boot starters? How do they work under the hood?

**Answer:**

A **starter** is a curated dependency descriptor — a single Maven/Gradle artifact that pulls in a coherent set of libraries for a use case (web, JPA, security, etc.).

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

That single line brings:
- `spring-webmvc`, `spring-web`
- `jackson-databind` (JSON)
- `tomcat-embed-core` (embedded server)
- `spring-boot-starter-json`, `-tomcat`, `-validation`
- compatible versions tested together via the **BOM** (`spring-boot-dependencies`).

### Why Starters Exist
Pre-Boot Spring meant manual version juggling: which `spring-webmvc` works with which `jackson` with which `validator-api`? Starters solve this — pick a Boot version → all transitive versions known-good.

### How Auto-Configuration Hooks In
Each starter brings classes annotated with `@AutoConfiguration` (Boot 2.7+) or `@Configuration` + listed in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Boot scans these on startup.

```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
public class DataSourceAutoConfiguration { ... }
```

`@ConditionalOnX` annotations gate configuration:
- `@ConditionalOnClass` — class on classpath?
- `@ConditionalOnMissingBean` — user hasn't defined their own?
- `@ConditionalOnProperty` — config flag set?

### Common Starters
| Starter | Brings |
|---|---|
| `spring-boot-starter-web` | MVC, Tomcat, Jackson |
| `spring-boot-starter-webflux` | WebFlux + Netty |
| `spring-boot-starter-data-jpa` | Hibernate, Spring Data JPA |
| `spring-boot-starter-data-redis` | Lettuce + Spring Data Redis |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-actuator` | Health/metrics endpoints |
| `spring-boot-starter-test` | JUnit 5, AssertJ, Mockito, Spring Test |
| `spring-boot-starter-validation` | Jakarta Bean Validation |

### Custom Starter (Library Authors)
1. Module: `acme-spring-boot-starter` (just dependency aggregator).
2. Module: `acme-spring-boot-autoconfigure` (the actual `@AutoConfiguration` classes).
3. Register classes in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
4. Provide `@ConfigurationProperties` for tunables.

```java
@AutoConfiguration
@ConditionalOnClass(AcmeClient.class)
@EnableConfigurationProperties(AcmeProperties.class)
public class AcmeAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    AcmeClient acmeClient(AcmeProperties p) {
        return AcmeClient.builder().url(p.url()).build();
    }
}
```

### Override / Disable
- Add your own `@Bean` of the same type → Boot's auto-config backs off (`@ConditionalOnMissingBean`).
- Disable explicitly:
  ```java
  @SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
  ```
- Property: `spring.autoconfigure.exclude=...`.

### Key Takeaway
Starter = "pull dependencies" + "trigger auto-config". You stop writing infrastructure beans; convention does it. Override anywhere via `@Bean` or properties.
