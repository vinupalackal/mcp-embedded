# Multi-Plane Runtime Manager User Guide

This guide explains the runtime features that are currently supported, where the configuration lives, and how to tune the service for a deployment.

## 1. Overview

Multi-Plane Runtime Manager is a lightweight runtime that routes tool requests to the correct plane, resolves the matching catalog entry or plugin, enforces safety policies, and executes the selected command in a constrained and observable way.

## 1.1 Quick start for operators

For a standard deployment, the service startup sequence is:

1. install the runtime and package config files
2. ensure the plugin state directory exists
3. validate the catalog JSON files for each plane
4. enable or disable plugin loading with the runtime config
5. set any environment overrides needed for the deployment
6. start the binary and confirm the startup feature table is logged

Typical startup flow:

```bash
# default config path
/etc/rdk-service-hub/rdk-service-hub.conf

# optional runtime override examples
export MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_ENABLE=1
export MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DIR=/usr/lib/rdk-service-hub/plugins
export MULTI_PLANE_RUNTIME_MANAGER_TIMEOUT_DEFAULT_SEC=30

/usr/local/bin/rdk-service-hub
```

If the process is launched under systemd or a container, the runtime should create or use the required directories under `/usr/lib/rdk-service-hub` and `/var/lib/rdk-service-hub` as defined by the package installation and runtime config.

The runtime supports four execution planes:

- `triage` – diagnostic and read-only actions
- `management` – operational status and management workflows
- `control` – administrative command paths with stronger restrictions
- `config-apply` – configuration-change actions with the highest caution level

The runtime also supports:

- static tool catalogs in JSON files
- optional dynamic plugin loading from shared libraries
- configurable execution gating for commands and resources
- environment-variable overrides over the default config file
- optional logging to a file and startup feature table output
- push-policy enforcement for catalog updates

### 1.2 Startup defaults and runtime overrides

The current runtime is intentionally conservative by default and continues running even when no CLI arguments are supplied. The default catalog backend is `json`, LMDB remains an optional compile-time capability, and the runtime uses a compact tmpfs-backed catalog index to keep memory use low without requiring a database dependency at runtime.

The startup arguments that are currently supported include:

- `--catalog-dir <path>`
- `--catalog-backend <json|lmdb>`
- `--load-all-triage-config <0|1>`
- `--resource-usage-display <0|1>`
- command policy/resource flags that can be overridden at process startup when explicitly enabled

Example startup:

```bash
/usr/local/bin/rdk-service-hub \
  --catalog-dir /etc/rdk-service-hub \
  --catalog-backend json \
  --load-all-triage-config 0 \
  --resource-usage-display 0
```

If LMDB support is not compiled into the binary, the runtime automatically falls back to JSON mode instead of failing startup. When `--resource-usage-display 1` is enabled, a single `PROCESS_RESOURCE_USAGE` snapshot is printed once at startup or reload, and it is not emitted repeatedly unless the flag is explicitly set again.

A configuration reload can be triggered with `SIGHUP`, which re-reads the runtime values and applies the effective configuration without restarting the process.

---

## 2. Features currently supported

### 2.1 Plane-aware tool routing

The service resolves requests against a plane-specific catalog or plugin registry. A request can be scoped to a plane so the runtime can choose the correct tool set and policy class.

Supported plane classes:

| Plane | Default policy class | Typical use |
|---|---|---|
| `triage` | `diagnostic` | status, health, diagnostics |
| `management` | `diagnostic` | management and monitoring actions |
| `control` | `admin_unsafe` | operational control actions |
| `config-apply` | `admin_unsafe` | configuration mutation actions |

### 2.2 Static catalog execution

The runtime loads catalog definitions from JSON files. This is the default tool resolution path for packaged and baked-in tool sets.

Default catalog directory:

- `/etc/rdk-service-hub`

Default catalog files:

- `rdk-service-hub-triage-catalog.json`
- `rdk-service-hub-management-catalog.json`
- `rdk-service-hub-control-catalog.json`
- `rdk-service-hub-config-apply-catalog.json`

The values can be overridden with the following environment variables:

- `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_DIR`
- `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_FILE_TRIAGE`
- `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_FILE_MANAGEMENT`
- `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_FILE_CONTROL`
- `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_FILE_CONFIG_APPLY`

