# Multi-Plane Runtime Manager Components and Workflow

This diagram is derived from the implementation in [src/rdk-service-hub.c](../src/rdk-service-hub.c), the optional plugin runtime in [src/plugin_manager.c](../src/plugin_manager.c), and the design references in [docs/ARCHITECTURE.md](ARCHITECTURE.md), [README.md](../README.md), and [docs/guides/API_WORKFLOW_FUNCTIONALITY_GUIDE.md](guides/API_WORKFLOW_FUNCTIONALITY_GUIDE.md).

## 1. Component architecture

```mermaid
flowchart TB
    subgraph Cloud[Cloud / OPS path]
        OPS[OPS Gateway / cloud caller]
    end

    subgraph Device[Device runtime]
        PAR[Parodus\nWRP transport\nPULL/PUSH sockets]

        subgraph MPRM[Multi-Plane Runtime Manager]
            MAIN[Main loop\nstartup + registration + receive loop]
            DEC[WRP decode\ninner payload decode]
            META[Metadata validation\nACL gate\nrequest normalization]
            CAT[Catalog manager\nper-plane JSON catalogs\nindex lookup]
            PLUG[Plugin manager\noptional dynamic plugins]
            SAFE[Safety + policy gate\nblocked tokens + resource checks\nstatic vs dynamic handling]
            EXEC[Executor\nfork/exec + timeout + stdout capture]
            RESP[Response builder\nWRP payload + status]
        end
    end

    CATALOGS[(Per-plane catalog files\ntriage / management / control / config-apply)]
    PLUGINS[(Plugin .so modules\n/usr/lib/.../plugins)]

    OPS -->|WRP request type=3| PAR
    PAR -->|PULL 127.0.0.1:6669| MAIN
    MAIN --> DEC
    DEC --> META
    META --> CAT
    META --> PLUG
    CAT --> SAFE
    PLUG --> SAFE
    SAFE --> EXEC
    EXEC --> RESP
    RESP -->|WRP response type=3| PAR
    PAR --> OPS

    MAIN -->|startup registration\nmsg_type=9| PAR
    MAIN -->|keepalive\nmsg_type=10| PAR

    CATALOGS --> CAT
    PLUGINS --> PLUG
```

### Responsibilities

- Main loop: startup, registration, socket lifecycle, receive loop, and keepalives.
- Catalog manager: loads plane-specific catalog JSON files and resolves tool names by plane.
- Metadata / ACL gate: validates request metadata and denies unauthorized execution before catalog/path execution.
- Plugin manager: loads dynamic plugins and intercepts tool requests before the static catalog path.
- Safety gate: enforces static vs dynamic semantics, blocklisted program tokens, resource policy, and catalog validation state.
- Executor: runs the final command, applies timeout and output limits, and returns exit/status information.
- Response builder: wraps execution output in a WRP response and sends it back to Parodus.

---

## 2. Request processing workflow

```mermaid
sequenceDiagram
    participant O as OPS Gateway
    participant P as Parodus
    participant M as Main loop
    participant D as Request decoder
    participant C as Catalog manager
    participant G as Dynamic plugin manager
    participant S as Safety / policy gate
    participant E as Command executor
    participant R as Response builder

    O->>P: WRP request (type=3)
    P->>M: Receive on PULL socket
    M->>D: Decode outer WRP and inner payload
    D->>D: Extract tool, command, plane, metadata
    alt Dynamic plugin matches tool
        D->>G: plugin_manager_invoke(...)
        G-->>D: plugin output / exit status
        D->>R: Build WRP response
    else Static catalog path
        D->>C: resolve tool in one plane or across planes
        C-->>D: catalog entry or miss
        D->>S: enforce static/dynamic rules, resource checks, blocklist
        S->>E: execute selected command
        E-->>D: stdout + exit code
        D->>R: Build WRP response
    end
    R->>P: Send response over PUSH socket
    P->>O: Return response
```

### Execution path from source

1. The service receives a WRP request in the receive loop and decodes the payload in [src/rdk-service-hub.c](../src/rdk-service-hub.c#L3453-L3995).
2. The request is checked for metadata and ACL constraints before catalog execution in [src/rdk-service-hub.c](../src/rdk-service-hub.c#L3525-L3595).
3. A plugin invocation path runs before the static catalog path when plugins are enabled, as shown in [src/rdk-service-hub.c](../src/rdk-service-hub.c#L3595-L3685).
4. For static tools, the catalog is looked up by tool and optional plane in [src/rdk-service-hub.c](../src/rdk-service-hub.c#L979-L1165).
5. The final command is run with timeout and output capture in [src/rdk-service-hub.c](../src/rdk-service-hub.c#L2874-L3230).
6. The response is wrapped back into WRP and sent to Parodus in the same handle flow in [src/rdk-service-hub.c](../src/rdk-service-hub.c#L3995-L4065).

---

## 3. Startup and lifecycle view

```mermaid
flowchart LR
    START[Process start] --> LOAD[Load runtime config]
    LOAD --> CATS[Load per-plane catalogs]
    CATS --> IDX[Refresh tmpfs catalog index]
    IDX --> REG[Register with Parodus]
    REG --> LOOP[Enter message loop]
    LOOP --> RECV{Request or keepalive?}
    RECV -->|Keepalive| ACK[Respond to keepalive]
    RECV -->|Request| DISPATCH[Dispatch request worker / inline handler]
    DISPATCH --> EXECUTE[Run tool / plugin]
    EXECUTE --> SEND[Send response]
    SEND --> LOOP
```

### Startup model

- The runtime can load multiple catalog files from a directory, not just a single file; this is documented in [src/rdk-service-hub.c](../src/rdk-service-hub.c#L237-L243) and implemented in [src/rdk-service-hub.c](../src/rdk-service-hub.c#L979-L1047).
- A tmpfs-backed catalog index is refreshed during startup, as seen in [src/rdk-service-hub.c](../src/rdk-service-hub.c#L943-L978).
- Dynamic plugins are optional and discovered from configured plugin directories, as implemented in [src/plugin_manager.c](../src/plugin_manager.c#L103-L220).

---

## 4. Design mapping to the docs

| Source area | Design meaning |
| --- | --- |
| [src/rdk-service-hub.c](../src/rdk-service-hub.c) | Runtime service implementation and full control flow |
| [src/plugin_manager.c](../src/plugin_manager.c) | Optional dynamic plugin loading and dispatch |
| [docs/ARCHITECTURE.md](ARCHITECTURE.md) | High-level architecture diagram and text |
| [README.md](../README.md) | Current runtime model and operational defaults |
| [docs/guides/PLANES_AND_USE_CASES.md](guides/PLANES_AND_USE_CASES.md) | Conceptual plane routing and use cases |
| [docs/guides/API_WORKFLOW_FUNCTIONALITY_GUIDE.md](guides/API_WORKFLOW_FUNCTIONALITY_GUIDE.md) | Detailed request handling guidance |

This document intentionally tracks the source-of-truth behavior in code and cross-references the project documentation rather than showing a stale or simplified legacy design.
