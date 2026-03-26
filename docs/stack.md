# FEAST Stack — Architecture Reference

## Stack Map

```
External consumers  (HTTP REST /v0/..., dashboards, scripts)
        │
anolis  (C++17 — ProviderRegistry, DeviceRegistry, StateCache, CallRouter, HttpServer, BTRuntime)
        │  ADPP  (protobuf + uint32_le framing over stdio)
        ├── anolis-provider-sim     (simulated devices — physics + fault injection)
        └── anolis-provider-bread   (real BREAD hardware — Linux only w/ ENABLE_HARDWARE)
                │  crumbs::Session  (C++ session layer — retry, mutex, query_read)
                │  CRUMBS  (C — I2C framing, CRC-8, controller/peripheral roles)
                │  /dev/i2cN  (linux-wire)
                BREAD hardware  (RLHT / DCMT MCUs on I2C bus)

Shared at compile time:
  bread-crumbs-contracts  →  type IDs, opcodes, payload layouts, caps schema, version helpers
  anolis-protocol         →  ADPP protobuf schema (git submodule in each consuming repo)
```

---

## Layers

### CRUMBS

**Job:** I2C transport for embedded microcontrollers (Arduino/PlatformIO) and Linux native hosts.
Encode/decode variable-length framed messages (4–31 bytes, 0–27 byte payload), CRC-8 protection,
controller/peripheral role model, per-opcode handler dispatch.

**Platforms:** Arduino and compatible MCUs (Arduino IDE + PlatformIO `cameronbrooks11/CRUMBS@^0.12.0`);
Linux hosts via linux-wire. The core C library compiles with no platform dependency — HAL is
injected via fn-pointer structs.

**Invariants:**

- CRUMBS has zero knowledge of BREAD, RLHT, DCMT, or any payload semantics.
- HAL is entirely fn-pointer based — the core C library compiles with no platform includes.
- `CRUMBS_VERSION` enables a single `>=` integer comparison across the version chain.
- `CRUMBS_MAX_HANDLERS` is a compile-time constant. PlatformIO consumers set it via
  `build_flags = -DCRUMBS_MAX_HANDLERS=N`.

---

### bread-crumbs-contracts

**Job:** BREAD application layer on top of CRUMBS. Defines the wire contract for all BREAD Slice
family devices: type IDs, per-device opcodes, payload byte layouts, capability discovery schema,
version helpers. Header-only C.

**Type IDs:** `0x01` = RLHT, `0x02` = DCMT. Type IDs are stable — capability variations do not get new IDs. See [devices.md](devices.md) for per-device opcodes, payload layouts, and capability flag details.

**Invariants:**

- Capabilities are **additive only** — new features add bits; existing bit assignments never change.
  No type-ID splits for capability variations.
- `bread_check_crumbs_compat` requires `CRUMBS_VERSION >= 1200` (v0.12.0).
- The `rlht_get_state` / `dcmt_get_state` C helpers use `crumbs_device_t` fn-pointers suited for
  embedded controllers. **C++ providers must not use these helpers** — they bypass `crumbs::Session`
  and cannot be unit-tested without hardware. Use `session.query_read()` and parse frame payload
  bytes directly.
- Headers include `crumbs.h`. All consumers must have CRUMBS headers on their include path. This
  transitive dependency is not yet propagated by the CMake target.

---

### anolis-provider-bread

**Job:** ADPP hardware provider for BREAD devices. Owns the CRUMBS session lifecycle, runs device
discovery and compatibility checking, builds the ADPP capability inventory, and serves all ADPP
RPCs over stdio + uint32_le framing.

**Module structure** (`src/`):

