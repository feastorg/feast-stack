# ADPP — Anolis Device Provider Protocol

ADPP is the standard interface between the anolis runtime and device providers. Any process
that speaks ADPP over stdio can be managed by anolis without changes to anolis. Providers are
isolated subprocesses — a crash in a provider does not crash the runtime.

The schema is defined in `anolis-protocol` and vendored as a git submodule into each consuming
repo (`external/anolis-protocol/`). Both sides must reference the same submodule commit.

---

## Transport

Providers communicate with anolis over **stdio** (stdin for requests, stdout for replies).
Each message uses a 4-byte little-endian length prefix:

```
[message_length : uint32_le (4 bytes)]  [protobuf_bytes : variable]
```

- `message_length` is the byte count of the following protobuf payload.
- No session state is carried in the framing layer — all state is in the protobuf messages.
- There is no multiplexing — messages are processed one at a time per provider process.

---

## RPCs

| RPC              | Required | Direction                  | Purpose                                                    |
| ---------------- | -------- | -------------------------- | ---------------------------------------------------------- |
| `Hello`          | Yes      | anolis → provider → anolis | Protocol handshake, version negotiation, provider metadata |
| `ListDevices`    | Yes      | anolis → provider → anolis | Enumerate all managed device IDs                           |
| `DescribeDevice` | Yes      | anolis → provider → anolis | Return device descriptor + capability set                  |
| `ReadSignals`    | Yes      | anolis → provider → anolis | Read current values for a set of signals                   |
| `Call`           | Yes      | anolis → provider → anolis | Dispatch a function call to a device                       |
| `GetHealth`      | Optional | anolis → provider → anolis | Provider and per-device health diagnostics                 |
| `WaitReady`      | Optional | anolis → provider → anolis | Block until provider has completed initialization          |

`HelloResponse.metadata.supports_wait_ready` declares whether `WaitReady` is implemented.
anolis calls `WaitReady` before `ListDevices` when the provider declares support.

---

## Key Schema Types

### Device

```
device_id    : string   — stable identifier used for all routing (e.g. "rlht0", "tempctl0")
type_id      : string   — namespaced device type (e.g. "bread.rlht", "sim.tempctl")
type_version : string   — semantic version string
label        : string   — human-readable name
address      : string   — physical address (e.g. I2C hex "0x08")
tags         : map<string, string>
```

### CapabilitySet

```
functions : FunctionSpec[]
signals   : SignalSpec[]
```

### FunctionSpec

```
function_id   : uint32
function_name : string
description   : string
args          : ArgSpec[]
policy        : FunctionPolicy
```

**`FunctionPolicy.category`** classifies intent for the CallRouter:

| Category  | Meaning                                                  |
| --------- | -------------------------------------------------------- |
| `READ`    | Non-mutating — reads device state without side effects   |
| `CONFIG`  | Mutating — changes persistent device settings            |
| `ACTUATE` | Mutating — drives an actuator or changes physical output |

Additional policy fields: `allowed_modes` (set of mode strings), `requires_lease` (bool),
`min_interval_ms` (uint32), `is_idempotent` (bool).

### SignalSpec

```
signal_id     : string
description   : string
value_type    : ValueType
poll_hint_hz  : float    — if > 0, StateCache auto-polls at this rate
stale_after_ms: uint32   — staleness threshold for quality tracking
```

`poll_hint_hz > 0` is how a provider tells anolis to include a signal in background polling.
Provider-bread sets `poll_hint_hz = 1.0` on all signals.

### SignalValue

```
signal_id : string
value     : Value
quality   : Quality   — OK | STALE | FAULT | UNKNOWN
timestamp : uint64    — milliseconds since epoch
```

### ArgSpec

```
arg_name    : string
value_type  : ValueType
required    : bool
description : string
min_value   : Value    (optional — numeric range enforcement)
max_value   : Value    (optional)
allowed_values : string[]  (optional — STRING enum gating)
```

### ValueType

`BOOL`, `INT64`, `UINT64`, `DOUBLE`, `STRING`

---

## Status Codes

All RPC responses include a `Status` message. Non-`CODE_OK` responses carry a human-readable
`message` string.

| Code                       | Typical meaning                                                      |
| -------------------------- | -------------------------------------------------------------------- |
| `CODE_OK`                  | Success                                                              |
| `CODE_INVALID_ARGUMENT`    | Type mismatch, bad arg name, value violates range/allowed constraint |
| `CODE_NOT_FOUND`           | Requested device ID or function ID/name does not exist               |
| `CODE_FAILED_PRECONDITION` | Capability flag not present; device in wrong state for this call     |
| `CODE_OUT_OF_RANGE`        | Numeric value outside hardware-enforced range                        |
| `CODE_UNIMPLEMENTED`       | RPC not supported by this provider                                   |
| `CODE_DEADLINE_EXCEEDED`   | Hardware read/write timed out                                        |
| `CODE_UNAVAILABLE`         | Hardware not reachable; session not open; device powered off         |
| `CODE_RESOURCE_EXHAUSTED`  | Rate limit hit; buffer full                                          |
| `CODE_INTERNAL`            | Frame decode error; unexpected response content                      |
| `CODE_DATA_LOSS`           | Response data corrupt or truncated                                   |

---

## Provider Contract

Providers must satisfy these invariants to work correctly with anolis:

1. **Safe initialization.** Before completing `Hello` (or `WaitReady` if supported), all managed
   hardware must be initialized to a safe, inactive state. No actuator should be energised at
   startup until an explicit `Call` requests it.

2. **Stable device IDs.** `device_id` values returned by `ListDevices` must be stable across
   restarts. anolis uses them as routing keys in `DeviceRegistry`.

3. **Stateless RPCs.** Each RPC is independent. Providers must handle any legal RPC sequence
   including `DescribeDevice` before `ListDevices` or repeated calls to the same RPC.

4. **Crash safety.** anolis will restart the provider subprocess after a crash with exponential
   backoff. The provider must reach a consistent state on each fresh invocation.

5. **Stdio discipline.** Providers must not write anything to stdout other than ADPP framed
   messages. Diagnostic output goes to stderr.

6. **`CallRequest` resolution.** `CallRequest` carries both `function_id` (u32) and
   `function_name` (string). anolis resolves by name from `DeviceRegistry` and sends both.
   Providers should honor either field.
