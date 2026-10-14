# RDK Service Hub

RDK Service Hub is a lightweight diagnostic execution service for RDK CPE devices. It integrates with Parodus over nanomsg, receives WRP requests, executes a catalog-defined diagnostic command, and returns command output in a WRP response.

This repository currently contains:
- `rdk-service-hub.c`: Main service implementation.
- `rdk-service-hub-*-catalog.json`: Per-plane catalog JSON files (triage example + management/control/config-apply templates).
- `CMakeLists.txt`: Build configuration.

## Documentation Guides

- Full docs index: [docs/README.md](docs/README.md)
- Architecture overview: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)
- Requirements specification: [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md)
- Repository knowledge base: [docs/REPOSITORY_KNOWLEDGE_BASE.md](docs/REPOSITORY_KNOWLEDGE_BASE.md)

## Canonical runtime layout

The runtime uses one explicit layout everywhere it reads live assets from the filesystem:

- runtime-owned plugin directory: `/var/lib/rdk-service-hub/plugins/...`
- capability/config directory: `/etc/rdk-service-hub/capabilities`
- optional example metadata directory: `/etc/rdk-service-hub/examples`
- runtime config file: `/etc/rdk-service-hub/rdk-service-hub.conf`

This is the authoritative layout for runtime discovery, plugin loading, and capability reads. Packaging trees under `/usr/lib` may exist for install staging, but they are not the live runtime-owned root.

## Current Runtime Model

The runtime is intentionally conservative by default:

- catalog backend default: `json`
- LMDB support is optional and compile-time gated, and remains off by default
- the active catalog lookup path uses a compact tmpfs-backed index plus lazy loading instead of a full in-memory catalog snapshot
- process startup continues normally even when no runtime arguments are supplied
- command-policy guardrails default to safe/off behavior unless explicitly enabled
- `--resource-usage-display <0|1>` prints a one-time `PROCESS_RESOURCE_USAGE` snapshot only when enabled
- runtime reload is supported via `SIGHUP` for safe re-application of configuration without process restart

This keeps the daemon operational on constrained targets while allowing explicit opt-in to stricter or richer runtime modes.

> **Corrections applied 2026-08-14** — the review that found these issues
> concluded that: (1) the build/source filename mismatch this document
> previously warned about did not actually exist in `CMakeLists.txt` —
> that file already built `rdk-service-hub.c` correctly, and the warning
> below has been removed. (2) `CMakeLists.txt` linked `wrp-c`,
> `libparodus`, and `cimplog`, none of which the code calls into (it
> talks to Parodus over raw nanomsg directly) — those unused
> dependencies have been removed from the build configuration and from
> the prerequisites list below. (3) Catalog `timeout` is now enforced —
> see §1.4 and §2.2 below; the "not currently used" gap language has
> been removed. (4) Command execution no longer uses a shell at all
> (`/bin/sh -c` removed, not just `popen()`) — see §1.4 and §3.5
> below. (5) `is_command_safe()` now also pins a caller override's
> program to the catalog's declared program, unless the tool's catalog
> entry opts out via `"type": "dynamic"` — see §1.4 item 4 and §3.5.
> (6) Every catalog tool now also declares `"type"` and `"plane"` —
> see §3.5. (7) Every tool's static catalog command is
> now validated once, at process startup, instead of on every request
> — see §1.4 item 6 and §2.5 (new).

## 1. Architecture Guide

### 1.1 High-Level Components

1. Service Runtime (`rdk-service-hub.c`)
- Owns process lifecycle and signal handling.
- Loads diagnostic tool catalog at startup.
- Registers itself with Parodus.
- Receives and decodes WRP traffic.
- Dispatches diagnostic requests to worker threads.
- Sends responses back to Parodus.

2. Catalog (`rdk-service-hub-triage-catalog.json`)
- Defines allowed tools and default shell commands.
- Example structure:
  - `tools.<tool-name>.command`
  - `tools.<tool-name>.timeout` — enforced (corrected 2026-08-14): used
    as the per-tool wall-clock ceiling; falls back to a 30s default
    when absent.

3. Transport Layer (nanomsg + WRP over msgpack)
- PUSH socket to Parodus at `tcp://127.0.0.1:6666`.
- PULL socket bound locally at `tcp://127.0.0.1:6669`.
- WRP registration type: 9.
- WRP request/response type: 3.
- WRP keepalive type: 10.

### 1.2 Data and Control Flow

```mermaid
flowchart LR
    A[OPS Gateway] --> B[Parodus]
    B -->|WRP REQ type=3| C[RDK Service Hub PULL 127.0.0.1:6669]
    C --> D[Decode outer WRP msgpack]
    D --> E[Decode inner payload msgpack]
    E --> F[Tool lookup in catalog]
    F --> G[Safety gate blocked command check]
    G --> H[Execute via fork/exec, timeout-enforced]
    H --> I[Build inner response payload]
    I --> J[Build outer WRP response type=3]
    J --> K[RDK Service Hub PUSH 127.0.0.1:6666]
    K --> B
    B --> A
```

### 1.3 Threading Model

- Main thread:
  - Initializes sockets and registration.
  - Runs receive loop on PULL socket.
  - Handles keepalive (type 10) inline.
- Worker threads:
  - Each WRP request (type 3) is processed in a detached pthread.
  - This prevents long command execution from blocking socket receive.

### 1.4 Security and Safety Controls

1. Allowlist by catalog
- Requested `tool` must exist in catalog.

2. Blocklist by first token
- The executable token of the command is checked against blocked commands:
  - `rm`, `rmdir`, `reboot`, `shutdown`, `halt`, `poweroff`, `factory_reset`,
  - `kill`, `killall`, `pkill`, `dd`, `mkfs`, `fdisk`, `mount`, `umount`, `iptables`, `passwd`.

3. Output size cap
- Command stdout capture is limited to 64 KiB.

4. Program pinning for command overrides (added 2026-08-14; **superseded
   for static tools 2026-08-16, see item 5 below**)
- `is_command_safe()` is the single, common check that both a
  catalog-default command and a caller-supplied `command` override go
  through before execution — it's not two separate checks in two
  places, one call site enforces both.
