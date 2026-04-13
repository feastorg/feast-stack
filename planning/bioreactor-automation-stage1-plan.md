# Bioreactor Automation Plan - Stage 1 (Stir + Feed)

Status: Locked execution plan (final)
Scope: First automation cut for the current lab machine, focused on stable stirring and scheduled feed dosing only.

## 1) Objective

Implement a low-risk automation layer that:

1. Keeps impeller behavior deterministic.
2. Runs feed dosing on a predictable schedule.
3. Preserves manual operator authority and safe fallback.

Stage 1 excludes pH closed-loop dosing logic.

## 2) Non-Negotiable Architecture Boundary

1. `anolis` core stays provider-agnostic.
2. No provider/device IDs in core C++ (`bread0`, `ezo0`, `dcmt0`, etc.).
3. No provider function names in core C++ (`set_open_loop`, `set_mode`, etc.).
4. No machine/stage naming in core filenames/types.
5. Machine semantics live in machine BT/config assets only.

## 3) External BT Usage Alignment (BehaviorTree.CPP)

1. Use ports/blackboard remapping as the primary dataflow model.
2. Keep per-tick safety checks in a `ReactiveSequence` so conditions are re-evaluated continuously.
3. Keep BT actions short; avoid long blocking work in `tick()`.
4. If long-running work is needed later, use asynchronous/stateful node patterns that return `RUNNING`.

References:

- https://www.behaviortree.dev/docs/tutorial-basics/tutorial_02_basic_ports/
- https://www.behaviortree.dev/docs/tutorial-basics/tutorial_04_sequence
- https://www.behaviortree.dev/docs/guides/ports_vs_blackboard/

## 4) Current Internal Facts (Anolis As-Is)

1. Core BT node set: `ReadSignal`, `CheckQuality`, `CallDevice`, `GetParameter`.
2. `GetParameter` is numeric-only today (`double`/`int64`).
3. `CallDevice` is synchronous and accepts JSON args string.
4. BT ticks only in `AUTO` mode.
5. Valid mode transitions are enforced by `ModeManager`:
   - `MANUAL <-> AUTO`
   - `MANUAL <-> IDLE`
   - `Any -> FAULT`
   - `FAULT -> MANUAL`
   - `AUTO -> IDLE` is invalid.
6. `CallRouter` blocks control operations in `IDLE`.

Consequence:

1. We need a small generic foundation pass before machine actuation logic is maintainable.
2. Handoff writes must execute before entering `IDLE`.

## 5) Stage 1.0 Foundation Pass (Core Generic Only)

### 5.1 Generic BT primitives

1. `GetParameterBool` node.
2. `PeriodicPulseWindow` node:
   - Inputs: enable, startup delay, interval, pulse width, max/hour.
   - Output: active-window bool.
3. `EmitOnChangeOrInterval` node/decorator:
   - Emits only on command change or keepalive timeout.
4. `BuildArgsJson` generic node:
   - Build JSON args from typed input ports.
   - No provider/device semantics in the node.

### 5.2 Generic runtime transition hooks

Add generic transition hooks in runtime config with explicit timing:

1. `before_transition` hooks (for safety writes that must happen before mode switch).
2. `after_transition` hooks (observability/non-actuating side-effects).

This avoids IDLE-gating conflicts for safety actuation.

## 6) Machine Mapping (Machine Assets Only)

Bioreactor-specific mapping (outside core code):

1. `bread0/rlht0 @ 0x0A`: temp sensing context.
2. `bread0/dcmt0 @ 0x14`:
   - channel 1 = impeller
   - channel 2 = feed
3. `bread0/dcmt1 @ 0x15`:
   - channel 1 = acid (Stage 2)
   - channel 2 = base (Stage 2)
4. `ezo0/ph0 @ 0x63`
5. `ezo0/do0 @ 0x61`

## 7) Stage 1 BT Behavior (Machine Layer)

Per tick in `AUTO`:

1. Compute impeller command from parameters.
2. Compute feed active window from scheduler output.
3. Compose one atomic command for `dcmt0` open-loop write:
   - `motor1_pwm` = impeller command
   - `motor2_pwm` = feed command (or 0 when inactive)
4. Emit only on change or keepalive interval.

Stage 1 safety scope note:

1. Signal-quality gating for actuation is deferred to Stage 2.
2. Stage 1 safety relies on deterministic command shaping + transition-hook handoff to safe outputs.

Design rule:

1. Use one write path for both DCMT channels to avoid channel clobbering.

## 8) Mode-Exit Handoff Policy (Valid Transitions Only)

Required safety behavior:

1. `AUTO -> MANUAL`: feed off; impeller policy explicit.
2. `Any -> FAULT`: safe outputs (feed off at minimum; optional full-off policy by config).
3. `MANUAL -> IDLE`: if actuation shutdown required, execute in `before_transition` hook.

Do not plan against `AUTO -> IDLE` directly; that transition is invalid.

## 9) Stage 1 Parameters (Machine Profile)

1. `impeller_enable` (bool)
2. `impeller_pwm` (int64)
3. `feed_enable` (bool)
4. `feed_pwm` (int64)
5. `feed_interval_s` (int64)
6. `feed_pulse_s` (int64)
7. `feed_startup_delay_s` (int64)
8. `feed_max_pulses_per_hour` (int64)
9. `command_keepalive_s` (int64)
10. `command_min_spacing_ms` (int64)

## 10) Validation Sequence

1. Unit-test new generic primitives.
2. Run read-only BT sanity with no actuation.
3. Run Stage 1 with feed disabled (impeller only).
4. Enable feed schedule and verify timing.
5. Verify no excess writes (change/keepalive behavior).
6. Verify transition-hook handoff behavior.
7. Verify write failures surface as BT/runtime error signals and operator-visible health degradation.

Acceptance criteria:

1. Deterministic stir/feed behavior with no channel clobber.
2. Tunable schedule without restart.
3. Safe recovery via `MANUAL` path.
4. Boundary audit passes (no provider semantics in core code).

## 11) Out of Scope

1. pH control logic.
2. Acid/base dosing logic.
3. DO-driven control logic.
