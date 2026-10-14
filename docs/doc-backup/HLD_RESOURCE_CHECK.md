# High-Level Design: Resource Availability Check Before Tool Execution

## 1. Overview

This design adds a pre-execution resource gate to the multi-plane runtime manager. The runtime manager shall validate host resource availability before launching static commands or invoking dynamic plugin tools. The goal is to protect the device from overload conditions and ensure that tool execution only occurs when the system has adequate headroom.

## 2. Problem Statement

The runtime manager executes tools on behalf of callers. These tools may be:
- static catalog-backed commands
- dynamically loaded shared-library plugins

Without a resource gate, a tool request may execute while the system is already under memory, CPU, disk, or process pressure. This can produce hangs, excessive load, or out-of-memory conditions. A pre-execution guard is therefore required.

## 3. Goals

- prevent unsafe execution under overload conditions
- support both static and dynamic tool paths
- maintain low execution overhead
- allow runtime configuration via config file and environment override
- log and expose resource availability and usage clearly

## 4. Non-Goals

- full cluster-level scheduler or orchestration
- advanced per-user quotas
- dynamic priority assignment
- cross-device resource federation

## 5. Architecture

The resource gate sits in front of the execution layer and is invoked before any command or plugin callback is started.

### 5.1 Components

- Tool resolution layer
  - resolves a tool name from catalog or registry
- Resource policy layer
  - loads thresholds from config/env
- Resource collector
  - reads current CPU, memory, disk, and process usage
- Decision engine
  - returns allow / deny / defer
- Execution layer
  - static run_command() path
  - dynamic plugin invoke path
- Observability layer
  - logs resource state and output before and after execution

## 6. Execution Flow

### 6.1 Static tools

1. Client sends request with tool name and optional command
2. Runtime resolves tool in catalog
3. Runtime computes effective command
4. Resource gate evaluates current host health
5. If allowed, command is executed
6. Output is captured and returned
7. Resource snapshot is logged

### 6.2 Dynamic plugin tools

1. Client sends request with tool name
2. Runtime resolves tool name in plugin registry
3. Resource gate evaluates current host health
4. If allowed, plugin `invoke()` is called
5. Response is returned
6. Resource snapshot is logged

## 7. Policy Model

A resource policy contains the following:
- enabled flag
- CPU max percent
- minimum available memory
- minimum free disk space
- maximum process count
- fail mode

Policy precedence:
1. environment variable
2. config file
3. built-in default

## 8. Decision Rules

The runtime shall deny a tool if any of the following thresholds are violated:
- CPU usage above configured maximum
- memory available below minimum
- disk free below minimum
- process count above maximum

A fail-mode policy can optionally permit deferral instead of denial for transient overload conditions.

## 9. Observability

The runtime shall print and log resource availability and usage data during tool execution.

Required output should include:
- tool name
- plane
- command or plugin name
- CPU usage
- available memory
- free disk space
- process count
- decision result: allow / deny / defer
- execution status: start / finish

## 10. Failure Behavior

If resource collection fails or returns invalid metrics, the runtime shall default to a safe deny or configured fail-safe policy.

## 11. Security and Safety

- no execution occurs before the resource gate approves it
- denial reason must be explicit and logged
- no shell spawns are permitted beyond the already-safe execution model
- dynamic plugin tools are subject to the same gate as static tools

## 12. Success Metrics

- blocked execution under overload conditions
- reduced risk of OOM and runaway tool execution
- clear operator visibility into resource state
- maintainable configuration model