- On top of the blocklist (item 2), it requires that a caller-supplied
  override's program (its first argument token) be identical to the
  program the catalog itself declares for that tool. An override can
  still change arguments (e.g. request a different ping target), but
  can't redirect a tool to run an entirely different, non-blocklisted
  program.
- **As of 2026-08-16, this whole check only ever runs for `"dynamic"`
  tools** — see item 5. It's kept, not deleted, but its single call site
  in `handle_request()` no longer reaches it for static tools at all.
- Historical residual (closed 2026-08-16 for static tools, see item 5):
  this pinned the *program*, not the *arguments* — an override naming
  the same program with different arguments (e.g. pointing
  `device_uptime`'s `cat` at a different file) used to still execute.
  See `REQUIREMENTS.md` §12 Risk item 5.

5. Static vs. dynamic tools (added 2026-08-14; static-override behavior
   changed 2026-08-16)
- A catalog tool entry declares `"type": "static"` (the default; every
  pre-existing tool) or `"type": "dynamic"` (see the `adhoc_diagnostic`
  example in `rdk-service-hub-triage-catalog.json`).
- **As of 2026-08-16:** a static tool no longer honors a caller-supplied
  `command` override at all. Whatever
  the catalog declares as that tool's `command` always runs, whether or
  not the request includes an override, and regardless of the
  override's content — item 4's program-pin check isn't even reached
  for a static tool anymore, since there's nothing left to validate.
  This fully closes the "same program, different arguments" gap noted
  in item 4, rather than narrowing it. A request with an override for a
  static tool now logs an informational note (`LOG_INFO`) that the
  override was ignored, and proceeds using the catalog command.
- For a **dynamic** tool, nothing changed: the caller is expected to
  supply the command outright, item 4's program-pin is skipped by
  design (no fixed program to pin against), and the blocklist (item 2)
  still applies unconditionally.
- `"plane"` is a separate, informational field (`config-apply`,
  `management`, `control`, or `triage`) categorizing the tool for the
  broader toolset metadata. rdk-service-hub logs it but does not act
  on it.

6. Init-time static command validation and the "skipped" category
   (added 2026-08-14)
- Every tool's own catalog `command` (not a caller override — see
  below) is validated exactly once, right after the catalog loads at
  process startup, before either transport socket exists and before
  any request can possibly arrive. See §2.5.
- A tool whose static command fails this check (blocklisted, or
  unparseable — e.g. empty/whitespace-only) is marked **skipped**: it
  is rejected for the rest of the process's lifetime, for *any*
  request naming it — including one that supplies its own,
  independently-safe override command. A skipped tool doesn't get a
  second chance through a clever override; the whole entry is taken
  out of service. This is deliberately a blanket exclusion, not a
  narrower "only the default-command path is blocked" rule.
- For a tool that passes validation, its program name is cached once
  (as `argv[0]` of its `command`) and reused for every subsequent
  program-pin comparison (item 4) — no request re-tokenizes or
  re-blocklist-checks the catalog's own command string, since it
  cannot have changed since startup (there is no catalog hot-reload
  anywhere in this code). Only a caller-supplied override — whose
  content genuinely cannot be known ahead of time — still runs a live
  check per request.
- Every skip is logged individually at startup (`LOG_WARNING`) with
  the tool name, its command, and the reason, plus a one-line summary
  (`static command validation: N checked, M skipped`) — so a
  misconfigured catalog is visible in syslog at service start, not
  discovered later only when something happens to invoke the broken
  tool.

7. Current gaps to know
- ~~Catalog `timeout` field is not currently used to terminate
  commands.~~ Fixed 2026-08-14 — enforced with a 30s default fallback;
  see §2.2.
- ~~Command execution uses `popen`/a shell, so shell expansion and
  redirection are possible.~~ Fixed 2026-08-14 — execution is now
  `execvp()` with a tokenized argument vector (`tokenize_argv()`); no
  shell is invoked anywhere in the path. `command` strings (catalog or
  caller-supplied) are split into literal argv tokens, not interpreted
  — see §3.5/§3.6 and NFR-17 in `REQUIREMENTS.md`.
- ~~A caller-supplied `command` field overrides the catalog entry and is
only checked by first token~~ — this was a real command-injection path,
not just a style concern. **Narrowed in two  stages, both 2026-08-14**: (1) since there's no shell anymore, a
  compound override like `cat /proc/uptime; rm -rf /tmp/x` no longer
  runs `rm` — the whole string becomes literal arguments to `cat`,
  which just fails; (2) `is_command_safe()` (item 4 above) additionally
  rejects an override naming a *different program* than the catalog
  declares for that tool — for **static** tools. A **dynamic** tool
  (item 5 above) intentionally has no program to pin, so this
  narrowing doesn't apply to it; that's the tradeoff of declaring a
  tool dynamic, not a bug. **Closed 2026-08-16, for static tools**: an
  override naming the *same* program with different arguments than
  intended (e.g. redirecting `device_uptime`'s `cat /proc/uptime` to
  `cat /etc/shadow`) no longer executes at all — static tools stop
  honoring overrides entirely (item 5 above), rather than restricting
  them to a catalog-declared parameter set. Dynamic tools are
  unaffected.
- ~~There is no access-control check of any kind on who may invoke a
  tool.~~ Narrowed 2026-08-15 — `rdksh_acl_check()` gates every request
in `handle_request()`; see §4.1's ACL gate bullet above. Still
Phase-2-gated: `acl_policy_store_query()`'s transport isn't implemented
yet, so the  check compiles and is wired in but can't link into a runnable binary
  until that's chosen.

## 2. Workflow Guide

### 2.1 Startup Workflow

1. Process starts and installs signal handlers (`SIGTERM`, `SIGINT`).
2. **Changed 2026-08-15:** catalogs are loaded per plane, not from one file:
- Default directory: `/etc/rdk-service-hub`
- Override: first CLI argument (now a *directory*, not a file path).
- Each of `rdk-service-hub-triage-catalog.json`, `rdk-service-hub-management-catalog.json`,
  `rdk-service-hub-control-catalog.json`, `rdk-service-hub-config-apply-catalog.json` is
  loaded from that directory if present; a plane with no file there is
  simply not served (not an error).