| Module                                        | Responsibility                                                                                      |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `config/`                                     | YAML config loading: bus path, discovery mode, device specs, timeouts, retry settings               |
| `core/handlers.cpp`                           | All ADPP RPC handlers: Hello, WaitReady, ListDevices, DescribeDevice, ReadSignals, Call, GetHealth  |
| `core/health.cpp`                             | `GetHealth` / `WaitReady` responses from `RuntimeState`                                             |
| `core/runtime_state.cpp`                      | Process-global state (config, device list, session ptr); protected by mutex                         |
| `core/startup.cpp`                            | `probe_device()` and `run_discovery()` — version query, compat check, caps query, baseline fallback |
| `core/transport/framed_stdio.*`               | uint32_le length-prefix read/write loop                                                             |
| `crumbs/session.hpp/.cpp`                     | `Transport` ABC + `Session` — `open()`, `scan()`, `send()`, `query_read()`, retry, mutex            |
| `crumbs/linux_transport.hpp/.cpp`             | `LinuxTransport : Transport` — real Linux I2C via CRUMBS Linux HAL (hardware builds only)           |
| `devices/common/adapter_helpers.hpp`          | Shared payload helpers, error mapping (`read_error_from_session`, `call_error_from_session`)        |
| `devices/common/inventory.hpp/.cpp`           | `InventoryDevice`, capability builders, signal/function existence checks                            |
| `devices/common/bread_compatibility.hpp/.cpp` | Probe result evaluation, `provider_type_id()` string generation                                     |
| `devices/rlht/rlht_adapter.hpp/.cpp`          | RLHT: `read_signals()` parses 19-byte GET_STATE frame; `call()` dispatches 6 functions              |
| `devices/dcmt/dcmt_adapter.hpp/.cpp`          | DCMT: `read_signals()` parses mode-conditional GET_STATE frame; `call()` dispatches 5 functions     |
| `logging/logger.hpp/.cpp`                     | Stderr logger with `[INFO]`/`[WARN]`/`[ERROR]` prefixes                                             |

**Build modes:**

| Preset                                 | Hardware | Platform                           |
| -------------------------------------- | -------- | ---------------------------------- |
| `dev-foundation-debug/release`         | Off      | Any (Linux, Windows)               |
| `dev-windows-foundation-debug/release` | Off      | Windows MSVC                       |
| `dev-linux-debug/release`              | On       | Linux only                         |
| `ci-foundation-release`                | Off      | Linux (CI)                         |
| `ci-linux-release`                     | On       | Linux (manual hardware validation) |

`ANOLIS_PROVIDER_BREAD_ENABLE_HARDWARE=ON` requires Linux. Foundation builds compile and test the
full CRUMBS session layer and adapters using `ScriptedTransport` with no real hardware.

**Discovery modes:**

| Mode     | Mechanism                                                  |
| -------- | ---------------------------------------------------------- |
| `scan`   | I2C address probe 0x08–0x77; probe each responding address |
| `manual` | Explicit address list from config; probe each              |

Both paths run through `probe_device()` → `build_inventory_from_probes()`. Config `devices:` entries
provide labels and expected IDs; missing expected devices surface as `DEGRADED` provider health.

**Capability gating:** `build_rlht_capabilities(flags)` / `build_dcmt_capabilities(flags)` gate
which `FunctionSpec` and `SignalSpec` entries appear in `DescribeDevice`. This happens once at
inventory-build time — adapters do not re-check flags per call.

---

### anolis-provider-sim

**Job:** Simulated ADPP provider. Implements the full provider contract over simulated devices with
optional physics. Used for development, integration testing, and validation without hardware. Also
serves as the canonical reference implementation of the ADPP provider contract.

**Device roster** (config-driven, multiple instances per type):

| Device type     | Signals                                                                            | Functions                               | Physics                                             |
| --------------- | ---------------------------------------------------------------------------------- | --------------------------------------- | --------------------------------------------------- |
| `tempctl`       | `tc1_temp`, `tc2_temp`, `relay1_state`, `relay2_state`, `control_mode`, `setpoint` | `set_mode`, `set_setpoint`, `set_relay` | First-order thermal lag τ=6s, bang-bang closed-loop |
| `motorctl`      | `motor1_speed`, `motor2_speed`, `motor1_duty`, `motor2_duty`                       | `set_motor_duty`                        | First-order motor lag τ=0.8s toward duty×max_rpm    |
| `relayio`       | relay state signals                                                                | relay control                           | None                                                |
| `analogsensor`  | analog reading signals                                                             | None                                    | None                                                |
| `chaos_control` | None                                                                               | fault injection functions               | None                                                |

**Simulation engines:**

