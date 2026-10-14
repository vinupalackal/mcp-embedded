# Dynamic Tool Registration and Remote Execution HLD

## 1. Purpose

This document defines the high-level design for allowing the Multi-Plane Runtime Manager (MPRM) to register, discover, and invoke tools exposed by three provider classes:

- native shared libraries (`.so` plugins)
- script or executable plugins
- external processes registered dynamically at runtime

The design preserves a single registry and invocation model across all providers while introducing a brokered execution path for remote callers. Remote callers never talk directly to an external provider process; instead, the daemon resolves the tool, validates plane and policy, forwards the request over the provider transport, tracks execution state, and returns a normalized response.

This document is the HLD companion to the implementation-ready specification in [guides/DYNAMIC_TOOL_REGISTRATION_SPEC.md](guides/DYNAMIC_TOOL_REGISTRATION_SPEC.md).

## 2. Scope

### In scope

- unified tool registry for `.so`, script, and external-process providers
- dynamic registration and deregistration from trusted local callers
- plane-scoped lookup and conflict resolution
- sync and async invocation semantics
- execution tracking and polling for long-running tools
- Unix domain socket as the primary external-process transport
- pipe-based invocation for supervisor-managed subprocesses
- security validation for socket ownership, auth, and provider metadata

### Out of scope

- arbitrary external code execution from untrusted sources
- direct remote access to provider processes
- undocumented transport fallbacks or compatibility-only modes
- implementation of remote provider logic in this document beyond the broker contract

## 3. Goals

- unify runtime registration and invocation across provider types
- keep a single registry and tool lookup path regardless of provider implementation
- allow dynamic registration by trusted local process or broker
- enforce plane isolation, conflict policy, and auth rules consistently
- support both immediate and long-running tool execution
- preserve control-plane safety and observability for tool execution

## 4. Non-goals

- exposing raw process execution primitives to cloud callers
- accepting untrusted registration without validation
- bypassing runtime registry resolution and dispatch
- allowing the same tool name to be registered without policy enforcement

## 5. Design principles

### 5.1 Single broker model

The MPRM daemon is the sole broker for tool execution. It resolves tools, checks security and plane policy, and dispatches to the provider. This makes the runtime the enforcement point and ensures consistent logging, telemetry, and timeout handling.

### 5.2 Provider neutrality

The runtime must treat providers uniformly:

- native `.so` plugins implement the native ABI
- script providers implement the JSON metadata and invoke contract
- external providers use a local registration API and local IPC transport

The registry contracts are normalized before they reach caller-facing APIs.

### 5.3 Security-first transport policy

All external providers must validate:

- socket path existence and socket type
- socket ownership and trust boundary
- auth token or signed registration challenge
- tool metadata before registration enters the active registry

The runtime quarantines any provider that fails repeated validation or shows stale or unauthorized behavior.

### 5.4 JSON-first result contract

The brokered runtime adopts a JSON-first message contract for all tool requests and responses. The outer envelope is always JSON, and the top-level `payload` and `result` fields are structured JSON objects or arrays. If a tool needs to return opaque bytes, the protocol carries them in a typed `binary` field with explicit `content_type`, `encoding`, and base64-encoded `data` metadata instead of exposing raw binary as the top-level result object. This preserves broker validation, logging, and schema enforcement while still supporting binary tool outputs when required.

The provider-side protocol is intentionally symmetric across transport types. The daemon sends the same JSON envelope whether the provider is a one-shot child process over stdin/stdout or a long-lived process over a Unix domain socket. In both cases, the provider interprets the `type` field (`register`, `list-tools`, `invoke`, `status`, `cancel`), and it answers with `ack`, `result`, or `error`.

## 6. System overview

The runtime contains the following major subsystems:

1. Provider discovery and registration
2. Registry and conflict resolution
3. Invocation broker and dispatcher
4. Async execution tracker
5. Policy and auth guard
6. Provider transport adapters
7. Observability and health

The runtime architecture is intentionally layered:

- transport and IPC layer handles Unix socket or pipe communication
- provider adapter layer maps provider-specific behavior to the generic tool contract
- registry layer stores tool metadata and provider binding
- execution broker routes remote requests and tracks lifetime
- policy layer enforces plane, auth, ownership, and conflict rules

## 7. Runtime architecture

### 7.1 Components

#### Provider discovery module

Responsible for locating provider candidates, including plugin libraries and known external provider endpoints. It performs validation and normalizes metadata before registration.

#### Native plugin adapter

Loads `.so` libraries with `dlopen()`, resolves the required ABI symbols, enumerates tool metadata, and invokes the provider function through the native interface.

#### Script plugin adapter

Executes the provider script with `--list-tools` and `--invoke`, validates JSON output, and normalizes it to the canonical response contract.

#### External process adapter

Maintains a registration API and a local IPC transport. It accepts provider registration from trusted local callers and handles message exchange over a Unix domain socket or pipe.

#### Registry and conflict resolver

Stores provider records and tool metadata keyed by tool name and plane. Handles conflict policies such as `deny`, `replace`, `priority`, and `merge`.

#### Invocation dispatcher

Resolves tool names to a provider, enforces plane and authorization rules, performs transport-specific send/receive operations, and normalizes provider results before returning them to the caller.

#### Async execution tracker