3. **Added 2026-08-14**: every tool's static catalog `command` is
   validated once (blocklist + parseability), *before* any socket is
   opened — see §2.5. Tools that fail are marked skipped for the rest
   of the process's life; tools that pass get their program name
   cached for fast override checks later.
4. Service binds PULL socket at `tcp://127.0.0.1:6669`.
5. Service connects PUSH socket to Parodus at `tcp://127.0.0.1:6666` with retry/backoff.
6. Sends WRP registration message with:
- `msg_type = 9`
- `service_name = rdk-service-hub`
- `url = tcp://127.0.0.1:6669`

### 2.2 Request Processing Workflow

1. Receive outer WRP map.
2. Accept only request messages (`msg_type = 3`) for tool execution.
3. Decode inner payload map:
- `tool` (required)
- `command` (optional)
4. **Added 2026-08-14**: if the tool was marked skipped at startup
   (§2.5), reject immediately here — regardless of whether this
   request supplies its own `command` — and skip steps 5–6 below.
5. Resolve command:
- If payload command is empty, fallback to catalog command (already
  proven safe at startup — no re-check needed).
- If payload command is present (an override), apply the blocklist and
  program-pin check (§1.4 item 4) live, using the cached program name
  from step 3 of §2.1.
6. Execute allowed command.
7. Build inner payload response:
- `tool`
- `exit_code`
- `stdout` (binary field)
8. Wrap into outer WRP response and send to requester.

### 2.3 Keepalive Workflow

- If incoming WRP has `msg_type = 10`, service immediately returns type 10 ack.

### 2.4 Shutdown Workflow

1. Signal sets runtime flag to stop loop.
2. Sockets are closed.
3. Catalog memory is released.
4. Syslog is closed.

### 2.5 Init-Time Static Command Validation (added 2026-08-14)

Design goal (as requested): call the safety check for each tool's
*static* catalog command during process initialization itself, so
runtime request handling never repeats work whose answer cannot have
changed since startup.

**Why this is safe to do once**: the catalog loads exactly once, in
`load_catalog()`, and nothing in this codebase ever reloads it while
the process runs — there's no hot-reload path, no `SIGHUP` handler for
it, nothing. A tool's own declared `command` string is therefore a
fixed value for the entire life of the process. Checking it against
the blocklist (or confirming it tokenizes to at least one argument) on
every single request that happens to use the catalog default was pure
repeated work with a guaranteed-identical answer every time.

**Sequence, exactly as it runs in `main()`**:

```
load_catalogs()                   -- parses each plane's catalog file into g_planes[] (§15 B.5)
validate_static_commands()        -- NEW: walks every tool, once, per plane
  for each tool with a non-empty "command":
    tokenize it
      -> fails (empty/unparseable)?      mark "_skipped", reason recorded
      -> is_blocked() on the full string? mark "_skipped", reason recorded
      -> otherwise: cache argv[0] as "_program_token" on the entry
  log each skip individually (LOG_WARNING) + a one-line summary
-- only now are the PULL/PUSH sockets created --
nn_socket(...) / nn_bind(...) / nn_connect(...)
-- only now can a request possibly arrive --
```

A tool with no `command` field at all (a `"type": "dynamic"` tool with
no fallback default, like `adhoc_diagnostic`) has nothing to validate
here and is left untouched — neither skipped nor given a
`_program_token`, exactly as it behaved before this change.

