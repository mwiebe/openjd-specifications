# CONTROL_FLOW investigation 01: Prior art in OpenJD

This note surveys what the OpenJD community has already proposed about run-time control flow, and
what the accepted EXPR work (RFCs 0005, 0006, 0007) gives a CONTROL_FLOW extension to build on.
External workflow systems (Argo, Airflow, Houdini PDG, CWL, and similar) are out of scope for
this note and are surveyed in a companion note, `02-external-prior-art.md`.

Sources read in full, including every comment and reply (fetched 2026-10-15):

| Source | Title | Opened | Participants |
|---|---|---|---|
| [#111](https://github.com/OpenJobDescription/openjd-specifications/discussions/111) | Idea for RFC for more dynamic job structure via variable data flow along step dependencies | 2026-02-13 | mwiebe, crowecawcaw, folouiseAWS |
| [#81](https://github.com/OpenJobDescription/openjd-specifications/discussions/81) | Idea for RFC to use an expression to include/exclude parts of a template | 2025-03-25 | mwiebe (no replies) |
| [#82](https://github.com/OpenJobDescription/openjd-specifications/discussions/82) | Idea for RFC to represent task-task dependencies, between individual tasks of steps | 2025-03-25 | mwiebe, andymanaws, epmog |
| [#22](https://github.com/OpenJobDescription/openjd-specifications/discussions/22) | Immutability vs. dynamic adjustments (General category) | 2024-01-12 | jvanns, ddneilson, mwiebe, pkpriority, rvinluan-sidefx |
| [#79](https://github.com/OpenJobDescription/openjd-specifications/discussions/79) | Idea for RFC to extend types and the template substitution language (became EXPR) | 2025-03-10 | mwiebe |
| Ideas category | All 22 open Ideas, skimmed for control-flow-adjacent proposals (see [§1.4](#14-adjacent-ideas)) | — | — |
| [RFC 0002](../../rfcs/0002-model-extensions.md) | Model Extensions | accepted 2025-01-17 | — |
| [RFC 0005](../../rfcs/0005-expression-language.md) | Expression Language (EXPR) | — | — |
| [RFC 0006](../../rfcs/0006-expression-function-library.md) | Expression Function Library | — | — |
| [RFC 0007](../../rfcs/0007-extend-parameter-types.md) | Extended Parameter Types | — | — |
| [How-Jobs-Are-Run](../../wiki/How-Jobs-Are-Run.md), [Template Schemas §3.2, §7.3–7.4](../../wiki/2023-09-Template-Schemas.md) | Current scheduling and evaluation model | — | — |

## Summary

* The community proposals fall into three families: **data flow** (#111: typed step outputs,
  `openjd_output_var:`, `Step.<Name>.<Var>`, reductions), **conditional structure** (#81:
  `enableIf` on steps, environments, and host requirements), and **finer-grained dependencies**
  (#82: `parameterSubspace`/`dependsOnSubspace`, `anyOf`/`allOf`, self-dependencies). #111 already
  composes all three: its scene-graph example feeds a computed adjacency list into a #82
  self-dependency, and it names #81 `enableIf` as a consumer of output variables.
* Every proposal keeps the **shape** of the job static (steps, names, types, dependency edges are
  declared in the template) and makes only **sizes, values, and per-task edges** dynamic. That is
  the line #22 drew: the maintainers chose ahead-of-time structure as the base and asked for
  "specific dynamic job structures" to be added through RFCs. Fully open-ended job growth
  (Houdini PDG style) was pointed at a separate mechanism, standalone StepTemplates submitted
  into a running job.
* EXPR already provides most of what CONTROL_FLOW needs: the types (including `list[list[int]]`
  for adjacency lists), progressive evaluation with `unresolved[T]` placeholders, symbol
  collection for data-flow analysis, and the list functions needed for most reductions. The
  main gaps are a namespace for other steps' outputs (the proposed `Step.<Name>.<Var>` collides
  with the existing `Step.Name` and cannot address step names that are not identifiers), a new
  evaluation stage between job creation and task execution, and a handful of functions
  (`argmin`/`argmax`, a list `index`, `zip`/`enumerate`) for the pick-a-winner case.
* The parameter-sweep comment (folouiseAWS, 2026-10-12) proposes **shipping in two parts**:
  first task outputs reported and visible to the scheduler, then data flow into downstream steps.
  That split is worth taking seriously when drafting the RFC.

## 1. Catalogue of proposed mechanisms

Each entry gives the source, a syntax sketch as written in the thread (normalized only where
noted), and the open questions. "Thread" questions were raised by participants; "noted"
questions are ones this survey found while reading and are marked as such.

### 1.1 Data flow along step dependencies (#111)

#### M1. Declared, typed step output variables — `outputVariables`

Source: #111 body, §Idea bullets 2–4 and Example 1.

```yaml
steps:
  - name: VideoToFrames
    outputVariables:
      - name: FrameCount
        type: INT
```

The template declares every variable each step outputs, with its type, so a validator can type
check downstream references before any task runs. A task that finishes without emitting a declared
variable is an error. The sweep comment uses an inline-mapping form,
`- {name: mpjpe_mm, type: FLOAT}`, which is the same schema.

Open questions:

* Thread: none raised beyond the proposal itself.
* Noted: Which types are allowed? The example uses `INT` and #111 says the scene-graph case needs
  `LIST[LIST[INT]]`, which suggests the job-parameter type vocabulary (RFC 0007), but that
  vocabulary only has `LIST[LIST[INT]]` for nesting, not `LIST[LIST[STRING]]` etc.
* Noted: Are constraints (`minValue`, `allowedValues`, `maxLength`) allowed on output variables, as
  they are on job parameters? They would let the scheduler bound fan-out before it happens (see
  M6).
* Noted: Is an output optional (`INT?` style, cf. RFC 0007 Open Questions) or always required?

#### M2. Runtime emission — `openjd_output_var:` stdout message

Source: #111 §Idea bullet 5, Example 1 script; sweep comment shows the Python form.

```bash
echo "openjd_output_var: FrameCount=$FRAME_COUNT"
```

```python
print(f"openjd_output_var: mpjpe_mm={score.mpjpe_mm}")
```

The value after `=` is JSON convertible to the declared type. Modeled on `openjd_env:` (How Jobs
Are Run, §Stdout/Stderr Messages).

Open questions:

* Noted: `openjd_env` may only be emitted by an Environment's `onEnter`. Can `openjd_output_var`
  be emitted by `onRun` only, or also by environment actions (e.g. an environment that discovers
  a license server)?
* Noted: Under TASK_CHUNKING one `onRun` executes a whole chunk. How does a chunk report a value
  per task? (Possibly an index or task-parameter key in the message, or a list value.) RFC
  discussion #169 (checkpointing) needs a similar "which tasks in the chunk" report and may share
  a message design.
* Noted: Multiple emissions of the same variable: last-wins, error, or append?
* Noted: Retried tasks: does a retry discard the earlier attempt's values?
* Noted: Size limits. One stdout line has no defined maximum; a scene-graph adjacency list can be
  large. EXPR bounds evaluation memory at ~100 MB by default; output values need their own bound.
* Noted: Under WRAP_ACTIONS the scheduler scans the wrap script's stdout, which forwards the
  wrapped process verbatim, so the message passes through unchanged. Worth stating explicitly.
* Noted: Should `openjd_redacted_env`-style redaction exist for outputs (secrets flowing between
  steps)?

#### M3. Referencing upstream outputs — `Step.<StepName>.<VarName>`

Source: #111 §Idea bullet 2, Example 1 (`range: "1-{{Step.VideoToFrames.FrameCount}}"`),
Example 2 (`Step.AnalyzeSceneGraph.AdjList[...]`).

A variable is "available for template expression substitution in all steps that have a
downstream dependency from it". The scheduler cannot evaluate an expression that references it
until the upstream step finishes.

Open questions:

* Noted: "Downstream dependency" is ambiguous between direct dependents and transitive
  descendants. Transitive access is convenient but makes the data flow less obvious to tooling.
* Noted: Name collision. EXPR already defines `Step.Name` (string, the current step's name; Schemas
  §7.3.1). A step called `Name` makes `Step.Name` ambiguous, and `Step` otherwise means "the
  current step". See [§3.2](#32-symbol-namespaces).
* Noted: `<StepName>` allows any Unicode except category Cc (Schemas §3.1), so `Render Frames` or
  `Export-v2` cannot be written as `Step.Render Frames.X`. Either referenced steps must have
  identifier names, or the syntax needs another form.
* Noted: Can a step reference its own outputs (e.g. a self-dependent step reading its predecessor
  task's output)? This interacts with M9.

#### M4. Reductions over multi-task steps

Source: #111 §Idea bullet 7.

> "There will need to be some kind of reduction operation when a step has multiple tasks. E.g. put
> all the variable values into a list in the parameter space iteration order, take the
> maximum/minimum value, take the sum, etc."

No syntax was proposed. Two designs are implied: a declared reduction per output (e.g.
`reduce: LIST | MAX | MIN | SUM`), or always exposing the per-task list and letting EXPR reduce it
(`max(Step.Sweep.fps)`). The sweep comment needs the second form, because "pick the winner" must
correlate three metrics with the task parameters that produced them, which a scalar reduction
loses.

Open questions:

* Noted: "Parameter space iteration order" is defined for a step's tasks, but schedulers may run
  tasks in any order and some may be retried; the list order must be the defined order, not the
  completion order.
* Noted: Do downstream expressions also need the task parameter values that produced each element
  (`Step.Sweep.Task.Param.Window[i]`?) to pick a winning configuration?
* Noted: EXPR caps list nesting at two levels, so a per-task `LIST[INT]` output collects into
  `list[list[int]]`, but a per-task `LIST[LIST[INT]]` output cannot be collected.
* Noted: Does a step with a single task (no parameter space) expose a scalar or a one-element
  list? #111 Example 1 treats it as a scalar.

#### M5. "Unable to evaluate yet" step state

Source: #111 §Idea bullet 6.

A step whose definition references unavailable outputs is in a new state. A farm can either delay
creating the step until values exist, or create it pending and fill it in later. Both are allowed.

Open questions:

* Noted: Which fields may be deferred: only `parameterSpace` ranges, or also `hostRequirements`,
  `stepEnvironments`, `let`, `enableIf`? The Job-creation stage (Schemas §7.4) evaluates all
  TEMPLATE-scope format strings, so each deferred field needs a defined stage.
* Noted: What does a UI or `openjd summary` show for task count before the step is evaluable?
* Noted: #22's operations concern ("when will it be done?") argues for exposing this state
  explicitly rather than hiding it inside the scheduler.

#### M6. Dynamic parameter-space ranges from upstream outputs

Source: #111 Examples 1 and 2; crowecawcaw's S3-prefix comment.

```yaml
range: "1-{{Step.VideoToFrames.FrameCount}}"
range: "0-{{len(Step.AnalyzeSceneGraph.AdjList) - 1}}"
```

The S3 case implies a `STRING` or `PATH` range from a list output, e.g.
`range: "{{Step.Discover.Keys}}"`, which EXPR already permits syntactically
(`ListExpressionString`, RFC 0005 §Task Parameter Range Field Extensions).

Open questions:

* Noted: Bounding fan-out. A buggy discovery step could emit a million keys. Declared limits on
  the output (M1 constraints) or on the step (`maxTaskCount`?) let the scheduler fail fast.
* Noted: An empty range (zero objects under the prefix) is a step with no tasks. Is that success,
  skip, or error? This overlaps with `enableIf` (M7).

#### M7. Computed DAG flowing into task-task dependencies

Source: #111 Example 2.

```yaml
- name: ProcessGraphNode
  dependencies:
    - dependsOn: AnalyzeSceneGraph
    - dependsOn: ProcessGraphNode
      dependsOnSubspace:
        - name: NodeIndex
          range: "{{Step.AnalyzeSceneGraph.AdjList[Task.Param.NodeIndex]}}"
  parameterSpace:
    taskParameterDefinitions:
      - name: NodeIndex
        type: INT
        range: "0-{{len(Step.AnalyzeSceneGraph.AdjList) - 1}}"
```

This is M3 + M6 + M9 combined: both the task count and the intra-step edges come from an upstream
step. The template knows the job's shape but not the DAG's size or structure.

Open questions:

* Noted: A computed adjacency list may contain a cycle. Static validation cannot catch it; the
  scheduler must detect it when the step becomes evaluable and fail the job.
* Noted: Root nodes have an empty dependency list `[]`. An `INT` task-parameter range today is a
  list literal or a range string; whether an empty dependency subspace is valid needs stating.

### 1.2 Conditional structure (#81)

#### M8. `enableIf` boolean expression

Source: #81 body. Motivated by issue
[#80](https://github.com/OpenJobDescription/openjd-specifications/issues/80) (skip a step via a
boolean param in Deadline Cloud).

```yaml
parameterDefinitions:
- name: ShouldEncodeVideo
  type: BOOL
steps:
- name: CreateVideoFromRender
  enableIf: "{{Param.ShouldEncodeVideo}}"
  dependencies:
  - dependsOn: RenderScene
```

Proposed on three entities: Steps, Job/Step Environments, and individual host requirements (a
CPU-only vs GPU-accelerated template). #111's intro adds the dynamic case: an `enableIf` that
references an upstream output variable.

Open questions (none were raised in the thread, which has no replies; all are noted):

* What happens to steps that depend on a disabled step? Options: they are also disabled
  (cascade), the dependency is treated as satisfied, or it is a validation error. Workflow
  engines differ here; see the external prior-art note.
* Is a disabled step absent from the job (as if never written), or present with a "skipped"
  status? This matters for UI, for reductions (M4) over a skipped step, and for `Step.X.Var`
  references from downstream (null? error?).
* When `enableIf` depends only on `Param.*` it is a job-creation decision; when it depends on
  `Step.*` it is a run-time decision. Same field, two stages.
* Granularity: #81 is step/environment/host-requirement level. Is per-task enablement (filter a
  parameter space by expression) in scope, or already covered by computing the range?
* Overlap with RFC 0005's null-dropping rule for optional fields and list items: a host
  requirement list item that evaluates to `null` is already dropped, so `enableIf` on host
  requirements may be expressible today.

### 1.3 Task-task dependencies (#82)

#### M9. Per-task dependency subspace — `parameterSubspace` / `dependsOnSubspace`

Source: #82 body (as `parameterSubspace`); mwiebe 2026-03-19 comment and #111 Example 2 (as
`dependsOnSubspace`). The field was renamed between the two; neither name is settled.

```yaml
- name: RenderFrame
  dependencies:
    - dependsOn: ExportPerFrame
      parameterSubspace:
        - name: Frame
          range: "{{Task.Param.Frame}}"   # each RenderFrame task waits on the same Frame
```

The `range` expressions are evaluated per dependent task, with `Task.Param.*` bound to the
**dependent** task's parameters, and select tasks of the **depended-upon** step. Values outside
the upstream parameter space are dropped, which makes boundary conditions free: `TimeStepNumber - 1`
for task 0 selects nothing. mwiebe (2026-09-10): "the flexible expressions together with the rule
to drop any out of bounds values handles a lot of similar cases with simple syntax." andymanaws
confirmed it covers look-forward/look-back temporal offsets (motion blur, temporal curves).

Patterns shown in #82 and mapped to OpenCue dependency modes (mwiebe, 2026-03-19):

| Pattern | Subspace expression | OpenCue mode |
|---|---|---|
| Frame-by-frame | `range: "{{Task.Param.Frame}}"` | FrameByFrame |
| Whole step | (no subspace; today's behavior) | LayerOnLayer |
| Previous task, self | `range: ["{{Task.Param.TimeStepNumber - 1}}"]` | PreviousFrame |
| Every Nth upstream | `range: ["{{Task.Param.Frame * Param.TimeStepsPerFrame}}"]` | — |
| First upstream task | `range: [0]` or `"{{Param.Frames[0]}}"` | LayerOnSimFrame |
| Any upstream task | `anyOf:` (M10) | LayerOnAny |
| Static DAG, self | `range: "{{Param.GraphAdjList[Task.Param.NodeIndex]}}"` | — |
| Multi-parameter, self | one entry per parameter: `Frames - 1`, same `Layers` (epmog) | — |

Open questions:

* Thread (epmog): The common 1:1 case is verbose. Suggested sugar `parameterSubspace: [{type: TASK}]`
  (map each task to the identically-parameterized upstream task), while saying "doesn't quite look
  right".
* Noted: epmog's layers × frames example (`combination: "Frames * Layers"`) gives one subspace
  entry per task parameter. Whether an omitted parameter means "all values" or "same value" is
  unstated.
* Noted: How does this interact with TASK_CHUNKING? A chunked downstream task depends on the union
  of its tasks' subspaces; a chunked upstream completes several tasks at once.
* Noted: Edges evaluated per task can be O(N×M). The scheduler needs this to be computable without
  materializing every pair, or a bound.

#### M10. `anyOf` / `allOf` in the subspace

Source: #82 body ("inspired by OpenCue LayerOnAny"); example "any frame creates sim cache".

```yaml
- dependsOn: RenderWithSim
  parameterSubspace:
    anyOf:
      - name: Frame
        range: ["{{Step.RenderWithSim.ParameterSpace.Frame.Range}}"]
```

`allOf` is the default when the list is directly under `parameterSubspace`.

Open questions:

* Noted: `anyOf` is satisfied by the first completed upstream task, but which task that is is
  nondeterministic. If outputs (M2) also flow, downstream results depend on scheduling order.
* Noted: The list-vs-mapping shape changes depending on whether `anyOf` is used, which complicates
  the schema.

#### M11. Introspecting another step's parameter space — `Step.<Name>.ParameterSpace.<Param>.Range`

Source: #82, the `anyOf` example. Not discussed further.

This is a second proposed use of a `Step.<Name>.*` namespace, for static structure rather than
outputs. It should be designed together with M3.

#### M12. Self-dependency

Source: #79 (motivating cases), #82 sim and DAG examples, #111 Example 2. epmog suggests a
reserved keyword (`dependsOn: itself`) so the step name isn't repeated.

Today the step-dependency graph must be acyclic and self-dependency is rejected (openjd-rs
`structure.rs`: "cannot depend on itself."). A self-dependency is only meaningful with a subspace;
the CONTROL_FLOW RFC must permit it only in that case, and the task-level graph must still be
acyclic.

#### M13. `dependsOn` accepting a list

Source: epmog, #82 (2026-09-10).

```yaml
dependencies:
  - dependsOn: [A, B, C, D]
    parameterSubspace:
      name: frame
      range: ["{{Task.Param.Frame}}"]
```

Fan-in from several steps with the same subspace mapping. Pure syntax sugar.

### 1.4 Adjacent ideas

#### M14. Standalone StepTemplate submitted into a running job (#22)

Source: rvinluan-sidefx and ddneilson (2024-04/05), #22.

Houdini PDG creates work items dynamically. The proposed answer was not a template feature but a
scheduler API (`submitTaskToJob(job_id, task_template)`) plus, optionally, a new root element for
a standalone StepTemplate (like Environment Templates) and an `openjd` model function that checks
appended steps are compatible with the job.

Relevance: this is the escape hatch for fully dynamic structure. CONTROL_FLOW should state that
it does not replace it and how the two compose (can an appended step reference `Step.X.Var`?).

#### M15. Status-quo workarounds named in threads

Not mechanisms to adopt, but evidence of the need:

* A pre-submission script that discovers inputs and generates the template (crowecawcaw). It
  breaks job bundle submitters, which need the template to exist up front.
* A task that calls the farm's own submit command (mwiebe, #22). Not portable across schedulers.
* Writing a file per task plus an aggregation step, packing numbers into `openjd_status`, or
  logging to an external tracker (folouiseAWS).

#### Other Ideas-category proposals checked

| Idea | Relation to CONTROL_FLOW |
|---|---|
| [#42](https://github.com/OpenJobDescription/openjd-specifications/discussions/42) Conditionally show job parameter UI | Same "expression gates a part of the template" idea as `enableIf`, applied to `userInterface`. Submit-time only. Out of scope but should share syntax if both land. |
| [#97](https://github.com/OpenJobDescription/openjd-specifications/discussions/97) `onRun` as a list of actions | Raises failure semantics for sequential actions. If adopted, `openjd_output_var` from any action in the list must be defined. |
| [#122](https://github.com/OpenJobDescription/openjd-specifications/discussions/122) Safely serializing JSON data | Notes EXPR has no dict/map type, so structured data can only be passed as parallel lists. Same limitation hits structured step outputs (sweep metrics). |
| [#145](https://github.com/OpenJobDescription/openjd-specifications/discussions/145) Service entity (draft RFC 0009 on `mwiebe/rfc-service`) | Precedent for a late-bound scope (`Service.<Name>.<port>.*`, typed `unresolved[...]` until task execution) and for adding a new key to `<StepDependency>` (`service:`). Closest existing design to copy for `Step.<Name>.<Var>` in host-scope fields. |
| [#169](https://github.com/OpenJobDescription/openjd-specifications/discussions/169) Task checkpointing | New stdout messages (`openjd_checkpoint_files:` with JSON payload), reporting completed sub-tasks within a chunk, and output data ready before the task ends. Likely shares message conventions with `openjd_output_var`. |
| [#52](https://github.com/OpenJobDescription/openjd-specifications/discussions/52) Task chunking (RFC 0001) | Chunks interact with per-task outputs (M2) and per-task dependencies (M9). |
| [#79](https://github.com/OpenJobDescription/openjd-specifications/discussions/79) EXPR idea | Original home of the self-dependency, `Task.Param.Index - 1` chain, 1:1 render-on-export, and "skip a step" motivations; spun out into #81 and #82. |

The other Ideas (#17, #35, #36, #41, #49, #59, #77, #83, #84, #104, #118, #136) have no
control-flow content.

## 2. User use cases collected

| ID | Use case | Source | What is unknown at submit | Mechanisms |
|---|---|---|---|---|
| U1 | **Video frame count.** Extract frames, process one task per frame, re-encode. The submitter may not have the video or a reliable frame counter. | #111 Ex. 1 | Task count of the middle step | M1, M2, M3, M5, M6 |
| U2 | **Scene-graph DAG.** Analyze a 3D scene (lightmap baking, GI, spatial partitioning) into a DAG; leaves run in parallel, parents wait for children. | #111 Ex. 2 | Task count **and** intra-step edges | M1–M3, M5–M7, M9, M12 |
| U3 | **S3-prefix fan-out.** One discovery task lists objects under a prefix; one processing task per object. Currently a pre-submission script, which blocks use of the Deadline job bundle submitter. | crowecawcaw, #111 | Task count and `STRING`/`PATH` task-parameter values | M1–M3, M5, M6 |
| U4 | **Parameter sweep → pick winner.** 27 configs × 10 benchmark episodes = 270 tasks (e.g. video → 3D hand trajectories), each reporting `mpjpe_mm`, `coverage_pct`, `fps`. Sort and pick the best; then run the winner on the full dataset in the same job, possibly one task per video under an S3 prefix (= U3). | folouiseAWS, #111 | Which configuration the final step uses | M1, M2 (phase 1); M3, M4 (phase 2) |
| U5 | **Optional step from a bool parameter** (e.g. skip video encode). | #81, issue #80, #79 | Nothing (submit-time) | M8 |
| U6 | **Optional environment** (`onEnter`/`onExit` gated by a parameter). | #81 | Nothing | M8 |
| U7 | **CPU-only vs GPU host requirements** chosen by a parameter. | #81 | Nothing | M8 (or EXPR null-dropping) |
| U8 | **Export → render per frame.** Windows-only export, Linux render; render each frame as soon as its export finishes. | #82, #79 | Nothing | M9 |
| U9 | **Simulation chain + render every Nth step.** Each time step depends on the previous; each rendered frame depends on its time step. | #82 | Nothing | M9, M12 |
| U10 | **Static DAG over a dataset** from a `LIST[LIST[INT]]` adjacency-list parameter. | #82, RFC 0007 | Nothing | M9, M12 |
| U11 | **Sim cache from any frame / first frame.** Renders start once any (LayerOnAny) or the first (LayerOnSimFrame) sim frame is done. | #82, mwiebe comment | Nothing | M9, M10, M11 |
| U12 | **Layers × frames, previous frame of same layer.** | epmog, #82 | Nothing | M9, M12 |
| U13 | **Temporal offsets** (look ahead/behind for motion blur, temporal curves). | andymanaws, #82 | Nothing | M9 |
| U14 | **Fan-in from several steps** with the same per-frame mapping. | epmog, #82 | Nothing | M9, M13 |
| U15 | **Houdini PDG dynamic work items** (crowd FX). | jvanns, rvinluan-sidefx, #22 | Number and shape of work items | M14 (out of template) |

Observations:

* U5–U14 are fully static: everything is known at job creation. They need new **expressiveness**
  (conditionals, per-task edges) but no new **evaluation stage**. U1–U4 need data from a running
  task, which is the novel part of CONTROL_FLOW and where #22's predictability concerns apply.
* U2 is the only case that needs every mechanism at once, which makes it the stress test for the
  RFC's examples.
* U4 phase 1 (report outputs, no downstream use) has value on its own: the scheduler and UI can
  show per-task metrics. That supports folouiseAWS's split.
* U15 is explicitly not something the template mechanisms aim to cover.

## 3. What CONTROL_FLOW can rely on from EXPR vs. would need to add

EXPR is defined by RFCs 0005–0007 and specified in
[2026-02-Expression-Language](../../wiki/2026-02-Expression-Language.md). In this worktree all
three RFC files still read "Accepted On: (pending)"; this survey treats them as the baseline the
epic was given.

### 3.1 Types

| Need | EXPR today | Verdict |
|---|---|---|
| Scalar outputs (`FrameCount`, metrics) | `int` (64-bit), `float`, `bool`, `string`, `path` | Rely on |
| Frame ranges | `range_expr`; `CHUNK[INT]` task params are `range_expr` | Rely on |
| Lists of keys/paths (U3) | `list[T]` for scalars; `LIST[STRING]`, `LIST[PATH]` job params (RFC 0007) | Rely on |
| Adjacency list (U2) | `list[list[int]]`; `LIST[LIST[INT]]` job param exists specifically for #82 (RFC 0007) | Rely on |
| Optional/missing outputs, skipped steps | `T?`, `nulltype`; null-dropping for optional fields and list items | Rely on |
| Values not yet known | `unresolved[T]` with type-level coercion rules | Rely on — this is the core enabler |
| Reduction of per-task `list[T]` outputs | `list[list[T]]` (max nesting 2) | Rely on, with a limit: per-task `list[list[T]]` outputs cannot be collected |
| Structured per-task results (3 metrics + config per task) | No map/record type (rejected in RFC 0007 "Map/Dictionary Types"; also noted in #122) | Gap. Workaround: parallel lists, one per output variable. Decide whether CONTROL_FLOW lives with that. |
| Output-variable type names | Job-parameter type names are case-insensitive (RFC 0007) and map 1:1 to expression types | Rely on: reuse the job-parameter type vocabulary |
| Nested job-parameter list types other than `LIST[LIST[INT]]` | Not defined | Gap only if outputs need e.g. `LIST[LIST[STRING]]` |

### 3.2 Symbol namespaces

Existing symbols (Schemas §7.3.1 and RFC 0005 §Built-in Symbols):

| Symbol | Type | Where available |
|---|---|---|
| `Param.*`, `RawParam.*` | per declaration | everywhere (PATH `Param` only in host contexts) |
| `Task.Param.*`, `Task.RawParam.*` | per declaration | step script actions and embedded files |
| `Task.File.*`, `Env.File.*` | `path` | step / environment scripts |
| `Session.WorkingDirectory`, `Session.PathMappingRulesFile`, `Session.HasPathMappingRules` | `path`, `path`, `bool` | environment and step scripts |
| `Job.Name` | `string` | everywhere except the job `name` field |
| `Step.Name` | `string` | the current step's `stepEnvironments`, `hostRequirements`, `parameterSpace`, `script` |
| `let` names | inferred | must match `[a-z_][A-Za-z0-9_]*`, so they cannot collide with spec symbols |
| `WrappedStep.Name`, `WrappedAction.*` | various | `onWrapTaskRun` etc. (RFC 0008 WRAP_ACTIONS) |
| `Service.<Name>.<port>.*` | `string`/`int` | draft RFC 0009, host stage only |

What CONTROL_FLOW needs and how it fits:

1. **Other steps' outputs.** #111 and #82 both write `Step.<StepName>.<...>`. Problems:
   * `Step` already means "the current step" (`Step.Name`). Overloading it for "some other step"
     means a step named `Name` is unreachable, and future `Step.<Property>` additions become
     breaking changes. RFC 0008, which also needed to name a step other than the current one, used
     a separate `WrappedStep.*` root.
   * Step names may contain spaces, dots, and dashes; dotted access can't express them. EXPR's
     subscript operator works on lists, ranges, and strings only, so `Steps["Render Frames"]` is
     not available either.
   * Candidate directions for the RFC (not decided here): a distinct root such as `Steps.<Name>`
     or `Upstream.<Name>`, plus a rule that a step must have an `<Identifier>`-shaped name to be
     referenced.
2. **Dependent task's parameters inside a dependency subspace.** #82 evaluates
   `Task.Param.Frame` inside `dependencies[].parameterSubspace` of the dependent step. Today
   `Task.Param.*` is only in scope in script actions and embedded files, and the subspace is
   evaluated at a different time and place (scheduler, not worker). The RFC must extend the scope
   table for this field.
3. **Another step's static structure.** #82's `Step.<Name>.ParameterSpace.<Param>.Range` (M11).
4. **Per-task values and their task parameters** for reductions that must correlate results with
   configurations (U4). Not proposed yet.
5. **Static data-flow checking** is already supported by EXPR's symbol collection ("Analyze data
   dependencies — Track how values flow between ... steps"). A validator can collect `Step.*`
   symbols and check they name declared outputs of steps reachable through `dependencies`.

### 3.3 Evaluation stages

Today (Schemas §7.4) there are three stages: template validation, job creation, task execution.
`parameterSpace` ranges are TEMPLATE scope and are fully evaluated at job creation.

* **Rely on:** progressive evaluation. The same evaluator with `unresolved[T]` placeholders
  type-checks an expression referencing an unknown output at validation and job creation, and
  evaluates it once the value exists. No second type checker is needed.
* **Rely on:** the host stage for script-side references. A step script reading
  `Step.X.Var` behaves like draft RFC 0009's `Service.*`: unresolved until the task runs.
* **Need to add:** a stage between job creation and task execution — call it *step evaluation* —
  at which a step's TEMPLATE-scope fields (`parameterSpace`, `enableIf`, `hostRequirements`,
  step `let`, dependency subspaces) that reference upstream outputs are evaluated. This is #111's
  "unable to evaluate yet" state (M5) expressed in EXPR terms. Its position relative to scheduling
  decisions (host requirements affect placement) must be specified.
* **Need to add:** where per-task dependency subspaces (M9) are evaluated. They are evaluated by
  the scheduler per dependent task, which is neither job creation (once per job) nor task
  execution (on the worker).

### 3.4 Functions (RFC 0006)

Rely on:

* `len`, `range`, list subscripting and slicing (`AdjList[Task.Param.NodeIndex]`).
* Reductions: `min`, `max`, `sum` (with empty-list rules), `any`, `all`, `sorted`, `reversed`,
  `unique`, `flatten`.
* List comprehensions with one `for` and an optional `if` filter.
* Conditional expressions and short-circuit `and`/`or` for `enableIf`.
* `fail(message)` to reject bad upstream data with a clear error (e.g. frame count ≤ 0).
* `range_expr(list[int])` to turn a computed list of frames back into a range.
* `repr_json`, `repr_sh`, `repr_py`, `repr_pwsh`, `repr_cmd` to hand outputs to scripts safely.

Gaps for the use cases:

* **Pick the winner (U4).** No `argmin`/`argmax`, no list `index()` (only the string `index`), no
  `sorted` with a key, no `zip`/`enumerate`, and comprehensions can't nest. It is still
  expressible, if awkwardly:

  ```yaml
  let:
    - scores = Step.Sweep.mpjpe_mm
    - best = min(scores)
    - winner = [i for i in range(len(scores)) if scores[i] == best][0]
  ```

  A small addition such as `argmin(list[T]) -> int` / `argmax`, or a list `index(list, item)`,
  would make this direct. Correlating `winner` with the configuration also needs per-task
  parameter values (§3.2 item 4).
* **No JSON parsing function.** Not needed if the runtime parses the `openjd_output_var` JSON
  payload against the declared type, which is what #111 proposes. Keep parsing out of EXPR.

### 3.5 Bounds and safety

* **Rely on:** EXPR's memory limit (default 100,000,000 bytes) and operation limit (default
  10,000,000) for evaluating expressions over output values.
* **Need to add:** a limit on output value size at emission, a limit on fan-out (task count from
  a computed range), and failure semantics when a computed value violates the declared type.

### 3.6 Extension declaration (RFC 0002)

* Name: `CONTROL_FLOW` fits `[A-Z0-9_]{3,128}`.
* CONTROL_FLOW depends on `EXPR`; #111's example also lists `FEATURE_BUNDLE_1` (for `bash:`
  sugar), which should stay optional.
* RFC 0002 allows a superseding `_2` suffix, and extensions are opt-in per scheduler. If the RFC
  follows folouiseAWS's split, the choice is between two extensions (e.g. outputs only, then data
  flow) or one extension whose first version is the smaller part. RFC 0002 doesn't address
  extensions that depend on other extensions; the RFC should state the dependency explicitly.

## 4. Today's scheduling model and what changes

From How-Jobs-Are-Run and Template Schemas §3.2:

* A `<StepDependency>` is just `dependsOn: <StepName>`, and the dependent step's tasks are
  scheduled "only when the depended-upon Step has fully completed successfully." No per-task
  edges, no self-dependency (rejected by implementations), no cycles.
* Jobs are immutable after submission (#22). The job structure, including every step's task
  count, is known at job creation.
* Sessions may run tasks from any step whose environment stack is identical. Data passes between
  tasks only through files (shared storage or the session working directory) or environment
  variables set by `openjd_env` in an environment's `onEnter`.
* Stdout messages (`openjd_progress`, `openjd_status`, `openjd_fail`, `openjd_env`,
  `openjd_redacted_env`, `openjd_unset_env`) flow from the action to the runtime, never between
  tasks.

CONTROL_FLOW changes three of these:

1. A task's stdout can carry **values** the scheduler persists and makes available to other steps
   (M2).
2. A step's structure can be **undetermined at job creation** and resolved later (M5, M6),
   relaxing the immutability that #22 defended, but only for declared fields and with types known
   up front.
3. Dependencies can be **per task**, including self-edges (M9, M12), so "step complete" stops being
   the only gate.

#22's operations concerns (pkpriority: "when will it be done?", "is it supposed to take this long
or is it sick?") become concrete design requirements: expose the "not yet evaluable" state, keep
fan-out bounded, and keep the step graph static so tooling can still reason about it.

## 5. Consolidated open questions for the RFC

1. Namespace for other steps' outputs: reuse `Step.*` (collides with `Step.Name`) or a new root?
   How to reference steps whose names aren't identifiers?
2. Direct dependents only, or transitive descendants?
3. Reduction model: declared per-output reductions, always-a-list, or both? How to correlate
   values with the task parameters that produced them?
4. Which fields may reference outputs, and at which new stage are they evaluated?
5. Output emission rules: actions allowed, chunks, duplicates, retries, size limit, redaction.
6. Fan-out bounds and empty ranges.
7. `enableIf` semantics for dependents of a disabled step, and whether disabled means absent or
   skipped.
8. Subspace field name (`parameterSubspace` vs `dependsOnSubspace`), defaults for unmentioned
   parameters, 1:1 sugar, `dependsOn` lists, `itself` keyword.
9. `anyOf` determinism when combined with data flow.
10. Cycle detection for computed DAGs, at step-evaluation time.
11. Chunking interaction with per-task outputs and per-task dependencies.
12. Relationship to standalone StepTemplates appended to running jobs (#22).
13. One extension or two (folouiseAWS's split), and how RFC 0002 expresses the EXPR dependency.
14. Functions to add: `argmin`/`argmax` or list `index`; possibly `zip`/`enumerate`.

## 6. Errata in the source discussions

Small inconsistencies in the threads' examples, so later drafts don't copy them:

* #82 DAG example: `range: "0-{{len(Param.GraphAdjList)}}"` creates one task too many (should be
  `len(...) - 1`, as #111 Example 2 writes it); `dependsOn: SimulateStep` should be
  `ProcessGraphNode`; the subspace entry `name: ProcessGraphNode` should be `NodeIndex`.
* #82 simulation example: `{{Param.FrameCount * TimeStepsPerFrame - 1}}` is missing `Param.` on
  `TimeStepsPerFrame`; the self-dependency subspace names `StepNumber` but the task parameter is
  `TimeStepNumber`.
* #82 uses `parameterSubspace`; mwiebe's later comment and #111 use `dependsOnSubspace`.
* epmog's fan-in example has `parameterSubspace` as a mapping (`name:`/`range:`) rather than a
  list of entries.
