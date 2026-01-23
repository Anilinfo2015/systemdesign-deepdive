# The Art of Sharding — Part III: The Expert Zone (Hard)

## Table of Contents

- [4. Rebalancing Strategies](#4-rebalancing-strategies)
- [5. Cross-Shard Operations](#5-cross-shard-operations)
- [6. Shard Management](#6-shard-management)
- [7. High Availability per Shard](#7-high-availability-per-shard)
- [8. Data Locality & Co-location](#8-data-locality--co-location)
- [9. Indexing Strategies](#9-indexing-strategies)
- [10. Transaction Management](#10-transaction-management)
- [11. Monitoring & Observability](#11-monitoring--observability)

## 4. Rebalancing Strategies

### Rebalancing Process

```mermaid
sequenceDiagram
    participant Admin
    participant Controller
    participant OldShard
    participant NewShard
    participant Clients

    Admin->>Controller: Add new shard
    Controller->>Controller: Calculate data to move
    Controller->>Clients: Start dual-write mode

    Note over OldShard,NewShard: Phase 1: Copy Data
    Controller->>OldShard: Read data range
    OldShard->>Controller: Return data
    Controller->>NewShard: Write data

    Note over OldShard,NewShard: Phase 2: Sync Recent Changes
    Controller->>OldShard: Get delta changes
    OldShard->>Controller: Return changes
    Controller->>NewShard: Apply changes

    Controller->>Clients: Update routing<br/>(switch to new shard)
    Controller->>OldShard: Delete migrated data
    OldShard->>Controller: Confirm deletion
    Controller->>Admin: Rebalancing complete
```

### Fixed Partitions Strategy

```mermaid
graph TB
    subgraph "Initial: 2 Physical Servers, 8 Partitions"
        S1[Server 1<br/>P0 P1 P2 P3]
        S2[Server 2<br/>P4 P5 P6 P7]
    end

    subgraph "After Adding Server 3"
        S3[Server 1<br/>P0 P1 P2]
        S4[Server 2<br/>P3 P4 P5]
        S5[Server 3<br/>P6 P7]
    end

    S1 --> S3
    S2 --> S4
    S2 --> S5

    style S5 fill:#9f9,stroke:#333,stroke-width:3px
```

### When Rebalancing is Needed

- Adding new shards (scaling out)
- Removing failed/decommissioned shards
- Fixing hot partitions
- Optimizing uneven data distribution

### Techniques

#### Fixed Number of Partitions
- Create many more partitions than nodes upfront
- Assign multiple partitions to each node
- Move entire partitions when rebalancing
- **Example**: Elasticsearch, Kafka (partition reassignment)

#### Virtual Partitioning
- Shard keys map to virtual shards
- Virtual shards map to fewer physical servers
- Change mappings without code modifications
- Reduces impact during rebalancing

#### Automatic Rebalancing
- Database handles migration automatically
- Consistent hashing enables self-organization
- **Examples**: Cassandra, DynamoDB, Vitess

### Rebalancing Challenges

```mermaid
graph TB
    subgraph "Rebalancing Risks"
        R1[Heavy I/O<br/>Operations]
        R2[Downtime<br/>Risk]
        R3[Consistency<br/>Concerns]
        R4[Time<br/>Consuming]
    end

    subgraph "Mitigation Strategies"
        M1[Throttle<br/>Migration Rate]
        M2[Dual-Write<br/>Mode]
        M3[Atomic<br/>Switchover]
        M4[Background<br/>Migration]
    end

    R1 --> M1
    R2 --> M2
    R3 --> M3
    R4 --> M4

    style R1 fill:#f99,stroke:#333
    style R2 fill:#f99,stroke:#333
    style R3 fill:#f99,stroke:#333
    style R4 fill:#f99,stroke:#333
    style M1 fill:#9f9,stroke:#333
    style M2 fill:#9f9,stroke:#333
    style M3 fill:#9f9,stroke:#333
    style M4 fill:#9f9,stroke:#333
```

---

## 5. Cross-Shard Operations

### The Challenge

Queries requiring data from multiple shards are complex and expensive.

### Scatter-Gather Pattern

```mermaid
sequenceDiagram
    participant Client
    participant Router
    participant S1 as Shard 1
    participant S2 as Shard 2
    participant S3 as Shard 3
    participant S4 as Shard 4

    Client->>Router: SELECT * FROM users<br/>WHERE age > 25

    Note over Router,S4: Scatter Phase (Parallel)
    par Query All Shards
        Router->>S1: Query
        Router->>S2: Query
        Router->>S3: Query
        Router->>S4: Query
    end

    par Responses
        S1-->>Router: 100 results
        S2-->>Router: 150 results
        S3-->>Router: 120 results
        S4-->>Router: 130 results
    end

    Note over Router: Gather Phase<br/>Merge & Sort Results
    Router->>Router: Aggregate 500 results
    Router->>Client: Return merged results
```

#### Code Example

```python
import threading
from concurrent.futures import ThreadPoolExecutor
import time

class ScatterGatherQuery:
    def __init__(self, num_shards):
        self.shards = {f'shard_{i}': self._create_sample_data(i)
                      for i in range(num_shards)}

    def _create_sample_data(self, shard_id):
        """Create sample user data for each shard"""
        return [
            {'user_id': f's{shard_id}_u{i}', 'age': 20 + (i % 40),
             'name': f'User_{shard_id}_{i}'}
            for i in range(100)
        ]

    def query_shard(self, shard_name, condition):
        """Query a single shard (simulates network latency)"""
        time.sleep(0.1)  # Simulate network latency
        results = [user for user in self.shards[shard_name]
                  if condition(user)]
        print(f"{shard_name}: found {len(results)} results")
        return results

    def scatter_gather(self, condition):
        """Execute query across all shards in parallel"""
        print(f"Executing scatter-gather across {len(self.shards)} shards...")
        start_time = time.time()

        # Scatter: Query all shards in parallel
        with ThreadPoolExecutor(max_workers=len(self.shards)) as executor:
            futures = {
                shard_name: executor.submit(self.query_shard, shard_name, condition)
                for shard_name in self.shards
            }

            # Gather: Collect results
            all_results = []
            for shard_name, future in futures.items():
                all_results.extend(future.result())

        elapsed_time = time.time() - start_time
        print(f"Gathered {len(all_results)} total results in {elapsed_time:.2f}s")
        return all_results

# Example usage
sg = ScatterGatherQuery(num_shards=4)

# Query: Find all users older than 30
results = sg.scatter_gather(lambda user: user['age'] > 30)
print(f"\nFound {len(results)} users older than 30")

# Show some results
print("\nSample results:")
for user in results[:5]:
    print(f"  {user['user_id']}: {user['name']}, age {user['age']}")
```

**Output:**
```
Executing scatter-gather across 4 shards...
shard_0: found 60 results
shard_1: found 60 results
shard_2: found 60 results
shard_3: found 60 results
Gathered 240 total results in 0.11s

Found 240 users older than 30

Sample results:
  s0_u11: User_0_11, age 31
  s0_u12: User_0_12, age 32
  s0_u13: User_0_13, age 33
  s0_u14: User_0_14, age 34
  s0_u15: User_0_15, age 35
```

### Patterns

#### Scatter-Gather
- Query all relevant shards in parallel
- Aggregate results at application layer
- **Trade-off**: Higher latency, more network traffic

#### Denormalization
- Duplicate data across shards to avoid joins
- Store complete records where needed
- **Trade-off**: Storage overhead, consistency complexity

#### Application-Level Joins
- Fetch data from multiple shards separately
- Join in application memory
- **Trade-off**: Application complexity, memory usage

### Cross-Shard Join Example

```mermaid
graph TB
    subgraph "Problem: Cross-Shard Join"
        P1[Users Shard 1]
        P2[Users Shard 2]
        P3[Orders Shard 1]
        P4[Orders Shard 2]

        P5[Want to JOIN<br/>users and orders<br/>by user_id]
    end

    subgraph "Solution: Denormalization"
        S1[Orders Shard 1<br/>Includes user data:<br/>user_id, user_name, user_email]
        S2[Orders Shard 2<br/>Includes user data:<br/>user_id, user_name, user_email]
    end

    P5 --> S1
    P5 --> S2

    style P5 fill:#f99,stroke:#333
    style S1 fill:#9f9,stroke:#333
    style S2 fill:#9f9,stroke:#333
```

---

## 6. Shard Management

### Routing Architectures

```mermaid
graph TB
    subgraph "Client-Side Routing"
        C1[Smart Client]
        C1 -->|Knows shard locations| S1[(Shard 1)]
        C1 -->|Direct connection| S2[(Shard 2)]
        C1 -->|Direct connection| S3[(Shard 3)]
    end

    subgraph "Proxy-Based Routing"
        C2[Thin Client]
        C2 --> Proxy[Query Router/Proxy]
        Proxy --> S4[(Shard 1)]
        Proxy --> S5[(Shard 2)]
        Proxy --> S6[(Shard 3)]
    end

    style C1 fill:#faa,stroke:#333
    style Proxy fill:#9f9,stroke:#333,stroke-width:3px
```

### Configuration Service

Tracks shard topology and routing information:
- **What it stores**: Shard locations, mappings, health status
- **Technologies**: ZooKeeper, etcd, Consul
- **Purpose**: Service discovery, configuration management

### Routing Layer (Query Router)

```mermaid
sequenceDiagram
    participant C as Client
    participant R as Router
    participant CS as Config Service
    participant S1 as Shard 1
    participant S2 as Shard 2

    Note over C,S2: Initialization
    R->>CS: Get shard topology
    CS->>R: Return shard map

    Note over C,S2: Query Routing
    C->>R: INSERT user_id=12345
    R->>R: hash(12345) % 2 = 1
    R->>S2: Route to Shard 2
    S2->>R: Success
    R->>C: Success

    Note over C,S2: Config Changes
    CS->>R: Shard topology updated
    R->>R: Refresh routing table
```

#### Client-Side Routing
- Application knows shard topology
- No proxy overhead
- **Example**: Cassandra native drivers

#### Proxy-Based Routing (Server-Side)
- Centralized routing logic
- Client unaware of sharding
- **Examples**: ProxySQL, Vitess, MongoDB mongos, MySQL Router

---

## 7. High Availability per Shard

### Shard Replication Architecture

```mermaid
graph TB
    subgraph "Shard 1 Replica Set"
        S1P[Primary]
        S1R1[Replica 1]
        S1R2[Replica 2]

        S1P -->|Async<br/>Replication| S1R1
        S1P -->|Async<br/>Replication| S1R2
    end

    subgraph "Shard 2 Replica Set"
        S2P[Primary]
        S2R1[Replica 1]
        S2R2[Replica 2]

        S2P -->|Async<br/>Replication| S2R1
        S2P -->|Async<br/>Replication| S2R2
    end

    C[Client] -->|Writes| S1P
    C -->|Writes| S2P
    C -->|Reads| S1R1
    C -->|Reads| S2R1

    style S1P fill:#f96,stroke:#333,stroke-width:3px
    style S2P fill:#f96,stroke:#333,stroke-width:3px
    style S1R1 fill:#9cf,stroke:#333
    style S1R2 fill:#9cf,stroke:#333
    style S2R1 fill:#9cf,stroke:#333
    style S2R2 fill:#9cf,stroke:#333
```

### Auto-Failover Process

```mermaid
sequenceDiagram
    participant C as Coordinator
    participant P as Primary
    participant R1 as Replica 1
    participant R2 as Replica 2
    participant M as Monitor

    Note over P: Primary is healthy
    C->>P: Write request
    P->>C: Success

    Note over P: Primary fails!
    P-xC: Connection lost
    M->>P: Health check
    P-xM: No response
    M->>R1: Health check
    R1->>M: Healthy
    M->>R2: Health check
    R2->>M: Healthy

    Note over M,R1: Promote Replica 1
    M->>R1: Promote to Primary
    R1->>M: Promotion complete

    C->>R1: Retry write request
    R1->>C: Success
```

### Replication

- **Primary-Replica (Master-Slave)**: Writes to primary, reads from replicas
- **Multi-Primary**: Multiple write-capable nodes
- **Quorum-based**: Require majority consensus for operations

### Consensus Protocols

- **Raft**: Leader election, log replication
- **Paxos**: Distributed consensus
- **Purpose**: Maintain consistency across replicas

---

## 8. Data Locality & Co-location

### Tenant-Based Sharding

```mermaid
graph TB
    subgraph "Multi-Tenant Sharding"
        T1[Tenant A<br/>Small startup]
        T2[Tenant B<br/>Small startup]
        T3[Tenant C<br/>Enterprise]
        T4[Tenant D<br/>Medium company]
    end

    subgraph "Shards"
        S1[(Shard 1<br/>Tenant A + B<br/>All their data)]
        S2[(Shard 2<br/>Tenant C<br/>Dedicated)]
        S3[(Shard 3<br/>Tenant D<br/>All their data)]
    end

    T1 --> S1
    T2 --> S1
    T3 --> S2
    T4 --> S3

    style S2 fill:#faa,stroke:#333,stroke-width:3px
```

### Entity Group Co-location

```mermaid
graph LR
    subgraph "Co-located: Good ✓"
        G1[(Shard 1<br/>user_123)]
        G2[User Profile<br/>user_123]
        G3[Orders<br/>for user_123]
        G4[Preferences<br/>for user_123]

        G2 --> G1
        G3 --> G1
        G4 --> G1
    end

    subgraph "Scattered: Bad ✗"
        B1[(Shard 1<br/>User Profile)]
        B2[(Shard 2<br/>Orders)]
        B3[(Shard 3<br/>Preferences)]

        B4[Need JOIN<br/>across 3 shards!]
        B4 -.-> B1
        B4 -.-> B2
        B4 -.-> B3
    end

    style G1 fill:#9f9,stroke:#333
    style B4 fill:#f99,stroke:#333
```

### Benefits

- **Performance**: Minimize network hops
- **Transactions**: Enable ACID within shard
- **Cost**: Reduce cross-shard query overhead

---

## 9. Indexing Strategies

### Local vs Global Secondary Indexes

```mermaid
graph TB
    subgraph "Local Secondary Index"
        L1[(Shard 1<br/>Users A-M<br/>+ Local Index)]
        L2[(Shard 2<br/>Users N-Z<br/>+ Local Index)]

        LQ[Query: email=alice@email.com<br/>Must check ALL shards]
        LQ -.->|Scan| L1
        LQ -.->|Scan| L2
    end

    subgraph "Global Secondary Index"
        G1[(Shard 1<br/>Users A-M)]
        G2[(Shard 2<br/>Users N-Z)]
        G3[(Global Email Index<br/>Separate sharding)]

        GQ[Query: email=alice@email.com<br/>Check index only]
        GQ -->|Direct lookup| G3
        G3 -->|Found in Shard 1| G1
    end

    style LQ fill:#faa,stroke:#333
    style GQ fill:#9f9,stroke:#333
```

### Local Secondary Indexes

**How it works**: Each shard maintains its own indexes

**Advantages**:
- Fast writes (update only one shard)
- Simpler consistency model
- Lower coordination overhead

**Disadvantages**:
- Queries may need to hit all shards (scatter-gather)
- Slower reads for non-shard-key queries

**Use case**: Write-heavy workloads

### Global Secondary Indexes

**How it works**: Index spans across all shards, separately partitioned

**Advantages**:
- Fast reads (query only relevant index shards)
- Efficient for non-shard-key queries

**Disadvantages**:
- Slower writes (update multiple index shards)
- Complex consistency management
- Higher coordination overhead

**Use case**: Read-heavy workloads with diverse query patterns

**Example**: DynamoDB Global Secondary Indexes

---

## 10. Transaction Management

### Single-Shard vs Cross-Shard Transactions

```mermaid
graph TB
    subgraph "Single-Shard Transaction (Fast)"
        ST1[BEGIN]
        ST2[UPDATE account_123<br/>balance -= 100]
        ST3[INSERT transaction_log]
        ST4[COMMIT]

        ST1 --> ST2 --> ST3 --> ST4

        STS[(Shard 1<br/>All in one shard<br/>ACID guaranteed)]
        ST2 -.-> STS
        ST3 -.-> STS
    end

    subgraph "Cross-Shard Transaction (Complex)"
        CT1[BEGIN]
        CT2[UPDATE account_123<br/>in Shard 1]
        CT3[UPDATE account_456<br/>in Shard 2]
        CT4[2-Phase Commit<br/>or Saga Pattern]
        CT5[COMMIT]

        CT1 --> CT2 --> CT3 --> CT4 --> CT5

        CTS1[(Shard 1)]
        CTS2[(Shard 2)]
        CT2 -.-> CTS1
        CT3 -.-> CTS2
    end

    style STS fill:#9f9,stroke:#333,stroke-width:3px
    style CTS1 fill:#faa,stroke:#333
    style CTS2 fill:#faa,stroke:#333
```

### Two-Phase Commit (2PC)

```mermaid
sequenceDiagram
    participant C as Coordinator
    participant S1 as Shard 1
    participant S2 as Shard 2
    participant S3 as Shard 3

    Note over C,S3: Phase 1: Prepare
    C->>S1: Prepare transaction
    C->>S2: Prepare transaction
    C->>S3: Prepare transaction

    S1->>C: Ready to commit
    S2->>C: Ready to commit
    S3->>C: Ready to commit

    Note over C: All shards ready

    Note over C,S3: Phase 2: Commit
    C->>S1: Commit
    C->>S2: Commit
    C->>S3: Commit

    S1->>C: Committed
    S2->>C: Committed
    S3->>C: Committed
```

### Saga Pattern

```mermaid
sequenceDiagram
    participant O as Order Service
    participant P as Payment Service
    participant I as Inventory Service
    participant S as Shipping Service

    Note over O,S: Happy Path
    O->>O: Create order (local tx)
    O->>P: Process payment (local tx)
    P->>O: Payment successful
    O->>I: Reserve inventory (local tx)
    I->>O: Inventory reserved
    O->>S: Schedule shipping (local tx)
    S->>O: Shipping scheduled

    Note over O,S: Failure & Compensation
    O->>O: Create order
    O->>P: Process payment
    P->>O: Payment successful
    O->>I: Reserve inventory
    I->>O: Inventory failed (out of stock)

    Note over O,S: Compensating Transactions
    O->>P: Refund payment
    O->>O: Cancel order
```

### Single-Shard Transactions

- Full ACID guarantees within a shard
- Use traditional database transactions
- Fast and simple
- **Design principle**: Shard key should enable single-shard transactions

### Cross-Shard Transactions

#### Two-Phase Commit (2PC)
- Coordinator ensures atomic commits across shards
- **Disadvantages**: Blocking, coordinator is SPOF, high latency

#### Saga Pattern
- Break transaction into local transactions per shard
- Compensating transactions for rollback
- Eventual consistency model

---

## 11. Monitoring & Observability

### Monitoring Dashboard

```mermaid
graph TB
    subgraph "Metrics Collection"
        M1[Data Distribution<br/>Size, row count per shard]
        M2[Performance<br/>Latency, throughput]
        M3[Health<br/>Uptime, replication lag]
        M4[Hotspot Detection<br/>Query patterns]
    end

    subgraph "Alerting"
        A1[Imbalanced Shards<br/>(>20% difference)]
        A2[High Latency<br/>(>100ms p99)]
        A3[Replication Lag<br/>(>5 seconds)]
        A4[Hot Partition<br/>(>2x avg load)]
    end

    M1 --> A1
    M2 --> A2
    M3 --> A3
    M4 --> A4

    style A1 fill:#f99,stroke:#333
    style A2 fill:#f99,stroke:#333
    style A3 fill:#f99,stroke:#333
    style A4 fill:#f99,stroke:#333
```

### Key Metrics

#### Data Distribution
- Size per shard
- Row count per shard
- Growth rate per shard
- Identify imbalanced shards

#### Performance Metrics
- Query latency per shard
- Throughput (reads/writes per second)
- CPU, memory, disk I/O per shard
- Cache hit rates

#### Health Monitoring
- Shard availability/uptime
- Replication lag
- Failed queries
- Connection pool status

### Tools

- Prometheus + Grafana for metrics
- Distributed tracing (Jaeger, Zipkin)
- Database-specific tools (MongoDB Atlas, Vitess VTGate metrics)

---
