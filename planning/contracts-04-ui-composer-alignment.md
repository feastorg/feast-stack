# UI and Composer Contract Alignment Plan

Status: Implemented and closed (2026-04-14).
Priority: High (next after contracts 01-03 foundations).
Intent: Eliminate drift between System Composer, Operator UI, and runtime contracts.

## Execution Closeout (2026-04-14)

Completed:

1. 04A Composer save validation hardening (schema + semantic server-side, structured 400 errors).
2. 04B Validation de-drift (frontend semantic save validation removed; backend canonical).
3. 04C Custom provider lock (v1 unsupported in Composer UI/catalog, server-side rejection for `kind: custom`).
4. 04D Operator UI fixture alignment (`js/contracts.js` normalizers + fixture tests against canonical runtime-http examples).
5. 04E CI/local gate completion (new CI lane for UI+Composer contracts, `verify-local.sh` updated).
6. 04F Docs cleanup (`tools/system-composer/README.md`, `tools/operator-ui/README.md` updated with current contract behavior and dependencies).

## 1) Why This Phase Exists

Current state proves we have the contract foundations, but still have tool-layer drift risk.

Critical findings from audit:

1. Composer frontend contains duplicate function blocks in `frontend/js/app.js`.
2. Composer save path does not enforce server-side schema + semantic validation before persistence.
3. Frontend and backend validation logic are duplicated and can drift.
4. Custom provider UX allows args editing, but runtime render path does not honor that model consistently.
5. Operator UI request/response assumptions are not fixture-tested against OpenAPI.
6. CI does not currently enforce the full UI/Composer contract gate set.
7. Operator UI docs are partially stale relative to current modular implementation.

## 2) Goals

1. Make backend validation authoritative for Composer persistence.
2. Make Composer generation deterministic and contract-verified.
3. Make Operator UI parsing/rendering contract-tested against OpenAPI fixtures.
4. Wire all alignment checks into CI so drift is blocked pre-merge.

## 3) Non-Goals

1. No repo split in this phase.
2. No framework rewrite of Operator UI or Composer frontend.
3. No machine-specific business logic in generic UI core.

## 4) Locked Architecture Decisions

1. **Composer backend is the source of truth for validation.**
   Frontend validation remains for fast UX feedback only.
2. **Save-time validation is mandatory server-side.**
   A project cannot be persisted if schema or semantic validation fails.
3. **Contract enforcement sequence for Composer output is fixed:**
   generate -> runtime-config schema validation -> runtime `--check-config` smoke (where supported).
4. **Operator UI API behavior is validated via fixture tests, not just manual checks.**
5. **Custom provider handling must be made explicit and coherent before closure.**

## 4.1) Decision Locks (Approved Defaults)

These are now locked for Contracts-04 execution unless explicitly reopened.

1. **Custom provider strategy:** disable custom provider editing/support in Composer v1.
   - Rationale: current UX and renderer behavior are mismatched; disabling removes an unsafe half-state.
2. **Save-time enforcement strictness:** hard fail save on schema or semantic validation errors.
3. **Backend validation stack:** enforce both `system.schema.json` structural checks and backend semantic validation.
4. **Frontend validation role:** keep lightweight UX validation only; backend remains authoritative.
5. **Composer API validation error shape:** structured error payloads (machine-readable fields), not single-string-only errors.
6. **Legacy field policy (`system.json`):** normalize legacy aliases during load/save and surface deprecation messaging; do not hard-fail legacy in this phase.
7. **Operator UI field compatibility policy:** canonical contract fields first, with temporary compatibility fallback where required to avoid breakage.
8. **UI contract testing approach:** add fixture-based JS tests for parsing/normalization plus minimal smoke coverage.
9. **CI gate level:** Contracts-04 fast checks are required in primary CI; heavier runtime smoke can remain targeted/advisory where needed.
10. **Composer CI scope:** require render + schema + semantic checks broadly; keep runtime `--check-config` in targeted lanes.
11. **Fixture source of truth:** reuse `tests/contracts/runtime-http/examples` as the canonical HTTP fixture set.
12. **Composer determinism standard:** semantic stability (parsed YAML equality), not strict byte-for-byte formatting lock.
13. **System schema version for this phase:** keep `schema_version: 1` (no format version bump).

## 5) Scope and Slices

### Slice 04A: Composer Integrity Hardening

1. Remove duplicate function blocks and dead churn in Composer frontend code.
2. Enforce server-side validation on project save:
   - `system.schema.json` structural validation.
   - backend semantic validation (`validator.py`).
3. Return actionable 4xx error payloads for validation failures.
4. Ensure validation failure blocks both persistence and renderer output emission.

