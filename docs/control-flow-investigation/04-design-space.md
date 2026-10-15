# CONTROL_FLOW investigation 04: Design space and v1 scope

Status: investigation note, input to the CONTROL_FLOW RFC draft. Not normative.

> **Superseded in part.** After review, the emission-channel decision in T1 was reversed: outputs are reported with
> an `openjd_output_var:` stdout directive, with an `@<file>` form for large values, and Task metrics were folded in
> as output variables with no consumer. See [§8](#8-addendum-emission-channel-collect-modes-and-metrics). The rest of
> this note, including its samples, still reflects the earlier file-based design where it mentions emission, and
> [§9](#9-addendum-path-outputs-and-job-outputs) adds `PATH` outputs and job outputs;
> [RFC 0010](../../rfcs/0010-control-flow.md) is current.

This note synthesizes the three earlier notes into one design for a `CONTROL_FLOW` extension:

* [01-prior-art.md](01-prior-art.md): OpenJD discussions #111, #81, #82, #22, #79, the use cases U1–U15, and what
  EXPR (RFCs 0005–0007) provides or lacks. Mechanism IDs M1–M14 below refer to that note.
* [02-external-prior-art.md](02-external-prior-art.md): Argo, Airflow, Nextflow, Snakemake, GitHub Actions, Tekton,
  Step Functions, CWL, Slurm, DAGMan, and lessons L1–L12.
* [03-runtime-semantics.md](03-runtime-semantics.md): step lifecycle, emission protocol, reductions, skip semantics,
  computed task DAGs, static vs. runtime validation.

Where those notes disagree, [§1](#1-tensions-between-the-investigation-notes-and-their-resolution) records the
decision and the reason. [§2](#2-the-model-one-new-stage-one-new-symbol-class) states the model,
[§3](#3-control-flow-taxonomy) walks the five components, [§4](#4-end-to-end-sample-templates) gives five sample
templates, [§5](#5-scoping-decision-for-v1) makes the scoping decision, [§6](#6-open-questions-for-the-discussion-thread)
lists the questions to raise before writing the RFC, and [§7](#7-design-tenet-check) checks the result against the
design tenets.

## Summary

* **Recommended v1:** one extension, `CONTROL_FLOW`, requiring `EXPR`. It contains typed step outputs
  (`outputVariables`), a new `Steps.<Name>.Output.<Var>` and `Steps.<Name>.TaskParam.<Param>` symbol class, dynamic
  parameter-space ranges, reductions as ordered lists plus the RFC 0006 functions, and `enableIf` on steps and
  environments (with static or dynamic predicates).
* **Separate extension:** task-task dependencies (#82 `dependsOnSubspace`, self-dependencies, `anyOf`) go in their own
  extension, provisionally `TASK_DEPENDENCIES`, which does not depend on CONTROL_FLOW. Most of its use cases (U8–U14)
  are fully static, and its design surface is large. When a template enables both, CONTROL_FLOW's one rule ("a
  materialization expression may reference upstream outputs") makes computed DAGs (U2) work without further syntax.
* **The model:** the step graph stays fixed. Only task counts, task-parameter values, enablement, and per-task edges
  may depend on upstream outputs. One new evaluation stage, *step input binding*, sits between job creation and task
  execution, and one new step state, `PENDING_INPUTS`, covers a step waiting for it.
* **Emission channel, reversing #111:** tasks write outputs to a session-supplied file (`Task.OutputFile`, also
  exported as `OPENJD_TASK_OUTPUT_FILE`), not with an `openjd_output_var:` stdout line. Stdout carries text from
  third-party tools, so a stdout channel would let that text forge the values that shape the rest of the job.
  GitHub Actions deprecated `::set-output` for this reason. The line grammar and type coercion that 03 designed for
  stdout carry over unchanged.
* **Deferred:** outputs from chunked steps, PATH-typed outputs, producer-declared defaults, partial reductions, cascade
  recompute on re-run, `enableIf` on individual host requirements, and new EXPR functions such as `argmin`/`argmax`.
  Each can be added later without breaking v1 templates.

## 1. Tensions between the investigation notes, and their resolution

| # | Topic | 01 says | 02 says | 03 says | Decision |
|---|---|---|---|---|---|
| T1 | Emission channel | #111 proposes `openjd_output_var:` on stdout | Use a session-provided file; stdout outputs are forgeable (L3) | Designs the stdout directive in detail | **Session-supplied file.** Keep 03's grammar and coercion. |
| T2 | Namespace | `Step.<Name>` collides with `Step.Name`; names may not be identifiers | `tasks.A.outputs.p` style is universal | Uses `Step.<Name>.<Var>` | **New root `Steps`**, with `Output` and `TaskParam` sub-namespaces. |
| T3 | Dependents of a skipped step | Open | Tekton rule: ordering dependents run, data dependents skip (L7) | Skip satisfies ordering; outputs become `null` with type `T?` | **03's rule.** Data dependents run and must handle `null`, which the type checker enforces. |
| T4 | Empty computed range | Open | Treat as skipped (L6) | Legal, step is a no-op `SUCCEEDED` | **03's rule.** Zero tasks, `SUCCEEDED`, outputs are `[]`. |
| T5 | Who may reference outputs | Direct or transitive? | GHA and Argo require a direct edge | Transitive dependents | **Direct dependents only.** |
| T6 | Output shape | #111 treats a single-task step as scalar | Reductions are ordered lists (L8) | Always `list[T]` | **Static rule:** scalar if the producer has no `parameterSpace`, otherwise `list[T]`. |
| T7 | Size limits | Need a bound | Fail the producer (L5) | Inherit the 64 KiB stdout line cap | **≥ 1 MiB per task, declared constraints checked at the producer.** |
| T8 | Task-task deps placement | — | Proven pattern (L9) | Static case could stand alone (Q7) | **Separate extension**, composed through the materialization rule. |

### T1. Session-supplied output file instead of `openjd_output_var:` on stdout

> Superseded by [§8](#8-addendum-emission-channel-collect-modes-and-metrics).

03 shows that a stdout directive slots into the `openjd-sessions` pipeline next to `openjd_env`. 02 counters with
the GitHub Actions precedent. The decision rests on what a forged value can do, which differs from the existing
directives:

* `openjd_progress`, `openjd_status`, and `openjd_fail` affect only the reporting of the action that printed them.
  Forging one is a nuisance.
* `openjd_env` is high-impact, but only an environment's `onEnter` may emit it. Environment authors control that
  output closely, and it usually runs setup tools, not production workloads.
* An output variable comes from `onRun`, where the job's heaviest third-party code runs (DCC applications, renderers,
  plugins, `aws s3 ls` listings of user-named objects). Its value then decides how many tasks run downstream, which
  values they receive, and which steps run at all. An S3 object key may contain a newline, so a discovery script that
  logs the keys it found could inject `openjd_output_var: Keys=[...]` without any malicious intent from the job
  author.

The file channel also avoids three problems 03 had to work around:

* The 64 KiB stdout line cap limited values to 64 KiB. A file has no per-line cap, so the limit can be set where it
  belongs (see T7).
* Under WRAP_ACTIONS, the wrap script's stdout is scanned and must forward the wrapped process verbatim. With a file,
  the wrapped process only needs the environment variable, which wrap scripts already forward.
* Under TASK_CHUNKING, a future per-task report from a chunk can be a structured file format. A single stdout line per
  value cannot express it as cleanly.

What carries over from 03 unchanged: the `<Name>=<JSON>` line grammar, last-write-wins within an attempt, NUL
rejection, coercion through EXPR's implicit non-destructive rules, and "declared but not written fails the task". The
runtime change moves from `action_filter.rs` (stream parsing) to the end of `run_task` (read and validate one file
after `onRun` exits successfully).

The file is exposed as a template variable first, `Task.OutputFile` (type `path`), and also as an environment
variable, `OPENJD_TASK_OUTPUT_FILE`, for nested subprocesses such as adaptors that cannot see template variables.
How-Jobs-Are-Run asks that new environment variables be weighed against the "Tooling parseable" tenet. This one passes
the same test `OPENJD_SESSION_WORKING_DIR` passed: the data flow is declared in `outputVariables` whichever way the
script reaches the file, so tooling loses nothing.

Cost: `echo "openjd_output_var: N=3"` becomes `echo "N=3" >> {{repr_sh(Task.OutputFile)}}`. The GitHub Actions
migration from `::set-output` to `$GITHUB_OUTPUT` suggests users accept this. Because #111 proposed stdout, this is
open question [Q1](#q1-emission-channel).

### T2. `Steps.<Name>.Output.<Var>` instead of `Step.<Name>.<Var>`

`Step` already means "the current step" (`Step.Name`). Reusing it for other steps makes a step named `Name`
unreachable and makes every future `Step.<Property>` a breaking change. RFC 0008 avoided the same problem with a
separate `WrappedStep` root. CONTROL_FLOW uses:

* `Steps.<StepName>.Output.<VarName>`: the step's output value (shape per T6).
* `Steps.<StepName>.TaskParam.<ParamName>`: a `list[T]` of that task parameter's value for every task of the step,
  in parameter-space iteration order. This aligns element-for-element with `Output` lists and is what lets a
  consumer find the configuration that produced a winning score (U4). 01 §3.2 item 4 and 03 §3.3 both flagged
  this as a gap.

The `Output` level costs one word per reference. It buys room for `TaskParam` and later per-step symbols (01's M11
parameter-space introspection, a task count, a status) without collisions with user variable names.

A step referenced through `Steps.` must have an `<Identifier>` name (`[A-Za-z_][A-Za-z0-9_]*`), since dotted access
cannot express `Render Frames`. That is a validation error on the referencing expression, not a new constraint on all
step names.

### T3. A skipped step satisfies ordering and yields `null` data

02 suggests Tekton's rule, under which a step that reads a skipped step's outputs is itself skipped. 03 proposes that
it runs and sees `null`. 03's rule wins for three reasons:

1. It is explicit. If producer `P` has an `enableIf`, every `Steps.P.*` symbol has type `T?`, and the consumer handles
   `null` in plain sight with `Steps.P.Output.X or 0`, a conditional, `fail(...)`, or an `enableIf` of its own.
2. Tekton's rule makes enablement flow along an implicit edge (data references) that a reader must find by scanning
   every expression, and its pitfall ("why did my publish step vanish?") is the join problem every surveyed system
   has (02, "Skip semantics are a recurring source of bugs").
3. A consumer that wants Tekton's behavior can write it in one line:
   `enableIf: "{{Steps.P.Output.X != null}}"`.

03 §4.3 expected the type checker to force consumers to handle `null`. It does not: EXPR's coercion of a union
placeholder is existential (RFC 0005 §Implicit Type Coercion), so `unresolved[int?]` is accepted wherever an `int` is,
and the check is deferred to the concrete value. Testing confirmed this: `string(video_bytes + 1)` with
`video_bytes : int?` passes `openjd check` and fails only when evaluated with `null`. In CONTROL_FLOW that evaluation
happens at the consumer's binding for TEMPLATE-scope fields, before any of its tasks run, or at task start for script
fields. Both are loud failures that name the expression.

One case stays silent: a format string with text around the expression turns `null` into `""` (EXPR §1.3.2), so
`"--size {{Steps.P.Output.X}}"` becomes `--size `. This is the GitHub Actions empty-string pitfall in a narrower
form. Mitigation: the model library emits a validation warning for every `Steps.P.*` reference to a conditional
producer inside a format string with surrounding text ([Q11](#q11-enableif-semantics-and-placement)).

Only the skipped step's own `Steps.P.*` symbols become optional. A step that runs after a skipped step behaves
normally, and its own outputs keep type `T`. 03 §4.3 said "transitively", which is inconsistent with "skip satisfies
ordering", so this note drops it.

Failure is different: a `FAILED` producer blocks its dependents, as today.

### T4. An empty computed range is a successful zero-task step

02 treats it as skipped; 03 as an empty success. Success wins because of the type system. If a zero-task step were
skipped, every step whose parameter space depends on upstream data could be skipped, so all its `Steps.*` symbols
would have to be typed `T?`, which pushes `null` handling into every reduction. As an empty success, the step's
`Output` lists are `[]` with type `list[T]`, `sum([])` is `0`, and an author who considers emptiness an error writes
`fail(...)` in the range expression or sets `minLength: 1` on the producer's output.

This changes one EXPR rule: a task-parameter range that evaluates to an empty list is accepted under CONTROL_FLOW.
Range strings stay non-empty (`range_expr` has no empty value), so `"1-{{N}}"` with `N = 0` is an error, while
`"{{range(1, N + 1)}}"` gives zero tasks. The RFC must say this plainly, because the two look alike.

### T5. Only direct dependents may reference a step's outputs

A step may reference `Steps.P.*` only if it lists `P` in its own `dependencies`. Transitive access saves a line but
hides data flow in long chains. The extra `dependsOn` never changes scheduling, since a transitive dependency already
orders the steps. A reference does not create an edge; a missing edge is a validation error that names the fix. This
follows GHA (`needs`) and Argo, and keeps OpenJD's explicit `dependencies` as the only source of edges.

The edge must be a whole-step dependency. With `TASK_DEPENDENCIES`, a `dependsOnSubspace` edge to `P` lets tasks start
before all of `P` has finished, when `Steps.P.*` is not yet concrete, so it does not count. Sample 4.4 lists
`AnalyzeSceneGraph` as a plain dependency for this reason.

### T6. Output shape is determined by the template, not by the task count

`Steps.P.Output.X` has type `T` if `P` has no `parameterSpace`, and `list[T]` in iteration order otherwise, even if
the space happens to have one task or zero. The type is therefore known at template validation, which progressive
evaluation requires. #111 Example 1 (scalar `FrameCount` from a single-task step) and 03's lossless lists are both
satisfied.

EXPR allows two levels of list nesting, so a multi-task producer may declare scalar or `LIST[<scalar>]` outputs, and
`LIST[LIST[INT]]` only on a producer without a `parameterSpace`.

### T7. Limits are enforced at the producer

* An implementation must accept at least 1 MiB of output file per task (GHA's per-job figure; Step Functions allows
  256 KiB per state) and may accept more. Exceeding the limit fails the producing task.
* `outputVariables` accept the same constraint vocabulary as the job parameter of the same type (`minValue`,
  `maxValue`, `allowedValues`, `minLength`, `maxLength`, `item`). They are checked when the output file is read, so a
  discovery step that finds 1,000,000 keys against `maxLength: 5000` fails itself, next to the cause (L5, Airflow
  `max_map_length`).
* A scheduler may impose a maximum task count per step. A computed space over that cap fails the consuming step at
  binding, before any of its tasks run.
* Reductions are evaluated under EXPR's existing memory and operation limits.

### T8. Task-task dependencies are a separate extension

See [§5.2](#52-what-is-a-separate-extension).

## 2. The model: one new stage, one new symbol class

CONTROL_FLOW follows the "fixed graph, late-bound width" strategy shared by Argo, Airflow, GHA, Tekton, Step Functions,
and CWL (02). The set of steps, their names, their types, and their step-to-step dependencies are in the template and
are validated before submission. What may depend on data from a running task:

* how many tasks a step has, and their task-parameter values (fan-out),
* whether a step or environment runs (`enableIf`),
* the values a step's scripts receive,
* which tasks of a step wait for which other tasks (only with `TASK_DEPENDENCIES`).

Fully dynamic structure, where tasks create new steps, stays outside templates. #22 pointed it at a separate
mechanism (standalone StepTemplates submitted into a running job), and CONTROL_FLOW does not replace that.

### 2.1 Evaluation stages

RFC 0005 defines three stages. CONTROL_FLOW adds a fourth, which runs once per step:

| Stage | When | New in CONTROL_FLOW |
|---|---|---|
| Template validation | Before submission | `Steps.P.Output.X` is `unresolved[T]` (or `unresolved[T?]`), using the declared type. Every check in [§2.3](#23-static-validation-added-by-control_flow). |
| Job creation | Submission | Static `enableIf` is decided. `Steps.P.TaskParam.*` is concrete for producers whose own space is static. Steps whose materialization expressions reference outputs enter `PENDING_INPUTS`. |
| **Step input binding** | When every producer referenced by the step's materialization expressions is terminal (`SUCCEEDED` or `SKIPPED`) | The step's materialization expressions are evaluated in a fixed order and validated as a unit. The step then becomes `READY`, `SKIPPED`, or `FAILED`. |
| Task execution | On the worker | `Steps.*` symbols are concrete in host-scope fields. `Task.OutputFile` is defined. |

A step's **materialization expressions** are its TEMPLATE-scope fields:

1. `enableIf`,
2. step `let`,
3. `hostRequirements`,
4. `parameterSpace` ranges (not `combination`, which is not a format string and stays static),
5. with `TASK_DEPENDENCIES`, dependency subspace ranges,
6. TEMPLATE-scope fields of its `stepEnvironments`, including their `enableIf`.

They are evaluated in that order. `enableIf` comes first and cannot reference step `let` bindings, so the `let`
bindings of a disabled step are never evaluated and may assume the step runs (for example `min(...)` over a list that
is only non-empty when it is enabled). Binding is atomic: an evaluation error, a computed task DAG with a cycle or an
out-of-range index, or a task count over the scheduler cap fails the step before any of its tasks is dispatched (03
§5.2).

A step with no `Steps.*` reference in its materialization expressions never enters `PENDING_INPUTS` and behaves
exactly as today. A step may use `Steps.*` only in host-scope fields (its scripts) and still bind at job creation; its
ordinary dependencies already guarantee the values exist before its tasks run.

Schedulers may create a `PENDING_INPUTS` step's tasks lazily or create the step first and fill it in. The observable
contract is the same: a step's tasks are enumerable if and only if its materialization inputs are concrete (03 §1.3).

### 2.2 Step states

```
             job creation
                  │
      ┌───────────┴─────────────┐
      ▼                         ▼
 PENDING_INPUTS ──binding──► READY ──► RUNNING ──► SUCCEEDED
      │                         │                ╰► FAILED
      ├──binding: enableIf false ──► SKIPPED
      ├──binding: error / cycle / cap ──► FAILED
      └──referenced producer FAILED ──► (blocked; never runs, as today)
```

`SKIPPED` and a zero-task `SUCCEEDED` both satisfy `dependsOn`. `FAILED` does not.

### 2.3 Static validation added by CONTROL_FLOW

All of these are properties of the template text and are checked by the model library before submission:

1. `outputVariables` names are unique `<Identifier>`s within the step; types are from the allowed set (T6); a
   multi-task producer does not declare `LIST[LIST[INT]]`; a producer with a `CHUNK[INT]` task parameter declares no
   outputs (deferred, see [Q8](#q8-outputs-from-chunked-steps)).
2. Every `Steps.P.Output.X` names a declared step with an `<Identifier>` name and a declared output `X`. Every
   `Steps.P.TaskParam.Y` names a task parameter of `P`.
3. The referencing step lists `P` directly in `dependencies`, without a `dependsOnSubspace` (T5). A step does not
   reference its own outputs.
4. `Steps.*` is not used in job-level fields (job `name`, job environments, job parameters) or in
   `<EnvironmentTemplate>`s.
5. Types check: the symbols enter EXPR's progressive evaluator as `unresolved[T]` or, for an `enableIf` producer,
   `unresolved[T?]`. No new type-checking logic is needed (03 §6.3).
6. `Task.OutputFile` is used only in the script of a step that declares `outputVariables`.
7. `enableIf` evaluates to `bool` and does not reference step `let` bindings.
8. A `Steps.P.*` symbol of a conditional producer inside a format string with surrounding text draws a warning
   (T3).
9. The step DAG is acyclic, unchanged.

Runtime-only properties: concrete values and their constraints (checked at the producer), task counts, empty spaces,
computed DAG shape (checked at binding), and whether each declared output was written.

## 3. Control-flow taxonomy

Each component lists its use cases, a schema sketch, the EXPR surface it needs, and the scheduler state it adds.

### 3.1 (a) Data flow between steps: typed output variables

**Use cases.** U1 frame count, U3 discovered keys, U4 sweep metrics, U2 adjacency list. Also on its own, with no
downstream consumer: per-task metrics visible to the scheduler and UI (U4 phase 1, folouiseAWS).

**Schema.**

```yaml
# <StepTemplate>
outputVariables: [ <OutputVariableDefinition>, ... ]   # @optional @extension CONTROL_FLOW

# <OutputVariableDefinition>
name: <Identifier>
type: INT | FLOAT | STRING | BOOL | RANGE_EXPR
    | LIST[INT] | LIST[FLOAT] | LIST[STRING] | LIST[BOOL]
    | LIST[LIST[INT]]          # only on a step without parameterSpace
description: <Description>     # @optional
# @optional constraints, same names and meaning as the job parameter of the same type:
#   minValue, maxValue, allowedValues, minLength, maxLength, item
```

Type names are case-insensitive and map to EXPR types exactly as the job-parameter types do (RFC 0007), so a
`Steps.P.Output.X` behaves in an expression like a `Param.X` of the same type. `PATH` and `LIST[PATH]` are deferred
([Q15](#q15-path-typed-outputs)).

**Emission.** Before each `onRun` of a step that declares outputs, the runtime creates an empty file and exposes its
path as `Task.OutputFile` and `OPENJD_TASK_OUTPUT_FILE`. The action appends lines:

```bnf
<OutputFile> ::= (<OutputLine> | <BlankLine>)*
<OutputLine> ::= <Identifier> "=" <JSON> <EOL>     ; <JSON> is one JSON value on one line
<EOL>        ::= "\n" | "\r\n"
```

After `onRun` exits successfully the runtime reads the file. The task fails if any line is malformed, names an
undeclared variable, contains NUL, has a value that does not coerce to the declared type, or violates a declared
constraint; if a declared variable has no line; or if the file exceeds the size limit. The last line for a name wins.
Values are committed only when the task succeeds. A failed attempt's file is discarded, so a retry starts clean.

Only step `onRun` actions can write outputs; environment actions cannot ([Q7](#q7-emission-rules)). Outputs are
stored by the scheduler and may be displayed, so they must not carry secrets, and they are a control channel, not a
data channel. Bulk data keeps flowing through files and `PATH` job parameters.

**EXPR.** New symbols `Steps.<Name>.Output.<Var>` and `Steps.<Name>.TaskParam.<Param>` (T2, T6) and `Task.OutputFile`.
No new types or functions.

**Scheduler state.** Per committed task attempt, a map from output name to typed value, plus its size. The values
survive the session and are retained for the life of the job.

### 3.2 (b) Dynamic fan-out: parameter-space ranges from upstream data

**Use cases.** U1 (one task per frame), U3 (one task per S3 object), U4 phase 2 (one task per dataset video).

**Schema.** No new fields. A task-parameter `range` may reference `Steps.*`:

```yaml
range: "1-{{Steps.VideoToFrames.Output.FrameCount}}"     # INT, RangeString
range: "{{Steps.Discover.Output.Keys}}"                  # STRING, ListExpressionString
range: "{{range(len(Steps.Analyze.Output.Children))}}"   # INT, zero tasks if the list is empty
```

`combination` stays static, so the shape of the space (which parameters, how they combine) is fixed even when its
size is not. `CHUNK[INT]` consumers are fine: chunking happens after binding, on the materialized range.

**EXPR.** Existing range-field extensions (RFC 0005 §Task Parameter Range Field Extensions), plus the empty-list rule
from T4.

**Scheduler state.** The `PENDING_INPUTS` state, the binding trigger (all referenced producers terminal), the
materialized task set, and the per-step task cap. A UI or `openjd summary` reports the task count of a pending step as
"determined at run time from `Steps.VideoToFrames.Output.FrameCount`", which answers #22's "when will it be done?"
concern as well as possible.

### 3.3 (c) Conditional execution: `enableIf`

**Use cases.** U5 optional step from a `BOOL` parameter (issue #80), U6 optional environment, and the dynamic case
named in #111: run a step only if an upstream metric crosses a threshold.

**Schema.**

```yaml
# <StepTemplate> and <Environment> (job and step environments)
enableIf: <FormatString>     # @optional @extension CONTROL_FLOW, evaluates to bool
```

* On a step, `enableIf` may reference `Param.*`, `RawParam.*`, `Job.Name`, `Step.Name`, and `Steps.*` of direct
  dependencies. With only job-parameter references it is decided at job creation; with `Steps.*` references at
  binding.
* On a step environment, the same symbols, evaluated at the step's binding. A disabled environment is not entered.
* On a job environment, only `Param.*`, `RawParam.*`, and `Job.Name`, decided at job creation, because job
  environments wrap tasks of every step.
* A disabled step is `SKIPPED`: it runs no tasks, satisfies `dependsOn`, and every `Steps.<It>.*` symbol is `null`
  (type `T?`) (T3). A statically disabled step is still part of the job and shown as skipped, so the job has the same
  steps regardless of parameter values.

`enableIf` on individual host requirements (#81, U7) is deferred: amounts can already be computed with EXPR (a `min`
of `0` imposes nothing) and attribute `anyOf` lists can be computed, which covers the CPU-vs-GPU example.

**EXPR.** Conditional expressions, `and`/`or`/`not`, comparisons, and `null` handling, all present. The new part is the
`T?` typing of `Steps.*` symbols of conditional producers.

**Scheduler state.** The `SKIPPED` step status and the evaluated `enableIf` result, recorded for display.

### 3.4 (d) Dynamic dependency structure: task-task edges from computed data

**Use cases.** U2 scene-graph DAG with a computed adjacency list. The static cases U8–U14 belong to
`TASK_DEPENDENCIES` alone.

**Schema.** Defined by `TASK_DEPENDENCIES` (sketched from #82 and #111; 01 M9–M13 list its open questions):

```yaml
dependencies:
  - dependsOn: BakeNode                    # self-dependency, legal only with a subspace
    dependsOnSubspace:
      - name: Node                         # task parameter of the depended-upon step
        range: "{{Steps.Analyze.Output.Children[Task.Param.Node]}}"   # evaluated per dependent task
```

CONTROL_FLOW's only contribution: a subspace `range` is a materialization expression, so it may reference `Steps.*`,
and the resulting task DAG is built and checked (acyclic, indices in range) at binding, before any task of the step
runs. An empty list selects no tasks, which is what a root node needs.

**EXPR.** List subscripting by `Task.Param.*`. `TASK_DEPENDENCIES` must extend the scope of `Task.Param.*` to subspace
ranges, where it means the dependent task's parameters.

**Scheduler state.** Per-task readiness instead of per-step readiness, the computed edges, and cycle detection on the
materialized graph (O(N+E), the same DFS as `StepDependencyGraph`). This is the largest scheduler change in the whole
design space, which is one reason it is a separate extension.

### 3.5 (e) Fan-in and reduction

**Use cases.** U4 pick the winning configuration, U3 manifest of all processed objects, any "sum the bytes" or "max of
the scores" summary.

**Schema.** None. A multi-task producer's `Steps.P.Output.X` is a `list[T]` in parameter-space iteration order
(the order the `combination` expression defines, independent of completion order), and `Steps.P.TaskParam.Y` is
aligned with it. A consumer reduces with existing functions:

```yaml
let:
  - total = sum(Steps.Transcode.Output.OutputBytes)
  - best = min(Steps.Sweep.Output.mpjpe_mm)
  - best_i = [i for i in range(len(Steps.Sweep.Output.mpjpe_mm)) if Steps.Sweep.Output.mpjpe_mm[i] == best][0]
  - best_window = Steps.Sweep.TaskParam.Window[best_i]
```

Multi-dimensional spaces flatten in iteration order (CWL's `flat_crossproduct`). A list is concrete only when every
task of the producer is terminal, so there are no partial reductions in v1.

No declared reductions (`reduce: MAX`): the list keeps everything, and EXPR already has `min`, `max`, `sum`, `len`,
`sorted`, `unique`, `any`, `all`, `flatten`, slicing, and comprehensions. The pick-the-winner index search above is
awkward. An `argmin`/`argmax` pair (or a list `index`) would fix it, but that is a change to RFC 0006 and doesn't block
CONTROL_FLOW ([Q5](#q5-correlating-a-reduced-value-with-its-task)).

**Scheduler state.** Assembling the ordered list from per-task values when the producer finishes. Implementations may
build it lazily on first use, which avoids Airflow's eager-materialization cost (L12).

## 4. End-to-end sample templates

These are written against EXPR as specified in `wiki/2026-02-Expression-Language.md` and the CONTROL_FLOW schema
sketched above. Application command lines (`ffmpeg`, `stylize`, `bake-node`, ...) are illustrative.

Testing: no implementation can check the CONTROL_FLOW fields yet. For each sample, a proxy template replaced every
`Steps.P.Output.X` and `Steps.P.TaskParam.Y` with a job parameter of the type T6 assigns, and `Task.OutputFile` with a
path under `Session.WorkingDirectory`. It dropped `outputVariables`, `enableIf`, and `dependsOnSubspace`, moving their
expressions into `let` bindings. Proxies generated mechanically from the YAML in this document pass `openjd check` in
the openjd-rs CLI (commit `be266ed`). Hand-written copies pass `openjd run` with mock values, including empty upstream
lists for 4.2 and 4.5, the `fail(...)` paths in 4.3 and 4.4, and `video_bytes = null` in 4.5. The proxies caught three
bugs, all fixed in the samples below: a PATH `Param` in a job `name` (PATH values are host-only, so the names use
`path(RawParam.X)`), a task parameter in a step-level `let`, and a YAML colon inside a `let` string. The proxies cannot
test the CONTROL_FLOW semantics themselves (binding, skipping, the output file).

| Sample | Use case | (a) data | (b) fan-out | (c) enableIf | (d) task DAG | (e) fan-in |
|---|---|---|---|---|---|---|
| [4.1](#41-video-frame-count) | U1 | ✔ | ✔ | | | |
| [4.2](#42-s3-prefix-fan-out-with-a-manifest) | U3 | ✔ | ✔ | | | ✔ |
| [4.3](#43-parameter-sweep-then-run-the-winner) | U4 + U3 | ✔ | ✔ | | | ✔ |
| [4.4](#44-scene-graph-dag) | U2 | ✔ | ✔ | | ✔ | |
| [4.5](#45-render-quality-check-and-optional-steps) | U5 + dynamic | ✔ | ✔ (may be empty) | ✔ static and dynamic | | |

### 4.1 Video frame count

#111 Example 1. The submitter doesn't know the frame count; the first step finds it.

```yaml
specificationVersion: jobtemplate-2023-09
extensions: [EXPR, CONTROL_FLOW, FEATURE_BUNDLE_1]
name: "Stylize {{path(RawParam.InputVideo).name}}"
parameterDefinitions:
  - name: InputVideo
    type: PATH
    objectType: FILE
    dataFlow: IN
  - name: WorkDir
    type: PATH
    objectType: DIRECTORY
    dataFlow: OUT
  - name: OutputVideo
    type: PATH
    objectType: FILE
    dataFlow: OUT
steps:
  - name: VideoToFrames
    outputVariables:
      - name: FrameCount
        type: INT
        minValue: 1
        maxValue: 200000
      - name: FrameRate
        type: STRING
    bash:
      script: |
        set -euo pipefail
        frames={{repr_sh(Param.WorkDir / 'frames')}}
        mkdir -p "$frames"
        ffmpeg -nostdin -i {{repr_sh(Param.InputVideo)}} "$frames/in_%06d.png"
        count=$(find "$frames" -name 'in_*.png' | wc -l)
        rate=$(ffprobe -v error -select_streams v:0 -show_entries stream=r_frame_rate \
                 -of default=noprint_wrappers=1:nokey=1 {{repr_sh(Param.InputVideo)}})
        echo "FrameCount=$count" >> {{repr_sh(Task.OutputFile)}}
        echo "FrameRate=\"$rate\"" >> {{repr_sh(Task.OutputFile)}}

  - name: StylizeFrame
    dependencies:
      - dependsOn: VideoToFrames
    parameterSpace:
      taskParameterDefinitions:
        - name: Frame
          type: INT
          range: "1-{{Steps.VideoToFrames.Output.FrameCount}}"
    bash:
      script: |
        stylize --in {{repr_sh((Param.WorkDir / 'frames' / 'in_%06d.png').with_number(Task.Param.Frame))}} \
                --out {{repr_sh((Param.WorkDir / 'frames' / 'out_%06d.png').with_number(Task.Param.Frame))}}

  - name: FramesToVideo
    dependencies:
      - dependsOn: VideoToFrames
      - dependsOn: StylizeFrame
    bash:
      script: |
        ffmpeg -nostdin -framerate {{repr_sh(Steps.VideoToFrames.Output.FrameRate)}} \
               -i {{repr_sh(Param.WorkDir / 'frames' / 'out_%06d.png')}} {{repr_sh(Param.OutputVideo)}}
```

Notes:

* `StylizeFrame` is in `PENDING_INPUTS` until `VideoToFrames` succeeds. `minValue: 1` makes an empty video fail
  `VideoToFrames` itself rather than produce the invalid range string `"1-0"` downstream.
* `FramesToVideo` uses `FrameRate` only in its script (host scope), so it binds at job creation. It still has to list
  `VideoToFrames` directly (T5) even though `StylizeFrame` already orders it after.

### 4.2 S3 prefix fan-out with a manifest

crowecawcaw's case. Today this needs a pre-submission script, which job bundle submitters can't run.

```yaml
specificationVersion: jobtemplate-2023-09
extensions: [EXPR, CONTROL_FLOW, FEATURE_BUNDLE_1]
name: "Transcode s3://{{Param.Bucket}}/{{Param.Prefix}}"
parameterDefinitions:
  - name: Bucket
    type: STRING
  - name: Prefix
    type: STRING
  - name: OutputPrefix
    type: STRING
steps:
  - name: Discover
    outputVariables:
      - name: Keys
        type: LIST[STRING]
        maxLength: 5000        # Discover fails if the prefix holds more .mov objects
    python:
      script: |
        import boto3, json
        keys = []
        pages = boto3.client("s3").get_paginator("list_objects_v2").paginate(
            Bucket={{repr_py(Param.Bucket)}}, Prefix={{repr_py(Param.Prefix)}})
        for page in pages:
            keys += [o["Key"] for o in page.get("Contents", []) if o["Key"].endswith(".mov")]
        with open({{repr_py(Task.OutputFile)}}, "a") as f:
            f.write("Keys=" + json.dumps(keys) + "\n")

  - name: Transcode
    dependencies:
      - dependsOn: Discover
    parameterSpace:
      taskParameterDefinitions:
        - name: Key
          type: STRING
          range: "{{Steps.Discover.Output.Keys}}"    # zero tasks if nothing matched
    outputVariables:
      - name: OutputBytes
        type: INT
        minValue: 0
    python:
      let:
        - out_key = Param.OutputPrefix + '/' + path(Task.Param.Key).stem + '.mp4'
      script: |
        import boto3, subprocess
        s3 = boto3.client("s3")
        s3.download_file({{repr_py(Param.Bucket)}}, {{repr_py(Task.Param.Key)}}, "in.mov")
        subprocess.run(["ffmpeg", "-nostdin", "-i", "in.mov", "out.mp4"], check=True)
        s3.upload_file("out.mp4", {{repr_py(Param.Bucket)}}, {{repr_py(out_key)}})
        size = s3.head_object(Bucket={{repr_py(Param.Bucket)}}, Key={{repr_py(out_key)}})["ContentLength"]
        with open({{repr_py(Task.OutputFile)}}, "a") as f:
            f.write(f"OutputBytes={size}\n")

  - name: Manifest
    dependencies:
      - dependsOn: Transcode
    let:
      - keys = Steps.Transcode.TaskParam.Key
      - sizes = Steps.Transcode.Output.OutputBytes
    python:
      script: |
        import boto3, json
        manifest = {"inputs": {{repr_py(keys)}}, "outputBytes": {{repr_py(sizes)}},
                    "totalBytes": {{sum(sizes)}}}
        boto3.client("s3").put_object(Bucket={{repr_py(Param.Bucket)}},
                                      Key={{repr_py(Param.OutputPrefix + '/manifest.json')}},
                                      Body=json.dumps(manifest))
```

Notes:

* `Manifest` gets keys from `Steps.Transcode.TaskParam.Key`, aligned with `OutputBytes`, so it needs no edge to
  `Discover`.
* If the prefix holds no `.mov` objects, `Transcode` is a zero-task success and `Manifest` writes an empty manifest
  with `totalBytes` 0 (T4).
* Python's `json.dumps` writes a list on one line, which the output grammar requires.

### 4.3 Parameter sweep, then run the winner

folouiseAWS's case. 27 configurations × 10 benchmark episodes report two metrics each; the job picks the most accurate
configuration that runs fast enough and then runs it on every video under an S3 prefix.

```yaml
specificationVersion: jobtemplate-2023-09
extensions: [EXPR, CONTROL_FLOW, FEATURE_BUNDLE_1]
name: Hand-tracking sweep and full run
parameterDefinitions:
  - name: BenchmarkDir
    type: PATH
    objectType: DIRECTORY
    dataFlow: IN
  - name: EpisodeCount
    type: INT
    default: 10
    minValue: 1
  - name: MinFps
    type: FLOAT
    default: 20
  - name: DatasetBucket
    type: STRING
  - name: DatasetPrefix
    type: STRING
  - name: OutputDir
    type: PATH
    objectType: DIRECTORY
    dataFlow: OUT
steps:
  - name: Sweep
    parameterSpace:
      taskParameterDefinitions:
        - name: Window
          type: INT
          range: [8, 16, 32]
        - name: Overlap
          type: FLOAT
          range: [0.25, 0.5, 0.75]
        - name: DetConf
          type: FLOAT
          range: [0.3, 0.5, 0.7]
        - name: Episode
          type: INT
          range: "0-{{Param.EpisodeCount - 1}}"
      combination: "Window * Overlap * DetConf * Episode"   # Episode varies fastest
    outputVariables:
      - name: mpjpe_mm
        type: FLOAT
        minValue: 0
      - name: fps
        type: FLOAT
        minValue: 0
    python:
      script: |
        from handtrack import benchmark
        score = benchmark({{repr_py(Param.BenchmarkDir)}}, episode={{Task.Param.Episode}},
                          window={{Task.Param.Window}}, overlap={{Task.Param.Overlap}},
                          det_conf={{Task.Param.DetConf}})
        with open({{repr_py(Task.OutputFile)}}, "a") as f:
            f.write(f"mpjpe_mm={score.mpjpe_mm}\n")
            f.write(f"fps={score.fps}\n")

  - name: Discover
    outputVariables:
      - name: Videos
        type: LIST[STRING]
        minLength: 1
        maxLength: 2000
    python:
      script: |
        import boto3, json
        pages = boto3.client("s3").get_paginator("list_objects_v2").paginate(
            Bucket={{repr_py(Param.DatasetBucket)}}, Prefix={{repr_py(Param.DatasetPrefix)}})
        keys = [o["Key"] for p in pages for o in p.get("Contents", []) if o["Key"].endswith(".mp4")]
        with open({{repr_py(Task.OutputFile)}}, "a") as f:
            f.write("Videos=" + json.dumps(keys) + "\n")

  - name: RunWinner
    dependencies:
      - dependsOn: Sweep
      - dependsOn: Discover
    let:
      - n = Param.EpisodeCount
      - err = Steps.Sweep.Output.mpjpe_mm
      - rate = Steps.Sweep.Output.fps
      - configs = range(len(err) // n)
      - mean_err = [sum(err[c * n:(c + 1) * n]) / n for c in configs]
      - mean_fps = [sum(rate[c * n:(c + 1) * n]) / n for c in configs]
      - fast = [c for c in configs if mean_fps[c] >= Param.MinFps]
      - ok = fast if len(fast) > 0 else fail("No configuration reached MinFps")
      - best_err = min([mean_err[c] for c in ok])
      - first_task = [c for c in ok if mean_err[c] == best_err][0] * n
      - window = Steps.Sweep.TaskParam.Window[first_task]
      - overlap = Steps.Sweep.TaskParam.Overlap[first_task]
      - det_conf = Steps.Sweep.TaskParam.DetConf[first_task]
    parameterSpace:
      taskParameterDefinitions:
        - name: Video
          type: STRING
          range: "{{Steps.Discover.Output.Videos}}"
    python:
      script: |
        from handtrack import track_s3_video
        track_s3_video({{repr_py(Param.DatasetBucket)}}, {{repr_py(Task.Param.Video)}},
                       window={{window}}, overlap={{overlap}}, det_conf={{det_conf}},
                       out_dir={{repr_py(Param.OutputDir)}})
```

Notes:

* `Sweep` and `Discover` have no dependency on each other and run in parallel.
* The per-configuration means rely on the documented iteration order: with `Episode` last in `combination`, the tasks
  of configuration `c` occupy indices `c*n` to `c*n + n - 1`. Without EXPR records or `zip`, this index arithmetic is
  the price of correlating results with configurations. It is the clearest readability cost in the design and the
  motivation for [Q5](#q5-correlating-a-reduced-value-with-its-task).
* If no configuration is fast enough, `RunWinner` fails at binding with the `fail(...)` message and runs no tasks.
* U4 phase 1 is just the `Sweep` step: a scheduler can show all 270 tasks' `mpjpe_mm` and `fps` with no consumer.

### 4.4 Scene-graph DAG

#111 Example 2: the analysis step computes both the number of tasks and the edges between them. Needs
`TASK_DEPENDENCIES`.

```yaml
specificationVersion: jobtemplate-2023-09
extensions: [EXPR, CONTROL_FLOW, TASK_DEPENDENCIES, FEATURE_BUNDLE_1]
name: "Bake lighting for {{path(RawParam.Scene).stem}}"
parameterDefinitions:
  - name: Scene
    type: PATH
    objectType: FILE
    dataFlow: IN
  - name: BakeDir
    type: PATH
    objectType: DIRECTORY
    dataFlow: OUT
steps:
  - name: AnalyzeSceneGraph
    outputVariables:
      - name: Children
        type: LIST[LIST[INT]]
        maxLength: 10000
        description: Children[i] lists the nodes whose bakes node i consumes.
      - name: NodeNames
        type: LIST[STRING]
        maxLength: 10000
    python:
      script: |
        import json
        from scenetools import load_graph
        g = load_graph({{repr_py(Param.Scene)}})
        with open({{repr_py(Task.OutputFile)}}, "a") as f:
            f.write("Children=" + json.dumps([n.child_indices for n in g.nodes]) + "\n")
            f.write("NodeNames=" + json.dumps([n.name for n in g.nodes]) + "\n")

  - name: BakeNode
    dependencies:
      - dependsOn: AnalyzeSceneGraph
      - dependsOn: BakeNode
        dependsOnSubspace:
          - name: Node
            range: "{{Steps.AnalyzeSceneGraph.Output.Children[Task.Param.Node]}}"
    let:
      - children = Steps.AnalyzeSceneGraph.Output.Children
      - names = Steps.AnalyzeSceneGraph.Output.NodeNames
      - node_count = len(children) if len(children) == len(names) else fail("Children and NodeNames differ in length")
    parameterSpace:
      taskParameterDefinitions:
        - name: Node
          type: INT
          range: "{{range(node_count)}}"
    bash:
      script: |
        bake-node --scene {{repr_sh(Param.Scene)}} --node {{repr_sh(names[Task.Param.Node])}} \
          {{repr_sh(flatten([['--input', string(Param.BakeDir / (names[c] + '.bake'))] for c in children[Task.Param.Node]]))}} \
          --out {{repr_sh(Param.BakeDir / (names[Task.Param.Node] + '.bake'))}}

  - name: AssembleLightmaps
    dependencies:
      - dependsOn: BakeNode
    bash:
      script: |
        assemble-lightmaps {{repr_sh(Param.BakeDir)}}
```

Notes:

* The subspace range is evaluated per `BakeNode` task with `Task.Param.Node` bound to that task. Leaf nodes have
  `[]` children and wait on nothing but `AnalyzeSceneGraph`.
* At binding the scheduler evaluates `node_count`, materializes `node_count` tasks, evaluates every task's subspace,
  and checks the resulting graph for cycles and out-of-range indices. A cyclic scene graph fails `BakeNode` before any
  bake runs, with the cycle in the message.
* The `node_count` binding cross-checks the two outputs, so a mismatch fails at binding instead of as an index error in
  some task's script.

### 4.5 Render, quality check, and optional steps

Exercises static `enableIf` (U5), dynamic `enableIf`, an output of a conditional producer, and a fan-out that is
often empty.

```yaml
specificationVersion: jobtemplate-2023-09
extensions: [EXPR, CONTROL_FLOW, FEATURE_BUNDLE_1]
name: "Render {{path(RawParam.SceneFile).stem}}"
parameterDefinitions:
  - name: SceneFile
    type: PATH
    objectType: FILE
    dataFlow: IN
  - name: Frames
    type: RANGE_EXPR
    default: "1-240"
  - name: OutputPattern
    type: PATH
    objectType: FILE
    dataFlow: OUT
    default: "out/beauty_####.exr"
  - name: NoiseThreshold
    type: FLOAT
    default: 0.02
  - name: EncodeVideo
    type: BOOL
    default: true
  - name: OutputVideo
    type: PATH
    objectType: FILE
    dataFlow: OUT
    default: "out/beauty.mp4"
steps:
  - name: Render
    parameterSpace:
      taskParameterDefinitions:
        - name: Frame
          type: INT
          range: "{{Param.Frames}}"
    bash:
      script: |
        render --scene {{repr_sh(Param.SceneFile)}} --frame {{Task.Param.Frame}} \
               --out {{repr_sh(Param.OutputPattern.with_number(Task.Param.Frame))}}

  - name: QualityCheck
    dependencies:
      - dependsOn: Render
    outputVariables:
      - name: BadFrames
        type: LIST[INT]
      - name: MeanNoise
        type: FLOAT
        minValue: 0
    python:
      script: |
        import json
        from qc import check_frames
        report = check_frames({{repr_py(Param.OutputPattern)}}, {{repr_py(list(Param.Frames))}})
        with open({{repr_py(Task.OutputFile)}}, "a") as f:
            f.write("BadFrames=" + json.dumps(report.bad_frames) + "\n")
            f.write(f"MeanNoise={report.mean_noise}\n")

  - name: Rerender
    dependencies:
      - dependsOn: QualityCheck
    parameterSpace:
      taskParameterDefinitions:
        - name: Frame
          type: INT
          range: "{{Steps.QualityCheck.Output.BadFrames}}"   # usually empty: zero tasks
    bash:
      script: |
        render --scene {{repr_sh(Param.SceneFile)}} --frame {{Task.Param.Frame}} --safe-mode \
               --out {{repr_sh(Param.OutputPattern.with_number(Task.Param.Frame))}}

  - name: Denoise
    dependencies:
      - dependsOn: QualityCheck
      - dependsOn: Rerender
    enableIf: "{{Steps.QualityCheck.Output.MeanNoise > Param.NoiseThreshold}}"   # decided at binding
    parameterSpace:
      taskParameterDefinitions:
        - name: Frame
          type: INT
          range: "{{Param.Frames}}"
    bash:
      script: |
        denoise --in-place {{repr_sh(Param.OutputPattern.with_number(Task.Param.Frame))}}

  - name: EncodeVideo
    dependencies:
      - dependsOn: Rerender
      - dependsOn: Denoise          # satisfied whether Denoise ran or was skipped
    enableIf: "{{Param.EncodeVideo}}"   # decided at job creation
    outputVariables:
      - name: VideoBytes
        type: INT
    bash:
      script: |
        set -euo pipefail
        ffmpeg -nostdin -start_number {{min(Param.Frames)}} -i {{repr_sh(Param.OutputPattern)}} \
               {{repr_sh(Param.OutputVideo)}}
        echo "VideoBytes=$(stat -c %s {{repr_sh(Param.OutputVideo)}})" >> {{repr_sh(Task.OutputFile)}}

  - name: Publish
    dependencies:
      - dependsOn: EncodeVideo
    let:
      - video_bytes = Steps.EncodeVideo.Output.VideoBytes    # int?, because EncodeVideo has enableIf
      - summary = 'video of ' + string(video_bytes) + ' bytes' if video_bytes != null else 'frames only'
    bash:
      script: |
        publish --pattern {{repr_sh(Param.OutputPattern)}} --note {{repr_sh(summary)}}
```

Notes:

* With `EncodeVideo: false`, `EncodeVideo` is `SKIPPED` at job creation and `Publish` still runs with
  `video_bytes = null`, so its note reads `frames only`. Without the null check, `string(video_bytes)` would give the
  text `null`, and arithmetic such as `video_bytes + 1` would pass validation but fail `Publish` when it binds (T3).
  The `summary` text avoids `': '` because a colon followed by a space would turn the `let` item into a YAML mapping.
* `Denoise` binds when `QualityCheck` succeeds and is skipped if the noise is under the threshold. Its own
  `Rerender` dependency still gates its tasks.
* `Rerender` with `BadFrames = []` is a zero-task success, so the common clean render costs nothing.

## 5. Scoping decision for v1

### 5.1 What ships in CONTROL_FLOW v1

| Component | In v1 | Notes |
|---|---|---|
| (a) `outputVariables`, output file, `Steps.<Name>.Output.*`, `Steps.<Name>.TaskParam.*` | ✔ | Non-chunked producers, non-PATH types |
| (b) Dynamic ranges, `PENDING_INPUTS`, step input binding, empty spaces | ✔ | `combination` stays static |
| (c) `enableIf` on steps and environments, static and dynamic, `SKIPPED`, `T?` outputs | ✔ | Not on individual host requirements |
| (d) Computed task-task edges | Composition rule only | Requires `TASK_DEPENDENCIES` |
| (e) Ordered lists plus RFC 0006 functions | ✔ | No declared reductions, no partial reductions |

**One extension rather than outputs-then-data-flow.** folouiseAWS suggested shipping task outputs first and data flow
second. Shipping them as two extensions would freeze an outputs-only `CONTROL_FLOW` and force a second name
(`CONTROL_FLOW_2` or a sibling) for the half most users want, with little in exchange, since outputs-only templates
are a subset of full ones. The phasing is still useful inside implementations: a scheduler can build output collection
and display first and advertise `CONTROL_FLOW` once binding works.

**`enableIf` belongs with data flow.** Static `enableIf` alone is cheap and would close issue #80 sooner. It stays in
CONTROL_FLOW because its hard questions are data-flow questions: what a skipped step's outputs evaluate to, whether
dependents run, how `T?` propagates. Every surveyed system that designed skip semantics separately from data flow has
join pitfalls (02). If CONTROL_FLOW's schedule slips, static-only `enableIf` could be split out later without changing
its syntax ([Q11](#q11-enableif-semantics-and-placement)).

**CONTROL_FLOW requires EXPR.** Every new expression is an EXPR expression, and output types are EXPR types. A
validator rejects a template that lists `CONTROL_FLOW` without `EXPR`. RFC 0002 doesn't define extension
dependencies, so the RFC states it, as a rule of the CONTROL_FLOW extension.

### 5.2 What is a separate extension

**Task-task dependencies, provisionally `TASK_DEPENDENCIES`, independent of CONTROL_FLOW.** Reasons:

* Seven of the eight task-task use cases (U8–U14) are fully static and need nothing from CONTROL_FLOW. Bundling would
  make render pipelines wait for data flow, or data-flow users wait for per-task scheduling.
* For a scheduler, the costs are independent and both large: CONTROL_FLOW needs value storage and a binding stage;
  task-task dependencies need per-task readiness instead of per-step readiness.
* The open questions don't overlap: subspace field name, omitted-parameter defaults, 1:1 sugar, `anyOf`/`allOf`,
  `dependsOn` lists, the `itself` keyword, chunking of both ends, O(N×M) edge evaluation (01 M9–M13).
* Composition needs no new syntax. With both enabled, a subspace range is a materialization expression (§2.1), so the
  computed DAG of U2 works and sample 4.4 is valid.

03 suggested the static case stand alone and the computed case live in CONTROL_FLOW "layered on it". That is this
design: the computed case is just CONTROL_FLOW's materialization rule applied to TASK_DEPENDENCIES' field.

Neither extension depends on the other. The RFC that lands second states the composition rule; if CONTROL_FLOW lands
first it states it conditionally ("when `TASK_DEPENDENCIES` is also enabled").

### 5.3 Explicitly deferred

| Deferred | Why it can wait | Compatible later? |
|---|---|---|
| Outputs from steps with a `CHUNK[INT]` parameter | Needs a per-task record format inside a chunk; producers in the use cases are unchunked. Shares a design with #169 checkpointing. | Yes: lifts a validation error |
| `PATH` / `LIST[PATH]` outputs | Path mapping of a value produced on another worker is unspecified; `STRING` + join on a PATH parameter covers the cases | Yes: new types |
| Producer-declared `default` for missing or skipped outputs | `or` and conditionals cover it | Yes: new optional field |
| Partial reductions (values before all producer tasks finish) | "Iteration order" is ill-defined mid-flight | Yes: new symbol |
| Cascade recompute on re-run of a succeeded producer | Large blast radius; scheduler policy | Yes: scheduler capability |
| `enableIf` on individual host requirements | Computed amounts and attribute lists cover U7 | Yes: new optional field |
| `argmin`/`argmax`, list `index`, `zip`/`enumerate` | Comprehensions work; belongs to RFC 0006 | Yes: new functions |
| `openjd_output_var:` stdout directive | Security (T1) | Yes, if the thread insists |
| Emission from environment actions | No use case yet | Yes |
| Standalone StepTemplates appended to running jobs (#22) | Different mechanism | Separate RFC |

## 6. Open questions for the discussion thread

These consolidate 01 §5 (14 questions) and 03 §9 (8 questions), plus questions this synthesis raised. Each has a
recommended answer to put on #111 before drafting the RFC.

#### Q1. Emission channel

*Stdout `openjd_output_var:` (#111) or a session-supplied file?* **Recommend the file** (`Task.OutputFile`,
`OPENJD_TASK_OUTPUT_FILE`), for the reasons in T1. Ask specifically whether anyone has a runtime where writing a file
is harder than printing a line. Sources: 01 M2, 02 L3, 03 §2.

#### Q2. Namespace for other steps

**Recommend `Steps.<Name>.Output.<Var>` and `Steps.<Name>.TaskParam.<Param>`**, with referenced steps required to
have `<Identifier>` names (T2). Alternative to float: drop `.Output` and reserve a short list of property names.
Source: 01 Q1.

#### Q3. Direct or transitive dependents

**Recommend direct only** (T5). Source: 01 Q2.

#### Q4. Output shape and reduction model

**Recommend:** scalar for a producer without `parameterSpace`, `list[T]` in iteration order otherwise; reductions are
ordinary EXPR; no declared reductions (T6, §3.5). Sources: 01 Q3, M4; 03 Q8.

#### Q5. Correlating a reduced value with its task

**Recommend `Steps.<Name>.TaskParam.*` lists in v1**, and a separate, small RFC 0006 addendum for
`argmin(list[T]) -> int` and `argmax`. Ask whether a list `index(list, item)` or `zip` is preferred instead. Sample 4.3
is the test case. Sources: 01 Q14, 03 Q4.

#### Q6. Which fields may reference outputs, and when they are evaluated

**Recommend:** the materialization expressions of §2.1 (evaluated at step input binding, in order, atomically) plus
host-scope script fields. Not `combination`, not job-level fields, not environment templates. Source: 01 Q4.

#### Q7. Emission rules

**Recommend:** only step `onRun`; last line per name wins; a missing declared output, malformed line, undeclared name,
type mismatch, or constraint violation fails the task; a failed attempt's values are discarded; outputs are not
secrets and are not redacted. Ask whether anyone needs outputs from an environment's `onEnter` (for example a
discovered license server). Source: 01 Q5.

#### Q8. Outputs from chunked steps

**Recommend deferring:** a step with a `CHUNK[INT]` parameter cannot declare `outputVariables` in v1. Coordinate the
eventual per-task record format with #169 checkpointing. Source: 01 Q11, M2.

#### Q9. Size and fan-out limits

**Recommend:** at least 1 MiB of output per task guaranteed, constraints on `outputVariables` checked at the producer,
and a scheduler-defined per-step task cap checked at binding (T7). Source: 01 Q6.

#### Q10. Empty computed parameter space

**Recommend:** a zero-task `SUCCEEDED` step whose `Output` lists are `[]` (T4). Call out that range strings
(`"1-{{N}}"`) still cannot be empty. Sources: 01 Q6, 03 Q2.

#### Q11. `enableIf` semantics and placement

**Recommend:** `SKIPPED` satisfies `dependsOn`; a skipped producer's `Steps.*` symbols are `T?` and `null`; `enableIf`
is evaluated before step `let`; it is allowed on steps and environments, not host requirements; it ships in
CONTROL_FLOW, not as its own extension (T3, §3.3, §5.1). Because EXPR's type checker accepts `T?` where `T` is
required, the RFC should require a validation warning when a conditional producer's symbol is interpolated into a
format string with surrounding text, where `null` silently becomes `""`. Ask whether a warning is enough or whether
that use should be an error. Ask whether issue #80 users need static `enableIf` sooner than CONTROL_FLOW can ship.
Sources: 01 Q7, 03 Q3.

#### Q12. Producer-declared defaults

**Recommend deferring** (§5.3). Sources: 02 L4, 03 §4.3 option (c).

#### Q13. Re-running a succeeded producer

**Recommend:** values are committed when the producing task succeeds. A re-run that writes equal values is a no-op. A
re-run that writes different values after a consumer has bound fails the re-run with a message naming the consumer.
Cascade recompute is a possible scheduler capability, not part of v1. Ask render-farm operators whether "fail the
re-run" is acceptable in practice. Sources: 03 Q1, 02 L11.

#### Q14. Task-task dependencies: inside CONTROL_FLOW or beside it

**Recommend a separate, independent `TASK_DEPENDENCIES` extension** with its own RFC from #82, composed with
CONTROL_FLOW through the materialization rule (§5.2). Its own questions (field name, omitted parameters, 1:1 sugar,
`anyOf`, `itself`, chunking, edge-evaluation cost) move to that RFC. `anyOf` raises no determinism problem with data
flow, because outputs are only visible after every producer task is terminal. Sources: 01 Q8–Q10, 03 Q7.

#### Q15. PATH-typed outputs

**Recommend deferring;** emit `STRING` paths relative to a PATH job parameter and join in the consumer
(`Param.WorkDir / Steps.P.Output.RelPath`), which keeps path mapping on the parameter. New question from this synthesis.

#### Q16. Extension structure and the EXPR dependency

**Recommend:** one `CONTROL_FLOW` extension (no outputs-only first extension) that requires `EXPR` and is rejected
without it (§5.1). Sources: 01 Q13, 03 Q5.

#### Q17. A runtime that does not support CONTROL_FLOW

With the file channel this question from 03 mostly disappears: a non-supporting scheduler rejects the template under
RFC 0002, and there is no stdout directive to mishandle. **Recommend** stating that `Task.OutputFile` and
`OPENJD_TASK_OUTPUT_FILE` are defined only for steps that declare outputs. Source: 03 Q6.

#### Q18. Relationship to standalone StepTemplates (#22)

**Recommend** stating that CONTROL_FLOW does not replace them, and leaving open whether a later appended step may
reference `Steps.*`. Source: 01 Q12.

#### Q19. Display of pending steps

**Recommend** that `openjd summary` and scheduler UIs report a `PENDING_INPUTS` step's task count as determined at
run time and name the outputs it waits on. Source: 01 M5, #22.

## 7. Design tenet check

| Tenet | Assessment | Tension |
|---|---|---|
| **Portable** | Gated by the `extensions` list, so a non-supporting scheduler rejects the template instead of mis-running it. The file channel works the same on Linux, macOS, and Windows and through WRAP_ACTIONS. The observable contract (§2.1) allows both deferred-creation and create-then-update schedulers. | Moderate. This is the most demanding extension for schedulers so far: persisted values, a new stage, a new state. Splitting out `TASK_DEPENDENCIES` and deferring chunked outputs keep the minimum smaller. |
| **Expressive** | One mechanism (typed outputs plus a new symbol class in materialization expressions) covers fan-out, conditions, reductions, and, with `TASK_DEPENDENCIES`, computed DAGs, by composing with EXPR rather than adding special cases. | Low. The fixed step graph rules out PDG-style dynamic work (U15) by design, which #22 already accepted. |
| **Human readable and writable** | `outputVariables` declares what flows; `Steps.P.Output.X` reads plainly; ranges and conditions are ordinary EXPR. | Real. (1) Correlating results with configurations needs index arithmetic (sample 4.3); `argmin`/`argmax` would help (Q5). (2) `Steps.P.Output.X` is longer than #111's `Step.P.X`. (3) Appending to `Task.OutputFile` is slightly less obvious than echoing a directive. (4) `"1-{{N}}"` versus `"{{range(1, N + 1)}}"` behaves differently for `N = 0`. (5) A skipped producer's `null` output becomes `""` inside text interpolation, so a plain reading can mislead; the validation warning in Q11 is the mitigation. |
| **Tooling parseable** | Outputs and their types are declared, references and edges are explicit and checked before submission, and the output file has a strict line grammar. This replaces opaque workarounds (pre-submission generators, files per task, numbers packed into `openjd_status`). | Moderate. A job's task count is no longer always known at submission, a regression for `openjd summary` and capacity planning that #22 warned about. Mitigated by the fixed step graph, declared constraints that bound fan-out, and reporting pending steps explicitly (Q19). The new `OPENJD_TASK_OUTPUT_FILE` variable is justified the same way as `OPENJD_SESSION_WORKING_DIR` (T1). |

The tensions worth stating in the RFC's design-tenet section are readability of correlation (until Q5 lands), the
silent `null`-to-`""` case for skipped producers, and the loss of a fully known task count at submission.

## 8. Addendum: emission channel, collect modes, and metrics

Recorded after review of the RFC draft. These decisions replace T1 and Q1, and the corresponding parts of the
samples in §4 and the tenet check in §7.

**Stdout directive, with `@file`.** A Task reports an output with `openjd_output_var: <Name>=<JSON>`, as #111
proposed, or `openjd_output_var: <Name>=@<Path>` for a value in a file. T1's argument was that third-party text could
forge an output. Review rejected it as a security argument: code in `onRun` already runs as the Job's user, so a value
it prints grants nothing new, and GitHub's `::set-output` deprecation was about untrusted pull-request text steering
more privileged steps, which OpenJD lacks. What remains is a correctness and reliability concern, and strictness
answers it: a directive must name a declared output and be well formed and well typed, or the Task fails. The stdout
channel keeps one protocol for everything an Action reports, needs no per-attempt file, and lets the runtime validate
and forward values while the Task runs. `@<Path>` removes the 64 KiB line limit.

**`@` files are read immediately.** The runtime reads the file when it reads the directive, not when the Action exits,
so it can forward the value at once and reduce the latency between a producer finishing and its consumers being
bound. The Action must finish writing the file before printing the directive.

**The one security rule.** The runtime must read an `@` file with no more access than the Action's user. The runtime
may run as a different user from the Action, and without the rule a Task could copy a file it can't read, for example
through a hard link, into stored, displayed outputs. Separately, an `@` path must resolve to a regular file inside the
Session's Working Directory, which settles how relative paths work inside a container under WRAP_ACTIONS.

**Collect modes.** Each output declares `collect: ONCE | LAST | APPEND`, default `ONCE`:

* `ONCE` — exactly one directive; a repeat fails the Task. Replaces T1's last-write-wins, which would let a stray
  directive silently replace a value.
* `LAST` — the last of one or more directives, for running figures.
* `APPEND` — zero or more directives, each one element of a `LIST[T]`, in the order read. Append rather than
  concatenate, because concatenation is ambiguous for `LIST[LIST[INT]]`.

No mode reduces at the producer; consumers reduce with EXPR, as §3.5 already decided for reductions across Tasks.

**Metrics are output variables.** The Task metrics proposal (`TASK_METRICS`, `openjd_metric: <name>=<number>`) and
folouiseAWS's use case on #111 are covered by output variables with no consumer. Output declarations gain an optional
`unit` for display. Metrics follow the same strict rules as other outputs, rather than the proposal's
ignore-with-a-warning, so the system has one mechanism with one set of rules. A split into an outputs-only extension
was reconsidered for metrics and rejected again, for the same reason.

**Consequences.** `Task.OutputFile` and `OPENJD_TASK_OUTPUT_FILE` are removed. The runtime work moves back to the
action filter (03's design), plus reading `@` files, and the Rust stdout framer must stop truncating over-long lines
silently, because a truncated directive must be invalid rather than parsed.

## 9. Addendum: `PATH` outputs and job outputs

Recorded after further review. These decisions replace the deferral of `PATH`-typed outputs (Q15) and add job
outputs, which this note did not consider.

**`PATH` outputs are stored in the submission path context.** A Task reports a path in its worker's form; the runtime
applies the Session's path mapping rules in reverse and stores the result in the form `PATH` job parameters are given
in. Each consumer maps it forward with its own rules, as for `Param.*`. How the inverse is computed is
implementation-defined; Deadline Cloud reverses the mapping from the worker's context to the job's submission
context. A path with no submission-context form that maps back is invalid when the Session has rules (it is worker-local, typically in the Session
Working Directory) and is stored unchanged when it has none. `Steps.P.RawOutput.*` and `Steps.P.TaskRawParam.*` read
the unmapped forms, as `RawParam.*` does, which replaces the `STRING`-joined-to-`PATH` workaround recommended in Q15.

**Job outputs.** A Job Template declares `jobOutputDefinitions`, the counterpart of `parameterDefinitions`. Each is a
typed *value* expression over job parameters and the `Steps.*` symbols of any Step, never written by a Task:

* evaluated as soon as possible, one output at a time, so a user interface can show values before the Job finishes;
* a failure to evaluate fails the Job at once, matching the strict handling of invalid directives;
* not available as symbols in the template, since nothing in a Job runs after them;
* re-evaluated when a re-run of a producer commits new values; job outputs are not consumers for the re-run rule;
* part of `CONTROL_FLOW`, not a separate extension.
