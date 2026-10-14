# Multi-Plane Runtime Manager

## Module Features and Supported Capabilities

This module provides a runtime-owned plugin system for discovering, validating, and invoking operational tools without hardcoding them into the application binary.

---

## 1. Dynamic plugin discovery

- Scans configured plugin directories at startup
- Detects new plugin files automatically during runtime
- Supports both native shared-library plugins and script-based plugins
- Handles stale plugins, removed plugins, and changed plugin metadata
- Maintains a live registry of available tools

```mermaid
flowchart LR
    A[Plugin directory] --> B[Startup scan]
    B --> C[Runtime scan]
    C --> D{File type}
    D -->|.so| E[Load native plugin]
    D -->|.sh/.py/.pl/.rb| F[Load script plugin]
    E --> G[Register tools]
    F --> G
    G --> H[Runtime inventory]
```

Related implementation:
- [src/plugin_manager.c](../src/plugin_manager.c)
- [src/tool_registry.c](../src/tool_registry.c)

---

## 2. Native plugin support

- Loads compiled .so plugin modules with dlopen
- Validates the plugin API version before activation
- Resolves required symbols such as init, get_tool_count, get_tool, invoke, and deinit
- Ensures plugin compatibility with the runtime before registration
- Prevents activation of unsupported or mismatched plugin builds

```mermaid
sequenceDiagram
    participant M as Runtime Manager
    participant P as Native Plugin (.so)
    participant V as Validation

    M->>P: dlopen(path)
    P-->>M: Shared library handle
    M->>V: Resolve required symbols
    V-->>M: API and ABI valid
    M->>P: init(host_api)
    P-->>M: plugin context
    M->>P: get_tool_count / get_tool
    P-->>M: tool definitions
    M->>P: invoke(tool, req)
    P-->>M: response
```

Supported behavior:
- ABI validation
- Symbol resolution checks
- Compatibility enforcement
- Safe activation only after verification

---

## 3. Script plugin support

- Loads executable plugin scripts in supported formats such as .sh, .py, .pl, and .rb
- Parses tool metadata from script-provided JSON output
- Registers tool names and descriptions without requiring a compiled plugin binary
- Invokes script-based tools by name through the same runtime dispatch path
- Keeps script plugins consistent with the native plugin lifecycle model

```mermaid
flowchart TD
    A[Executable script plugin] --> B[--list-tools]
    B --> C[Parse JSON metadata]
    C --> D[Tool names + descriptions]
    D --> E[Register in runtime]
    E --> F[--invoke tool]
    F --> G[Return JSON response]
    G --> H[Dispatch to caller]
```

This enables a lower-friction extension model while preserving a unified runtime experience.

---

## 4. Runtime registration and tool binding

- Maintains a central tool registry keyed by tool name
- Binds each tool to its provider plugin instance
- Resolves invocation requests by name at runtime
- Tracks active plugin ownership and tool ownership for each binding
- Supports conflict handling when a tool name is duplicated

```mermaid
flowchart LR
    A[Tool request] --> B[Registry lookup]
    B --> C{Tool found?}
    C -->|No| D[Not found / error]
    C -->|Yes| E[Provider plugin]
    E --> F[Invoke plugin handler]
    F --> G[Return result]
```

Key capabilities:
- Duplicate detection
- Controlled registration policy
- Safe provider lookup
- Runtime dispatch by tool name

---

## 5. Startup and dynamic inventory visibility

- Emits a startup inventory log with the header: SUPPORTED TOOLS ON STARTUP
- Emits dynamic registration logs with the header: DYNAMIC TOOLS
- Logs plugin metadata such as tool name, description, and plan stage
- Makes supported capabilities visible at process boot and during plugin add events
- Helps operators confirm what the runtime is exposing at any time

```mermaid
flowchart TB
    A[Manager startup] --> B[SUPPORTED TOOLS ON STARTUP]
    B --> C[Enumerate active plugins]
    C --> D[Emit tool table]
    D --> E[Operator sees inventory]

    F[New plugin added] --> G[DYNAMIC TOOLS]
    G --> H[Emit new tool table]
    H --> I[Operator sees update]
```

