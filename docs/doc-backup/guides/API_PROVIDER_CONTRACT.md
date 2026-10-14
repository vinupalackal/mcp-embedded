# API-Style External Provider Contract

## 1. Purpose

This document defines the provider-side contract for an API-style external provider. This is the dynamic provider model in which a trusted external process or service registers with the runtime and then participates in a brokered local IPC contract using a JSON envelope protocol.

This style covers the case where the provider is not a native `.so` and not a script plugin; it is a stand-alone process that exposes one or more tools through a local API contract.

## 2. Provider type

- Type: `external-process`
- Delivery format: local executable or daemon process
- Transport: Unix domain socket or stdin/stdout pipe
- Broker model: runtime is the only caller-facing dispatcher

## 3. Trust and transport model

The provider must never be reached directly by remote callers. The runtime acts as the broker and always initiates requests.

Supported transport modes:

1. `stdin/stdout` pipe for spawn-per-request execution
2. Unix domain socket for long-lived provider processes

The runtime may prefer sockets for persistent providers and pipes for simpler one-shot providers, but the broker-visible contract remains the same.

## 4. Required command contract

A compliant provider must accept these commands in JSON form:

- `register`
- `list-tools`
- `invoke`
- `status`
- `cancel`
- `shutdown` or equivalent termination flow

The runtime must send a JSON request envelope; the provider must respond using the same envelope structure.

## 5. Required JSON envelope

Every request/response must use a JSON envelope.

### 5.1 Request envelope example

```json
{
  "version": 1,
  "type": "invoke",
  "request_id": "a1b2c3",
  "timestamp_ms": 1728319199000,
  "source": "cloud",
  "plane": "triage",
  "tool": "diag_capture",
  "payload": {"interface": "brlan0"},
  "async": true
}
```

### 5.2 Response envelope example

```json
{
  "version": 1,
  "type": "ack",
  "request_id": "a1b2c3",
  "timestamp_ms": 1728319200000,
  "status": "started",
  "execution_id": "d63f5e5d-9a4b-4c3d-bc87-e4f2d516acaa",
  "result": null
}
```

Required response types:

- `ack`
- `result`
- `error`

## 6. Provider handshake and registration

The provider must perform a registration or handshake phase before it becomes active.

Required registration metadata:

- `provider_id`
- `provider_name`
- `plane`
- `process_path` or daemon process identity
- `socket_path` when using a Unix socket
- `auth_token` if the deployment requires token validation
- tool list metadata for the provider

The daemon validates:

- socket existence and type
- socket ownership and trust boundary
- auth material or local ownership policy
- plane scoping
- duplicate provider or tool conflicts

Only after successful validation may the provider move from `REGISTERING` to `ACTIVE`.

## 7. Provider lifecycle states

A provider in this model must honor the lifecycle states below:

- `REGISTERING`
- `ACTIVE`
- `DRAINING`
- `DEGRADED`
- `FAILED`
- `UNLOADED`

Transitions:

- `REGISTERING -> ACTIVE` after validation succeeds
- `ACTIVE -> DRAINING` during restart or shutdown
- `DRAINING -> UNLOADED` once in-flight work drains or times out
- `ACTIVE -> DEGRADED` after repeated soft failures
- `DEGRADED -> FAILED` after repeated transport or validation failures
- `ACTIVE -> FAILED` if the process exits unexpectedly or the socket disappears

## 8. Execution lifecycle

For async or long-running tools, the provider must use a stable execution record keyed by `execution_id`.

Execution states:

- `pending`
- `running`
- `completed`
- `failed`
- `canceled`

The provider must return:

- `status = started` when work begins
- `status = completed` when final result is available
- `status = failed` when work fails or times out
- `status = canceled` when the runtime cancels the work

## 9. Required response semantics

### 9.1 `list-tools`

The provider must return a valid list of tool descriptors.

Example:

```json
{
  "version": 1,
  "type": "result",
  "request_id": "req-42",
  "timestamp_ms": 1728319200000,
  "status": "ok",
  "result": {
    "tools": [
      {
        "name": "diag_capture",
        "description": "Starts packet capture",
        "plane": "triage",
        "provider_type": "external-process",
        "invoke_mode": "async",
        "supports_async": true,
        "timeout_ms": 60000
      }
    ]
  }
}
```

### 9.2 `invoke`

The provider must respond with an `ack` on acceptance and a later `result` when final output is ready.

Example:

```json
{
  "version": 1,
  "type": "result",
  "request_id": "req-43",
  "timestamp_ms": 1728319210000,
  "status": "completed",
  "execution_id": "d63f5e5d-9a4b-4c3d-bc87-e4f2d516acaa",
  "result": {"capture_file": "/tmp/lan_capture.pcap"}
}
```

### 9.3 `cancel`

The provider must either:

- stop the active execution and return `status = canceled`, or
- admit that cancellation is unsupported and return a deterministic error

## 10. Validation rules

The runtime must reject or quarantine a provider if:

- the socket path is invalid or missing
- the socket owner or permissions violate local trust policy
- the provider returns invalid JSON for a request or a response
- the tool metadata is malformed or conflicts with the registry
- the provider reuses a stale execution ID
- auth validation fails

## 11. Error model

External providers must use deterministic runtime error codes, including:

- `ERR_PROVIDER_NOT_FOUND`
- `ERR_TOOL_NOT_FOUND`
- `ERR_PROVIDER_UNAVAILABLE`
- `ERR_PROVIDER_INVALID`
- `ERR_PROVIDER_AUTH_FAILED`
- `ERR_EXECUTION_TIMEOUT`
- `ERR_EXECUTION_CANCELED`
- `ERR_REMOTE_DISPATCH_FAILED`

## 12. Binary payload handling

Binary or opaque data must not be passed as a raw top-level result. If binary output is required, it must be wrapped in a typed `binary` field, such as:

```json
{
  "version": 1,
  "type": "result",
  "request_id": "req-44",
  "timestamp_ms": 1728319220000,
  "status": "completed",
  "result": {"artifact": "capture.pcap"},
  "binary": {
    "content_type": "application/octet-stream",
    "encoding": "base64",
    "data": "AAECAwQFBgcICQ=="
  }
}
```

## 13. Reference implementation behavior

A compliant API-style provider must:

1. establish a trusted local connection to the daemon
2. validate provider identity and auth scope
3. advertise tool metadata
4. accept `invoke`, `status`, and `cancel` commands only over the runtime broker protocol
5. return structured JSON results and normalized errors
6. track async execution IDs and liveness
7. terminate cleanly on shutdown or provider restart

## 14. Summary

The API-style external provider contract is the runtime’s most general dynamic provider style. It preserves the brokered runtime model while allowing a trusted local process to be treated as a first-class dynamic external tool provider with explicit lifecycle tracking, auth validation, and structured execution semantics.
