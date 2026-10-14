# Multi-Plane Runtime Manager: Runtime Options Reference

This document summarizes the runtime startup options and shows example usage for common deployment patterns.

## Optional flags

```text
--catalog-dir <path>
--catalog-backend <json|lmdb>
--load-all-triage-config <0|1>
--command-policy-default <off|diagnostic|admin_unsafe>
--command-policy-plane-triage <off|diagnostic|admin_unsafe>
--command-policy-plane-management <off|diagnostic|admin_unsafe>
--command-policy-plane-control <off|diagnostic|admin_unsafe>
--command-policy-plane-config-apply <off|diagnostic|admin_unsafe>
--command-allowlist-file <path>
--command-allowlist-diagnostic-file <path>
--command-denylist-admin-unsafe-file <path>
--resource-check-enable <0|1>
--resource-cpu-max-pct <pct>
--resource-mem-free-mb <mb>
--resource-disk-free-mb <mb>
--resource-max-procs <count>
--resource-fail-mode <deny|allow>
--resource-usage-display <0|1>
--timeout-default-sec <seconds>
--timeout-extra-margin-sec <seconds>
--help
```

## Option explanations

### Catalog options

#### `--catalog-dir <path>`
Sets the directory from which runtime catalogs are loaded.

Example:

```bash
./build/rdk-service-hub --catalog-dir /etc/rdk-service-hub/catalogs
```

#### `--catalog-backend <json|lmdb>`
Selects the catalog backend at process start.

Examples:

```bash
./build/rdk-service-hub --catalog-backend json
./build/rdk-service-hub --catalog-backend lmdb
```

#### `--load-all-triage-config <0|1>`
When enabled, merges all loaded catalog files into triage.

Example:

```bash
./build/rdk-service-hub --load-all-triage-config 1
```

### Command policy options

These set the command policy for each runtime plane.

- `off` = no enforcement for that plane
- `diagnostic` = only safe diagnostic commands are allowed
- `admin_unsafe` = broader admin-level command access is allowed

Examples:

```bash
./build/rdk-service-hub \
  --command-policy-default diagnostic \
  --command-policy-plane-triage diagnostic \
  --command-policy-plane-management diagnostic \
  --command-policy-plane-control admin_unsafe \
  --command-policy-plane-config-apply admin_unsafe
```

#### `--command-allowlist-file <path>`
Specifies the primary allowlist file for general command validation.

Example:

```bash
./build/rdk-service-hub \
  --command-allowlist-file /etc/rdk-service-hub/command_allowlist.json
```

#### `--command-allowlist-diagnostic-file <path>`
Specifies the allowlist used for diagnostic-safe command classes.

Example:

```bash
./build/rdk-service-hub \
  --command-allowlist-diagnostic-file /etc/rdk-service-hub/command_allowlist_diagnostic.json
```

#### `--command-denylist-admin-unsafe-file <path>`
Specifies the denylist used for admin-unsafe command filtering.

Example:

```bash
./build/rdk-service-hub \
  --command-denylist-admin-unsafe-file /etc/rdk-service-hub/command_denylist_admin_unsafe.json
```

### Resource gate options

These enforce host safety checks before command execution.

#### `--resource-check-enable <0|1>`
Enables or disables the resource gate before runtime execution.

Example:

```bash
./build/rdk-service-hub --resource-check-enable 1
```

#### `--resource-cpu-max-pct <pct>`
Maximum allowed CPU usage percentage before the command is blocked.

Example:

```bash
./build/rdk-service-hub --resource-cpu-max-pct 90
```

#### `--resource-mem-free-mb <mb>`
Minimum free memory required in MB.

Example:

```bash
./build/rdk-service-hub --resource-mem-free-mb 64
```

#### `--resource-disk-free-mb <mb>`
Minimum free disk space required in MB.

Example:

```bash
./build/rdk-service-hub --resource-disk-free-mb 32
```

#### `--resource-max-procs <count>`
Maximum number of active processes allowed before denial.

Example:

```bash
./build/rdk-service-hub --resource-max-procs 200
```

#### `--resource-fail-mode <deny|allow>`
Sets the behavior when the resource gate rejects a command.

Examples:

```bash
./build/rdk-service-hub --resource-fail-mode deny
./build/rdk-service-hub --resource-fail-mode allow
```

#### `--resource-usage-display <0|1>`
Prints the current process resource usage once when enabled.

Example:

```bash
./build/rdk-service-hub --resource-usage-display 1
```

### Timeout options

#### `--timeout-default-sec <seconds>`
Sets the default timeout for commands.

Example:

```bash
./build/rdk-service-hub --timeout-default-sec 30
```

#### `--timeout-extra-margin-sec <seconds>`
Adds a safety margin to command timeout calculations.

Example:

```bash
./build/rdk-service-hub --timeout-extra-margin-sec 60
```

#### `--help`
Displays the usage summary and supported command-line flags.

Example:

```bash
./build/rdk-service-hub --help
```

## Example: fully configured runtime

```bash
./build/rdk-service-hub \
  --catalog-dir /etc/rdk-service-hub/catalogs \
  --catalog-backend json \
  --load-all-triage-config 0 \
  --command-policy-default diagnostic \
  --command-policy-plane-triage diagnostic \
  --command-policy-plane-management diagnostic \
  --command-policy-plane-control admin_unsafe \
  --command-policy-plane-config-apply admin_unsafe \
  --command-allowlist-file /etc/rdk-service-hub/command_allowlist.json \
  --command-allowlist-diagnostic-file /etc/rdk-service-hub/command_allowlist_diagnostic.json \
  --command-denylist-admin-unsafe-file /etc/rdk-service-hub/command_denylist_admin_unsafe.json \
  --resource-check-enable 1 \
  --resource-cpu-max-pct 90 \
  --resource-mem-free-mb 64 \
  --resource-disk-free-mb 32 \
  --resource-max-procs 200 \
  --resource-fail-mode deny \
  --resource-usage-display 1 \
  --timeout-default-sec 30 \
  --timeout-extra-margin-sec 60
```

## Notes

- The runtime checks resource availability before executing commands when enabled.
- The resource gate is disabled by default for rollout safety.
- The current CPU threshold is set to 90% in the runtime default policy examples.
- The runtime policy model is plane-aware: triage and management are typically safer, while control and config-apply may permit broader command access.
