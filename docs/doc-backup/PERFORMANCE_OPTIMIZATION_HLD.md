# Performance Optimization High-Level Design

## 1. Overview

This document defines the high-level design for the performance optimization effort across the runtime manager. The design maintains business logic, execution semantics, and safety policy enforcement while addressing the major sources of CPU overhead, allocator churn, syscall overhead, and synchronization contention.

## 2. Design goals

1. Reduce CPU utilization during steady-state traffic.
2. Reduce memory footprint and allocator churn.
3. Improve request latency and throughput.
4. Reduce system call and file I/O overhead.
5. Increase scalability under concurrency.
6. Improve startup and shutdown behavior.
7. Keep the runtime safe and observable.

## 3. High-level approach

The runtime optimization strategy is structured in layers:

### 3.1 Event and dispatch layer

The runtime must maintain a single socket-driven event loop and move work off the hot path into bounded worker threads instead of creating a detached thread for each request.

Key design decisions:

- maintain a fixed worker pool
- move request processing off the main loop without blocking the socket poll path
- keep bounded queue lengths to avoid overload amplification

### 3.2 Execution layer

The command execution path must be bounded and efficient. Instead of a one-off `fork()`/`pipe()` pattern for every command under heavy concurrency, the design calls for a controlled execution model with concurrency caps and reduced per-command overhead.

Key design decisions:

- preserve the current secure exec model
- use bounded command execution fan-out
- keep command output capture and timeout semantics unchanged

### 3.3 Catalog and metadata layer

The runtime must reduce repeated lookup and serialization overhead. Catalog lookup should be index-based, not repeated full-scan or repeated JSON round-trips on hot paths.

Key design decisions:

- build compact lookup index from the loaded catalog metadata
- avoid repeated JSON encoding/decoding during dispatch
- store minimal runtime metadata required for a quick lookup and lazy-load deeper details only when needed

### 3.4 Plugin and watcher layer

The plugin manager already uses filesystem watching and scan logic; the optimization design keeps the semantics but reduces unnecessary rescans and wakeups.

Key design decisions:

- prefer event-driven notification when available
- avoid repeated full scans on unchanged directories
- deduplicate reload signals and state recomputation

### 3.5 Observability layer

The runtime must remain observable without imposing high performance cost.

Key design decisions:

- log on error or state transitions, not every low-value request event
- use a bounded logger or rate-limited syslog path
- preserve feature-table output and startup diagnostics without repeated noisy logging

## 4. Architectural components

### 4.1 Request queue and worker pool

This component owns the dispatch queue and worker threads used to handle WRP requests.

Responsibilities:

- dequeue request objects from the socket loop
- keep worker count bounded and deterministic
- maintain backpressure under high load
- allow profiling of queue depth and processing latency

### 4.2 Command execution scheduler

This component owns the external command execution path and ensures execution remains controlled.

Responsibilities:

- reduce per-command syscall cost
- preserve timeout and output cap behavior
- maintain concurrency limits
- avoid unbounded process fan-out

### 4.3 Catalog index manager

This component owns the runtime lookup index that maps `plane` + `tool` to the relevant metadata.

Responsibilities:

- build compact lookup metadata once at load time
- support refresh on reload without stopping the service
- remain safe when the optional backend is JSON or LMDB

### 4.4 Plugin watcher manager

This component owns plugin state tracking and event-driven updates.

Responsibilities:

- avoid duplicate scan work
- detect only relevant filesystem changes
- keep reload and unload paths consistent

## 5. Design constraints

- No change to business logic.
- No weakening of security or policy enforcement.
- No change to the external request/response contract.
- No change to command allowlist or denylist semantics.
- No regression in startup, shutdown, or SIGHUP reload handling.

## 6. Reliability and risk posture

The optimization design follows a staged rollout:

1. measure baseline with profiling tools
2. implement one low-risk optimization at a time
3. validate functional behavior after each change
4. measure and compare benchmark deltas

This approach reduces risk while preserving operational safety.

## 7. High-level data flow after optimization

1. Main event loop receives socket-ready events.
2. Request is enqueued to worker pool.
3. Bounded workers process request dispatch and action selection.
4. Command execution path is throttled and guarded.
5. Catalog lookup uses compact index instead of repeated linear scans.
6. Logs are emitted through a safe rate-limited path.
7. Plugin event handling triggers selective reload or rescan only when required.

## 8. Expected benefits

- lower request dispatch overhead
- lower CPU burn in high concurrency scenarios
- lower memory churn from allocator-heavy paths
- fewer context switches and more stable scheduling
- better scalability under request bursts
- lower startup and shutdown latency

## 9. Traceability

This HLD is mapped to the runtime implementation in:

- `src/rdk-service-hub.c`
- `src/plugin_manager.c`
- `src/catalog_backend.c`
- `src/tool_registry.c`

## 10. Related documents

- [PERFORMANCE_OPTIMIZATION_REQUIREMENTS.md](PERFORMANCE_OPTIMIZATION_REQUIREMENTS.md)
- [PERFORMANCE_OPTIMIZATION_TASKS.md](PERFORMANCE_OPTIMIZATION_TASKS.md)
- [PERFORMANCE_OPTIMIZATION_LLD.md](PERFORMANCE_OPTIMIZATION_LLD.md)