---

## 6. Plugin lifecycle management

The module manages a complete plugin lifecycle:

- discovery
- validation
- load
- registration
- active use
- file change detection
- unload on replacement or removal
- cleanup of stale state

```mermaid
stateDiagram-v2
    [*] --> DISCOVERED
    DISCOVERED --> LOADING
    LOADING --> ACTIVE: validation passed
    LOADING --> FAILED: validation failed
    ACTIVE --> DRAINING: file removed or changed
    DRAINING --> UNLOADED
    ACTIVE --> UNLOADED: explicit stop
    FAILED --> [*]
    UNLOADED --> [*]
```

This keeps the runtime robust when plugins are added, updated, or removed while the process remains running.

---

## 7. Watcher and refresh modes

- Poll-based refresh mode
- Linux inotify-based refresh mode
- Optional startup scan and periodic rescan logic
- Debounce control to avoid excessive churn during rapid file changes

```mermaid
flowchart LR
    A[Watch config] --> B{Mode}
    B -->|poll| C[Periodic directory scan]
    B -->|inotify| D[Filesystem event watch]
    C --> E[Reconcile plugin set]
    D --> E
    E --> F[Update registry]
```

This allows the runtime to adapt to filesystem updates without requiring a process restart.

---

## 8. Security and path validation

- Restricts plugin loading to trusted directories
- Rejects files outside the configured allowed plugin roots
- Verifies owner and file type before accepting a plugin
- Blocks writable group or world permissions
- Avoids loading unsafe or unexpected plugin artifacts

```mermaid
flowchart TD
    A[Candidate plugin file] --> B[realpath validation]
    B --> C{Inside allowed directory?}
    C -->|No| D[Reject]
    C -->|Yes| E[Check ownership]
    E --> F{Owner trusted and not writable?}
    F -->|No| D
    F -->|Yes| G[Accept and load]
```

This reduces the chance of untrusted or malformed plugin files being activated.

---

## 9. Metrics and observability

- Counts total plugins loaded
- Counts failed plugin loads
- Counts tools registered
- Tracks plugin invocations and invocation errors
- Exposes runtime metrics through the manager interface

```mermaid
flowchart LR
    A[Plugin manager] --> B[Metrics counters]
    B --> C[Loaded plugins]
    B --> D[Failed loads]
    B --> E[Registered tools]
    B --> F[Invocation count]
    B --> G[Invocation errors]
    C --> H[Operational telemetry]
    D --> H
    E --> H
    F --> H
    G --> H
```

This gives the runtime a measurable operational profile and helps diagnose plugin health.

---

## 10. Manifest and operational state persistence

- Generates a plugin manifest file describing currently active plugin paths and tools
- Captures a structured runtime inventory for operational inspection
- Supports later validation or auditing of the active plugin set

```mermaid
flowchart TD
    A[Runtime registry] --> B[Write plugin manifest]
    B --> C[JSON file]
    C --> D[Audit / inspection / recovery]
```

---

## 11. Packaging and deployment compatibility

- Builds native shared-library output as .so artifacts
- Stages plugin payloads in the configured dynamic directory
- Supports installation into runtime-owned dynamic plugin paths instead of legacy static library locations
- Keeps deployment aligned with runtime-owned plugin discovery behavior

```mermaid
flowchart LR
    A[Build step] --> B[.so plugin artifacts]
    B --> C[Install into dynamic directory]
    C --> D[Runtime discovery]
    D --> E[Activation without static rebuild]
```

---

## 12. Summary

The module’s primary capabilities are:

- dynamic tool discovery
- runtime plugin loading
- script-based extension support
- safe validation and path confinement
- lifecycle management
- registry-based dispatch
- startup and dynamic observability
- plugin metadata tracking
- operational metrics and manifest generation

This is a runtime extension framework, not just a list of commands or tools.
