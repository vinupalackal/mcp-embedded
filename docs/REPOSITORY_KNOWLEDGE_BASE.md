# Repository Knowledge Base

## Components

| Item | Purpose | Evidence | Source location |
| --- | --- | --- | --- |
| Service runtime core | Owns process lifecycle, signal handling, WRP request intake, catalog lookups, and command execution. | README describes the service as owning lifecycle, socket registration, request decoding, dispatch, and responses; CMake builds the `rdk-service-hub` executable from the runtime source set. | [README.md](../README.md), [CMakeLists.txt](../CMakeLists.txt), [src/rdk-service-hub.c](../src/rdk-service-hub.c) |
| Catalog backend | Loads diagnostic tool definitions from JSON and supports an optional LMDB-backed backend when enabled. | The runtime configuration identifies `RDK_SERVICE_HUB_CATALOG_BACKEND=json` and LMDB is compile-time optional; CMake conditionally builds the LMDB import utility and tests. | [config/rdk-service-hub.conf](../config/rdk-service-hub.conf), [CMakeLists.txt](../CMakeLists.txt), [src/catalog_backend.c](../src/catalog_backend.c) |
| Dynamic plugin manager | Loads and tracks native plugin modules and their tool metadata at runtime. | CMake enables `RDK_SERVICE_HUB_ENABLE_DYNAMIC_PLUGINS`, links `CMAKE_DL_LIBS`, and builds plugin manager and tool registry sources when enabled. | [CMakeLists.txt](../CMakeLists.txt), [src/plugin_manager.c](../src/plugin_manager.c), [src/tool_registry.c](../src/tool_registry.c) |
| Capability publication | Publishes catalog/capability metadata into a structured set of JSON definitions for planes and tools. | Capability publication source is part of the runtime and CMake builds capability publication tests. | [CMakeLists.txt](../CMakeLists.txt), [src/capability_publication.c](../src/capability_publication.c), [test/src/capability_publication_unit.c](../test/src/capability_publication_unit.c) |
| Metadata field handling | Serializes and validates metadata fields used in request and response payload processing. | The runtime and tests include metadata field source and unit/vector tests. | [CMakeLists.txt](../CMakeLists.txt), [src/metadata_fields.c](../src/metadata_fields.c), [test/src/metadata_fields_unit.c](../test/src/metadata_fields_unit.c) |

## Services

| Item | Purpose | Evidence | Source location |
| --- | --- | --- | --- |
| `rdk-service-hub` systemd unit | Runs the service under systemd with a fixed runtime directory and service root. | The unit defines `ExecStart=/usr/bin/rdk-service-hub /etc/rdk-service-hub`, `Restart=always`, and `After=network.target parodus.service`. | [systemd/rdk-service-hub.service](../systemd/rdk-service-hub.service) |
| Parodus transport integration | Connects the service to the Parodus message path over nanomsg and WRP traffic. | README identifies the transport layer and the runtime source defines WRP request/registration types and socket operations. | [README.md](../README.md), [src/rdk-service-hub.c](../src/rdk-service-hub.c) |
| Runtime reload path | Accepts runtime configuration updates through signal-driven reload behavior. | The docs call out `SIGHUP` reload support and the runtime contains a `on_sighup` callback. | [docs/README.md](README.md), [src/rdk-service-hub.c](../src/rdk-service-hub.c) |

## Libraries

