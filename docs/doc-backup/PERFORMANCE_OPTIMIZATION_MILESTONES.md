# Performance Optimization Milestone Plan

## Objective

Deliver the performance optimization work in small, testable milestones without changing functional behavior, business logic, or security policy semantics. Each milestone is tied to a concrete runtime hot path in:

- [src/rdk-service-hub.c](../src/rdk-service-hub.c)
- [src/plugin_manager.c](../src/plugin_manager.c)
- [src/catalog_backend.c](../src/catalog_backend.c)
- [src/tool_registry.c](../src/tool_registry.c)

## Milestone 0: Baseline and profiling

### Goal

Capture a clean before-state for CPU, memory, latency, and syscall behavior.

### Scope

- establish a steady-state benchmark under representative request load
- collect `perf`, `strace`, `heaptrack`, and runtime RSS data
- confirm the main hotspots in the current implementation

### Code focus

- dispatch loop around `service_one_message()` in [src/rdk-service-hub.c](../src/rdk-service-hub.c#L4601-L4619)
- command execution in `run_command()` in [src/rdk-service-hub.c](../src/rdk-service-hub.c#L2818-L2909)
- plugin watcher loop in [src/plugin_manager.c](../src/plugin_manager.c#L500-L531)
- catalog lookup path in [src/catalog_backend.c](../src/catalog_backend.c#L675-L691)

### Exit criteria

- perf report captures the dominant hotspot(s)
- strace reveals the syscall pattern under load
- CPU and memory baseline is recorded
- no optimization work starts before this baseline is captured

### Validation

```bash
perf stat -d ./build/rdk-service-hub
perf record -F 99 -g -- ./build/rdk-service-hub
perf report --stdio
strace -ff -tt -T -s 128 -e trace=process,network,fd ./build/rdk-service-hub
ctest --test-dir build --output-on-failure
```

---

## Milestone 1: Bound request dispatch concurrency

### Goal

Eliminate detached thread creation per request and replace it with a bounded worker pool.

### Why this is first

This is the clearest request-path bottleneck and the safest large-impact optimization.

### Code changes

- add a bounded request queue adjacent to the socket loop in [src/rdk-service-hub.c](../src/rdk-service-hub.c#L4947-L4994)
- replace the detached thread block in `service_one_message()` with queue push
- create a fixed number of worker threads during startup
- process requests from workers instead of spawning a new thread per message

### Implementation pattern

```c
typedef struct request_queue_s {
    wrp_req_t **items;
    size_t head;
    size_t tail;
    size_t count;
    size_t capacity;
    pthread_mutex_t lock;
    pthread_cond_t cond;
    int running;
} request_queue_t;
```

### Exit criteria

- no new detached thread is created for each request
- queue depth remains bounded under load
- request handling continues to match prior behavior
- tests still pass and p95 latency improves

### Validation

```bash
ctest --test-dir build --output-on-failure
perf stat -d ./build/rdk-service-hub
```

---

## Milestone 2: Reduce command execution overhead

### Goal

Keep the secure exec model but lower syscall and process overhead under load.

### Code focus

- [src/rdk-service-hub.c](../src/rdk-service-hub.c#L2818-L2909)

### Planned changes

- add command concurrency throttle with a bounded execution queue
- keep output timeout and cap semantics unchanged
- avoid unbounded simultaneous command execution during bursts
- reduce repeated expensive `fork()` / `pipe()` / `read()` sequences when possible

### Implementation pattern

```c
#define MAX_CONCURRENT_COMMANDS 8
sem_t g_cmd_gate;
```

### Exit criteria

- command execution remains correct and timeouts still enforced
- CPU usage drops during burst traffic
- no change to allowlist or denylist behavior

### Validation

```bash
ctest --test-dir build --output-on-failure
strace -ff -tt -T -s 128 -e trace=process,fd ./build/rdk-service-hub
```

---

## Milestone 3: Catalog lookup index and metadata reduction

### Goal

Remove repeated linear catalog scanning and reduce temporary allocation churn.

### Code focus

- [src/catalog_backend.c](../src/catalog_backend.c#L314-L691)
- [src/catalog_backend.c](../src/catalog_backend.c#L772-L820)

### Planned changes

- build a compact lookup index keyed by `plane + tool`
- reduce repeated JSON serialization and duplicate string copies
- reduce `strdup()` churn in the hot catalog path
- preserve reload safety and tmpfs-backed behavior

### Implementation pattern

```c
typedef struct {
    char plane[32];
    char tool[128];
    size_t entry_index;
} catalog_lookup_entry_t;
```

### Exit criteria

- request lookup no longer walks the whole catalog repeatedly
- memory profile shows less transient allocation churn
- JSON default backend remains correct

### Validation

```bash
ctest --test-dir build --output-on-failure
valgrind --tool=massif ./build/rdk-service-hub
```

---

## Milestone 4: Plugin watcher and reload efficiency

### Goal

Reduce redundant plugin scans and duplicated reload work without changing plugin behavior.

### Code focus

- [src/plugin_manager.c](../src/plugin_manager.c#L500-L531)
- [src/plugin_manager.c](../src/plugin_manager.c#L633-L728)

### Planned changes

- suppress duplicate scans when plugin directory state is unchanged
- add debounce or file-state fingerprinting for plugin change detection
- reduce unnecessary wakeups in poll mode
- keep watch behavior safe on Linux with inotify when available

### Exit criteria

- plugin scan count drops under steady state
- directory change handling remains correct
- no plugin churn regressions

### Validation

```bash
ctest --test-dir build --output-on-failure
perf record -F 99 -g -- ./build/rdk-service-hub
```

---

## Milestone 5: Logging and allocator cleanup

### Goal

Remove expensive synchronous logging and reduce allocator pressure from hot request paths.

### Code focus

- [src/rdk-service-hub.c](../src/rdk-service-hub.c#L4667-L4688)
- [src/rdk-service-hub.c](../src/rdk-service-hub.c#L2969-L3212)

### Planned changes

- introduce a bounded async or rate-limited logger
- avoid calling `syslog()` on low-value request events during high load
- replace repeated high-churn buffer allocations with small arena reuse
- review object lifetimes for request and response payloads

### Exit criteria

- less log syscall overhead under load
- memory profile improves in steady-state execution
- no loss of error visibility or startup diagnostics

### Validation

```bash
ctest --test-dir build --output-on-failure
heaptrack ./build/rdk-service-hub
```

---

## Milestone 6: Release tuning and compiler optimization

### Goal

Apply build-time tuning for the release binary without changing behavior.

### Planned changes

- build release with `-O2` and `-pipe`
- evaluate `-O3` only if per-benchmark gains justify it
- add LTO for final release image
- evaluate PGO on representative workload if feasible

### Exit criteria

- release binary shows measurable improvement and remains stable
- no warning regressions in strict build mode

### Validation

```bash
cmake -S . -B build-release -DCMAKE_BUILD_TYPE=Release
cmake --build build-release
cmake -S . -B build-warn -DCMAKE_C_FLAGS='-Wall -Wextra -Wpedantic -Werror'
cmake --build build-warn --target rdk-service-hub
```

---

## Milestone 7: Regression and final verification

### Goal

Confirm the runtime remains functionally correct and materially faster than baseline.

### Scope

- run full project test suite
- compare before/after perf, RSS, syscall, and latency numbers
- confirm startup, shutdown, and SIGHUP reload behavior remain stable

### Validation

```bash
ctest --test-dir build --output-on-failure
perf stat -d ./build/rdk-service-hub
strace -ff -tt -T -s 128 -e trace=process,network,fd ./build/rdk-service-hub
```

### Exit criteria

- all tests pass
- performance improvements are measured and documented
- no security or policy regressions are introduced

---

## Recommended implementation sequence

1. Milestone 0: baseline collection
2. Milestone 1: worker pool dispatch
3. Milestone 2: command execution throttle
4. Milestone 3: catalog lookup optimization
5. Milestone 4: plugin watcher dedupe
6. Milestone 5: logging and allocator cleanup
7. Milestone 6: release tuning
8. Milestone 7: final regression verification

## Success definition

The optimization effort is successful when the runtime shows measurable gains in:

- CPU utilization
- memory footprint
- request latency
- throughput
- syscall count
- context-switch rate

while preserving:

- startup behavior
- runtime reload safety
- command policy enforcement
- tooling and feature parity
- the project test suite results
