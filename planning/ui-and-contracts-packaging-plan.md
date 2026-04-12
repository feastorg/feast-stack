# UI and Contracts Packaging Plan (Transient)

Status: Working draft for near-term execution.  
Purpose: Decide how to package UI tooling and harden shared contracts to reduce drift across runtime, composer, and clients.

## 1) Problem Statement

The stack is now functionally viable for real hardware validation, but three related issues need a coordinated plan:

1. Tool packaging boundary:
   - `operator-ui` and `system-composer` currently live under `anolis/tools/`.
   - We need a stable long-term repo boundary strategy.

2. Runtime config contract drift risk:
   - Runtime config has a canonical prose reference.
   - Composer already uses a machine-readable schema for `system.json`.
   - Runtime YAML schema is not yet machine-readable as a single validation source of truth.

3. HTTP API contract drift risk:
   - API behavior is documented in prose.
   - There is no canonical OpenAPI contract + lint gate to keep runtime and UI/client assumptions synchronized.

## 2) Current State (Observed)

- `system-composer` has JSON Schema for `system.json`:
  - `anolis/tools/system-composer/schema/system.schema.json`
- Runtime config contract is currently prose-first:
  - `anolis/docs/configuration-schema.md`
- HTTP API is documented prose-first:
  - `anolis/docs/http-api.md`
- Packaging discussion already exists in `anolis/TODO.md`, but needs an execution-ready plan.

## 3) Decision: Near-Term Packaging Model

Recommended immediate decision:

- Keep `operator-ui` and `system-composer` in-repo (inside `anolis`) for now.

Why:

- We are still actively changing runtime behavior, BT integration, and machine profile flows.
- Early repo splits increase coordination burden and increase contract drift risk unless schemas are already strong and enforced.
- In-repo keeps iteration speed high while hardening contracts.

## 4) Target Mid-Term Packaging Model

After contract hardening (Section 5) and stabilization gates are met, move to:

- One dedicated tools/UI repo (single app/repo boundary), not two independent repos initially.

Reasoning:

- `operator-ui` and `system-composer` share domain objects (devices, signals, capabilities, mode/control semantics).
- A single repo allows shared typed client, shared schema package, unified CI, and lower drift probability.
- Splitting into two repos can still be done later if ownership and release cadence diverge significantly.

## 5) Contract Hardening Plan (Before Any Repo Split)

## 5.1 Runtime config schema (machine-readable)

Goal:

- Introduce canonical runtime YAML schema (`JSON Schema 2020-12`) with validation tooling.

Plan:

1. Create `anolis/schemas/runtime-config.schema.json`.
2. Encode all current constraints from docs and loader behavior:
   - required/optional sections
   - numeric ranges
   - enum sets
   - CORS invariants (`*` vs credentials)
   - provider stanza constraints
3. Add schema validation to:
   - CI
   - local verification script
4. Update docs:
   - prose doc becomes explanatory
   - schema file becomes normative

Acceptance:

- Known-good configs pass.
- Known-bad fixtures fail with deterministic messages.
- Composer/runtime generated configs validated by the same schema.

## 5.2 Composer/runtime shared source of truth

Goal:

- Ensure generated runtime configs and runtime loader expectations stay synchronized.

Plan:

1. Define mapping between composer `system.json` and runtime YAML as explicit transform rules.
2. Add test fixtures:
   - `system.json -> rendered YAML -> schema validate`.
3. Add regression tests for critical machine profiles (bioreactor, mixed-bus mock).

Acceptance:

- No generated config can bypass schema checks.
- CI fails on mapping drift.

## 5.3 HTTP API contract (OpenAPI)

Goal:

- Make `/v0` behavior machine-readable and linted.

Plan:

1. Add `anolis/api/openapi-v0.yaml`.
2. Cover active endpoints first:
   - devices, state, call, mode, runtime status, providers health
3. Define status/error envelope consistently with runtime outputs.
4. Add OpenAPI lint in CI.
5. Optionally generate typed client artifacts for UI/composer integration.

Acceptance:

- OpenAPI lint passes in CI.
- Endpoint behavior examples round-trip against real runtime responses.

## 6) Packaging Decision Gates

Do not move UI/tools out of `anolis` until all are true:

1. Runtime config schema exists and is CI-enforced.
2. Composer-to-runtime transform is tested and stable.
3. OpenAPI v0 spec exists and is CI-linted.
4. At least one complete machine profile runbook executes without ad-hoc edits.
5. Two consecutive development cycles complete without contract breakage incidents.

If all gates pass:

- Execute repo split to a single tools/UI repo, preserving shared schema/client package.

## 7) Proposed Execution Sequence

1. Implement runtime config machine schema + CI validation.
2. Wire composer output validation to the same schema.
3. Add OpenAPI v0 + lint gate.
4. Run one full cycle with bioreactor machine profile.
5. Re-evaluate packaging boundary and decide:
   - stay in-repo, or
   - move to single tools/UI repo.

## 8) Out of Scope

- Immediate split into multiple tool repos.
- Public API versioning policy beyond current v0 baseline.
- Full auth model redesign for HTTP (tracked separately).
