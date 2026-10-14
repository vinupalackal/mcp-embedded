# Script-Based Plugin Provider Contract

## 1. Purpose

This document defines the provider-side contract for a script-based plugin. A script provider is an executable or script discovered from a runtime-owned plugin directory and registered by the same runtime registry model used for native `.so` providers.

The runtime remains the broker: it is responsible for discovery, validation, registry binding, invocation, and policy enforcement.

## 2. Provider type

- Type: `script`
- Delivery format: executable script, shell, Python, or similar runtime-managed entry point
- Discovery: plugin directory scan
- Registration: metadata discovered via `--list-tools`

## 3. Required runtime interface

A script plugin must support the following CLI calls:

```bash
<plugin> --list-tools
<plugin> --invoke <tool_name> <payload-json>
```

Optional async commands:

```bash
<plugin> --status <execution_id>
<plugin> --cancel <execution_id>
```

## 4. Discovery contract

The runtime invokes the script with `--list-tools` and expects valid JSON describing one or more tools.

Example:

```json
{
  "tools": [
    {
      "name": "system_cpu_usage",
      "description": "Returns current CPU usage snapshot",
      "plane": "management",
      "provider_type": "script",
      "invoke_mode": "sync",
      "supports_async": false,
      "timeout_ms": 15000
    }
  ]
}
```

Rules:

- output must be valid JSON
- the top-level object must include a `tools` array
- every array item must include tool metadata usable by the runtime registry
- invalid JSON or empty output must be treated as failed registration

## 5. Invocation contract

The runtime invokes the script using `--invoke <tool_name> <payload-json>`.

The script must return structured JSON, not raw text output.

Example success response:

```json
{
  "status": "ok",
  "tool": "system_cpu_usage",
  "execution_id": "",
  "result": {"cpu_percent": 23.8}
}
```

Example async response:

```json
{
  "status": "started",
  "tool": "long_running_capture",
  "execution_id": "d63f5e5d-9a4b-4c3d-bc87-e4f2d516acaa",
  "result": null
}
```

Example failure response:

```json
{
  "status": "failed",
  "tool": "system_cpu_usage",
  "execution_id": "",
  "error": {
    "code": "ERR_PROVIDER_UNAVAILABLE",
    "message": "script execution failed"
  }
}
```

## 6. Async lifecycle support

When a script supports async execution, it must implement the following:

- `--invoke` returns `status` = `started` with a non-empty `execution_id`
- `--status <execution_id>` returns current execution state
- `--cancel <execution_id>` either cancels the job or returns a deterministic canceled state

Allowed status values:

- `pending`
- `running`
- `completed`
- `failed`
- `canceled`

## 7. Validation and security requirements

A script plugin must:

- be located in a trusted runtime-owned or approved plugin directory
- return valid UTF-8 JSON only
- reject unsupported tool names without silently succeeding
- not emit raw binary as the top-level response object
- not bypass runtime policy or plane isolation
- return tool metadata that matches the runtime’s canonical descriptor format

## 8. Error handling contract

The script must not emit non-JSON output on successful tool execution. If it fails, it must still emit valid JSON with a deterministic error payload.

Recommended error structure:

```json
{
  "status": "failed",
  "tool": "tool_name",
  "execution_id": "",
  "error": {
    "code": "ERR_TOOL_NOT_FOUND",
    "message": "tool not implemented by this provider"
  }
}
```

## 9. Runtime-enforced policy

The runtime must enforce:

- uniqueness of tool names under active conflict policy
- plane scoping
- ownership and path validation for script candidates
- timeout enforcement for script invocation
- detection of stale async execution records

## 10. Reference behavior summary

A valid script provider must:

1. advertise tool metadata via `--list-tools`
2. accept `--invoke` requests for the advertised tool names
3. return structured JSON results or errors
4. support async tracking when advertised
5. respect runtime policy and cancellation requests
6. remain fail-safe under timeout and runtime restart conditions

## 11. Summary

The script provider contract keeps the runtime model consistent with the native plugin model while replacing the compiled ABI with a command-line JSON contract. This is the lightest-weight dynamic provider style when a trusted script or executable is sufficient for the capability being exposed.
