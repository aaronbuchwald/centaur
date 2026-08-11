# Grounded Data Pipeline — Component Interfaces

Companion to `ROADMAP.md`. This pins down each component's interface and,
critically, **who may call what** — the access rules are the design.

## 1. Framing: plan = program, verifier = compiler, executor = interpreter

The planner does not call tools. It emits a **script** — a small DAG-shaped
program in which steps name tools and reference each other's outputs. The
verifier is that language's **compiler front-end**: name resolution, type and
unit checking, scope checking against the bound data environment. The executor
is its **interpreter**, and the only component that ever invokes a tool. The
LLM's relationship to the tool suite is entirely descriptive: it sees schemas
and safe data references, never callables.

A consequence worth making load-bearing: because ops take only step references
and `get_metric` takes only names (entity/metric/period strings), **the plan
language has no numeric literal**. `Arg` is `string | Ref` — there is no
syntax by which the planner can inject a number into the computation. "The plan
is structure, not values" stops being a convention and becomes a property of
the type.

## 2. The environment (the "job" binding)

An `Env` is the intermediate environment a job runs in: a dataset revision +
its semantic layer + the minted tool registry, bound together. Everything else
is expressed against it.

```go
// engine
type Env struct{ /* unexported: rev-bound DataSource, semantic layer, registry */ }

func Bind(src store.DataSource, reg Registry) Env

func (e Env) Catalog() model.Catalog                       // planner's world
func (e Env) Verify(p model.Plan) (ValidatedPlan, []Violation)
func (e Env) Run(v ValidatedPlan, l *Ledger) (Execution, error)
func (e Env) Render(d model.Draft, x Execution, l *Ledger) (Answer, []Violation)
```

Re-binding v1→v2 (Slice 5) means constructing a new `Env`; the same `Plan`
verifies and runs against it unchanged.

## 3. The tool: one object, three views

```go
// engine (interface value never leaves this package)
type Capability interface {
    Schema() model.ToolSchema
    Invoke(args map[string]model.ArgValue) (model.TypedValue, bool, error)
    // ok=false => not defined / not in scope (normal); err => genuine fault.
}

type Registry struct{ /* unexported name → Capability map */ }
```

| Caller | View | Mechanism |
| --- | --- | --- |
| Planner | descriptive only | receives `Catalog` — plain data extracted via `Schema()`. Never holds a `Capability` value, so there is nothing to invoke. |
| Verifier | static only | checks plan steps against the same schemas + env manifest. Also never invokes. |
| Executor | dynamic | the sole caller of `Invoke`, via registry lookup inside `engine`. |
| Narrator | none | tools do not exist at narration time; only executed values do. |

The registry's lookup is unexported: outside `engine`, a `Capability` is
unreachable as a value. That is invariant #1 (LLM touches capability
descriptions in, `TypedValue` out) implemented as package visibility rather
than convention.

## 4. Planner

```go
// model — all plain data
type Catalog struct {
    Tools    []ToolSchema  // name, arg specs (kinds + unit constraints), return spec, doc
    Entities []string
    Metrics  []MetricInfo  // name, unit class, stored|derived, doc
    Periods  []string      // ordered; latest marked
}

type Ref string
type Arg struct{ Str *string; Ref *Ref }   // no numeric literal — see §1
type PlanStep struct {
    Ref  Ref
    Tool string
    Args map[string]Arg          // ref-list args (e.g. mean's inputs) are []Ref
}
type Plan struct {
    Steps   []PlanStep
    Outputs []Ref                // the steps whose values answer the question
}

// planner
type Planner interface {
    Plan(question string, cat model.Catalog) (model.Plan, error)
}
```

`Catalog` is exactly "the list of tools plus the data references that are safe
to use": the planner's entire observable universe. Data in, data out; the
planner package imports `model` and nothing else. `Plan.Outputs` lets the
planner declare which values answer the question — the narrator focuses there,
while intermediates remain available for citation.

## 5. Verifier (the compiler)

```go
func (e Env) Verify(p model.Plan) (ValidatedPlan, []Violation)

type ValidatedPlan struct{ plan model.Plan } // unexported field
type Violation struct{ Step model.Ref; Code, Detail string }
```

Checks: tool names exist; arity and arg kinds match schemas; every `Ref`
resolves to an earlier step (implies acyclic); unit constraints satisfied
(e.g. `pct_change` rejects ratio-typed inputs); every entity/metric/period
literal is in the env's manifest; `Outputs` non-empty and resolvable.
Violations accumulate — the planner gets the full error list in one round.

`ValidatedPlan` is constructible **only** by `Verify` (unexported field, same
package as `Run`), and `Run` accepts only `ValidatedPlan`. "Verify before
execute" is therefore not a calling convention — an unverified plan cannot
reach the executor by construction.

