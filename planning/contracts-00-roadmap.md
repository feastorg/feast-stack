# UI, Contracts, and Packaging Roadmap (Index)

Status: Active roadmap index (`contracts-01`, `contracts-02`, and `contracts-03` slices A-D complete; `03E` deferred).
Purpose: Point to the focused planning documents and lock execution order.

## Focused Plans

1. Runtime config contract:
   - `contracts-01-runtime-config.md`
2. Runtime HTTP contract:
   - `contracts-02-runtime-http.md`
3. Machine profile contract and packaging (+ cross-contract parity follow-up):
   - `contracts-03-machine-profile-packaging.md`
4. Operator UI and composer contract alignment:
   - `contracts-04-ui-composer-alignment.md`
5. Packaging/repo-boundary decision record:
   - `contracts-05-packaging-boundary-decision.md`

## Execution Order

1. Runtime config contract (normative schema first, including parser-alignment and duplicate-key close-out).
2. Runtime HTTP contract (OpenAPI baseline + conformance checks).
3. Machine profile packaging contract (profile manifest + validation built on the config contract).
4. UI/composer alignment against the finalized config + HTTP contracts and machine profile structure.
5. Revisit repo/tool split only after contract gates are stable.

Order rationale:

1. We need stable machine-readable contracts first (`01` + `02`) before downstream tool alignment.
2. We package machine assets next (`03`) so UI/composer target a stable package layout.
3. We then harden UI/composer behavior against those locked contracts (`04`).
4. Only after measurable stability do we decide repo boundaries (`05`).

## Decision Gates

Do not split tools out of `anolis` until all pass:

1. Runtime config schema is validated in CI.
2. Runtime config parser-alignment close-out is complete (duplicate-key rejection + parser edge fixtures).
3. HTTP OpenAPI spec is linted and conformance checked.
4. Machine profile package can be validated and executed from a single entrypoint.
5. UI and composer pass fixture-based contract checks against runtime config + HTTP contracts.

## Out of Scope (for this wave)

1. Immediate multi-repo split of `operator-ui` and `system-composer`.
2. Auth redesign or API governance beyond current `v0`.
3. Provider-specific logic inside core runtime automation code.

## Future Progression Note

After `contracts-03` slices A-D are complete and green in CI, execute the deferred
follow-up item `03E` for parity hardening and potential shared schema fragments.

1. Goal: define overlapping data structures once (where technically feasible).
2. Constraint: only proceed if schema dialect/tooling choices are explicit and do not
   destabilize existing gates.
3. Priority: lower than completing current contract gates and machine packaging work.
