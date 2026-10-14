# Dynamic Tool Registration and Remote Execution LLD

## 1. Purpose

This document provides the low-level design for the dynamic tool registration and remote execution system defined by the HLD and the implementation-ready spec. It focuses on module boundaries, runtime data structures, protocol mapping, validation behavior, and execution flow.

## 2. Module map

### 2.1 Core modules

- `tool_registry.c` / `tool_registry.h`
  - provider registry maintenance
  - tool lookup and conflict resolution
  - active provider snapshot generation

- `tool_dispatcher.c` / `tool_dispatcher.h`
  - brokered request dispatch
  - transport selection and message serialization
  - sync/async invoke normalization

- `tool_execution_tracker.c` / `tool_execution_tracker.h`
  - execution record lifecycle management
  - polling and cancellation helpers
  - async result aggregation

- `tool_provider_adapter.c` / `tool_provider_adapter.h`
  - `.so`, script, and external provider normalization
  - provider validation and registration checks

- `tool_policy.c` / `tool_policy.h`
  - plane enforcement
  - auth and ownership validation
  - conflict policy evaluation

- `tool_ipc_socket.c` / `tool_ipc_socket.h`
  - Unix socket listener and message framing
  - connection health and retry handling

## 3. Data model

### 3.1 Provider record

```c
typedef struct {
    char provider_id[128];
    char provider_name[128];
    char plane[32];
    char process_path[512];
    char socket_path[512];
    char auth_token[256];
    uint32_t flags;
    uint32_t request_timeout_ms;
    bool supports_async;
    enum provider_state state;
    size_t tool_count;
    tool_descriptor_t tools[64];
    uint64_t last_heartbeat_ms;
    uint64_t created_ms;
    uint64_t updated_ms;
} provider_record_t;
```

### 3.2 Tool descriptor

```c
typedef struct {
    char name[128];
    char description[256];
    char plane[32];
    char provider_type[32];
    char invoke_mode[16];
    bool supports_async;
    uint32_t timeout_ms;
    uint32_t capabilities_mask;
} tool_descriptor_t;
```

### 3.3 Execution record

```c
typedef struct {
    char execution_id[128];
    char tool_name[128];
    char provider_id[128];
    char transaction_uuid[64];
    char plane[32];
    enum exec_state state;
    int exit_code;
    char progress[256];
    char result_json[4096];
    uint64_t started_ms;
    uint64_t updated_ms;
    uint32_t timeout_ms;
} execution_record_t;
```

### 3.4 Registry snapshot

```c
typedef struct {
    provider_record_t *providers;
    size_t provider_count;
    size_t provider_capacity;
    execution_record_t *executions;
    size_t execution_count;
    size_t execution_capacity;
} registry_snapshot_t;
```

## 4. API contracts

### 4.1 External provider registration

```c
int tool_registry_register_external(const tool_register_req_t *req,
                                   tool_register_resp_t *resp);

int tool_registry_unregister_external(const char *provider_id,
                                     const char *tool_name,
                                     tool_register_resp_t *resp);

int tool_registry_list_external(tool_descriptor_t **tools, size_t *count);
```

### 4.2 Runtime invocation API

```c
int tool_runtime_dispatch(const runtime_tool_invoke_req_t *req,
                         runtime_tool_invoke_resp_t *resp);

int tool_runtime_get_status(const char *execution_id,
                            tool_execution_status_t *status);

int tool_runtime_fetch_result(const char *execution_id,
                             tool_execution_result_t *result);

int tool_runtime_cancel(const char *execution_id);
```

### 4.3 Internal helper functions

```c
int tool_provider_validate_socket(const char *socket_path);
int tool_provider_validate_auth(const char *socket_path, const char *auth_token);
int tool_provider_validate_metadata(const provider_record_t *provider);
int tool_registry_resolve_tool(const char *tool_name,
                              const char *plane,
                              provider_record_t **provider_out,
                              tool_descriptor_t **tool_out);
int tool_execution_record_create(const char *tool_name,
                                 const char *provider_id,
                                 const char *plane,
                                 const char *transaction_uuid,
                                 char execution_id_out[128]);
int tool_execution_record_update(const char *execution_id,
                                 enum exec_state state,
                                 int exit_code,
                                 const char *progress,
                                 const char *result_json);
```

## 5. Provider registration process

### 5.1 Register flow

1. receive registration request from trusted local process
2. validate request envelope fields and payload size
3. validate socket path and ownership
4. validate auth token and provider identity
5. validate tool descriptors and plane values
6. check duplicate names under current conflict policy
7. move provider to `ACTIVE` and store it in registry
8. return registration result to caller

