# Q: JWT vs Session Cookies — picking auth state.

**Answer:**

Two ways to track "this request belongs to user X":

- **Session cookie**: opaque ID; server stores the session in DB/Redis.
- **JWT**: signed self-contained token; server reads claims directly.

JWT is hyped; session cookies are often the better choice for browser apps. Use the right tool.

### Session Cookies

```
1. Client logs in.
2. Server creates session: { user_id: 42 } stored in Redis/DB with TTL.
3. Server sets HTTP-only cookie: Set-Cookie: sid=abc123
4. Browser sends cookie on every request.
5. Server looks up session by sid → has user identity.
```

Properties:
- **Stateful server**: session store required.
- **Trivial revocation**: `DELETE FROM sessions WHERE id = ?`.
- **Small payload**: opaque ID, ~30 bytes.
- **Browser-native**: cookies + same-origin → no XSS exposure (HTTP-only).

### JWT

```
1. Client logs in.
2. Server signs JWT: { sub: 42, exp: ... }
3. Client stores it (in cookie, header, or memory).
4. Client sends JWT in Authorization header on each request.
5. Server verifies signature; reads claims; no DB lookup.
```

Properties:
- **Stateless server**: no session store.
- **Self-contained**: claims travel with the token.
- **Bigger payload**: 200–1000 bytes.
- **Hard to revoke**: token valid until expiry unless you add a blocklist.

### When To Pick Which

| Scenario | Choice |
|---------|--------|
| Browser-based monolith / SPA | **Session cookies** |
| Mobile API client | JWT or opaque token |
| Cross-domain / federated SSO | **JWT (OIDC)** |
| Server-to-server (service-to-service) | **JWT or mTLS** |
| Microservices with API gateway | JWT validated at gateway |
| Need fine-grained revocation | **Session cookies** or short-lived JWT |
| 99% of web apps | Session cookies |

### Why JWTs Are Often The Wrong Default

1. **No real revocation.** Token issued; valid until exp. You can add a blocklist, but now you have stateful sessions — defeats the purpose.

2. **Token bloat.** Embed roles, claims → 1 KB token sent on every request. Network cost.

3. **XSS exfiltration risk.** Stored in JS-accessible memory → XSS = token theft.

4. **Refresh-token complexity.** Short-lived access + long-lived refresh = two tokens to manage.

5. **Key rotation pain.** JWT signature key change requires careful rollout.

Session cookies are battle-tested for browser apps. JWTs solve a different problem (stateless federated identity).

### The Hybrid: BFF Pattern

For SPAs:

```
SPA ─── session cookie ─── Backend-for-Frontend
                                    │
                                    │ holds OAuth tokens
                                    ▼
                            Backend services
```

SPA only knows about cookies. BFF holds the JWT to call backends. Best of both:
- No JWT exposed to browser (XSS-safe).
- Stateless backend services (BFF passes JWT).

### Revocation

**Session cookie**: just delete the session.

**JWT**: options:
- **Short TTL** (5–15 min): revocation = wait it out.
- **Blocklist**: add jti to revoked-list (DB/Redis); check on each request — defeats statelessness.
- **Token versioning**: token has user version; bump on logout; check at request.

Choose by how fast you must revoke.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| JWT in localStorage | XSS bait |
| JWT with `alg: none` | Forgery; libraries used to accept this |
| JWT secret in client | Defeats signing |
| Session cookie with no `SameSite=Lax`/`Strict` | CSRF |
| Forever-valid JWT | Stolen token = forever access |
| Verifying JWT signature with `HS256` shared secret in distributed system | Hard to rotate; use RS256 (asymmetric) |

### Security Headers for Cookies

```
Set-Cookie: sid=abc;
   HttpOnly;            ← inaccessible to JS
   Secure;              ← only over HTTPS
   SameSite=Lax;        ← CSRF defense
   Max-Age=86400;
   Path=/;
   Domain=example.com
```

### JWT Best Practices

- Use **RS256** or **EdDSA** asymmetric signing (not HS256 for multi-service).
- **Short TTL** (≤ 15 min) + refresh token.
- Verify `iss`, `aud`, `exp`, `nbf` on every call.
- Cache JWKS; rotate keys without downtime.
- Never accept `alg: none`.

### Sliding Sessions

Session-cookie variant: extend session expiry on each request (within reason).

User stays logged in as long as they're active. Common UX.

JWT equivalent: refresh on activity. More complex.

> [!NOTE]
> JWTs are great for **stateless federated identity** (OIDC). They're poor for **browser session management**, even though tutorials often misuse them. Pick by the actual problem.

### Interview Follow-ups

- *"Why not store JWT in localStorage?"* — XSS reads it. Always cookie (HttpOnly) or memory.
- *"How does CSRF apply to JWT?"* — If JWT in cookie: same CSRF concerns as session — add `SameSite` and CSRF tokens. If in Authorization header: JS must add it, so no CSRF risk (XSS instead).
- *"What's session fixation?"* — Attacker sets victim's session ID before login; if you don't rotate session ID on auth, attacker can log in as victim. Always regenerate.
