# Q: OAuth 2.0 and OpenID Connect — what they are and how they differ.

**Answer:**

**OAuth 2.0** is an *authorization* framework — it lets a user grant a client app limited access to their resources on another server. **OIDC** layers *authentication* on top — it tells the client app who the user is.

OAuth is "what can you do?" OIDC is "who are you?"

### Roles

- **Resource Owner**: the user.
- **Client**: app wanting access (your web app, mobile app).
- **Authorization Server**: issues tokens (Auth0, Okta, Google, Cognito).
- **Resource Server**: API that consumes tokens.

### OAuth 2.0 Flows

#### Authorization Code (with PKCE)

The recommended flow for web/mobile apps:

```
1. User clicks "Sign in with Google"
2. Client redirects to authorization server with PKCE challenge:
   https://auth/authorize?client_id=...&code_challenge=...

3. User authenticates with Google, grants permission.
4. Auth server redirects back with code:
   https://app/callback?code=abc123

5. Client exchanges code + verifier for token (server-side):
   POST /token { code, code_verifier, client_id }
   → { access_token, refresh_token, id_token (OIDC) }

6. Client uses access_token on API calls.
```

PKCE (Proof Key for Code Exchange) prevents token interception in mobile apps where the redirect URI isn't fully protected.

#### Client Credentials

Service-to-service. No user involved.

```
POST /token { client_id, client_secret, grant_type=client_credentials }
→ { access_token }
```

For backend daemons, server APIs.

#### Device Code

CLI tools, smart TVs.

```
1. Device shows user a code + URL.
2. User opens URL on phone, logs in, enters code.
3. Device polls token endpoint until authorized.
```

#### Deprecated / Don't Use

- **Implicit grant**: tokens in URL fragment; insecure; replaced by Code + PKCE.
- **Resource owner password credentials**: client collects user password; defeats SSO; don't use.

### Tokens

**Access Token**: bearer credential for API calls. Short-lived (5–60 min). Opaque or JWT.

**Refresh Token**: long-lived (days, weeks). Used to get new access tokens. Should be revocable.

**ID Token** (OIDC only): signed JWT containing user identity claims. For the *client* to identify the user; not for API authorization.

### JWT Anatomy

```
header.payload.signature

header:  { "alg": "RS256", "kid": "abc" }
payload: { "sub": "user123", "exp": 1700000000, "iss": "...", ... }
signature: RSA-SHA256(header + "." + payload, private_key)
```

Verifier checks signature against issuer's public key (JWKS endpoint).

Claims:
- `iss`: issuer URL.
- `sub`: subject (user ID).
- `aud`: intended audience.
- `exp`: expiry.
- `iat`: issued at.
- `nbf`: not-before.
- `scope`: granted scopes.

### Scopes

Restrict what the token can do:

```
scope=read:orders write:orders admin:users
```

Resource server checks scopes before serving.

OIDC standard scopes:
- `openid`: required for ID token.
- `profile`: name, picture.
- `email`: email + verified flag.
- `offline_access`: refresh tokens.

### OIDC vs SAML

- **OIDC**: modern, JSON/JWT, REST-friendly. Mobile and SPA support.
- **SAML**: XML-based, enterprise SSO, older.

For new systems: OIDC. SAML still common in enterprises.

### Token Storage in Browser

- **Access token in memory**: safest. Lost on refresh; refetch via cookie session.
- **HTTP-only cookie**: server sets cookie; protected from XSS.
- **localStorage**: vulnerable to XSS. Don't do this.
- **sessionStorage**: same vulnerability, scoped to tab.

Modern: BFF (Backend-for-Frontend) pattern: cookie session to BFF; BFF holds tokens; SPA never sees them.

### Token Revocation

Access tokens: short-lived; "revocation" effectively waits for expiry.

Refresh tokens: must be revocable.
- Auth server tracks issued refresh tokens.
- Client calls `/revoke` on logout.
- Lookup on each refresh; reject if revoked.

JWT-only systems without a revocation store can't truly revoke; mitigate with very short token TTLs.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Using OAuth for login (instead of OIDC) | OAuth was never designed for authentication |
| Storing tokens in localStorage | XSS exfiltration |
| Trusting `iss` without checking against expected | Token-injection attack |
| Long-lived access tokens | Stolen token usable for months |
| Skipping `aud` claim verification | Token meant for service A used at B |
| Implicit flow in 2025 | Deprecated; use Code + PKCE |

### Production Architecture

```
client (SPA / mobile)
   │
   ▼ user logs in via redirect
Auth server (Auth0, Okta, custom)
   │ returns tokens
   ▼
client uses access_token →  API gateway → backend services
                              │
                              ▼
                         token validator (cached JWKS)
```

API gateway validates JWT once; injects authenticated user context to backend.

### Federation

OIDC supports SSO across providers:

```
client → your-auth-server → "log in with Google/Microsoft/Apple"
                              ↓
                           Google IdP → tokens
                              ↓
                           your-auth-server reissues your token
```

User logs in to one IdP; trusted by all federated apps.

> [!NOTE]
> Use a provider. Auth0, Cognito, Keycloak, Okta. Implementing your own OAuth server is a multi-quarter security investment. Worth it only at very specific scales.

### Interview Follow-ups

- *"What's the difference between authentication and authorization?"* — Authn = "who are you" (OIDC). Authz = "what can you do" (OAuth scopes + custom policies).
- *"How would you implement role-based access?"* — Roles in JWT claims; backend reads `roles` claim + per-endpoint check. Or external policy service (OPA).
- *"What is OAuth 2.1?"* — A consolidated update of OAuth 2.0 + best practices: requires PKCE everywhere, deprecates implicit, no password grant. Production default.
