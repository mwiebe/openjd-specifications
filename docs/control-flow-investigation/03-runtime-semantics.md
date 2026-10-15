# CONTROL_FLOW investigation — 03: Scheduler and session-runtime semantics

Status: investigation note (input to the CONTROL_FLOW RFC draft). Not normative.

This note works through what a **scheduler** (the render-management system that
turns a Job Template into Steps, Tasks, and Sessions) and a **session runtime**
(the worker-host component modelled by `openjd-sessions`) must do to support
dynamic control flow: typed step output variables, dynamic parameter spaces,
conditional steps, computed task-task DAGs, and fan-in reductions.

It is grounded in two reference models:

* **`wiki/How-Jobs-Are-Run.md`** — the current scheduling and Session model, the
  `openjd_env` / `openjd_unset_env` stdout protocol, and the rule that *all
  failures and cancellations are terminal for the Session*.
* **The `openjd-sessions` crate** in `~/projects/rfc-control-flow/openjd-rs`
  (read-only here) — the concrete, shipping implementation of the stdout
  directive protocol (`action.rs`, `action_filter.rs`, `session.rs`) that the
  proposed `openjd_output_var:` directive must mirror, and the progressive
  `unresolved[T]` evaluation model from the EXPR work (RFCs 0005/0006/0007).

Everything here builds on accepted EXPR behaviour rather than redefining it. In
particular it relies on three EXPR facts:

1. **Three evaluation stages** (RFC 0005 "Evaluation Stages" / Template Schemas
   §7.4): *template validation*, *job creation (submit)*, *task execution*. At
   each stage a symbol is either a concrete value or an `unresolved[T]`
   placeholder, and the *same* evaluator runs at every stage.
2. **Symbol collection** — an expression's external symbol references can be
   extracted without evaluating it, which is what lets a scheduler build the
   data-flow graph statically.
3. **Typed values with implicit, non-destructive coercion** — including the
   `range_expr` type and `list[T]` / `list[list[int]]` types from RFC 0007 that
   a computed parameter space or adjacency list needs.

The central new idea is a fourth logical point on the stage timeline that sits
*between* job creation and task execution and recurs per step:

> **Step input binding** — the moment at which a step's upstream output
> variables become concrete, so its own TEMPLATE-scope expressions (parameter
> space ranges, `enableIf`, task-task adjacency) can finally be evaluated.

The whole of CONTROL_FLOW is, in scheduler terms, the machinery for reaching and
acting on that moment.

---

## 1. Step lifecycle: the "pending inputs" state

### 1.1 Today's lifecycle

In the 2023-09 model a Step is schedulable as soon as its `dependencies` have
completed successfully; its parameter space is fully known at **job creation**
because every `range` resolves from job parameters and literals. The scheduler
materializes all Tasks of all Steps up front and only *gates* them on dependency
completion. There is no notion of a Step whose Tasks are not yet enumerable.

### 1.2 The new state

