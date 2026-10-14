# What to Implement Next Roadmap

## Scope
This roadmap is derived strictly from the repository evidence, the requirements baseline, and the implementation status review. It focuses on the most likely next workstreams needed to move the project from a strong runtime foundation toward a fully productized, production-ready CPE Device Agent.

## Decision basis
The following items are ranked based on evidence gaps rather than generic SDLC assumptions:

- The repo already implements core runtime execution, registry behavior, command safety, and validation.
- The major remaining gaps are in transport security, platform-aware routing, operational protection, and complete dynamic capability governance.
- The roadmap below prioritizes the gaps that are most likely to affect production readiness and compliance.

## Roadmap

### Phase 1: Close the secure transport and authorization gap

#### 1. Complete XMiDT secure request path
- Objective: Make the communication path match the requirement for secure, controlled cloud-to-device communication over the existing XMiDT transport.
- Why this is next: The runtime already has the message loop and socket patterns; the evidence gap is the full secure transport/auth flow.
- Evidence: [README.md](README.md), [src/rdk-service-hub.c](src/rdk-service-hub.c), [spec.md](spec.md), [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md)
- Deliverables:
  - explicit XMiDT-facing request validation
  - authenticated request handling model
  - authorizer integration for request consumer identity
  - secure message path validation in tests

#### 2. Finalize per-request authorization model
- Objective: Replace permissive config-only enforcement with a complete request-by-request authorization flow.
- Why this is next: The repo has allowlist and denylist configuration, but the full request authorization model is not fully evidenced.
- Evidence: [config/rdk-service-hub.conf](config/rdk-service-hub.conf), [README.md](README.md), [spec.md](spec.md)
- Deliverables:
  - request identity validation
  - tool-level authorization mapping
  - rejection and audit logging for unauthorized calls

### Phase 2: Finish platform-aware routing and capability governance

#### 3. Implement model/platform-aware tool selection
- Objective: Route commands to the correct tool set based on platform, model, and capability support.
- Why this is next: Capability metadata and plane definitions exist, but explicit model-aware routing is still a missing production behavior.
- Evidence: [config/rdk-service-hub.conf](config/rdk-service-hub.conf), [docs/README.md](docs/README.md), [spec.md](spec.md)
- Deliverables:
  - platform/capability model definition
  - tool-to-device match logic
  - tests for routing failures and unsupported devices

#### 4. Complete dynamic capability governance for approved updates
- Objective: Move dynamic capability delivery from a runtime feature to a governed production workflow.
- Why this is next: Dynamic plugin support is present, but the repo does not evidence a complete governance model for approved delivery and lifecycle control.
- Evidence: [CMakeLists.txt](CMakeLists.txt), [src/plugin_manager.c](src/plugin_manager.c), [src/tool_registry.c](src/tool_registry.c), [spec.md](spec.md)
- Deliverables:
  - approval and provenance model for dynamic tools
  - signed/approved plugin policy
  - update lifecycle and rollback behavior
  - safety validation in live tests

### Phase 3: Harden runtime protection and resilience

#### 5. Validate full resource guardrail enforcement under load
- Objective: Ensure CPU, memory, concurrency, and execution constraints protect the device during heavy operational load.
- Why this is next: The repo defines resource parameters, but the complete enforcement and load-case evidence is still incomplete.
- Evidence: [config/rdk-service-hub.conf](config/rdk-service-hub.conf), [README.md](README.md), [spec.md](spec.md)
- Deliverables:
  - load test scenarios for each guardrail
  - failure mode validation
  - structured metrics and logs for resource events

#### 6. Close the monitoring, failure handling, and recovery gap
- Objective: Make the runtime operationally observable and recoverable under real-world execution and communication failures.
- Why this is next: Logging and reload behavior are implemented, but the repo does not fully evidence a complete failure/recovery model.
- Evidence: [src/rdk-service-hub.c](src/rdk-service-hub.c), [config/rdk-service-hub.conf](config/rdk-service-hub.conf), [spec.md](spec.md)
- Deliverables:
  - structured error taxonomy
  - command execution monitoring
  - recovery hooks and operational telemetry
  - negative-path test coverage

### Phase 4: Complete product and profile readiness

#### 7. Formalize read-only diagnostic enforcement and policy segmentation
- Objective: Make read-only execution a verified product requirement rather than a general safe-default pattern.
- Why this is next: The architecture favors safe behavior, but a robust read-only enforcement model across tool classes is not fully evidenced.
- Evidence: [README.md](README.md), [config/rdk-service-hub.conf](config/rdk-service-hub.conf), [spec.md](spec.md)
- Deliverables:
  - read-only capability classification per tool
  - enforcement rules for each plane
  - validation for allowed vs denied operations

#### 8. Finalize cross-profile architecture reuse and extensibility model
- Objective: Turn the current multi-plane runtime pattern into a verified cross-profile framework for Broadband and Entertainment use.
- Why this is next: The design concept is present, but the repo does not show a complete multi-profile implementation.
- Evidence: [README.md](README.md), [docs/README.md](docs/README.md), [spec.md](spec.md)
- Deliverables:
  - profile abstraction model
  - profile-specific capability manifests
  - interface contract for future diagnostic and LCM extensions

## Recommended sequence

1. Secure transport + authorization
2. Platform-aware routing
3. Dynamic capability governance
4. Resource guardrail validation
5. Monitoring/recovery hardening
6. Read-only policy enforcement
7. Cross-profile extensibility finalization

## Expected outcome
By the end of this roadmap, the project should move from a strong runtime foundation to a production-ready, secure, policy-driven, model-aware diagnostic service that more closely matches the requirements in [spec.md](spec.md) and the intent definitions in [intent.md](intent.md).
