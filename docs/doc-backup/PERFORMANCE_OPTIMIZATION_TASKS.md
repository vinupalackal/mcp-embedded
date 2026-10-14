# Performance Optimization Task List

## Objective

Improve runtime efficiency while preserving functional behavior and business logic. The optimization effort focuses on reducing CPU and memory overhead, reducing syscall churn, increasing request throughput, and improving startup/shutdown behavior without changing the tool execution semantics or policy model.

## Scope

This task list applies to the runtime management service implemented in:

- `src/rdk-service-hub.c`
- `src/plugin_manager.c`
- `src/catalog_backend.c`
- `src/tool_registry.c`

## Execution model

The optimization work is organized in phases:

1. Data-driven profiling and hotspot confirmation.
2. Low-risk performance improvements with minimal functional risk.
3. Concurrency and memory churn reduction.
4. Architecture-level scalability improvements.
5. Release verification against the existing test suite and targeted benchmarks.

## Task list

### Phase 1: Profiling and baseline capture

| ID | Task | Priority | Effort | Exit criterion |
|---|---|---|---|---|
| PERF-01 | Establish steady-state perf baseline with `perf`, `strace`, and RSS capture. | P0 | 1 day | Baseline recorded for CPU, memory, syscall count, and latency. |
| PERF-02 | Capture flamegraph output for request path and command execution path. | P0 | 1 day | Hotspots confirmed with actual source-level evidence. |
| PERF-03 | Validate current lock contention and context-switch profile. | P1 | 0.5 day | Root cause of scheduler churn identified and documented. |

### Phase 2: Low-risk request-path optimization

| ID | Task | Priority | Effort | Exit criterion |
|---|---|---|---|---|
| REQ-01 | Replace detached per-request thread creation with a bounded worker pool. | P0 | 2 days | No thread created per request; request queue is bounded and worker-based. |
| REQ-02 | Add request queue and worker lifecycle management to the main event loop. | P0 | 1 day | Single event loop remains responsive while worker threads process requests. |
| REQ-03 | Rate-limit and batch logging in the request path. | P1 | 0.5 day | Syslog overhead is reduced without losing error visibility. |

### Phase 3: Command execution and system call optimization

| ID | Task | Priority | Effort | Exit criterion |
|---|---|---|---|---|
| CMD-01 | Reduce `fork()`/`pipe()` overhead in command execution path. | P0 | 2-3 days | Command execution uses a bounded execution model and lower syscall count. |
| CMD-02 | Add per-command concurrency throttling or a command worker pool. | P1 | 1 day | No unbounded command fan-out under load. |
| CMD-03 | Reduce `read()`/`poll()` loop overhead and tighten buffer handling. | P1 | 1 day | Output capture remains correct and avoids unnecessary wakeups. |

### Phase 4: Catalog and metadata memory optimization

| ID | Task | Priority | Effort | Exit criterion |
|---|---|---|---|---|
| CAT-01 | Build a compact lookup index for catalog resolution. | P0 | 2 days | Request lookup avoids repeated linear scans. |
| CAT-02 | Reduce duplicated string allocation in catalog and plugin metadata paths. | P1 | 1 day | Memory churn reduced without altering metadata semantics. |
| CAT-03 | Reuse temporary buffers and arena memory for request decode/encode paths. | P1 | 1-2 days | Allocator pressure reduced during steady-state traffic. |

### Phase 5: Plugin and watcher optimization

| ID | Task | Priority | Effort | Exit criterion |
|---|---|---|---|---|
| PLG-01 | Improve plugin watch efficiency with event-driven re-scan logic. | P1 | 1-2 days | Reduced redundant scans and no extra wakeups when no changes occur. |
| PLG-02 | Deduplicate plugin reload work and avoid repeated full scans. | P1 | 1 day | Plugin reload remains correct and cheaper under churn. |
| PLG-03 | Reduce lock scope around plugin registry mutation and unload paths. | P1 | 1-2 days | Lower lock contention and less critical-section imbalance. |

### Phase 6: Release build and compile-time tuning

| ID | Task | Priority | Effort | Exit criterion |
|---|---|---|---|---|
| REL-01 | Tune release build flags (`-O2`, optional `-O3`, LTO, PGO). | P1 | 0.5 day | Release binary demonstrates measurable CPU gains without regressions. |
| REL-02 | Validate warning-clean strict build remains intact after optimization changes. | P0 | 0.5 day | `-Wall -Wextra -Wpedantic -Werror` remains satisfied. |

### Phase 7: Regression validation

| ID | Task | Priority | Effort | Exit criterion |
|---|---|---|---|---|
| REG-01 | Run existing project test suite after each optimization milestone. | P0 | 0.5 day | No business-logic regressions. |
| REG-02 | Run targeted benchmark scripts under steady-state load. | P0 | 1 day | Observed improvement against baseline metrics. |
| REG-03 | Validate startup, shutdown, and reload behavior with signal-driven reload tests. | P0 | 1 day | Runtime continues operating under reload and shutdown transitions. |

## Dependency ordering

1. PERF-01, PERF-02, PERF-03
2. REQ-01, REQ-02, REQ-03
3. CMD-01, CMD-02, CMD-03
4. CAT-01, CAT-02, CAT-03
5. PLG-01, PLG-02, PLG-03
6. REL-01, REL-02
7. REG-01, REG-02, REG-03

## Definition of done

The optimization program is complete when all of the following are true:

- CPU utilization is reduced under load.
- Memory footprint is reduced without harming runtime stability.
- Request latency and throughput are improved at the target workload.
- Syscall count and context-switch rate are reduced compared with the baseline.
- Startup and shutdown remain stable.
- The project test suite and relevant integration tests continue to pass.
- No functional business logic is changed beyond performance-related refactoring.

## Exit gate for each task

Each task must satisfy all of the following before closure:

- Code compiles in the project’s strict build configuration.
- Test suite passes for the touched runtime area.
- Profiling confirms a measurable improvement or no regression.
- The change is reviewed for any effect on safety, policy enforcement, or command execution semantics.