### 2.3 Dynamic plugin loading

Dynamic plugins are optional and are loaded through shared objects (`.so`) so the runtime can expand its tool surface without recompiling the main binary.

Supported plugin controls:

- `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_ENABLE`
- `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DIR`
- `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DIR_TRIAGE`
- `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DIR_MANAGEMENT`
- `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DIR_CONTROL`
- `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DIR_CONFIG_APPLY`
- `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DISCOVERY_MODE`
- `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_POLL_INTERVAL_SEC`
- `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_CONFLICT_POLICY`
- `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_VERIFY_MODE`

The runtime stores active plugin metadata in a manifest file, which is useful for persisting the list of loaded tool names and plugin-path state instead of relying only on RAM.

Default plugin manifest path:

- `/var/lib/rdk-service-hub/plugin_manifest.json`

This can be overridden with:

- `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_MANIFEST_PATH`

### 2.4 Command allowlist and denylist enforcement

Before a command is launched, the runtime uses a policy file to validate the executable and arguments. This prevents unauthorized or destructive commands from being executed.

Configuration keys:

- `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_ALLOWLIST_FILE`
- `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_ALLOWLIST_DIAGNOSTIC_FILE`
- `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_DENYLIST_ADMIN_UNSAFE_FILE`
- `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_POLICY_DEFAULT`
- `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_POLICY_PLANE_TRIAGE`
- `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_POLICY_PLANE_MANAGEMENT`
- `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_POLICY_PLANE_CONTROL`
- `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_POLICY_PLANE_CONFIG_APPLY`

The runtime can select different policy files for different planes. The default split is:

- `triage` and `management` use the diagnostic-safe policy file
- `control` and `config-apply` use the admin-unsafe denylist path

### 2.5 Resource gate checks before execution

The runtime can perform a resource gate before a tool runs. This protects the host from running commands when CPU, memory, disk, or process limits are already exhausted.

Supported knobs:

- `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_CHECK_ENABLE`
- `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_CPU_MAX_PCT`
- `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_MEM_FREE_MB`
- `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_DISK_FREE_MB`
- `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_MAX_PROCS`
- `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_FAIL_MODE`

Behavior:

- when enabled, the runtime checks the host state before accepting the tool execution request
- the request can be denied, logged, and returned with a clear failure reason if the resource policy fails

### 2.6 Timeout and loop protection

Commands are executed with a timeout and output guard rails so they cannot sit indefinitely or produce unbounded output.

Supported settings:

- `MULTI_PLANE_RUNTIME_MANAGER_TIMEOUT_DEFAULT_SEC`
- `MULTI_PLANE_RUNTIME_MANAGER_TIMEOUT_EXTRA_MARGIN_SEC`
- `MULTI_PLANE_RUNTIME_MANAGER_REQUEST_THREAD_STACK_BYTES`

The runtime can:

- stop a tool when the time budget is exceeded
- reject commands that are clearly looping or producing excessive output
- return a clear status such as timeout or loop-detected at the API layer

### 2.7 Persistent runtime state and plugin manifest

To avoid keeping only tool names in memory, the runtime persists plugin metadata to a file-based manifest. This is especially useful when dynamic tools are loaded and unloaded over time.

The main state path is:

- `/var/lib/rdk-service-hub`

The plugin manifest path is:

- `/var/lib/rdk-service-hub/plugin_manifest.json`

### 2.8 Logging and feature visibility

The runtime logs startup feature state and details about supported security, plugin, and execution features. This helps operator validation without reading the full source code.

Relevant settings:

- `MULTI_PLANE_RUNTIME_MANAGER_LOG_ENABLE`
- `MULTI_PLANE_RUNTIME_MANAGER_LOG_FILE_ENABLE`
- `MULTI_PLANE_RUNTIME_MANAGER_LOG_FILE_PATH`
- `MULTI_PLANE_RUNTIME_MANAGER_LOG_TRACE_API`
- `MULTI_PLANE_RUNTIME_MANAGER_LOG_TRACE_DATA_IO`

### 2.9 Push policy and payload guardrails

The runtime includes a push-control policy layer to protect catalog updates.

Relevant controls:

