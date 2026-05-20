# Q: Encryption at rest, in transit, end-to-end — what each protects against.

**Answer:**

Three different threats, three different solutions:

- **Encryption in transit (TLS)**: protects from eavesdroppers on the wire.
- **Encryption at rest**: protects from someone with disk / backup access.
- **End-to-end encryption (E2EE)**: protects from the *server itself*.

Get them confused and you'll have false confidence.

### Encryption in Transit (TLS)

Almost ubiquitous. Browser ↔ server, service ↔ service.

```
TLS handshake:
  client      ─── ClientHello (cipher suites) ──►   server
  client ◄─── ServerHello + cert ── server
  client validates cert (chain to a CA root)
  ECDHE key exchange → shared session key
  Encrypted application data flows
```

Modern TLS 1.3:
- AEAD ciphers (AES-GCM, ChaCha20-Poly1305).
- Perfect Forward Secrecy mandatory.
- Faster handshake (1-RTT, 0-RTT for resumed sessions).

What it protects:
- Network sniffers, MITM (if cert verified).
- ISP / proxy observation.

What it doesn't protect:
- Server logs (server sees plaintext).
- Storage (decrypted before storage).

### Certificates & PKI

- CA signs server cert.
- Browser trusts CA root.
- Cert validation: chain + hostname + not expired + not revoked.

Let's Encrypt: free certs via ACME protocol; 90-day rotation, automated.

mTLS (mutual TLS): both client and server present certs. Used service-to-service.

### Encryption at Rest

Encrypt data on disk so theft of disk / backup is useless.

| Level | Mechanism | What it protects |
|-------|-----------|------------------|
| Disk encryption | LUKS, BitLocker, AWS EBS encryption | Physical disk theft |
| Filesystem encryption | ZFS native, eCryptfs | Same |
| Application-level: Transparent | Postgres TDE, MySQL TDE | DB files on disk |
| Field-level | App encrypts specific fields before writing | DB compromise, internal access |

A running process still sees plaintext. Doesn't protect against:
- Live attacker on the machine.
- DB credentials leak.
- SQL injection.

### Key Management

The hard part of crypto isn't algorithms; it's keys.

- **KMS (Key Management Service)**: AWS KMS, Google Cloud KMS, Azure Key Vault, HashiCorp Vault.
- **Envelope encryption**: data encrypted with data-key (random); data-key encrypted with master-key (KMS).
- **Key rotation**: ability to re-encrypt with a new key without downtime.
- **HSM**: hardware module storing the master key. Used for high-assurance keys.

Never check keys into source. Never log keys. Limit who can access KMS.

### End-to-End Encryption (E2EE)

Server can't read the data. Only sender and recipient have keys.

```
Sender A           Server                Recipient B
   │ encrypt(msg, B's pubkey)              │ decrypt(ct, B's privkey)
   ▼                                       ▲
ciphertext ────────► relay ────────► ciphertext
                       │
              server sees: random bytes
```

Used by: Signal, WhatsApp, iMessage, ProtonMail, Apple iCloud Advanced Data Protection.

Properties:
- Server compromise = blobs only.
- Server can't deliver "search inside content."
- Server can't recover lost passwords (data lost too).
- Key management = user's problem.

### Signal Protocol (E2EE)

Forward-secret, multi-device, post-compromise security.

- **X3DH**: initial key agreement using prekeys.
- **Double Ratchet**: per-message key derivation; symmetric + DH ratchets.
- **Each message has a unique key**; lost keys can't decrypt past messages (forward secrecy).
- **Compromised key gets healed** as both sides ratchet (post-compromise security).

WhatsApp, Signal Messenger, parts of Facebook Messenger, Google Messages.

### Database Encryption Modes

| Mode | Description |
|------|------------|
| **TDE** (Transparent Data Encryption) | DB encrypts at rest; queries unchanged |
| **Column-level** | Specific columns encrypted by DB (Postgres pgcrypto, SQL Server Always Encrypted) |
| **App-level** | App encrypts fields before write; DB sees ciphertext |
| **Searchable encryption** | Specialized; supports limited query patterns over ciphertext |

App-level encryption protects against DB compromise but loses indexability. Compromise: encrypt sensitive fields only (PII, SSN), keep indexes on hashed/tokenized forms.

### Tokenization

Replace sensitive value with random token; map kept in a secured vault.

```
real CC: 4111-1111-1111-1111
token:  tok_abc123                 ← stored in your DB
vault:  tok_abc123 → 4111-1111-1111-1111 (secured)
```

Used by PCI-compliant systems to avoid storing card numbers. Vault becomes the only PCI scope.

### Hashing for Passwords

Never store passwords plain or encrypted. Hash with a slow KDF:

- **bcrypt**: legacy, still acceptable.
- **scrypt**: memory-hard.
- **Argon2id**: modern default.

```
password → argon2(password, salt, memory=64MB, iterations=3) → hash
```

Per-password salt mandatory; per-system pepper recommended.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| TLS but no cert validation | MITM trivial; verify always |
| AES-ECB | Identical blocks → patterns. Use GCM. |
| MD5 / SHA-1 for new code | Broken; use SHA-256+ |
| Storing AES key alongside ciphertext | Defeats the purpose |
| Implementing your own crypto | Always use a vetted library |
| Encrypting then HMAC vs HMAC-then-encrypt | Order matters; use AEAD (GCM) to avoid the choice |
| Encrypting at rest "because compliance" while plaintext logs leak | Cover the whole data flow |

### Threat Model First

Before encrypting:
- Who's the adversary?
- What can they reach?
- What's the cost of breach vs encryption overhead?

A startup probably needs TLS + KMS + bcrypt + HTTP-only cookies. Not Signal Protocol.

> [!NOTE]
> Encryption is one piece of security; access control, monitoring, auditing matter equally. A perfectly encrypted database is undone by an over-privileged service account that decrypts and exports nightly.

### Interview Follow-ups

- *"How would you encrypt user PII while allowing search?"* — Encrypt PII field; for search, store HMAC of normalized value as separate column. Lookups work; raw value protected.
- *"What's the difference between symmetric and asymmetric encryption?"* — Symmetric: same key both ways (AES). Asymmetric: keypair (RSA, EC). Symmetric is fast; asymmetric is used for key exchange + signatures.
- *"What is forward secrecy?"* — Compromising long-term key today doesn't decrypt past sessions. TLS 1.3 enforces it.
