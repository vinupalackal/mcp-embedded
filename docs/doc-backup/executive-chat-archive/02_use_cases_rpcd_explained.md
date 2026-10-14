# 02 — Use Cases and RPCD Explanation

## Chat

"explain this with the use cases which we used and rpcd approach mentioned"

## Response

This module is acting as a runtime control plane for a multi-plane device architecture. The project docs define four operational use cases that map directly to the design:

- Triage: read-only diagnostics and status checks
- Management: operational actions and routine service oversight
- Control: runtime behavior changes and policy toggles
- Config-apply: high-impact configuration changes and rollbacks

Those are not arbitrary categories—they are the operating boundaries that separate low-risk visibility from high-risk mutation.

The design is intentionally separating intent by plane:

- Triage is for investigation and health verification
- Management is for service and lifecycle operations
- Control is for real-time runtime steering
- Config-apply is for persisted configuration changes

This matters because the same device cannot be treated as one flat operational environment. A cloud caller may need:
- a quick health check,
- a runtime toggle during a fault,
- a config change during a rollout,
- or a support diagnostic without touching production state.

The architecture ensures those actions are routed to the right safety boundary instead of all being treated as equivalent commands.

The runtime manager accepts requests, resolves the target plane, validates the request, and then either:
- invokes a plugin, or
- falls back to a catalog-defined tool

This is the operational flow described in the project docs:
- Cloud request enters the system
- Request is decoded and validated
- Metadata and access controls are checked
- Tool is dispatched by plane
- Response is returned through the existing messaging path

So the actual system is not just “a command runner.” It is a governed command broker for different operational contexts.

### RPCD-style model

The repo explicitly describes the dynamic plugin model as rpcd-style. In practical terms, that means:

- a daemon loads plugin modules at runtime
- each plugin exposes a fixed calling contract
- the host process invokes a named tool through a stable ABI
- the tool result is returned in a controlled response structure
- the daemon can switch between plugin execution and catalog execution depending on what is available

This is similar to an RPC daemon model:
- the daemon is the service endpoint
- the plugins are the service providers
- the request names the tool
- the host dispatches based on the contract, not ad hoc scripting

This is an important design choice because it creates:
- extensibility without rewriting the host
- predictable tool invocation
- consistent safety and governance
- separation of product logic from operational plumbing

The rpcd approach complements the plane model very well:

- Triage tools can be plugin-driven and lightweight
- Management tools can be safely isolated to routine operations
- Control actions can be enabled only through the appropriate plane
- Config-apply can be protected with stricter validation and versioning

That gives us a clean rule:
- low-risk investigative capabilities are easy to add and discover
- high-risk mutating actions are not mixed into diagnostic channels
- new tools can be introduced without changing the core daemon contract

The local endpoint and PUSH/CHANGED flow also fit this pattern. The system supports both external cloud-driven requests and local coordination without breaking the public path. That means a local orchestrator can discover tools, verify process health, push catalog updates, and notify peers when a capability changed.

### Executive summary

The best way to explain this module is:

This is a distributed runtime orchestration and policy enforcement layer that lets the organization safely expose, validate, and evolve device capabilities across separate operational planes—while using an rpcd-style plugin model to keep the system extensible, governed, and resilient.

In short:
- the use cases define the operating boundaries,
- the rpcd model gives standardized execution,
- and the dual-path design supports both cloud control and local orchestration safely.