| `simulation.mode` | Engine         | Physics                                                               |
| ----------------- | -------------- | --------------------------------------------------------------------- |
| `inert`           | `NullEngine`   | Perfectly static state; reads always return configured initial values |
| `non_interacting` | `LocalEngine`  | In-process first-order lag models; no cross-device coupling           |
| `sim`             | `RemoteEngine` | External FluxGraph simulation (`ENABLE_FLUXGRAPH=ON` build option)    |

**Signal routing:** `SignalRegistry` provides cross-device physics coordination. Physics ticks
write values by `"device_id/signal_id"` path (e.g. `"tempctl0/tc1_temp"`). Device `read_signals()`
implementations check the registry and use physics values when present, falling back to internal
state otherwise.

**Fault injection** (via `chaos_control` ADPP device — standard `Call` RPC):

| Function                                            | Effect                                   |
| --------------------------------------------------- | ---------------------------------------- |
| `inject_device_unavailable(device_id)`              | `ReadSignals` returns `CODE_UNAVAILABLE` |
| `inject_signal_fault(device_id, signal_id)`         | Signal reported with `QUALITY_FAULT`     |
| `inject_call_latency(device_id, delay_ms)`          | `Call` sleeps before responding          |
| `inject_call_failure(device_id, function_id, code)` | `Call` returns specified error code      |
| `clear_faults(device_id)`                           | Remove all injected faults for device    |

Fault injection is exercised through the normal ADPP `Call` path — no special test-only protocol.

**Per-device module pattern** (the contract all providers follow):

```
init(device_id, config)
get_capabilities()           → CapabilitySet   (static, capability-flag-gated)
read_signals(device_id, ids) → []SignalValue
call_function(device_id, function_id, args)  → CallResult
```

---

### anolis

**Job:** Runtime kernel. Spawns and supervises provider processes, speaks ADPP to discover all
devices and their capabilities, maintains a live state cache, routes all control calls through a
single validated path, exposes a unified HTTP REST API, and optionally drives behavior-tree
automation.

**Core subsystems:**

| Subsystem                          | Responsibility                                                                                       |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `ProviderRegistry`                 | Maps provider ID → `IProviderHandle`; thread-safe                                                    |
| `ProviderHost` + `ProviderProcess` | Spawn/kill subprocess, manage stdio pipes, frame ADPP messages                                       |
| `ProviderSupervisor`               | Crash detection, exponential-backoff restart, circuit breaker                                        |
| `DeviceRegistry`                   | Immutable post-discovery capability inventory; returns by-value (no dangling pointers)               |
| `StateCache`                       | Polls `is_default` signals at 500ms; tracks timestamps, quality, staleness; thread-safe snapshot API |
| `CallRouter`                       | Validates request → per-provider mutex → execute → post-call poll; single path for all callers       |
| `EventEmitter`                     | Fan-out to per-subscriber bounded queues; slow subscribers do not block `StateCache`                 |
| `HttpServer`                       | REST `/v0/...`; 40-thread pool; localhost bind by default                                            |
| `InfluxSink` (opt.)                | Async batched telemetry to InfluxDB v2 → Grafana dashboards                                          |
| `ModeManager` (opt.)               | IDLE / MANUAL / AUTO / FAULT state machine                                                           |
| `BTRuntime` (opt.)                 | BehaviorTree.CPP engine; ticks at configured Hz; runs only in AUTO mode                              |
| `ParameterManager` (opt.)          | Config-tunable setpoints and limits accessible from BT nodes and HTTP                                |

**Provider supervision lifecycle:**

```
RUNNING → RECOVERING → RESTARTING → CIRCUIT_OPEN → DOWN
              ↑_______________↓
```

Circuit breaker opens after `max_attempts` restarts without recovery. Provider enters DOWN state.
anolis continues running — other providers are not affected.

**Mode state machine:**

| Transition     | Allowed                |
| -------------- | ---------------------- |
| IDLE ↔ MANUAL  | ✓                      |
| MANUAL ↔ AUTO  | ✓                      |
| Any → FAULT    | ✓ (automatic on error) |
| FAULT → MANUAL | ✓ (explicit recovery)  |
| FAULT → AUTO   | ✗                      |
| FAULT → IDLE   | ✗                      |
| AUTO → IDLE    | ✗                      |

