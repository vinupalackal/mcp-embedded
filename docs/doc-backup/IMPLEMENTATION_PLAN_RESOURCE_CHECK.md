# Resource Check Implementation Plan

## 1. Goal

Add a lightweight pre-execution resource gate before static commands and dynamic plugin invocation so the runtime refuses to execute a tool when host resources are insufficient.

## 2. Scope

Apply the resource check to both of these execution paths:
- static catalog tool execution
- dynamically loaded plugin invocation

## 3. Design Principles

- fail safe
- low overhead
- config/env override support
- explicit deny reason
- no change to current catalog validation flow other than an added gate

## 4. Proposed Config Keys

The runtime already supports config-file and environment overrides. Add these keys:

- MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_CHECK_ENABLE
- MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_CPU_MAX_PCT
- MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_MEM_FREE_MB
- MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_DISK_FREE_MB
- MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_MAX_PROCS
- MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_FAIL_MODE

Precedence:
1. environment variable
2. config file value
3. built-in default

## 5. Default Values

Recommended safe defaults:

- resource check enable: 0 (off by default for rollout safety)
- CPU max pct: 90
- memory free MB: 64
- disk free MB: 32
- max procs: 200
- fail mode: deny

## 6. Runtime Flow

### 6.1 Static tool path

Before command execution in the static tool flow in [src/rdk-service-hub.c](../src/rdk-service-hub.c), add:

1. Resolve the tool name
2. Determine the effective command
3. Run `resource_check_tool()`
4. If denied, return a resource-denied response immediately
5. Otherwise continue to `run_command()`

### 6.2 Dynamic plugin path

Before plugin invocation in [src/rdk-service-hub.c](../src/rdk-service-hub.c), add:

1. Resolve tool name in tool registry
2. Run `resource_check_tool()`
3. If denied, return a resource-denied response
4. Otherwise call plugin `invoke()`

## 7. Resource Check API

Proposed helper functions:

```c
typedef enum {
    RESOURCE_ALLOW = 0,
    RESOURCE_DENY = 1,
    RESOURCE_DEFER = 2,
    RESOURCE_ERROR = 3
} resource_check_result_t;

typedef struct {
    int enabled;
    int cpu_max_pct;
    int mem_free_mb;
    int disk_free_mb;
    int max_procs;
    int fail_mode;
} resource_policy_t;

static resource_check_result_t resource_check_tool(
    const char *tool,
    const char *command,
    const char *plane,
    int is_dynamic,
    const resource_policy_t *policy,
    char *reason_out,
    size_t reason_len);
```

## 8. Resource Checks

### CPU
- Read current load or CPU usage
- Reject if CPU usage > configured maximum pct

### Memory
- Read current free/available memory
- Reject if available memory < configured minimum MB

### Disk
- Use `statvfs()` or equivalent
- Reject if free space < configured minimum MB

### Process count
- Count running processes or threads
- Reject if count exceeds max_procs

### Timeout and safety
- Ensure the requested timeout remains within legal bounds
- Apply conservative timeout defaults

## 9. Reason Strings

Return a short explicit reason on denial, for example:
- `cpu usage above threshold`
- `memory below threshold`
- `disk free space below threshold`
- `process count above threshold`

## 10. Logging

For every denial or deferral, log:
- tool name
- request source
- plane
- resource type
- threshold value
- timestamp

## 11. Default Policy Behavior

- allow when all checks pass
- deny when thresholds are violated
- use `fail_mode` to choose between deny and defer on transient overload
- if the checker itself fails, default to deny or configured safe behavior

## 12. Acceptance Criteria

1. Static tools do not execute when CPU, memory, or disk thresholds are exceeded.
2. Dynamic plugin tools do not invoke their plugin callback when the resource gate fails.
3. The config file and environment override the runtime settings in the correct order.
4. Denied requests include a clear reason.
5. The runtime remains low-overhead and does not add expensive background scanning.

## 13. Implementation Order

1. Add config keys and default values
2. Add the resource check helper
3. Call it in static tool execution path
4. Call it in dynamic plugin path
5. Add logs and reason strings
6. Validate with unit/integration tests

## 14. Future Enhancements

- per-tool resource policy overrides
- per-plane thresholds
- adaptive throttle / defer policy
- plugin-defined resource hints
- per-source quotas
