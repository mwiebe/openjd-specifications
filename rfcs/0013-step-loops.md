* Feature Name: step_loops
* Author(s): Mark Wiebe <[mwiebe](https://github.com/mwiebe)>
* RFC Tracking Issue: (pending). The numbers `0011`, `0012`, and `0013` are placeholders assigned before tracking
  issues exist, and may change when they are created.
* Start Date: 2026-10-20
* Specification Version: 2023-09 extension CONTROL_FLOW
* Accepted On: (pending)
* Part of: the `CONTROL_FLOW` extension, which this RFC defines together with
  [RFC 0011, Control Flow](0011-control-flow.md) and [RFC 0012, Step Output Variables](0012-step-output-variables.md).
  A template that lists `CONTROL_FLOW` gets all three.
* Depends On:
  * RFC 0011, Control Flow, for the `Steps.*` scope, step input binding, `enableIf`, and the Step states
  * RFC 0012, Step Output Variables, for the output variable types, collect modes, and the re-run rule
* Related:
  * Use case U17 of [RFC 0011 §Use cases](0011-control-flow.md#use-cases), which this RFC covers
  * Discussion #22, immutability vs. dynamic adjustments
    (https://github.com/OpenJobDescription/openjd-specifications/discussions/22)

> **Draft status.** This is the first draft. It replaces the single-Step loop sketch that was in RFC 0011's Future
> Work. [Open Questions](#open-questions) lists the decisions to confirm before it leaves draft.

## Summary

Some Jobs need to repeat part of their work until a condition holds: re-render the frames a quality check rejected,
with more samples, until the check passes; step a solver until it converges; list a bucket one page at a time until
the listing runs out. The work repeated is often more than one Step, and how many times it runs is not known when
the Job is submitted. Today a template can only unroll such a loop into a fixed number of copies of its Steps,
chained with `enableIf`.

This RFC adds **loops** to `CONTROL_FLOW`. A loop is an entry of the Job Template's `steps` list that contains a
group of ordinary Steps, its **body**, instead of a script. The body runs once per **iteration**, and iterations run
one after another, each starting only when the one before it has finished. State passes from one iteration to the
next only through typed **loop variables**: each has an `initial` value and a `next` expression that reads the
outputs of the iteration that just finished. The body's Steps read the current values as `Loop.<Name>`. A `while` or
`until` condition decides whether another iteration runs, and a required `maxIterations` bounds the loop; reaching
it fails the loop. Steps after the loop read the loop's own declared outputs, each evaluated after every iteration
and kept as the last iteration's value (`collect: LAST`) or a list with one value per iteration (`collect: APPEND`).
The template's Steps, the loop's body included, stay fixed and are validated before submission. Only the number of
iterations is late-bound, as a Task count is under RFC 0011.

## Overview

### What this RFC adds

* **Loops in the step list.** A `<StepLoop>` entry, beside `<StepTemplate>` entries, with a name, `dependencies`,
  an optional `enableIf`, a `loop` definition, and `outputVariables`: [`<StepLoop>`](#steploop).
* **Loop variables.** Typed values that each iteration starts with, set by `initial` before the first iteration and
  by `next` after each one: [`<LoopVariableDefinition>`](#loopvariabledefinition).
* **Conditions and a bound.** Exactly one of `while` and `until`, and a required `maxIterations`:
  [`<LoopDefinition>`](#loopdefinition).
* **The `Loop.*` scope.** `Loop.<VarName>`, `RawLoop.<VarName>`, and `Loop.Iteration`, available in the body's
  Steps and in the loop's own expressions: [The `Loop.*` scope](#the-loop-scope).
* **Loop outputs.** Values that Steps after the loop and job outputs read as `Steps.<LoopName>.Output.<VarName>`:
  [`<LoopOutputVariableDefinition>`](#loopoutputvariabledefinition) and [Loop outputs](#loop-outputs).
* **Running a loop.** Iterations, body Step instances, the iteration boundary, failure, Sessions, re-runs, and
  limits: [Running a loop](#running-a-loop).

### A loop at a glance

```yaml
parameterDefinitions:
  - name: Target
    type: FLOAT
steps:
  - name: Setup
    outputVariables:
      - name: Baseline
        type: FLOAT
    # ...
  - name: Refine                  # a loop: it has `loop`, and no `script` or `parameterSpace`
    dependencies:
      - dependsOn: Setup
    loop:
      variables:                  # loop-carried state, shared by every Step in the body
        - name: Pass
          type: INT
          initial: "{{1}}"
          next: "{{Loop.Pass + 1}}"
        - name: Quality
          type: FLOAT
          initial: "{{Steps.Setup.Output.Baseline}}"
          next: "{{Steps.Check.Output.Score}}"
      until: "{{Steps.Check.Output.Score >= Param.Target}}"   # or `while:`; exactly one
      maxIterations: 10
      steps:
        - name: Render
          # ... may read Loop.Pass, Loop.Quality, Loop.Iteration, and Steps.Setup.*
        - name: Check
          dependencies:
            - dependsOn: Render
          outputVariables:
            - name: Score
              type: FLOAT
          # ...
    outputVariables:              # the loop's own outputs, read by the Steps after the loop
      - name: FinalScore
        type: FLOAT
        collect: LAST             # the final iteration's value
        value: "{{Steps.Check.Output.Score}}"
      - name: Passes
        type: INT
        collect: LAST
        value: "{{Loop.Pass}}"
  - name: Publish
    dependencies:
      - dependsOn: Refine
    # ... reads Steps.Refine.Output.FinalScore
```

### Key concepts

**A loop is a node of the step graph.** `Refine` has a name, dependencies, an optional `enableIf`, and the states of
a Step, and the Steps after it depend on it as a whole. It has no Tasks of its own: its work is the Tasks of its
body's Steps.

**An iteration instantiates the whole body.** Each iteration creates one **instance** of every body Step, with its
own Tasks, states, and outputs, and runs the body as a small step graph. Iteration `N + 1` starts only after every
body Step instance of iteration `N` has succeeded or been skipped. Iterations never overlap.

**The back edge is `next`, not `dependsOn`.** The step graph, the body included, has no cycle. The "circular"
dependency of a loop is the evaluation of `next`, `until`, and `while` at the **iteration boundary**, between two
iterations, where expressions read the outputs of the iteration that just finished and produce the values the next
one starts with.

**State flows only through loop variables.** A body Step reads the loop variables, its own dependencies within the
same iteration, and the Steps the loop depends on. It never reads an earlier iteration's Steps. Whatever one
iteration hands to the next is declared, typed, and visible as a loop variable.

**Fixed graph, late-bound count.** The template's Steps, including the body's, are fixed and validated before
submission, as RFC 0011 requires. What is known only at run time is the number of iterations, within the declared
bound, and through each iteration's body Steps, the Task counts RFC 0011 already late-binds.

### Evaluation stages

A loop adds two evaluation stages to those of
[RFC 0011 §Evaluation stages](0011-control-flow.md#evaluation-stages), both after the loop is bound:

```
  Step Input Binding        Iteration Start            Body Step binding and         Iteration Boundary
  (of the loop)             (iteration N)              Task Execution (iteration N)  (after iteration N)
  ──────────────────        ──────────────────         ──────────────────────────    ────────────────────────────
  enableIf, then            Loop.* CONCRETE for N;     as RFC 0011, per body Step    loop outputs for N,
  maxIterations, then       a fresh instance of        instance, with Loop.*         then until, then next
  every initial, then       every body Step;           concrete                      (all read N's values), then
  the first while           groups whose inputs are                                  while (reads N + 1's values),
                            concrete are evaluated                                   then the bound
```

### Loop states

A loop has the Step states of [RFC 0011 §Step states](0011-control-flow.md#step-states):

```
                 job creation
                       │
      ┌────────────────┼──────────────────────┐
      ▼                ▼                      ▼
   SKIPPED      PENDING_INPUTS ──bound──► READY ──► RUNNING ──loop ends──┬──every minLength holds──► SUCCEEDED
                       │                    │         │      ▲           │
                       │                    │   fails │      │ (1)       ├──a minLength violated──► FAILED
                       │                    │         ▼      │           │
                       │                    │        FAILED ─┘           │
                       │                    └── first while false ───────┘
                       │                        (zero iterations)
                       ├──enableIf false──► SKIPPED
                       ├──a group fails ──► FAILED, at once
                       └──a dependency FAILED ──► (blocked; never runs)

  (1) every failed body Step instance of the iteration in progress has succeeded after a retry or requeue of its
      Tasks
```

A loop is `RUNNING` while its iterations run. When *until* or *while* ends it, or its first *while* is `false`, the
`minLength` of each `APPEND` loop output is checked: the loop is `SUCCEEDED` if every one holds and `FAILED` if one does
not. A loop is also `FAILED` when a body Step instance of the iteration in progress fails, when one of its own groups or
an evaluation at an iteration boundary fails, when it exceeds a limit, or when it reaches *maxIterations* while
continuing (see [Failure](#failure)). Only a failure of the Tasks of a body Step instance of the iteration in progress
is recoverable. A loop whose failure is recoverable is `FAILED` until every failed body Step instance of the iteration
in progress has succeeded after a retry or requeue of its Tasks; it then returns to `RUNNING`, and the iteration
continues to its boundary, edge (1). Every other loop failure is final (see [Failure](#failure)).

## Basic Examples

These examples use the conventions of [RFC 0011 §Basic Examples](0011-control-flow.md#basic-examples). Application
command lines and Python modules (`render`, `qc`, `solver`, `pipeline`) are illustrative.

| Example | Body | Condition | Carried state | Loop outputs | After the loop |
|---|---|---|---|---|---|
| [Render, check, and re-render until clean](#render-check-and-re-render-until-clean) | Two Steps, one a fan-out | `until` | `INT`, `LIST[INT]` | `LAST`, `APPEND` | A Step and job outputs |
| [Iterative solver](#iterative-solver) | One Step | `until` | `FLOAT` | `LAST`, `APPEND` | Job outputs |
| [Paginated listing, then fan out](#paginated-listing-then-fan-out) | One Step | `while` | `STRING` | `APPEND` (`LIST[PATH]`, `LIST[INT]`) | A fan-out Step and a job output |

### Render, check, and re-render until clean

A render is checked for noise, and the frames that fail are rendered again with twice the samples until none fails.
The body has two Steps, the first of which fans out over the frames still to do. This is the case that the loop
sketch in earlier drafts of RFC 0011 did not cover.

```yaml
specificationVersion: jobtemplate-2023-09
extensions: [EXPR, CONTROL_FLOW, FEATURE_BUNDLE_1]
name: "Render {{path(RawParam.SceneFile).stem}} until clean"
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
  - name: OutputVideo
    type: PATH
    objectType: FILE
    dataFlow: OUT
    default: "out/beauty.mp4"
  - name: StartSamples
    type: INT
    default: 64
    minValue: 1
  - name: NoiseThreshold
    type: FLOAT
    default: 0.02
  - name: MaxPasses
    type: INT
    default: 6
    minValue: 1
    maxValue: 20
jobOutputDefinitions:
  - name: Passes
    type: INT
    value: "{{len(Steps.Refine.Output.NoisyFrameCounts)}}"
  - name: FinalSamples
    type: INT
    value: "{{Steps.Refine.Output.FinalSamples}}"
steps:
  - name: Refine
    loop:
      variables:
        - name: Samples
          type: INT
          initial: "{{Param.StartSamples}}"
          next: "{{Loop.Samples * 2}}"
        - name: Frames
          type: LIST[INT]
          minLength: 1
          initial: "{{list(Param.Frames)}}"
          next: "{{Steps.CheckPass.Output.NoisyFrames}}"
      until: "{{len(Steps.CheckPass.Output.NoisyFrames) == 0}}"
      maxIterations: "{{Param.MaxPasses}}"
      steps:
        - name: RenderPass
          parameterSpace:
            taskParameterDefinitions:
              - name: Frame
                type: INT
                range: "{{Loop.Frames}}"
          bash:
            script: |
              render --scene {{repr_sh(Param.SceneFile)}} --frame {{Task.Param.Frame}} \
                     --samples {{Loop.Samples}} \
                     --out {{repr_sh(Param.OutputPattern.with_number(Task.Param.Frame))}}

        - name: CheckPass
          dependencies:
            - dependsOn: RenderPass
          outputVariables:
            - name: NoisyFrames
              type: LIST[INT]
            - name: MeanNoise
              type: FLOAT
              minValue: 0
          python:
            script: |
              import json
              from qc import check_frames
              report = check_frames({{repr_py(Param.OutputPattern)}}, {{repr_py(Loop.Frames)}},
                                    threshold={{Param.NoiseThreshold}})
              print("openjd_output_var: NoisyFrames=" + json.dumps(report.noisy_frames))
              print(f"openjd_output_var: MeanNoise={report.mean_noise}")
    outputVariables:
      - name: FinalSamples
        type: INT
        collect: LAST
        value: "{{Loop.Samples}}"
      - name: NoisyFrameCounts
        type: LIST[INT]
        collect: APPEND
        value: "{{len(Steps.CheckPass.Output.NoisyFrames)}}"
      - name: MeanNoise
        type: LIST[FLOAT]
        collect: APPEND
        value: "{{Steps.CheckPass.Output.MeanNoise}}"

  - name: EncodeVideo
    dependencies:
      - dependsOn: Refine
    bash:
      script: |
        ffmpeg -nostdin -start_number {{min(Param.Frames)}} -i {{repr_sh(Param.OutputPattern)}} \
               -metadata comment={{repr_sh('max samples ' + string(Steps.Refine.Output.FinalSamples))}} \
               {{repr_sh(Param.OutputVideo)}}
```

* Iteration 0 renders every frame with `StartSamples`. Each later iteration renders only the frames that the
  iteration before it found noisy, because `Frames` is set from `NoisyFrames` at the boundary, and with twice the
  samples. `CheckPass` checks only the frames rendered in its own iteration.
* `until` is checked before `next`, so the loop ends while `Frames` still holds the last frames rendered.
  `minLength: 1` on `Frames` is never violated: `next` runs only when `NoisyFrames` is not empty.
* If frames are still noisy after `MaxPasses` iterations, `Refine` fails, and `EncodeVideo` never runs. A loop that
  should accept the result instead says so in its condition, for example
  `until: "{{len(Steps.CheckPass.Output.NoisyFrames) == 0 or Loop.Iteration + 1 >= Param.MaxPasses}}"`.
* `EncodeVideo` reads only `Refine`'s declared outputs. `FinalSamples` has type `int`, not `int?`, because an `until`
  loop runs at least once and `Refine` has no `enableIf`.
* `FinalSamples` is the value of `Loop.Samples` in the last iteration, the samples that iteration rendered with.

The scheduler's view of the Job over time:

```
Refine           READY ─► RUNNING ──────────────────────────────────────────────────► SUCCEEDED (3 iterations)
  [0] RenderPass   240 Tasks, Samples=64  ─► SUCCEEDED
  [0] CheckPass    ─► SUCCEEDED (NoisyFrames=[12, 97, 98])      boundary: Samples=128, Frames=[12, 97, 98]
  [1] RenderPass   3 Tasks, Samples=128   ─► SUCCEEDED
  [1] CheckPass    ─► SUCCEEDED (NoisyFrames=[97])              boundary: Samples=256, Frames=[97]
  [2] RenderPass   1 Task, Samples=256    ─► SUCCEEDED
  [2] CheckPass    ─► SUCCEEDED (NoisyFrames=[])                boundary: until is true
EncodeVideo      READY (gated on Refine) ──────────────────────────────────────────────────────────► RUNNING ─► ...
```

### Iterative solver

A solver steps a relaxation until its residual is below a tolerance. The body is one Step with one Task, and the
loop carries the residual and a damping factor that it halves whenever a step makes the residual worse. Bulk state,
the mesh being relaxed, stays in a file, and only control data flows through the loop.

```yaml
specificationVersion: jobtemplate-2023-09
extensions: [EXPR, CONTROL_FLOW, FEATURE_BUNDLE_1]
name: "Relax {{path(RawParam.Mesh).stem}}"
parameterDefinitions:
  - name: Mesh
    type: PATH
    objectType: FILE
    dataFlow: IN
  - name: WorkDir
    type: PATH
    objectType: DIRECTORY
    dataFlow: OUT
  - name: Tolerance
    type: FLOAT
    default: 0.000001
    minValue: 0
  - name: StartDamping
    type: FLOAT
    default: 0.8
jobOutputDefinitions:
  - name: Iterations
    type: INT
    value: "{{len(Steps.Relax.Output.ResidualHistory)}}"
  - name: FinalResidual
    type: FLOAT
    value: "{{Steps.Relax.Output.FinalResidual}}"
steps:
  - name: Prepare
    outputVariables:
      - name: Residual
        type: FLOAT
        minValue: 0
    python:
      script: |
        from solver import prepare
        residual = prepare({{repr_py(Param.Mesh)}}, state={{repr_py(Param.WorkDir / 'state.npz')}})
        print(f"openjd_output_var: Residual={residual}")

  - name: Relax
    dependencies:
      - dependsOn: Prepare
    loop:
      variables:
        - name: Residual
          type: FLOAT
          initial: "{{Steps.Prepare.Output.Residual}}"
          next: "{{min(Loop.Residual, Steps.RelaxStep.Output.Residual)}}"
        - name: Damping
          type: FLOAT
          minValue: 0.001       # the loop fails if the damping collapses
          initial: "{{Param.StartDamping}}"
          next: "{{Loop.Damping * 0.5 if Steps.RelaxStep.Output.Residual > Loop.Residual else Loop.Damping}}"
      until: "{{Steps.RelaxStep.Output.Residual < Param.Tolerance}}"
      maxIterations: 200
      steps:
        - name: RelaxStep
          outputVariables:
            - name: Residual
              type: FLOAT
              minValue: 0
          python:
            script: |
              from solver import relax_step
              # relax_step keeps the state file at its best so far, so a rejected step changes nothing.
              residual = relax_step({{repr_py(Param.WorkDir / 'state.npz')}}, damping={{Loop.Damping}},
                                    best={{Loop.Residual}})
              print(f"openjd_output_var: Residual={residual}")
    outputVariables:
      - name: FinalResidual
        type: FLOAT
        collect: LAST
        value: "{{Steps.RelaxStep.Output.Residual}}"
      - name: ResidualHistory
        type: LIST[FLOAT]
        collect: APPEND
        value: "{{Steps.RelaxStep.Output.Residual}}"
```

* Both `next` expressions read the values the iteration ran with: `Damping`'s `next` compares the new residual with
  `Loop.Residual` before `Residual`'s own `next` has replaced it.
* A loop whose body is one single-Task Step is the single-Step loop of earlier drafts. Its history is
  `ResidualHistory`, one element per iteration in order.
* The job outputs show the number of iterations and the final residual when the loop succeeds. If the residual is
  still above the tolerance after 200 iterations, or `Damping` falls below its `minValue`, the loop fails with a
  message that names the bound or the constraint.

### Paginated listing, then fan out

A bucket listing is read one page of up to 1,000 keys at a time, following S3's continuation token, and each page is
then processed by its own Task. This replaces the single `Discover` Task of
[RFC 0011 §S3 prefix fan-out with a manifest](0011-control-flow.md#s3-prefix-fan-out-with-a-manifest) when a prefix
holds more keys than one output value should, and it shows a `while` loop with a `STRING` loop variable.

```yaml
specificationVersion: jobtemplate-2023-09
extensions: [EXPR, CONTROL_FLOW, FEATURE_BUNDLE_1]
name: "Process s3://{{Param.Bucket}}/{{Param.Prefix}}"
parameterDefinitions:
  - name: Bucket
    type: STRING
  - name: Prefix
    type: STRING
  - name: WorkDir
    type: PATH
    objectType: DIRECTORY
    dataFlow: OUT
jobOutputDefinitions:
  - name: ObjectCount
    type: INT
    value: "{{sum(Steps.ListObjects.Output.PageSizes)}}"
steps:
  - name: ListObjects
    loop:
      variables:
        - name: Token
          type: STRING
          initial: "{{''}}"
          next: "{{Steps.ListPage.Output.NextToken}}"
      # The first request has no token, and the last page returns none.
      while: "{{Loop.Iteration == 0 or Loop.Token != ''}}"
      maxIterations: 1000
      steps:
        - name: ListPage
          outputVariables:
            - name: Manifest
              type: PATH
              objectType: FILE
            - name: Count
              type: INT
              minValue: 0
              maxValue: 1000
            - name: NextToken
              type: STRING
          python:
            script: |
              import boto3, json, os
              request = {"Bucket": {{repr_py(Param.Bucket)}}, "Prefix": {{repr_py(Param.Prefix)}}, "MaxKeys": 1000}
              token = {{repr_py(Loop.Token)}}
              if token:
                  request["ContinuationToken"] = token
              page = boto3.client("s3").list_objects_v2(**request)
              keys = [o["Key"] for o in page.get("Contents", [])]
              manifest = {{repr_py((Param.WorkDir / 'pages' / 'page_%04d.json').with_number(Loop.Iteration))}}
              os.makedirs(os.path.dirname(manifest), exist_ok=True)
              with open(manifest, "w", encoding="utf-8") as f:
                  json.dump(keys, f)
              print("openjd_output_var: Manifest=" + json.dumps(manifest))
              print(f"openjd_output_var: Count={len(keys)}")
              print("openjd_output_var: NextToken=" + json.dumps(page.get("NextContinuationToken", "")))
    outputVariables:
      - name: Manifests
        type: LIST[PATH]
        objectType: FILE
        collect: APPEND
        value: "{{Steps.ListPage.RawOutput.Manifest}}"
      - name: PageSizes
        type: LIST[INT]
        collect: APPEND
        value: "{{Steps.ListPage.Output.Count}}"

  - name: ProcessPage
    dependencies:
      - dependsOn: ListObjects
    parameterSpace:
      taskParameterDefinitions:
        - name: Manifest
          type: PATH
          range: "{{Steps.ListObjects.RawOutput.Manifests}}"
    python:
      script: |
        import json
        from pipeline import process_key
        with open({{repr_py(Task.Param.Manifest)}}, encoding="utf-8") as f:
            keys = json.load(f)
        for key in keys:
            process_key({{repr_py(Param.Bucket)}}, key)
```

* `while` is checked before each iteration, including the first, and reads only loop variables, `Loop.Iteration`,
  and the Steps the loop depends on. `Loop.Iteration == 0` lets the first request run with no token. At each
  boundary `next` sets `Token` from the page just listed, and `while` then reads the new value.
* `Manifests` collects one `PATH` per iteration. A loop output's *value* is evaluated by the scheduler, so it reads
  the stored, submission-context form, `RawOutput`, as a job output does. `ProcessPage` builds its `PATH` Task
  parameter from the same form, and each Task receives its manifest's path mapped for its own worker.
* `ProcessPage` has one Task per page, so a prefix with 3,500 keys runs four `ListPage` instances and four
  `ProcessPage` Tasks. An empty prefix runs one `ListPage` instance, which reports `Count` 0, and one `ProcessPage`
  Task over an empty manifest.
* The scheduler shows each iteration's `ListPage` instance with its outputs as the loop runs, and `ProcessPage`'s Task
  count becomes known when the loop succeeds.

## Motivation

[RFC 0011](0011-control-flow.md) makes a Job's width depend on data produced while it runs: how many Tasks a Step
has, and whether it runs. It keeps the step graph fixed, so it can't express work whose amount depends on its own
results: work that runs, looks at what it produced, and decides whether to run again. RFC 0011 lists this as use
case U17. An early sketch covered it with a single Step whose Tasks ran one after another, each one's parameters
computed from the Task before it, and noted what that sketch could not do: repeat a group of Steps, such as render,
check, and re-render until clean.

The requirement that motivates this RFC is that group. Users need a set of Steps, each with its own parameter space,
host requirements, and environments, repeated as a unit until a condition holds, with what each repetition learns
passed to the next. The cases:

* **Render, check, re-render.** A render Step fans out over frames, a check Step finds the bad ones, and the frames
  that failed are rendered again with different settings until the check passes. The render and the check need
  different hosts, so they can't be one Step.
* **Converge.** A solver, an optimizer, or a training loop steps until a residual, a loss, or a validation score is
  good enough, and adapts a step size or learning rate from one iteration to the next. Some bodies are one Step;
  others alternate a simulation Step and an analysis Step.
* **Paginate.** An API that returns results a page at a time, with a continuation token, is read until the token runs
  out. One Step per page keeps each output value small.
* **Refine adaptively.** A coarse pass, an error estimate, and a refined pass over the regions with the largest
  error, repeated until the estimate is below a threshold.

### Workarounds in use today

* **Unrolling.** The template repeats the body a fixed number of times, as `Render1`, `Check1`, `Render2`,
  `Check2`, and so on, each copy disabled by `enableIf` once an earlier check passes. It works under RFC 0011 but
  makes the template as long as the largest number of passes it allows, every copy's `Steps.*` symbols optional, and
  the history scattered over differently named Steps.
* **Resubmission.** The last Step of a Job submits the next Job. It isn't portable across schedulers, needs
  credentials in the Task, and splits one piece of work over several Jobs that nothing ties together.
* **A loop inside one Task.** One long Task runs every pass itself. It can't fan a pass out over many hosts, can't
  use different hosts for different Steps of the body, and a scheduler sees one opaque Task.

### Why a group of Steps

A loop over one Step can't use different hosts or environments for different parts of the body, and can't fan out
inside the body, which the render, check, re-render case needs. A loop whose body is a group of Steps covers the
one-Step case as a body of one Step, so one construct serves both. Because the body is made of ordinary Step
templates, everything RFC 0011 and RFC 0012 define for a Step, such as parameter spaces computed at binding,
`enableIf`, outputs, chunking, and host requirements, works inside it without new rules.

### Backward compatibility

A loop is available only in a template that lists `CONTROL_FLOW`, which already requires a scheduler that implements
it to reject what it doesn't understand (see
[RFC 0011 §Backward compatibility](0011-control-flow.md#backward-compatibility)). Nothing in a template without a
`<StepLoop>` entry changes meaning.

## Technical Requirements

The requirements of [RFC 0011 §Technical Requirements](0011-control-flow.md#technical-requirements) apply. For loops
they mean:

1. **Fixed step graph (RFC 0011 item 3).** The body's Steps, their names, their dependencies, and their output types
   are in the template and validated before submission. The number of iterations is late-bound, as a Task count is.
2. **Bounded (RFC 0011 item 7).** Every loop declares a maximum number of iterations, within a limit of this
   specification, so its total work is bounded when the Job is created.
3. **Fail fast and at the cause (RFC 0011 items 4 and 5).** Every reference in a loop and its body is checked at
   template validation, and an error in a loop's own expressions fails the loop when it is evaluated, naming the
   expression.
4. **Deterministic (RFC 0011 item 6).** What an iteration sees does not depend on scheduling. Iterations run in order,
   never overlap, and read only declared state.
5. **Visible.** A scheduler can show each iteration, its Steps, its Tasks, and the values it ran with, and a reader can
   see from the template exactly what one iteration passes to the next.

## Design Choices

### 1. A loop is a Step-list entry that contains a sub-graph of Steps

The loop is one entry in `steps`, with the Steps it repeats nested under it. Which Steps repeat, where the loop
begins and ends, and what crosses its boundary are all visible in the template's structure. The rest of the step
graph treats the loop as one node, so the rule of the base specification that Step dependencies form an acyclic graph
holds unchanged, at the top level and inside the body, and a scheduler that orders Steps by their dependencies needs
no notion of a back edge.

The alternatives are in [Rejected Ideas](#rejected-ideas): cyclic `dependsOn` edges between Steps, the single-Step
loop of RFC 0011's earlier sketch, Tasks that add Steps, and unbounded loops.

### 2. The back edge is the loop variables' `next`

The "circular dependency" that a loop needs, a later iteration depending on an earlier one, is the evaluation of
`next`, `until`, and `while` at the iteration boundary. Those expressions read the outputs of the iteration that
just finished and produce the values the next one starts with. Making it an expression rather than an edge:

* **States the data that crosses.** A reader sees every value that one iteration passes to the next, with its type
  and constraints, in `variables`. A cyclic edge would let any Step of iteration `N + 1` read anything of iteration
  `N`, and leave the reader to find out what matters.
* **Keeps `Loop.*` concrete for an iteration.** Every body Step reads the same values for the whole iteration, so
  its parameter space, host requirements, and Step Environments can be bound when the iteration starts, as RFC 0011's
  progressive binding wants.
* **Uses the existing evaluator.** `next` is an EXPR expression over RFC 0011's `Steps.*` symbols, evaluated as a
  materialization expression is, by the scheduler, under EXPR's limits.

All `next` expressions read the values of the iteration that just finished, never each other's new values, so their
order in `variables` doesn't matter and a swap (`next: "{{Loop.B}}"` and `next: "{{Loop.A}}"`) works.

### 3. Iterations run one after another and instantiate the whole body

Each iteration creates a fresh instance of every body Step. An instance is an ordinary Step for every rule of
RFC 0011 and RFC 0012: it has its own states, its own binding, its own Tasks, and its own outputs. Iteration `N + 1`
starts only after every instance of iteration `N` has succeeded or been skipped and the loop has decided to go on.

Overlapping iterations, for example starting iteration `N + 1`'s render while iteration `N`'s check still runs, would
need the next iteration's inputs before the current one has produced them. Pipelining of that kind is a different
construct, and the dependent half of it is already `TASK_DEPENDENCIES`. A loop that wants parallelism puts it inside
the body, where any Step can fan out.

Instantiating the body afresh, rather than appending Tasks to the same Steps, keeps each iteration's outputs and
states separate, so `Steps.CheckPass.Output.NoisyFrames` always means this iteration's check, a scheduler can show
iterations side by side, and RFC 0011's rule that a producer's outputs are complete only when all of its Tasks have
succeeded needs no exception.

### 4. Exactly one of `while` and `until`

A loop declares either `while`, checked before each iteration including the first, or `until`, checked after each
iteration. `until` can read the iteration's body Steps directly, and its loop always runs at least once, so its
`LAST` outputs are never `null`. `while` reads only loop variables and the loop's dependencies, so it can be checked
before any body runs, and its loop may run zero times.

A loop must have one. A loop with neither would run until `maxIterations`, and because reaching the bound fails the
loop (see [Design Choice 5](#5-maxiterations-is-required-and-reaching-it-fails-the-loop)), it would always fail; if
instead reaching the bound succeeded for such a loop, a forgotten condition would silently run every loop to its
bound. A fixed number of iterations is written with a condition: `while: "{{Loop.Iteration < Param.Passes}}"`. A loop
with both would need an order between them and an answer to which one's zero-iteration rule applies, for no case
that one condition plus `and` can't express.

### 5. `maxIterations` is required, and reaching it fails the loop

Whether a loop terminates can't be checked statically, so every loop declares a bound, at most 1,000, that is known
when the Job is created (see [Limits](#limits)). This is how a loop meets
[RFC 0011 Technical Requirement 7](0011-control-flow.md#technical-requirements): the work of a Job with loops is
bounded at Job creation, and a scheduler can show the bound.

A loop that would start iteration `maxIterations` fails. The alternative, ending the loop quietly with the last
iteration's values, would hand the Steps after the loop a result that its own condition says isn't done, such as a
render that is still noisy or a solver that hasn't converged, as if it were. The condition is where an author says
what "done" means. A loop that should accept its result after a number of tries says so there, as in the first
example, and then the bound is a safety net that a correct template never reaches. Earlier drafts of RFC 0011
sketched the same rule for loop Steps.

### 6. Body Steps read the loop, their dependencies in the body, and the loop's dependencies

A body Step may read `Loop.*`, the `Steps.*` symbols of body Steps it lists in its own `dependencies` (the instances of
the same iteration), and the `Steps.*` symbols of the Steps the loop depends on. It may not read any other Step, nor
an earlier iteration's instances.

* **The loop's dependencies stand in for the body's.** A body Step's `dependencies` name only Steps in the same body,
  because a body Step can't start before its loop does, and its loop starts only after its own dependencies are
  satisfied. Requiring each body Step that reads `Steps.Setup.*` to name the loop's dependency again would be an
  edge that never changes scheduling. RFC 0011's rule that a Step reads only its direct dependencies is kept in
  spirit: the edge that orders the read is on the loop, one level up, where a reader looks for it.
* **No reads of earlier iterations.** A body that could read `Steps.CheckPass` of iteration `N - 1` would have a
  second, undeclared channel between iterations, and which iteration a reference meant would depend on context. State
  that must cross goes through a loop variable, where it is declared and typed.

The loop's own boundary expressions, `next`, `until`, and the loop outputs' *value*, may read any body Step of the
iteration that just finished, not only the ones without dependents, because the boundary waits for the whole body.

### 7. Steps after a loop read only its declared outputs

A Step after the loop reads `Steps.<LoopName>.Output.<VarName>`, and never a body Step. A body Step has one instance
per iteration, so `Steps.CheckPass.Output.Score` outside the loop would have no single meaning: the last iteration's,
a list over every iteration, or whichever iteration ran it last. Loop outputs make the choice explicit, with a type
and a collect mode, and leave the body's Steps free to change without breaking the Steps after it. Job outputs follow
the same rule. This is the same choice RFC 0011 makes for Step outputs in general, a declared interface rather than
access to internals, applied to the loop as a producer.

### 8. Loop outputs are evaluated after each iteration and collected as `LAST` or `APPEND`

A loop output has a *value* expression, evaluated at each iteration boundary, and a `collect` mode from RFC 0012 that
says how the values combine: `LAST` keeps the final iteration's value, and `APPEND` keeps one value per iteration, in
order. Evaluating the value every iteration, rather than once at the end, lets a scheduler show the history as the
loop runs, checks each value against the output's constraints where it is produced, and keeps `APPEND`'s list in
iteration order, which a final evaluation could not reconstruct.

`collect` is required, and `ONCE` is not allowed. Under RFC 0012, `ONCE` means exactly one report; for a loop that
would mean "exactly one iteration produces a value", which is rarely what an author wants and leaves open what a
second iteration does. Requiring `LAST` or `APPEND` makes a reader see at once whether a value is the final one or a
history.

A loop with zero iterations, which only a `while` loop can have, gives `null` for a `LAST` output and `[]` for an
`APPEND` output, as RFC 0011 gives `null` for a skipped Step and `[]` for an empty fan-out. Its `LAST` outputs are
therefore typed `T?`, and an `until` loop's `T`.

### 9. `Loop.Iteration` counts from 0

`Loop.Iteration` is the index of the current iteration, starting at 0. Expression lists, `range`, and the lists of
RFC 0011 are indexed from 0, and element `i` of an `APPEND` loop output belongs to iteration `i`, as element `i` of a
multi-Task producer's output belongs to its `i`-th Task. A 1-based counter reads better in a file name or a log
(`pass 1`) and is one loop variable away, as in the example in [A loop at a glance](#a-loop-at-a-glance), or
`Loop.Iteration + 1` in place.

### 10. Step names are unique across the whole template

Every Step of the template, at the top level or in a loop's body, and every loop, has a name that differs from all
the others. `Steps.<Name>` then always names one Step, a log or an error message names a Step unambiguously, and a
scheduler can key its records by name. Scoping body Step names to their loop would let two loops each have a
`Render`, at the cost of a rule for what `Steps.Render` means in a loop's expressions versus outside it, and of
qualified names everywhere else. An author who wants the same body in two loops can prefix the names.

### 11. A separate `<StepLoop>` element

The `steps` list accepts two kinds of entry: a `<StepTemplate>`, as today, and a `<StepLoop>`, recognized by its
`loop` property. A loop could instead be a `<StepTemplate>` with an optional `loop` field that excludes `script`,
`parameterSpace`, and the other fields that only a Step with Tasks has. A separate element:

* **Validates simply.** In JSON Schema, a `<StepLoop>` is an object with a required `loop` and no other properties
  than its own, and a `<StepTemplate>` is unchanged apart from not allowing `loop`. A `loop` field on
  `<StepTemplate>` would need a list of properties that it excludes, kept up to date as later extensions add
  properties to Steps.
* **Reads as what it is.** A loop has no script, Tasks, host requirements, or Step Environments, and its
  `outputVariables` have a required *value* that a Step's do not. Separate elements document separate shapes.

The two share `name`, `description`, `enableIf`, and `dependencies`, with the same meaning, so a tool that walks the
step graph treats both alike.

### 12. Loops are part of `CONTROL_FLOW`

RFC 0011's sketch proposed a separate `STEP_LOOPS` extension. This RFC puts loops in `CONTROL_FLOW`, for the reasons
[RFC 0011 Design Choice 1](0011-control-flow.md#1-variable-data-flow-is-part-of-control_flow-not-a-standalone-extension)
gives for one extension: a loop needs every other part of `CONTROL_FLOW`, its body Steps are ordinary
`CONTROL_FLOW` Steps, and splitting the name would freeze one half. The cost is that a scheduler must implement loops
to advertise `CONTROL_FLOW`, which makes the extension larger still; [Open Questions](#open-questions) item 4 asks
whether that cost is worth a separate name.

### 13. Design tenets

* **Portable.** A loop is defined in terms of Steps, which every scheduler already runs, plus one evaluation point.
  A scheduler can implement it by creating each iteration's Steps when the iteration starts. The tension is the
  added work for schedulers whose Jobs have a fixed set of Steps once created.
* **Expressive.** One construct covers loops of one Step and of many, `while` and `until`, carried state of any output
  type, and histories. The fixed graph rules out loops that change their body or nest, which are Future Work.
* **Human readable and writable.** The body is written as ordinary Steps, and what crosses an iteration is listed in
  `variables`. The tensions: a `while` loop that must run at least once needs `Loop.Iteration == 0 or ...`, and a
  1-based counter needs a variable.
* **Tooling parseable.** The loop, its body, its state, its bound, and its outputs are declared and checked before
  submission, and the bound is known at Job creation. The tension is that a Job's total Task count is not known until
  its loops end, which RFC 0011 already accepts for computed parameter spaces; a scheduler reports each iteration as
  it runs.

## Specification

Every field, symbol, and behavior in this section is part of the `CONTROL_FLOW` extension (see
[RFC 0011 §Extension name and dependencies](0011-control-flow.md#extension-name-and-dependencies)), and is
available only in a Job Template whose `extensions` list includes `CONTROL_FLOW`.

### Terminology

The terms of [RFC 0011 §Terminology](0011-control-flow.md#terminology) and
[RFC 0012 §Terminology](0012-step-output-variables.md#terminology) apply. This RFC adds:

| Term | Definition |
|------|------------|
| **Loop** | An entry of the Job Template's `steps` list that is a `<StepLoop>`. It is a node of the step graph, with a name, dependencies, an optional `enableIf`, and the states of a Step, and has no Tasks of its own. |
| **Body** | The Steps listed in a loop's `loop.steps`. Each is a **body Step**. |
| **Top-level Step** | A Step or loop listed directly in the Job Template's `steps`, not in a body. |
| **Iteration** | One run of a loop's body. Iterations are numbered from 0 in the order they run. |
| **Body Step instance** | The Step that a body Step becomes in one iteration, with its own states, binding, Tasks, and outputs. It is identified by its loop, its iteration index, and its body Step's name, written `<LoopName>[<Iteration>].<BodyStepName>` in this RFC. |
| **Loop variable** | A typed value declared in a loop's `variables`, set by `initial` before the first iteration and by `next` at each iteration boundary, and read as `Loop.<Name>`. |
| **Loop output** | A typed value declared in a loop's `outputVariables`, evaluated at each iteration boundary, collected by `LAST` or `APPEND`, and read after the loop as `Steps.<LoopName>.Output.<Name>`. |
| **Iteration boundary** | The evaluation point after every body Step instance of an iteration has succeeded or been skipped, at which the loop outputs, `until`, `next`, `while`, and the bound are evaluated. |

### Schema modifications

> Changes to [the template schema](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas).

#### Job Template

> A modification to [`1.1. Job Template`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#11-job-template)

```diff
- steps: [<StepTemplate>, ...]
+ steps: [ <StepTemplate> | <StepLoop>, ... ]
```

Changed semantics of *steps*: with `CONTROL_FLOW`, an element that has a `loop` property is a
[`<StepLoop>`](#steploop), and every other element is a `<StepTemplate>`. Constraints:

1. The names of all Steps of the template, top-level Steps, loops, and the body Steps of every loop, are distinct.
   This replaces the rule of §3, item 1, that each Step's name differs from those of the other Steps.
2. A top-level Step's `dependencies` name only top-level Steps.

#### `<StepLoop>`

A `<StepLoop>` is the object:

```yaml
name: <StepName>
description: <Description> # @optional
enableIf: <bool> | <FormatString> # @optional @fmtstring
dependencies: [ <StepDependency>, ... ] # @optional
loop: <LoopDefinition>
outputVariables: [ <LoopOutputVariableDefinition>, ... ] # @optional
```

Where:

1. *name* — The loop's name, by which other Steps depend on it and read its outputs as `Steps.<name>.*`. A
   `<StepName>`, unique among all Steps of the template.
2. *description* — As on `<StepTemplate>`.
3. *enableIf* — Whether the loop runs, with the meaning, scope, and constraints of a `<StepTemplate>`'s *enableIf*
   (see [RFC 0011 §`<StepTemplate>`](0011-control-flow.md#steptemplate)), where `Step.Name` is the loop's name. If it
   is `false`, the loop is `SKIPPED`: no iteration runs, none of its other fields is evaluated, and its
   `Steps.<name>.*` symbols are `null`.
4. *dependencies* — The top-level Steps the loop depends on. The loop's first iteration starts only when every one is
   satisfied, and the Steps listed here are those whose `Steps.*` symbols the loop's expressions and its body Steps
   may read (see [`Steps.*` in and around a loop](#steps-in-and-around-a-loop)). Constraints and satisfaction as for a
   `<StepTemplate>` (see [RFC 0011 §`<StepDependency>`](0011-control-flow.md#stepdependency)).
5. *loop* — The loop's variables, condition, bound, and body. See [`<LoopDefinition>`](#loopdefinition).
6. *outputVariables* — The loop's outputs. See
   [`<LoopOutputVariableDefinition>`](#loopoutputvariabledefinition). Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. Maximum number of elements: 32.
    3. No two elements in this list may have the same value for the `name` property.

A `<StepLoop>` has no other properties: in particular no `script`, `parameterSpace`, `hostRequirements`,
`stepEnvironments`, or `let`.

Where the base specification and RFC 0011 speak of a Step as a node of the step graph, by its name, its
`dependencies`, its *enableIf*, its state, or the `dependsOn` of another Step, a loop is a Step. It has no Tasks,
Task parameters, or Sessions of its own.

##### `<LoopDefinition>`

A `<LoopDefinition>` is the object:

```yaml
variables: [ <LoopVariableDefinition>, ... ] # @optional
while: <bool> | <FormatString> # @optional @fmtstring
until: <bool> | <FormatString> # @optional @fmtstring
maxIterations: <int> | <FormatString> # @fmtstring
steps: [ <StepTemplate>, ... ]
```

Where:

1. *variables* — The loop variables. See [`<LoopVariableDefinition>`](#loopvariabledefinition). Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. Maximum number of elements: 32.
    3. No two elements in this list may have the same value for the `name` property.
2. *while* — Whether to run another iteration, checked before each iteration, including the first. Its target type is
   `bool`, and a `null` result is an error. It may reference `Param.*` (except `PATH` and `LIST[PATH]` parameters),
   `RawParam.*`, `Job.Name`, `Loop.*` and `RawLoop.*` (with values whose type is or contains `path` only through
   `RawLoop.*`), and the `Steps.*` symbols of the loop's dependencies (see
   [`Steps.*` in and around a loop](#steps-in-and-around-a-loop)). It may not reference the body's Steps, `Step.Name`,
   `Task.*`, `Session.*`, or names bound by `let`.
3. *until* — Whether to stop, checked after each iteration. Its target type is `bool`, and a `null` result is an
   error. It may reference everything *while* may, and the `Steps.*` symbols of every body Step of the loop.
4. *maxIterations* — The most iterations the loop may run. Its target type is `int`. It may reference only `Param.*`
   (except `PATH` and `LIST[PATH]` parameters), `RawParam.*`, and `Job.Name`, so it is known at Job creation. Its
   value must be from 1 to 1,000.
5. *steps* — The body: the Steps that run in each iteration. Constraints:
    1. Minimum number of elements: 1.
    2. Each element is a `<StepTemplate>`. A `<StepLoop>` in a body is an error: loops do not nest.
    3. Each body Step's `dependencies` name only Steps in the same body, and the body's dependency graph is acyclic.

Constraints:

1. Exactly one of *while* and *until* is present.
2. With *while*, the loop runs as long as *while* is `true` before an iteration. With *until*, the loop runs at least
   once and stops after the first iteration for which *until* is `true`. See
   [The iteration boundary](#the-iteration-boundary).

##### `<LoopVariableDefinition>`

A `<LoopVariableDefinition>` is the object:

```yaml
name: <Identifier>
type: <OutputVariableType>
initial: <FormatString> # @fmtstring
next: <FormatString> # @fmtstring
description: <Description> # @optional
# Constraint properties, @optional, as on <OutputVariableDefinition>.
```

Where:

1. *name* — The name by which the variable is read, as `Loop.<name>` and `RawLoop.<name>`. It may not be
   `Iteration`.
2. *type* — The variable's type, from the vocabulary of
   [RFC 0012 §`<OutputVariableDefinition>`](0012-step-output-variables.md#outputvariabledefinition), with the same
   mapping to expression types and the same constraint properties. `LIST[LIST[INT]]` is allowed.
3. *initial* — The value for iteration 0, as a format string whose target type is the expression type of *type*. A
   `null` result is an error. It may reference `Param.*` (except `PATH` and `LIST[PATH]` parameters), `RawParam.*`,
   `Job.Name`, and the `Steps.*` symbols of the loop's dependencies. It may not reference `Loop.*`, `RawLoop.*`, the
   body's Steps, `Step.Name`, `Task.*`, `Session.*`, or names bound by `let`.
4. *next* — The value for the following iteration, evaluated at each iteration boundary at which *until*, if
   present, did not end the loop, before *while* and the bound. So a `while` loop evaluates *next* at its final
   boundary too, and an error or constraint violation there fails the loop. Its target type and `null` rule are those
   of *initial*. It may reference everything *until* may.
5. *description* — For display only.
6. Constraint properties — Checked against every value the variable takes, from *initial* and from each *next*. A
   violation fails the loop.

A loop variable's value is converted to its type as a job output's value is (see
[RFC 0011 §`<JobOutputDefinition>`](0011-control-flow.md#joboutputdefinition) item 3), except that the target type is
not made optional, so a `null` result is an error. A `PATH` or `LIST[PATH]` variable's expressions read `PATH` values in
the submission path context, through `RawParam.*`, `RawOutput`, `TaskRawParam`, and `RawLoop.*`, so its value is in the
submission path context, and `Loop.<name>` maps it forward in the consuming Session.

##### `<LoopOutputVariableDefinition>`

A `<LoopOutputVariableDefinition>` is the object:

```yaml
name: <Identifier>
type: <OutputVariableType>
collect: <LoopOutputCollectMode>
value: <FormatString> # @fmtstring
description: <Description> # @optional
unit: <OutputVariableUnit> # @optional
# Constraint properties, @optional, as on <OutputVariableDefinition>.
```

```bnf
<LoopOutputCollectMode> ::= "LAST" | "APPEND"
```

Where:

1. *name* — The name by which the output is read, as `Steps.<LoopName>.Output.<name>`.
2. *type* — As on [RFC 0012 §`<OutputVariableDefinition>`](0012-step-output-variables.md#outputvariabledefinition),
   with the same mapping to expression types and constraint properties. `LIST[LIST[INT]]` is allowed.
3. *collect* — Required. How the values of the iterations become the output's value (see
   [Loop outputs](#loop-outputs)):
    * `LAST` — The value of the last iteration.
    * `APPEND` — The *type* must be a `LIST[...]` type. Each iteration supplies one element, and the value is the list
      of them in iteration order.
4. *value* — The iteration's value, as a format string evaluated at each iteration boundary. Its target type is the
   expression type of *type*, or under `APPEND` of its element type. A `null` result is an error. It may reference
   everything *until* may.
5. *description* and *unit* — For display, as on `<OutputVariableDefinition>`.
6. Constraint properties — Under `LAST`, checked against each iteration's value. Under `APPEND`, `item` constraints
   and `maxLength` are checked as each element is added, and `minLength` when *until* or *while* ends the loop, before
   it succeeds, including after zero iterations; `minLength` is not checked when the loop is skipped. A violation fails
   the loop.

A `PATH` value, read through the `Raw` forms, is in the submission path context, as for a loop variable.

#### Format strings

##### The `Loop.*` scope

> A modification to [`7.3.1. Value References`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#731-value-references)

CONTROL_FLOW adds three value references for loops:

| Value | Type | Description |
|---|---|---|
| `Loop.<VarName>` | The expression type of the variable's *type* | The value of loop variable `<VarName>` for the current iteration. A `PATH` value has the consuming Session's path mapping rules applied. |
| `RawLoop.<VarName>` | As `Loop.<VarName>` | The same value with no path mapping applied: a `PATH` value in the submission path context. For other types, identical to `Loop.<VarName>`. |
| `Loop.Iteration` | `int` | The index of the current iteration, from 0. |

**Where available.** For loop `L`:

1. In a body Step of `L`, in the fields where
   [RFC 0011 §The `Steps.*` scope](0011-control-flow.md#the-steps-scope) *Where available* rule 3 allows `Steps.*`:
   its *enableIf*, *let*, *hostRequirements*, *parameterSpace* Task parameter `range`s and the other format strings of
   *parameterSpace* (not *combination*), *stepEnvironments* (any format string, including their *enableIf*), *script*,
   and, with `TASK_DEPENDENCIES`, the subspace ranges of its *dependencies*. There, the values are those of the
   iteration that the body Step instance belongs to.
2. In `L`'s *while*, where the values are those of the iteration about to start; and in `L`'s *until*, *next*, and
   loop output *value*s, where the values are those of the iteration that just finished. See
   [The iteration boundary](#the-iteration-boundary).
3. Nowhere else: not in `L`'s *enableIf*, *maxIterations*, or *initial*; not in another loop or its body; not in a
   top-level Step, a job output, a Job Environment, or an Environment Template.

A `Loop.<VarName>` whose type is or contains `path` is available only in format strings resolved on the worker host,
as a `PATH` `Param.*` is. Format strings resolved earlier, including all of `L`'s own expressions and every
materialization expression of a body Step, use `RawLoop.<VarName>` instead.

`<VarName>` must name a variable of `L`.

**Concreteness.** At template validation every `Loop.*` and `RawLoop.*` symbol is an `unresolved` value of its type.
For a body Step instance, every `Loop.*` symbol is concrete from the start of its iteration, so a body Step's group
whose other inputs are concrete then is evaluated when the iteration starts (see
[Iterations](#iterations)).

##### `Steps.*` in and around a loop

> A modification to [RFC 0011 §The `Steps.*` scope](0011-control-flow.md#the-steps-scope)

RFC 0011's rules for where a `Steps.<StepName>.*` symbol is available apply to top-level Steps unchanged, with these
additions and exceptions:

1. **In a body Step `S` of loop `L`**, rule 1 of *Where available* is replaced: the symbol is available if
   `<StepName>` is a body Step of `L` that `S` lists in its own `dependencies`, which reads that Step's instance in
   the same iteration, or a Step that `L` lists in its `dependencies`, which reads the same value in every
   iteration. Rules 2 and 3 apply as written.
2. **In `L`'s *enableIf*, *while*, and every *initial***, the symbol is available if `L` lists `<StepName>` in its
   `dependencies`.
3. **In `L`'s *until*, *next*, and loop output *value*s**, the symbol is available if `L` lists `<StepName>` in its
   `dependencies`, or `<StepName>` is any body Step of `L`, with or without dependents, which reads its instance in
   the iteration that just finished. These are not materialization expressions of a Step, and are evaluated at the
   iteration boundary.
4. **A body Step is never available outside its loop**: not in a top-level Step, another loop or its body, or a job
   output. A loop is available as a producer to the Steps that list it in their `dependencies`, and to job outputs, as
   any Step is.
5. **A loop's symbols.** `Steps.<L>.Output.<VarName>` and `Steps.<L>.RawOutput.<VarName>` name loop output
   `<VarName>`. `Steps.<L>.TaskParam.*` and `Steps.<L>.TaskRawParam.*` are not available: a loop has no Task
   parameters.

The types of a body Step's symbols are those RFC 0011 gives any Step's. The types of a loop's symbols, where `T` is
the expression type of a `LAST` output's *type* and `list[E]` that of an `APPEND` output's:

| Loop `L` | `LAST` output | `APPEND` output |
|---|---|---|
| has *until*, no *enableIf* | `T` | `list[E]` |
| has *while*, no *enableIf* | `T?`: `null` after zero iterations | `list[E]`: `[]` after zero iterations |
| has *enableIf* | `T?`: also `null` when skipped | `list[E]?`: `null` when skipped |

A loop's `Output` and `RawOutput` symbols become concrete when the loop is terminal and has not failed: `SUCCEEDED`,
with the collected values, or `SKIPPED`, with `null`. A body Step instance's symbols become concrete as RFC 0011
says for any Step, within its iteration.

##### Template processing stages

> A modification to [`7.4. Template Processing Stages`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#74-template-processing-stages)

| Stage | When | Known Values | Unknown Values | What Happens |
|---|---|---|---|---|
| **Template validation** | as today | as today | as today, and `Loop.*`, `RawLoop.*` | Every rule of [Validation](#validation) is checked, with `Loop.*` and `RawLoop.*` as `unresolved` values of their types. |
| **Job creation** | as today | as today | as today, and every body Step's values | A loop is a Step for RFC 0011's Job creation rules: its *enableIf* and groups whose inputs are concrete are evaluated (see [Loop binding](#loop-binding)). Nothing of a body Step is evaluated, and the `minLength` check of the loop's `APPEND` outputs is never part of Job creation. |
| **Step input binding** | as RFC 0011, for a loop and for each body Step instance | as RFC 0011, and for a body Step instance, `Loop.*` | as RFC 0011 | As RFC 0011. A body Step instance is bound within its iteration. |
| **Iteration start** (new, `CONTROL_FLOW` only) | When a loop starts iteration `N` | `Param.*`, `RawParam.*`, `Job.Name`, `Step.Name` of each body Step, `Loop.*` and `RawLoop.*` for `N`, the `Steps.*` of the loop's dependencies | the body Step instances' `Steps.*` values, `Task.*`, `Session.*` | An instance of every body Step is created; each instance's groups whose inputs are concrete are evaluated. See [Iterations](#iterations). |
| **Iteration boundary** (new, `CONTROL_FLOW` only) | When every body Step instance of iteration `N` has succeeded or been skipped | `Param.*`, `RawParam.*`, `Job.Name`, `Loop.*` and `RawLoop.*` for `N` (for *while*, for `N + 1`), the `Steps.*` of the loop's dependencies and of every body Step instance of `N` | `Task.*`, `Session.*` | Loop outputs, then *until*, then *next*, then *while*, then the bound. See [The iteration boundary](#the-iteration-boundary). |
| **Job output evaluation** | as RFC 0011 | as RFC 0011 | as RFC 0011 | A job output may read a loop's `Steps.*` symbols, and not a body Step's. |
| **Task execution** | as today | as today, and `Loop.*` for a body Step instance | *(none)* | As RFC 0011. |

### Running a loop

#### Loop binding

A loop is bound by RFC 0011's rules for a Step (see
[RFC 0011 §Step input binding](0011-control-flow.md#step-input-binding)), with these materialization groups, in this
order:

1. **Enablement:** *enableIf*.
2. **Bound:** *maxIterations*. Its inputs are concrete at Job creation, so it is evaluated as soon as *enableIf* is
   decided `true`. A value outside 1 to 1,000 is an error.
3. **Start:** each variable's *initial*, in declaration order, then, if the loop has *while*, *while* for iteration 0
   with `Loop.Iteration` 0 and the initial values. Each value is converted and checked against its variable's type and
   constraints.

An error in any group fails the loop at once, or fails Job creation if the group is evaluated then, as for a Step. A
loop is bound when *enableIf* is `true` and every group has evaluated, and then is `READY`. A bound loop whose first
*while* is `false` has zero iterations. The `minLength` check of its `APPEND` loop outputs is not a materialization
group and is never part of Job creation: it runs when the loop would succeed, which for a zero-iteration loop is once
its own dependencies are satisfied, as a Step with zero Tasks succeeds then under
[RFC 0011 §Step states](0011-control-flow.md#step-states). The `minLength` of each `APPEND` loop output is checked
against `[]`, and if every one holds the loop succeeds; a violation fails the loop. Any other bound loop starts
iteration 0 when its dependencies are satisfied, and is `RUNNING` until it succeeds or fails.

#### Iterations

When a loop starts iteration `N`:

1. Every `Loop.*` and `RawLoop.*` symbol takes its value for iteration `N`: each variable's value from *initial* (for
   `N = 0`) or from the last boundary's *next*, and `Loop.Iteration` is `N`.
2. An instance of every body Step is created. Each instance is a Step for every rule of RFC 0011 and RFC 0012: it has
   the states of a Step, is bound progressively, has its own Tasks and outputs, and its *enableIf*, when present, is
   decided first. An instance's `dependencies` are the instances of the same iteration of the body Steps it names,
   and are satisfied as RFC 0011 says. An instance with no dependencies is gated only by its own binding.
3. Each instance's groups whose inputs are concrete at the start of the iteration MUST be evaluated then, as a
   Step's groups whose inputs are concrete at Job creation are. An error in one fails the instance, and so the loop,
   finally (see [Failure](#failure)).
4. Iteration `N` is complete when every instance of it is `SUCCEEDED` or `SKIPPED`. The loop then evaluates the
   iteration boundary. A re-run of a Task that has already succeeded leaves its instance `SUCCEEDED` here, so the
   boundary does not wait for it (see
   [RFC 0012 §Retries and re-runs](0012-step-output-variables.md#retries-and-re-runs)).

No instance of iteration `N + 1` is created, and no Task of it runs, before the boundary of iteration `N` has decided
to continue. Iterations never overlap.

A scheduler identifies a body Step instance by its loop, its iteration index, and its body Step's name, and its Tasks
by that instance and their Task parameters, as for any Step. A scheduler or tool that shows a loop SHOULD show each
iteration with its index, the values of the loop variables it ran with, its body Step instances with their states
and Tasks, and the loop outputs evaluated at its boundary, and SHOULD show for the loop as a whole the iteration in
progress, the number of iterations completed, and *maxIterations*. A loop's Task count is not known until the loop
ends; a scheduler reports it as not yet known, with the Tasks of the iterations so far.

#### The iteration boundary

When iteration `N` is complete, the loop evaluates, in this order, with `Loop.*` holding the values of iteration `N`
and `Loop.Iteration` equal to `N` unless stated:

1. **Loop outputs.** Each loop output's *value*, converted and checked against its type and constraints, and recorded
   as iteration `N`'s value (see [Loop outputs](#loop-outputs)).
2. ***until***, if the loop has it. If it is `true`, the loop ends here; no *next* is evaluated. Then the
   `minLength` of each `APPEND` loop output is checked, and if every one holds the loop succeeds with `N + 1`
   iterations.
3. ***next*** of every variable. Every *next* reads the values of iteration `N`, not another variable's new value.
   Each result is converted and checked against its variable's type and constraints. The results are the values for
   iteration `N + 1`.
4. ***while***, if the loop has it, with `Loop.*` holding the new values and `Loop.Iteration` equal to `N + 1`. If it
   is `false`, the loop ends here. Then the `minLength` of each `APPEND` loop output is checked, and if every one
   holds the loop succeeds with `N + 1` iterations.
5. **The bound.** If `N + 1` equals *maxIterations*, the loop fails, with a message that names *maxIterations* and its
   value. Otherwise the loop starts iteration `N + 1`.

If an evaluation or check in steps 1 to 4 fails, including a `fail(...)` call, a `null` result, a value that does not
convert, and a violated constraint, the loop fails at once, with a message that names the field, the expression, the
iteration, and the problem. No later step is evaluated.

#### Loop outputs

A loop's outputs are evaluated at each iteration boundary, and collected by their *collect* mode:

| *collect* | Value after `n` iterations | After zero iterations | When the loop is `SKIPPED` |
|---|---|---|---|
| `LAST` | The value of iteration `n - 1` | `null` | `null` |
| `APPEND` | The list of the values of iterations `0` to `n - 1`, in order | `[]` | `null` |

The outputs are committed, and the loop's `Output` and `RawOutput` symbols become concrete, when the loop succeeds or
is skipped. A scheduler SHOULD show each iteration's values as they are evaluated, and MAY show a running loop's
outputs so far, marked as not yet committed, as it may a running Task's values under
[RFC 0012 §Committing values](0012-step-output-variables.md#committing-values). A loop that fails commits no outputs.
Loop outputs are stored and displayed, so they are not redacted and MUST NOT be used to pass secrets. The same holds
for loop variables.

#### Failure

* A body Step instance of the iteration in progress that fails, because an attempt of one of its Tasks that has not yet
  succeeded fails or one of its groups fails to evaluate, fails the loop at once. No further iteration starts, and the
  loop does not satisfy the `dependsOn` of the Steps that depend on it, as a failed Step does not. Other instances of
  the same iteration that do not depend on the failed one may still run, as independent Steps of a Job may today after a
  Step fails.
* An error in one of the loop's own groups (*enableIf*, *maxIterations*, an *initial*, or the first *while*) fails the
  loop at once, or fails Job creation if the group is evaluated then (see [Loop binding](#loop-binding)).
* Exceeding a limit of [Limits](#limits) item 2 or 4 fails the loop; for item 4 checked at Job creation, it fails Job
  creation.
* An error at the iteration boundary fails the loop at once (see
  [The iteration boundary](#the-iteration-boundary)).
* Reaching the bound fails the loop (step 5 of the boundary).
* A loop whose dependency fails, or can never run, never runs, as a Step behind a failed dependency never does.

**Recovery.** Only a failure of the Tasks of a body Step instance of the iteration in progress is recoverable, in the
same way as any Step's failure under RFC 0011 and RFC 0012: a scheduler that lets a failed Task be retried or requeued
may do so for the Tasks of a failed instance of that iteration. A loop whose failure is recoverable is `FAILED` until
every failed body Step instance of the iteration in progress has succeeded after a retry or requeue of its Tasks; it
then returns to `RUNNING`, and the iteration continues to its boundary. Until then the Steps that depend on the loop
are blocked, exactly as for a failed Step, and a job output that reads it is not evaluated (see
[RFC 0011 §Job outputs](0011-control-flow.md#job-outputs) item 4).

A re-run of a body Task that has already succeeded, in any iteration or after the loop has ended, leaves the Task, and
so its instance, `SUCCEEDED` (see
[RFC 0012 §Retries and re-runs](0012-step-output-variables.md#retries-and-re-runs)), so it never fails the loop or
changes its state, its loop variables, or its loop outputs. Only a failed attempt of a Task that has not yet
succeeded counts as a failure of a body Step instance.

Every other failure of a loop is final: the loop is `FAILED` and cannot return to `RUNNING`. These are a group of a body
Step instance that fails to evaluate; a group of the loop itself that fails to evaluate; exceeding a limit of
[Limits](#limits), including an instance whose Tasks would take the loop over a Task limit; any failure at the iteration
boundary (an error in *while*, *until*, *next*, or a loop output *value*, or a violated loop variable or loop output
constraint); a violated `minLength` when the loop ends, including after zero iterations; and reaching the bound. Once
any final failure occurs, the loop cannot return to `RUNNING`, even if a failed Task of the iteration in progress is
later retried or requeued and succeeds.

#### Sessions

A body Step instance is a Step for the rules of Sessions: a Session holds the Step Environments of one Step, so a
Session that runs Tasks of two instances, of the same body Step in two iterations or of two body Steps, exits the
first instance's Step Environments and enters the second's. A scheduler MAY run instances of different iterations in
one Session under those rules, keeping its Job Environments entered, but need not.

#### Retries and re-runs

[RFC 0012 §Retries and re-runs](0012-step-output-variables.md#retries-and-re-runs) applies to every Task of a body Step
instance, with one addition to when an output is **consumed**: any `Steps.*` output, of a body Step instance or of one
of the loop's dependencies, that a loop's *while*, *until*, *next*, *initial*, or loop output *value* references, as
found by symbol collection, is consumed once that expression has been evaluated, consistent with
[RFC 0012 §Retries and re-runs](0012-step-output-variables.md#retries-and-re-runs). So a re-run of a Task of a completed
iteration, after the loop has gone past its boundary, that succeeds with different values for a consumed output fails
under rule 3, because the next iteration was built from them. Loop variables and loop outputs are never re-evaluated,
and an iteration is never run again. A failed attempt of a Task of the iteration in progress that has not yet succeeded
may be retried as any Task's may, and the loop recovers as [Failure](#failure) describes. A re-run of a body Task that
has already succeeded leaves its instance `SUCCEEDED`, as RFC 0012 says, so it never fails the loop and the boundary
does not wait for it (see [Failure](#failure)).

#### Limits

1. *maxIterations* is at most 1,000. A scheduler MUST support every value from 1 to 1,000.
2. [RFC 0011 §Limits](0011-control-flow.md#limits) item 1, a scheduler's maximum number of Tasks per Step, applies to
   each body Step instance. A scheduler MAY also impose a maximum on the total number of Tasks of all instances of a
   loop, or on the number of body Step instances a loop creates. An instance whose materialized parameter space would
   take the loop over such a limit fails when its Task enumeration group is evaluated, and an iteration that would
   take it over the instance limit fails the loop when it would start; the message names the limit. RFC 0011's
   maximum total number of Tasks per Job counts the Tasks of every body Step instance of every iteration, and an
   instance whose materialized parameter space would take the Job over it fails in the same way. A loop has no
   Tasks of its own, so it adds nothing to a Task count. These failures are final (see [Failure](#failure)).
3. A loop and each of its body Steps count once each as a Step of the template, as a top-level Step does, wherever
   the base specification or a scheduler counts the Steps of a Job Template. Body Step instances are not Steps of the
   template, and the number of iterations does not change the count.
4. A scheduler MUST accept loop variable and loop output values totaling at least 1 MiB (1,048,576 bytes) per loop at
   any moment, and MAY accept more. The size counted is that of the values the loop currently holds, each as its JSON
   serialization ([RFC 0011 §Limits](0011-control-flow.md#limits)): the current value of each loop variable (a value
   that *next* replaces stops counting), the current value of each `LAST` loop output (nothing before its first
   iteration's value), and each element of each `APPEND` loop output, counted on its own, so the brackets and separators
   of the list do not count. The total is checked after the *initial* values are evaluated, at loop binding (at Job
   creation if the start group is evaluated then, where exceeding the limit fails Job creation), and at each iteration
   boundary after step 1 (loop outputs) and after step 3 (*next*) of [The iteration boundary](#the-iteration-boundary).
   Exceeding its limit fails the loop.
5. Expressions in a loop are evaluated under EXPR's existing memory and operation limits.

#### Composition with other extensions

* **`TASK_CHUNKING`.** A body Step may be chunked. RFC 0012's rules for chunked Steps apply to each instance.
* **`TASK_DEPENDENCIES`.** A body Step's subspace dependencies may name only body Steps of the same loop, and select
  Tasks of their instances in the same iteration; their subspace ranges may reference `Loop.*` and `RawLoop.*` (see
  [The `Loop.*` scope](#the-loop-scope)). A top-level Step may not have a subspace dependency on a loop, which has no
  Tasks, and a loop's own `dependencies` must be whole-Step dependencies. Dependencies between iterations are Future
  Work.
* **`WRAP_ACTIONS`** and other extensions that change how a Step's Actions run apply to body Step instances as to any
  Step.

### Modifications to How Jobs Are Run

> A modification to [How Jobs Are Run](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run)

Add a section, *Step Loops*: a loop runs its body Steps once per iteration, one iteration after another; each iteration
has its own instance of every body Step, with its own Tasks; at the end of each iteration the scheduler evaluates the
loop's outputs and decides, from `until` or `while`, whether another iteration runs; a loop that would exceed
`maxIterations` fails; and a failure of a body Step instance of the iteration in progress fails the loop. Only a failure
of the Tasks of a body Step instance of the iteration in progress is recoverable: a loop whose failure is recoverable is
`FAILED` until every failed body Step instance of the iteration in progress has succeeded after a retry or requeue of
its Tasks; it then returns to `RUNNING`, and the iteration continues to its boundary. A re-run of a body Task that has
already succeeded leaves the Task `SUCCEEDED`, under
[RFC 0012 §Retries and re-runs](0012-step-output-variables.md#retries-and-re-runs), so it never fails the loop or
changes its state, variables, or outputs, and the iteration boundary does not wait for it. Only a failed attempt of a
Task that has not yet succeeded counts as a body Step instance failure. Every other loop failure, including an
expression or constraint error in the loop or at an iteration boundary, a binding error of a body Step instance,
exceeding a limit, a violated `minLength` when the loop ends, and reaching `maxIterations`, is
final, and once one occurs the loop cannot return to `RUNNING`. See [Running a loop](#running-a-loop).

### Validation

Every rule this RFC adds, and when it is checked, in the form of
[RFC 0011 §Validation](0011-control-flow.md#validation).

**At template validation** (by the model library, before submission):

1. A `<StepLoop>` has *name* and *loop*, and only the properties of [`<StepLoop>`](#steploop); its *loop* has *steps*
   and *maxIterations*, and exactly one of *while* and *until*.
2. A body is a non-empty list of `<StepTemplate>`s, none a `<StepLoop>`.
3. The names of all Steps of the template, top-level, loops, and body Steps, are distinct.
4. A top-level Step's `dependencies` name only top-level Steps; a body Step's name only Steps of the same body; each
   body's dependency graph is acyclic, as is the top-level graph.
5. *variables*: 1 to 32 elements with unique `<Identifier>` names other than `Iteration`; types are
   `<OutputVariableType>`s; constraint properties are valid for the type.
6. *outputVariables* of a loop: 1 to 32 elements with unique `<Identifier>` names; *collect* is present and is `LAST`
   or `APPEND`, and `APPEND` only with a `LIST[...]` type; *value* is present; types, constraint properties, and
   *unit* as for RFC 0012's *outputVariables*.
7. *maxIterations*: an integer from 1 to 1,000, or a format string that references only `Param.*` (except `PATH`
   and `LIST[PATH]` parameters), `RawParam.*`, and `Job.Name` and has type `int`.
8. `Loop.*` and `RawLoop.*` appear only where [The `Loop.*` scope](#the-loop-scope) allows, name a declared variable
   or `Iteration`, and a `Loop.*` whose type is or contains `path` only in format strings resolved on the worker host.
9. `Steps.*` symbols in a loop's expressions and in its body Steps name only the Steps that
   [`Steps.*` in and around a loop](#steps-in-and-around-a-loop) allows; no top-level Step or job output names a body
   Step; `Steps.<L>.*` of a loop `L` names a declared loop output and is not `TaskParam` or `TaskRawParam`.
10. Every expression type-checks with `Loop.*` and `RawLoop.*` as `unresolved` values of their types; *while* and
    *until* have type `bool`; each *initial* and *next* has its variable's type; each loop output *value* has its
    output's type, or under `APPEND` its element type.

**At Job creation and loop binding** (by the scheduler): *maxIterations* evaluates to an integer from 1 to 1,000;
every *initial*, and the first *while*, evaluate, convert, and satisfy their constraints; and the size limit of
[Limits](#limits) item 4 holds for the *initial* values. A failure fails the loop at once, finally, or Job creation if
the group is evaluated then.

**When a zero-iteration loop would succeed** (by the scheduler): when the first *while* is `false`, the `minLength` of
each `APPEND` loop output holds for `[]`. This check is not a materialization group and is never part of Job
creation; it runs once the loop's dependencies are satisfied, as a Step with zero Tasks succeeds then under
[RFC 0011 §Step states](0011-control-flow.md#step-states). A violation fails the loop, finally.

**At iteration start and body Step binding** (by the scheduler): every rule of RFC 0011 and RFC 0012 for each body
Step instance, and the limits of [Limits](#limits) item 2. A failure fails the loop; only a failure of an
instance's Tasks is recoverable (see [Failure](#failure)).

**At the iteration boundary** (by the scheduler): each loop output *value*, *until*, *next*, and *while* evaluates
to a non-`null` value that converts to its type and satisfies its constraints, the size limit of [Limits](#limits)
item 4 holds, the loop does not reach *maxIterations* while continuing, and, when *until* or *while* ends the loop,
the `minLength` of each `APPEND` loop output holds. A failure fails the loop, finally.

**On re-run** (by the scheduler): the rules of [Retries and re-runs](#retries-and-re-runs).

## Prior Art

| System | Loop over | Condition | Carried state | Bound | Outputs after the loop |
|---|---|---|---|---|---|
| CWL (`cwltool:Loop`, proposed for v1.3) | One workflow step, which may run a sub-workflow | `loopWhen`, before each iteration; false before the first skips the step | `loop` maps step outputs to the next iteration's inputs | none | `outputMethod`: the last iteration's or all of them |
| Argo Workflows | A template that calls itself | `when` on the recursive call | the call's arguments | none; recursion depth | the last call's |
| AWS Step Functions | Any states, through a Choice that transitions back | Choice rules | the state JSON | the execution history limit | the state JSON |
| Nextflow (preview recursion) | A process or workflow, with `recurse` | `until` or `times` | the outputs of each iteration as the next inputs | `times` | the final outputs |
| Apache Airflow, GitHub Actions, Tekton | none; their graphs are acyclic | | | | |

What this RFC takes from them:

* **Last or all** (CWL `outputMethod`). A loop output's `collect: LAST` and `APPEND` are CWL's two output methods,
  chosen per output.
* **A loop over a sub-graph** (CWL's step that runs a sub-workflow, Step Functions' transition back over several
  states, Argo's recursive template that contains a DAG). Every system that loops over more than one unit does it by
  repeating a contained group, never by allowing cycles between peers.
* **Explicit carried state** (CWL's `loop` mapping, Nextflow's iteration inputs, Argo's arguments). What passes
  between iterations is named.

What this RFC does differently:

* **A required bound that fails the loop.** None of the surveyed systems requires one. Argo and Step Functions run
  into a resource limit instead, with an error far from the cause. OpenJD Jobs run on shared farms, where a runaway
  loop costs real hosts, and a declared bound lets a scheduler show the most work a Job can do.
* **`while` and `until`.** CWL's `loopWhen` is checked before each iteration, so a CWL loop that must run once needs
  its condition to be true initially; `until` states that directly.
* **Declared, typed loop outputs.** Steps after the loop read its outputs, not the body's, so the body is an internal
  detail of the loop, as a sub-workflow is in CWL.

## Implementation Impact

- **Python (openjd-model)** and **Rust (openjd-rs)**: The `<StepLoop>`, `<LoopDefinition>`,
  `<LoopVariableDefinition>`, and `<LoopOutputVariableDefinition>` schemas, the template-wide name rule, and nested
  scopes for the `Loop.*` symbol family and the loop rules of `Steps.*`. Job creation treats a loop as a Step for
  binding, and the library exposes three functions schedulers and the CLI call: one that evaluates a loop's bound and
  start groups, one that instantiates the body for an iteration given the loop variables' values (returning each body
  Step's instance template ready for RFC 0011's binding), and one that evaluates an iteration boundary given the
  `Steps.*` values of its instances, returning the loop outputs recorded, and either the next iteration's variables
  or the loop's end and its outcome.
- **Sessions (openjd-sessions, Python and Rust)**: Nothing new. A body Step instance runs as a Step, with `Loop.*`
  values supplied with its other symbols, and with `PATH` values marked as paths so that the Session maps them.
- **CLI (`openjd run`)**: Runs a loop's iterations in order, each iteration's body Steps in dependency order, prints
  each iteration's index, loop variable values, and loop outputs, and fails the run as the loop fails. `openjd run
  --step` naming a loop runs the whole loop. `--tasks`, `--task-param`, and `--maximum-tasks` choose which Tasks of the
  selected Step run, and a loop has no Task parameters or Tasks of its own, so `--tasks` and `--task-param` are rejected
  with an error whenever the selected Step is a loop, whether named by `--step` or selected as the Job's only top-level
  Step. Only the rejection of an explicitly given `--maximum-tasks` is limited to an explicit `--step` naming a loop;
  this needs the CLI to tell a given `--maximum-tasks` from its default. In every other run that includes a loop,
  including a full run, the CLI's selection of a single top-level loop, and `--run-dependencies` for a `--step` that is
  not a loop, `--maximum-tasks` applies to each body Step instance as it does to any Step. A body Step instance that did
  not run all its Tasks is incomplete. The other instances of its iteration follow the CLI's usual rule for an
  incomplete producer: only the Steps that read the incomplete instance's outputs are blocked. A failure of another
  instance of the iteration fails the loop as usual. The iteration never completes, so the boundary is not evaluated,
  the loop is reported as blocked by an incomplete body Step instance, and every Step that depends on the loop is
  blocked behind it, as for a blocked dependency. `--step` naming a body Step is rejected with an error that names its
  loop, because a body Step runs only within its loop's iterations. `openjd summary` reports a loop's bound and body.
- **Schedulers**: Creating body Step instances as iterations start, running iterations in order, evaluating the
  boundary, storing the loop variables and outputs, and showing iterations. A scheduler whose Jobs have a fixed set
  of Steps from submission must add the ability to create Steps while a Job runs, or represent each iteration's
  instances in some other way that keeps their states, Tasks, and outputs separate.

## Rejected Ideas

### Cyclic `dependsOn` edges between Steps

Letting a Step depend on a later Step, so that `Render` depends on `Check` and `Check` on `Render`, with a condition
on the back edge. Rejected because validation could no longer tell an intended back edge from a mistake, since a
cycle is today always an error; because every scheduler that orders Steps by a topological sort, and the base
specification's rule that Step dependencies form an acyclic graph, would break; and because which Steps repeat, and
what each repetition may read from the last, would be implicit in which edges happened to form a cycle.

### A single-Step loop Step

The sketch in RFC 0011's earlier Future Work: a Step whose Tasks run one after another, each with parameters from an
`initial` and a `next` that read the Task before it. Rejected because it can't repeat a group of Steps, which is the
requirement, and because a loop whose body is one single-Task Step does everything it did, with the same history
through `collect: APPEND`.

### Tasks that add Steps or iterations

A Task, or a directive, that adds another iteration's Steps to the running Job. Rejected for the reasons RFC 0011
gives against Tasks that add Steps (see
[RFC 0011 §Tasks that add Steps to a running Job](0011-control-flow.md#tasks-that-add-steps-to-a-running-job)): the
step graph would no longer be in the template, and no tool could validate or show it.

### Unbounded loops

A loop with no `maxIterations`, or a bound that a scheduler chooses. Rejected because termination can't be checked,
because a Job's work would no longer be bounded at Job creation (RFC 0011 Technical Requirement 7), and because a
scheduler's own bound would make the same template behave differently on different farms.

### Ending quietly at `maxIterations`

Treating the bound as a success with the last iteration's values. Rejected because it passes an unfinished result on
as finished (see [Design Choice 5](#5-maxiterations-is-required-and-reaching-it-fails-the-loop)).

### Reading earlier iterations from the body

A symbol such as `Steps.CheckPass.Previous.Output.Score`, or `Steps.CheckPass.Output.Score` meaning the last
completed iteration in a body Step that comes before `CheckPass`. Rejected because it is a second, undeclared channel
between iterations beside the loop variables (see
[Design Choice 6](#6-body-steps-read-the-loop-their-dependencies-in-the-body-and-the-loops-dependencies)).

### Reading body Steps after the loop

Letting a Step after the loop read `Steps.CheckPass.Output.Score`, as the last iteration's value. Rejected because a
body Step has many instances, and a declared loop output says which value is meant (see
[Design Choice 7](#7-steps-after-a-loop-read-only-its-declared-outputs)).

### `collect: ONCE` for loop outputs

Rejected in favor of requiring `LAST` or `APPEND` (see
[Design Choice 8](#8-loop-outputs-are-evaluated-after-each-iteration-and-collected-as-last-or-append)).

### A `loop` field on `<StepTemplate>`

Rejected in favor of a separate `<StepLoop>` element (see [Design Choice 11](#11-a-separate-steploop-element)).

### Body Step names scoped to their loop

Rejected in favor of names unique across the template (see
[Design Choice 10](#10-step-names-are-unique-across-the-whole-template)).

### Overlapping iterations

Starting iteration `N + 1`'s first Steps while iteration `N`'s last Steps still run, when they don't read each other.
Rejected because the next iteration's variables come from the end of the current one, so an overlap needs a second
notion of which values an early Step may use, and because parallelism belongs inside the body (see
[Design Choice 3](#3-iterations-run-one-after-another-and-instantiate-the-whole-body)).

## Open Questions

1. **Nested loops.** This RFC forbids a loop in a body. Use cases exist, such as an outer loop over refinement levels
   with an inner convergence loop. Nesting needs rules for which `Loop.*` a body reads (`Loop` for the innermost, and
   a name for each outer one) and makes the bound multiplicative. Should a later revision allow one level?
2. **Partial re-run of an iteration.** Render farms requeue Tasks. Under this draft a re-run that changes a consumed
   value fails. Should a scheduler be able to re-run a loop from iteration `N`, discarding the iterations after it?
   That is cascade recompute (RFC 0012 Future Work) applied to a loop.
3. **Independent loops per parameter combination.** "Refine each of 20 shots until clean" is 20 independent loops.
   This draft needs one loop per shot written out, or a body that fans out over the shots and a condition over all of
   them, which makes every shot wait for the slowest. Should a loop have a `parameterSpace` of its own, with one loop
   instance per Task parameter combination?
4. **One extension name.** Loops are the largest new burden in `CONTROL_FLOW` for a scheduler. Should they have their
   own extension name, so that a scheduler can offer data flow and conditions without them, at the cost of the split
   RFC 0011 Design Choice 1 argues against?
5. **The bound.** Is 1,000 iterations the right maximum, and should *maxIterations* be able to read the outputs of the
   loop's dependencies, at the cost of the bound not being known at Job creation?
6. **Early evaluation of body Steps.** Nothing of a body Step is evaluated at Job creation, so an error in a body Step
   field that reads only `Param.*` is found when iteration 0 starts. Should a scheduler evaluate such fields at Job
   creation, with each iteration evaluating them again?
7. **Environments shared by a loop.** A license or a cache warmed once for all iterations would need Step
   Environments on the loop itself, entered by every Session that runs a body Step instance. Is that needed?
8. **Reporting.** Should a scheduler report the most Tasks a loop could run, from *maxIterations* and the body's
   static parameter spaces, as an upper bound before the loop starts?

## Future Work

Each of these was deferred from this draft and can be added later without breaking templates written against it.

| Deferred | Why it can wait |
|---|---|
| Nested loops | No accepted use case in this draft's discussion; needs scoping rules (Open Question 1). |
| Loops with a parameter space, one loop per combination | Can be written out by hand or with a body that fans out (Open Question 3). |
| Re-running a loop from an iteration | Scheduler policy, like cascade recompute in RFC 0012. |
| Step Environments on a loop | Each body Step instance can enter its own (Open Question 7). |
| `TASK_DEPENDENCIES` between iterations | Iterations do not overlap in this draft. |
| Overlapping or pipelined iterations | Parallelism inside the body covers the known cases. |

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is more permissive.