CONTROL_FLOW breaks the "parameter space is known at job creation" assumption:
a range like `"1-{{Step.VideoToFrames.FrameCount}}"` cannot be evaluated until
`VideoToFrames` has run and emitted `FrameCount`. We therefore need a step
lifecycle state the current model lacks. Proposed name: **`PENDING_INPUTS`**
(discussion #111 calls it "unable to evaluate yet").

A Step is in `PENDING_INPUTS` when at least one of its *materialization
expressions* references an output variable (`Step.<Name>.<Var>`) that is not yet
concrete. Materialization expressions are the TEMPLATE-scope expressions whose
values the scheduler needs in order to enumerate Tasks and edges:

* `parameterSpace.*.range` and `combination`
* `enableIf` (if present on the step)
* task-task `dependsOnSubspace` adjacency ranges (§6)

> Note the scope boundary. `Task.Param.*`-dependent and `Session.*`-dependent
> expressions are *host-context* and were always unresolved at job creation;
> they are not what gates `PENDING_INPUTS`. Only `Step.<Name>.<Var>` references
> — a genuinely new symbol class — introduce the new wait.

Proposed step state machine:

```
                 (deps declared with Step.* refs)
   PENDING_INPUTS ───────────────► READY ───────► RUNNING ───────► SUCCEEDED
        │   ▲                        ▲                │   │              │
        │   │ upstream var           │ all inputs     │   └── FAILED ────┘
        │   │ not yet concrete       │ concrete       │
        │   └────────────────────────┘                └── (enableIf false) ──► SKIPPED
        │
        └── upstream FAILED / SKIPPED and input is required ──► FAILED (or SKIPPED; see §5)
```

A step with no `Step.*` references in any materialization expression never
enters `PENDING_INPUTS`; it behaves exactly as today. This keeps CONTROL_FLOW a
zero-cost addition for existing templates (mirrors how `openjd-sessions` keeps
the non-WRAP_ACTIONS path a zero-cost addition).

### 1.3 When does the parameter space get materialized?

**Proposal: lazily, at step input binding, exactly once, after all upstream
producers that feed the step's materialization expressions have reached a
terminal state and their variables are concrete.**

Concretely the scheduler:

1. At job creation, resolves everything it can (EXPR stage 2) and runs symbol
   collection over each step's materialization expressions. Steps whose
   collected symbols include an unresolved `Step.*` enter `PENDING_INPUTS`;
   all others enter `READY`/gated as today.
2. When an upstream step reaches `SUCCEEDED` and all its declared output
   variables are concrete (§3.4), the scheduler re-evaluates each dependent's
   materialization expressions with those `Step.*` symbols now bound.
3. When a dependent's *last* unresolved materialization input becomes concrete,
   it transitions `PENDING_INPUTS → READY`, the Task set and interior edges are
   enumerated, and normal scheduling resumes.

Discussion #111 explicitly allows two implementation strategies and the RFC
should permit both:

* **Deferred creation** — do not create the step's Tasks until inputs are
  available (what a stateless scheduler naturally does).
* **Create-then-update** — create the step in `PENDING_INPUTS` and fill in the
  Task set when values arrive (what a database-backed render manager naturally
  does, so the job's shape is visible in the UI immediately).

Both are observationally equivalent to a job author; the RFC should specify the
*observable* contract (a step's Tasks are enumerable iff its materialization
inputs are concrete) and leave the strategy to the scheduler.

### 1.4 Dependency failure

If a producer step **fails**, every dependent whose materialization expression
reads one of that producer's output variables can never have its inputs bound.
Default proposal: the dependent transitions to a terminal **blocked/failed**
state and does not run — consistent with today's rule that a `dependsOn` step
must "fully complete successfully" before dependents schedule. See §5 for the
interaction with conditional/skip semantics, which is the one place the author
may *want* a non-fatal outcome.

### 1.5 Retry and re-run idempotency — the hard case

This is the subtlest runtime question. An output variable is produced by a Task;
Tasks are retried on transient failure, and render managers routinely let an
operator *re-run* an already-succeeded Task. Both can cause a variable that was
already emitted to be emitted again, possibly with a different value, possibly
*after* downstream steps have already consumed the old value and created Tasks.

The design tension: idempotency (re-running a Task should be safe) versus
correctness (a downstream parameter space built from a stale value is wrong).

Proposed rules, ordered from least to most disruptive:

* **Within a single Task attempt — last write wins.** If one Task emits
  `openjd_output_var: X=1` then `X=2` on stdout, the final value is `2`. This
  mirrors `openjd_env` exactly: `apply_message` in `session.rs` inserts into the
  env map, so a later directive overwrites an earlier one. Rationale: a script
  computing a value in stages is normal; forcing single-emission would be
  hostile. (Multi-*task* same-var is different — see §3.6.)
* **A declared variable not emitted by the end of a Task is an error.** The
  Session fails the Task. This matches discussion #111 ("If a task runs and does
  not output a declared variable, that's an error") and mirrors how a malformed
  `openjd_env` cancels the action in `action_filter.rs`.
* **Retry of a *failed* Task re-collects outputs from scratch.** A failed Task
  produced no committed outputs (its Session failed and is terminal), so its
  retry starts clean. No idempotency problem — there was no prior committed
  value.
* **Re-run of a *succeeded* producer after consumers materialized — reject by
  default; opt-in recompute is out of scope for v1.** If the new value equals
  the old, treat as a no-op. If it differs, the honest default is to **fail the
  re-run with a clear error** ("output variable `X` already consumed by step
  `Y`; changing it would require re-materializing downstream steps"), rather
  than silently leaving a half-stale DAG. Cascading recompute (invalidate and
  re-materialize all transitive dependents) is powerful but is a scheduler
  policy with large blast radius; v1 should *specify the safe default* (reject /
  no-op) and leave cascade as a documented, optional scheduler capability. This
  keeps the first version analyzable and avoids baking a re-run storm into the
  spec.

> Recommendation for the RFC: specify last-write-wins *within an attempt*,
> never-emitted-is-error, and "committed once a producing Task succeeds; a
> differing re-emission after consumption is an error unless the scheduler opts
> into cascade recompute." Flag the cascade policy as an open question (§9).

---

## 2. Emission protocol: `openjd_output_var:`

### 2.1 Mirror `openjd_env` deliberately

The existing stdout directives are parsed line-by-line by the session runtime.
In `openjd-sessions`:

* `action.rs` models each directive as an `ActionMessage` variant
  (`SetEnv { name, value }`, `UnsetEnv`, `Progress`, …).
* `action_filter.rs` recognizes a line via `parse_directive` (strict
  `openjd_<kind>: <payload>` with a literal `": "` separator), validates it,
  and emits a `FilterCallback`; near-miss lines like `openjd_env foo=bar` are
  detected as *malformed* and **cancel the action with a failure**.
* `session.rs` `apply_message` commits the parsed directive to Session state.

`openjd_output_var:` should slot into exactly this pipeline:

```
openjd_output_var: FrameCount=12
openjd_output_var: Scores=[1.2, 3.4, 5.6]
openjd_output_var: "Label=a=b"      # JSON-wrapped form for values containing '='
```

Add an `ActionMessage::SetOutputVar { name, json_value }` variant, a
`parse_directive` arm for `output_var`, and an `apply_message` arm that records
the value against the running Task (not the Session env map). The validation,
malformed-line→cancel, and redaction code paths are reused verbatim.

### 2.2 Payload grammar and the `name=` boundary

`openjd_env` resolves the `name=value` split at the *first* `=`, and supports a
JSON-string-wrapped form (`"NAME=VALUE"`) so that values may contain newlines,
leading/trailing spaces, or an `=`. `parse_env_variable` in `action_filter.rs`
does exactly this and additionally **rejects NUL bytes** in both the name and
the decoded value.

`openjd_output_var:` differs in one way: the *value* is JSON, not an opaque
string. Proposed grammar:

```
openjd_output_var: <VarName>=<JSON>
```

* `<VarName>` matches the output-variable identifier regex (same family as env
  var names: `[A-Za-z_][A-Za-z0-9_]*`), so the `=` boundary is unambiguous — the
  split is at the first `=`, and everything after it is a single-line JSON
  document.
* `<JSON>` is a JSON scalar or array on one logical line. For an `INT` output
  the JSON is `12`; for a `STRING`, `"hello"`; for `LIST[LIST[INT]]`,
  `[[0],[0,1]]`. This is unambiguous and tool-parseable (design tenet 4) and
  means the producer does not have to invent an ad-hoc serialization — exactly
  the complaint folouiseAWS raised ("packing numbers into `openjd_status`").
* Because the value is JSON, the `"NAME=VALUE"`-wrapping trick that `openjd_env`
  needs for awkward *string* values is unnecessary for the value; a string value
  that contains `=` is simply `"a=b"` in JSON. The JSON-wrapped *whole-directive*
  form may still be offered for parity, but is not required.

### 2.3 Size limits

The session runtime caps a logical stdout line at **64 KiB**
(`LOG_LINE_MAX_LENGTH` / `MAX_LINE_BYTES` in `subprocess.rs` / the helper
`framer.rs`); longer lines are truncated. An output-variable value is a single
line and is therefore already bounded by that cap. The RFC should:

* State explicitly that an `openjd_output_var:` value shares the 64 KiB
  per-line limit, and that a value truncated by that cap is a **Task failure**
  (truncated JSON will fail to parse — fail loudly rather than commit a
  truncated structure).
* Note that this makes output variables suitable for *control data* (counts,
  scores, adjacency lists, S3 keys) and explicitly **not** a bulk data channel.
  Large data continues to flow through `PATH`/`dataFlow: OUT` parameters and
  shared storage; CONTROL_FLOW moves *the shape of the job*, not its payloads.
  This should be called out as a design tenet alignment ("Expressive, but not a
  side-channel for arbitrary data").
* Permit a scheduler to impose a tighter policy cap, consistent with the
  existing `CallerLimits` / `SessionLimits` lever in `openjd-sessions`
  (`max_resolved_data_len` etc.). A new `max_output_var_bytes` lever is the
  natural home.

### 2.4 Type coercion to the declared type

Each output variable is declared with a `type` in the producing step's
`outputVariables`. When the runtime parses the emitted JSON it must coerce to
the declared type using the **same implicit, non-destructive coercion rules EXPR
already defines** (RFC 0005 "Implicit Type Coercion"):

* `INT` declared, JSON `12` → `int 12`. JSON `12.0` → `int 12` (non-destructive).
  JSON `12.5` → **error** (`int(12.5)` is destructive; RFC 0006 forbids it).
* `FLOAT` declared, JSON `12` → `float 12.0` (int→float is allowed).
* `STRING` declared, JSON `"abc"` → `string`. JSON `123` → **error** by default
  (keep it strict; a producer that wants a string emits a JSON string).
* `LIST[INT]` declared, JSON `[1,2,3]` → `list[int]`. A `range_expr` consumer
  reads it via `range_expr(Step.X.Y)` using the existing RFC 0006 conversion.
* `BOOL`, `RANGE_EXPR`, `LIST[...]`, `LIST[LIST[INT]]` follow the EXPR type for
  the lower-cased declared type, as RFC 0007 already specifies for parameters.

Reusing EXPR coercion (rather than inventing output-variable-specific rules) is
important: it means `Step.X.Y` behaves in expressions identically to a job
parameter of the same type, and the static type checker (§7) needs no new logic.

### 2.5 Declared but never emitted

A `Task` of a step with `outputVariables` **must** emit every declared variable
exactly once (last-write-wins resolves repeats within the attempt). If a Task
exits successfully without having emitted a declared variable, the Session
**fails the Task** with a clear message (`output variable 'X' was declared but
never emitted by step 'S'`). This is the discussion #111 rule and matches the
"fail-fast, failures are terminal" posture of the runtime.

### 2.6 Multiple tasks emitting the same variable (producer side)

A step may have N Tasks, each of which emits the step's declared output
variable(s). This is the normal case for a sweep (folouiseAWS's example: 270
tasks each emitting `mpjpe_mm`). The variable is therefore **not** a single
scalar at the step level — it is one value *per Task*. The step-level value a
consumer sees is a **reduction** over the per-Task values. This is the subject
of §3; the key runtime point is:

* The runtime collects, per step, a map `task_coordinates → declared_value` as
  Tasks complete. "Last write wins" applies *within a Task attempt*, never
  *across Tasks* — two different Tasks emitting `X` are two distinct data points,
  not a conflict.
* The step's reduced output variable(s) are not concrete, and dependents cannot
  leave `PENDING_INPUTS`, until **all** Tasks of the producing step are terminal
  (so the reduction domain is complete). A single-Task step is just the N=1 case
  and its reduction is trivially that one value.

---

## 3. Reductions: N tasks → one logical value

### 3.1 The requirement

Discussion #111: "There will need to be some kind of reduction operation when a
step has multiple tasks. E.g. put all the variable values into a list in the
parameter space iteration order, take the maximum/minimum value, take the sum,
etc." folouiseAWS's sweep makes it concrete: sort 270 tasks' `mpjpe_mm` and pick
the winner.

### 3.2 Where is the reduction declared — producer or consumer?

**Proposal: the base datum is producer-declared; the reduction is
consumer-expressed.**

* The **producer** step declares, per output variable, its *per-task element
  type* (`type: FLOAT` means "each Task emits a float"). It does **not** declare
  how the values are combined.
* The step-level symbol `Step.<Producer>.<Var>` is then a **`list[T]` in
  parameter-space iteration order** — the natural, lossless aggregation. The
  iteration order is already well-defined by the `combination` expression and
  `StepParameterSpaceIterator` in `openjd-model`, so "iteration order" is a
  precise, implementable contract, not a vague one.
* The **consumer** applies whatever reduction it needs using the **existing EXPR
  function library** (RFC 0006): `max(Step.Sweep.mpjpe_mm)`,
  `min(...)`, `sum(...)`, `len(...)`, `sorted(...)`, `any(...)`, `all(...)`,
  list comprehensions, slicing, `"".join(...)`, etc.

Worked example (pick-the-winner):

```yaml
# Producer: Sweep step, FLOAT per task
outputVariables:
  - { name: mpjpe_mm, type: FLOAT }
# Consumer: a RunWinner step that depends on Sweep
let:
  - best = min(Step.Sweep.mpjpe_mm)                       # scalar reduction
  - best_index = [i for i, v in enumerate(Step.Sweep.mpjpe_mm) if v == best][0]
```

(`enumerate` is illustrative; if EXPR lacks it, the index can be recovered with
a comprehension over `range(len(...))` — see §9.)

### 3.3 Is "bare list + EXPR functions" enough?

**Largely yes, and that is the recommendation for v1.** A list in iteration
order plus the RFC 0006 function library already expresses min/max/sum/any/all,
concatenation (`join`), de-dup (`unique`), sort, and arbitrary comprehensions.
This keeps CONTROL_FLOW small and leans on accepted work (design tenets:
expressive, human-readable, tooling-parseable) instead of inventing a reduction
mini-language.

Two gaps to flag rather than solve now:

* **Correlating the reduced value back to the winning task's coordinates.**
  `min()` gives the best score but not *which* `(Video, Window, Overlap,
  DetConf)` produced it. The list-in-iteration-order contract makes this
  recoverable by index (the i-th element corresponds to the i-th point of the
  parameter space, which the consumer can reconstruct), but it is awkward. A
  future convenience — e.g. exposing `Step.<Producer>.<Var>` as a list of
  `{coords, value}` records, or an `argmin`/`argmax` — is worth an open question,
  not v1 scope.
* **Multi-variable joint reduction** (pick the row minimizing `mpjpe_mm` subject
  to `fps > 20`) needs the per-task values of *several* output variables aligned
  by task. The iteration-order contract gives this for free (all of a step's
  output-variable lists share the one iteration order), but the ergonomics are a
  comprehension zipping two lists; acceptable for v1, candidate for sugar later.

### 3.4 "Concrete" for a reduced variable

A reduced output variable is concrete (and so can release a `PENDING_INPUTS`
dependent) **only when every Task of the producing step is terminal**, because
the list is defined over the whole parameter space. Partial reductions are not
offered in v1 (they would make "iteration order" ill-defined mid-flight).

---

## 4. Conditional steps (`enableIf`) — skip semantics

Discussion #81 proposes `enableIf: <expr>` on a step/task. The runtime questions
are entirely about what a **skipped** step means to the rest of the graph.

### 4.1 When is `enableIf` evaluated?

`enableIf` is a materialization expression (§1.2): if it references
`Step.*.*`, the step waits in `PENDING_INPUTS` until that input is concrete,
then `enableIf` is evaluated. Result `false` → the step is **`SKIPPED`** and no
Tasks run. Result `true` → normal scheduling.

### 4.2 Does a skipped step count as succeeded or absent — for scheduling?

**Proposal: for pure `dependsOn` gating, a `SKIPPED` step satisfies the
dependency (treated as "done"), so a dependent that only orders itself after the
skipped step still runs.** Rationale: `dependsOn` expresses *ordering*, and a
skipped step has nothing left to order against. Treating skip as a hard block
would make conditional branches poison every downstream step, which is the
opposite of the author's intent when they wrote `enableIf`. This matches the
mainstream (Argo `when`, GitHub Actions `if:` with `needs`, Tekton `when`) — a
skipped node does not fail its dependents.

### 4.3 What do a skipped step's *output variables* evaluate to?

This is the sharp edge. A step can be *skipped* yet be *named* in a downstream
`Step.<Skipped>.<Var>` reference. Options:

* (a) The reference evaluates to `null`/`None`. Clean in EXPR's optional-type
  model (`T?`), and lets a consumer write
  `Step.X.Count if Step.X.Count != null else 0`. Requires the output-variable
  symbol to carry type `T?` whenever its producer is conditionally enabled.
* (b) Referencing an output variable of a skipped step is an **error**. Simple,
  but brittle — it forces every consumer to also be conditional, defeating the
  purpose.
* (c) The producer declares a **default** per output variable, used when the
  step is skipped. Most ergonomic, costs a schema field.

**Recommendation: (a) as the base rule, (c) as optional sugar.** A
`Step.<Name>.<Var>` reference has type `T` when the producer is unconditional
and type `T?` when the producer (transitively) carries an `enableIf`; the static
type checker already propagates `T?` and will *force* the consumer to handle the
`null` (via `?`-aware coercion, a default, or `fail()`), which is exactly the
fail-fast behaviour we want. A producer may additionally declare a default to
collapse `T?` back to `T`. Option (b) is rejected as too viral.

### 4.4 Skip and dependency *data* flow vs *ordering* flow

Note the asymmetry: a skipped step satisfies **ordering** (`dependsOn`, §4.2)
but yields **absent data** (`Step.X.Var` is `null`, §4.3). These are two
different edges and the RFC must keep them distinct. A dependent that *only*
orders after the skipped step runs normally; a dependent that *reads data* from
the skipped step must cope with `null`. §1.4's "dependency failure blocks
dependents" rule therefore applies to **failure**, not to **skip**.

---

## 5. Task-task dependencies with computed adjacency (`dependsOnSubspace`)

Discussion #82 proposes task-task edges within a step via `dependsOnSubspace`,
with the adjacency given as a `LIST[LIST[INT]]`. Discussion #111 extends it: the
adjacency list is *computed by a prior step* and flows in via
`Step.Analyze.AdjList`.

### 5.1 Two DAGs, two validation times

There are two distinct graphs:

* The **step DAG** (steps linked by `dependencies`). This is fully static — it
  is in the template text — and must be acyclic at **template-validation time**.
  `openjd-model` already does this: `detect_dependency_cycles` in
  `validate_v2023_09/structure.rs` and `StepDependencyGraph` in
  `job/step_dependency_graph.rs`. CONTROL_FLOW adds nothing to *when* this is
  checked.
* The **intra-step task DAG** (tasks of one step linked by `dependsOnSubspace`).
  When the adjacency is a literal/job-parameter (#82's original case) this is
  knowable at **job creation**. When the adjacency is `Step.Analyze.AdjList`
  (#111's extension) it cannot exist until `Analyze` has run — it is a
  **materialization expression**, so the step sits in `PENDING_INPUTS` until the
  adjacency is concrete, and the task DAG is built at **step input binding**.

### 5.2 What must be true *before any task runs* vs *after the producer finishes*

| Property | Checked at | Why |
|---|---|---|
| Step DAG is acyclic | template validation | static text |
| `Step.<N>.<V>` references point at a declared output var of a step this step depends on | template validation | symbol collection + the output-var declarations are static (§7) |
| Types of `Step.<N>.<V>` references are compatible | template validation | EXPR static type checking |
| The adjacency expression evaluates without error | step input binding (after producer `SUCCEEDED`) | needs the concrete value |
| The **computed intra-step task DAG is acyclic** and indices are in range | step input binding, *before any Task of this step is dispatched* | the edges only exist now |
| The parameter-space size matches the adjacency length | step input binding | both derive from the same upstream value; cross-check |

The critical rule: **a computed intra-step DAG must be fully validated — acyclic,
in-range indices, consistent sizing — at materialization time, before the
scheduler dispatches *any* Task of that step.** A cycle discovered after some
tasks have already run is unrecoverable (the Session-is-terminal rule gives no
way to "unrun" them). So materialization is an atomic gate: evaluate adjacency →
validate the resulting graph → either admit the whole step to `READY` or fail
the step as a unit with a precise error (`computed task graph for step 'S'
contains a cycle: 2 -> 5 -> 2`).

### 5.3 Cycle detection at materialization time

The scheduler reuses the same algorithm the model already uses for the step DAG
(DFS with a stack, building the cycle path for the error message — see
`step_dependency_graph.rs`), but over the *materialized* task adjacency rather
than the static step list. The RFC should state that the complexity budget is
the author's responsibility (an adjacency of N tasks with E edges is an O(N+E)
check) and that a scheduler may impose a `max_tasks_per_step` / `max_edges`
policy cap, consistent with existing caps.

### 5.4 Self-dependency

#82/#111 model intra-step edges as a `dependsOn: <SameStep>` with a
`dependsOnSubspace`. The model must permit a step to name itself *only* in this
subspace form (a plain `dependsOn: Self` remains a cycle error). This is a
targeted exception the validator must encode: a self-edge is legal iff it
carries a `dependsOnSubspace`, and it expresses *task-task* ordering, not
*step-step* ordering, so it does not create a step-DAG cycle.

---

## 6. Static validation vs. runtime validation

The guiding principle (design tenet 4 + EXPR's fail-fast requirement): **check
everything that is a property of the template text at validation time; defer only
what genuinely requires a runtime value.**

### 6.1 Checkable at template-validate time (model library)

* **Output-variable declarations are well-formed** — names unique within a step,
  types are valid parameter types.
* **Every `Step.<N>.<V>` reference resolves** — `N` is a declared step, `V` is a
  declared output variable of `N`. Pure symbol-table lookup; the output-var
  declarations are static.
* **Dependency-direction check** — a `Step.<N>.<V>` reference is only legal from
  a step that (transitively) `dependsOn` `N`. You may not read data from a step
  you are not ordered after; otherwise the value could be read before it exists.
  This is a graph reachability check over the static step DAG and is a **new,
  important validation** CONTROL_FLOW must add. (It is the data-flow analogue of
  the existing cycle check.)
* **Type compatibility** — `"1-{{Step.X.FrameCount}}"` type-checks with
  `Step.X.FrameCount : unresolved[int]`, using the existing EXPR evaluator with
  the declared output-var type as the placeholder type. A string method on an
  `INT` output var fails here, before submission, exactly as RFC 0005 shows for
  parameters.
* **Step DAG acyclicity** — unchanged, as today.
* **Extension gating** — the template requests `CONTROL_FLOW` and the validator
  only accepts the new fields when it is present (§7).

### 6.2 Only checkable at run time (scheduler / session runtime)

* **The concrete value of an output variable** — its magnitude, list length,
  adjacency contents.
* **Parameter-space size and whether it is empty** — `"1-{{Step.X.FrameCount}}"`
  with `FrameCount = 0` is an empty space; the RFC must say whether an empty
  computed space is legal (recommend: legal, step becomes a no-op `SUCCEEDED`,
  so a video with 0 frames doesn't fail the job) — an open question (§9).
* **Computed task-DAG acyclicity and index ranges** — §5.2.
* **Whether a declared output variable was actually emitted** — §2.5.
* **Coercion that depends on the concrete payload** — e.g. a `STRING` output
  consumed as `INT` type-checks as `unresolved[int]` but may fail to parse at
  run time, consistent with EXPR's deliberately asymmetric unresolved-coercion
  rule (RFC 0005: the type level may accept a pair the concrete value later
  rejects, never the reverse).

### 6.3 How "unresolved until run time" is represented

No new mechanism is needed. `Step.<N>.<V>` is just another symbol that is
`unresolved[T]` until a stage binds it — the fourth stage from the intro ("step
input binding"). The model library seeds the symbol table with
`ExprValue.unresolved(declared_type)` for every output variable and the existing
progressive evaluator does the rest. This is the single most important reuse in
the whole proposal: **output variables are not a new type system, they are a new
symbol class on the existing one.**

---

## 7. Backward compatibility via the `extensions` list

CONTROL_FLOW is breaking: a 2023-09 scheduler that has never heard of
`outputVariables`, `openjd_output_var:`, `Step.<N>.<V>`, `enableIf`, or
`dependsOnSubspace` must **reject the template cleanly rather than mis-run it**.
RFC 0002 already provides the mechanism and the RFC should lean on it entirely:

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [CONTROL_FLOW]        # plus EXPR, and whatever else
```

* The extension name is `CONTROL_FLOW` (regex `[A-Z0-9_]{3,128}`, per RFC 0002).
* A scheduler that does not implement `CONTROL_FLOW` **must reject** a template
  listing it (RFC 0002: unsupported requested extensions are a submission error)
  — not silently ignore the field. This is the clean-rejection guarantee.
* The new schema fields (`outputVariables`, `enableIf`, `dependsOnSubspace`,
  dynamic `Step.*` ranges) are each annotated `@extension CONTROL_FLOW` in the
  Template Schemas, so a validator *with* the extension accepts them and one
  *without* it treats their presence as invalid — the same pattern RFC 0001
  (`CHUNK[INT]`), 0007, and 0008 already use.
* **Dependency on EXPR.** `Step.<N>.<V>` references live inside format-string
  expressions and the reductions use the EXPR function library, so a template
  using CONTROL_FLOW will in practice also list `EXPR`. The RFC should state the
  relationship (CONTROL_FLOW's expression surface requires EXPR) and whether a
  scheduler must enforce "EXPR present if CONTROL_FLOW present" — recommend yes,
  flagged as an open question (§9). Likewise the structured outputs
  (`LIST[LIST[INT]]` adjacency) require the RFC 0007 types.
* The `openjd_output_var:` **stdout directive** deserves the same treatment the
  REDACTED_ENV_VARS extension got for redaction: because a mis-handled directive
  is a correctness/safety issue, a conforming runtime that sees
  `openjd_output_var:` while the extension is *not* active should treat it as a
  malformed directive (fail the Task) rather than echo it as plain stdout — so an
  output variable is never silently dropped on a non-supporting runtime. (Open
  question §9: mirror REDACTED_ENV_VARS's "honor even when the extension is not
  specified", or strict-reject.)

---

## 8. Session-runtime impact summary (concrete changes to the `openjd-sessions` model)

Collecting the above into the specific touch-points a prototype in
`openjd-sessions` would hit (feeds the prototype-plan task):

* `action.rs` — new `ActionMessage::SetOutputVar { name, json_value }` variant.
* `action_filter.rs` — new `parse_directive` arm for `output_var`; JSON payload
  validation; reuse of NUL rejection, malformed→cancel, redaction, and the
  64 KiB line cap. A declared-but-unknown or undeclared variable name is a
  malformed-directive failure.
* `session.rs` — `run_task` collects a per-Task map of emitted output variables;
  a declared-but-not-emitted variable fails the Task at Task end (new check
  alongside the existing success path); the collected values are surfaced to the
  caller (the scheduler) as part of the Task result (extend `ActionResult` /
  the task-completion callback). Output variables are recorded against the
  **Task**, not the Session env map — they outlive the Session and feed the
  scheduler's cross-step data flow.
* A new runtime limit `max_output_var_bytes` alongside `SessionLimits` /
  `CallerLimits`.
* **Scheduler-side (not `openjd-sessions`)** — the `PENDING_INPUTS` state, step
  input binding, per-step reduction to iteration-order lists, computed-DAG
  cycle/range validation, and the dependency-direction static check. The session
  runtime's only new job is *parse, validate-against-declared-type, and report*
  output variables; everything about *acting* on them is scheduler policy.

This split keeps the worker-host runtime minimal (it already does the hard
line-parsing work for `openjd_env`) and puts the genuinely new orchestration in
the scheduler, which is where the per-job state lives.

---

## 9. Open questions to carry into the design-space and RFC

1. **Re-run of a succeeded producer** whose value changes after consumers
   materialized: reject (safe default, recommended for v1) vs. opt-in cascade
   recompute. Where does cascade live — spec capability or scheduler-defined?
2. **Empty computed parameter space** (`FrameCount = 0`): legal no-op
   `SUCCEEDED` step (recommended) vs. error.
3. **Skipped-producer output-variable value**: base rule `null`/`T?`
   (recommended) and/or producer-declared default. Confirm the type checker
   forces consumers to handle `T?`.
4. **Correlating a reduced value to the winning task's coordinates** —
   is iteration-order-index recovery enough for v1, or do we want
   `argmin`/`argmax` or `{coords, value}` records? (folouiseAWS's winner pick.)
5. **EXPR dependency enforcement** — must a scheduler require `EXPR` (and the
   RFC 0007 types) whenever `CONTROL_FLOW` is present?
6. **`openjd_output_var:` on a non-supporting runtime** — honor-anyway (like
   REDACTED_ENV_VARS redaction) vs. strict malformed-reject.
7. **Does task-task dependency (`dependsOnSubspace`, #82) belong *inside*
   CONTROL_FLOW or beside it** as a separate extension CONTROL_FLOW depends on?
   The runtime analysis suggests the *computed-adjacency* case is inseparable
   from output variables (it is just a `Step.*`-typed materialization
   expression), while the *static-adjacency* case (#82 original) could stand
   alone. Candidate scoping: static `dependsOnSubspace` in its own small
   extension; computed adjacency as a CONTROL_FLOW capability layered on it.
8. **Reduction surface** — is "iteration-order list + RFC 0006 functions"
   sufficient, or do multi-variable joint reductions need `enumerate`/zip sugar
   in the EXPR function library?

---

## 10. Design-tenet check

* **Portable** — the `extensions` gate (§7) means a non-supporting scheduler
  rejects cleanly; nothing silently degrades. ✔
* **Expressive** — one mechanism (typed output variables + a new symbol class)
  covers scalars, lists, computed fan-out, computed DAGs, conditionals, and
  reductions, by composing with accepted EXPR work rather than special-casing. ✔
* **Human readable/writable** — `outputVariables` + `openjd_output_var:` +
  `Step.X.Y` read plainly; the reduction is ordinary EXPR. The one readability
  risk is index-based winner correlation (open question 4). ◑
* **Tooling parseable** — output variables are *declared* in the template, so
  data flow is statically analyzable (dependency-direction check §6.1), and the
  emission is a strict, JSON-valued, single-line directive. This is a strict
  improvement over the status-quo workarounds (files-per-task, packing into
  `openjd_status`, external trackers) that are opaque to tooling — the exact
  motivation folouiseAWS and crowecawcaw gave. ✔