| Item | Purpose | Evidence | Source location |
| --- | --- | --- | --- |
| nanomsg | Provides the PUSH/PULL transport used to exchange messages with Parodus. | CMake links `nanomsg` directly to the runtime; README describes the nanomsg + WRP transport layer. | [CMakeLists.txt](../CMakeLists.txt), [README.md](../README.md) |
| msgpackc | Encodes and decodes WRP outer envelopes and inner payloads. | CMake links `msgpackc`; runtime source defines WRP message helpers and msgpack packing/unpacking. | [CMakeLists.txt](../CMakeLists.txt), [src/rdk-service-hub.c](../src/rdk-service-hub.c) |
| cJSON | Parses and validates JSON payloads, catalogs, and policy files. | CMake links `cjson`, and runtime config and catalog files are JSON-based. | [CMakeLists.txt](../CMakeLists.txt), [config/rdk-service-hub.conf](../config/rdk-service-hub.conf), [src/catalog_backend.c](../src/catalog_backend.c) |
| pthread | Supports detached per-request worker threads and concurrency patterns. | CMake links `pthread`; README documents the threaded execution model. | [CMakeLists.txt](../CMakeLists.txt), [README.md](../README.md) |
| LMDB | Provides optional catalog persistence in database mode. | `RDK_SERVICE_HUB_ENABLE_CATALOG_LMDB` conditionally enables the library and import utility. | [CMakeLists.txt](../CMakeLists.txt), [src/catalog_lmdb_import.c](../src/catalog_lmdb_import.c) |
| dlopen/dl | Loads dynamic plugin shared objects. | Dynamic plugin support links `CMAKE_DL_LIBS` and installs shared libraries under runtime plugin directories. | [CMakeLists.txt](../CMakeLists.txt), [src/plugin_manager.c](../src/plugin_manager.c) |

## APIs

| Item | Purpose | Evidence | Source location |
| --- | --- | --- | --- |
| CLI runtime entry point | Runs the service binary with optional config and runtime overrides. | `add_executable(rdk-service-hub ...)` and the runtime parses CLI overrides such as `--catalog-backend`, `--load-all-triage-config`, and `--resource-usage-display`. | [CMakeLists.txt](../CMakeLists.txt), [src/rdk-service-hub.c](../src/rdk-service-hub.c) |
| WRP message API | Defines the request/registration/heartbeat message types and encodes/decodes outer WRP envelopes. | The runtime defines `WRP_MSG_TYPE_REQ`, `WRP_MSG_TYPE_REG`, and `WRP_MSG_TYPE_ALIVE` and builds/decodes messages. | [src/rdk-service-hub.c](../src/rdk-service-hub.c) |
| Catalog lookup API | Finds plane-specific capability definitions and validates tool entries within the active catalog. | The runtime includes catalog search and file-recursive lookup logic and uses per-plane directory defaults. | [src/catalog_backend.c](../src/catalog_backend.c), [src/rdk-service-hub.c](../src/rdk-service-hub.c) |
| Plugin provider contract | Defines how dynamic script/native providers expose their metadata and invoke behavior. | Documentation indexes provider contract files covering native scripts, APIs, and dynamic tool registration. | [docs/README.md](README.md), [docs/guides/NATIVE_PLUGIN_PROVIDER_CONTRACT.md](guides/NATIVE_PLUGIN_PROVIDER_CONTRACT.md), [docs/guides/SCRIPT_PLUGIN_PROVIDER_CONTRACT.md](guides/SCRIPT_PLUGIN_PROVIDER_CONTRACT.md) |
| Config/env override API | Reads environment variables and runtime config defaults to override runtime behavior without recompilation. | The config file includes `RDK_SERVICE_HUB_*` variables and the runtime reads config and env override values. | [config/rdk-service-hub.conf](../config/rdk-service-hub.conf), [src/rdk-service-hub.c](../src/rdk-service-hub.c) |

## Build system

| Item | Purpose | Evidence | Source location |
| --- | --- | --- | --- |
| CMake project definition | Declares the project and its build options for runtime, plugins, metadata, and tests. | `project(rdk-service-hub C)` and the option block defines feature toggles such as dynamic plugins, ACL, LMDB, and test builds. | [CMakeLists.txt](../CMakeLists.txt) |
| Native executable build | Produces the `rdk-service-hub` runtime binary from the source list, compile definitions, libraries, and include directories. | CMake creates `add_executable(rdk-service-hub ${RDK_SERVICE_HUB_SOURCES})` and links the required libraries. | [CMakeLists.txt](../CMakeLists.txt) |
| Optional LMDB build path | Adds a separate import utility and DB-mode tests only when LMDB support is enabled. | `if(RDK_SERVICE_HUB_ENABLE_CATALOG_LMDB)` branches build `rdk-service-hub-catalog-lmdb-import` and LMDB tests. | [CMakeLists.txt](../CMakeLists.txt) |
| Sample plugin build | Builds sample plugin shared libraries and copies them into staged and runtime plugin directories. | CMake defines `rdk-service-hub-sample-plugin` and `rdk-service-hub-network-traffic-plugin` with `POST_BUILD` copy steps. | [CMakeLists.txt](../CMakeLists.txt) |
| CTest test registration | Registers runtime and spec tests using `enable_testing()` and `add_test()`. | The CMake file registers Yocto regression checks, requirement tests, plugin tests, metadata tests, dynamic plugin tests, and guardrail tests. | [CMakeLists.txt](../CMakeLists.txt) |

