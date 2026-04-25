# ETL Data Pipeline (Go)

A high-performance, modular ETL (Extract, Transform, Load) data pipeline built in Go. Designed for production-grade extensibility, reliability, and autonomous self-healing.

## Core Features
- **Support diverse data sources and targets**: Connectors for databases, message queues, file systems, and APIs.
- **Multi Pipeline Multi Sink**: Supports multiple independent pipelines and sinks with per-sink isolation within pipeline, allowing for flexible data routing and transformation.
- **Graceful Schema Evolution Handling**: Built-in support for schema evolution with backward and forward compatibility, ensuring seamless data processing during changes.
- **Zero-Crash Architecture**: Robust first-principle lifecycle management (Cancel -> Drain -> Close) and dual-layer panic recovery for maximum uptime.
- **Autonomous Supervision Tree**: Recursive worker supervisor that detects unplanned crashes and automatically reboots pipelines with exponential backoff.
- **Efficient Serialization**: Utilizes **MessagePack** for high-performance, zero-allocation serialization, significantly reducing CPU and memory overhead compared to JSON.
- **Graceful Transitions**: Two-phase "Drain -> Shutdown -> Restart" protocol for zero-downtime configuration reloads and maintenance.
- **Live Configuration Reloading**: Dynamic configuration management with hot-reload capabilities, allowing for seamless updates without downtime.
- **Load Balancing**: Multiple worker can run in parallel to distribute the load. Pipelines is automatically rebalanced across workers when a new worker joins or leaves the cluster with reasonable waiting time (to prevent aggressive rebalancing).
---

## Architecture Overview

### High-Level Components
- **Worker**: The main process that manages multiple pipelines. It supervises the lifecycle of pipelines and handles configuration reloads.
  - **Pipeline**: Represents a data flow from source to target, each pipeline manage its own lifecycle. Each pipeline have config inlcuding one source and multiple sinks for different targets. Worker will notify pipelines to reload config when it changes, and pipelines will gracefully restart with new config.
    - **Producer**: Responsible for extracting data from the source and publishing it to the message stream. Producer is the only component that interacts with the source, and manages ingress state and checkpointing.
    - **Consumers**: Responsible for consuming data from the message stream, applying transformations, and loading. Consumer is isolated per sink target, so that failure in one sink does not affect others. Consumers manage egress state and checkpointing independently.
- **Message Stream**: A persistent, ordered message stream that decouples Producers and Consumers. A message stream main purpose is to prevent data loss and provide easy routing to distribute messages across multiple Consumers.
- **Distributed KV Store**: A distributed key-value store (e.g., etcd, Consul, NATS KV) for storing pipeline state, checkpoints, and configuration. It provides strong consistency guarantees and is used for coordination between components. Alternatively the kv can also used as distributed LOCK with CAS to coordinate the pipeline lifecycle and prevent multiple pipelines from running simultaneously on different workers.
- **Dead Letter Queue (DLQ)**: A separate message stream or database for storing messages that failed processing after retries. This allows for later analysis and reprocessing without affecting the main data flow.

### At-Least-Once Delivery Flow
1.  **Ingress**: Producer extracts data from the source and persists the ingress state.
2.  **Transport**: Batches are published to the persistent message stream (Circuit Breaker protected).
3.  **Consumption**: Each Sink's Consumer pulls batches independently.
4.  **Transformation**: Consumers apply pluggable **Transformers** (Masking, Filtering, Enrichment) (Optional).
5.  **Egress**: Consumer loads data to the target and commits the checkpoint.

---

## Development & Decisions

### Architectural Decisions (RFCs)
We use a formal **RFC (Request for Comments)** process for all major architectural and design decisions. This ensures that the logic behind our choices is documented, debated, and preserved as a living history.

- **Discovery**: See the [RFC Index](docs/rfcs/README.md) for a list of all decisions.
- **Process**: To propose a change, follow the instructions in the [RFC directory](docs/rfcs/).

---

## Resilience & Reliability

- **Fail-Fast Orchestration**: If a Producer fails, the pipeline immediately signals sibling Consumers to stop, preventing "zombie" states and stopping the source for sending new messages until the issue is resolved this preventing data loss on the middle of downtime.
- **Isolation Mode**: Automatically identifies and routes failing "poison-pill" messages to the **DLQ** while allowing the healthy message to continue loaded to the target.
- **Backpressure Management**: Tuneable per-sink flow control to prevent overwhelming downstream targets.

---

## Core Technologies

- **Go 1.26+**: Leveraging the latest language features for performance and safety.
- **MessagePack (`msgp`)**: Ultra-fast serialization format.
- **Testcontainers-go**: Comprehensive E2E testing with real infrastructure in Docker.


## Development & Testing

### Prerequisites
- **Go 1.26+**
- **Docker/Podman** (for E2E tests)

### Running Tests
```bash
go test -v ./...
```

---
