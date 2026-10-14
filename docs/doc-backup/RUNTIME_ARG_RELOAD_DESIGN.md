# Runtime Argument and Live Reload Design

## Objective

Provide a configurable runtime model for the multi-plane runtime manager in which:

- catalog backend selection can be supplied as a runtime argument
- non-breaking defaults remain in place when no argument is provided
- a reload signal can re-apply configuration without a process restart
- the system remains safe, predictable, and compatible with the existing startup flow

This design aligns with the current startup override mechanism already present in the codebase and extends it to a reload-safe runtime configuration model.

---

## Current state

The runtime currently supports startup-time overrides such as:

- `--catalog-dir`
- `--catalog-backend <json|lmdb>`
- `--load-all-triage-config <0|1>`
- command-policy and resource override flags
- `--resource-usage-display <0|1>`

These are parsed in the CLI override path of [src/rdk-service-hub.c](../src/rdk-service-hub.c). Backend selection and the tmpfs-backed low-memory catalog index are implemented in [src/catalog_backend.c](../src/catalog_backend.c). The runtime continues to start normally with no arguments, using the configured defaults instead of failing or requiring explicit startup parameters.

The key design decision is that default behavior is intentionally safe and lightweight: JSON is the default backend, LMDB remains optional and compile-time gated, and the runtime keeps a compact tmpfs index rather than a full catalog snapshot in process memory.

---

## Goals

1. Keep startup behavior simple and backwards compatible.
2. Support runtime argument selection for catalog backend and catalog merge settings.
3. Allow live refresh of settings with a signal, without requiring the process to exit.
4. Avoid unsafe partial updates while the runtime is actively serving requests.
5. Preserve a clearly-defined default behavior when no override is provided.

---

## Proposed runtime argument model

### CLI surface

Add runtime startup arguments such as:

- `--catalog-backend <json|lmdb>`
- `--catalog-dir <path>`
- `--load-all-triage-config <0|1>`
- existing command policy and resource overrides remain supported

The argument parsing logic should remain centralized in the override application function, matching the current pattern in [src/rdk-service-hub.c](../src/rdk-service-hub.c).

### Default behavior

If no runtime argument is supplied:

- use the configured default backend
- use the configured default catalog directory
- keep the current behavior for triage merging and resource control

This preserves compatibility with the current service startup model and is consistent with the low-risk deployment expectation.

---

## Reload design

### Signal choice

Use `SIGHUP` as the primary reload trigger because it is already a common daemon control signal and is easy to handle without changing request protocol behavior.

The runtime should support a signal-driven reload workflow such as:

- on `SIGHUP`, set a `reload_requested` flag
- main loop observes the flag and calls the reload routine
- after reload completes, continue serving requests without restart

### Reload function responsibilities

The reload routine should re-run the same configuration read steps currently used during startup, specifically:

1. parse config file if present
2. reapply environment overrides if present
3. refresh catalog directory and file names
4. refresh backend selection
5. reload allowlist/denylist policy tables if the relevant paths were modified
6. reload resource policy settings if they are runtime-configurable
7. reload catalog contents if catalog backend or catalog directory changed
8. rebuild any in-memory merged triage catalog entries

### Reload safety requirements

To avoid unsafe mid-request state churn:

- reload should operate in phases
- reads of the active catalog should be protected by the existing catalog mutex
- the old catalog snapshot must remain valid until the new catalog is fully built
- only after a successful swap should the runtime begin using the new catalog

This aligns with the existing catalog swap pattern described in [src/rdk-service-hub.c](../src/rdk-service-hub.c) and is consistent with the planned catalog version/swap semantics already present in the code comments.

---

## Catalog backend refresh behavior

### Runtime backend switching

The runtime should be able to switch between:

- JSON catalog mode
- LMDB-backed catalog mode

The decision should be based on a runtime value and not hard-coded in the startup path.

The backend selection logic already exists in [src/catalog_backend.c](../src/catalog_backend.c), where the code evaluates:

- config file key `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_BACKEND`
- environment variable `MULTI_PLANE_RUNTIME_MANAGER_CATALOG_BACKEND`
- fallback to JSON if LMDB is not supported

The reload flow should simply re-run this selection logic and reinitialize the backend if the effective choice changes.

### Safe swap lifecycle

