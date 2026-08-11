# Grounded Data Pipeline — Build Roadmap

Status: plan agreed, no implementation code yet. This document is the contract
for the build; each slice below lands as its own PR.

## 1. What this is

A minimal but complete instance of the **correctness axis** of a scoped,
verifiable data pipeline, in Go, over a fixed finance dataset: plan → validate →
execute → narrate → verify, with provenance on every value, then dataset
versioning and staleness propagation on top.

Two orthogonal axes underpin the wider architecture. They stay separate in code:

- **Access confinement (security).** What can this pipeline touch at all? The
  pipeline holds narrow capabilities, never ambient authority.
- **Grounding (correctness).** Given what it can touch, does every output trace
  to an exact, verified source, with numbers produced by deterministic code
  rather than the model?

Governing principle: the LLM lives at the boundaries — intent in, narration out
— and is untrusted in the middle. Confinement makes "untrusted" safe; grounding
makes it correct. **This build exercises grounding only.** Confinement (live
credentials, brokered grants, multi-source adapters) is deliberately out of
scope: a single trusted local dataset is sufficient here.

Two structural invariants the code must preserve:

1. The LLM only ever touches a `Capability` (in) and a `TypedValue` (out). There
   is no type-level path from the model to a data source or a raw scope.
2. Every value is a provenance-carrying `TypedValue`. There is no path for a
   bare number to reach narration.

## 2. Decisions

Resolved for this build:

| Decision | Choice |
| --- | --- |
| Location | `experiments/grounded-pipeline/`, own `go.mod`, outside every ownership tree in the root `AGENTS.md`. Not a service; liftable into its own repo. |
| Data source | Embedded JSON fixtures via `go:embed`. Determinism is the point, and it makes the versioning demo trivial. A live source can slot behind `DataSource` later. |
| Planner order | `StubPlanner` first, `LLMPlanner` second. The pipeline does not block on model integration. |
| Narrator/gate contract | **Template substitution.** The narrator emits text with placeholder tokens and a claim list; it never emits a digit. See §3. |
| Dirty granularity | Per-cell. |
| Persistence | Fixtures on disk (embedded); ledger in memory with a JSON dump for inspection. |
| Dependencies | Standard library only, except the model client in Slice 4. |
| Review slicing | Six PRs (§5). Every slice ends at a green `go test ./...`. |

Deferred: live API auth and the whole confinement axis, multi-source adapters,
non-finance instantiations.

## 3. Design deltas from the handoff sketch

Four changes to the type sketch, agreed before implementation.

**a. "Undefined" is a return value, not an error.** The handoff had
`Invoke(args) (TypedValue, error)` with the note "error==Undefined is normal",
which contradicts the design call that not-in-scope must not surface as an
exception — an error leaks existence. Instead:

```go
type Result struct {
    Value   TypedValue
    Defined bool   // false => not in scope / not defined. Normal.
}

type Capability interface {
    Name() string
    Describe() ToolSchema
    Invoke(args map[string]any) (Result, error) // error == genuine fault only
}
```

**b. Plan validation is its own step, not part of the executor.** Before
anything runs: the DAG is acyclic, every `$ref` resolves, every argument
typechecks against the tool schema, and every entity/metric/period is in the
allowlist. This is the correctness-axis analogue of the mint step — the place
where a malformed or out-of-scope model plan dies before touching data. A
rejected plan produces a structured reason and **zero** tool invocations.

**c. The gate is structural, not a scan.** The narrator returns:

```go
type Claim struct {
    Token  string // "v3"
    ProvID string // provenance id of the value that backs it
}
type Draft struct {
    Template string  // "AAPL's gross margin rose from {{v1}} to {{v3}}."
    Claims   []Claim
}
```

The gate verifies that every token in the template has a claim, every claim
resolves to an executed non-stale value in the ledger, and no claim is unused.
Only then does it substitute. The narrator writing a number is not
*detected* — it is *unrepresentable*, because the narrator's output type has
nowhere to put one. This is invariant #2 enforced by the type system rather than
by a regex over prose. (A backstop scan of the rendered text for stray numerals
is cheap to add later if the model turns out to smuggle digits into prose; not
in the initial contract.)

**d. Staleness: content hash and provenance walk are complementary.** The hash
(`hash(op, sorted(input hashes))`) is the cheap *detection* mechanism — a value
is stale iff its recomputed input hash differs from the stored one. The forward
walk over the provenance graph is what produces the human-readable *report*
("these three sentences are out of date because `AAPL.revenue.FY2025` changed").
Slice 6 builds both.

One semantic detail this surfaces early: **comparing a ratio to a ratio is not a
percent change.** `gross_margin` latest vs. its own three-year average is a
difference in *percentage points*, not `pct_change`. So the op set is
unit-aware: `delta_pp` for ratio-typed metrics, `pct_change` for level metrics
like revenue, and the ops reject unit mismatches. This makes `Unit` load-bearing
rather than decorative, and is the main argument for getting it right in Slice 1.

## 4. Shape

