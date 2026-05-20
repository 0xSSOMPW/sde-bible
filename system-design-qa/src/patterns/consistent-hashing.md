# Q: Consistent hashing — why, how, virtual nodes.

**Answer:**

Consistent hashing distributes keys across nodes such that adding/removing one node moves only `1/N` of the keys, not "almost all of them" like naïve `hash(key) % N`. It's the foundation of distributed caches (Memcached, Redis Cluster), wide-column stores (Cassandra, DynamoDB), and CDNs.

### The Naïve Approach Fails

```
N = 4 nodes
shard = hash(key) % 4

Add a 5th node:
shard = hash(key) % 5

Now ~80% of keys remap. Cache is cold; rebalancing is catastrophic.
```

### Consistent Hashing — The Ring

Map both nodes and keys onto the same hash space (often 2^64 or 2^32). Each key is owned by the **next node clockwise** on the ring.

```
                    0
              ●─────────●
             ╱     A     ╲
            ╱             ╲
           ●               ●
          ╱                 ╲
         ╱                   ╲
        ●       (keys)        ●
       D                     B
        ●                   ●
         ╲                 ╱
          ●               ●
           ╲             ╱
            ╲     C     ╱
              ●─────────●
                  2^64/2

Node positions = hash(node_id)
Each key → next node clockwise
```

### Adding a Node

```
Add node X between A and B.

Before: keys in range (A, B] owned by B.
After:  keys in range (A, X] owned by X; (X, B] still owned by B.

Only the keys between A and X move from B to X.
```

Roughly `1/N` of total keys move. Other nodes untouched.

### Removing a Node

```
Remove B.
Keys that B owned (between A and B] → now owned by next node clockwise = C.
Only B's keys move; A, C, D untouched.
```

### The Virtual Nodes Problem

Pure consistent hashing with N=3 nodes places 3 points on the ring. Distribution is uneven — a node "lucky" to sit in a sparse region owns much more than its share.

```
Hash distribution with 3 nodes (random):
node A: 50% of ring
node B: 30%
node C: 20%
```

Solution: **virtual nodes** (vnodes). Each physical node owns many positions on the ring:

```
3 physical nodes × 100 vnodes each = 300 points on ring

Distribution evens out — average per node ≈ 1/3
```

Cassandra defaults to 256 vnodes/node. Redis Cluster uses 16,384 fixed "hash slots" assigned to nodes (same idea, fixed-size space).

### Replication

To replicate K times, each key is owned by the **K successive nodes** on the ring clockwise from its hash position.

```
key = X
primary  = next node clockwise = A
replica1 = next node after A     = B
replica2 = next node after B     = C
```

This is how Cassandra and DynamoDB pick which N replicas hold which key.

### Implementation Sketch (Python)

```python
import bisect
import hashlib

class ConsistentHash:
    def __init__(self, vnodes=100):
        self.vnodes = vnodes
        self.ring = []           # sorted list of (hash, node)
        self.nodes = set()

    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node):
        self.nodes.add(node)
        for i in range(self.vnodes):
            h = self._hash(f"{node}:{i}")
            bisect.insort(self.ring, (h, node))

    def remove_node(self, node):
        self.nodes.discard(node)
        self.ring = [(h, n) for h, n in self.ring if n != node]

    def get_node(self, key):
        if not self.ring:
            return None
        h = self._hash(key)
        idx = bisect.bisect(self.ring, (h, ""))
        if idx == len(self.ring):
            idx = 0
        return self.ring[idx][1]
```

Time complexity:
- Add/remove node: O(V log V).
- Lookup: O(log(N×V)).

### Bounded Loads

Vanilla consistent hashing can still have "lucky" hot nodes. **Consistent hashing with bounded loads** (Google, 2016) caps each node's load:

```
on add(key):
    primary = ring.next_clockwise(hash(key))
    if primary.load < threshold:
        return primary
    else:
        return next_clockwise_until_under_threshold()
```

Used by Vimeo's load balancer, Google's Maglev, several CDNs.

### Rendezvous Hashing (Highest Random Weight, HRW)

Alternative without a ring:

```
for each node n:
    score = hash(key, n)
chosen = node with max score
```

Properties:
- No precomputed ring.
- O(N) per lookup; fine for N ≤ 1000.
- Better statistical distribution than basic CH.
- Used by some CDNs and HAProxy `hash` mode.

### Real-World Usage

| System | Mechanism |
|--------|-----------|
| Cassandra | Murmur3 partitioner + 256 vnodes per node |
| DynamoDB | Internal consistent hash, masked from user |
| Redis Cluster | CRC16 → 16384 hash slots → assigned to nodes |
| Memcached (client-side) | Consistent hashing in client library |
| Discord | Consistent hashing for guild sharding |
| Cloudflare | Maglev for L4 LB |

### Failover

When a node goes down, its keys are taken over by the next node clockwise. With replication (factor K), the replicas already have the data.

After permanent loss + replacement, anti-entropy / streaming repairs the new node's share.

### Common Gotchas

| Gotcha | Reality |
|--------|---------|
| "Just use hash(key) % N" | Re-shards almost everything on resize |
| Too few vnodes | Distribution stays uneven; default to ≥100 per node |
| Different vnode counts per node | Intentional weighting (bigger nodes get more keys) |
| Using a poor hash (e.g., `id.hashCode()` in Java) | Skew; use MD5, Murmur3, xxHash |
| Hashing on a mutable field | Rebalances when value changes |

### When NOT to Use Consistent Hashing

- Very small N (3–5 nodes): a simple lookup table is clearer.
- Dynamic shard sizes (massively skewed): use range partitioning with splits.
- Strict capacity caps: use consistent-hashing-with-bounded-loads or a directory layer.

> [!NOTE]
> Consistent hashing is the right answer when you need a stable, low-friction mapping of keys → nodes that survives membership changes. It's overkill if your node set is fixed.

### Interview Follow-ups

- *"How would you handle a 'hot shard' in a CH cluster?"* — Identify hot key via metrics; either pre-route (special-case in client) or split the key (salting) across multiple shards.
- *"What's the trade-off of bigger vs smaller vnode count?"* — More vnodes = smoother distribution but bigger ring = slower lookup. 100–256 per node is a good sweet spot.
- *"How does Redis Cluster decide which node has which slot?"* — Operator command (`CLUSTER ADDSLOTS`), or auto via tools. Slots are explicit, not hash-position-derived.
