# Q: DDoS protection and rate limiting strategies.

**Answer:**

DDoS (Distributed Denial of Service) attacks overwhelm services with traffic. Modern mitigation is layered: **edge**, **WAF**, **rate limit**, **bot detection**, **load shedding**. Each catches different attacks.

### Attack Categories

**Volumetric (L3/L4)**:
- Goal: saturate the network pipe.
- Examples: UDP flood, ICMP flood, amplification (DNS, NTP).
- Defense: ISP-level / CDN absorption (Cloudflare, AWS Shield).

**Protocol (L4)**:
- Goal: exhaust connection / state tables.
- Examples: SYN flood, ACK flood.
- Defense: SYN cookies, connection-rate limits.

**Application (L7)**:
- Goal: exhaust app capacity with valid-looking requests.
- Examples: HTTP flood, slowloris, Layer-7 logical attacks.
- Defense: rate limit, CAPTCHA, behavioral analysis.

### Defense In Layers

```
Internet
   │
   ▼
Anycast L4 absorb       ←── volumetric (CDN/CloudFront, Cloudflare)
   │
   ▼
WAF                     ←── known bad patterns, OWASP Top 10
   │
   ▼
Bot detection           ←── JS challenge, fingerprinting
   │
   ▼
Per-IP / per-key rate   ←── token bucket / sliding window
   │
   ▼
Per-tenant / per-user  ←── quotas
   │
   ▼
Service                ←── load shedding when over capacity
```

Each layer drops a portion. The deeper, the smarter. The wider the funnel at top, the better.

### Edge Absorption

For volumetric attacks: only the network can stop it. Cloud providers:
- AWS Shield (Standard free, Advanced paid).
- Cloudflare DDoS Protection (Free tier substantial).
- GCP Cloud Armor.
- Akamai Prolexic.

They have terabits of capacity at PoPs across the world. Your origin couldn't handle 1% of what they absorb.

### Web Application Firewall (WAF)

Rule-based filtering:
- SQL injection patterns.
- XSS attempts.
- Path traversal.
- Known-bad User-Agents.
- Geographic blocks (if your service isn't global).
- Custom rules per app.

Examples: Cloudflare WAF, AWS WAF, Imperva. Managed rulesets (OWASP Core Rule Set) catch common attacks.

### Rate Limiting

For application-level brute force / scraping:
- Per IP.
- Per API key.
- Per endpoint.
- Per tenant.

See [Rate Limiting Algorithms](../patterns/rate-limiting.md).

Catches: brute-force login, credential stuffing, scraping, runaway clients.

### Bot Detection

Distinguish real users from bots:

- **JS challenge**: serve JS that computes a token; humans pass, simple bots can't.
- **TLS / TCP fingerprinting**: real browsers have specific TLS signatures (JA3 hash).
- **Behavioral**: mouse movement, request pacing.
- **CAPTCHA**: last resort; high friction.
- **Device fingerprinting**: canvas, fonts, screen size.

Services: Cloudflare Bot Management, Akamai Bot Manager, reCAPTCHA, hCaptcha.

### Application Layer Hardening

**Connection limits**:
- Max concurrent per IP.
- Max in-flight per IP.
- TCP keepalive tuning.

**Slowloris defense**:
- Read header timeout (5–10 s).
- Max request size.
- Max request duration.

**Per-endpoint capacity**:
- Heavy endpoints get tighter limits.
- Expensive operations require auth + lower per-user rate.

### Authentication Defense

**Credential stuffing**: attackers use leaked credentials elsewhere.

- Rate-limit login per IP + per username.
- Require CAPTCHA after N failures.
- Detect impossible travel (login from two distant locations within minutes).
- MFA for accounts with elevated privileges.

**Account enumeration**: don't disclose "user exists" via different error messages.

### Load Shedding (When All Else Fails)

When you're being legitimately overwhelmed:

- Shed low-priority traffic first.
- Return clear `503` with `Retry-After`.
- Stay alive over serve-perfectly.

See [Graceful Degradation](../reliability/graceful-degradation.md).

### Costs of Defense

- **DDoS service**: $20–200K/year at enterprise scale.
- **WAF**: ~$0.06 per million requests + per-rule cost.
- **Bot mitigation**: subscription.
- **Engineering**: incident-response runbook + rehearsals.

For small services: Cloudflare free tier covers most. For enterprise: dedicated DDoS package + WAF.

### Detection & Response

Monitor:
- Connection rate.
- Request rate per source.
- 4xx error rate (failed auth = brute force).
- Latency spike (under-provisioned).

Automated responses:
- Auto-block IP if rate > threshold.
- Trigger CAPTCHA challenge.
- Shift to "challenge all" mode at edge.

Runbook for: who decides to block a country, when to engage AWS Shield Response Team, etc.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Origin IP exposed | Bypass CDN; attacker hits origin directly |
| Trusting `Remote-Addr` behind proxy | Use `X-Forwarded-For` with care (verify proxy) |
| Rate-limit only at one layer | Defense in depth — layer them |
| Blocking IPs forever | Botnets cycle IPs; targets shared NATs |
| No CAPTCHA challenge tier | Pure block kills false positives; challenge gives human-vs-bot disambiguation |

### Hide Origin

If origin IP is in DNS, attackers bypass your CDN.

- Use a CDN-managed IP.
- Move DNS A records to CDN; origin only accessible via signed CDN routes.
- Firewall: drop traffic not from CDN's IP ranges.

### Common Attack Patterns

- **Layer 7 GET flood**: high QPS to `GET /` from many IPs. Mitigate: edge + per-IP rate + WAF.
- **Reflection attack**: spoofed source IP makes amplifier (DNS, NTP) flood you. ISP-level only.
- **Application bug DDoS**: one specific endpoint exhausts DB. Code fix + rate limit on that endpoint.
- **Coordinated bot scraping**: many IPs, low individual rate. Bot detection + fingerprinting.

> [!NOTE]
> The first line of defense against DDoS is being behind a major CDN/edge provider. The actual rules are the second line. If you don't have edge absorption, you can't run a public service safely.

### Interview Follow-ups

- *"What if an attacker uses 100k legitimate residential IPs?"* — Per-IP rate limit useless; rely on behavioral + bot fingerprinting + CAPTCHA tiering.
- *"How do you protect a write endpoint from being abused?"* — Authentication required + per-user quota + cost-per-request analysis + abuse-pattern detection.
- *"What's a SYN cookie?"* — Server doesn't allocate state on SYN; encodes connection info in initial sequence number. Defeats SYN flood without tracking half-open connections.
