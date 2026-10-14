# Performance Optimization Low-Level Design

## 1. Scope

This document describes the low-level implementation plan for the optimization program. It is intentionally tied to the current runtime architecture and hot spots found in the implementation.

## 2. Target hot paths

The following code paths are the main optimization targets:

- request dispatch and socket handling in `src/rdk-service-hub.c`
- detached thread creation in `service_one_message()`
- command execution in `run_command()`
- plugin discovery/watcher logic in `src/plugin_manager.c`
- catalog lookup and tmpfs index handling in `src/catalog_backend.c`
- registry and metadata operations in `src/tool_registry.c`

## 3. Detailed design

### 3.1 Request worker pool

#### Summary

Replace detached thread-per-request dispatch with a bounded request worker pool.

#### Implementation

Add new state to the runtime:

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

Add global runtime state:

```c
static request_queue_t g_request_queue;
static pthread_t *g_request_workers;
static size_t g_request_worker_count;
```

Initialize at startup and destroy gracefully during shutdown.

#### Behavior

- main event loop receives socket message
- request is enqueued to the queue
- worker threads pop request and call `handle_local_request()`
- no thread is created for each message

#### Safety

- queue depth is bounded
- if queue is full, back pressure is applied and the runtime can reject or defer work without affecting process health

### 3.2 Command execution scheduler

#### Summary

The runtime will preserve the `fork/exec` security model but control the number of active command invocations.

#### Implementation

Introduce a command gate with a semaphore or queue:

```c
sem_t g_cmd_gate;
```

Set a maximum like `MAX_CONCURRENT_COMMANDS 8` or a deployment-tuned value.

#### Behavior

- command execution waits on the gate before running
- the gate is released after command completion
- timeout, output cap, and exit code semantics remain unchanged

This keeps the model operational but prevents overload spikes.

### 3.3 Catalog lookup index

#### Summary

Build a compact lookup index for `plane` + `tool` entries and use it during request dispatch.

#### Implementation

Add a lookup table alongside the tmpfs/index metadata:

```c
typedef struct {
    char plane[32];
    char tool[128];
    size_t entry_index;
} catalog_lookup_entry_t;
```

At catalog load:

- parse JSON once
- build index entries
- compute a stable hash on `plane:tool`
- store result in an array or hash map

During request processing:

- compute key from requested plane and tool
- use the lookup index instead of scanning all entries

### 3.4 Logging reduction

#### Summary

Move request-path logging to a bounded asynchronous logger.

#### Implementation

Add a ring buffer and logger thread:

```c
typedef struct log_ring_s {
    char *bufs[LOG_RING_CAPACITY];
    size_t head;
    size_t tail;
    size_t count;
    pthread_mutex_t lock;
    pthread_cond_t cond;
} log_ring_t;
```

Behavior:

- request path enqueues low-value log records
- logger thread serializes writes to `syslog()`
- high-priority errors still emit immediately

This reduces cost during request bursts without losing observability.

### 3.5 Plugin watch deduplication

#### Summary

Reduce duplicate plugin scans and notifications while preserving plugin detection semantics.

#### Implementation

Add a change fingerprint based on:

- plugin directory mtime or event signature
- plugin file list hash
- debounce window

If the directory state has not changed, skip full rescan. Only when relevant changes occur does the plugin manager run a refresh.

### 3.6 Memory arena

#### Summary

Reduce repeated `malloc()` and `strdup()` pressure in hot paths.

#### Implementation

Provide a small arena for temporary runtime objects:

```c
typedef struct {
    char *base;
    size_t used;
    size_t capacity;
} temp_arena_t;
```

Use it for temporary names and transient request metadata. Reuse it per worker thread or per request rather than calling `malloc()` for every small object.

## 4. Error handling and rollback

Each optimization must preserve existing error behavior:

- timeout semantics stay intact
- command output cap remains enforced
- plugin conflict policy remains unchanged
- reload path remains signal-safe and operational

If a patch cannot preserve these guarantees, it is not eligible.

## 5. Configuration knobs

New runtime knobs may be introduced for tuning, but they must remain optional and default-safe:

- `MULTI_PLANE_RUNTIME_MANAGER_REQUEST_WORKER_COUNT`
- `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_CONCURRENCY`
- `MULTI_PLANE_RUNTIME_MANAGER_LOG_RATE_LIMIT_MS`
- `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_SCAN_DEBOUNCE_MS`

These knobs are tuning aids, not business-logic changes.

## 6. Validation plan

This low-level design is validated through:

- compile and strict warning build
- unit and integration test suite
- `perf` profiling
- `strace` syscall count comparison
- `valgrind`/`heaptrack` memory profile
- steady-state throughput benchmarking

## 7. Related documents

- [PERFORMANCE_OPTIMIZATION_REQUIREMENTS.md](PERFORMANCE_OPTIMIZATION_REQUIREMENTS.md)
- [PERFORMANCE_OPTIMIZATION_HLD.md](PERFORMANCE_OPTIMIZATION_HLD.md)
- [PERFORMANCE_OPTIMIZATION_TASKS.md](PERFORMANCE_OPTIMIZATION_TASKS.md)