1. detect new backend request
2. if switching to LMDB, open the LMDB environment and validate readiness
3. if switching to JSON, keep the JSON catalog path, but rebuild the live catalog object
4. build the new catalog snapshot in memory
5. lock catalog access
6. swap active pointer(s) and version metadata
7. release lock
8. free or recycle the old snapshot only after the swap is committed

This avoids races with in-flight requests.

---

## Dynamic option support

Dynamic option support should be modeled as a reloadable runtime config map rather than ad-hoc scattered environment writes. The design should use a central configuration table keyed by option name, for example:

- `catalog-dir`
- `catalog-backend`
- `load-all-triage-config`
- `command-policy-default`
- `resource-check-enable`
- `timeout-default-sec`

Each option should be able to support:

- startup parsing
- reload parsing
- validation
- defaulting
- safe conversion to environment/config equivalents

This reduces duplication and keeps the CLI and reload paths consistent.

---

## Example flow

### Startup

```sh
./build/rdk-service-hub --catalog-backend lmdb --load-all-triage-config 1
```

The runtime starts with the task-specific configuration and loads the catalog in the selected backend.

### Runtime refresh

```sh
kill -HUP <pid>
```

The process reloads:

- backend selection
- catalog directory / merge option
- command policy overrides
- timeout values
- resource gate settings

No process restart is required.

---

## Risks and mitigations

### Risk: partial reload during active requests

Mitigation:

- use the existing catalog lock during swaps
- do not mutate the active catalog in-place
- build a new snapshot first, then swap atomically

### Risk: invalid backend selection

Mitigation:

- validate `json|lmdb` before applying reload
- fallback to JSON on invalid or unsupported values
- log the effective backend at reload time

### Risk: config drift between env and file state

Mitigation:

- define precedence clearly:
  1. CLI override
  2. environment variable
  3. config file
  4. compile-time default

This precedence model is already familiar in the existing project code and should be retained.

---

## Compatibility plan

1. Keep the current startup override path unchanged for existing users.
2. Introduce the runtime reload path behind the signal handler and config reload routine.
3. Add logging for reload events and the effective runtime values chosen.
4. Validate each reload with a minimal smoke check before replacing the active catalog.

---

## Tmpfs-backed low-memory catalog model

A dependency-free design that preserves performance without keeping a full catalog snapshot resident in process memory is to keep a compact catalog index in tmpfs and lazy-load the full tool entries on demand.

### Design

- keep the authoritative catalog on disk in JSON or msgpack format
- generate a compact index in tmpfs at startup or reload
- the tmpfs index contains only:
  - plane name
  - tool name
  - record offset or address
  - tool kind / timeout summary
  - validation checksum or version tag
- on tool lookup, read only the matching record from the disk-backed catalog and parse it into a short-lived entry
- keep a small LRU cache for frequently used tools to avoid repeated parsing

### Why this is a good fit

- avoids the full in-memory snapshot of every tool
- uses RAM for only the small hot index and recent entries
- keeps reload latency low because the tmpfs index can be rebuilt quickly
- avoids any database dependency
- preserves fast access for the most commonly used tools

### Safe reload pattern

1. build a new tmpfs index in a scratch location
2. validate the index against the catalog files
3. lock the catalog access mutex
4. atomically switch the active index pointer
5. release the lock
6. retire the old index after the swap completes

This matches the existing reload safety requirements already described earlier in this document: build a replacement snapshot first, then swap it in atomically.

### Recommended runtime policy

- default mode: JSON catalog + tmpfs index + lazy record load
- optional mode: LMDB only when the deployment genuinely needs a database-backed lookup layer
- keep tmpfs as a performance optimization, not as a complete in-memory copy of the catalog

### Operational note

tmpfs is best used for the index and hot metadata, not for a giant full snapshot of every tool definition. That keeps memory bounded while still reducing lookup latency for common requests.

---

## Recommendation

The most stable implementation is:

- keep startup arguments as the primary configuration source
- add `SIGHUP`-driven reload support for the same configuration keys
- rebuild catalogs and backend state behind a lock
- prefer a tmpfs-backed compact index + lazy record loading over a full in-memory snapshot
- preserve default behavior when no runtime overrides are supplied

This keeps the runtime simple, predictable, backward compatible, and materially lighter in memory without a database dependency.