### 5.2 Deregister flow

1. resolve provider by `provider_id`
2. mark provider `DRAINING`
3. wait for in-flight invocations to complete or timeout
4. remove registry entries when safe
5. transition to `UNLOADED`

## 6. Dispatch and invocation path

### 6.1 Request normalization

The dispatcher must normalize all requests into the canonical broker contract:

- tool name
- plane
- source
- transaction UUID
- timeout
- async flag
- payload buffer + length

### 6.2 Resolution and validation steps

1. look up tool name in registry by plane
2. ensure provider is `ACTIVE` or degraded but recoverable
3. validate source and auth scope
4. check if tool exists on provider
5. reject stale or disabled provider records
6. serialize request to provider-specific message envelope

### 6.3 Provider transport handling

The runtime chooses transport according to the configured provider record and supported provider modes:

- Unix domain socket: preferred path for persistent providers
- pipe transport: supported for supervisor-managed subprocesses and spawn-per-request execution

Transport-specific responsibilities:

- `tool_ipc_socket_connect()` for Unix socket connection management
- `tool_ipc_pipe_open()` for pipe-backed provider launch or relay
- `tool_ipc_send_message()` for framed JSON writes
- `tool_ipc_recv_message()` for response parsing and timeout handling

### 6.4 Provider-side protocol contract

The provider transport always uses the same JSON envelope semantics regardless of transport type. The daemon is the client and the provider is the responder. Every command is a JSON object containing the envelope metadata (`version`, `type`, `request_id`, `timestamp_ms`, optional `source`, `plane`, `tool`, and `payload`), and every response must echo the originating `request_id`.

Required provider commands:

- `register`: publishes tool metadata and validates the provider record
- `list-tools`: returns the available tools for the provider
- `invoke`: starts a sync or async execution and returns `started` or `completed`
- `status`: returns the current progress or final state for a known `execution_id`
- `cancel`: terminates or marks a running execution as canceled

Required provider response types:

- `ack`: one-shot acknowledgement for registration or command acceptance
- `result`: final or intermediate execution result payload
- `error`: deterministic error with a normalized error code and message

This contract keeps the broker logic independent from the provider transport while ensuring identical message validation, request correlation, and async-state management across socket-based and pipe-based provider execution.

## 7. State machines

### 7.1 Provider state machine

```text
REGISTERING -> ACTIVE
ACTIVE -> DRAINING
ACTIVE -> DEGRADED
ACTIVE -> FAILED
DEGRADED -> FAILED
DRAINING -> UNLOADED
FAILED -> REGISTERING
```

### 7.2 Execution state machine

```text
PENDING -> RUNNING
PENDING -> CANCELED
RUNNING -> COMPLETED
RUNNING -> FAILED
RUNNING -> CANCELED
```

### 7.3 Error mapping

Each failure is normalized into a stable error code before returning to the caller. Implementation mapping should be centralized in the dispatcher to ensure one error classification path for all providers.

## 8. JSON envelope validation

The runtime must validate every IPC exchange before parsing or acting on a payload. The concluded protocol is JSON-first at the envelope boundary: `payload` and `result` are structured JSON objects or arrays, while opaque binary content is carried in an optional `binary` field with explicit metadata and base64 encoding. Raw top-level binary is not accepted in the baseline contract.

### Required request fields

- `version`: integer, required, must be `1`
- `type`: string, required, allowed commands are `register`, `list-tools`, `invoke`, `status`, `cancel`, `ack`, `result`, `error`
- `request_id`: string, required, length 1..128, pattern `[A-Za-z0-9._:-]+`
- `timestamp_ms`: integer, required, positive and within valid range
- `source`: string, required, non-empty
- `plane`: required for `invoke`, `status`, and `cancel`
- `tool`: required for `invoke` and `cancel`
- `payload`: object for invoke requests, optional otherwise
- `auth_token`: required for `register`, optional for other commands when needed

### Required response fields

- `version`
- `type`
- `request_id`
- `timestamp_ms`
- `status`
- `result` or `error`
- `binary` when an opaque payload is explicitly being returned
- `execution_id` when async result is returned

### Validation behavior

- reject unknown fields unless explicitly permitted
- enforce field length limits before message acceptance
- reject payload sizes above configured limit
- reject partial or malformed envelopes with immediate `error` payloads
- reject top-level raw binary results; require a typed `binary` field instead
- quarantine provider when repeated validation failures happen for same socket or auth token

