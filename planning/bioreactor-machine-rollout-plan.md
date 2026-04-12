# Bioreactor Machine Rollout Plan (Transient)

Status: Working draft for execution and coordination.
Scope: Align the current lab-validated mixed-bus stack into a stable machine profile, then stage automation safely.

## 1) Current Baseline

What is already working:

- Anolis runtime with `bread0` + `ezo0` on one shared I2C bus.
- 5-device topology visible and controllable:
  - `rlht0 @ 0x0A`
  - `dcmt0 @ 0x14`
  - `dcmt1 @ 0x15`
  - `ph0 @ 0x63`
  - `do0 @ 0x61`
- Operator UI manual operation verified (device state + function calls).
- DO `% saturation` output commissioned and now available.

Known operational caveat:

- Intermittent BREAD read/write errors still occur at low rate (mostly DCMT addresses).
- Mitigation is currently applied via BREAD config timing margins.
- Investigation is deferred and tracked separately.

## 2) Phase A: Align Config To Real Machine

Goal: lock the runtime/provider config to the actual lab wiring and intended use.

### A1. Runtime/profile cleanup

- Keep one canonical manual profile as source of truth.
- Keep one telemetry-enabled profile derived from that manual profile.
- Ensure both reference the same provider config files and device IDs.

### A2. Device intent alignment

- Preserve current addresses and IDs as machine contract.
- Confirm labels reflect process purpose:
  - `dcmt0`: stir + feed
  - `dcmt1`: acid/base pumps
  - `rlht0`: temperature sensing context (no active temp control yet)
  - `ph0`, `do0`: sensing only

### A3. Operational defaults

- Automation disabled by default.
- Telemetry disabled in manual profile, enabled in telemetry profile.
- Retain current BREAD timing margins until reliability work is revisited.

### A4. Acceptance criteria

- Runtime starts cleanly with no discovery mismatch.
- `/v0/devices` reports 5 devices exactly.
- `/v0/providers/health` shows both providers AVAILABLE in steady state.
- Operator UI manual controls work end-to-end.

## 3) Phase B: BT Automation Bring-Up (Guarded, Staged)

Goal: introduce automation without jumping directly into closed-loop bioprocess control.

### B0. Foundation primitives (required)

- Add DCMT command composer helper (logical intents -> one atomic two-channel command).
- Add periodic pulse scheduler primitive (startup delay + interval + pulse + max/hour).
- Add bool-capable BT parameter access path.
- Add open-loop mode guard helper (enforce + verify).
- Add edge-triggered command emission policy (emit-on-change + keepalive).
- Add deterministic mode-exit handoff behavior in runtime mode-change callback path.
- Add timeout/retry pattern for synchronous BT action calls.

### B1. Read-only BT validation

- Build BT tree that reads signals and evaluates health/quality gates only.
- No function calls to pumps/heaters.
- Validate mode transitions and runtime behavior under MANUAL/AUTO boundaries.

### B2. Stage 1 automation (stir + feed)

- Add impeller enforcement and feed pulse scheduling on `dcmt0`.
- Ensure composed atomic write each cycle to prevent channel clobbering.
- Ensure open-loop mode guard is active.
- Emit calls on edges, not every tick.

### B3. Stage 2 extension (pH acid/base)

- Add deadband pulse dosing on `dcmt1` using `ph0` quality and value gates.
- Enforce lockout windows and maximum pulse caps.
- Ensure open-loop mode guard and deterministic mode-exit handoff for dosing channels.

### B4. Deferred (future)

- Closed-loop policies beyond deadband pulse control.
- DO-driven control policies.

### B-stage acceptance criteria

- Each stage has repeatable pass/fail runbook steps.
- No hidden side paths: all actuation still through CallRouter/function API.
- Mode-exit handoff paths are verified independently of BT ticking.
- Operator can force MANUAL and recover cleanly at all times.

## 4) Phase C: Package As A Machine Profile

Goal: make this setup portable and reproducible across labs/operators.

Proposed location:

- `feast-stack/machines/bioreactor/`

Proposed contents:

- `README.md` (machine runbook: prerequisites, launch, validation, shutdown)
- `config/` (runtime + provider YAMLs)
- `scripts/` (validation capture, quick health checks)
- `artifacts/` (ignored runtime captures)
- `versions.md` (pinned repo versions/commits used for validated runs)

Packaging rules:

- Machine profile should reference canonical configs in source repos, or include exact copies with clear ownership.
- No hidden mutable dependencies.
- Commands should use standard presets and existing validation scripts where possible.

## 5) Execution Order

1. Finalize Phase A config alignment and verify with hardware.
2. Complete B0 foundation primitives.
3. Bring up B1 read-only BT and validate mode handling.
4. Execute B2 stir/feed automation validation.
5. Execute B3 pH extension validation.
6. Build machine package structure and runbook.
7. Re-run full validation from machine package entrypoint.

## 6) Out Of Scope (For This Rollout)

- Full advanced closed-loop bioreactor control strategy.
- Advanced fault-tolerant bus remediation for intermittent CRUMBS transport errors.
- Long-term archival documentation format decisions.