Creates and updates execution records keyed by `execution_id`, stores lifecycle and progress, and exposes polling and cancellation operations.

#### Policy and auth guard

Validates plane scoping, auth tokens, socket ownership, path trust policy, and message schema restrictions before enabling a provider or executing a tool.

## 8. Registration flow

1. The provider sends or receives a registration request.
2. The daemon validates identity, socket ownership, auth token, and plane.
3. Metadata is converted into canonical descriptors.
4. The daemon checks for conflicts under the active conflict policy.
5. If accepted, the provider moves from `REGISTERING` to `ACTIVE`.
6. The runtime stores the provider and tool records in the registry.

### Registration state model

- `REGISTERING`
- `ACTIVE`
- `DRAINING`
- `DEGRADED`
- `FAILED`
- `UNLOADED`

Transitions:

- `REGISTERING -> ACTIVE` when metadata, socket, and auth validation pass
- `ACTIVE -> DRAINING` when reload or shutdown begins
- `DRAINING -> UNLOADED` after inflight work drains or times out
- `ACTIVE -> DEGRADED` under repeated soft failure or degraded health
- `DEGRADED -> FAILED` on repeated validation or transport failure
- `ACTIVE -> FAILED` when the process exits unexpectedly or the socket disappears

## 9. Invocation flow

### 9.1 Sync invocation

1. caller submits tool request with `tool`, `plane`, and transaction metadata
2. runtime resolves the tool in the registry
3. policy guard validates plane, source, and provider health
4. broker dispatches the request to the provider over the selected transport
5. provider returns final result or explicit error
6. runtime normalizes error and response before returning to caller

### 9.2 Async invocation

1. caller submits async request with `async=true`
2. runtime creates an `execution_id` and execution record
3. provider starts work and returns a started status
4. caller polls status or fetches final result
5. runtime updates the execution record as `RUNNING`, `COMPLETED`, `FAILED`, or `CANCELED`

### Execution lifecycle

- `PENDING`
- `RUNNING`
- `COMPLETED`
- `FAILED`
- `CANCELED`

Transitions:

- `PENDING -> RUNNING` when request is accepted by provider
- `RUNNING -> COMPLETED` when final result is returned
- `RUNNING -> FAILED` when timeout or provider error occurs
- `RUNNING -> CANCELED` when caller cancels or daemon times out
- `PENDING -> CANCELED` when request is rejected before provider accepts it

## 10. Transport model

The runtime supports the following external-provider transport order:

1. Unix domain socket (default and recommended)
2. stdin/stdout pipe for supervisor-managed subprocesses

The runtime rejects or quarantines providers whose socket is not trusted or whose auth material fails validation.

### Why Unix domain sockets are primary

- low overhead compared with a loopback network socket
- same-host trust boundary without exposing a network listener
- cleaner lifecycle management and connection supervision
- low operational complexity for brokered local execution

## 11. Policy and security model

### 11.1 Plane policy

- tool registration is plane-scoped
- remote request must specify a valid plane or inherit a configured default
- tool invocation across plane boundaries must be explicitly allowed

### 11.2 Security checks

- only trusted local callers may register providers
- provider endpoints must be local sockets or approved local paths
- auth tokens or signed registration requests must be validated
- event logs and telemetry must capture registration and execution transitions

### 11.3 Conflict policy

The conflict resolver supports:

- `deny`: reject duplicate tool names
- `replace`: unload old provider and accept new one
- `priority`: prefer provider with higher precedence
- `merge`: permit same tool name across different planes or provider classes

## 12. Error model

The runtime normalizes failures into deterministic runtime error conditions:

- `ERR_PROVIDER_NOT_FOUND`
- `ERR_TOOL_NOT_FOUND`
- `ERR_PROVIDER_UNAVAILABLE`
- `ERR_PROVIDER_INVALID`
- `ERR_PROVIDER_AUTH_FAILED`
- `ERR_TOOL_CONFLICT`
- `ERR_EXECUTION_TIMEOUT`
- `ERR_EXECUTION_CANCELED`
- `ERR_REMOTE_DISPATCH_FAILED`

The daemon preserves execution state even when a provider disappears unexpectedly and does not retry forever for known-invalid providers.

## 13. Observability and health

The daemon tracks:

- provider status transitions
- execution count, duration, and timeout rate
- invalid registration attempts and rejected auth messages
- per-plane invoke failures and provider health degradations
- status polling and final-result latency

This supports both operational diagnosis and deterministic runtime failure handling.

## 14. Validation and testing strategy

At the HLD level, validation includes:

- unit tests for provider metadata validation and conflict policy
- live tests for provider registration and deregistration
- async execution tests for polling, timeout, and cancellation
- socket and auth failure tests for quarantine behavior
- plane-scoped invoke tests for routing and policy enforcement

## 15. Traceability to the spec

This design directly implements the requirements defined in [guides/DYNAMIC_TOOL_REGISTRATION_SPEC.md](guides/DYNAMIC_TOOL_REGISTRATION_SPEC.md), including:

- provider registration contract
- JSON request/response envelope validation
- brokered remote execution model
- async execution tracking
- provider lifecycle state machine
- Unix socket primary transport selection

## 16. Compatibility

This design is compatible with the existing `.so` plugin interface and the script plugin contract already used in the runtime. It extends the model to trusted external process providers while preserving one canonical registry and invocation API for remote callers.
