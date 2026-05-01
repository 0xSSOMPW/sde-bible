# Q: How do Spring profiles work? How do you handle environment-specific config?

**Answer:**

Profiles = named groups of beans + properties. Activate per-environment (dev/staging/prod) without code changes.

### Activate Profiles
Multiple ways, increasing priority:
```bash
# property
spring.profiles.active=prod

# env var
export SPRING_PROFILES_ACTIVE=prod

# CLI
java -jar app.jar --spring.profiles.active=prod

# JVM
-Dspring.profiles.active=prod
```

Multiple: `dev,debug,local`.

### Profile-Specific Properties
Boot automatically loads:
```
application.yml             # base — always loaded
application-dev.yml         # only when "dev" active
application-prod.yml        # only when "prod" active
```

Profile properties override base.

### Profile-Specific Beans
```java
@Configuration
@Profile("prod")
public class ProdMailConfig {
    @Bean MailSender mailSender() { return new SesMailSender(); }
}

@Configuration
@Profile("!prod")  // any non-prod
public class DevMailConfig {
    @Bean MailSender mailSender() { return new ConsoleMailSender(); }
}
```

Method-level too:
```java
@Bean @Profile("prod") DataSource prodDs() { ... }
@Bean @Profile({"dev","test"}) DataSource devDs() { ... }
```

### YAML Multi-Document
Single file, multiple profiles (Boot 2.4+):
```yaml
spring:
  application.name: my-app
---
spring:
  config.activate.on-profile: dev
server:
  port: 8080
---
spring:
  config.activate.on-profile: prod
server:
  port: 80
```

### Profile Groups (Boot 2.4+)
```yaml
spring:
  profiles:
    group:
      production: prod, monitoring, audit
```
Activate `production` → all three flip on.

### Default Profile
If none active, `default` profile is. `application-default.yml` loads. Override:
```properties
spring.profiles.default=local
```

### Conditional Beans Beyond Profiles
For finer control:
```java
@ConditionalOnProperty(name = "feature.payments.v2", havingValue = "true")
@Bean PaymentClient v2Client() { ... }
```

### Programmatic Activation
```java
SpringApplication app = new SpringApplication(App.class);
app.setAdditionalProfiles("prod");
app.run(args);
```

### Tests
```java
@SpringBootTest
@ActiveProfiles({"test", "h2"})
class OrderServiceTest { ... }
```

### Common Patterns

**1. External secrets per env**
```yaml
# application.yml
db:
  url: ${DB_URL}
  password: ${DB_PASSWORD}
```
Profile decides which env vars are set in deployment manifest.

**2. Mock vs real integrations in dev**
```java
@Profile("local") @Service class FakePaymentClient implements PaymentClient {...}
@Profile("!local") @Service class StripePaymentClient implements PaymentClient {...}
```

**3. Cloud config + profiles**
Spring Cloud Config server can serve profile-specific files (`app-prod.yml`) from git.

### Pitfalls
- **Forgot to activate** → bean missing → `NoSuchBeanDefinitionException`.
- **Multiple profile files** but typo in profile name → silently uses defaults.
- **Tests inheriting prod profile** → hitting real services. Always set `@ActiveProfiles("test")`.
- **Property precedence**: command-line > env vars > application-{profile}.yml > application.yml. Knowing this matters when debugging "why is this value not what I set".

### Profile-Aware Property Sources Order (highest precedence first)
1. Command-line args
2. `SPRING_APPLICATION_JSON`
3. `application-{profile}.properties/yml` (external)
4. `application.properties/yml` (external)
5. `application-{profile}.properties/yml` (classpath)
6. `application.properties/yml` (classpath)
7. `@PropertySource`
8. Default properties

Profile-specific always wins over base at the same level.
