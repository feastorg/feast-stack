# Bioreactor Automation Plan - Stage 1 (Overall + Stir/Feed)

Status: Execution plan
Scope: First automation cut for the current lab machine, focused on stable stirring and scheduled feed dosing only.

## 1) Objective

Implement a low-risk automation layer that:

1. Keeps impeller behavior deterministic.
2. Runs feed dosing on a time schedule.
3. Preserves manual operator authority and safe fallback.

This stage intentionally excludes pH closed-loop dosing logic.

## 2) Machine Mapping (Current Hardware Contract)

- `bread0/rlht0 @ 0x0A`: temperature sensing context only.
- `bread0/dcmt0 @ 0x14`:
  - channel 1 = impeller stirring motor
  - channel 2 = media/feed dosing motor
- `bread0/dcmt1 @ 0x15`:
  - channel 1 = acid pump (reserved for Stage 2)
  - channel 2 = base pump (reserved for Stage 2)
- `ezo0/ph0 @ 0x63`: pH sensing.
- `ezo0/do0 @ 0x61`: DO sensing.

## 3) Control Model

- Runtime startup remains safe-by-default (`IDLE`).
- Automation runs only in `AUTO`.
- Recommended gating policy for this stage: `manual_gating_policy: BLOCK`.
- Operator can always drop to `MANUAL` for recovery.

Important implementation constraints from current stack:

1. `dcmt0 set_open_loop` updates both motor channels in one call.
2. DCMT firmware ignores `set_open_loop` when mode is not `open_loop`.
3. `CallDevice` is synchronous in current Anolis BT nodes, so blocking calls stall the tree tick loop.
4. BT tick loop executes only in `AUTO`, so mode-exit actuator handoff must be implemented in runtime mode-change callback path, not as BT-only logic.

## 4) Foundation Sprint (Required Before Stage 1 BT Logic)

These helper primitives should land first.

### A) DCMT command composer helper

Goal:
- Compose logical intents into one atomic DCMT open-loop command.

Input intents:
- impeller intent -> channel 1 command
- feed intent -> channel 2 command

Output:
- one `set_open_loop(motor1_pwm, motor2_pwm)` payload

Why first:
- Removes tree-level channel-clobbering risk.
- Keeps Stage 1 automation readable and deterministic.

### B) Periodic pulse scheduler primitive

Goal:
- Reusable schedule primitive for startup delay + interval + pulse width + max pulses/hour.

Why first:
- Avoid fragile ad-hoc timing logic in XML scripts.
- Reuse directly in Stage 2 pH pulse dosing.

### C) Bool-capable parameter access in BT

Goal:
- Add bool-friendly parameter access for BT without breaking existing numeric `GetParameter` behavior.

Preferred implementation:
- Keep `GetParameter` numeric for compatibility.
- Add dedicated bool node (e.g., `GetParameterBool`).

### D) Open-loop mode guard helper

Goal:
- Before first actuation, enforce and verify `dcmt0 mode == open_loop`.

Behavior:
1. Send `set_mode(open_loop)` if needed.
2. Verify via `mode` signal readback.
3. If mode cannot be confirmed, fail safe and prevent actuation.

### E) Edge-triggered command emission policy

Goal:
- Emit actuation calls only on command changes (or keepalive interval), not every tick.

Why first:
- Reduces bus traffic and post-call refresh load.
- Lowers chance of intermittent I2C noise amplifying into repeated failures.

### F) Mode-exit handoff behavior (Runtime callback path)

Goal:
- Define deterministic actuator state on `AUTO -> MANUAL/FAULT/IDLE`.

Implementation location:
- Runtime mode-change callback path (not BT tick path), because BT stops ticking outside `AUTO`.

Recommended default:
1. On `AUTO -> MANUAL`: send one handoff command with feed off, impeller preserved.
2. On `AUTO -> FAULT` or `AUTO -> IDLE`: force both channels off.

### G) Call timeout and retry pattern for BT actions

