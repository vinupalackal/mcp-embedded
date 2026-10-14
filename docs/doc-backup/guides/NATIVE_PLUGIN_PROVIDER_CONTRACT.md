# Native Plugin Provider Contract

## 1. Purpose

This document defines the provider-side contract for a native Multi-Plane Runtime Manager (MPRM) plugin implemented as a shared library (`.so`). It describes the required ABI, lifecycle, validation, invocation contract, and error behavior for a provider that is loaded directly by the runtime.

The runtime remains the broker and policy authority. The provider must not bypass the registry or invoke host internals outside the allowed plugin contract.

## 2. Provider type

- Type: `native`
- Delivery format: compiled shared object loaded via `dlopen()`
- Discovery: runtime-owned plugin directory or explicit registration path

## 3. Required ABI

A native provider must implement the following functions:

```c
int diag_plugin_get_api_version(void);
int diag_plugin_init(const diag_host_api_t *host, diag_plugin_ctx_t **ctx);
size_t diag_plugin_get_tool_count(diag_plugin_ctx_t *ctx);
int diag_plugin_get_tool(diag_plugin_ctx_t *ctx, size_t index, diag_tool_def_t *out);
int diag_plugin_invoke(diag_plugin_ctx_t *ctx, const diag_invoke_req_t *req, diag_invoke_resp_t *resp);
void diag_plugin_deinit(diag_plugin_ctx_t *ctx);
```

## 4. Contract semantics

### 4.1 API version

- `diag_plugin_get_api_version()` must return the runtime-supported plugin API version.
- Unknown or unsupported versions must be rejected cleanly.
- The version comparison must be performed before tool enumeration or invocation begins.

### 4.2 Initialization

- `diag_plugin_init()` must allocate and initialize provider context.
- It must validate required host callbacks and configuration dependencies.
- It must fail without partially initialized state.

### 4.3 Tool enumeration

- `diag_plugin_get_tool_count()` returns the number of tools exported by this provider.
- `diag_plugin_get_tool()` fills one tool descriptor at a time.
- Every tool descriptor must contain at least:
  - `name`
  - `description`
  - `plane`
  - `provider_type`
  - `invoke_mode`
  - `supports_async`
  - `timeout_ms`

### 4.4 Invocation

- `diag_plugin_invoke()` receives a canonical invoke request.
- It must act only on the tool requested by `req->tool`.
- It must return a canonical response object or error code.
- Any failure must be normalized into a deterministic runtime error before returning to the caller.

### 4.5 Shutdown

- `diag_plugin_deinit()` must release all provider resources.
- It must leave no stale execution handles or lingering state behind.
- Any in-flight async execution must be marked as failed, canceled, or drained according to runtime policy.

## 5. Tool descriptor contract

A tool descriptor must be structurally consistent with the registry contract:

```c
typedef struct {
    char name[128];
    char description[256];
    char plane[32];
    char provider_type[32];
    char invoke_mode[16];
    int supports_async;
    uint32_t timeout_ms;
    uint32_t capabilities_mask;
} tool_descriptor_t;
```

Rules:

- `name` must be unique within the active conflict policy scope.
- `plane` must match a configured runtime plane name.
- `invoke_mode` must be `sync` or `async`.
- `supports_async` must be `true` only when the plugin can track and return execution state.

## 6. Runtime request and response contract

The provider operates on the runtime’s canonical invoke request and response objects.

```c
typedef struct {
    char transaction_uuid[64];
    char source[64];
    char plane[32];
    char tool[128];
    char provider_id[128];
    void *payload;
    size_t payload_len;
    int async_mode;
    uint32_t timeout_ms;
} runtime_tool_invoke_req_t;
```

```c
typedef struct {
    int status;
    int exit_code;
    char execution_id[128];
    char result_json[4096];
    char error[256];
    int is_async;
} runtime_tool_invoke_resp_t;
```

Required semantics:

- `payload` is a canonical serialized JSON buffer or equivalent structured payload
- `result_json` must be valid JSON when `status` is success
- `error` must be populated with a normalized runtime error code when the provider fails
- `execution_id` must be non-empty for async execution and stable for the work item

## 7. Error model

The provider must map failures to deterministic runtime errors:

- `ERR_PROVIDER_NOT_FOUND`
- `ERR_TOOL_NOT_FOUND`
- `ERR_PROVIDER_UNAVAILABLE`
- `ERR_PROVIDER_INVALID`
- `ERR_PROVIDER_AUTH_FAILED`
- `ERR_TOOL_CONFLICT`
- `ERR_EXECUTION_TIMEOUT`
- `ERR_EXECUTION_CANCELED`
- `ERR_REMOTE_DISPATCH_FAILED`

## 8. Security and validation rules

A native plugin must satisfy the same host policy expectations as any other provider:

- no direct use of untrusted network sockets from the plugin
- no bypass of the runtime registry
- no invalid tool names or duplicate names under deny/priority policy
- no raw binary top-level output; binary must be wrapped in an explicit typed payload field if required
- no stale execution handles after shutdown

## 9. Reference implementation behavior

A compliant native plugin must:

1. validate the API version
2. initialize context and dependencies
3. enumerate tool descriptors
4. register each tool in the runtime registry
5. accept invoke requests only through the canonical contract
6. return normalized result/error payloads
7. honor timeout and cancellation logic
8. cleanly deinitialize on unload or failure

## 10. Summary

The `.so` provider contract is the strictest and most direct plugin style. It defines a clear and compact ABI boundary, keeps the runtime as the broker, and requires the provider to behave like a well-defined runtime extension instead of an unguarded host callback.