- `MULTI_PLANE_RUNTIME_MANAGER_PUSH_REQUIRE_LOCAL_ONLY`
- `MULTI_PLANE_RUNTIME_MANAGER_PUSH_AUTH_TOKEN`
- `MULTI_PLANE_RUNTIME_MANAGER_PUSH_MAX_PAYLOAD_BYTES`
- `MULTI_PLANE_RUNTIME_MANAGER_PUSH_MIN_INTERVAL_MS`
- `MULTI_PLANE_RUNTIME_MANAGER_PUSH_RATE_LIMIT_MODE`
- `MULTI_PLANE_RUNTIME_MANAGER_PUSH_PAYLOAD_LIMIT_MODE`

This allows deployments to enforce local-only updates, payload limits, request spacing, and optional shared tokens.

---

## 3. Default configuration file

The default runtime configuration is in:

- [rdk-service-hub.conf](../../rdk-service-hub.conf)

This file is the canonical default template. Operators can modify it for a deployment, but environment variables take precedence when they are set.

### Config precedence

The precedence model is:

1. Environment variables
2. config file values
3. built-in fallback defaults

This is important for deployment-specific overrides such as plugin directories, resource thresholds, and policy file locations.

### Example override pattern

```bash
export MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DIR=/opt/mprm/plugins
export MULTI_PLANE_RUNTIME_MANAGER_COMMAND_POLICY_PLANE_CONTROL=admin_unsafe
export MULTI_PLANE_RUNTIME_MANAGER_TIMEOUT_DEFAULT_SEC=45
```

These values override what is defined in the config file at runtime.

---

## 4. Runtime directories and installation expectations

The current install model expects the runtime to create or use runtime state directories as part of the package lifecycle.

Common paths:

- binary directory: `/usr/local/bin` or installation prefix
- config directory: `/etc/rdk-service-hub`
- plugin directory: `/usr/lib/rdk-service-hub/plugins`
- runtime state: `/var/lib/rdk-service-hub`
- log file: `/tmp/logs/rdk-service-hub.log` (default in config)

Per-plane plugin folders are also supported:

- `/usr/lib/rdk-service-hub/plugins/triage`
- `/usr/lib/rdk-service-hub/plugins/management`
- `/usr/lib/rdk-service-hub/plugins/control`
- `/usr/lib/rdk-service-hub/plugins/config-apply`

---

## 5. Supported request flows

The service supports the main request types used by the runtime and cloud-facing flow:

- `EXEC` – executes a tool and returns execution status
- `DESCRIBE` – describes the available tools and metadata
- `HEALTH` – reports process health and readiness
- `PUSH` – updates catalog or config state under policy checks

For deeper request semantics and payload details, see:

- [CLOUD_SIDE_USER_GUIDE.md](CLOUD_SIDE_USER_GUIDE.md)
- [API_WORKFLOW_FUNCTIONALITY_GUIDE.md](API_WORKFLOW_FUNCTIONALITY_GUIDE.md)

---

## 6. Example runtime configuration summary

This is the practical summary of the main active options:

| Setting | Purpose | Default example |
|---|---|---|
| `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_DIR` | base catalog directory | `/etc/rdk-service-hub` |
| `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_ENABLE` | enable dynamic plugins | `0` |
| `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DIR` | fallback plugin directory | `/usr/lib/rdk-service-hub/plugins` |
| `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_MANIFEST_PATH` | plugin state manifest | `/var/lib/rdk-service-hub/plugin_manifest.json` |
| `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_ALLOWLIST_FILE` | base allowlist file | `/etc/rdk-service-hub/command_allowlist.json` |
| `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_ALLOWLIST_DIAGNOSTIC_FILE` | safe tool file | `/etc/rdk-service-hub/command_allowlist_diagnostic.json` |
| `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_DENYLIST_ADMIN_UNSAFE_FILE` | unsafe command deny file | `/etc/rdk-service-hub/command_denylist_admin_unsafe.json` |
| `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_CHECK_ENABLE` | enable pre-check gate | `0` |
| `MULTI_PLANE_RUNTIME_MANAGER_TIMEOUT_DEFAULT_SEC` | normal execution timeout | `30` |
| `MULTI_PLANE_RUNTIME_MANAGER_TIMEOUT_EXTRA_MARGIN_SEC` | extra safety margin | `60` |
| `MULTI_PLANE_RUNTIME_MANAGER_LOG_FILE_PATH` | log sink | `/tmp/logs/rdk-service-hub.log` |
| `MULTI_PLANE_RUNTIME_MANAGER_PUSH_REQUIRE_LOCAL_ONLY` | local-only push gate | `1` |