## 9. Security and auth design

### 9.1 Socket trust checks

The runtime validates socket ownership and mode before accepting a provider. For Unix sockets, this includes:

- path existence and socket type check
- parent directory ownership check
- socket ownership match against trusted runtime user/group
- optional permission and path confinement checks

### 9.2 Auth token validation

The registration flow and provider requests may carry a token or signed registration payload. The runtime validates it before binding the provider record to the registry.

### 9.3 Policy guard role

The policy guard is responsible for:

- validating plane name against registry and request context
- rejecting unauthorized provider registration
- enforcing conflict policy before active registration
- ensuring asynchronous execution status is only returned to authorized callers

## 10. Concurrency and locking

### 10.1 Locking strategy

The runtime uses short-lived lock sections for hot-path operation while keeping provider registration and reload work in a serialized commit phase.

Recommended locks:

- registry lock for provider table mutation
- execution lock for execution record insert/update
- policy lock for auth and plane validation state
- socket state lock for transport health and connection management

### 10.2 Safety rules

- do not mutate registry contents without committing under lock
- preserve execution history while updating execution state
- drain provider on shutdown before removing active provider record
- avoid retries on repeated invalid registration while provider remains quarantined

## 11. Failure handling

### 11.1 Provider failure handling

- socket disappears or process exits: mark provider `FAILED`
- auth mismatch: quarantine provider and return `ERR_PROVIDER_AUTH_FAILED`
- invalid metadata: reject registration and return `ERR_PROVIDER_INVALID`
- stale provider record: return `ERR_PROVIDER_NOT_FOUND`

### 11.2 Execution failure handling

- timeout: `ERR_EXECUTION_TIMEOUT`
- cancellation: `ERR_EXECUTION_CANCELED`
- host dispatch error: `ERR_REMOTE_DISPATCH_FAILED`
- tool not found: `ERR_TOOL_NOT_FOUND`

### 11.3 Retry policy

Retry logic should be bounded and explicit. The runtime must not loop forever on known invalid provider state. It should re-check provider health, then either quarantine or mark the request failed.

## 12. Implementation notes

### 12.1 Native plugin adapter

- use `dlopen()` and resolve required symbols
- validate `diag_plugin_get_api_version()` return value
- enumerates tool metadata through `diag_plugin_get_tool_count()` and `diag_plugin_get_tool()`
- route invoke through `diag_plugin_invoke()`

### 12.2 Script plugin adapter

- run script with `--list-tools`
- parse JSON array of tool descriptors
- run script with `--invoke {tool} {payload-json}`
- accept only JSON-based results and normalize to runtime response schema

### 12.3 External provider adapter

- maintain a local provider registry keyed by `provider_id`
- use Unix SOCK_STREAM for local broker communication
- include request correlation using `request_id` and `execution_id`
- maintain execution state in-memory and persist only if required by runtime policy

## 13. Example call flow

```text
cloud caller
    -> daemon
    -> registry resolve tool name and plane
    -> policy validation
    -> provider transport send
    -> provider executes
    -> daemon normalizes response
    -> caller receives final response or execution handle
```

For async work:

```text
cloud caller
    -> daemon
    -> create execution_id
    -> dispatch provider request
    -> provider returns started status
    -> daemon stores execution record
    -> caller polls get_status / fetch_result
    -> daemon updates final state and result
```

## 14. Testing strategy

The implementation should include tests for:

- provider registration success and failure
- duplicate tool conflict behavior
- plane mismatches and policy rejection
- socket auth validation failure
- request schema validation failure
- async status transitions and polling
- sync result normalization and error normalization
- provider unload and drain behavior

## 15. Traceability to implementation artifacts

This LLD maps directly to the C reference header and runtime functions described in the spec. The implementation should preserve these responsibilities:

- `tool_registry_register_external()` for provider registration
- `tool_registry_unregister_external()` for provider teardown
- `tool_dispatch_invoke()` for brokered dispatch
- `tool_execution_record_create()` and `tool_execution_record_update()` for runtime tracking
- `tool_provider_validate_socket()` and `tool_provider_validate_auth()` for security enforcement

## 16. Acceptance criteria

The implementation is considered complete when:

- providers register through a single runtime registry interface
- all tool names resolve by plane and provider metadata
- sync and async invocation semantics are supported
- execution status and final results are available through a stable API
- invalid registration or execution attempts are normalized into deterministic errors
- the runtime prefers Unix domain sockets with a strict security posture