**What "marked as skipped" means for `handle_request()`**: a skipped
tool is rejected for *every* request naming it, for the rest of the
process's life — including a request that supplies its own,
independently-safe `command` override. This is a deliberate,
whole-tool exclusion: a catalog entry whose own declared command is
dangerous or malformed signals a misconfigured catalog, and the safer
default is to take the entry out of service entirely rather than leave
a path where a well-chosen override could still reach it. (Verified:
see the standalone-harness test that injects a deliberately
blocklisted tool, confirms it gets marked skipped with the right
reason, and confirms an independently-safe override sent against it is
*still* rejected — not just the tool's own broken default.)

**What still runs at request time, and why it has to**: only a
caller-supplied override's live content — which cannot be known ahead
of time — still goes through `is_command_safe()` per request. That
check itself got cheaper too: it now takes the tool's cached
`_program_token` instead of the raw catalog command string, so it no
longer re-tokenizes the (unchanging) catalog side on every call — only
the override's own string gets tokenized, since that's the only piece
that's actually new each time.

### 3.1 Prerequisites

Runtime/build dependencies inferred from code and CMake:
- Compiler with C11 support.
- CMake >= 3.10.
- pthread.
- nanomsg.
- msgpack-c.
- cJSON.

### 3.2 Build

```bash
cmake -S . -B build
cmake --build build
```

The repository includes a comprehensive test suite covering the feature set
and validation paths described in this document.

### 3.2.1 Requirement-spec test harness (added 2026-08-19)

An executable test harness now covers the newly added requirement test
cases for dynamic plugin loading and metadata-field handling:

- Source: `tests/requirements_spec_tests.c`
- CMake option: `RDK_SERVICE_HUB_BUILD_REQUIREMENTS_TESTS` (default `ON`)
- CTest name: `rdk-service-hub-requirements-spec-tests`

Build/run only the harness target (useful on systems without full
runtime dependencies like nanomsg headers):

```bash
cmake -S . -B build
cmake --build build --target rdk-service-hub-requirements-spec-tests
ctest --test-dir build -R rdk-service-hub-requirements-spec-tests --output-on-failure
```

### 3.2.2 Plugin integration runner (added 2026-08-19)

Runtime plugin test cases that need a live environment are split into
a dedicated integration runner:

- Source: `tests/plugin_integration_runner.c`
- CMake option: `RDK_SERVICE_HUB_BUILD_PLUGIN_INTEGRATION_TESTS` (default `ON`)
- CTest name: `rdk-service-hub-plugin-integration-tests`
- Covers: `TC-PLUG-002`, `TC-PLUG-003`, `TC-PLUG-004`, `TC-PLUG-009`

The runner is environment-driven and uses per-case command hooks.
If integration mode is not enabled, it exits with skip (CTest handles
this as skipped, not failed).

Environment variables:

- `RDK_SERVICE_HUB_PLUGIN_IT_ENABLE=1` (required to execute integration mode)
- `RDK_SERVICE_HUB_PLUGIN_DIR=/path/to/plugin/dir` (optional context)
- `RDK_SERVICE_HUB_IT_TC_PLUG_002_CMD='<command>'`
- `RDK_SERVICE_HUB_IT_TC_PLUG_003_CMD='<command>'`
- `RDK_SERVICE_HUB_IT_TC_PLUG_004_CMD='<command>'`
- `RDK_SERVICE_HUB_IT_TC_PLUG_009_CMD='<command>'`

Run:

```bash
cmake -S . -B build
cmake --build build --target rdk-service-hub-plugin-integration-tests
ctest --test-dir build -R rdk-service-hub-plugin-integration-tests --output-on-failure
```

One-command CI helper:

```bash
./tests/run_plugin_integration_ci.sh
```

The helper sets default pass-through hooks (`true`) so CI wiring can be
validated even before a full plugin runtime fixture exists. Override the
`RDK_SERVICE_HUB_IT_TC_PLUG_*_CMD` variables with real integration commands
to execute actual scenario checks.

### 3.2.3 Metadata field feature flags and unit tests (added 2026-08-19)

Metadata-field support is now implemented with compile-time and runtime
gates:

- CMake option: `RDK_SERVICE_HUB_ENABLE_METADATA_FIELDS` (default `ON`)
- Runtime enable switch: `RDK_SERVICE_HUB_METADATA_ENABLE`
  - `1`/`true`/`on`: enable metadata validation + policy
  - `0`/`false`/`off`: ignore metadata (legacy behavior)
- Runtime mode: `RDK_SERVICE_HUB_METADATA_MODE`
  - `compat` (default): type/conflict/policy checks with tolerant enums
  - `strict`: additionally enforces `plane`/`plane_type` allowlists
- Optional response echo: `RDK_SERVICE_HUB_METADATA_ECHO=1`
  - Adds `metadata_applied` to EXEC response payload.

Standalone unit test target:

- Target: `rdk-service-hub-metadata-unit-tests`
- CTest name: `rdk-service-hub-metadata-unit-tests`

Metadata request-flow vector target (synthetic request vectors covering
compat/strict modes and policy outcomes):

- Target: `rdk-service-hub-metadata-flow-vectors`
- CTest name: `rdk-service-hub-metadata-flow-vectors`

Metadata msgpack byte-level vector target (packs raw WRP/payload maps,
decodes metadata fields, then validates policy outcomes):

- Target: `rdk-service-hub-metadata-msgpack-vectors`
- CTest name: `rdk-service-hub-metadata-msgpack-vectors`

Note: this target is always built. With `msgpack.h` present it runs the
native msgpack path; otherwise it runs a built-in fallback
serializer/parser path.

Run:

```bash
cmake -S . -B build
cmake --build build --target rdk-service-hub-metadata-unit-tests rdk-service-hub-metadata-flow-vectors rdk-service-hub-metadata-msgpack-vectors
ctest --test-dir build -R 'rdk-service-hub-metadata-(unit-tests|flow-vectors|msgpack-vectors)' --output-on-failure
```

### 3.2.4 Dynamic plugin support (updated 2026-10-01)

Dynamic plugin loading is implemented with an rpcd-style model and now supports both compiled shared objects and trusted script-based plugins:
- native plugins are `.so` files loaded via `dlopen`/`dlsym`
- script plugins are executable files discovered in the same runtime-owned plugin directories
- each plugin exports or advertises a fixed tool contract (see [src/plugin_api.h](src/plugin_api.h) and [src/plugin_manager.c](src/plugin_manager.c))
- one plugin may register one or more tool names
- request dispatch resolves plugin tool first (or rejects/overrides per conflict policy),
  then falls back to catalog execution path.

Current runtime examples:
- [plugins/triage/shared/packet_capture_rdk_service_hub_plugin.json](plugins/triage/shared/packet_capture_rdk_service_hub_plugin.json) declares `lan_packetcapture` and `wan_packetcapture`
- [plugins/triage/script/system_performance_plugin.sh](plugins/triage/script/system_performance_plugin.sh) exposes script-driven performance tools via `--list-tools` and `--invoke`

Build flag:
- `RDK_SERVICE_HUB_ENABLE_DYNAMIC_PLUGINS` (default `ON`)

Runtime flags:
- `RDK_SERVICE_HUB_CONFIG_FILE` (optional `.conf` path override)
- `RDK_SERVICE_HUB_CATALOG_DIR` (default `/etc/rdk-service-hub/capabilities`)
- `RDK_SERVICE_HUB_CATALOG_FILE_TRIAGE` (default `rdk-service-hub-triage-catalog.json`)
- `RDK_SERVICE_HUB_CATALOG_FILE_MANAGEMENT` (default `rdk-service-hub-management-catalog.json`)
- `RDK_SERVICE_HUB_CATALOG_FILE_CONTROL` (default `rdk-service-hub-control-catalog.json`)
- `RDK_SERVICE_HUB_CATALOG_FILE_CONFIG_APPLY` (default `rdk-service-hub-config-apply-catalog.json`)
- `RDK_SERVICE_HUB_PLUGIN_ENABLE` (`0`/`1`, default `0`)
- `RDK_SERVICE_HUB_PLUGIN_DIR` (default `/var/lib/rdk-service-hub/plugins`)
- `RDK_SERVICE_HUB_PLUGIN_DIR_TRIAGE` (optional; default `/var/lib/rdk-service-hub/plugins/triage`)
- `RDK_SERVICE_HUB_PLUGIN_DIR_MANAGEMENT` (optional; shared runtime data-management dir)
- `RDK_SERVICE_HUB_PLUGIN_DIR_CONTROL` (optional; shared runtime data-management dir)
- `RDK_SERVICE_HUB_PLUGIN_DIR_CONFIG_APPLY` (optional; shared runtime data-management dir)
- `RDK_SERVICE_HUB_PLUGIN_DISCOVERY_MODE` (`notify`, `poll`, or `hybrid`; default `hybrid`)
- `RDK_SERVICE_HUB_PLUGIN_POLL_INTERVAL_SEC` (periodic reconcile interval; default `60`)
- `RDK_SERVICE_HUB_PLUGIN_DEBOUNCE_MS` (file stability debounce before load; default `200`)
- `RDK_SERVICE_HUB_PLUGIN_CONFLICT_POLICY` (`reject-plugin-tool` default, or `plugin-priority`)
- `RDK_SERVICE_HUB_PLUGIN_VERIFY_MODE` (`off` in v1; other values logged as unsupported)

Per-plane plugin directory behavior:
- If any `RDK_SERVICE_HUB_PLUGIN_DIR_<PLANE>` key is set, the runtime uses
  a combined directory list from the plane-specific keys and scans all configured plane paths.
- If none of the plane-specific keys are set, `RDK_SERVICE_HUB_PLUGIN_DIR` is used.
- Runtime-owned plugin discovery is constrained to trusted plugin directories under `/var/lib/rdk-service-hub/plugins` and the project test staging area under `./output/staged_plugins/<plane>`.
- Script and `.so` plugin artifacts are both expected to live in the same runtime-managed directory tree so they can be scanned and bound to the tool registry consistently.

Reload behavior:
- `RDK_SERVICE_HUB_PLUGIN_DISCOVERY_MODE=hybrid` (default):
  - Linux/OpenWrt: notify-based event handling (`inotify`) + periodic reconcile scans.
  - Other targets: falls back to poll mode with periodic reconcile.
- `RDK_SERVICE_HUB_PLUGIN_DISCOVERY_MODE=notify`:
  - Linux/OpenWrt: notify-based auto reload (`inotify`).
  - Other targets: watcher setup may fail; prefer `hybrid` or `poll` for portability.
- `RDK_SERVICE_HUB_PLUGIN_DISCOVERY_MODE=poll`:
  - periodic directory scanning only.
- `RDK_SERVICE_HUB_PLUGIN_POLL_INTERVAL_SEC` controls reconcile/scan cadence for `poll` and `hybrid`.
- `RDK_SERVICE_HUB_PLUGIN_DEBOUNCE_MS` reduces partial-write load risk by delaying load until file metadata is stable.

Daemon-style manual reload:
- `kill -HUP <rdk-service-hub-pid>` triggers plugin directory rescan/reload once.

Default config file:
- [rdk-service-hub.conf](rdk-service-hub.conf)

Load precedence:
1. Environment variable value (if set)
2. Config file value (`RDK_SERVICE_HUB_CONFIG_FILE`, else `/etc/rdk-service-hub/rdk-service-hub.conf`, else `./rdk-service-hub.conf`)
3. Built-in defaults

Plugin errors map to deterministic response tokens:
- `ERR_PLUGIN_INVOKE` (`exit_code=70`)
- `ERR_PLUGIN_UNAVAILABLE` (`exit_code=71`)
- `ERR_PLUGIN_API_VERSION` (`exit_code=72`)

Plugin authoring guide:
- [plugins/README.md](plugins/README.md)

Live plugin integration test targets:
- `rdk-service-hub-sample-plugin` (sample `.so`)
- `rdk-service-hub-dynamic-plugins-live-tests`

Default build/install plugin artifact:
- CMake option: `RDK_SERVICE_HUB_BUILD_SAMPLE_PLUGIN` (default `ON`)
- Installs sample plugin to `/var/lib/rdk-service-hub/plugins/triage`

### 3.2.5 ACL compile-time gate (added 2026-08-19)

ACL API usage is now fully compile-time gated:

- CMake option: `RDK_SERVICE_HUB_ENABLE_ACL` (default `OFF`)
- When `OFF`, ACL types/calls (`caller_identity_t`, `acl_policy_store_query()`, `rdksh_acl_check()` usage)
  are compiled out.
- When ACL implementation is ready, enable with:
  `-DRDK_SERVICE_HUB_ENABLE_ACL=ON`

### 3.2.6 Runtime log file controls (added 2026-08-19)

rdk-service-hub logging is now runtime-configurable using `rdk-service-hub.conf`
with environment-variable overrides.

Defaults:

- `RDK_SERVICE_HUB_LOG_ENABLE=1`
- `RDK_SERVICE_HUB_LOG_FILE_ENABLE=1`
- `RDK_SERVICE_HUB_LOG_FILE_PATH=/tmp/logs/rdk-service-hub.log`
- `RDK_SERVICE_HUB_LOG_TRACE_API=1`
- `RDK_SERVICE_HUB_LOG_TRACE_DATA_IO=1`

Behavior:

- `RDK_SERVICE_HUB_LOG_ENABLE=0` disables all rdk-service-hub logging output.
- `RDK_SERVICE_HUB_LOG_FILE_ENABLE=1` writes logs to `RDK_SERVICE_HUB_LOG_FILE_PATH`
  in addition to syslog.
- `RDK_SERVICE_HUB_LOG_TRACE_API` controls API/functionality tracing logs.
- `RDK_SERVICE_HUB_LOG_TRACE_DATA_IO` controls request/response data I/O logs.

Override precedence is the same as other runtime settings:

1. Environment variable (highest priority)
2. Config file (`RDK_SERVICE_HUB_CONFIG_FILE`, else `/etc/rdk-service-hub/rdk-service-hub.conf`, else `./rdk-service-hub.conf`)
3. Built-in defaults

### 3.2.7 Feature-matrix spec tests (added 2026-08-19)

Broad executable-spec coverage is available for config/logging precedence,
kind routing, push result signaling, multi-plane lookup resolution,
and core safety guardrails.

- Target: `rdk-service-hub-feature-matrix-spec-tests`
- CTest: `rdk-service-hub-feature-matrix-spec-tests`
- CMake option: `RDK_SERVICE_HUB_BUILD_FEATURE_MATRIX_SPEC_TESTS` (default `ON`)

Run:

```bash
cmake -S . -B build
cmake --build build --target rdk-service-hub-feature-matrix-spec-tests
ctest --test-dir build -R rdk-service-hub-feature-matrix-spec-tests --output-on-failure
```

### 3.2.8 Linux container build, run, and test

Container files included:
- `Dockerfile`
- `docker-compose.yml`
- `scripts/container-ci.sh`
- `scripts/container-smoke-daemon.sh`

Build the Linux dev image:

```bash
docker build -t rdk-service-hub:dev .
```

Run full container CI (build + CTest suite):

```bash
docker compose run --rm rdksh-ci
```

Open an interactive Linux container:

```bash
docker compose run --rm rdksh-dev bash
```

Inside container, run:

```bash
./scripts/container-ci.sh
./scripts/container-smoke-daemon.sh
```

What this validates:
- dependency resolution (`nanomsg`, `msgpack`, `cjson`)
- full CMake configure/build path
- unit/spec/live plugin tests via CTest
- daemon startup/shutdown smoke path using local `.conf` and per-plane catalog JSON files

Notes:
- `rdk-service-hub-plugin-integration-tests` may report as skipped when external runtime hooks are not provisioned.
- ACL-enabled end-to-end behavior requires an ACL backend implementation and should be tested separately once available.

**Build-time options (added 2026-08-16):** two of `rdk-service-hub.c`'s
compiled-in behavior flags are now exposed as CMake cache options,
`#ifndef`-guarded in the source so the defaults below match today's
source exactly when left untouched:

| CMake option | Default | Controls |
|---|---|---|
| `RDK_SERVICE_HUB_REGISTER_WITH_PARODUS` | `ON` | `REGISTER_WITH_PARODUS` — send the WRP type-9 registration to Parodus at startup |
| `RDK_SERVICE_HUB_PUSH_REQUIRE_LOCAL_ONLY` | `OFF` | `PUSH_REQUIRE_LOCAL_ONLY` — restrict catalog PUSH to the local-only endpoint |

```bash
cmake -S . -B build -DRDK_SERVICE_HUB_REGISTER_WITH_PARODUS=OFF -DRDK_SERVICE_HUB_PUSH_REQUIRE_LOCAL_ONLY=ON
```

See `rdk-service-hub_1.0.bb`'s `PACKAGECONFIG` entries for the Yocto-level
equivalent of these two flags, and each `#define`'s own comment in
`rdk-service-hub.c` for the security tradeoffs behind their current
defaults.

### 3.2.9 Runtime startup options (added 2026-09-30)

The service also accepts runtime overrides as CLI flags. When no flags are
provided, the process starts using the default config values and continues running
normally.

Example commands:

```bash
./build/rdk-service-hub
./build/rdk-service-hub /path/to/catalog/dir
./build/rdk-service-hub \
  --command-policy-default off \
  --command-policy-plane-control admin_unsafe \
  --resource-check-enable 0 \
  --resource-cpu-max-pct 90
```

Supported flags:

```text
--catalog-dir <path>
--command-policy-default <off|diagnostic|admin_unsafe>
--command-policy-plane-triage <off|diagnostic|admin_unsafe>
--command-policy-plane-management <off|diagnostic|admin_unsafe>
--command-policy-plane-control <off|diagnostic|admin_unsafe>
--command-policy-plane-config-apply <off|diagnostic|admin_unsafe>
--command-allowlist-file <path>
--command-allowlist-diagnostic-file <path>
--command-denylist-admin-unsafe-file <path>
--resource-check-enable <0|1>
--resource-cpu-max-pct <pct>
--resource-mem-free-mb <mb>
--resource-disk-free-mb <mb>
--resource-max-procs <count>
--resource-fail-mode <deny|allow>
--timeout-default-sec <seconds>
--timeout-extra-margin-sec <seconds>
--help
```

The same values are also available from the config file
`rdk-service-hub.conf`, and the CLI flags override those values at
startup time.

### 3.3 Install

```bash
cmake --install build
```

By default, target installs to:
- `${CMAKE_INSTALL_PREFIX}/bin/rdk-service-hub`
- `/etc/rdk-service-hub/rdk-service-hub.conf`
- `/etc/rdk-service-hub/rdk-service-hub-triage-catalog.json`
- `/etc/rdk-service-hub/rdk-service-hub-management-catalog.json`
- `/etc/rdk-service-hub/rdk-service-hub-control-catalog.json`
- `/etc/rdk-service-hub/rdk-service-hub-config-apply-catalog.json`

### 3.4 Prepare Catalog

**Changed 2026-08-15 (§15 B.5)**: one catalog file per plane, not a
single `catalog.json`.

1. Place per-plane catalog file(s) at the default directory:
- `/etc/rdk-service-hub/rdk-service-hub-triage-catalog.json`,
  `/etc/rdk-service-hub/rdk-service-hub-management-catalog.json`,
  `/etc/rdk-service-hub/rdk-service-hub-control-catalog.json`,
  `/etc/rdk-service-hub/rdk-service-hub-config-apply-catalog.json` — only the planes
  actually populated are served; the rest are simply absent.
2. Or pass a custom catalog *directory* as argument.

Example start command:

```bash
./rdk-service-hub /path/to/catalog/dir
```

### 3.5 Catalog Format

Minimum format:

```json
{
  "tools": {
    "device_uptime": {
      "command": "cat /proc/uptime",
      "timeout": 5,
      "type": "static",
      "plane": "triage"
    },
    "active_connections": {
      "command": "/bin/netstat -n",
      "suppress_stderr": true,
      "count_lines_matching": "ESTABLISHED",
      "timeout": 5,
      "type": "static",
      "plane": "triage"
    },
    "adhoc_diagnostic": {
      "timeout": 15,
      "type": "dynamic",
      "plane": "triage"
    }
  }
}
```

Notes:
- `command` is the executable string. **Corrected 2026-08-14 (NFR-17):**
  this is now tokenized directly into an argument vector for `execvp()`
  — there is no shell in the execution path. It must be a program path
  or name plus its arguments, split on whitespace (a double-quoted
  segment counts as one token, e.g. `foo "arg with spaces"`). Shell
  syntax — pipes (`|`), redirects (`>`, `2>`), command separators
  (`;`, `&&`), backticks, `$()`, globs — has no effect; it's passed
  through as literal argument text, not interpreted. Optional for a
  `"type": "dynamic"` tool (see below) — if omitted, the caller must
  supply a `command` in the request or nothing runs.
- `timeout` is enforced: per-tool wall-clock ceiling in seconds. If
  absent or non-positive, a 30s default applies.
- `suppress_stderr` (boolean, optional, added 2026-08-14): redirects
  the command's stderr to `/dev/null` — replaces a shell command
  string's former inline `2>/dev/null`.
- `count_lines_matching` (string, optional, added 2026-08-14): if set,
  the response's stdout becomes the decimal count of the command's own
  output lines containing this substring, followed by a newline —
  replaces a shell command string's former inline
  `| grep <substring> | wc -l`.
- `type` (string, optional, added 2026-08-14): `"static"` (the
  default) or `"dynamic"`. A static tool's program is pinned to its
  own `command` (§1.4 item 4) even if a caller override changes
  arguments. A dynamic tool has no program pin — the caller is
  expected to supply the command — but the blocklist (§1.4 item 2)
  still applies to it unconditionally.
- `plane` (string, optional, added 2026-08-14): one of `config-apply`,
  `management`, `control`, `triage` — categorization metadata for the
  wider RDK Dispatcher project, not consumed by rdk-service-hub's own
  logic (see §1.4 item 5).

### 3.6 Request and Response Payloads

Inner payload request (msgpack map):

```json
{
  "tool": "device_uptime",
  "command": "",
  "plane": ""
}
```

- If `command` is omitted/empty, catalog command is used.
- `plane` (string, optional, **added 2026-08-15, §15 B.5** — not to be
  confused with the catalog entry's own `plane` metadata field, §3.5
  above): if supplied, the tool is looked up *only* in that plane's
  catalog — a miss there is a miss, it does not fall through to search
  other planes. If omitted (every request before this change, and any
  caller not yet updated to send it), rdk-service-hub searches every loaded
  plane's catalog for the name; if that name exists in more than one
  plane, the request is rejected as ambiguous rather than guessed at —
  a plane-qualified request for either one still resolves normally.

Inner payload response (msgpack map):

```json
{
  "tool": "device_uptime",
  "exit_code": 0,
  "stdout": "<binary stdout bytes>"
}
```

The above (EXEC) is the only shape this service spoke until 2026-08-15.
**Added 2026-08-15:** four more message kinds ride this same
inner-payload/outer-WRP-envelope
convention, disambiguated by an optional `"kind"` string field EXEC
itself never sends:

- **DESCRIBE** — `{"kind":"DESCRIBE", "plane": ""}` (`plane` optional)
  → one plane's `{plane, version, tools:[{name,type,plane,timeout}]}`,
  or (no `plane` given) an array of that shape per currently-loaded
  plane.
- **HEALTH** — `{"kind":"HEALTH"}` → `{"status":"ok", "push_guardrails": {...}, "catalog_db": {...}}`.
  Side-effect-free; includes guardrail posture/counters and catalog backend
  observability (backend mode, LMDB readiness, generation, cache/reload stats).
- **PUSH** — `{"kind":"PUSH", "plane", "base_version", "target_version", "diff": {"added":{},"removed":[],"modified":{}}}`
  → `{"status":"loaded","plane","version"}` or
  `{"status":"rejected","plane","reason"}`. `base_version` must match
  the target plane's *current* live version exactly (compare-and-swap,
  not "newer than") or the push is rejected before anything else is
  attempted. On success, the promoted catalog is also persisted to disk
  (§15 B.2.5) before the response is sent.
