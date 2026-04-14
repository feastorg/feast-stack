# Machine Profile Contract and Packaging Plan

Status: Implemented for slices 03A-03D. Slice 03E deferred.
Priority: High (next after `contracts-01` and `contracts-02` closeout).
Intent: Make each machine setup a contract-validated package without putting machine semantics into core runtime code.

## 1) Problem

Machine assets already exist (`config/bioreactor`, `config/mixed-bus-providers`, behavior XMLs, runbooks), but packaging is implicit and not contract-checked as a first-class unit.

Current risk:

1. Package completeness is convention-based, not schema-validated.
2. Tooling can drift on file layout and entrypoint expectations.
3. Future machine reuse has no locked manifest contract.

## 2) Hard-Nosed Audit Findings (Code-Verified)

This plan was reviewed against current code/docs/tests in `anolis`:

1. `config/bioreactor` is already the de-facto machine package and is used by checked-in runbooks.
2. System Composer tests currently assert parity against files in `config/bioreactor`.
3. Proposing an immediate move to `config/machines/<id>/` would create churn and break existing parity tests with no immediate functional gain.
4. `systems/<project>/` is a composer workspace/output area (gitignored), not a source-controlled machine package location.
5. Runtime-config and HTTP contracts had CI gates while machine packaging did not (pre-implementation baseline).
6. No machine manifest schema or validator existed at audit time (pre-implementation baseline).

Conclusion:

1. Keep location stable now.
2. Add manifest + validator first.
3. Defer directory migration and repo-boundary decisions until after packaging gates are green.

## 3) Decision Lock (This Wave)

1. Machine package location (v1): keep under existing `config/<machine_id>/` paths.
2. First machine package baseline: `config/bioreactor/`.
3. Workspace/generated project files remain under `systems/<project>/` (composer concern), not machine package source.
4. Manifest format: YAML (`machine-profile.yaml`) to align with existing operator-facing config workflows.
5. Runtime entrypoint files remain as currently shipped names (for stability), and manifest maps roles (`manual`, `telemetry`, `automation`, `full`) to concrete file paths.

## 4) Contract Rules

1. Core runtime remains provider-agnostic.
2. Provider/device/channel mapping lives only in machine package config.
3. Behavior trees are machine-owned assets and referenced through config/parameters, not core constants.
4. Machine packages must be independently lintable/validatable by contract tooling.
5. Machine package validation is additive to runtime-config and HTTP contracts, not a replacement.

## 5) Manifest Contract (v1)

Each machine package must include `machine-profile.yaml` with required fields:

1. `schema_version` (manifest schema version).
2. `machine_id` (stable package identifier).
3. `display_name` (human-readable name).
4. `runtime_profiles` map:
   - `manual`
   - `telemetry` (optional)
   - `automation` (optional)
   - `full` (optional)
5. `providers` map of required provider IDs to provider config paths.
6. `behaviors` list (optional) for BT assets referenced by profiles.
7. `validation` section for recommended command entrypoints and expected provider IDs.
8. Compatibility metadata:
   - runtime contract baseline reference (required).
   - provider compatibility notes/ranges (required, policy format locked in schema).

## 6) Execution Slices

### Slice 03A — Schema and Baseline Spec (Implemented)

1. Add `anolis/schemas/machine-profile.schema.json`.
2. Add `anolis/docs/contracts/machine-profile-baseline.md`.
3. Document required fields and compatibility policy for v1.
4. Do not move existing machine directories in this slice.

### Slice 03B — Contract Validator and Fixtures (Implemented)

1. Add `anolis/tools/contracts/validate-machine-profiles.py`.
2. Add fixture set under `anolis/tests/contracts/machine-profile/`:
   - `valid/`
   - `invalid/schema/`
   - `invalid/references/`
3. Validator responsibilities:
   - schema validation,
   - required file existence checks,
   - runtime profile file naming/pattern checks,
   - referenced provider/behavior file existence,
   - optional runtime-config schema validation of referenced runtime profiles.

### Slice 03C — Bioreactor Package Adoption (Implemented)

1. Add `anolis/config/bioreactor/machine-profile.yaml`.
2. Update `anolis/config/bioreactor/README.md` to reference manifest as package authority.
3. Update `anolis/config/mixed-bus-providers/README.md` to keep scenario-pack positioning clear vs canonical machine package.

### Slice 03D — CI and Local Gate Wiring (Implemented)

1. Add machine-profile validator to CI strict lane.
2. Add machine-profile validator to `tools/verify-local.sh`.
3. Ensure validator failures are blocking for contract drift.

### Slice 03E — Parity and Shared-Structure Follow-Up (From 02, Deferred)

1. Add explicit parity checks for overlapping structures where practical (without cross-dialect fragility).
2. Evaluate shared schema fragments only after 03A-03D are stable and green.
3. Keep this follow-up scoped; do not block machine package baseline rollout on speculative refactors.

## 7) Non-Goals (This Wave)

1. No immediate move to `config/machines/<id>/`.
2. No immediate repo split for machine packages/UI/composer.
3. No runtime API redesign.
4. No provider protocol/schema redesign.

## 8) Risks and Mitigations

1. Risk: Manifest becomes stale relative to package files.
   Mitigation: CI validator includes file existence and reference integrity checks.
2. Risk: Composer workspace model and machine package model drift.
   Mitigation: add parity assertions in composer tests against canonical machine package where applicable.
3. Risk: Over-scoping into packaging/repo-boundary decisions too early.
   Mitigation: keep boundary decisions in `contracts-05` and gate them on measured stability.

## 9) Exit Criteria

`contracts-03` is complete when:

1. Machine profile schema exists and is documented.
2. Machine profile validator + fixtures are in CI.
3. `config/bioreactor` includes a valid manifest and passes gates.
4. Local verification path includes machine profile validation.
5. Packaging rules are clear enough to onboard a second machine package without ad-hoc conventions.

## 10) Closeout Snapshot

Implemented and now in place:

1. machine-profile schema and baseline documentation.
2. machine-profile validator with valid/invalid fixture sets.
3. canonical `config/bioreactor/machine-profile.yaml`.
4. machine-profile validation in CI strict lane and `tools/verify-local.sh`.

Remaining deferred item:

1. Slice 03E (cross-contract parity/shared-structure follow-up).
