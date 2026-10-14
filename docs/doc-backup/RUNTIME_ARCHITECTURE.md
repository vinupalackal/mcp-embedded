# Multi-Plane Runtime Manager Single-Page Architecture

## Runtime architecture overview

```mermaid
flowchart TB
    subgraph OPS[Operational / cloud side]
        O1[Ops Gateway]
        O2[External callers]
    end

    subgraph TRANS[Transport layer]
        T1[Parodus / WRP transport]
        T2[PULL socket for requests]
        T3[PUSH socket for responses]
    end

    subgraph RUNTIME[Multi-Plane Runtime Manager process]
        R1[Broker / request loop]
        R2[Request decoder]
        R3[Metadata + ACL validation]
        R4[Tool registry]
        R5[Policy and auth guard]
        R6[Provider adapter layer]
        R7[Execution tracker]
        R8[Static catalog executor]
        R9[Dynamic provider backends]
        R10[Response builder]
        R11[Logging + metrics]
    end

    subgraph DATA[Runtime data sources]
        D1[Per-plane JSON catalogs]
        D2[Plugin discovery directory]
        D3[Native .so plugins]
        D4[Script plugins]
        D5[External provider sockets]
        D6[Runtime state / manifests]
    end

    O1 -->|WRP request| T1
    O2 --> O1
    T1 --> T2 --> R1
    R1 --> R2
    R2 --> R3
    R3 --> R4
    R4 --> R5
    R5 --> R6
    R6 --> R8
    R6 --> R9
    R8 --> R7
    R9 --> R7
    R7 --> R10
    R10 --> T3 --> T1

    D1 --> R4
    D2 --> R6
    D3 --> R6
    D4 --> R6
    D5 --> R6
    R6 --> D6
    R3 --> R11
    R7 --> R11
```

---

## Runtime flow in one pass

```mermaid
sequenceDiagram
    participant C as Caller
    participant P as Parodus
    participant M as Manager broker
    participant R as Registry
    participant G as Policy + auth guard
    participant A as Provider adapter
    participant E as Execution tracker
    participant X as Static or dynamic backend
    participant B as Response builder

    C->>P: Send WRP request
    P->>M: Receive request on PULL
    M->>R: Resolve tool name and target plane
    R-->>M: provider metadata + binding
    M->>G: Validate ACL, plane, auth, resource policy
    G->>A: Dispatch to correct provider class
    A->>X: Execute via native / script / external IPC
    X-->>A: result or started status
    A->>E: record execution state
    E-->>M: final state or execution ID
    M->>B: Normalize output
    B->>P: Send structured WRP response
    P-->>C: Return execution result
```

---

## Component responsibilities

### 1. Transport and entry
- Receives requests from the external operations path
- Dispatches responses back to the connected caller
- Maintains the runtime socket lifecycle and registration flow

### 2. Request validation
- Decodes and normalizes the incoming request
- Applies ACL and metadata rules
- Rejects invalid or unauthorized execution attempts before runtime work begins

### 3. Registry and provider resolution
- Resolves a tool against the current catalog set and active dynamic provider set
- Uses a unified registry to bind tool names to provider metadata
- Maintains provider state and handles conflict policy across provider classes

### 4. Safety and policy enforcement
- Confirms commands are allowed
- Validates provider trust, ownership, and socket/auth policy
- Applies blocklists, resource rules, and plane-scoping checks
- Ensures only valid and trusted providers are activated

### 5. Execution and output
- Executes approved commands or invokes provider handlers through the adapter layer
- Tracks async execution state, status, and cancellation
- Returns a consistent response payload for the caller

### 6. Observability and health
- Emits startup and dynamic tool inventory logs
- Records invocation, policy, and error metrics
- Generates a manifest of active provider state and tool coverage

---

## Architectural summary

The runtime is organized as a layered brokered control path:

1. External caller enters through Parodus / WRP transport.
2. The broker validates the request and resolves the correct provider path through the registry.
3. Catalogs and dynamic providers contribute to the runtime capability set.
4. Safety and policy checks gate execution.
5. The selected provider adapter executes the command or provider call.
6. Async state and final execution result are tracked and returned via a normalized runtime response.

This gives the system a clean separation across transport, validation, registry, provider dispatch, execution, and observability while keeping the runtime itself extensible across native, script, and external API-style provider models.