Goal:
- Wrap action calls with timeout/retry policy in tree design.

Why first:
- Current `CallDevice` node is synchronous; bounded retries/timeouts prevent long stalls from turning into unstable behavior.

### Foundation acceptance

1. Composer helper unit-tested.
2. Scheduler primitive unit-tested with deterministic time fixtures.
3. Bool BT parameter access available and tested.
4. Open-loop guard verified on real hardware.
5. Edge-triggered emission policy implemented and tested.
6. Mode-exit handoff behavior validated through runtime mode-change callback transitions.
7. Timeout/retry pattern validated in BT execution flow.

## 5) Stage 1 BT Behavior (Built On Foundation Primitives)

### A) Preconditions per cycle

1. Verify `bread0/dcmt0` quality is usable.
2. Verify `dcmt0` mode is `open_loop` (guarded path).
3. If either check fails, do not actuate.

### B) Impeller enforcement

- Maintain impeller command from parameters:
  - enabled -> configured PWM
  - disabled -> 0 PWM

### C) Feed pulse scheduling

- Feed is open-loop time dosing.
- Scheduler computes pulse windows from interval + pulse + limits.
- Active window -> feed channel uses configured feed PWM.
- Outside window -> feed channel is 0 PWM.

### D) Compose and emit command

1. Compose desired pair:
   - `motor1_pwm = impeller_command`
   - `motor2_pwm = feed_command`
2. Emit only if changed from last command (or keepalive due).

## 6) Runtime Parameters (Stage 1)

Recommended parameter set:

1. `impeller_enable` (bool), default `true`.
2. `impeller_pwm` (int64), default `35`, range `[-100,100]`.
3. `feed_enable` (bool), default `true`.
4. `feed_pwm` (int64), default `45`, range `[-100,100]`.
5. `feed_interval_s` (int64), default `900`, range `[30,86400]`.
6. `feed_pulse_s` (int64), default `3`, range `[1,600]`.
7. `feed_startup_delay_s` (int64), default `120`, range `[0,3600]`.
8. `feed_max_pulses_per_hour` (int64), default `8`, range `[1,120]`.
9. `dcmt0_enforce_open_loop` (bool), default `true`.
10. `dcmt0_command_keepalive_s` (int64), default `30`, range `[5,3600]`.
11. `dcmt_command_min_spacing_ms` (int64), default `500`, range `[50,10000]`.
12. `dcmt_write_failure_limit` (int64), default `5`, range `[1,100]`.

Operational rules:

1. If `feed_enable=false`, feed command stays 0.
2. If mode guard fails, actuation is blocked.

## 7) Failure and Recovery Behavior

1. On call failure for `dcmt0`, log and retry under configured policy.
2. If failures exceed configured limit in a rolling window, switch mode to `MANUAL` and alert operator.
3. Recovery path stays explicit: inspect in `MANUAL`, then transition to `AUTO`.

## 8) Validation Sequence (Stage 1)

1. Complete Foundation Sprint acceptance first.
2. Start in `IDLE`, verify runtime + providers healthy.
3. Transition to `MANUAL`, confirm manual motor control works.
4. Transition to `AUTO` with:
   - `impeller_enable=true`
   - `feed_enable=false`
   - verify impeller hold only.
5. Enable feed and verify pulse schedule accuracy.
6. Change feed params live via `/v0/parameters`; verify no restart required.
7. Verify edge-triggered emission (no unnecessary repeated writes when command unchanged).
8. Verify mode-exit handoff behavior via runtime transitions (`AUTO -> MANUAL`, `AUTO -> FAULT`).
9. Inject write-failure scenario and verify configured escalation.

Acceptance:

1. No channel clobbering between stir/feed.
2. Impeller remains stable between feed pulses.
3. Feed schedule is deterministic and tunable.
4. Operator can regain manual control safely at any time.

## 9) Out of Scope for Stage 1

1. pH closed-loop control.
2. Acid/base pump automation on `dcmt1`.
3. Any DO-driven control loop.