Acceptance:

1. Invalid `system.json` cannot be saved through API.
2. Save errors are deterministic and test-covered.
3. Composer frontend remains usable and receives clear validation messages.

### Slice 04B: Validation Surface De-Drift

1. Minimize duplicate rules between frontend and backend validation.
2. Frontend checks reduced to UX-level checks where possible.
3. Backend validator holds canonical semantic constraints.
4. Add regression tests for previously divergent cases (e.g., I2C address normalization differences).

Acceptance:

1. Same payload yields same pass/fail outcome regardless of UI path.
2. Semantic rule changes are made only once (backend) and tested.

### Slice 04C: Custom Provider Contract Reconciliation

We must close the current mismatch between UI expectations and renderer behavior.

Locked execution for this phase:

1. Disable custom provider editing/support in Composer v1.
2. Show explicit “not supported in contract v1” messaging in UI and docs.
3. Reject unsupported custom save shapes server-side.
4. Keep full custom pass-through support as deferred follow-up work, not in Contracts-04.

Acceptance:

1. No path where UI advertises unsupported custom behavior.
2. Tests cover the disabled-v1 behavior and rejection paths.

### Slice 04D: Operator UI HTTP Contract Alignment

1. Add fixture-based tests for representative `/v0` responses used by Operator UI modules.
2. Validate parsing assumptions for:
   - device/state payload fields,
   - capabilities/functions payloads,
   - mode/parameters payloads,
   - SSE event payload handling.
3. Fix known field-shape mismatches (for example, timestamp key assumptions) to match OpenAPI/runtime baseline.

Acceptance:

1. Operator UI response parsing is fixture-tested.
2. Contract shape changes break tests before merge.

### Slice 04E: CI Gate Completion

1. Add CI jobs/steps for:
   - System Composer test suite,
   - Operator UI contract fixture suite,
   - existing contract scripts as dependencies.
2. Keep local `tools/verify-local.sh` aligned with CI gate set.

Acceptance:

1. Contracts-04 checks run in CI on PRs.
2. Drift in Composer/UI contract behavior fails CI.

### Slice 04F: Docs and Baseline Cleanup

1. Update Operator UI README to reflect current module structure and configuration points.
2. Update Composer README with authoritative validation and save semantics.
3. Add concise “contract dependency” section linking runtime config schema + HTTP OpenAPI.

Acceptance:

1. Docs match current behavior and gate model.
2. No stale instructions for old file layout/config paths.

## 6) Test Strategy

### Unit/Fixture Coverage

1. Composer backend validation tests:
   - schema failures,
   - semantic failures,
   - save-path rejection behavior.
2. Composer renderer tests:
   - deterministic output and baseline parity.
3. Operator UI fixture tests:
   - parser/render behavior for current OpenAPI sample payloads.

### Integration Coverage

1. Composer flow: template -> save -> render -> runtime check-config.
2. UI flow smoke against fixture responses for core dashboard/device/automation paths.

## 7) CI and Local Gates

Required CI after this phase:

1. Composer tests (including save validation behavior).
2. Operator UI fixture contract tests.
3. Existing runtime config + machine profile + HTTP contract checks remain green.

Required local workflow:

1. `tools/verify-local.sh` runs all contract and tool alignment checks.
2. New UI fixture test command documented and included.

## 8) Exit Criteria

This phase is closed only when all are true:

1. Composer backend validation is authoritative and enforced on save.
2. Composer/frontend duplicate-churn and rule drift paths are removed.
3. Custom provider behavior is coherent and contract-covered.
4. Operator UI parsing/rendering assumptions are fixture-tested against contract payloads.
5. CI blocks UI/Composer drift before merge.
6. Tool READMEs reflect current architecture and usage.

## 9) Risks and Mitigations

1. Risk: frontend/backend validation divergence reappears.
   Mitigation: backend as authority + regression tests for previously divergent cases.
2. Risk: custom provider ambiguity creates hidden failures.
   Mitigation: explicit lock (disable or fully support), no middle state.
3. Risk: UI evolves faster than OpenAPI fixtures.
   Mitigation: fixture updates required in same PR for contract-surface changes.
4. Risk: new CI checks slow iteration.
   Mitigation: separate fast fixture unit checks from heavier runtime conformance checks.

## 10) Recommended Execution Order

1. 04A Composer integrity hardening.
2. 04B validation de-drift.
3. 04C custom provider contract lock and implementation.
4. 04D Operator UI fixture alignment.
5. 04E CI gate completion.
6. 04F docs cleanup and phase close-out.