Mode is always IDLE at startup — not configurable. Use `POST /v0/mode` to transition.

**HTTP API endpoints** (`/v0/`):

| Endpoint                        | Method     | Purpose                                   |
| ------------------------------- | ---------- | ----------------------------------------- |
| `/v0/devices`                   | GET        | List all devices across all providers     |
| `/v0/state`                     | GET        | All current signal values from StateCache |
| `/v0/state/{provider}/{device}` | GET        | Single-device state snapshot              |
| `/v0/call`                      | POST       | Dispatch a function call via CallRouter   |
| `/v0/runtime/status`            | GET        | Runtime health summary                    |
| `/v0/providers/health`          | GET        | Per-provider and per-device health        |
| `/v0/mode`                      | GET / POST | Read or set mode                          |
| `/v0/parameters`                | GET / POST | Read or update parameter values           |
| `/v0/automation/tree`           | GET        | Behavior tree structure and status        |

**Invariants:**

- All control calls go through `CallRouter.execute_call()` — HTTP handlers, BT nodes, and tests use
  the same path. No bypass.
- `DeviceRegistry` returns by-value copies — no pointer invalidation when providers restart.
- `StateCache` is the only source of signal state for all consumers. No direct provider read
  bypasses the cache.
- HTTP API is localhost-only by default (v0). No authentication. Must be addressed before
  networked deployment.
- `BTRuntime` uses only `{state_cache, call_router, provider_registry, parameter_manager}` — no
  new protocol paths or provider bypasses.

---

## Interfaces

### ADPP (Anolis Device Provider Protocol)

ADPP is the standard provider interface. Any process that speaks ADPP over stdio can be managed
by anolis without changes to anolis. The schema lives in `anolis-protocol`, vendored as a git
submodule in each consuming repo.

**Framing:** each message is a 4-byte little-endian length header followed by protobuf bytes:

```
[message_length : uint32_le (4 bytes)]  [protobuf_bytes : variable]
```

**RPCs:**

| RPC              | Required | Purpose                                                   |
| ---------------- | -------- | --------------------------------------------------------- |
| `Hello`          | Yes      | Protocol handshake, version check, provider metadata      |
| `ListDevices`    | Yes      | Enumerate all managed devices                             |
| `DescribeDevice` | Yes      | Return `Device` metadata + `CapabilitySet` for one device |
| `ReadSignals`    | Yes      | Read current values for a set of signals from a device    |
| `Call`           | Yes      | Dispatch a function call to a device                      |
| `GetHealth`      | Optional | Provider and per-device health diagnostics                |
| `WaitReady`      | Optional | Block until provider has completed initialization         |

`supports_wait_ready` in `HelloResponse.metadata` declares whether `WaitReady` is implemented.

`CallRequest` carries both `function_id` (u32) and `function_name` (string). anolis resolves by
name from the `DeviceRegistry` and sends both; providers should honor either.

**Key schema types:**

- `Device` — `device_id`, `type_id` (namespaced string, e.g. `"bread.rlht"`), `type_version`, `label`, `address`, `tags`
- `CapabilitySet` — `FunctionSpec[]` + `SignalSpec[]`
- `FunctionSpec.policy.category` — `READ` / `CONFIG` / `ACTUATE` (intent classification used by CallRouter)
- `SignalSpec.poll_hint_hz` — non-zero causes `StateCache` to auto-poll at that rate
- `SignalSpec.stale_after_ms` — staleness threshold for quality tracking
- `SignalValue.quality` — `OK` / `STALE` / `FAULT` / `UNKNOWN`

**Key coupling:** `provider_id` (anolis config `id:` key) is the routing identifier. `provider_name` in `HelloResponse` is the provider's self-reported label and is not used for routing. See [adpp.md](adpp.md) for the full ADPP schema and status code reference.

---

### crumbs::Session

C++ session layer over the C CRUMBS library, owned by `anolis-provider-bread`.

