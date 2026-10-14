# Runtime Argument and Reload User Guide

## Overview

This guide explains how to configure the multi-plane runtime manager at startup and how to re-apply runtime settings without restarting the process.

The runtime supports startup arguments such as:

- `--catalog-dir`
- `--catalog-backend <json|lmdb>`
- `--load-all-triage-config <0|1>`
- `--resource-usage-display <0|1>`
- command policy and resource overrides

The default behavior is intentionally conservative and keeps the service running even if no runtime arguments are supplied. The active design uses JSON mode by default, optionally supports LMDB only when the build includes it, and maintains a compact tmpfs-backed catalog index to lower memory usage without requiring a database dependency.

---

## Starting the service with runtime arguments

Use the help output to see the supported runtime options:

```sh
./build/rdk-service-hub --help
```

Example startup command:

```sh
./build/rdk-service-hub \
  --catalog-dir /etc/rdk-service-hub \
  --catalog-backend lmdb \
  --load-all-triage-config 1 \
  --resource-usage-display 1
```

When `--resource-usage-display 1` is set, the runtime prints a single `PROCESS_RESOURCE_USAGE` snapshot for the current process. The flag is off by default, so it does not add repeated runtime noise unless explicitly enabled.

### Supported backend values

- `json` : use the standard JSON catalog in memory
- `lmdb` : use the LMDB-backed catalog backend when available

If LMDB support is not enabled in the build, the runtime falls back to JSON automatically.

---

## Configuration precedence

When multiple sources define runtime configuration, the effective precedence is:

1. CLI argument
2. environment variable
3. config file
4. compile-time default

This keeps behavior predictable and preserves compatibility with earlier versions.

---

## Reloading configuration without restarting

The recommended refresh mechanism is `SIGHUP`.

Example:

```sh
kill -HUP <pid>
```

When the signal is received, the process should:

- detect that a reload is requested
- re-read runtime configuration values
- refresh catalog backend selection if needed
- reload catalog contents if the catalog directory or merge mode changed
- apply updated policy/resource values
- continue serving requests with the new state

This avoids the need to restart the service during routine configuration adjustments.

---

## Safe update model

The reload logic should not modify the active catalog in-place while requests are in flight. Instead, the runtime should:

1. build a replacement catalog snapshot
2. validate it
3. lock the catalog access path
4. swap it in atomically
5. release the lock

This approach keeps request handling stable and prevents inconsistent tool metadata from being read mid-update.

---

## Common examples

### Switch backend from JSON to LMDB

```sh
./build/rdk-service-hub --catalog-backend lmdb
```

Then reload after startup:

```sh
kill -HUP $(pidof rdk-service-hub)
```

### Enable merged triage catalogs

```sh
./build/rdk-service-hub --load-all-triage-config 1
```

Then refresh with:

```sh
kill -HUP $(pidof rdk-service-hub)
```

### Change catalog directory

```sh
./build/rdk-service-hub --catalog-dir /opt/rdk-service-hub/catalogs
```

Then apply the refresh signal to reload the catalog tree.

---

## Operational notes

- Startup-time arguments are the simplest way to set configuration before the daemon begins serving requests.
- Reloads are best used for configuration updates that are safe to apply while the process is running.
- If the selected backend is unavailable, the runtime should log the fallback and continue in JSON mode.
- All reload operations should log the effective values chosen so the operator can confirm the new runtime state.

---

## Recommended operator workflow

1. Start the service with the intended base configuration.
2. Verify the help output and active runtime values.
3. Use `SIGHUP` to apply configuration changes as needed.
4. Check logs to confirm the new backend, path, and merged-catalog state.
5. Avoid unsupported live changes that require a full service rebuild.

---

## Summary

The runtime supports startup-time overrides and can be extended to support safe live reconfiguration with a reload signal. This allows administrators to choose backend and catalog behavior without a process restart while preserving default safety and compatibility.
