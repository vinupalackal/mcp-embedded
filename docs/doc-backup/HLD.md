# Multi-Plane Runtime Manager — High-Level Design (Unified)

## Consolidation notice

This file is the unified HLD source for the project.

It consolidates and supersedes:

- DYNAMIC_PLUGIN_HLD.md
- CAPABILITY_PUBLICATION_HLD.md
- HYBRID_WATCHER_HLD.md
- METADATA_FIELDS_HLD.md
- METADATA_PLANE_TAXONOMY_HLD.md

## 1. System context

The runtime manager serves cloud/device diagnostic requests over the daemon broker,
resolves tool requests through a unified registry, applies policy and auth checks,
and dispatches work to static catalog entries or dynamic provider backends.

The architecture now reflects a brokered tool runtime rather than a single direct command executor.

Core architecture layers:

1. Transport and envelope layer (Parodus / WRP and local IPC transports).
2. Request broker and request validation layer.
3. Tool registry and conflict-resolution layer.
4. Policy, auth, and resource gate layer.
5. Provider adapter layer for native, script, and external-process providers.
6. Async execution tracking and cancellation layer.
7. Catalog compatibility and legacy static command execution path.
8. Observability and guardrail layer.

## 2. Functional subsystems

### 2.1 Brokered request subsystem

- accepts tool requests from the transport layer
- resolves the target plane and runtime tool name
- checks provider validity, auth, and policy before execution
- normalizes execution results into a canonical response format
- returns result payloads or execution IDs to the caller

### 2.2 Tool registry subsystem

- stores provider and tool metadata by name and plane
- supports static catalog entries and dynamic provider entries in one registry model
- enforces duplicate-name conflict policy under `deny`, `replace`, `priority`, or `merge`
- tracks provider health, state, and async execution records

### 2.3 Dynamic provider subsystem

- supports native `.so` providers, script providers, and API-style external providers
- validates provider identity, path, ownership, and transport trust boundary
- loads provider metadata and advertises tool descriptors into the registry
- supports reload/unload with in-flight invoke draining and stale-provider quarantine

### 2.4 Capability publication subsystem

- builds unified capability snapshots from catalog and dynamic sources
- applies deterministic conflict policy to the merged view
- reuses shared snapshot paths for describe and capability-sync flows
- publishes at startup and after relevant runtime changes

### 2.5 Metadata and policy subsystem

- decodes metadata fields from requests and provider metadata
- applies strict validation and policy decisions
- normalizes legacy aliases into canonical plane names
- produces deterministic reject tokens and logs for invalid tool definitions or provider metadata

### 2.6 PUSH security / availability subsystem

- enforces local-only transport policy by default
- optionally enforces shared-token authorization or stronger local trust checks
- applies payload-size and request-rate guardrails
- supports rollout-safe modes (`enforce`, `monitor`, `off`)

### 2.7 Observability subsystem

- emits structured logs for accept/reject and policy outcomes
- exposes health-time guardrail posture and counters
- tracks accepted/rejected/observed provider and execution events

## 3. Data and control flows

1. Inbound request decoded at the transport boundary.
2. Registration or provider metadata is normalized into the runtime registry.
3. Policy gates run before any provider dispatch or static command execution.
4. Registry resolves the provider and tool metadata for the requested plane.
5. Execution proceeds through the provider adapter or legacy executor path.
6. Async state is tracked and final results are normalized before response send.
7. Metrics, health, and request history are updated for observability.

## 4. Concurrency model

- short critical sections for request-path responsiveness
- discovery/reload operations separated from commit phases
- registry and provider state mutation protected by dedicated locks
- in-flight invoke tracking protects unload/reload safety
- async execution state is updated under explicit lifecycle control

## 5. Reliability and rollback posture

- guardrails can start in `monitor` mode for canary rollout
- health counters support live rollback decisioning
- enforced policy mode can be reverted by config without rebuild
- stale provider connections are quarantined and revalidated rather than silently reused
- execution timeouts and cancellation state are tracked as first-class runtime failures

## 6. Validation strategy (HLD-level)

- unit/spec vectors for policy decisions and clamping
- integration tests for provider lifecycle and churn safety
- integration tests for async execution state transitions
- end-to-end CI with dynamic plugin live tests and publication checks

## 7. Related artifacts

- [REQUIREMENTS.md](REQUIREMENTS.md)
- [LLD.md](LLD.md)
- [COMPREHENSIVE_TEST_CASES.md](COMPREHENSIVE_TEST_CASES.md)
- Phase implementation notes:
  - PHASES_IMPLEMENTATION.md

## 8. Provider and execution model

The runtime now treats provider execution as a brokered runtime function rather than as an ad hoc shell path.

### 8.1 Native provider path

- `.so` provider is loaded and validated at startup or reload time
- ABI symbols are resolved and tool descriptors normalized into registry entries
- invocation occurs via the canonical runtime contract

### 8.2 Script provider path

- script is discovered from a trusted plugin directory
- `--list-tools` returns metadata for registration
- `--invoke` returns structured JSON result or error payload
- async support uses `--status` and `--cancel` when declared

### 8.3 API-style external provider path

- provider connects through a local socket or spawn-per-request pipe
- request and response use a JSON-first broker envelope
- `register`, `list-tools`, `invoke`, `status`, and `cancel` commands are broker-visible
- runtime tracks provider health and execution state across reconnect and timeout events

## 9. Catalog storage and low-memory runtime model

### 9.1 Current default posture

The runtime defaults to the JSON catalog path with a compact tmpfs-backed index and lazy metadata loading. This preserves compatibility with resource-constrained platforms while avoiding a compulsory database dependency.

The design remains intentionally conservative:

1. `json` remains the default backend.
2. LMDB is optional and compile-time gated; it is not required for normal operation.
3. A tmpfs-backed catalog index provides quick lookups while keeping resident memory bounded.
4. Full content is loaded only when a specific tool entry becomes active.

### 9.2 Optional LMDB mode

The LMDB-backed path remains available as an explicit opt-in build/runtime mode, not as the default operational model. When enabled, it follows the same keying pattern used elsewhere in the project (`plane` + `tool`), but the runtime still prefers the JSON + tmpfs default in deployment unless the operator explicitly selects LMDB.

### 9.3 Reloadability and safety

The runtime reload flow remains signal-driven and swap-safe:

1. rebuild a replacement catalog snapshot or index
2. validate the updated state
3. lock catalog access during the swap
4. atomically replace the active pointer/version metadata
5. release the lock and continue serving requests

This avoids unsafe in-place mutation during active request servicing.

### 9.4 Observability and operator posture

- resource usage reporting is opt-in and emits a single snapshot when enabled
- default startup continues even without explicit runtime arguments
- backend selection, catalog path, and merged-triage mode are logged at startup and reload
- fallback to JSON remains the safe default when LMDB support is unavailable