```
experiments/grounded-pipeline/
  cmd/pipeline     // wiring + the sample query
  model            // TypedValue, Prov, Result, Cell, Unit, Plan, PlanStep
  store            // DataSource, DatasetVersion {raw, extracted}, redirectable resolver
  semantic         // metric defs (stored vs derived), resolvePeriod, listMetrics, allowlists
  tools            // capability surface: get_metric + ops
  plan             // validator: acyclicity, ref resolution, typecheck, allowlist
  planner          // Planner iface; StubPlanner, LLMPlanner
  executor         // runs a validated Plan DAG; writes to the ledger
  ledger           // provenance graph + dirty/staleness tracking
  narrator         // Narrator iface; StubNarrator, LLMNarrator
  gate             // token/claim verification + substitution
  fixtures         // v1.json, v2.json
```

**Dataset.** Two entities (AAPL, MSFT) × three fiscal years (FY2023–FY2025).

- Stored (7): `revenue`, `cost_of_revenue`, `gross_profit`, `operating_income`,
  `net_income`, `rnd_expense`, `diluted_shares`.
- Derived, semantic layer owns the formula (4): `gross_margin`,
  `operating_margin`, `net_margin`, `eps`.

**Tool surface.** `list_metrics()`, `resolve_period(text)`,
`get_metric(entity, metric, period)`, and the ops `mean`, `ratio`, `pct_change`,
`delta_pp`, `cagr`. Entities are constrained to `{AAPL, MSFT}` at mint time;
anything else returns `Defined: false`.

**Sample query** (the whole build targets this):

> How did gross margin move for AAPL and MSFT over the last three fiscal years,
> and how does each company's latest year compare to its own three-year average?

It exercises two entities, stored and derived metrics, period resolution, math
ops, and multi-value narration.

## 5. Slices

Each is one PR, independently reviewable, ending green.

### Slice 1 — Foundation: types, fixtures, semantic layer
Module scaffold, `model`, `store` (fixture-backed `DataSource`), `semantic`, v1
fixtures.
**Done when:** every stored cell loads; each derived metric resolves to its
input cells; unknown entity/metric/period returns `Defined: false` with no error.
**Reviewing:** are the core types right, and is provenance recorded at the right
granularity? Cheapest slice to be wrong in, most expensive to fix later.

### Slice 2 — Capability surface, validator, executor, ledger
`tools`, `plan` validator, `executor`, `ledger`, and `StubPlanner` with the
hardcoded plan for the sample query.
**Done when:** the stub plan executes to asserted numeric results; every
`TypedValue` traces to v1 cells; and each malformed plan — cycle, dangling
`$ref`, disallowed entity, argument type mismatch, unit mismatch — is rejected
with a distinct reason and zero tool invocations.
**Reviewing:** is the tool surface tight enough, and does the validator make
out-of-scope calls unrepresentable rather than merely caught?

### Slice 3 — Narrator, gate, cmd (completes Phase A)
`narrator` with `StubNarrator`, `gate`, `cmd/pipeline`.
**Done when:** `go run ./cmd/pipeline` answers the sample query end to end and
prints the provenance tree behind every number; a draft with an unbound token or
a claim pointing at a nonexistent provenance id is rejected.
**Reviewing:** this is the first full demo — the loop is real from here on.

### Slice 4 — Real planner and narrator (Phase B)
`LLMPlanner` and `LLMNarrator` behind the existing interfaces; tool schemas
generated from `Capability.Describe()`; recorded request/response fixtures so CI
stays deterministic, with the live path behind an env var.
**Done when:** the model builds a valid plan for the sample query *and* for one
unseen variant, and narration introduces no unbacked numbers.
**Reviewing:** does the tool surface survive contact with a real model?
**Blocked on:** model id, endpoint, and how to pass the key (see §6).

### Slice 5 — Versioning (Phase C)
`DatasetVersion = {raw, extracted}` where `extracted` is a deterministic
post-processing pass over `raw` (unit normalization, derived pre-materialization,
line-item mapping); redirectable resolver; v2 fixtures with a restated figure.
**Done when:** rebinding v1→v2 and re-running the *identical* plan object yields
updated numbers with updated `SourceRev`.
**Reviewing:** is the raw/extracted seam in the right place?

### Slice 6 — Dirty-bit (Phase D)
Content hashing, forward provenance walk, claim-level staleness, a "what's
stale" report, and stale-subgraph-only recompute.
**Done when:** changing one cell yields the exact set of stale values *and*
stale narrated sentences, each naming the changed cell; recompute touches only
the stale subgraph while unaffected values keep their identity.
**Reviewing:** precision of invalidation. This is the payoff slice.

## 6. Open items

- **Slice 4 credentials.** Needed before that slice starts: model id, endpoint,
  and the mechanism for supplying the key. Slices 1–3 are unblocked and do not
  need it, and Slices 5–6 can proceed against `StubPlanner` if the key is
  delayed.
- **v2 restatement choice.** Which figure changes between v1 and v2 — picked in
  Slice 5 to make the dirty-bit demo maximally legible (one cell, wide blast
  radius through derived metrics and narration).