---

## 7. Operational guidance

### 7.1 Before enabling plugins

- ensure the plugin directory exists and is writable by the service account
- confirm that the plugin code matches the runtime ABI expected by the service
- choose a conflict policy for duplicate tool names

### 7.2 Before allowing `config-apply` or `control`

- enable explicit command allowlist and denylist validation
- review destructive operations such as shutdown, package installation, or system management commands
- set strict timeout and output limits

### 7.3 Before production rollout

- validate the catalog JSON files for each plane
- verify plugin and runtime permissions
- confirm log file and state directory paths are writable
- test timeout, resource denial, and allowlist rejection scenarios
- verify that env overrides match deployment needs

---

## 8. Troubleshooting

### Tool not found

Check:

- plane name in the request
- matching catalog file or plugin filename
- whether the tool is intentionally excluded by policy

### Plugin not loaded

Check:

- `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_ENABLE`
- plugin directory and file permissions
- library format and symbol compatibility
- plugin conflict policy

### Command rejected by policy

Check:

- plane policy class
- allowlist file path
- denylist file contents
- whether the executable or argument list is not permitted

### Request times out or loops

Check:

- `MULTI_PLANE_RUNTIME_MANAGER_TIMEOUT_DEFAULT_SEC`
- `MULTI_PLANE_RUNTIME_MANAGER_TIMEOUT_EXTRA_MARGIN_SEC`
- output size and execution constraints
- repeated execution patterns versus tool design

### Resource denied

Check:

- `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_CHECK_ENABLE`
- CPU, memory, disk, and process thresholds
- whether the host is under load while the tool is requested

---

## 9. Configuration reference appendix

This appendix summarizes the key runtime knobs and what they do.

### 9.1 Catalog and runtime paths

| Variable | Purpose | Example |
|---|---|---|
| `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_DIR` | base directory for plane catalogs | `/etc/rdk-service-hub` |
| `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_FILE_TRIAGE` | triage catalog file | `rdk-service-hub-triage-catalog.json` |
| `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_FILE_MANAGEMENT` | management catalog file | `rdk-service-hub-management-catalog.json` |
| `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_FILE_CONTROL` | control catalog file | `rdk-service-hub-control-catalog.json` |
| `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_FILE_CONFIG_APPLY` | config-apply catalog file | `rdk-service-hub-config-apply-catalog.json` |
| `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_MANIFEST_PATH` | state manifest for loaded plugin metadata | `/var/lib/rdk-service-hub/plugin_manifest.json` |

### 9.2 Plugin settings

| Variable | Purpose | Example |
|---|---|---|
| `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_ENABLE` | enable or disable plugin loader | `0` |
| `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DIR` | fallback plugin directory | `/usr/lib/rdk-service-hub/plugins` |
| `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DIR_TRIAGE` | triage plugin directory | `/usr/lib/rdk-service-hub/plugins/triage` |
| `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DIR_MANAGEMENT` | management plugin directory | `/usr/lib/rdk-service-hub/plugins/management` |
| `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DIR_CONTROL` | control plugin directory | `/usr/lib/rdk-service-hub/plugins/control` |
| `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DIR_CONFIG_APPLY` | config-apply plugin directory | `/usr/lib/rdk-service-hub/plugins/config-apply` |
| `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DISCOVERY_MODE` | discovery strategy | `hybrid` |
| `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_POLL_INTERVAL_SEC` | plugin reconcile interval | `60` |
| `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_DEBOUNCE_MS` | debounce time to avoid partial .so loads | `200` |
| `MULTI_PLANE_RUNTIME_MANAGER_PLUGIN_CONFLICT_POLICY` | duplicate tool handling | `reject-plugin-tool` |

### 9.3 Command policy settings

