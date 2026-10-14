x# Dynamic Provider Comparison Matrix

## 1. Purpose

This document is the canonical reference matrix for the three supported dynamic provider styles in MPRM:

1. native `.so` providers
2. script-based providers
3. API-style external providers

The matrix summarizes the shared runtime contract and the provider-specific implementation details so that a developer can select the right provider model and verify compliance against the same runtime expectations.

## 2. Shared runtime contract

All provider styles must satisfy the same runtime-visible requirements:

- the daemon remains the broker and policy authority
- tool names are resolved through a unified registry
- plane scoping and conflict policy are enforced at registration time
- invocation must be normalized to the canonical runtime contract
- async execution, status polling, and cancellation are supported when the tool is long-running
- provider and execution failures must be converted into deterministic runtime error codes
- raw top-level binary output is not accepted; opaque bytes must be wrapped in an explicit typed `binary` field when required

## 3. Comparison matrix

| Dimension | Native `.so` provider | Script-based provider | API-style external provider |
| --- | --- | --- | --- |
| Provider type | `native` | `script` | `external-process` |
| Delivery format | compiled shared library | executable or script | local process or daemon |
| Discovery model | runtime load or explicit registration | runtime scan of trusted plugin directories | explicit registration or process discovery |
| Runtime binding | `dlopen()` + ABI symbol resolution | CLI invocation via `--list-tools` and `--invoke` | JSON IPC over pipe or Unix domain socket |
| Registration contract | `diag_plugin_init()` and tool enumeration | `--list-tools` JSON metadata | `register` + `list-tools` JSON command |
| Tool metadata source | plugin ABI `diag_plugin_get_tool()` | JSON `tools[]` list | JSON `list-tools` or registration payload |
| Invocation model | direct in-process function call | child process invocation | daemon-brokered JSON IPC |
| Async support | provider-managed execution context | `--status` / `--cancel` optional | `status` / `cancel` required for async tools |
| Result contract | canonical runtime response structure | JSON object with `status`, `tool`, `execution_id`, `result` | JSON envelope with `ack` / `result` / `error` |
| Security boundary | plugin loaded by runtime process | trusted local executable in approved directory | trusted local process with socket or auth validation |
| Preferred use case | high-performance native capabilities | lightweight scripts and operational wrappers | long-running external services or daemons |

## 4. Provider-specific contract summary

### 4.1 Native `.so` provider

Required API:

```c
int diag_plugin_get_api_version(void);
int diag_plugin_init(const diag_host_api_t *host, diag_plugin_ctx_t **ctx);
size_t diag_plugin_get_tool_count(diag_plugin_ctx_t *ctx);
int diag_plugin_get_tool(diag_plugin_ctx_t *ctx, size_t index, diag_tool_def_t *out);
int diag_plugin_invoke(diag_plugin_ctx_t *ctx, const diag_invoke_req_t *req, diag_invoke_resp_t *resp);
void diag_plugin_deinit(diag_plugin_ctx_t *ctx);
```

Key characteristics:

- fastest and most direct integration path
- in-process execution within the daemon boundary
- best when the capability needs low overhead and direct access to runtime data

### 4.2 Script-based provider

Required CLI contract:

```bash
<plugin> --list-tools
<plugin> --invoke <tool_name> <payload-json>
```

Optional async commands:

```bash
<plugin> --status <execution_id>
<plugin> --cancel <execution_id>
```

Key characteristics:

- good for lightweight wrappers around system commands and operational utilities
- lowest implementation cost
- same registry semantics as native plugins, but invoked via process boundary

### 4.3 API-style external provider

Required JSON contract:

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

Key characteristics:

- best for long-lived external services or daemons
- supports runtime-owned IPC over pipe or Unix socket
- most explicit lifecycle, session, auth, and reconnect semantics

## 5. Lifecycle equivalence

Although the transport differs, all three provider styles must honor the same lifecycle model:

- `REGISTERING`
- `ACTIVE`
- `DRAINING`
- `DEGRADED`
- `FAILED`
- `UNLOADED`

Execution lifecycle:

- `PENDING`
- `RUNNING`
- `COMPLETED`
- `FAILED`
- `CANCELED`

The runtime must normalize all provider failures into the same broker-visible behavior regardless of implementation style.

## 6. Operational guidance

### Choose native `.so` when

- the provider is performance-sensitive
- the provider is built and deployed with the runtime
- you need direct in-process access to runtime state or performance-critical operations

### Choose script provider when

- the capability can be expressed as a command or wrapper
- you want rapid onboarding with minimal build effort
- validation and execution isolation are more important than direct in-process speed

### Choose API-style external provider when

- the provider is a long-running service
- you need explicit auth, reconnect, liveness, or async lifecycle management
- the capability is best exposed behind a brokered local IPC contract

## 7. Decision rule

The runtime should prefer the simplest provider type that meets the operational and security requirements:

- use native `.so` for performance-critical embedded extensions
- use script providers for lightweight operational tools
- use API-style external providers for long-lived, managed, or service-oriented capabilities

## 8. Summary

The canonical runtime contract is the same across all dynamic provider styles: the daemon is the broker, the registry is the source of truth, and provider-specific mechanics are an implementation detail. What differs is how the provider presents itself, how it is discovered, and how it communicates with the runtime.