## Configuration files

| Item | Purpose | Evidence | Source location |
| --- | --- | --- | --- |
| Main runtime config | Defines runtime layout, backend selection, logging, plugin discovery, catalog paths, and command policy defaults. | The config file sets catalog directory, plugin manifest, plugin paths, logging flags, and policy-related variables. | [config/rdk-service-hub.conf](../config/rdk-service-hub.conf) |
| Command allowlist policy | Restricts executable and argument patterns via policy files for safe and unsafe command classes. | The config points to `command_allowlist*.json` and `command_denylist_admin_unsafe.json` files and describes plane-based policy selection. | [config/rdk-service-hub.conf](../config/rdk-service-hub.conf), [command_allowlist.json](../command_allowlist.json), [command_allowlist_diagnostic.json](../command_allowlist_diagnostic.json), [command_denylist_admin_unsafe.json](../command_denylist_admin_unsafe.json) |
| Capability catalogs | Stores plane-specific capability and tool catalog JSON files for triage, management, control, and config-apply. | CMake installs the JSON catalogs under `/etc/rdk-service-hub/capabilities/...`; the config file points to them. | [capabilities/](../capabilities), [config/rdk-service-hub.conf](../config/rdk-service-hub.conf), [CMakeLists.txt](../CMakeLists.txt) |
| Example metadata | Provides optional example metadata for runtime demonstration and validation. | CMake and README document the optional example directory under `/etc/rdk-service-hub/examples`. | [CMakeLists.txt](../CMakeLists.txt), [README.md](../README.md), [docs/README.md](README.md) |
| Service unit config | Pairs the service binary with the config root and declares systemd behavior. | The unit sets `ExecStart=/usr/bin/rdk-service-hub /etc/rdk-service-hub` and default environment override comment. | [systemd/rdk-service-hub.service](../systemd/rdk-service-hub.service) |

## Test framework

| Item | Purpose | Evidence | Source location |
| --- | --- | --- | --- |
| CTest harness | Runs binary-based test executables and reports pass/fail or skip status. | CMake declares `enable_testing()` and registers named tests through `add_test()`. | [CMakeLists.txt](../CMakeLists.txt) |
| Requirements/spec tests | Validates requirement and spec compliance for behavior under a compiled binary. | `rdk-service-hub-requirements-spec-tests` is defined in CMake and executed via CTest. | [CMakeLists.txt](../CMakeLists.txt), [test/src/requirements_spec_tests.c](../test/src/requirements_spec_tests.c) |
| Metadata tests | Exercises metadata field encoding/decoding and msgpack vector behavior. | Separate tests exist for unit, flow vectors, and msgpack vectors, all registered in CMake. | [CMakeLists.txt](../CMakeLists.txt), [test/src/metadata_fields_unit.c](../test/src/metadata_fields_unit.c), [test/src/metadata_flow_vectors.c](../test/src/metadata_flow_vectors.c), [test/src/metadata_msgpack_vectors.c](../test/src/metadata_msgpack_vectors.c) |
| Plugin tests | Covers dynamic plugin utility behavior, live plugin loading, lock-scope behavior, and integration paths. | CMake defines plugin unit, live, lock-scope, and integration test binaries; the docs and test names align with runtime plugin lifecycle behavior. | [CMakeLists.txt](../CMakeLists.txt), [test/src/dynamic_plugins_unit.c](../test/src/dynamic_plugins_unit.c), [test/src/dynamic_plugins_live.c](../test/src/dynamic_plugins_live.c), [test/src/plugin_integration_runner.c](../test/src/plugin_integration_runner.c) |
| Guardrail/feature tests | Validates baseline guardrails, WRP behavior, and capability publication snapshots. | CMake registers phase 1 guardrail, feature matrix spec, and capability publication tests. | [CMakeLists.txt](../CMakeLists.txt), [test/src/feature_matrix_spec_tests.c](../test/src/feature_matrix_spec_tests.c), [test/src/phase1_baseline_guardrails.c](../test/src/phase1_baseline_guardrails.c) |