- **CHANGED** — unsolicited, sent by rdk-service-hub itself right after a
  successful PUSH promote: `{"kind":"CHANGED","plane","version"}`.

These are local-protocol additions, not part of the frozen external
wire contract (§3) — they exist for the same connection EXEC already
answers on, and none of EXEC's own two original fields or byte shape
changed.

### 3.7 Observability

- Logging uses syslog with ident `rdk-service-hub`.
- Typical logs include:
  - startup and registration
  - received request UUID/source
  - executed command
  - exit code and response send status
  - connection retry errors

### 3.8 Troubleshooting

1. No responses from service
- Verify Parodus is listening on `127.0.0.1:6666`.
- Verify service bound `127.0.0.1:6669`.
- Check registration log event in syslog.

2. Tool not found
- Ensure requested `tool` key exists under catalog `tools`.

3. Command blocked or missing
- Requested/fallback command may be blocklisted.
- Verify command starts with an allowed executable token.
- **Added 2026-08-14**: a `command` override is also rejected if its
  program doesn't match the catalog entry's own declared program for
  that `tool` — check syslog for `unsafe command rejected for tool
  '<tool>'` to distinguish this from a plain blocklist hit (see §1.4
  item 4).

4. Output truncated
- Stdout is capped at 64 KiB by design.

