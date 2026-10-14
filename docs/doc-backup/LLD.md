# Multi-Plane Runtime Manager — Low-Level Design (Unified)

## Consolidation notice

This file is the unified LLD source for the project.

It consolidates and supersedes:

- DYNAMIC_PLUGIN_LLD.md
- CAPABILITY_PUBLICATION_LLD.md
- HYBRID_WATCHER_LLD.md
- METADATA_FIELDS_LLD.md
- METADATA_PLANE_TAXONOMY_LLD.md

## 1. Module map

Primary implementation units:

- `src/rdk-service-hub.c`
  - WRP decode/encode, kind dispatch, request broker path
  - metadata policy integration
  - transport handling, guardrails, HEALTH payload, notifications
- `src/plugin_manager.c/.h`
  - plugin discovery/load/reload/unload/invoke
  - provider registration and validation
- `src/tool_registry.c/.h`
  - canonical provider + tool registry
  - lookup, conflict resolution, and execution tracking
- `src/capability_publication.c/.h`
  - catalog + dynamic snapshot merge and conflict resolution
- `src/metadata_fields.c/.h`
  - metadata parsing/validation/policy/canonicalization
- `src/tool_registry.h`
  - runtime invocation and async status APIs

## 2. Request-path implementation details

### 2.1 Protocol dispatch

`handle_local_request()` routes by optional `kind`:

- default/`EXEC` → `handle_request()`
- `DESCRIBE` → `handle_describe_request()`
- `HEALTH` → `handle_health_request()`
- `PUSH` → `handle_push_request()`

The runtime treats this as request dispatch into the unified broker path, where the daemon resolves the tool through the registry before invoking a provider or legacy executor.

### 2.2 EXEC path

- decode request payload
- resolve tool name and plane in the runtime registry
- apply metadata gates and plane policy
- dispatch to the active provider adapter or static execution backend
- track execution state and normalize response output
- return either final result or execution ID for async work

### 2.3 DESCRIBE path

- builds per-plane capability snapshots
- uses merged catalog + dynamic entries with deterministic conflict policy
- returns a normalized capability inventory to callers and sync subsystems

### 2.4 HEALTH path

- returns `status=ok`
- exposes `push_guardrails` object with effective modes, limits, counters, and recent-attempt timestamp
- reports runtime provider health and stale execution cleanup metrics where applicable

### 2.5 PUSH path

Pre-apply ordering:

1. transport policy gate
2. payload-size guard
3. rate-limit guard
4. payload decode
5. authorization token gate
6. catalog apply/promote

Guard modes:

- `enforce`: reject violating request
- `monitor`: allow request, increment observed counters, emit warning logs
- `off`: bypass guardrail entirely

## 3. Dynamic provider internals

- candidate collection and normalization happen outside the hot invoke path
- commit-stage mutations occur under provider-manager and registry locks
- unload/reload waits release lock between drain polls
- scan/stop serialization avoids mutation race windows
- registry state is authoritative before provider execution begins

## 4. Registry and provider execution internals

The runtime registry maintains the canonical provider binding for all supported tool styles:

- native plugin binding from plugin adapter
- script plugin binding from script adapter
- external process binding from IPC registration adapter

The runtime resolves a requester’s tool name through the registry and then dispatches to the provider adapter responsible for the selected class.

Provider lifecycle state transitions:

- `REGISTERING` → `ACTIVE`
- `ACTIVE` → `DRAINING`
- `DRAINING` → `UNLOADED`
- `ACTIVE` → `DEGRADED`
- `DEGRADED` → `FAILED`
- `ACTIVE` → `FAILED`

Execution lifecycle state transitions:

- `PENDING` → `RUNNING`
- `RUNNING` → `COMPLETED`
- `RUNNING` → `FAILED`
- `RUNNING` → `CANCELED`
- `PENDING` → `CANCELED`

## 5. Capability publication internals

- snapshot structure contains normalized per-tool metadata fields
- conflict policy is applied during merge (`reject` / `plugin-priority` / `replace` / `merge`)
- shared output is used by both `DESCRIBE` and cloud capability sync
- provider metadata is merged with static catalog entries under unified policy rules

## 6. Metadata internals

- canonical planes include the runtime-defined plane names used by the registry and dispatch logic
- legacy aliases are mapped before strict-policy validation
- deterministic errors are mapped to stable response tokens and exit codes
- provider metadata must be normalized before entering the active registry