## Deployment artifacts

| Item | Purpose | Evidence | Source location |
| --- | --- | --- | --- |
| Installed runtime binary | Publishes the service binary to the install prefix’s `bin` directory. | `install(TARGETS rdk-service-hub DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)` is configured in CMake. | [CMakeLists.txt](../CMakeLists.txt) |
| Systemd package unit | Installs the Linux service definition into `/lib/systemd/system`. | CMake installs `src/systemd/rdk-service-hub.service` to `/lib/systemd/system`. | [CMakeLists.txt](../CMakeLists.txt), [systemd/rdk-service-hub.service](../systemd/rdk-service-hub.service) |
| Runtime output tree | Prepares a local output tree with `bin`, config, capabilities, and runtime plugin directories. | The build defines output directories under `output/`, `output/etc/rdk-service-hub`, and `output/var/lib/rdk-service-hub/plugins`. | [CMakeLists.txt](../CMakeLists.txt) |
| Sample plugin artifacts | Installs sample `.so` plugins under the runtime plugin directory tree. | CMake copies sample plugin binaries into runtime-owned plugin directories and creates staging directories. | [CMakeLists.txt](../CMakeLists.txt), [plugins/](../plugins) |
| Yocto packaging | Includes a Yocto sample plugin regression test using the recipe path. | CMake conditionally defines a test based on `yocto/recipes-diagnostics/rdk-service-hub/rdk-service-hub_1.0.bb`. | [CMakeLists.txt](../CMakeLists.txt), [yocto/](../yocto) |

## Security controls

| Item | Purpose | Evidence | Source location |
| --- | --- | --- | --- |
| Allowlist and denylist policy | Restricts commands by explicit allowlist and destructive denylist classes. | The config points to `command_allowlist*.json` and `command_denylist_admin_unsafe.json`; README documents blocklists and program validation. | [README.md](../README.md), [config/rdk-service-hub.conf](../config/rdk-service-hub.conf), [command_allowlist.json](../command_allowlist.json), [command_denylist_admin_unsafe.json](../command_denylist_admin_unsafe.json) |
| Blocklisted command protection | Rejects dangerous executables such as `rm`, `reboot`, `shutdown`, `kill`, `dd`, and similar destructive commands. | README enumerates the blocked command list and runtime code checks the first executable token. | [README.md](../README.md), [src/rdk-service-hub.c](../src/rdk-service-hub.c) |
| Output size cap | Limits command output capture to a bounded size to prevent excess resource consumption. | README states the 64 KiB stdout cap and the runtime enforces command output size boundaries. | [README.md](../README.md), [src/rdk-service-hub.c](../src/rdk-service-hub.c) |
| Local-only push gate | Restricts push traffic to local-only mode by default and supports an optional auth token. | The config file defines `RDK_SERVICE_HUB_PUSH_REQUIRE_LOCAL_ONLY=1` and optional `RDK_SERVICE_HUB_PUSH_AUTH_TOKEN`. | [config/rdk-service-hub.conf](../config/rdk-service-hub.conf) |
| Resource guardrails | Applies CPU, memory, disk, and process constraints for pre-execution safety checks. | The config file sets `RDK_SERVICE_HUB_RESOURCE_*` values and `RDK_SERVICE_HUB_RESOURCE_FAIL_MODE=deny`. | [config/rdk-service-hub.conf](../config/rdk-service-hub.conf) |
| Static vs. dynamic command policy | Distinguishes static catalog commands from dynamic commands and enforces different validation behavior. | The README records the static/dynamic tool distinction and the runtime handles tool validation paths accordingly. | [README.md](../README.md), [src/rdk-service-hub.c](../src/rdk-service-hub.c) |

## Evidence summary

This repository is a C-based diagnostic runtime with a native executable target, optional dynamic plugin support, JSON-based capability catalogs, systemd packaging, and CTest-driven validation. The authoritative sources for the repository-level model are the top-level build script, the runtime configuration file, the systemd unit, and the documentation index in [docs/README.md](README.md).