5. Response has `exit_code: 124` and output `"command timed out after
   Ns"` (added 2026-08-14)
- The command exceeded its timeout (catalog `timeout` field, or the
  30s default if the catalog doesn't declare one) and was killed.
- Not an error in rdk-service-hub itself — check whether the underlying
  command is expected to take that long, and raise the catalog
  `timeout` value for that tool if so.

6. A catalog `command` that used to work now fails, or a new tool with
   a pipe/redirect in its command never produces the expected output
   (added 2026-08-14)
- Command execution no longer goes through a shell (NFR-17, §3.5) —
  `command` is tokenized into literal arguments, so `|`, `>`, `2>`,
  `;`, `&&`, backticks, and `$()` are not interpreted; they're passed
  as plain text to whatever program is `argv[0]`, which will usually
  just fail on the unexpected argument.
- If the tool needs a stderr redirect, use `suppress_stderr: true`
  instead of `2>/dev/null` in the command string.
- If the tool needs to count matching output lines (the
  `| grep X | wc -l` pattern), use `count_lines_matching: "X"` instead.
- Anything else that genuinely needs shell features (globbing,
  variable expansion, multi-stage pipelines beyond a single count
  filter) currently has no catalog-level equivalent — file it as a gap
  rather than trying to smuggle shell syntax back into `command`.

7. Response has `exit_code: 1` and output starting with `"tool skipped
   at init:"`, and this happens for *every* request against that tool,
   even ones with their own `command` (added 2026-08-14)
- The tool's own catalog `command` failed init-time validation (§2.5)
  — it's either blocklisted or unparseable. Check syslog at service
  startup for `catalog validation: tool '<tool>' marked skipped at
  init (<reason>): command='<command>'`.
- This is not a per-request issue and cannot be worked around by
  sending a different `command` in the request — the whole tool is
  excluded until the catalog is fixed and the service is restarted
  (there's no hot-reload; a catalog fix only takes effect on the next
  process start).
- Fix the catalog entry's `command` field and restart the service.

## 4. Developer Notes

### 4.1 Key Constants

- Parodus URL: `tcp://127.0.0.1:6666`
- Client bind URL: `tcp://127.0.0.1:6669`
- Local endpoint (added 2026-08-15, §15 B.4 part 1 — additive, alongside
  the public pair above, best-effort at startup): recv
  `ipc:///run/dispatcher/diagnostics-in.sock`, send
  `ipc:///run/dispatcher/diagnostics-out.sock`. PUSH was originally
  restricted to this local endpoint only (a PUSH received via the
  public pair was rejected outright). **Phase 3 hardening (2026-08-20)**:
  default posture is secure-by-default local-only (`PUSH_REQUIRE_LOCAL_ONLY=1`),
  with optional shared-token authorization
  (`RDK_SERVICE_HUB_PUSH_AUTH_TOKEN`) when stronger gatekeeping
  is required. DESCRIBE/HEALTH/EXEC are reachable on both pairs, as before;
  EXEC alone is ACL-gated per tool (see `rdksh_acl_check()` above).
- Registration to Parodus (§15 B.4 part 2): `REGISTER_WITH_PARODUS` was
  briefly flipped to `0` on 2026-08-16 (after D.1/D.3 both passed) so
  rdk-service-hub would stop sending its WRP type-9 registration, making
  the local endpoint its only practically reachable address — the
  intended end state of §10.2 Option A, where Dispatch Core owns the
  public registration and forwards ACL-checked requests locally.
  **Reverted the same day, by direct instruction** — registration is
  needed at rdk-service-hub startup again, so `REGISTER_WITH_PARODUS` is
  back to `1` and rdk-service-hub registers with Parodus directly, as
  before. The public PULL/PUSH sockets were never unbound either way
  (outbound traffic like `capability_sync.updated` was unaffected
  throughout). **Caveat this revert reopens**: with registration back
  on, rdk-service-hub is directly reachable from Parodus/the cloud again,
  and `acl_policy_store_query()` (the function `rdksh_acl_check()`
  depends on) still has no implementation anywhere in this codebase —
  so there is currently no working ACL enforcement on that public path.
  See the `REGISTER_WITH_PARODUS` `#define`'s own comment in
  `rdk-service-hub.c` for the full history.
- ACL gate (added 2026-08-15, §13.4): every EXEC request now passes
  through `rdksh_acl_check()` before catalog lookup — denial returns
  `exit_code=126`/`stdout="access denied"`. Thin wrapper around
  `acl_policy_store_query()`, the same entry point every other toolset
  already uses; that function has no implementation anywhere in this
  codebase yet (transport still unresolved, Phase 2), so this code
  compiles clean but won't link into a runnable binary until that's
  chosen elsewhere.
- Capability-sync (added 2026-08-15, §13.4): a successful PUSH promote
  now also sends a `capability_sync.updated` JSON-RPC notification over
  the public Parodus connection, alongside the existing local-only
  CHANGED notification.
- Default catalog directory: `/etc/rdk-service-hub` (one file per plane — §15 B.5)
- Receive timeout: 2000 ms
- Send timeout: 5000 ms
- Max captured stdout: 65536 bytes
- Default per-tool execution timeout: 30s (corrected 2026-08-14;
  catalog-overridable via `tools.<name>.timeout`)
- Static command validation: runs once, at startup, before any socket
  exists (added 2026-08-14; see §2.5)

### 4.2 Known Improvement Areas

1. ~~Enforce per-tool execution timeout from catalog.~~ Done
   2026-08-14 — see §1.4/§2.2/§3.5.
2. ~~Move from shell-based execution (`/bin/sh -c`) to safer
   `execve`-style invocation without shell interpretation.~~ Done
   2026-08-14 — `run_command()` now calls `execvp()` with an argument
   vector built by `tokenize_argv()`; no shell is invoked anywhere.
   `active_connections`, the one catalog entry that relied on a shell
   pipe, was rewritten using the new `suppress_stderr`/
   `count_lines_matching` catalog fields (§3.5) instead.
3. ~~Add stricter argument-level validation for command overrides.~~
   **Done, but via the other named alternative, not argument-level
   validation itself.** 2026-08-14: `is_command_safe()` (§1.4 item 4)
   first pinned an override's *program* to the catalog's declared
   program, closing the "run an entirely different program" half of
   the gap. **2026-08-16 (§9 Q3, §1.4 item 5)**: rather than adding a
   per-tool allowed-argument schema, static tools stop honoring
   overrides at all — the catalog's own command always runs. This
   closes the remaining "same program, different arguments" gap
   completely for static tools, via the "drop overrides entirely"
   alternative rather than the "argument-level allowlist" alternative;
   dynamic tools are unaffected, since caller-supplied commands are
   their whole purpose.
4. ~~Add an access-control check before executing any tool.~~ Done
   2026-08-15 — see §1.4 item 7 above. Still Phase-2-gated on the ACL
   Policy Store's transport.
5. Add unit tests for msgpack encoding/decoding.
6. Add integration test harness with mock Parodus endpoint.

## 5. File Map

- `rdk-service-hub.c`: main daemon logic (transport, decoding, execution, response).
- `rdk-service-hub-triage-catalog.json`: sample diagnostics catalog.
- `CMakeLists.txt`: build, link, and install configuration.
