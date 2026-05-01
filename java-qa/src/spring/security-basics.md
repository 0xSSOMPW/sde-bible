# Q: How does Spring Security work? Filter chain, authentication, authorization, JWT.

**Answer:**

Spring Security = a chain of **servlet filters** that intercept every HTTP request. Each filter does one thing (auth, CSRF, logout, etc.).

### The Filter Chain
```
Request → SecurityContextPersistenceFilter
        → LogoutFilter
        → UsernamePasswordAuthenticationFilter (form login)
        → BearerTokenAuthenticationFilter (oauth2 resource server)
        → BasicAuthenticationFilter
        → ExceptionTranslationFilter
        → AuthorizationFilter
        → DispatcherServlet → Controller
```

Each filter can short-circuit (return 401/403) or pass through.

### Modern Configuration (Spring Security 6 / Boot 3)
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())          // disable for stateless API
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.GET, "/api/orders/**").hasAuthority("ORDERS_READ")
                .anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(withDefaults()))
            .exceptionHandling(e -> e
                .authenticationEntryPoint((req, res, ex) -> res.sendError(401))
                .accessDeniedHandler((req, res, ex) -> res.sendError(403)))
            .build();
    }

    @Bean
    PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }
}
```

### Authentication Pieces
| Type | Use |
|---|---|
| `Authentication` | The principal + credentials + authorities (roles) |
| `AuthenticationManager` | Validates credentials, returns authenticated `Authentication` |
| `AuthenticationProvider` | Specific strategy (DAO, LDAP, JWT, ...) |
| `UserDetailsService` | Loads user by username (DAO-based auth) |
| `SecurityContextHolder` | ThreadLocal for the current `Authentication` |

### Username/Password Auth
```java
@Service
public class DbUserDetailsService implements UserDetailsService {
    private final UserRepo repo;

    @Override
    public UserDetails loadUserByUsername(String username) {
        var u = repo.findByEmail(username).orElseThrow(() -> new UsernameNotFoundException(username));
        return User.withUsername(u.email())
            .password(u.passwordHash())
            .authorities(u.roles().stream().map(r -> "ROLE_" + r).toArray(String[]::new))
            .build();
    }
}
```

### Stateless JWT (Resource Server)
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.acme.com/realms/app
          # auto-discovers jwks-uri from /.well-known/openid-configuration
```

Boot auto-wires JWT decoder + filter. Just protect routes.

Custom claim → authority:
```java
@Bean
JwtAuthenticationConverter jwtAuthConverter() {
    JwtGrantedAuthoritiesConverter g = new JwtGrantedAuthoritiesConverter();
    g.setAuthoritiesClaimName("permissions");
    g.setAuthorityPrefix("");
    JwtAuthenticationConverter c = new JwtAuthenticationConverter();
    c.setJwtGrantedAuthoritiesConverter(g);
    return c;
}
```

### Method-Level Security
```java
@Configuration
@EnableMethodSecurity   // unlocks @PreAuthorize, @PostAuthorize, @Secured
public class MethodSecurityConfig { }

@PreAuthorize("hasAuthority('ORDERS_WRITE')")
public Order create(CreateOrderRequest r) { ... }

@PreAuthorize("#userId == authentication.name")  // SpEL — current user matches arg
public User get(String userId) { ... }

@PostAuthorize("returnObject.ownerId == authentication.name")
public Document load(Long id) { ... }
```

### Get Current User
```java
SecurityContextHolder.getContext().getAuthentication().getName();

// Or inject into controller
@GetMapping("/me")
User me(@AuthenticationPrincipal Jwt jwt) {
    return service.findByEmail(jwt.getSubject());
}
```

### Common Patterns

**1. CORS for SPA**
```java
.cors(c -> c.configurationSource(req -> {
    var cfg = new CorsConfiguration();
    cfg.setAllowedOrigins(List.of("https://app.acme.com"));
    cfg.setAllowedMethods(List.of("GET","POST","PUT","DELETE"));
    cfg.setAllowCredentials(true);
    return cfg;
}))
```

**2. Multiple filter chains** (e.g., public API + admin)
```java
@Bean @Order(1)
SecurityFilterChain admin(HttpSecurity http) throws Exception {
    return http.securityMatcher("/admin/**")...build();
}
@Bean @Order(2)
SecurityFilterChain api(HttpSecurity http) throws Exception {
    return http.securityMatcher("/api/**")...build();
}
```

**3. Password encoding**
Always BCrypt or Argon2. Never plaintext. Use `DelegatingPasswordEncoder` to support migrations.

### CSRF
- **Stateless API + token auth (JWT)** → disable CSRF.
- **Session-based browser app** → keep CSRF on. Spring Security uses cookie + header double-submit.

### Common Pitfalls
- **`hasRole("ADMIN")` vs `hasAuthority("ROLE_ADMIN")`** — `hasRole` auto-prefixes `ROLE_`. Authority strings either include `ROLE_` or not — pick one convention.
- **Forgetting `@EnableMethodSecurity`** — `@PreAuthorize` silently does nothing.
- **`permitAll()` in URL config but `@PreAuthorize` denies** — both layers run; deny wins.
- **`SecurityContextHolder` + thread pools** — child threads don't inherit context unless you use `DelegatingSecurityContextExecutor`.
- **CORS configured on Spring MVC but not Security** — preflight blocked by Security filter before MVC sees it.

### Test
```java
@WebMvcTest(OrderController.class)
class OrderControllerSecurityTest {
    @Autowired MockMvc mvc;

    @Test
    @WithMockUser(roles = "ADMIN")
    void adminCanDelete() throws Exception {
        mvc.perform(delete("/api/orders/1")).andExpect(status().isNoContent());
    }

    @Test
    void anonGetsUnauthorized() throws Exception {
        mvc.perform(get("/api/orders/1")).andExpect(status().isUnauthorized());
    }
}
```
