# Q: How do you secure a Kafka cluster — TLS, SASL, ACLs?

**Answer:**

Kafka security has three independent dimensions: **encryption in transit** (TLS), **authentication** (who are you?), and **authorization** (what can you do?). You configure them per *listener*, and most production clusters layer all three.

### Listeners

Brokers expose one or more listeners with distinct security profiles:

```properties
listeners=INTERNAL://:9092,EXTERNAL://:9093,CONTROLLER://:9094
listener.security.protocol.map=\
  INTERNAL:PLAINTEXT,\
  EXTERNAL:SASL_SSL,\
  CONTROLLER:SSL
inter.broker.listener.name=INTERNAL
```

- **INTERNAL**: broker-to-broker, often PLAINTEXT inside a VPC.
- **EXTERNAL**: client traffic — encrypted + authenticated.
- **CONTROLLER**: KRaft controller plane.

### TLS (Encryption + Optional mTLS Auth)

```properties
listeners=SSL://:9093
ssl.keystore.location=/etc/kafka/broker.keystore.jks
ssl.keystore.password=...
ssl.truststore.location=/etc/kafka/truststore.jks
ssl.truststore.password=...
ssl.client.auth=required        # mTLS — clients also present a cert
```

With `ssl.client.auth=required`, the cert's CN/SAN becomes the principal — that *is* the authentication. No separate SASL handshake.

### SASL (Authentication)

| Mechanism | Use when |
|-----------|----------|
| `PLAIN` | Username/password over TLS only (`SASL_SSL`) |
| `SCRAM-SHA-256/512` | Username/password, salted, supports rotation via ZK/KRaft |
| `GSSAPI` (Kerberos) | Enterprise SSO, AD-backed |
| `OAUTHBEARER` | OIDC/JWT — Azure AD, Okta, Confluent Cloud |

SCRAM example (server side):

```properties
listeners=SASL_SSL://:9093
sasl.enabled.mechanisms=SCRAM-SHA-512
listener.name.sasl_ssl.scram-sha-512.sasl.jaas.config=...
```

Client `jaas.conf`:

```
KafkaClient {
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="orders-svc"
  password="...";
};
```

> [!NOTE]
> Never use `SASL_PLAINTEXT` with `PLAIN` outside a closed test network — credentials cross the wire reversibly.

### Authorization (ACLs)

Set `authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer` (KRaft) or `kafka.security.authorizer.AclAuthorizer` (ZK).

```bash
# Allow service "orders-svc" to produce to "orders"
kafka-acls.sh --bootstrap-server b1:9093 --add \
  --allow-principal User:orders-svc \
  --operation Write --operation Describe \
  --topic orders

# Allow consumer group "orders-consumer"
kafka-acls.sh --bootstrap-server b1:9093 --add \
  --allow-principal User:orders-consumer \
  --operation Read --operation Describe \
  --topic orders --group orders-consumer
```

Operations: `Read`, `Write`, `Create`, `Delete`, `Alter`, `Describe`, `ClusterAction`, `AlterConfigs`, `DescribeConfigs`, `IdempotentWrite`.

Resources: `Topic`, `Group`, `Cluster`, `TransactionalId`, `DelegationToken`.

Default policy: **deny when authorizer is enabled** — explicit allow required. Be careful enabling on a running cluster without first auditing required principals.

### Common Production Setup

```
Clients  ──SASL_SSL (SCRAM or OAUTH)──>  Brokers
                                          │
                                  ─SSL (mTLS)─> KRaft controllers
                                          │
                                  ─SSL─> Inter-broker
```

Authorization via ACLs, with one principal per service, automated by Terraform or a topic registry.

### Encryption at Rest

Kafka doesn't encrypt log segments itself. Two options:
- Disk-level encryption (LUKS, EBS encryption) — protects only against disk theft.
- **Field-level / message-level encryption** in the producer (envelope encryption with KMS) — protects in the broker filesystem too. Required for PCI/PII often.

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| ACLs enabled, internal listener forgot a principal → cluster fails to form | Set `super.users=User:admin` and explicit allows for the broker principal |
| Cert rotation requires broker restart | Use `ssl.keystore.location` reloading (KIP-651) or run a rolling restart pipeline |
| OAUTHBEARER without JWKS caching → token endpoint melts | Configure `sasl.oauthbearer.jwks.endpoint.refresh.ms` |
| Client uses PLAINTEXT bootstrap, SASL listener — silent connect failure | Always match `security.protocol` to the listener you point at |

### Interview Follow-ups

- *"How do you rotate a SCRAM password without downtime?"* — Add new credentials, deploy client with both old+new in jaas (or just new), remove old SCRAM entry in metadata. SCRAM creds live in `__cluster_metadata` (KRaft) or ZK.
- *"How do mTLS and ACLs interact?"* — mTLS gives you the principal (`User:CN=...`); ACLs apply against that principal. No SASL needed.
- *"Is there a way to do row-level / field-level authorization?"* — Not natively. Use client-side encryption with KMS keys gated by an external policy engine, or a privacy proxy in front.
