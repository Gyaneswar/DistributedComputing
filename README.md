# Distributed Coordination Library

A Java library implementing core distributed coordination primitives — leader election, automatic failover, and dynamic cluster membership — designed for fault-tolerant distributed systems.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Cluster Manager                    │
│         (Membership tracking, node registry)         │
└──────────────┬──────────────────┬───────────────────┘
               │                  │
       ┌───────▼───────┐  ┌──────▼────────┐
       │ Leader Election│  │ Failure       │
       │ Module         │  │ Detector      │
       │                │  │               │
       │ - Candidate    │  │ - Heartbeat   │
       │   registration │  │   monitoring  │
       │ - Vote logic   │  │ - Timeout     │
       │ - Herd-effect  │  │   detection   │
       │   avoidance    │  │ - Re-election │
       └───────┬───────┘  │   trigger     │
               │          └──────┬────────┘
               │                 │
       ┌───────▼─────────────────▼───────┐
       │         Node (Process)           │
       │  - Joins/leaves cluster          │
       │  - Participates in elections     │
       │  - Sends/receives heartbeats     │
       └─────────────────────────────────┘
```

## Features

### Leader Election
Nodes in the cluster participate in an election process to choose a single leader. The algorithm ensures:
- **Exactly one leader** is elected at any point in time
- **Deterministic outcome** — given the same set of candidates, the same leader is chosen
- **No split-brain** — network partitions are handled gracefully

### Automatic Failover
When the current leader becomes unreachable:
1. Failure detector identifies the leader as unresponsive (missed heartbeats)
2. A new election is triggered automatically
3. A new leader is elected without manual intervention
4. Followers re-register with the new leader

### Herd Effect Avoidance
A naive approach to leader re-election notifies all nodes simultaneously, causing a **thundering herd** — every node tries to become leader at once, flooding the network. This library avoids the herd effect by:
- Assigning sequential watch order so only the **next candidate in line** is notified when the leader fails
- Other nodes remain idle until it's their turn, reducing unnecessary network traffic and contention

### Dynamic Cluster Membership
Nodes can join and leave the cluster at runtime:
- **Join**: New nodes register themselves and participate in future elections
- **Leave**: Departing nodes are deregistered; if the departing node was the leader, failover is triggered
- **Crash**: Unresponsive nodes are detected via heartbeat timeout and removed from the membership list

## Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Failure detection | Heartbeat with timeout | Simple, reliable, and widely used in production systems (e.g., ZooKeeper, etcd) |
| Herd effect avoidance | Sequential watch chain | O(1) notifications on leader failure instead of O(n) broadcast |
| Leader selection | Lowest sequence number | Deterministic, avoids ties, simple to reason about |
| Membership tracking | Ephemeral nodes | Automatic cleanup when a node crashes without graceful shutdown |

## Getting Started

### Prerequisites
- Java 11+
- Maven

### Build
```bash
mvn clean install
```

### Run a 3-Node Cluster
```bash
# Terminal 1 — Node 1
java -jar target/distributed-computing.jar --node-id 1 --port 8081

# Terminal 2 — Node 2
java -jar target/distributed-computing.jar --node-id 2 --port 8082

# Terminal 3 — Node 3
java -jar target/distributed-computing.jar --node-id 3 --port 8083
```

### Simulate Leader Failure
Kill the leader process (Ctrl+C). Observe:
1. Failure detector triggers after heartbeat timeout
2. Only the next candidate is notified (no herd effect)
3. New leader is elected within seconds
4. Remaining nodes acknowledge the new leader

## Failure Scenarios

| Scenario | Expected Behavior |
|----------|-------------------|
| Leader crashes | Next candidate elected, other nodes notified |
| Follower crashes | Removed from membership, no election triggered |
| Leader crashes during election | Election restarts with remaining candidates |
| New node joins mid-election | Waits for current election to complete, then joins |
| Network partition (minority side) | Minority side cannot elect leader (no quorum) |

## Project Structure
```
src/
├── main/java/
│   ├── election/          # Leader election logic
│   ├── membership/        # Cluster membership management
│   ├── heartbeat/         # Failure detection via heartbeats
│   ├── node/              # Node lifecycle (join, leave, crash handling)
│   └── util/              # Shared utilities
└── test/java/
    ├── election/          # Election correctness tests
    ├── membership/        # Membership change tests
    └── integration/       # Multi-node integration tests
```

> **Note**: Update the structure above to match your actual package layout.

## Concepts & References

This library draws from well-established distributed systems concepts:
- [The Chubby Lock Service (Google)](https://research.google/pubs/pub27897/) — inspiration for leader election with sequential watches
- [ZooKeeper: Wait-free Coordination](https://www.usenix.org/conference/usenix-atc-10/zookeeper-wait-free-coordination-internet-scale-systems) — herd effect avoidance via sequential ephemeral nodes
- [Raft Consensus Algorithm](https://raft.github.io/) — leader election and failure detection patterns

## License

MIT
