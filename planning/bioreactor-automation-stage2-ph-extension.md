# Bioreactor Automation Plan - Stage 2 (pH Acid/Base Extension)

Status: Planned extension
Scope: Add guarded pH automation on `dcmt1` only after Stage 1 stir/feed automation is stable.

## 1) Preconditions

Do not enable Stage 2 until all are true:

1. Stage 1 has passed repeatable hardware validation.
2. Foundation Sprint primitives from Stage 1 are complete and in production use:
   - DCMT command composer helper
   - periodic pulse scheduler primitive
   - bool-capable parameter access in BT path
   - open-loop mode guard helper
   - edge-triggered command emission policy
   - mode-exit handoff behavior via runtime mode-change callback
3. `ph0` readings are commissioned, calibrated, and stable.
4. Manual acid/base pump direction and flow behavior are verified in `MANUAL` mode.
5. Operators are comfortable with `AUTO -> MANUAL` recovery flow.

## 2) Control Goal

Maintain pH within a target band using simple pulse dosing:

- If pH is above upper band -> acid pulse.
- If pH is below lower band -> base pulse.
- If pH is inside band -> no pH dosing.

This remains intentionally simple pulse-based feedback control (not PID).

## 3) Hardware Mapping For pH Control

- `dcmt1/ch1` = acid pump
- `dcmt1/ch2` = base pump
- `ezo0/ph0` = control signal (`ph.value`)

Rules:

1. Never command acid and base simultaneously.
2. `dcmt1` actuation must use composed atomic writes.
3. `dcmt1` mode must be verified as `open_loop` before dosing.

## 4) Stage 2 BT Behavior

### A) Signal and mode gates

Before any dose decision:

1. Check `ph0/ph.value` quality is `OK`.
2. Verify `dcmt1` mode is `open_loop`.
3. If either check fails, skip dosing and hold both pH pumps off.

### B) Band decision

Given:

- `target`
- `deadband`

Compute:

- `upper = target + deadband`
- `lower = target - deadband`

Decision:

1. `pH > upper` -> acid dose branch.
2. `pH < lower` -> base dose branch.
3. otherwise -> no-dose branch.

### C) Dose pulse execution

For selected branch:

1. Apply pump PWM for `dose_pulse_s`.
2. End pulse (both channels off).
3. Enter lockout/mix wait.

### D) Lockout and anti-chatter

After any pulse:

1. Enforce `min_interdose_s` before another dose.
2. Enforce `mix_wait_s` before evaluating next correction pulse.
3. Enforce per-hour cap (`max_pulses_per_hour`) to limit runaway dosing.

### E) Edge-triggered emission

- Emit `dcmt1` write only on state transitions (dose start/dose stop or keepalive due).
- Avoid per-tick repeated writes when desired command is unchanged.

### F) Mode-exit handoff

- On `AUTO -> MANUAL/FAULT/IDLE`, send deterministic handoff command for `dcmt1` with both channels off.
- Implement this in runtime mode-change callback path, not BT tick flow.

## 5) Runtime Parameters (Stage 2 Additions)

Recommended additions:

1. `ph_auto_enable` (bool), default `false`.
2. `ph_target` (double), default `7.00`, range `[4.00,10.00]`.
3. `ph_deadband` (double), default `0.10`, range `[0.02,0.50]`.
4. `acid_pwm` (int64), default `40`, range `[-100,100]`.
5. `base_pwm` (int64), default `40`, range `[-100,100]`.
6. `dose_pulse_s` (int64), default `2`, range `[1,60]`.
7. `mix_wait_s` (int64), default `60`, range `[5,1800]`.
8. `min_interdose_s` (int64), default `120`, range `[5,3600]`.
9. `max_pulses_per_hour` (int64), default `12`, range `[1,120]`.
10. `dcmt1_enforce_open_loop` (bool), default `true`.
11. `dcmt1_command_keepalive_s` (int64), default `30`, range `[5,3600]`.

Operational rules:

1. If `ph_auto_enable=false`, acid/base channels remain off.
2. If parameter values are invalid/missing, fail safe to no-dose.

## 6) Safety and Operator Guardrails

1. Keep `manual_gating_policy: BLOCK` during initial Stage 2 rollout.
2. Any repeated call/write failures should trigger automatic fallback to `MANUAL`.
3. Maintain clear operator controls for:
   - mode changes (`IDLE/MANUAL/AUTO/FAULT`)
   - parameter adjustments via `/v0/parameters`
4. Keep dosing conservative initially (small pulses, longer lockout).

## 7) Validation Sequence (Stage 2)

1. Dry-run in `AUTO` with `ph_auto_enable=false`:
   - verify no acid/base actuation.
2. Controlled low-risk test with conservative params:
   - enable pH automation,
   - verify only one channel activates per decision,
   - verify lockout windows are honored.
3. Disturbance-response check:
   - confirm dosing stops once value re-enters deadband.
4. Fault-path check:
   - force signal quality failure,
   - verify pH dosing halts and remains halted until quality recovers.
5. Mode-exit check:
   - switch `AUTO -> MANUAL` and `AUTO -> FAULT`, confirm runtime callback handoff drives `dcmt1` channels off.

Acceptance:

1. No simultaneous acid/base drive.
2. No chatter around setpoint (deadband + lockout effective).
3. Operator can always disable pH automation and recover safely.
4. Dose history and state transitions are observable in logs/telemetry.

## 8) Out of Scope for Stage 2

1. Full PID/MPC pH control.
2. DO-driven control logic.
3. Coupled multi-variable bioprocess optimization.
