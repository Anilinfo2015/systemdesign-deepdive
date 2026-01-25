# Staff-Level Feedback & Practice Plan (Distributed Cache)

This note turns a real interview-style evaluation into a checklist + drills you can reuse.

---

## 1) The Feedback (verbatim, condensed)

**Summary**: Strong Senior. For Staff, show more proactive system-wide bottleneck detection, tighter terminology, and deeper node-level internals.

**Key gaps called out**:
- **Back-of-envelope first**: Capacity/scale math (e.g., 50TB → hundreds of nodes) should drive the architecture up front.
- **Concurrency internals**: Strict LRU + doubly-linked list lock contention is a classic trap.
- **Terminology precision**: Don’t say *leaderless* while describing a *primary/replica* write path.
- **Stampede mechanics**: Refresh-ahead alone isn’t the most robust protection; request coalescing (singleflight) should show up early.

---

## 2) What “Staff Ownership” Looks Like in System Design

Use this as an internal script during the interview:

### A. Start with the forcing functions (numbers)
You should be able to say within the first 5–10 minutes:
- **Data size**: $\text{Total bytes} = \text{items} \times \text{avg item size}$ (+ overhead)
- **Working set**: what fraction must be “hot” in cache?
- **Node count** (rough):

$$
N \approx \frac{\text{Total data} \times (1 + \text{replication factor})}{\text{usable RAM per node} \times \text{target utilization}}
$$

- **QPS per node**:

$$
\text{QPS/node} \approx \frac{\text{Total QPS}}{N \times \text{hit fraction}} \quad (\text{rough; validate later})
$$

**Staff signal**: you do this *before* you pick LB vs smart-client vs proxy.

### B. Name the top bottlenecks proactively
For a distributed cache, you can usually call these early:
- **Coordinator bottleneck** (if routing/membership goes through a single tier)
- **Hot keys / skew** (p99 latency + CPU saturation)
- **Eviction/TTL overhead** (metadata contention at high concurrency)
- **Rebalance blast radius** (network + DB load spikes)
- **Stampede/dogpile** (downstream collapse)

### C. Keep a “control plane vs data plane” split
- Data plane: `GET/SET` path, latency budget, tail behavior.
- Control plane: membership, topology epochs, rebalancing, tooling.

This prevents mixing ideas like “ZooKeeper is on the request path”.

---

## 3) Terminology Cheat Sheet (use exact words)

### Primary/Replica (Redis-style)
- **Write path**: client writes primary; primary replicates to replicas.
- **Consistency**: depends on sync/async replication; reads may be stale.
- **Failover**: elect/promote a new primary.

### Leaderless / Quorum (Dynamo-style)
- **Write path**: client writes to $W$ replicas; read from $R$ replicas.
- **Consistency**: tunable via $R/W/N$; handles partitions differently.
- **Conflict resolution**: versions/vector clocks/LWW, read-repair.

**Rule**: Don’t say “leaderless” unless you mean quorum writes/reads.

---

## 4) Node Internals: The “LRU Lock” Trap and Better Defaults

### The trap
A strict LRU often implies:
- Hash map lookup
- Then mutate a shared doubly-linked list (move-to-front)

At high concurrency this can become a **global write lock** and dominate CPU.

### Better defaults to propose
- **Sharded cache**: partition key space; each shard has single-writer eviction structures.
- **Buffered recency**: record access in a per-core buffer; batch apply.
- **CLOCK / second-chance**: approximate LRU with lower contention.
- **Sampling eviction**: random samples among candidates (common in practice).

**Staff signal**: call out “strict LRU is expensive; we’ll use an approximation” unprompted.

---

## 5) Stampede / Thundering Herd: What to Always Mention

Your “stampede toolkit” list should include:
- **Request coalescing (singleflight)** per key
- **TTL jitter** to avoid synchronized expirations
- **Stale-while-revalidate** (serve stale briefly while refreshing)
- **Probabilistic early refresh** (refresh before expiry based on load)
- **Negative caching** (cache misses for short TTL)

**Staff signal**: coalescing is the “backend protection” anchor.

---

## 6) Practice Drills (repeatable)

### Drill A — 5-minute math opener
Pick a scenario and do:
1) data size, replication factor, node count
2) QPS per node
3) bandwidth sanity check (peak value size * miss rate)

### Drill B — 3 failure modes, 3 mitigations
For each:
- “What breaks first?”
- “What will you see in metrics?”
- “How do we degrade safely?”

Suggested failures:
- hot key, stampede, partial partition, rebalancing storm

### Drill C — Write path consistency
Practice saying clearly:
- “Primary/replica async replication → stale reads possible.”
- “Leaderless quorum → write/read quorum trade-offs and conflict resolution.”

---

## 7) Where This Maps Into This Series

- Node internals + eviction trade-offs: see [02-mvp-architecture.md](02-mvp-architecture.md)
- Massive scale membership/topology control plane: see [03-scaling-challenges.md](03-scaling-challenges.md)
- Production patterns (stampede/runbooks/security): see [06-production-mastery.md](06-production-mastery.md)
