# Grounded Data Pipeline — Build Roadmap

Status: plan agreed, no implementation code yet. This document is the contract
for the build; each slice in §5 lands as its own PR.

## 1. What this is

A minimal but complete instance of the **correctness axis** of a scoped,
verifiable data pipeline, in Go, over a fixed finance dataset: plan → validate →
execute → narrate → verify, with provenance on every value, then dataset
versioning and staleness propagation on top.

The wider architecture rests on two orthogonal axes that stay separate in code:
**access confinement** (what can this pipeline touch at all?) and **grounding**
(given what it can touch, does every output trace to an exact, verified source,
with numbers produced by deterministic code rather than the model?). The
governing principle: the LLM lives at the boundaries — intent in, narration out
— and is untrusted in the middle. **This build exercises grounding only.**
Confinement (credentials, brokered grants, multi-source adapters) is
deliberately out of scope: a single trusted local dataset suffices.

Two structural invariants the code must preserve:

1. The LLM only ever touches a `Capability` (in) and a `TypedValue` (out).
   There is no type-level path from the model to a data source or raw scope.
2. Every value is a provenance-carrying `TypedValue`. There is no path for a
   bare number to reach narration.

## 2. Decisions

| Decision | Choice |
| --- | --- |
| Location | `experiments/grounded-pipeline/`, own `go.mod`, outside every ownership tree in the root `AGENTS.md`. Not a service; liftable into its own repo. |
| Data source | Embedded JSON fixtures via `go:embed`. Determinism is the point; a live source can slot behind `DataSource` later. |
| Planner order | `StubPlanner` first, `LLMPlanner` second. The pipeline does not block on model integration. |
| Narrator/gate contract | **Template substitution.** The narrator emits placeholder tokens plus a claim list; it never emits a digit. See §3c. |
| Period resolution | The planner picks concrete period keys; the validator enforces them. No free-text parsing in the deterministic core. See §3e. |
| Op surface | `mean`, `delta_pp`, `pct_change` — only ops with a caller. Ops are unit-aware and reject mismatches. See §3d. |
| Dirty granularity | Per-cell. |
| Persistence | Fixtures embedded; ledger in memory with a JSON dump for inspection. |
| Dependencies | Standard library only, except the model client in Slice 4. |
| Review slicing | Six PRs (§5), each ending at a green `go test ./...`. |

Deferred: the entire confinement axis, multi-source adapters, non-finance
instantiations.

## 3. Design notes (deltas from the handoff sketch)

**a. "Undefined" is a return value, not an error.** The sketch's
"error==Undefined is normal" contradicts the design call that not-in-scope must
not surface as an exception (an error leaks existence). Go already has the
idiom:

```go
Invoke(args map[string]any) (v TypedValue, ok bool, err error)
// ok=false => not defined / not in scope. Normal. err => genuine fault only.
```

**b. Plan validation is its own step, not part of the executor.** Before
anything runs: DAG acyclic, every `$ref` resolves, every argument typechecks,
every entity/metric/period in the allowlist, units consistent. This is the
correctness-axis analogue of the mint step — where a malformed or out-of-scope
model plan dies before touching data. A rejected plan produces a structured
reason and **zero** tool invocations. (A distinct step, not a distinct
package — it lives in `engine`.)

**c. The gate is structural, not a scan.** The narrator returns a template with
tokens and a claim list binding each token to a provenance ID:

```go
type Claim struct{ Token, ProvID string }
type Draft struct {
    Template string  // "AAPL's gross margin rose from {{v1}} to {{v3}}."
    Claims   []Claim
}
```

The gate verifies every token has a claim, every claim resolves to an executed
non-stale value in the ledger, and no claim is unused — then substitutes. A
narrator writing a number is not *detected*; it is *unrepresentable*, because
its output type has nowhere to put one.

**d. Ops are unit-aware.** Comparing a ratio to a ratio is not a percent
change: latest gross margin vs. its own three-year average is a difference in
*percentage points*. Running `pct_change` there yields a number that is
arithmetically correct and semantically wrong — exactly the failure class this
pipeline exists to prevent. So `delta_pp` applies to ratio-typed values,
`pct_change` to levels, and ops reject unit mismatches. `Unit` is load-bearing,
which is why it must be right in Slice 1. (`ratio` and `cagr` from the sketch
are cut: no caller. The semantic layer computes derived ratios internally; that
never needed to be a planner-visible op.)

**e. The planner picks periods.** The sketch's `resolve_period(text)` would put
free-text parsing ("last three fiscal years") inside the deterministic core —
either a brittle lookup table or NLP in the wrong layer. Mapping intent to
structure is the planner's job: it sees `list_periods()` (concrete keys,
ordered, latest marked) and emits keys in the plan; the validator enforces they
exist. Period keys are structure, not values, so this stays within the
LLM-at-the-boundary doctrine.

**f. The provenance graph is uniform.** The sketch's `Prov.Inputs` mixed
"upstream Prov IDs and/or cell keys". Instead, every fetched cell gets its own
source node (op `"source"`, carrying cell key and `SourceRev`), so inputs are
*always* Prov IDs. One node kind, one graph walk — which Slice 6 depends on.