## 7. Guardrail telemetry internals

Counters maintained for:

- attempts, accepted
- rejected_transport
- rejected_unauthorized
- rejected_rate_limit
- rejected_payload_too_large
- rejected_other
- observed_rate_limit
- observed_payload_too_large

Synchronization:

- counters and last-attempt timestamp are protected by the runtime guard mutex

## 8. Transport and provider protocol internals

Runtime IPC is broker-managed and transport-neutral:

- spawn-per-request pipe: lightweight subprocess execution path
- Unix domain socket: persistent provider execution path
- JSON-first request and result envelope across both paths
- provider command types include `register`, `list-tools`, `invoke`, `status`, and `cancel`
- response types include `ack`, `result`, and `error`

This keeps the broker logic independent of the provider transport while preserving a single registry and status contract.

## 9. Configuration knobs

Key runtime controls:

- `MULTI_PLANE_RUNTIME_MANAGER_PUSH_REQUIRE_LOCAL_ONLY`
- `MULTI_PLANE_RUNTIME_MANAGER_PUSH_AUTH_TOKEN`
- `MULTI_PLANE_RUNTIME_MANAGER_PUSH_MAX_PAYLOAD_BYTES`
- `MULTI_PLANE_RUNTIME_MANAGER_PUSH_MIN_INTERVAL_MS`
- `MULTI_PLANE_RUNTIME_MANAGER_PUSH_RATE_LIMIT_MODE`
- `MULTI_PLANE_RUNTIME_MANAGER_PUSH_PAYLOAD_LIMIT_MODE`

## 10. Test mapping (LLD-level)

- Policy/spec checks: `tests/feature_matrix_spec_tests.c`
- Plugin lifecycle/concurrency checks:
  - `tests/dynamic_plugins_unit.c`
  - `tests/dynamic_plugins_live.c`
  - `tests/plugin_lock_scope_live.c`
  - `tests/plugin_unload_wait_lock_scope_live.c`
- Capability merge correctness: `tests/capability_publication_unit.c`
- Provider broker and socket lifecycle: `tests/tool_registry_socket_unit.c`

## 11. Related artifacts

- [REQUIREMENTS.md](REQUIREMENTS.md)
- [HLD.md](HLD.md)
- [COMPREHENSIVE_TEST_CASES.md](COMPREHENSIVE_TEST_CASES.md)

## 12. Current catalog backend implementation

### 12.1 Runtime model in effect

The active runtime default remains the JSON catalog model with a compact tmpfs-backed index. This is the default mode for the service and is the path used when no runtime backend override is supplied.

The optional LMDB implementation remains available behind a compile-time flag and is used only when the operator explicitly selects it or when a build enables it for a deployment that requires the database-backed variant.

### 12.2 Data layout and lookup path

The runtime keeps catalog metadata in a compact per-plane structure plus a tmpfs index file. The index stores a lightweight summary of each `plane`+`tool` entry, including the metadata needed for rapid resolution without retaining a full resident snapshot of every tool definition.

Lookups proceed as follows:

1. resolve the requested `plane` and `tool`
2. check the active loaded catalog objects first
3. fall back to the tmpfs index for quick key resolution
4. lazy-load the relevant detailed record only when needed
5. continue through the brokered policy and execution path unchanged

### 12.3 Memory and reload behavior

- the tmpfs index is intentionally compact and file-backed
- memory growth is bounded by the active working set rather than by a full in-memory catalog snapshot
- reload events rebuild the index and swap the active metadata safely under the catalog mutex
- JSON remains the safe default when LMDB is unsupported or absent

### 12.4 Configuration and telemetry

Current runtime-control keys include:

- `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_BACKEND=json|lmdb`
- `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_DIR=<path>`
- `MULTI_PLANE_RUNTIME_MANAGER_LOAD_ALL_TRIAGE_CONFIG=<0|1>`
- `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_USAGE_DISPLAY=<0|1>`

Operational telemetry and logs include backend selection, effective runtime state, and the one-shot resource usage snapshot when the feature is enabled.

### 12.5 Test strategy additions

- unit tests for catalog index generation and fallback behavior
- integration checks for startup with no explicit runtime arguments
- validation of JSON-default behavior when LMDB is disabled
- smoke checks for `PROCESS_RESOURCE_USAGE` output when enabled
- reload validation through `SIGHUP` without process restart