## 6. Executor and its result format

```go
func (e Env) Run(v ValidatedPlan, l *Ledger) (Execution, error)

type Execution struct {
    ID      string
    Rev     int                             // dataset revision this ran against
    Values  map[model.Ref]model.TypedValue  // every step's output, intermediates included
    Outputs []model.Ref                     // echoed from the plan
}
```

The result is **not** a bare value or a positional list — it is the full
ref → `TypedValue` binding of the executed script, stamped with the revision:

- keyed by step ref, so narration claims can name values stably;
- includes intermediates, because the narrator may cite them and staleness
  tracking must cover them;
- every value carries `Prov`; the executor appends the provenance nodes
  (including source nodes for fetched cells) to the ledger as it runs;
- `ID` + `Rev` make the execution itself a first-class provenance object —
  the unit that Slice 6 marks stale.

There is no representation in which a number appears without its receipt.

## 7. Narrator and the gate

```go
// model
type Fact struct {                 // one row of the narrator's view
    Ref   model.Ref
    Value float64
    Unit  model.Unit
    Label string                   // "AAPL gross_margin FY2025", "MSFT 3yr avg"
}
type Claim struct{ Token string; Ref model.Ref }
type Draft struct {
    Template string                // "AAPL's margin rose from {{v1}} to {{v3}}."
    Claims   []Claim
}

// narrator
type Narrator interface {
    Narrate(question string, facts []model.Fact) (model.Draft, error)
}

// engine
func (e Env) Render(d model.Draft, x Execution, l *Ledger) (Answer, []Violation)

type Answer struct {
    Text   string
    Claims []BoundClaim            // token → ref → prov id → value: the receipt
}
```

The narrator sees values (it needs them to say "rose" vs. "fell" truthfully)
but speaks in **execution-local refs**, not ledger IDs — the gate translates
ref → provenance when binding claims. Its output type still has nowhere to put
a number.

**Is further verification necessary at narration time? Yes.** The narrator is
the second untrusted boundary, and the gate is its compiler:

1. Template ↔ claims closed both ways: every token claimed, no unused claims,
   no malformed tokens.
2. Every claimed ref exists in *this* execution.
3. No claimed value is stale at render time (Slice 6 makes this bite).
4. **The renderer formats the numbers**, deterministically per unit (ratio →
   `45.9%`, pp-delta → `+1.4 pp`, currency → `$391.0B`). The narrator does not
   even choose formatting.

Residual channels, named deliberately: the narrator could smuggle a digit into
prose words ("roughly forty-five percent") or make a false *qualitative* claim
("rose" when it fell). The first has a cheap backstop (numeral/number-word scan
of the final text) if a real model turns out to do it; the second —
direction/magnitude verification against claimed values — is genuinely
interesting and explicitly out of scope for this build.

## 8. Access matrix

| Component | Receives | Returns | May call | Must never touch |
| --- | --- | --- | --- | --- |
| Planner (LLM) | question, `Catalog` | `Plan` | nothing | store, semantic, tools, ledger |
| Verifier | `Plan`, env manifest + schemas | `ValidatedPlan` \| violations | schema/manifest reads | `Invoke`, store |
| Executor | `ValidatedPlan`, ledger | `Execution` | `Invoke` (sole caller), ledger append | model client |
| Tool impls | typed args | `TypedValue, ok, err` | semantic layer (→ store) | ledger, planner, narrator |
| Semantic layer | entity/metric/period names | cells, derived values, manifest | store `Get` | everything above it |
| Store | keys | `Cell, ok` | fixtures | everything |
| Narrator (LLM) | question, `[]Fact` | `Draft` | nothing | ledger, tools, store |
| Gate | `Draft`, `Execution`, ledger | `Answer` \| violations | ledger reads | model client |
| Driver (`cmd`) | question | final `Answer` | all of the above, in order | — |

Both LLM components have the same shape: plain data in, plain data out, zero
callable surface. Everything between them is deterministic and provenance-
preserving.

## 9. Deltas vs. ROADMAP.md

Refinements this document makes to the roadmap sketch (roadmap remains
authoritative for scope and slicing):

- `Claim` binds token → **`Ref`** (execution-local), not ProvID; the gate
  translates to provenance when binding. Narrator surface shrinks.
- `Plan` gains `Outputs []Ref`.
- Plan literals are strings only — numeric literals are unrepresentable (§1).
- `Env` is first-class: catalog, verify, run, render are methods on the job's
  bound environment; rebinding is constructing a new one.
- `ValidatedPlan` with an unexported field makes verify-before-execute a
  compile-time property (§5).
- The renderer owns number formatting (§7.4).