```
Transport (ABC)            — open, close, scan, send, read, delay_us
  LinuxTransport           — real Linux I2C via CRUMBS Linux HAL  (hardware builds only)
  ScriptedTransport        — deterministic scripted responses      (unit tests, any platform)

Session(transport, opts)   — retry wrapper, per-session mutex
  open() / close()
  scan(opts, &results)     — I2C address probe (wraps CRUMBS scan)
  send(addr, frame)        — fire-and-forget
  query_read(addr, reply_opcode, &out)  — send + wait for reply with retry
```

Session options: `bus_path`, `query_delay_us` (default 10000 µs), `timeout_ms` (default 100),
`retry_count` (default 2).

All adapter code uses `session.query_read()` and `session.send()` exclusively — never the CRUMBS
C API directly.

---

### bread-crumbs-contracts headers

Compile-time dependency. CRUMBS headers must be on the include path of any consumer. CMake
consumers add CRUMBS includes manually until exported targets propagate this transitively.

---

## Version / Compatibility Chain

```
bread_check_crumbs_compat(crumbs_version from opcode 0x00 reply)
  → pass if crumbs_version >= BREAD_MIN_CRUMBS_VERSION (1200 = v0.12.0)

bread_check_module_compat(module_major, module_minor, required_major, required_minor)
  → pass if major matches AND minor >= required minimum

probe_device() → ProbeStatus
  Supported | IncompatibleCrumbsVersion | IncompatibleModuleMajor | IncompatibleModuleMinor
  | VersionReadFailed | UnsupportedTypeId

build_inventory_from_probes()
  → excludes incompatible/unsupported with diagnostics; continues with compatible subset
  → GET_CAPS per supported device → baseline fallback (per-type policy) on caps query failure

ADPP DescribeDevice
  → anolis DeviceRegistry (immutable after discovery)

StateCache (polls) / CallRouter (routes)
```

---

## Dependency Rules

```
anolis
  → anolis-protocol   (git submodule external/anolis-protocol)
  → vcpkg: protobuf, yaml-cpp, cpp-httplib, nlohmann-json, gtest, BehaviorTree.CPP (opt), openssl

anolis-provider-sim
  → anolis-protocol   (same submodule pattern)
  → vcpkg: protobuf, yaml-cpp, gtest, BehaviorTree.CPP (opt), cpp-httplib

anolis-provider-bread
  → anolis-protocol   (git submodule)
  → CRUMBS            (sibling repo, source-ingested; Linux HAL gated by ENABLE_HARDWARE)
  → bread-crumbs-contracts  (sibling repo, headers only)
  → linux-wire        (sibling repo, required by CRUMBS Linux HAL, hardware builds only)
  → vcpkg: protobuf, yaml-cpp, gtest

CRUMBS
  → linux-wire        (sibling repo, Linux HAL only; no Linux dep in core C library)

bread-crumbs-contracts
  → CRUMBS headers    (included for inline helpers; not a full build dependency)
```

Each layer depends only on layers below it. No circular dependencies. No layer reaches past its
immediate lower neighbor.

---

## Design Principles

| Principle                           | Rule                                                                                                           |
| ----------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Transport / protocol separation     | CRUMBS = pure transport. BREAD semantics begin at bread-crumbs-contracts.                                      |
| Additive capabilities               | Cap flags only accumulate. Existing bit semantics never change. No type-ID splits for variations.              |
| Capability gating at inventory time | `DescribeDevice` is the gate. Adapters do not re-check caps per call.                                          |
| Single source of truth              | `StateCache` is the only source of signal state. Nothing reads from providers directly after discovery.        |
| Single control path                 | All calls go through `CallRouter`. BT automation, HTTP, and tests use the same path. No bypasses.              |
| Provider isolation                  | A crashing provider does not crash anolis. `ProviderSupervisor` restarts with exponential backoff.             |
| Safe hardware initialization        | Providers MUST initialize all hardware to a safe, inactive state before completing `WaitReady`.                |
| Safe mode defaults                  | anolis starts in IDLE. FAULT recovery requires an explicit transition through MANUAL — no FAULT→AUTO shortcut. |
| C++ providers use `crumbs::Session` | Never call CRUMBS or contracts C helpers directly from C++ adapter code.                                       |
| No early generalization             | Generic multi-family CRUMBS providers are speculative. Extract only when a second real consumer exists.        |