**g. Extracted = cleaning only.** The sketch suggested the `extracted` layer
might pre-materialize derived metrics. That would create a second source of
truth for `gross_margin` and muddy provenance. Derived metrics are computed in
exactly one place — the semantic layer — and `extracted` is limited to
deterministic normalization/cleaning of `raw`.

## 4. Shape

```
experiments/grounded-pipeline/
  cmd/pipeline   // wiring + the sample query
  model          // TypedValue, Prov, Cell, Unit, Plan, PlanStep, Draft, Claim
  store          // DataSource, DatasetVersion {raw, extracted}, redirectable resolver, fixtures
  semantic       // metric defs (stored vs derived), periods, allowlists
  tools          // capability surface: list_metrics, list_periods, get_metric, ops
  engine         // deterministic core: validator, executor, ledger, gate
  planner        // Planner iface; StubPlanner, LLMPlanner
  narrator       // Narrator iface; StubNarrator, LLMNarrator
```

The layout mirrors the trust story: `planner` and `narrator` are the untrusted
LLM boundary; `engine` is the deterministic middle; the rest is substrate.

**Dataset.** AAPL, MSFT × FY2023–FY2025. Stored (7): `revenue`,
`cost_of_revenue`, `gross_profit`, `operating_income`, `net_income`,
`rnd_expense`, `diluted_shares`. Derived (4, semantic layer owns the formulas):
`gross_margin`, `operating_margin`, `net_margin`, `eps`.

**Tool surface.** `list_metrics()`, `list_periods()`,
`get_metric(entity, metric, period)`, `mean`, `delta_pp`, `pct_change`.
Entities are constrained to `{AAPL, MSFT}` at mint time; anything else is
not-defined. (`Describe() ToolSchema` joins the `Capability` interface in
Slice 4, with its first caller.)

**Sample query** (the whole build targets this):

> How did gross margin move for AAPL and MSFT over the last three fiscal years,
> and how does each company's latest year compare to its own three-year average?

Exercises: two entities, stored + derived metrics, planner-side period
resolution, math ops, multi-value narration.

## 5. Slices

Each is one PR, independently reviewable, ending green.

### Slice 1 — Foundation: types, fixtures, semantic layer
`model`, `store` (fixture-backed `DataSource`), `semantic`, v1 fixtures.
**Done when:** every stored cell loads; each derived metric resolves to its
input cells with correct units; unknown entity/metric/period is not-defined,
no error.
**Reviewing:** are the core types — especially `Unit` and `Prov` — right?
Cheapest slice to be wrong in, most expensive to fix later.

### Slice 2 — Tools, engine, stub planner
`tools`, `engine` (validator, executor, ledger), `StubPlanner` with the
hardcoded plan for the sample query.
**Done when:** the stub plan executes to asserted numbers; every `TypedValue`
traces to v1 source nodes; each malformed plan — cycle, dangling `$ref`,
disallowed entity, type mismatch, unit mismatch — is rejected with a distinct
reason and zero tool invocations.
**Reviewing:** does the validator make out-of-scope plans unrepresentable
rather than merely caught?

### Slice 3 — Narrator, gate, cmd (first full loop)
`narrator` with `StubNarrator`, gate in `engine`, `cmd/pipeline`.
**Done when:** `go run ./cmd/pipeline` answers the sample query end to end and
prints the provenance tree behind every number; a draft with an unbound token
or a claim to a nonexistent provenance ID is rejected.
**Reviewing:** the first full demo — the loop is real from here on.

### Slice 4 — Real planner and narrator
`LLMPlanner` and `LLMNarrator` behind the existing interfaces; `Describe()`
added to `Capability`; recorded request/response fixtures keep CI
deterministic; live path behind an env var.
**Done when:** the model builds a valid plan for the sample query *and* one
unseen variant; narration introduces no unbacked numbers.
**Reviewing:** does the tool surface survive contact with a real model?
**Blocked on:** model id, endpoint, key mechanism (§6).

### Slice 5 — Versioning
`DatasetVersion = {raw, extracted}` with `extracted` a deterministic cleaning
pass (§3g); redirectable resolver; v2 fixtures with one restated figure.
**Done when:** rebinding v1→v2 and re-running the *identical* plan object
yields updated numbers with updated `SourceRev`.
**Reviewing:** is the raw/extracted seam in the right place?

### Slice 6 — Dirty-bit
Content hashing (`hash(op, sorted(input hashes))` for detection), forward walk
over the uniform provenance graph (for the report), claim-level staleness,
stale-subgraph-only recompute.
**Done when:** changing one cell yields the exact set of stale values *and*
stale narrated sentences, each naming the changed cell; recompute touches only
the stale subgraph while unaffected values keep their identity.
**Reviewing:** precision of invalidation. The payoff slice.

## 6. Open items

- **Slice 4 credentials:** model id, endpoint, key mechanism — needed only when
  that slice starts. Slices 5–6 can proceed against `StubPlanner` if delayed.
- **v2 restatement choice:** which figure changes between v1 and v2 — picked in
  Slice 5 for maximal dirty-bit legibility (one cell, wide blast radius through
  derived metrics and narration).
