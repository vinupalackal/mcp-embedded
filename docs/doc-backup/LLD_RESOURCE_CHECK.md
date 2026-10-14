# Low-Level Design: Resource Availability Check Before Tool Execution

## 1. Scope

This LLD describes the implementation of the resource gate inserted before static command execution and dynamic plugin invocation.

## 2. Key Components

### 2.1 Config Loader

The existing config loader in [src/rdk-service-hub.c](../src/rdk-service-hub.c) shall be extended to read the resource policy values.

Add configuration keys:
- MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_CHECK_ENABLE
- MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_CPU_MAX_PCT
- MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_MEM_FREE_MB
- MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_DISK_FREE_MB
- MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_MAX_PROCS
- MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_FAIL_MODE

### 2.2 Resource Policy Struct

```c
typedef struct {
    int enabled;
    int cpu_max_pct;
    int mem_free_mb;
    int disk_free_mb;
    int max_procs;
    int fail_mode;
} resource_policy_t;
```

### 2.3 Resource Check Helper

Add a helper function near the static validation and command execution logic:

```c
static resource_check_result_t resource_check_tool(
    const char *tool,
    const char *command,
    const char *plane,
    int is_dynamic,
    const resource_policy_t *policy,
    char *reason_out,
    size_t reason_len);
```

Responsibilities:
- read CPU, memory, disk, process metrics
- compare against policy
- return allow / deny / defer / error
- fill a human-readable denial reason

### 2.4 Runtime Hook Points

Insert the resource check before:
- static `run_command()` invocation
- dynamic `plugin_manager_invoke()` call

## 3. Metric Collection

### 3.1 CPU
Use `/proc/loadavg` or a system API to read recent load/usage.

Pseudo logic:
- read current CPU usage percent or load average
- if value > cpu_max_pct -> deny

### 3.2 Memory
Use `sysinfo()` or `/proc/meminfo`.

Pseudo logic:
- compute available memory in MB
- if available < mem_free_mb -> deny

### 3.3 Disk
Use `statvfs()`.

Pseudo logic:
- compute free blocks and bytes
- if free bytes < disk_free_mb * 1024 * 1024 -> deny

### 3.4 Process Count
Use `/proc` scan or counting APIs.

Pseudo logic:
- count processes or threads
- if count > max_procs -> deny

## 4. Output / Print Behavior

The runtime shall print resource information during execution in a simple, operator-readable format.

Suggested print format:

```text
RESOURCE_CHECK tool=plugin_echo plane=triage kind=dynamic cpu=25% mem_avail=640MB disk_free=120MB procs=82 decision=ALLOW
```

For a failure:

```text
RESOURCE_CHECK tool=plugin_echo plane=triage kind=dynamic cpu=92% mem_avail=32MB disk_free=15MB procs=201 decision=DENY reason=memory below threshold
```

This output shall also be logged with syslog so operators can correlate it with tool requests.

## 5. Denial Path

When denied, the runtime shall:
1. stop before command launch or plugin callback
2. return a structured error response
3. log the denial reason
4. include resource values in the logs

## 6. Error Handling

If resource collection fails:
- if fail_mode == deny -> reject the request
- if fail_mode == defer -> return a retry/defer response
- if fail_mode is invalid -> use deny-safe default

## 7. Observability Requirements

Before execution:
- print current resource snapshot

After execution:
- print final resource snapshot and outcome summary

Example:

```text
RESOURCE_START tool=plugin_version plane=triage kind=dynamic
RESOURCE_CHECK ... decision=ALLOW
RESOURCE_END tool=plugin_version exit=0 cpu=28% mem_avail=512MB disk_free=140MB
```

## 8. Integration Points

### 8.1 Static command path
Insert immediately before `run_command()` in the static-tool branch of the main request handler.

### 8.2 Dynamic plugin path
Insert immediately before `plugin_manager_invoke()` in the dynamic plugin dispatch block.

## 9. Validation Strategy

- unit tests for resource thresholds
- integration test for denied execution under low memory
- plugin test for denied plugin invocation under overload
- log assertion for output message formatting

## 10. Risks and Mitigations

### Risk: false denial under transient load
Mitigation: use conservative but not overly strict thresholds and allow a `defer` mode.

### Risk: high overhead
Mitigation: read only a small set of metrics and keep the check lightweight.

### Risk: inconsistent metrics on different OSes
Mitigation: use portable Linux-friendly reads first, with safe fallback behavior.

## 11. Acceptance Criteria

- Resource check runs before tool execution
- Static tool path and dynamic path both enforce the gate
- Denial reasons are clear and logged
- Resource availability/usage is printed for operator visibility
- Config/env override precedence is respected