| Variable | Purpose | Example |
|---|---|---|
| `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_ALLOWLIST_FILE` | default command allowlist | `/etc/rdk-service-hub/command_allowlist.json` |
| `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_ALLOWLIST_DIAGNOSTIC_FILE` | diagnostic-safe allowlist | `/etc/rdk-service-hub/command_allowlist_diagnostic.json` |
| `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_DENYLIST_ADMIN_UNSAFE_FILE` | admin-unsafe denylist | `/etc/rdk-service-hub/command_denylist_admin_unsafe.json` |
| `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_POLICY_DEFAULT` | default policy class | `diagnostic` |
| `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_POLICY_PLANE_TRIAGE` | triage policy class | `diagnostic` |
| `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_POLICY_PLANE_MANAGEMENT` | management policy class | `diagnostic` |
| `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_POLICY_PLANE_CONTROL` | control policy class | `admin_unsafe` |
| `MULTI_PLANE_RUNTIME_MANAGER_COMMAND_POLICY_PLANE_CONFIG_APPLY` | config-apply policy class | `admin_unsafe` |

### 9.4 Resource and timeout settings

| Variable | Purpose | Example |
|---|---|---|
| `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_CHECK_ENABLE` | enable pre-execution resource gate | `0` |
| `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_CPU_MAX_PCT` | max CPU utilization allowed | `90` |
| `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_MEM_FREE_MB` | minimum free memory in MB | `64` |
| `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_DISK_FREE_MB` | minimum free disk space in MB | `32` |
| `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_MAX_PROCS` | maximum process threshold | `200` |
| `MULTI_PLANE_RUNTIME_MANAGER_RESOURCE_FAIL_MODE` | action when resource gate fails | `deny` |
| `MULTI_PLANE_RUNTIME_MANAGER_TIMEOUT_DEFAULT_SEC` | default command timeout | `30` |
| `MULTI_PLANE_RUNTIME_MANAGER_TIMEOUT_EXTRA_MARGIN_SEC` | additional safety window | `60` |
| `MULTI_PLANE_RUNTIME_MANAGER_REQUEST_THREAD_STACK_BYTES` | worker stack size in bytes | `262144` |

### 9.5 Logging and push controls

| Variable | Purpose | Example |
|---|---|---|
| `MULTI_PLANE_RUNTIME_MANAGER_LOG_ENABLE` | global logging switch | `1` |
| `MULTI_PLANE_RUNTIME_MANAGER_LOG_FILE_ENABLE` | file sink switch | `1` |
| `MULTI_PLANE_RUNTIME_MANAGER_LOG_FILE_PATH` | log file path | `/tmp/logs/rdk-service-hub.log` |
| `MULTI_PLANE_RUNTIME_MANAGER_LOG_TRACE_API` | API tracing | `1` |
| `MULTI_PLANE_RUNTIME_MANAGER_LOG_TRACE_DATA_IO` | I/O tracing | `1` |
| `MULTI_PLANE_RUNTIME_MANAGER_PUSH_REQUIRE_LOCAL_ONLY` | reject non-local pushes | `1` |
| `MULTI_PLANE_RUNTIME_MANAGER_PUSH_AUTH_TOKEN` | optional auth requirement | `<secret>` |
| `MULTI_PLANE_RUNTIME_MANAGER_PUSH_MAX_PAYLOAD_BYTES` | maximum push payload size | `262144` |
| `MULTI_PLANE_RUNTIME_MANAGER_PUSH_MIN_INTERVAL_MS` | minimum inter-push delay | `250` |
| `MULTI_PLANE_RUNTIME_MANAGER_PUSH_RATE_LIMIT_MODE` | rate-limit guard mode | `enforce` |
| `MULTI_PLANE_RUNTIME_MANAGER_PUSH_PAYLOAD_LIMIT_MODE` | payload-size guard mode | `enforce` |

## 10. Related documents

- [README.md](../../README.md)
- [docs/ARCHITECTURE.md](../ARCHITECTURE.md)
- [docs/HLD.md](../HLD.md)
- [docs/LLD.md](../LLD.md)
- [docs/guides/PLANES_AND_USE_CASES.md](PLANES_AND_USE_CASES.md)
- [plugins/README.md](../../plugins/README.md)
- [docs/guides/PLUGIN_DEVELOPER_UPDATE_GUIDE.md](PLUGIN_DEVELOPER_UPDATE_GUIDE.md)

This guide covers the current runtime feature set and deployment knobs. For implementation details or protocol-level behavior, use the architecture and API guides in this repository.
