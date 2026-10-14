# Dynamic Script Plugin Loading Specification

## Purpose

This specification defines how a runtime-managed script plugin is discovered, registered, and invoked in the same dynamic model used for compiled `.so` plugins. The goal is to allow a trusted executable or script placed in a runtime plugin directory to participate in the tool registry without requiring a native library build.

This design intentionally keeps the runtime-owned security model intact:
- the runtime still controls plugin discovery
- the runtime still owns validation and registration
- only files copied into trusted plugin directories are considered candidates
- script-based plugins must expose a minimal contract for metadata and invocation

## Scope

This specification covers:
- discovery of script plugins in the plugin directories
- registration of tool names into the runtime registry
- generation of a runtime lookup table
- invocation dispatch by tool name
- script lifecycle behavior and security constraints

## Supported script types

The runtime may accept scripts or executables with a supported extension or shebang pattern, including:
- `.sh`
- `.py`
- `.pl`
- `.rb`
- generic executable files with a valid shebang

The runtime must continue to scan native `.so` plugins exactly as before, while script artifacts are treated as a second dynamic plugin class.

## Discovery model

The runtime scans each configured plugin directory in the same way it already scans `.so` files:
- iterate the plugin directory
- ignore dotfiles and non-plugin files
- accept supported script plugin files
- normalize the path and ensure it is within the allowed plugin tree
- validate ownership and mode constraints
- exclude untrusted or non-executable files

A script plugin is considered a valid candidate only if:
- it exists in an approved plugin directory
- it is executable
- it declares the expected script plugin interface
- it passes runtime validation rules

## Script plugin contract

Each script plugin must be a self-describing executable that supports the following runtime commands:

- `--list-tools`
  - prints a JSON array or a JSON object containing tool definitions
  - required for registration
- `--describe-tool <tool_name>`
  - optional but recommended for detailed tool metadata lookup
- `--invoke <tool_name> [payload-json]`
  - executes the named tool and prints a JSON response

### Required output format

`--list-tools` must return a JSON object shaped like this:

{
  "tools": [
    {
      "name": "lan_packetcapture",
      "description": "Start a LAN packet capture on brlan0.",
      "command": "tcpdump -i brlan0 ...",
      "schema": "tool-v1"
    }
  ]
}

The runtime must accept at least these fields:
- `name`: tool name string
- `description`: human-readable summary
- `command`: optional command text for static listing / lookup generation
- `schema`: optional version tag; default may be `tool-v1`

For script plugins, `command` is not required to be executed directly by the runtime. It is primarily used for lookup generation and human-facing metadata. The actual execution path is defined by `--invoke`.

## Lifecycle

### 1. Discovery

The runtime scans plugin directories and identifies script plugin files.

### 2. Validation

The runtime validates:
- file path confinement to approved plugin directories
- executor bit is set
- script is readable and executable
- script responds to `--list-tools` successfully
- JSON output is parseable and contains valid tool metadata

### 3. Registration

After a successful `--list-tools`, the runtime registers each returned tool name into the same tool registry used by native `.so` plugins.

A script plugin is registered by binding:
- `tool_name -> script plugin record`
- plugin record includes:
  - script path
  - execution metadata
  - `list-tools` and `invoke` command wrapper
  - active state

### 4. Lookup generation

The runtime creates a lookup table entry for each discovered tool using the same tab-delimited layout already used by the JSON-backed native plugins:

<tool_name>\t<description>\t<command>

This ensures the runtime can resolve tool names for both static JSON entries and script-based dynamically loaded tools.

### 5. Invocation

At runtime, when a tool request arrives, the runtime does the following:
- resolves the tool name through the registry
- identifies that the provider is a script plugin
- executes the plugin process with:
  - `--invoke <tool_name> <payload-json>`
- captures stdout/stderr
- parses a JSON result object
- maps the result into the standard response contract: exit_code, stdout, stderr

## Required invocation response JSON

The script plugin must return JSON in this shape:

{
  "exit_code": 0,
  "stdout": "...",
  "stderr": "",
  "tool": "lan_packetcapture"
}

The runtime must normalize this into the existing `diag_invoke_resp_t` contract:
- `resp->exit_code`
- `resp->out`
- `resp->out_len`

## Security and policy requirements

Script plugins must follow the same runtime policy boundaries as native plugins:
- untrusted file system paths are rejected
- owner and mode checks are enforced
- script files must be copied into a runtime-owned plugin directory
- no direct mutation of the tool registry from external processes
- runtime must reject script plugins that do not return valid JSON or do not respond to `--list-tools`
- script execution should use a restricted environment when possible

## Tool metadata discovery

For script plugins, the runtime resolves tool metadata exclusively from the JSON tool list returned by `--list-tools`.

The runtime should not require or generate a separate `.lookup` file for dynamic plugin discovery or tool registration.

## Compatibility with existing `.so` loading

The script plugin model is intentionally parallel to the `.so` plugin model:

- `.so` plugin: native ABI with `diag_plugin_*` exports
- script plugin: self-describing executable with `--list-tools` and `--invoke`

Both models must satisfy the same runtime requirements:
- tool registration by name
- registry binding
- lookup generation
- invocation by tool name
- response normalization

The runtime may dispatch by provider type:
- `TOOL_PROVIDER_PLUGIN_NATIVE`
- `TOOL_PROVIDER_PLUGIN_SCRIPT`

The registry must treat these as separate provider classes but share the same tool resolution interface.

## Recommended implementation boundary

The minimal implementation in the runtime should:
1. extend the plugin directory scan to include script files
2. add a script plugin adapter layer
3. parse JSON tool metadata from `--list-tools`
4. bind tool names into the registry
5. call `--invoke <tool_name>` for execution
6. normalize JSON result output into the runtime response object

## Non-goals

This specification does not define:
- arbitrary remote code execution
- direct external registration APIs
- ad-hoc mutation of the runtime registry from shell commands
- support for untrusted script paths outside the runtime-owned plugin directory

## Summary

The runtime should accept script-based dynamic tools using the same discovery and registration pattern as native plugins, but through a lightweight executable contract instead of a compiled ABI. Tool metadata remains JSON-based, and the runtime remains the only authority for discovery, policy enforcement, and tool dispatch.
