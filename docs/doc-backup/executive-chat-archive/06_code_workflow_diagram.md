# 06 — Code Workflow Diagram (Textual / Mermaid)

## Runtime request workflow

```mermaid
flowchart TD
    A[Cloud / Operations Request] --> B{Request received}
    B --> C[Decode WRP envelope]
    C --> D[Extract tool + plane + metadata]
    D --> E{Metadata valid?}
    E -- No --> F[Reject with deterministic error]
    E -- Yes --> G{Tool exists in catalog?}

    G -- No --> H{Dynamic plugin enabled?}
    H -- Yes --> I[Resolve plugin by tool contract]
    H -- No --> J[Return tool not found / invalid request]

    G -- Yes --> K[Resolve tool from static catalog]
    I --> L{Plugin invocation successful?}
    K --> M[Execute tool command]

    L -- No --> N[Map plugin failure to error response]
    L -- Yes --> O[Return plugin output]
    M --> P{Command result valid?}
    P -- No --> Q[Return tool execution error]
    P -- Yes --> R[Build response payload]

    O --> R
    R --> S[Send WRP response]
    S --> T[Cloud / caller receives result]

    M --> U{Was this a mutating action?}
    U -- Yes --> V[Promote / update in-memory catalog if applicable]
    V --> W[Emit CHANGED notification]
    W --> X[Optional capability-sync notification]
    U -- No --> Y[No state mutation]
```

## Plane-aware capability routing

```mermaid
flowchart LR
    A[Incoming request] --> B{plane specified?}
    B -- No --> C[Use default or reject if ambiguous]
    B -- Yes --> D[Route to triage / management / control / config-apply]

    D --> E[Triage: diagnostics and read-only checks]
    D --> F[Management: operational management]
    D --> G[Control: runtime toggles and policy changes]
    D --> H[Config-apply: mutation and rollback operations]

    E --> I[Low blast radius]
    F --> J[Moderate operational risk]
    G --> K[Controlled runtime change]
    H --> L[High-impact mutation path]
```

## Static vs dynamic tool handling

```mermaid
flowchart TB
    A[Tool request] --> B{Tool resolution path}
    B --> C[Static Catalog]
    B --> D[Dynamic Plugin]

    C --> C1[Known, versioned tool]
    C1 --> C2[Predictable execution path]
    C2 --> C3[Stable governance and validation]

    D --> D1[Runtime-loaded module]
    D1 --> D2[Fixed ABI / rpcd-style contract]
    D2 --> D3[Dynamic extension without core rebuild]

    C3 --> E[Return structured result]
    D3 --> E
```

## Communication and transport model

```mermaid
flowchart LR
    A[Cloud / Control System] --> B[Public transport channel]
    B --> C[Runtime manager]

    D[Local orchestrator / Dispatch core] --> E[Local transport channel]
    E --> C

    C --> F[Static catalog tools]
    C --> G[Dynamic plugins]
    C --> H[DESCRIBE / HEALTH / PUSH / CHANGED]
    C --> I[Notifications to downstream consumers]
```

## Summary in plain language

The workflow is:
1. Receive the request.
2. Decode and validate request metadata.
3. Resolve the right tool by plane.
4. Prefer a static catalog path when applicable; otherwise use the dynamic plugin path.
5. Execute the action under governance controls.
6. Return a structured response.
7. If the action mutates runtime state, emit the relevant change notification.

This gives the system a disciplined runtime lifecycle: request validation, safe execution, controlled delivery, and operational visibility.
