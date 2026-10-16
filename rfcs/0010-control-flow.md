* Feature Name: control_flow
* Author(s): Mark Wiebe <[mwiebe](https://github.com/mwiebe)>
* RFC Tracking Issue: (pending). The number `0010` is a placeholder assigned before a tracking issue exists, and may
  change when one is created.
* Start Date: 2026-10-15
* Specification Version: 2023-09 extension CONTROL_FLOW
* Accepted On: (pending)
* Depends On:
  * RFC 0002, Model Extensions (https://github.com/OpenJobDescription/openjd-specifications/issues/57)
  * RFC 0005, Expression Language (EXPR) (https://github.com/OpenJobDescription/openjd-specifications/issues/112)
  * RFC 0006, Expression Function Library
  * RFC 0007, Extended Parameter Types
* Related:
  * Discussion #111, variable data flow along step dependencies, the seed idea for this RFC
    (https://github.com/OpenJobDescription/openjd-specifications/discussions/111)
  * Discussion #81, `enableIf` to include or exclude parts of a template
    (https://github.com/OpenJobDescription/openjd-specifications/discussions/81)
  * Discussion #82, task-task dependencies (https://github.com/OpenJobDescription/openjd-specifications/discussions/82),
    proposed here as a separate `TASK_DEPENDENCIES` extension
  * Discussion #22, immutability vs. dynamic adjustments
    (https://github.com/OpenJobDescription/openjd-specifications/discussions/22)
  * Investigation notes behind this draft: [`docs/control-flow-investigation/`](../docs/control-flow-investigation/)

> **Draft status.** This is the first draft. Blocks marked **TODO** are parts of the design space that this draft
> deliberately defers (see [Future Work](#future-work)); each states what is missing and why it can wait. The
> [Open Questions](#open-questions) section lists the decisions to confirm on discussion #111 before this RFC
> leaves draft.

## Summary

Open Job Description Jobs have a structure that is fixed when the Job is created: every Step's Tasks are enumerated
from Job Parameters at submission. Many real pipelines need part of that structure to come from data produced while
the Job runs: how many frames a video has, which objects are under an S3 prefix, which configuration won a parameter
sweep, or whether a quality check found a problem. Today that data reaches a Job only through a pre-submission script,
which job bundle submitters cannot run, or through files and conventions that no scheduler or tool can see.

This RFC proposes a `CONTROL_FLOW` extension that lets a Step declare typed **output variables**, lets a Task report
them with an **`openjd_output_var:` stdout directive** (with an `@<file>` form for large values), and lets the Steps
that depend on it use those values in EXPR expressions through a new `Steps.<Name>.Output.<Var>` symbol class. With those values a dependent Step can compute
its parameter space (**dynamic fan-out**), decide whether it runs at all (**conditional execution** with `enableIf`),
and reduce the values of every Task of a multi-Task Step (**fan-in**) with the existing EXPR function library. The
Job's Steps and their dependencies stay fixed and are validated before submission; only Task counts, Task parameter
values, and enablement are late-bound, in one new evaluation stage called **step input binding**. Output variables
are also how a Task reports **metrics**, such as an error score or a frame rate, for a scheduler to show beside the
Task's parameters: a metric is an output variable that no Step reads. A Job can also declare **job outputs**, typed
values computed from its parameters and its Steps' outputs, which complement job parameters by reporting results
back to the submitter as soon as each one is known. `PATH` values that flow out of a Task are stored in the
submission path context, so a path a Task reports on a Linux worker reaches a Windows submitter in a form it can use.

## Overview

### What CONTROL_FLOW adds

The `CONTROL_FLOW` extension adds the following, each specified in the section linked:

* **Declaring outputs.** An `outputVariables` list on `<StepTemplate>`, typed with the job parameter type vocabulary
  and constrained with the same properties, with an optional display `unit` and a `collect` mode (`ONCE`, `LAST`, or
  `APPEND`) for repeated reports: [`<OutputVariableDefinition>`](#outputvariabledefinition).
* **Reporting outputs.** An `openjd_output_var: <Name>=<JSON>` stdout directive, or `openjd_output_var: <Name>=@<Path>`
  to take the value from a file, printed by a Task's `onRun` and validated strictly as soon as it is read:
  [Output emission](#output-emission).
* **Reading outputs.** The `Steps.<StepName>.Output.<VarName>` and `Steps.<StepName>.TaskParam.<ParamName>` symbols,
  available in Steps that depend directly on the producer: [The `Steps.*` scope](#the-steps-scope).
* **Late-bound structure.** Parameter-space ranges, step `let` bindings, host requirements, and Step Environment
  fields that reference `Steps.*` are evaluated at a new stage, step input binding, while the Step waits in a new
  `PENDING_INPUTS` state: [Step input binding](#step-input-binding).
* **Conditional execution.** An `enableIf` predicate on Steps and Environments, decided at Job creation or at step
  input binding. A disabled Step is `SKIPPED`: [`enableIf`](#conditional-execution-enableif).
* **Fan-in.** A multi-Task producer's outputs are ordered lists in parameter-space iteration order, reduced with
  RFC 0006 functions: [Reductions](#reductions).
* **Job outputs.** A `jobOutputDefinitions` list on the Job Template, each an expression over job parameters and
  `Steps.*`, evaluated as soon as its inputs are known: [Job outputs](#job-outputs).
* **Paths.** `PATH` and `LIST[PATH]` outputs, reverse-mapped to the submission path context, and `RawOutput` and
  `TaskRawParam` forms that read them unmapped: [Path values](#path-values).
* **Composition with task-task dependencies.** A rule that makes computed task DAGs work when a template also enables
  the separate `TASK_DEPENDENCIES` extension: [Composition with `TASK_DEPENDENCIES`](#composition-with-task_dependencies).
* **Validation.** An index of every rule the extension adds, static and runtime: [Validation](#validation).

### Key concepts

**Fixed step graph, late-bound width.** The set of Steps, their names, their dependencies, and the types of their
outputs are in the template, so a validator can check every reference before submission. What may depend on a value
produced at run time is how many Tasks a Step has, the values those Tasks receive, and whether a Step or Environment
runs. Steps never create new Steps.

**Output variables are control data.** An output is a small typed value that shapes the rest of the Job: a count, a
list of keys, a score, an adjacency list. Bulk data keeps flowing through files and `PATH` job parameters.

**Metrics are outputs too.** A scheduler stores and can show every Task's committed outputs, whether or not a Step
reads them, so a Step that reports measurements declares them as output variables like any other. There is one
reporting mechanism with one set of rules.

**One new stage, two new states.** EXPR evaluates format strings at template validation, Job creation, and Task
execution. CONTROL_FLOW inserts a fourth stage per Step, between Job creation and Task execution, at which the Step's
TEMPLATE-scope expressions that read upstream outputs are evaluated. A Step waiting for that stage is in the
`PENDING_INPUTS` state, and a Step whose `enableIf` is false ends in the `SKIPPED` state. A Step that has no
`enableIf` and reads no upstream outputs in those fields never enters either state and behaves exactly as today.

**Types first.** Every `Steps.*` symbol has a type known at template validation, derived from the producer's
declarations. EXPR's existing `unresolved[T]` placeholders type-check expressions that use them, so no second type
checker is needed.

### Evaluation stages

RFC 0005 defines three stages. CONTROL_FLOW adds step input binding, which runs once per Step that needs it:

```
  Template            Job Creation         Step Input Binding          Task Execution
  Validation          (submit time)        (per Step, when its         (worker host)
                                           producers are terminal)
  ──────────────      ──────────────       ──────────────────────      ──────────────
  Param.*:            Param.*:             Param.*:                    Param.*:
    unresolved[T]       CONCRETE             CONCRETE                    CONCRETE
  Steps.P.Output.*:   Steps.P.Output.*:    Steps.P.Output.*:           Steps.P.Output.*:
    unresolved[T]       unresolved[T]        CONCRETE                    CONCRETE
  Task.Param.*:       Task.Param.*:        Task.Param.*:               Task.Param.*:
    unresolved[T]       unresolved[T]        unresolved[T]               CONCRETE

  ► Type-check        ► Evaluate static    ► Evaluate the Step's       ► Evaluate all
    all exprs,          enableIf and         materialization             remaining
    check Steps.*       TEMPLATE scope;      expressions; Step           expressions;
    references          Steps that read      becomes READY,              collect onRun's
                        outputs enter        SKIPPED, or FAILED          output directives
                        PENDING_INPUTS
```

### Step states

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

`SKIPPED` and a zero-Task `SUCCEEDED` both satisfy `dependsOn`. `FAILED` does not. A Step whose `enableIf` is decided
at Job creation goes directly to `SKIPPED` or to the left-hand path.

A Step with no Tasks to run, because it is `SKIPPED` or its parameter space is empty, still respects its own
dependencies: the Steps that depend on it are released only once its own dependencies are satisfied. So skipping a
Step never lets work that was ordered after it run sooner than the Steps it was ordered after.

## Basic Examples

These examples use EXPR as specified in the [Expression Language](../wiki/2026-02-Expression-Language.md) and the
schema in [Specification](#specification). Application command lines (`ffmpeg`, `stylize`, `bake-node`, ...) are
illustrative.

An earlier revision of these examples was tested as far as is possible without an implementation of CONTROL_FLOW.
For each one, a proxy template replaced every `Steps.P.Output.X` and `Steps.P.TaskParam.Y` with a job parameter of
the type this RFC assigns, and dropped `outputVariables`, `enableIf`, and `dependsOnSubspace`, moving their
expressions into `let` bindings. The proxies passed `openjd check` in the openjd-rs CLI, and hand-written copies passed
`openjd run` with mock values, including empty upstream lists, the `fail(...)` paths, and a `null` output from a
skipped Step. That revision wrote outputs to a file. The examples now print `openjd_output_var:` directives instead,
and [Task metrics from a parameter sweep](#task-metrics-from-a-parameter-sweep) is new; neither change has been
re-run through the proxies. Without `CONTROL_FLOW`, a directive is ordinary output, so it does not affect what the
proxies test. The proxies cannot test the CONTROL_FLOW semantics themselves (binding, skipping, collecting
directives).

| Example | Use case | Data flow | Fan-out | `enableIf` | Task DAG | Fan-in |
|---|---|---|---|---|---|---|
| [Video frame count](#video-frame-count) | U1 | ✔ | ✔ | | | |
| [S3 prefix fan-out with a manifest](#s3-prefix-fan-out-with-a-manifest) | U3 | ✔ | ✔ | | | ✔ |
| [Task metrics from a parameter sweep](#task-metrics-from-a-parameter-sweep) | U16 | (no consumer) | | | | |
| [Parameter sweep, then run the winner](#parameter-sweep-then-run-the-winner) (with job outputs) | U4 + U3 + U18 | ✔ | ✔ | | | ✔ |
| [Scene-graph DAG](#scene-graph-dag) | U2 | ✔ | ✔ | | ✔ (`TASK_DEPENDENCIES`) | |
| [Render, quality check, and optional steps](#render-quality-check-and-optional-steps) | U5 + dynamic | ✔ | ✔ (may be empty) | ✔ static and dynamic | | |

### Video frame count

Discussion #111, Example 1. The submitter doesn't know the frame count; the first Step finds it, and the second Step
runs one Task per frame.

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
        echo "openjd_output_var: FrameCount=$count"
        echo "openjd_output_var: FrameRate=\"$rate\""

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

* `StylizeFrame` is in `PENDING_INPUTS` until `VideoToFrames` succeeds. `minValue: 1` makes an empty video fail
  `VideoToFrames` itself. Without it, an empty video would produce the range string `"1-0"` downstream, which is
  not empty: under §3.4.1.1.1 it is `[1]`.
* `FramesToVideo` uses `FrameRate` only in its script, which is evaluated on the worker host, so it never enters
  `PENDING_INPUTS`. It still lists `VideoToFrames` directly, because only direct dependents may read a Step's
  outputs, even though `StylizeFrame` already orders it after.

The scheduler's view of the Job over time:

```
VideoToFrames   READY ─► RUNNING ─► SUCCEEDED (FrameCount=1440, FrameRate="24000/1001")
StylizeFrame    PENDING_INPUTS ─────────────► binding: range "1-1440" ─► READY (1440 Tasks) ─► ... ─► SUCCEEDED
FramesToVideo   READY (gated on both dependencies) ──────────────────────────────────────────────► RUNNING ─► ...
```

### S3 prefix fan-out with a manifest

One discovery Task lists the objects under an S3 prefix, one Task per object transcodes it, and a final Task writes
a manifest. Today this needs a pre-submission script, which job bundle submitters can't run.

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
        # 5000 keys can exceed one stdout line, so write the value to a file and name it.
        with open("keys.json", "w", encoding="utf-8") as f:
            json.dump(keys, f)
        print("openjd_output_var: Keys=@keys.json", flush=True)

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
        print(f"openjd_output_var: OutputBytes={size}")

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

* `Manifest` gets the keys from `Steps.Transcode.TaskParam.Key`, which is aligned element-for-element with
  `Steps.Transcode.Output.OutputBytes`, so it needs no dependency on `Discover`.
* If the prefix holds no `.mov` objects, `Transcode` is a zero-Task success and `Manifest` writes an empty manifest
  with `totalBytes` 0.
* `Discover` writes `keys.json` in the Session's Working Directory, where an Action starts, and names it relative
  to that directory. The runtime reads the file as soon as it reads the directive, so the file must be complete and
  closed first. A short list could be printed inline instead: `print("openjd_output_var: Keys=" + json.dumps(keys))`.

### Task metrics from a parameter sweep

From the use case folouiseAWS posted on #111 and the Task metrics proposal that preceded this RFC: a hand-tracking
pipeline is swept over 27 configurations and 10 benchmark episodes, 270 Tasks, and judged by three numbers. No Step
reads them. The scheduler stores each Task's outputs and can show them as sortable columns beside the Task parameters,
which is the whole point. The example uses each of the three `collect` modes.

```yaml
specificationVersion: jobtemplate-2023-09
extensions: [EXPR, CONTROL_FLOW, FEATURE_BUNDLE_1]
name: hand-tracking-sweep-01
steps:
  - name: ExtractHandActions
    parameterSpace:
      taskParameterDefinitions:
        - {name: Video,   type: STRING, range: [P0001_a68, P0002_273c, P0003_c701, P0009_02511, P0010_0ecbf,
                                                P0011_76ea6, P0012_915e71, P0014_8254f, P0015_42b8b, P0020_ef4d6]}
        - {name: Window,  type: INT,    range: [120, 150, 200]}
        - {name: Overlap, type: INT,    range: [20, 30, 40]}
        - {name: DetConf, type: FLOAT,  range: [0.5, 0.7, 0.75]}
    outputVariables:
      - name: mpjpe_mm
        type: FLOAT
        unit: mm
        description: Mean per-joint position error over one-second action chunks.
        minValue: 0
      - name: coverage_pct
        type: FLOAT
        unit: "%"
        description: Share of visible-hand frames with a direct prediction.
        minValue: 0
        maxValue: 100
      - name: fps
        type: FLOAT
        unit: frames/s
        description: End-to-end throughput.
      - name: frames_done
        type: INT
        collect: LAST           # a running count, updated as the Task works
        unit: frames
      - name: chunk_error_mm
        type: LIST[FLOAT]
        collect: APPEND         # one directive per one-second segment
        unit: mm
        description: Error of each one-second segment, in order.
        maxLength: 20000        # keeps the list well under the guaranteed 1 MiB per Task
    python:
      script: |
        from pipeline import extract_hand_actions, score_against_hot3d

        def on_progress(frames):
            print(f"openjd_output_var: frames_done={frames}", flush=True)

        on_progress(0)           # LAST needs at least one directive
        trajectory = extract_hand_actions({{repr_py(Task.Param.Video)}}, window={{Task.Param.Window}},
                                          overlap={{Task.Param.Overlap}}, det_conf={{Task.Param.DetConf}},
                                          on_progress=on_progress)
        score = score_against_hot3d(trajectory)
        for err in score.chunk_errors_mm:
            print(f"openjd_output_var: chunk_error_mm={err}")
        print(f"openjd_output_var: mpjpe_mm={score.mpjpe_mm}")
        print(f"openjd_output_var: coverage_pct={min(score.coverage * 100, 100.0)}")
        print(f"openjd_output_var: fps={score.fps}")
```

By default the Task's log contains the directives, as it does the other stdout messages:

```
0:00:00.812004  openjd_output_var: frames_done=0
0:00:41.102233  openjd_output_var: frames_done=1200
0:01:52.408115  openjd_output_var: chunk_error_mm=48.1
0:01:52.408190  openjd_output_var: chunk_error_mm=57.3
0:01:52.410211  openjd_output_var: mpjpe_mm=52.6
0:01:52.410305  openjd_output_var: coverage_pct=81.44
0:01:52.410371  openjd_output_var: fps=15.6
```

* The runtime validates each directive as it reads it and can forward the value to the scheduler right away. A
  scheduler may show `frames_done` while the Task runs, marked as not yet committed; the values are committed when the
  Task succeeds.
* `mpjpe_mm`, `coverage_pct`, and `fps` use the default `collect: ONCE`, so printing one of them twice fails the Task.
  `chunk_error_mm` collects its directives into a list, which is `[]` if none is printed.
* A value that isn't a number (including Python's `nan`), a misspelled name, or a `coverage_pct` over 100 fails the
  Task. In a sweep, a bad measurement is a bad result.
* `unit` and `description` are for display only. How to rank the results (lower or higher is better), thresholds such
  as "coverage at least 75", and grouping over `Video` describe how a person reads the results, and stay out of the
  template.
* A later Step could read the same outputs as lists and pick the winner, as in the next example.

### Parameter sweep, then run the winner

27 configurations × 10 benchmark episodes report two metrics each. The Job picks the most accurate configuration that
runs fast enough and then runs it on every video under an S3 prefix. Its job outputs report results back to the
caller as they become available.

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
jobOutputDefinitions:
  - name: VideoCount
    type: INT
    value: "{{len(Steps.Discover.Output.Videos)}}"            # available when Discover succeeds
  - name: BestSingleTaskErrorMm
    type: FLOAT
    unit: mm
    value: "{{min(Steps.Sweep.Output.mpjpe_mm)}}"             # available when the whole sweep succeeds
  - name: ResultDirs
    type: LIST[PATH]
    objectType: DIRECTORY
    value: "{{Steps.RunWinner.RawOutput.ResultDir}}"          # submission-context paths, one per video
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
        print(f"openjd_output_var: mpjpe_mm={score.mpjpe_mm}")
        print(f"openjd_output_var: fps={score.fps}")

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
        with open("videos.json", "w", encoding="utf-8") as f:
            json.dump(keys, f)
        print("openjd_output_var: Videos=@videos.json", flush=True)

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
    outputVariables:
      - name: ResultDir
        type: PATH
        objectType: DIRECTORY
    python:
      script: |
        import json
        from handtrack import track_s3_video
        result_dir = track_s3_video({{repr_py(Param.DatasetBucket)}}, {{repr_py(Task.Param.Video)}},
                                    window={{window}}, overlap={{overlap}}, det_conf={{det_conf}},
                                    out_dir={{repr_py(Param.OutputDir)}})
        print("openjd_output_var: ResultDir=" + json.dumps(str(result_dir)))
```

* `Sweep` and `Discover` don't depend on each other and run in parallel.
* The per-configuration means rely on the defined iteration order: with `Episode` last in `combination`, the Tasks of
  configuration `c` occupy indices `c*n` to `c*n + n - 1` of every `Steps.Sweep.*` list. This index arithmetic is the
  clearest readability cost of the design; see [Open Questions](#open-questions) item 5.
* If no configuration is fast enough, `RunWinner` fails at binding with the `fail(...)` message and runs no Tasks.
* `ResultDir` is an absolute path on the worker, under `Param.OutputDir` with the worker's path mapping applied. The
  runtime maps it back to the submission path context before storing it, so `ResultDirs` names the directories as the
  submitter sees them: given the farm's path mapping rule for `P:\`, a Windows workstation that submitted with
  `OutputDir` set to `P:\results` gets `P:\results\...` paths even though the Tasks ran on Linux (see [Path values](#path-values)).
* The job outputs appear one at a time: `VideoCount` when `Discover` succeeds, `BestSingleTaskErrorMm` (the lowest error of any
  one Task, not the winner's mean) when the sweep does, and `ResultDirs` when `RunWinner` does. If any of them failed to evaluate, the Job would fail at that point. Reporting the winning configuration itself as a
  job output would need `RunWinner` to report it as an output variable, because a job output can't read a Step's
  `let`.
* The `Sweep` Step alone is useful: a scheduler can show all 270 Tasks' `mpjpe_mm` and `fps` with no consumer (see
  [Task metrics from a parameter sweep](#task-metrics-from-a-parameter-sweep)).

### Scene-graph DAG

Discussion #111, Example 2: an analysis Step computes both the number of Tasks and the edges between them. This
example also needs the separate `TASK_DEPENDENCIES` extension, whose `dependsOnSubspace` syntax is sketched from #82
and is not specified by this RFC.

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
        for name, value in [("Children", [n.child_indices for n in g.nodes]),
                            ("NodeNames", [n.name for n in g.nodes])]:
            with open(f"{name}.json", "w", encoding="utf-8") as f:
                json.dump(value, f)
            print(f"openjd_output_var: {name}=@{name}.json", flush=True)

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

* The subspace range is evaluated once per `BakeNode` Task with `Task.Param.Node` bound to that Task. Leaf nodes have
  `[]` children and wait only on `AnalyzeSceneGraph`.
* At binding the scheduler evaluates `node_count`, materializes that many Tasks, evaluates every Task's subspace, and
  checks the resulting graph for cycles and out-of-range indices. A cyclic scene graph fails `BakeNode` before any
  bake runs, with the cycle in the message.
* `AnalyzeSceneGraph` is listed as a plain dependency. A `dependsOnSubspace` edge doesn't count as the direct
  dependency that reading `Steps.AnalyzeSceneGraph.*` requires, because it lets Tasks start before every upstream
  Task has finished.

### Render, quality check, and optional steps

A static `enableIf` from a `BOOL` parameter, a dynamic `enableIf` from an upstream metric, an output of a conditional
producer, and a fan-out that is usually empty.

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
        print("openjd_output_var: BadFrames=" + json.dumps(report.bad_frames))
        print(f"openjd_output_var: MeanNoise={report.mean_noise}")

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
        echo "openjd_output_var: VideoBytes=$(stat -c %s {{repr_sh(Param.OutputVideo)}})"

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

* With `EncodeVideo: false`, the `EncodeVideo` Step is `SKIPPED` at Job creation and `Publish` still runs with
  `video_bytes = null`, so its note reads `frames only`. Without the null check, `string(video_bytes)` gives the text
  `null`, and arithmetic such as `video_bytes + 1` passes validation but fails `Publish` when it binds. (The
  `summary` text avoids `': '`, which would turn the `let` item into a YAML mapping.)
* `Denoise` binds when `QualityCheck` succeeds and is skipped if the noise is under the threshold. Its `Rerender`
  dependency still gates its Tasks.
* `Rerender` with `BadFrames = []` is a zero-Task success, so the common clean render costs nothing.

## Motivation

An Open Job Description Job is created from a template and Job Parameter values, and every Step's Tasks are known at
that moment. Discussion #22 defended this immutability: it lets a scheduler show the whole Job, estimate when it will
finish, and tell a healthy Job from a stuck one. It also asked that "specific dynamic job structures" be added through
RFCs rather than by opening the Job to arbitrary mutation. This RFC is that addition for the cases the community has
asked for.

The cases share a shape. A Job's first Step discovers something, and the rest of the Job depends on what it found:

* How many frames does this video have? (#111, Example 1)
* Which objects are under this S3 prefix? (crowecawcaw, #111)
* What does this scene graph look like? (#111, Example 2)
* Which of these 27 configurations scored best? (folouiseAWS, #111)
* Is the render noisy enough to need denoising? (#111 introduction, #81)

None of these is known to the person submitting the Job, and some can't be computed on the submitter's machine at all
because the input lives on the farm or in a bucket the workstation can't reach.

### Use cases

| ID | Use case | Source | Unknown at submission | Covered by |
|---|---|---|---|---|
| U1 | **Video frame count.** Extract frames, process one Task per frame, re-encode. | #111 Ex. 1 | Task count of the middle Step | Outputs, dynamic ranges |
| U2 | **Scene-graph DAG.** Analyze a 3D scene into a DAG for lightmap baking, GI, or spatial partitioning; leaves run in parallel and parents wait for their children. | #111 Ex. 2 | Task count and the edges between Tasks | Outputs, dynamic ranges, and `TASK_DEPENDENCIES` |
| U3 | **S3-prefix fan-out.** List the objects under a prefix and process each in its own Task. Today this is a pre-submission script, which blocks use of the job bundle submitter. | crowecawcaw, #111 | Task count and `STRING` Task parameter values | Outputs, dynamic ranges, fan-in |
| U4 | **Parameter sweep, then run the winner.** 27 configurations × 10 episodes = 270 Tasks, each reporting accuracy and speed metrics; pick the best and run it on the full dataset in the same Job. | folouiseAWS, #111 | Which configuration the final Step uses | Outputs, `Steps.*.TaskParam`, fan-in |
| U5 | **Optional Step from a `BOOL` parameter**, such as skipping a video encode. | #81, issue #80 | Nothing | Static `enableIf` |
| U6 | **Optional Environment** gated by a parameter. | #81 | Nothing | Static `enableIf` |
| U7 | **CPU-only vs. GPU host requirements** chosen by a parameter. | #81 | Nothing | Already expressible with EXPR (see [Future Work](#future-work)) |
| U8–U14 | **Static task-task dependencies**: export→render per frame, simulation chains, temporal offsets, static DAGs, fan-in by frame. | #82, #79 | Nothing | `TASK_DEPENDENCIES` (separate extension) |
| U15 | **Fully dynamic work items** such as Houdini PDG. | #22 | Number and shape of the work | Out of scope |
| U16 | **Task metrics.** Each Task reports measurements (error, coverage, frames per second, render time, peak memory) that a scheduler shows as sortable columns beside the Task's parameters. | folouiseAWS, #111; the Task metrics proposal | Nothing; the values are the result | Outputs with no consumer, `unit`, `collect` |
| U17 | **Loop until done.** Run a Step's Tasks one after another, each one's parameters computed from the outputs of the Task before it, until a condition holds: refine until converged, list S3 one page at a time with a continuation token, or re-render with more samples until a quality check passes. | Discussion of this draft | How many Tasks run, and each one's parameters | Future Work: a separate loop extension (see [Loop Steps](#loop-steps)) |
| U18 | **Job outputs.** Report a Job's results back to its caller as typed values: the winning configuration of a sweep, the number of items processed, the directories that hold the results. A caller reads them as it supplies job parameters, and a user interface shows each as soon as it is known. | Discussion of this draft | The values, until the Steps they come from run | `jobOutputDefinitions`, `PATH` reverse mapping |

U1–U4 need a value from a running Task and are the core of this RFC. U5 and U6 need no run-time data, but their hard
questions (what happens to a disabled Step's outputs and dependents) are data-flow questions, so they are answered
here too. U8–U14 need per-Task scheduling and nothing else from this RFC; they belong to a separate extension (see
[Task-task dependencies are a separate extension](#12-task-task-dependencies-are-a-separate-extension)). U15 is the
territory of the standalone StepTemplate mechanism discussed in #22, and CONTROL_FLOW does not replace it.

U4 also shows that outputs are useful with no consumer at all, which is U16. A scheduler or UI that shows each
Task's metrics next to its parameters replaces the workarounds users reach for today. U16 was first proposed as a
separate `TASK_METRICS` extension with its own `openjd_metric:` directive; this RFC covers it with output variables
instead (see [Design Choices](#4-task-metrics-are-output-variables)).

U17 needs everything this RFC adds, and one thing it doesn't: a Step whose Task count isn't known when it is bound,
because each Task's parameters come from the Task before it. Today it can only be written by unrolling the loop into
a fixed number of copies of the Step, chained with *enableIf*. It is deferred to a separate extension (see
[Loop Steps](#loop-steps)).

U18 is the Job-level counterpart of U16. Today a caller learns a Job's results only from files at agreed-upon
locations, and a downstream system has to know the Job's internals to find them. A job output names each result, its
type, and for a `PATH`, where it is in the caller's own filesystem.

### Workarounds in use today

* **A pre-submission script** discovers the inputs and generates the template. It requires the submitter to be able
  to reach the data and to run code at submission, which job bundle submitters and web UIs can't do, and the template
  that reaches the scheduler no longer describes how it was made.
* **A Task that calls the farm's submit command** to add work. It isn't portable across schedulers, and the scheduler
  can't show the Job's shape until it happens.
* **Files and conventions**: each Task writes a file, and an aggregation Step reads them; or numbers are packed into
  `openjd_status` messages; or results go to an external tracker. None of these is visible to the scheduler or to
  tooling, and none can change how many Tasks a later Step has.

### Why now

EXPR (RFCs 0005–0007) supplies nearly everything this RFC needs: a type system with lists, nested lists, and optional
types; progressive evaluation with `unresolved[T]` placeholders, which type-checks an expression whose inputs are not
known yet; symbol collection for data-flow analysis; list-valued parameter ranges; and a function library with the
reductions (`min`, `max`, `sum`, `len`, `sorted`, comprehensions) that fan-in needs. What is missing is a way for a
Task to produce a value, a symbol to read it by, and a stage at which to evaluate expressions that depend on it.

### Backward compatibility

This RFC is additive and gated by the `CONTROL_FLOW` extension name declared under RFC 0002:

* `CONTROL_FLOW` requires `EXPR`. A template that lists `CONTROL_FLOW` must also list `EXPR`, and a scheduler must
  reject one that does not.
* No existing field changes meaning in a template that doesn't list `CONTROL_FLOW`. A scheduler that does not
  implement `CONTROL_FLOW` rejects a template that lists it, so a Job is never run with its outputs or conditions
  silently ignored.
* Within a template that lists `CONTROL_FLOW`, one existing rule is relaxed: a Task parameter range given as an
  expression may evaluate to an empty list (see [Empty parameter spaces](#empty-parameter-spaces)).
* A Step that declares no outputs, has no `enableIf`, and reads no `Steps.*` symbols behaves exactly as it does
  without the extension.

## Technical Requirements

1. **Backward compatibility.** Existing templates, and templates that list `CONTROL_FLOW` but don't use a given
   feature, behave exactly as they do today.
2. **Opt-in activation.** Every new field and symbol is available only when `CONTROL_FLOW` is listed in
   `extensions`, and `CONTROL_FLOW` requires `EXPR`.
3. **Fixed step graph.** The Steps of a Job, their names, their dependencies, and their output types are determined by
   the template and are the same for every Job created from it, whatever the parameter values or run-time data.
4. **Fail fast where possible.** Every reference to an output, every expression that uses one, and every type error
   in those expressions is caught at template validation, before submission. Only the values themselves, and the
   checks that depend on them, are deferred.
5. **Fail at the cause.** A value that violates its declared type or constraints fails the Task that produced it, not
   a consumer that reads it later. A consumer whose expressions fail on valid values fails before any of its Tasks
   runs.
6. **Deterministic results.** The value a consumer sees does not depend on the order in which the producer's Tasks
   ran or finished. A Job run twice on the same data has the same structure.
7. **Bounded.** Output values have a size limit with a guaranteed minimum, a computed parameter space can be bounded
   by a scheduler limit, and reductions are evaluated under EXPR's existing memory and operation limits.
8. **Strict emission.** An output value is set only by a directive that names a declared output and is well formed
   and well typed. Anything else that looks like a directive fails the Task rather than being ignored, truncated, or
   silently overriding a value, so a script bug or a stray log line is caught where it happens.
9. **Implementation strategy is the scheduler's choice.** A scheduler may create a waiting Step's Tasks only once its
   inputs are known, or create the Step first and fill it in. The specification states the observable contract.
10. **Composable.** The design composes with `TASK_CHUNKING`, `WRAP_ACTIONS`, and a future `TASK_DEPENDENCIES`
    extension without special cases in either direction.
11. **Prompt delivery.** A runtime can validate and forward each value as soon as the Task reports it, so the values
    are at the scheduler when the Task ends, and figures such as metrics can be shown while it runs.

## Design Choices

### 1. Variable data flow is part of CONTROL_FLOW, not a standalone extension

Discussion #111 framed the idea as variable data flow along step dependencies, and folouiseAWS suggested shipping it
in two parts: Task outputs reported to the scheduler first, data flow into downstream Steps second. An early outline
of this work also had a standalone `VAR_DATA_FLOW` extension. This RFC folds all of it into one `CONTROL_FLOW`
extension, for three reasons.

* **Outputs exist to steer the Job.** Every use case for passing data between Steps (U1–U4) passes it in order to
  change the Job's structure: its fan-out, its enablement, or the configuration a later Step runs. A data-flow
  extension without dynamic ranges or `enableIf` would solve none of them, and dynamic ranges or dynamic `enableIf`
  without data flow have nothing to read. The pieces are one feature.
* **The hard questions are shared.** What a skipped Step's outputs evaluate to, whether its dependents run, how an
  empty fan-out behaves, when expressions are evaluated: each of these involves both data flow and control flow.
  Every surveyed system that designed skip semantics separately from data flow has a pitfall at the join between
  them (see [Prior Art](#prior-art)).
* **Two extensions would freeze the smaller one.** RFC 0002 extensions are versioned by name. An outputs-only
  extension would have to stay as it is, and the data-flow half most users want would need a second name. Templates
  that use only outputs are a subset of templates that use the whole extension, so splitting buys nothing for
  authors. Implementations can still phase their work: a scheduler can build output collection and display first
  and advertise `CONTROL_FLOW` once binding works.

Static `enableIf` alone (U5, U6) is cheap and would close issue #80 sooner. It stays here because its semantics are
data-flow semantics, and because nothing in its syntax would change if it later had to be split out.

### 2. Outputs are reported with a stdout directive, with `@file` for large values

A Task reports an output by printing `openjd_output_var: <Name>=<JSON>`, as discussion #111 proposed, or
`openjd_output_var: <Name>=@<Path>` to take the value from a file. For three reasons:

* **One protocol.** Every way an Action talks to the runtime is already a stdout message: `openjd_progress`,
  `openjd_status`, `openjd_fail`, `openjd_env`. Runtimes already scan for them, `WRAP_ACTIONS` already forwards them,
  and logs already record them. A file channel would add a second contract: a file per attempt that the runtime must
  create, place, let the Job's user write, expose inside a container, and clean up.
* **Values arrive while the Task runs.** The runtime processes each directive when it reads the line, including
  reading an `@` file, so it can validate the value at once and may forward it to the scheduler. The values can be at
  the scheduler when the Task ends, with nothing left to read after it exits, which shortens the time between a producer
  finishing and its consumers being bound. A scheduler can also show a running figure, such as a metric, live.
* **`@file` removes the line-length limit.** Stdout lines are bounded (64 KiB in openjd-rs's `openjd-sessions`). A value of any
  size up to the per-Task limit goes through a file, and the directive that names it is short.

An earlier draft of this RFC used a session-supplied output file instead (`Task.OutputFile`, like GitHub Actions'
`$GITHUB_OUTPUT`), on the grounds that text printed by third-party tools could forge an output. That argument does
not carry over to OpenJD. Code in `onRun` already runs as the Job's user, with everything that user can do, so a value
it prints grants no capability it didn't have. GitHub's case differs: untrusted pull-request text in a log could steer
workflows that hold more privilege than its author. What remains is a correctness problem. A tool that echoes data,
such as an object key, at the start of a line might print something shaped like a directive. The rules in
[Output emission](#output-emission) handle it: a directive must name a declared output and be well formed and well
typed, and by default may appear only once per output (see
[Design Choices](#3-repeated-reports-follow-a-declared-collect-mode)). An accidental directive therefore fails the
Task instead of silently changing a value.

`@file` has one security rule. The runtime may run as a different operating-system user from the Action, with access
the Action's user lacks, so it must read the file with no more access than the Action's user has. Otherwise a Task
could copy a file it can't read into outputs that are stored and displayed. Separately, for portability, a `@` path
must resolve inside the Session's Working Directory, which also makes relative paths work the same inside a container
under `WRAP_ACTIONS`.

### 3. Repeated reports follow a declared collect mode

A Task can print a directive for the same output more than once, by design or by mistake. Each output declares which
it expects, with `collect`:

* **`ONCE`**, the default: exactly one directive. Most outputs are computed once, and a second directive means a
  print inside a loop, two code paths that both report, or a stray line. It fails the Task.
* **`LAST`**: one or more directives, and the last is the value. A long Task can update a running figure, such as the
  frames done or the current loss, which a scheduler can show as it changes.
* **`APPEND`**: zero or more directives, collected into a list in the order printed. The output's type is `LIST[T]`,
  and each directive supplies one `T`. A Task that evaluates 10 episodes or renders 50 frames reports one value per
  item as it goes.

`APPEND` adds one element per directive, so a list printed under `APPEND` becomes one element of a list of lists,
never a batch to splice in. A mode that concatenated list values would make a `LIST[LIST[INT]]` output ambiguous, and
is not needed: a Task with a batch of values can print one directive per value, or report the whole list once.

No mode reduces values at the producer (a sum, minimum, or count). Consumers reduce with EXPR (see
[Design Choices](#10-reductions-are-ordinary-expr-over-ordered-lists)), and keeping every value is what a Task that
reports measurements wants. A reducing mode can be added later if the size of collected lists becomes a problem.

### 4. Task metrics are output variables

Reporting per-Task measurements, such as an error score, coverage, frames per second, or render time, was first
proposed as a separate extension with its own directive, `openjd_metric: <name>=<number>`. In this RFC a metric is an
output variable that no Step reads. It is declared with a type and constraints, has an optional `unit` and
`description` for display, and is reported with `openjd_output_var:`. A scheduler stores and can show every Task's
committed outputs whether or not a Step reads them (see [Committing values](#committing-values)), so the same
declaration gives a UI its columns and, if a later Step needs it, gives that Step its input (see the sweep examples).

One mechanism instead of two means one directive, one set of validation rules, and one way of storing values per Task.
A metric that later feeds a Step needs no change to the producer. The costs:

* **Metrics follow the strict rules.** An undeclared name or a malformed value fails the Task, where the metrics
  proposal ignored it with a warning. A Task whose result is a measurement, as in a sweep, has produced a bad result
  if its measurement is bad, and the author fixes the script once. A lenient mode for outputs that no Step reads would
  be a second set of rules.
* **Metrics need the whole extension.** A scheduler that wants only metrics must still accept `CONTROL_FLOW`, which
  also brings step input binding. As
  [Design Choices](#1-variable-data-flow-is-part-of-control_flow-not-a-standalone-extension) notes, it can build output
  collection and display first and advertise `CONTROL_FLOW` once binding works.

How results are read, such as whether lower or higher is better, a threshold such as "coverage at least 75", or
averaging over one Task parameter, stays out of the template. Those choices describe how a person reads the results,
often change after the Job has run, and may reasonably differ between schedulers.

### 5. A new `Steps` root, with `Output` and `TaskParam` levels

#111 wrote references as `Step.<StepName>.<VarName>`. `Step` already means "the current Step" in EXPR
(`Step.Name`), so reusing it would make a Step named `Name` unreachable and turn every future `Step.<Property>` into a
breaking change. RFC 0008 avoided the same problem with a separate `WrappedStep` root. This RFC uses:

* `Steps.<StepName>.Output.<VarName>` for an output value,
* `Steps.<StepName>.TaskParam.<ParamName>` for the producer's Task parameter values, aligned with its output lists,
  and
* `Steps.<StepName>.RawOutput.<VarName>` and `Steps.<StepName>.TaskRawParam.<ParamName>` for the same values with no
  path mapping applied, as `RawParam.*` is to `Param.*` (see
  [Design Choices](#14-path-values-are-stored-in-the-submission-path-context)).

The `Output` level costs one word per reference. It leaves room for `TaskParam` and for later per-Step symbols such as
a Task count or a parameter-space introspection (#82 proposed `Step.<Name>.ParameterSpace.<Param>.Range`) without any
risk of collision with user-chosen output names.

Dotted access can't name a Step called `Render Frames`, so a Step referenced through `Steps.` must have an
`<Identifier>` name. That is a validation error on the referencing expression, not a new rule for every Step name.

### 6. Only direct dependents may read a Step's outputs

A Step may read `Steps.P.*` only if it lists `P` in its own `dependencies`. Transitive access would save a line in a
long chain, but it would hide data flow from a reader and from tools, which would have to compute reachability to know
what a Step consumes. The extra `dependsOn` never changes scheduling, since a transitive dependency already orders the
Steps. A reference does not create an edge: a missing edge is a validation error that names the fix. GitHub Actions
(`needs`) and Argo make the same choice, and it keeps the template's explicit `dependencies` the only source of
edges.

Job outputs are the exception: a job output's *value* may read any Step. A job output is not scheduled, so there is
no edge to declare, and the Job is the consumer of all of its Steps.

### 7. Output shape is determined by the template

`Steps.P.Output.X` has the declared type `T` if `P` has no `parameterSpace`, and `list[T]` in iteration order if it
has one, even if that space happens to contain one Task or none. The type is therefore known at template validation,
which EXPR's static type checking requires, and a reader can tell the shape of a value from the producer's definition
alone. #111 Example 1 (a scalar `FrameCount` from a single-Task Step) and the sweep's per-Task lists both fall out of
this rule.

### 8. A skipped Step satisfies ordering and yields `null` data

A Step whose `enableIf` is false is `SKIPPED`. It satisfies `dependsOn`, so its dependents run, and every
`Steps.<It>.*` symbol is `null`. If a Step has an `enableIf`, its `Steps.*` symbols are typed `T?` everywhere, so a
consumer handles `null` where a reader can see it: `Steps.P.Output.X or 0`, a conditional expression, a `fail(...)`,
or an `enableIf` of its own.

The alternative most often proposed is Tekton's: a dependent that reads a skipped Step's outputs is itself skipped.
That makes enablement flow along an edge that is implicit in expression text, and its pitfall ("why did my publish
step vanish?") is the join problem every surveyed system has. A consumer that wants Tekton's behavior gets it in one
line: `enableIf: "{{Steps.P.Output.X != null}}"`. The rule is applied to the skipped Step only: a Step that runs
after a skipped Step behaves normally, and its own outputs keep type `T`.

"Satisfies ordering" means the skipped Step stands in for a Step that finished instantly at the moment it would have
started. Its dependents wait for its own dependencies, so skipping a Step in the middle of a chain never lets the end
of the chain start early.

EXPR's type checker accepts a `T?` placeholder where `T` is required and defers the check to the concrete value
(RFC 0005, *Coercion of Unresolved Values*). So misuse of a skipped Step's `null` is caught when the consumer binds,
or when its Task starts, not at validation. Both are loud failures that name the expression. The silent cases are the
places where EXPR discards a `null` by design: text interpolation, where it becomes `""`
(`"--size {{Steps.P.Output.X}}"` becomes `--size `); a list item such as an `args` entry, which is dropped; and an
optional field, which is omitted. This RFC requires a validation warning for each, and makes *enableIf* treat `null`
as an error rather than as an omitted field, so that a skipped producer can never silently enable a Step.

### 9. An empty computed parameter space is a zero-Task success

If a Step's computed parameter space is empty, the Step succeeds with zero Tasks and its `Output` lists are `[]`.
Treating it as skipped instead (as Airflow and Argo do) would make every Step with a computed parameter space
potentially skipped, so all of its `Steps.*` symbols would have to be typed `T?`, which would push `null` handling
into every reduction. As a success, `sum([])` is `0` and the empty manifest is just empty. An author who considers
emptiness an error says so with `minLength: 1` on the producer's output or a `fail(...)` in the range expression.

### 10. Reductions are ordinary EXPR over ordered lists

There are no declared reductions (`reduce: MAX` on an output, for example). A multi-Task producer's output is the full
list in parameter-space iteration order, and the consumer reduces it with RFC 0006 functions. (`collect: APPEND`
collects the values one Task reports into that Task's value; it doesn't reduce across Tasks.) Keeping the full list
loses nothing, lets one consumer take the `max` while another takes the `sum`, and lets the sweep correlate three
metrics with the configuration that produced them, which no scalar reduction can do. `Steps.P.TaskParam.*` provides
the configuration side of that correlation without any new EXPR type.

The order is the iteration order the producer's `combination` defines, not the order in which Tasks finished, so the
result is deterministic.

### 11. Limits are enforced at the producer

Output values are bounded per Task, and a producer's declared constraints (`maxLength: 5000`, `minValue: 1`) are
checked as its Tasks report values. A discovery Step that finds a million keys fails itself, next to the cause, with
a message naming the constraint, rather than producing a consumer that fails later with a less obvious error.
Airflow's `max_map_length`, which fails the producing task, is the precedent.

### 12. Task-task dependencies are a separate extension

Per-Task dependencies (#82 `dependsOnSubspace`, self-dependencies, `anyOf`) go in their own extension, provisionally
`TASK_DEPENDENCIES`, which neither depends on CONTROL_FLOW nor is required by it. Seven of the eight task-task use cases
(U8–U14) are fully static and need nothing from this RFC; bundling them would make render pipelines wait for data flow,
or data-flow users wait for per-Task scheduling. For a scheduler the costs are independent and both large: this RFC
needs value storage and a binding stage, and task-task dependencies need per-Task readiness instead of per-Step
readiness. Their open questions don't overlap either.

Composition needs no new syntax. A subspace range is a TEMPLATE-scope field of the dependent Step, so under this RFC's
rules it is evaluated at binding and may read `Steps.*`, which is all the computed DAG of U2 requires (see
[Composition with `TASK_DEPENDENCIES`](#composition-with-task_dependencies)).

### 13. Job outputs are expressions over Step outputs

A job output is declared at the Job level with a *value* expression, and is never written by a Task. GitHub Actions
reusable workflows (`outputs.<name>.value`), Tekton Pipeline `results`, CWL workflow `outputs` (`outputSource`), and
Argo workflow outputs all take this shape. It keeps one reporting mechanism, `openjd_output_var:`, and makes the Job
one more consumer of Step outputs, with the reductions of EXPR available to it. Three further choices:

* **Evaluated as soon as possible**, one output at a time, so a user interface can show `VideoCount` long before the
  renders that follow it finish. An output that reads only job parameters, such as the results directory, is known
  at Job creation.
* **A failure fails the Job.** A job output that can't be evaluated means the Job can't report what it promised, and
  finding that out at once, at its cause, beats finding it out at the end. This matches the strict handling of
  invalid directives (see [Processing a directive](#processing-a-directive)).
* **Not a symbol in the template.** Nothing in a Job runs after its outputs are known, so no format string can use
  them. They are for callers, including a future mechanism that feeds one Job's outputs into another's parameters,
  for which the shared type vocabulary is the point.

### 14. `PATH` values are stored in the submission path context

A `PATH` that a Task reports names a location on its worker. A consumer on another host, or the submitter reading a
job output, needs it in their own form. This RFC stores every such value in the form `PATH` job parameters are given
in, the submission path context, by applying the producing Session's path mapping rules in reverse, and maps it
forward for each consumer, as a `PATH` job parameter is mapped. Path mapping thus stays the responsibility of the
runtime and scheduler that already have the rules, and `PATH` values behave the same whether they started as job
parameters or Task outputs. How a runtime computes the inverse is up to it, provided the stored value maps forward
correctly. Deadline Cloud, for example, reverses the mapping from the worker's context to the job's submission context.

A path for which the runtime can find no submission-context form is invalid when the Session has rules, because it
names a location that only that worker can see, most often the Session Working Directory, which is deleted when the Session ends. When a Session has no
rules, as with `openjd run` on one host, the worker and submission contexts are the same and an absolute value is
stored unchanged.

Read in a format string resolved on the worker host, `Steps.P.Output.X` has the consuming Session's rules applied, as
`Param.X` does. Everywhere else the unmapped value is read as `Steps.P.RawOutput.X`, as `RawParam.X` is. The same
pair, `TaskParam` and `TaskRawParam`, applies to a producer's `PATH` Task parameters.

### 15. Design tenets

* **Portable.** A scheduler that doesn't implement the extension rejects the template rather than mis-running it. The
  directive is the stdout protocol every runtime already implements, on Linux, macOS, and Windows and through
  `WRAP_ACTIONS`, and a relative `@` path resolves against the Session's Working Directory wherever the Action runs.
  The observable contract of
  binding allows both deferred-creation and create-then-update schedulers. The tension is that this is the most
  demanding extension for schedulers so far (persisted values, a new stage, two new states); keeping task-task
  dependencies out of it keeps the minimum smaller.
* **Expressive.** One mechanism, typed outputs read by EXPR expressions at a defined stage, covers fan-out,
  conditions, reductions, and (with `TASK_DEPENDENCIES`) computed DAGs. The fixed step graph rules out PDG-style work
  by design, as #22 accepted.
* **Human readable and writable.** `outputVariables` declares what flows, and `Steps.P.Output.X` reads plainly. The
  tensions: correlating results with configurations needs index arithmetic until `argmin`/`argmax` or similar land;
  `Steps.P.Output.X` is longer than #111's `Step.P.X`; a directive's value must be JSON, so a shell script that
  reports arbitrary text needs a JSON encoder such as `jq`; `"1-{{N}}"` and `"{{range(1, N + 1)}}"` behave differently for `N = 0`; and EXPR silently discards a
  skipped Step's `null` in text interpolation, list items, and optional fields, which the validation warning
  mitigates.
* **Tooling parseable.** Outputs and their types are declared, references and edges are explicit and checked before
  submission, and the directive has a strict grammar. This replaces opaque workarounds. The tension is that a Job's
  Task count is no longer always known at submission, which #22 warned about. The fixed step graph, declared
  constraints that bound fan-out, and the requirement to report pending Steps explicitly limit the loss.

## Specification

### Extension name and dependencies

The extension name is `CONTROL_FLOW`. Every field, symbol, and behavior in this section is available only in a Job
Template whose `extensions` list includes `CONTROL_FLOW`.

* A template that lists `CONTROL_FLOW` MUST also list `EXPR`. A scheduler or validator MUST reject a template that
  lists `CONTROL_FLOW` without `EXPR`. RFC 0002 does not define dependencies between extensions; this is a rule of the
  `CONTROL_FLOW` extension, as the same rule is for `WRAP_ACTIONS`.
* `CONTROL_FLOW` applies to Job Templates. An Environment Template cannot declare it, and nothing in an Environment
  Template can reference the symbols it adds.

### Terminology

| Term | Definition |
|------|------------|
| **Producer** | A Step whose `Steps.*` symbols another Step reads. A Step that declares `outputVariables` is a producer of `Steps.*.Output`; any Step with a `parameterSpace` can be a producer of `Steps.*.TaskParam`. |
| **Consumer** | A Step that reads a producer's `Steps.*` symbols. It must list the producer directly in its `dependencies`. |
| **Output variable** | A typed value declared in a Step's `outputVariables` and reported by each of its Tasks. A **metric** is an output variable that no Step reads. |
| **Output directive** | A line `openjd_output_var: <Name>=<JSON>` or `openjd_output_var: <Name>=@<Path>` that a Task's `onRun` action prints to stdout or stderr to report an output value. |
| **Collect mode** | The `collect` property of an output variable, `ONCE`, `LAST`, or `APPEND`, which says how a Task's directives for that name become its value. |
| **Materialization expressions** | The format strings of a Step that the scheduler must evaluate to know whether the Step runs and what its Tasks are. See [Step input binding](#step-input-binding). |
| **Step input binding** | The evaluation stage, once per Step, at which the Step's materialization expressions are evaluated with every `Steps.*` symbol they reference concrete. |
| **Conditional Step** | A Step that has an `enableIf`. |
| **Job output** | A typed value declared in the Job Template's `jobOutputDefinitions`, computed from job parameters and `Steps.*`, and reported to the Job's callers. |
| **Submission path context** | The form in which `PATH` job parameter values are given when a Job is submitted, before any Session's path mapping rules are applied. `PATH` output values are stored in this form. |
| **`PENDING_INPUTS`** | The state of a Step whose materialization expressions reference a `Steps.*` symbol that is not concrete yet. |
| **`SKIPPED`** | The terminal state of a Step whose `enableIf` evaluated to `false`. |

### Schema modifications

> Changes to [the template schema](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas).

#### Job Template

> A modification to [`1.1. Job Template`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#11-job-template)

```diff
  specificationVersion: "jobtemplate-2023-09"
  $schema: <string> # @optional
  extensions: [ <ExtensionName>, ... ] # @optional
  name: <JobName> # @fmtstring
  description: <Description> # @optional
  parameterDefinitions:  [ <JobParameterDefinition>, ... ] # @optional
+ jobOutputDefinitions: [ <JobOutputDefinition>, ... ] # @optional @extension CONTROL_FLOW
  jobEnvironments: [ <Environment>, ... ] # @optional
  steps: [<StepTemplate>, ...]
```

Extension list addition to item 3 (*extensions*): `CONTROL_FLOW` (this RFC). New property:

* *jobOutputDefinitions* — The Job's outputs: typed values computed from the Job's parameters and its Steps' outputs,
  for the submitter and other callers to read. See [`<JobOutputDefinition>`](#joboutputdefinition) and
  [Job outputs](#job-outputs). Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. Maximum number of elements: 50, whatever extensions are enabled.
    3. No two elements in this list may have the same value for the `name` property. Job output names and job
       parameter names are separate namespaces, so a job output may have the same name as a job parameter.

#### `<StepTemplate>`

> A modification to [`3. <StepTemplate>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#3-steptemplate)

```diff
  name: <StepName>
  description: <Description> # @optional
+ enableIf: <bool> | <FormatString> # @optional @fmtstring @extension CONTROL_FLOW
  let: <LetBindings> # @optional @extension EXPR
  dependencies: [ <StepDependency>, ... ] # @optional
  stepEnvironments: [ <Environment>, ... ] # @optional
  hostRequirements: <HostRequirements> # @optional
  parameterSpace: <StepParameterSpaceDefinition> # @optional
+ outputVariables: [ <OutputVariableDefinition>, ... ] # @optional @extension CONTROL_FLOW
  script: <StepScript>
```

New properties:

* *enableIf* — Whether the Step runs. Although the field is optional, the expression's target type is `bool`, not
  `bool?`: an expression that evaluates to `null` is an error, not an omitted field, so a skipped producer's `null`
  can never silently enable a Step. If it evaluates to `false`, the Step is `SKIPPED`: none of its Tasks runs, its
  Step Environments are not entered, its `Steps.*` symbols are `null`, and it satisfies the `dependsOn` of the Steps
  that depend on it once its own dependencies are satisfied. If omitted, the Step is always enabled. See
  [Conditional execution](#conditional-execution-enableif). Constraints:
    1. It may reference `Param.*` (except `PATH` parameters, which are available only on the worker host),
       `RawParam.*`, `Job.Name`, `Step.Name`, and the `Steps.*` symbols of the Step's direct dependencies (see
       [The `Steps.*` scope](#the-steps-scope)).
    2. It may not reference names bound by the Step's own `let`, `Task.*`, or `Session.*`.
* *outputVariables* — The output variables that every Task of the Step reports with `openjd_output_var:` directives.
  See [`<OutputVariableDefinition>`](#outputvariabledefinition) and [Output emission](#output-emission). Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. Maximum number of elements: 32.
    3. No two elements in this list may have the same value for the `name` property.

  In a Step whose parameter space contains a `CHUNK[INT]` Task parameter, one `onRun` runs a whole chunk of Tasks,
  and each directive names the Task it reports for (see [Chunked Steps](#chunked-steps)).

Changed semantics of existing properties:

* *dependencies* — A dependency on a `SKIPPED` Step, or on a Step that succeeded with zero Tasks, is satisfied once
  that Step's own dependencies are satisfied (see [`<StepDependency>`](#stepdependency)). Listing a producer here
  is also what makes its `Steps.*` symbols visible to this Step.
* *let*, *hostRequirements*, *parameterSpace*, *stepEnvironments*, *script* — These fields may reference the `Steps.*`
  symbols of the Step's direct dependencies. A format string among them that is resolved at Job creation (not
  annotated `@fmtstring[host]`) and references `Steps.*` is evaluated at step input binding instead. See
  [Step input binding](#step-input-binding).

##### `<OutputVariableDefinition>`

An `<OutputVariableDefinition>` is the object:

```yaml
name: <Identifier>
type: <OutputVariableType>
collect: <OutputCollectMode> # @optional
description: <Description> # @optional
unit: <OutputVariableUnit> # @optional
# Constraint properties, @optional. The same names and meanings as the
# <JobParameterDefinition> of the same type: see the table below.
```

```bnf
<OutputVariableType> ::= "INT" | "FLOAT" | "STRING" | "PATH" | "BOOL" | "RANGE_EXPR" |
                         "LIST[INT]" | "LIST[FLOAT]" | "LIST[STRING]" | "LIST[PATH]" | "LIST[BOOL]" |
                         "LIST[LIST[INT]]"
<OutputCollectMode>  ::= "ONCE" | "LAST" | "APPEND"
```

Where:

1. *name* — The name by which the output is reported in an `openjd_output_var:` directive and referenced as
   `Steps.<StepName>.Output.<name>`. See [`<Identifier>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#71-identifier).
2. *type* — The type of the value each Task reports. Type names are case-insensitive, as job parameter type names are
   with `EXPR` enabled, and each maps to the expression type of the job parameter type of the same name (RFC 0007):

   | `<OutputVariableType>` | Expression type of one Task's value | Constraint properties, as on |
   |---|---|---|
   | `INT` | `int` | `<JobIntParameterDefinition>`: `allowedValues`, `minValue`, `maxValue` |
   | `FLOAT` | `float` | `<JobFloatParameterDefinition>`: `allowedValues`, `minValue`, `maxValue` |
   | `STRING` | `string` | `<JobStringParameterDefinition>`: `allowedValues`, `minLength`, `maxLength` |
   | `PATH` | `path` | `<JobPathParameterDefinition>`: `allowedValues`, `minLength`, `maxLength`, `objectType` |
   | `BOOL` | `bool` | `<JobBoolParameterDefinition>`: none |
   | `RANGE_EXPR` | `range_expr` | `<JobRangeExprParameterDefinition>`: `minLength`, `maxLength` |
   | `LIST[INT]` | `list[int]` | `<JobListIntParameterDefinition>`: `minLength`, `maxLength`, `item` |
   | `LIST[FLOAT]` | `list[float]` | `<JobListFloatParameterDefinition>`: `minLength`, `maxLength`, `item` |
   | `LIST[STRING]` | `list[string]` | `<JobListStringParameterDefinition>`: `minLength`, `maxLength`, `item` |
   | `LIST[PATH]` | `list[path]` | `<JobListPathParameterDefinition>`: `minLength`, `maxLength`, `item`, `objectType` |
   | `LIST[BOOL]` | `list[bool]` | `<JobListBoolParameterDefinition>`: `minLength`, `maxLength` |
   | `LIST[LIST[INT]]` | `list[list[int]]` | `<JobListListIntParameterDefinition>`: `minLength`, `maxLength`, `item` |

3. *collect* — How the directives a Task prints for this output become the Task's value. Default `ONCE`. See
   [Collect modes](#collect-modes).
    * `ONCE` — Exactly one directive.
    * `LAST` — One or more directives; the last one is the value.
    * `APPEND` — Zero or more directives, each supplying one element of the declared `LIST[T]` type, in the order
      printed.
4. *description* — A description of the output. It has no functional purpose, but may appear in UI elements.
5. *unit* — The unit of the output's values, such as `mm`, `%`, or `frames/s`, for display. It has no functional
   purpose: values are not converted. A string of 1 to 32 characters, none of which is in the Unicode Cc (control)
   category.
6. Constraint properties — Checked against each Task's value as the Task reports it (see
   [Processing a directive](#processing-a-directive)). A value that violates one fails the Task. Only the properties
   the definition declares are checked: a default that a job parameter definition implies when a property is omitted (such as the
   1024-character default `maxLength` of a `RANGE_EXPR` parameter) does not apply. The `default` and `userInterface`
   properties of job parameter definitions are not available.

Constraints:

1. `LIST[LIST[INT]]` may only be declared by a Step without a `parameterSpace`. EXPR allows two levels of list
   nesting, and a multi-Task producer's outputs are collected into a list (see [Reductions](#reductions)).
2. A constraint property must be valid for the declared type, with the same validity rules as on the job parameter
   definition of that type (for example, `minValue` may not exceed `maxValue`).
3. `collect: APPEND` requires a `LIST[...]` type.

A `PATH` or `LIST[PATH]` value is stored in the submission path context, and its constraint properties are checked
against that stored form (see [Path values](#path-values)). `objectType` (`FILE` or `DIRECTORY`) describes the value
for tools and user interfaces and is not checked. `dataFlow` is not available: an output is by nature produced by
the Job.

> **TODO (deferred): producer-declared defaults.** A `default` for an output that a Task doesn't report, or that a
> skipped Step would otherwise leave `null`, was proposed by Argo's `valueFrom.default` and Tekton's TEP-0048. The
> `or` operator and conditional expressions cover the use cases at the consumer. Adding an optional `default` later is
> backward compatible.

##### `<JobOutputDefinition>`

A `<JobOutputDefinition>` is the object:

```yaml
name: <Identifier>
type: <OutputVariableType>
value: <FormatString> # @fmtstring
description: <Description> # @optional
unit: <OutputVariableUnit> # @optional
# Constraint properties, @optional, as on <OutputVariableDefinition>.
```

Where:

1. *name* — The name by which a caller reads the output. Job outputs are not symbols in any format string of the
   template: nothing in a Job runs after them.
2. *type* — The output's type, from the same vocabulary as `<OutputVariableDefinition>`, with the same mapping to
   expression types and the same constraint properties (including `objectType` for `PATH` and `LIST[PATH]`).
   `LIST[LIST[INT]]` is allowed.
3. *value* — The value, as a format string whose target type is the expression type of *type*, made optional: an
   expression that reads a conditional Step can evaluate to `null`, which means the output has no value. It may
   reference `Param.*` (except `PATH` and `LIST[PATH]` parameters), `RawParam.*`, `Job.Name`, and the `Steps.*`
   symbols of any Step of the Job, with values whose type is or contains `path` read through `RawOutput` and
   `TaskRawParam`. It may not reference `Step.Name`,
   `Task.*`, `Session.*`, or names bound by `let`.
4. *description* and *unit* — For display, as on `<OutputVariableDefinition>`.
5. Constraint properties — Checked against the value when it is evaluated. A violation fails the Job. A `null`
   value is not checked.

There is no `collect`: a job output is evaluated, not reported. See [Job outputs](#job-outputs).

##### `<StepDependency>`

> A modification to [`3.2. <StepDependency>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#32-stepdependency)

The object is unchanged. With `CONTROL_FLOW`, the dependency is satisfied when the depended-upon Step has completed
successfully, which includes completing with zero Tasks, or has been `SKIPPED` and its own dependencies are
satisfied. The second condition keeps ordering transitive: if `C` depends on `B` and `B` on `A`, then `C` never starts
before `A` has finished, whether or not `B` runs. A dependency on a Step that failed, or that can never run because one
of its own dependencies failed, is never satisfied, as today.

##### Task parameter ranges

> A modification to [`3.4.1. <TaskParameterDefinition>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#341-taskparameterdefinition)
> and to EXPR's [Task Parameter Range Field Extensions](https://github.com/OpenJobDescription/openjd-specifications/wiki/2026-02-Expression-Language#1312-task-parameter-range-field-extensions)

The schema of each `<TaskParameterDefinition>` is unchanged. With `CONTROL_FLOW`, a `range` that is a format string may
reference the `Steps.*` symbols of the Step's direct dependencies, which makes it a materialization expression
evaluated at step input binding. The `combination` expression may not reference `Steps.*`: the shape of a parameter
space (which parameters it has and how they combine) is fixed by the template, even when its size is not.

```yaml
range: "1-{{Steps.VideoToFrames.Output.FrameCount}}"     # INT, a range string
range: "{{Steps.Discover.Output.Keys}}"                  # STRING, a ListExpressionString
range: "{{range(len(Steps.Analyze.Output.Children))}}"   # INT, list[int]; zero Tasks if the list is empty
```

###### Empty parameter spaces

With `CONTROL_FLOW`, a `range` given as a format string may evaluate to an empty list, for any Task parameter type.

* If the Step's `combination` combines that parameter by product (`*`, the default), the parameter space is empty.
* If it combines it by association (`(A, B)`), the usual rule that associated ranges have equal lengths applies: an
  empty range associated with a non-empty one is an error at binding, and associated empty ranges give an empty
  space.

A Step with an empty parameter space has zero Tasks and completes successfully as soon as its own dependencies are
satisfied. Its `Steps.*.Output` and `Steps.*.TaskParam` lists are `[]`.

This applies to list results only. A range string has no empty value: `"1-{{N}}"` with `N = 0` is the range `1-0`,
which §3.4.1.1.1 defines as `[1]` (some implementations reject it instead). An author who wants zero Tasks for
`N = 0` writes `"{{range(1, N + 1)}}"`, which is a valid empty list. A `range` written as a YAML list literal must
have at least one item as written; after its items are evaluated (where EXPR flattens a list-valued item), the
result follows the same rules as a `range` given as a format string, and so may be empty.

###### Range length

With `CONTROL_FLOW`, the 1024-element maximum of §3.4.1 applies to the items of a `range` written as a YAML list
literal, counted as written, and not to the values a format string evaluates to, which follows the rule §3.4.1.1.1 already states for `<IntRangeExpr>`: the
number of values is not constrained by this specification, and a scheduler that needs to bound it does so with its
own limit on the Task count of a Step (see [Limits](#limits)). Computed fan-out is the purpose of this extension, and
a discovered list of 5,000 object keys is ordinary.

#### `<Environment>`

> A modification to [`4. <Environment>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#4-environment)

```diff
  name: <EnvironmentName>
  description: <Description> # @optional
+ enableIf: <bool> | <FormatString> # @optional @fmtstring @extension CONTROL_FLOW
  script: <EnvironmentScript> # @optional
  variables: <EnvironmentVariables> # @optional
```

New property:

* *enableIf* — Whether the Environment is entered. The expression's target type is `bool`, and a `null` result is an
  error, as for a Step's *enableIf*. If it evaluates to `false`, the Environment is disabled: it is not entered or exited in any Session, its `variables` are not set, and
  it is not part of the Session's Environment stack. If omitted, the Environment is always enabled. Constraints:
    1. In a Job Environment (`jobEnvironments`), it may reference `Param.*` (except `PATH` parameters), `RawParam.*`,
       and `Job.Name`, and is evaluated at Job creation.
    2. In a Step Environment (`stepEnvironments`), it may also reference `Step.Name` and the `Steps.*` symbols of the
       enclosing Step's direct dependencies, and is evaluated with the Step's other materialization expressions.
    3. It may not reference `Session.*`, `Env.*`, or names bound by `let`.
    4. Permitted only in a Job Template. An `<Environment>` in an Environment Template may not have *enableIf*.

> **TODO (deferred): `enableIf` on individual host requirements.** #81 proposed it for CPU-only vs. GPU templates
> (U7). EXPR already covers that case: an amount requirement's `min` can be computed (a `min` of `0` imposes nothing)
> and an attribute requirement's `anyOf` list can be computed. Adding *enableIf* to `<AmountRequirement>` and
> `<AttributeRequirement>` later is backward compatible.

#### Format strings

> A modification to [`7.3. Format Strings`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#73-format-strings)

##### The `Steps.*` scope

CONTROL_FLOW adds four symbol families to the value references of §7.3.1. The `Raw` forms relate to the others as
`RawParam.*` relates to `Param.*`:

| Value | Type | Description |
|---|---|---|
| `Steps.<StepName>.Output.<VarName>` | See below | The value of output variable `<VarName>` of Step `<StepName>`. A `PATH` value has the consuming Session's path mapping rules applied. |
| `Steps.<StepName>.RawOutput.<VarName>` | As `Output` | The same value with no path mapping applied: a `PATH` value in the submission path context (see [Path values](#path-values)). For other types, identical to `Output`. |
| `Steps.<StepName>.TaskParam.<ParamName>` | See below | The value of Task parameter `<ParamName>` for every Task of Step `<StepName>`, in iteration order. A `PATH` value has the consuming Session's path mapping rules applied. |
| `Steps.<StepName>.TaskRawParam.<ParamName>` | As `TaskParam` | The same values with no path mapping applied. For other types, identical to `TaskParam`. |

**Types.** Let `T` be the expression type of one Task's value of output `<VarName>` (from the table in
[`<OutputVariableDefinition>`](#outputvariabledefinition)), and `U` the expression type of Task parameter
`<ParamName>` for one Task:

| Producer `<StepName>` | `Steps.<StepName>.Output.<VarName>` | `Steps.<StepName>.TaskParam.<ParamName>` |
|---|---|---|
| no `parameterSpace`, no `enableIf` | `T` | (validation error: no Task parameters) |
| no `parameterSpace`, has `enableIf` | `T?` | (validation error) |
| has `parameterSpace`, no `enableIf` | `list[T]` | `list[U]` |
| has `parameterSpace`, has `enableIf` | `list[T]?` | `list[U]?` |

The `RawOutput` and `TaskRawParam` forms have the same types as `Output` and `TaskParam`. `U` is `int`, `float`,
`string`, or `path` for a Task parameter of type `INT`, `FLOAT`, `STRING`, or `PATH`. For a
`CHUNK[INT]` Task parameter, `U` is `int`: the list has one element per Task, not per chunk, because the Tasks of a
Step are defined before they are grouped into chunks.

Element `i` of an `Output` list and element `i` of every `TaskParam` list of the same Step belong to the same Task:
the `i`-th Task in the Step's parameter-space iteration order (see [Reductions](#reductions)).

**Where available.** A `Steps.<StepName>.*` symbol is available in a format string of Step `S` if and only if all of
the following hold:

1. `S` lists `<StepName>` in its own `dependencies` as a whole-Step dependency. A transitive dependency is not
   enough, and with `TASK_DEPENDENCIES` a dependency that selects a subspace of the producer's Tasks is not enough.
2. `<StepName>` is not `S` itself.
3. The format string is in one of `S`'s *enableIf*, *let*, *hostRequirements*, *parameterSpace* Task parameter
   `range`s, *stepEnvironments* (any format string, including their *enableIf*), or *script*.

In addition, the *value* of a [`<JobOutputDefinition>`](#joboutputdefinition) may read the `Steps.*` symbols of any
Step of the Job, with no dependency required (see [Job outputs](#job-outputs)).

`Steps.*` is not available in the Job Template's `name`, in `jobEnvironments`, in `parameterDefinitions`, in a
`combination` expression, or anywhere in an Environment Template.

A `PATH` or `LIST[PATH]` value of `Steps.<StepName>.Output.<VarName>` or `Steps.<StepName>.TaskParam.<ParamName>` is
available only in format strings resolved on the worker host (`@fmtstring[host]`), as `PATH` `Param.*` values are,
because path mapping depends on the consuming Session. Format strings resolved earlier, including the materialization
expressions and job output *value*s, use `RawOutput` and `TaskRawParam` instead, as they use `RawParam.*` for a job
parameter. A consumer's `PATH` Task parameter whose `range` comes from `RawOutput` or `TaskRawParam` holds
submission-context values, and its `Task.Param.*` is mapped forward on the worker host, as a range from `RawParam.*`
is.

`<StepName>` must be an `<Identifier>`: a Step whose name is not of that form cannot be referenced through `Steps.`
(the Step itself is still valid). `<VarName>` must name an output declared by the Step, and `<ParamName>` a Task
parameter of the Step.

**Concreteness.** At template validation every `Steps.*` symbol is an `unresolved` value of the type above, so every
expression that uses one is type-checked by EXPR's existing progressive evaluator. The value becomes concrete:

* `Steps.<StepName>.Output.<VarName>` and `RawOutput.<VarName>` — when Step `<StepName>` is terminal and has not
  failed: `SUCCEEDED` (with the values of all of its Tasks) or `SKIPPED` (with the value `null`).
* `Steps.<StepName>.TaskParam.<ParamName>` and `TaskRawParam.<ParamName>` — when Step `<StepName>`'s parameter space has been materialized and its
  `enableIf`, if any, has been decided. For a producer whose parameter space and `enableIf` read no `Steps.*`, that is
  at Job creation.

##### Template processing stages

> A modification to [`7.4. Template Processing Stages`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#74-template-processing-stages)

| Stage | When | Known Values | Unknown Values | What Happens |
|---|---|---|---|---|
| **Template validation** | as today | as today | as today, and `Steps.*` | With `CONTROL_FLOW`: `Steps.*` symbols are `unresolved` with the types of [The `Steps.*` scope](#the-steps-scope), and every rule of [Validation](#validation) is checked. |
| **Job creation** | as today | as today | as today, and `Steps.*` symbols that are not yet concrete | With `CONTROL_FLOW`: an *enableIf* that reads no `Steps.*` is evaluated. A Step whose materialization expressions read no unresolved `Steps.*` is bound now; every other Step enters `PENDING_INPUTS`. A job output that reads no unresolved `Steps.*` is evaluated. |
| **Step input binding** (new, `CONTROL_FLOW` only) | Per Step, once every `Steps.*` symbol its materialization expressions reference is concrete | `Param.*`, `RawParam.*`, the referenced `Steps.*` | `Task.*`, `Session.*` | The Step's materialization expressions are evaluated, in order and atomically. The Step becomes `READY`, `SKIPPED`, or `FAILED`. See [Step input binding](#step-input-binding). |
| **Job output evaluation** (new, `CONTROL_FLOW` only) | Per job output, once every `Steps.*` symbol its *value* references is concrete | `Param.*`, `RawParam.*`, `Job.Name`, the referenced `Steps.*` | `Step.Name`, `Task.*`, `Session.*` | The output's *value* is evaluated, converted, and checked; a failure fails the Job. See [Job outputs](#job-outputs). |
| **Task execution** | as today | as today, and the `Steps.*` symbols of the Step's direct dependencies | *(none)* | As today. With `CONTROL_FLOW`, the runtime processes `onRun`'s output directives as they are printed, reverse-maps `PATH` values, and commits the values when the Task succeeds. |

### Output emission

#### The `openjd_output_var` directive

A Task of a Step that declares *outputVariables* reports an output value by printing a line to stdout or stderr from
its `onRun` action, in one of two forms:

```
openjd_output_var: <Name>=<JSON>
openjd_output_var: <Name>=@<FilePath>
```

The runtime recognizes the directive the way it recognizes the other messages of How Jobs Are Run: a line that begins
with `openjd_output_var:`. It is recognized only in a Session running a Job whose Job Template lists `CONTROL_FLOW`. In
any other Session the line is ordinary output, as it is today, and so it is for an Action of an Environment Template
(see below). Like the other messages, the line appears in the Action's log by default. A runtime that offers an
option to hide recognized messages from the log may hide a valid directive too, but an invalid directive always stays
in the log, with the error, so that the cause of the failure can be seen.

The text after `openjd_output_var:` has the grammar:

```bnf
<Directive>    ::= <WS>* <Identifier> <TaskSelector>? "=" <WS>* <Value> <WS>*
<TaskSelector> ::= "[" <Int> "]"
<Value>        ::= <JSON> | "@" <FilePath>
<WS>           ::= " " | "\t"
```

Where:

* The line terminator (`\n` or `\r\n`) is not part of the directive.
* `<Identifier>` is the output's name, matched case-sensitively against the Step's declared output names. There is no
  whitespace between it, the `<TaskSelector>`, and the `=`.
* `<TaskSelector>` names the Task of a chunk that the directive reports for, by its value of the Step's `CHUNK[INT]`
  parameter. `<Int>` is a decimal integer, optionally negative, with no whitespace inside the brackets. It is
  required in a Step with a `CHUNK[INT]` parameter and not allowed in any other Step (see
  [Chunked Steps](#chunked-steps)).
* `<JSON>` is one JSON value (RFC 8259). It is on one line, because a directive is a line, so a string that contains a
  line break is written with the escape `\n`. `NaN`, `Infinity`, and `-Infinity` are not JSON and are invalid. Tools
  that write JSON on one line, such as Python's `json.dumps` with default arguments, produce valid values directly for
  finite numbers; `allow_nan=False` makes Python raise rather than print a non-finite number.
* `<FilePath>` is the rest of the line after `@`, with trailing spaces and tabs removed. It must not be empty. A JSON
  value never begins with `@`, so the two forms can't be confused.

```bash
echo "openjd_output_var: FrameCount=1440"
echo "openjd_output_var: FrameRate=\"24000/1001\""
echo "openjd_output_var: Keys=[\"clips/a.mov\", \"clips/b.mov\"]"
echo "openjd_output_var: Children=@children.json"
# Arbitrary text needs a JSON encoder:
echo "openjd_output_var: Note=$(jq -Rn --arg v "$note" '$v')"
```

**Which Actions may print it.** Only the `onRun` action of a Task of a Step that declares *outputVariables*. Under
`WRAP_ACTIONS`, the runtime scans a wrap hook's stdout in place of the wrapped Action's, so a line printed by a wrap
hook, or forwarded by it from the wrapped process, is treated as output of the Action the hook runs in place of. A
directive from `onWrapTaskRun` is therefore a directive of the Task, whichever template defines the wrapping
Environment. In a Session running a Job that lists `CONTROL_FLOW`:

* a directive attributed to any other Action defined in the Job Template, such as a Job or Step Environment's
  `onEnter` or `onExit` (whether or not a wrap hook runs in its place), or the `onRun` of a Step that declares no
  *outputVariables*, is invalid, and cancels and fails that Action;
* a directive attributed to an Action defined in an Environment Template, which has not opted in to `CONTROL_FLOW`, is
  ordinary output.

> **TODO (deferred): outputs from Environment actions.** No use case requires them yet. A candidate is an
> Environment whose `onEnter` discovers a license server address that a later Step needs. Adding them later is
> backward compatible.

#### File values

A value too large for one line, or more convenient to write as a file, is written to a file that the directive names
with `@<FilePath>`:

1. A relative `<FilePath>` is resolved against the Session's Working Directory, which is also where an Action starts.
   An absolute path is used as written.
2. The file that is opened must be a regular file inside the Session's Working Directory, after following symbolic
   links. Otherwise the directive is invalid. A runtime checks this on the file it actually opened, for example by
   resolving the path beneath a handle to the Working Directory (Linux `openat2` with `RESOLVE_BENEATH`) or by
   checking the opened handle's final path, so that the Action can't swap a link in between a check and the read.
   A runtime SHOULD open the path without blocking and check that it is a regular file before reading, because
   opening a FIFO for reading blocks until a writer appears.
3. The runtime MUST read the file with no more access than the Action's user has. The runtime may run as a different
   operating-system user from the Action, and without this rule a directive could copy a file that the Action's user
   cannot read, for example through a hard link placed in the Working Directory, into outputs that are stored and
   displayed. A runtime that runs Actions as its own user meets this rule directly; one that runs them as another user
   reads the file as that user, for example through the helper process it already uses to run the Action.
4. The file holds exactly one JSON value, UTF-8 encoded, optionally preceded by a byte order mark and surrounded by
   whitespace, including line breaks. Unlike an inline value, it may span lines.
5. The runtime reads the file when it processes the directive, before it processes any later line of the Action's
   output. The value is the file's content at that moment, so the Action must finish writing and close the file before
   it prints the directive. Later changes to the file have no effect. The runtime does not modify or delete the file.

Under `WRAP_ACTIONS`, a wrap script that runs the wrapped Action with the Session's Working Directory at another path,
for example in a container, need do nothing for a relative `<FilePath>`. An absolute path printed by the wrapped
Action must be valid for the runtime, so authors should prefer relative paths.

> **Note for PowerShell authors.** Windows PowerShell 5.1's `Out-File` and `>` default to UTF-16, which is not a
> valid `@` file. Use `Set-Content -Encoding utf8` (or PowerShell 7, whose default is UTF-8).

#### Collect modes

The *collect* property of each `<OutputVariableDefinition>` says how the directives a Task prints for that output
become the Task's value:

| *collect* | Directives per Task | The Task's value | With no directive |
|---|---|---|---|
| `ONCE` (default) | Exactly one. A second one is invalid. | The value of the directive. | The Task fails. |
| `LAST` | One or more. | The value of the last directive. | The Task fails. |
| `APPEND` | Zero or more. The declared type is `LIST[T]`, and each directive supplies one `T`. | The list of the directives' values, in the order the runtime read them. | `[]` |

Under `APPEND`, a `<FilePath>` value supplies one element, as an inline value does. The order is the order in which
the runtime reads the lines. A runtime that reads stdout and stderr separately does not define their relative order,
so a Task should print all the directives for an `APPEND` or `LAST` output on one stream.

#### Chunked Steps

With `TASK_CHUNKING`, a Step may have one `CHUNK[INT]` Task parameter, and each `onRun` runs a chunk: a set of that
parameter's values, with the Step's other Task parameters fixed. Each value in the chunk is one Task, and each Task
has its own outputs. A directive says which Task it is for with a `<TaskSelector>` holding that Task's value of the
`CHUNK[INT]` parameter:

```yaml
- name: Render
  parameterSpace:
    taskParameterDefinitions:
      - name: Frame
        type: CHUNK[INT]
        range: "1-240"
        chunks:
          defaultTaskCount: 10
          rangeConstraint: CONTIGUOUS
  outputVariables:
    - name: RenderTimeS
      type: FLOAT
      unit: s
  bash:
    script: |
      for frame in {{repr_sh([string(f) for f in list(Task.Param.Frame)])}}; do
        start=$(date +%s.%N)
        render --frame "$frame"
        echo "openjd_output_var: RenderTimeS[$frame]=$(echo "$(date +%s.%N) - $start" | bc)"
      done
```

1. **Which Tasks.** The selector must be a value in the chunk being run. A scheduler may form chunks differently from
   one run to the next, for example under `targetRuntimeSeconds`, so the check is against the chunk the runtime was
   given, not the parameter's whole range.
2. **Per Task.** Collect modes, constraint properties, and the size limit of [Limits](#limits) apply to each Task's
   values separately: under `ONCE`, `RenderTimeS[12]` may be reported once and `RenderTimeS[13]` once. When the
   Action exits successfully, every Task in the chunk must have each of its `ONCE` and `LAST` outputs, and each of its
   `APPEND` outputs meets its `minLength`.
3. **The chunk is one attempt.** If the Action fails, is canceled, or times out, or a directive is invalid, no Task in
   the chunk commits any output, as no Task in a failed chunk succeeds under `TASK_CHUNKING`.
4. **Consumers see Tasks, not chunks.** `Steps.<Name>.Output.*` and `Steps.<Name>.TaskParam.*` have one element per
   Task in parameter-space iteration order, however the Tasks were chunked.

A value, not a position within the chunk, identifies the Task, so a directive keeps its meaning however the scheduler
splits or merges chunks. A selector on each line, rather than a directive that sets a current Task for the lines
after it, can't be attributed to the wrong Task when stdout and stderr interleave.

#### Processing a directive

The runtime processes each directive when it reads the line:

1. It checks that the Action may print directives, parses the directive, and checks that the name is a declared
   output of the Step, and in a chunked Step that the selector names a Task of the chunk.
2. It applies the collect mode: under `ONCE`, a second directive for the same name, and in a chunked Step the same
   Task, is invalid.
3. For `@<FilePath>`, it opens the file under the rules of [File values](#file-values) and reads at most as many bytes
   as remain of the Task's limit (see [Limits](#limits)), not counting the current value of a `LAST` output that this
   directive replaces, plus one to detect a file that exceeds it.
4. It converts the value to the declared type, or under `APPEND` to the element type (see below). A `PATH` value,
   or each element of a `LIST[PATH]` value, is then reverse-mapped to the submission path context (see
   [Path values](#path-values)).
5. It checks the constraint properties against the value: under `ONCE` and `LAST`, every declared constraint property
   (so each value a `LAST` output reports must satisfy them, not only the last); under `APPEND`, the `item`
   constraints on the new element and that the list does not exceed `maxLength`.
6. It updates the Task's value and its size (see [Limits](#limits)).

A directive that fails any of these checks is **invalid**. The runtime MUST NOT ignore an invalid directive. As soon
as it finds one, it MUST cancel the Action it is attributed to and fail it (for a Task's `onRun`, the Task's
attempt), whatever the exit code, with a message that names the output, the problem, and the offending text.
Canceling at once avoids spending the rest of the run on an attempt that can't succeed, and makes the outcome the
same in every runtime. The cancel is the Action's normal cancelation, with its notify period, if any. Specifically, a
directive is invalid if:

1. the text after `openjd_output_var:` does not match the grammar, or `<JSON>` is not valid JSON;
2. it names an output that the Step does not declare;
3. the value is JSON `null` or a JSON object, contains either, or doesn't convert to the declared type;
4. the value violates a constraint property as in step 5;
5. it is a second directive for a `ONCE` output;
6. for `@<FilePath>`: the path does not resolve to a regular file inside the Session's Working Directory, the file
   can't be read, or its content is not valid UTF-8, contains a NUL character, or is not exactly one JSON value;
7. it exceeds a limit of [Limits](#limits);
8. it is attributed to an Action of the Job Template other than the `onRun` of a Task of a Step that declares
   *outputVariables*;
9. the line is not valid UTF-8;
10. a `PATH` value is relative, or the runtime finds no submission-context form for it, under the requirements of
    [Path values](#path-values); or
11. it has no `<TaskSelector>` in a Step with a `CHUNK[INT]` parameter, has one in any other Step, or names a value
    that is not in the chunk being run.

When the Action exits successfully, the runtime also checks that every `ONCE` and `LAST` output has a value and that
every `APPEND` output's list satisfies its `minLength`, for each Task of a chunk. A missing value or a list that is too
short fails the Task.

If the attempt fails for any reason, including an invalid directive, a missing value, or an `onRun` action that
fails, is canceled, or times out, it has no outputs. Values from it that the runtime has already forwarded are
discarded.

**Type conversion.** A JSON value is first given its natural EXPR type: a number with neither a fraction nor an
exponent that fits in a signed 64-bit integer is an `int`, any other number is a `float`, a string is a `string`,
`true` and `false` are `bool`, and an array is a list. Array elements are typed by the same rule and the array is
inferred as a list literal with the target type (§1.2.6 of the Expression Language). The value is then converted to
the target type with EXPR's implicit non-destructive coercion (§1.2.3), so that a `Steps.*` value behaves in an
expression exactly like a `Param.*` value of the same type. The target type is the declared type, or for `APPEND` its
element type. `null` anywhere in a value, an object anywhere in a value, and nesting deeper than the target type are
errors, as is a number too large to represent as a `float`. For example:

| Declared type | Directive (after `openjd_output_var: `) | Result |
|---|---|---|
| `INT` | `N=12` | `12` |
| `INT` | `N=12.0` | `12` |
| `INT` | `N=12.5` | Invalid: `12.5` cannot be represented exactly as an `int` |
| `FLOAT` | `X=3` | `3.0` |
| `STRING` | `S="a=b"` | `"a=b"` |
| `STRING` | `S=123` | `"123"` |
| `BOOL` | `B=true` | `true` |
| `RANGE_EXPR` | `R="1-10:2"` | `1-10:2` |
| `RANGE_EXPR` | `R=[1, 3, 5]` | Invalid: a `list[int]` does not implicitly coerce to `range_expr` |
| `LIST[INT]` | `L=[]` | `[]` |
| `LIST[FLOAT]` | `L=[1, 2.5]` | `[1.0, 2.5]` |
| `LIST[STRING]` | `L=[1, "a"]` | `["1", "a"]` |
| `LIST[INT]` | `L=[1, null]` | Invalid: `null` cannot be a list element |
| `LIST[LIST[INT]]` | `G=[[1], []]` | `[[1], []]` |
| `LIST[FLOAT]`, `collect: APPEND` | `E=1`, then `E=2.5` | `[1.0, 2.5]` |
| `LIST[FLOAT]`, `collect: APPEND` | `E=[1]` | Invalid: each directive supplies one `float` |
| `LIST[LIST[INT]]`, `collect: APPEND` | `G=[1]`, then `G=[]` | `[[1], []]` |
| `PATH` | `P="/mnt/proj/shot01/out"` | `P:\shot01\out`, with a rule from `P:\` (`WINDOWS`) to `/mnt/proj` |
| `PATH` | `P=7` | Invalid: an `int` does not implicitly coerce to `path` |
| any | `X=null` | Invalid: outputs are not optional |

#### Path values

A `PATH` value that a Task reports is a path in that Task's Session: on its worker host, with its Session's path
mapping rules applied to the paths it was given. A consumer may run on another host with other rules, and a job
output is read by the submitter. So the runtime stores a `PATH` value in the **submission path context**, the form in
which `PATH` job parameter values are given, by applying the Session's path mapping rules in reverse.

The requirements are on the outcome. They assume, as forward mapping of `PATH` job parameters already does, that a
Session's path mapping rules map from the Job's submission path context; a runtime that knows the submission path
format SHOULD use only rules whose `source_path_format` is that format or `URI`.

* A relative path is invalid: it would be relative to a Session Working Directory that no consumer can see.
* A URI value (EXPR's URI form, such as `s3://bucket/key`) is stored unchanged, as a URI job parameter value is
  given unchanged.
* If the Session has no path mapping rules (before any filtering by source format), any other (absolute) value is
  stored unchanged.
* Otherwise the stored value MUST be in the submission path context, and forward mapping it with the producing
  Session's own rules MUST give back the reported path, where two paths are equal if they are equal after lexical
  normalization (`.` removed, `..` applied, trailing separators dropped except at a root) and compared under the rules of the worker
  host's path format: Windows paths case-insensitively with `\` and `/` both separators, POSIX paths
  case-sensitively. If the runtime can find no such form, the value is invalid.

How a runtime finds the stored form is up to it; a runtime with more knowledge of the farm's storage may do better
than the rules alone. A runtime that has only the rules SHOULD use this reference inverse:

1. A value that is a URI (EXPR's URI form, such as `s3://bucket/key`) is stored unchanged, as a URI job parameter
   value passes through forward mapping unchanged when no rule matches it.
2. Any other value must be an absolute path. A relative path is invalid: it would be relative to a Session Working
   Directory that no consumer can see.
3. The path is lexically normalized: `.` components are removed and `..` components applied, without resolving
   symbolic links, and trailing separators are dropped, except that a root (`/`, `C:\`) keeps its separator. Each
   rule's `destination_path` is normalized the same way.
   Paths in the worker host's format are compared with that format's rules: a Windows path case-insensitively, with
   `\` and `/` both separators; a POSIX path case-sensitively, with `/`. A rule matches when its `destination_path` is
   the path, or a prefix of it that ends on a component boundary.
4. Each matching rule gives a candidate: the remaining components of the path after `destination_path`, joined with
   the separator of the rule's `source_path_format` (`/` for `URI`), then joined to `source_path` with one such
   separator unless `source_path` already ends with one. If no components remain, the candidate is `source_path`.
   Components are appended unchanged, including to a URI. Candidates are tried in
   order of decreasing `destination_path` length, then in the order the rules are listed, and the first candidate
   that forward-maps back to the reported path is stored. Forward mapping can choose a different rule when rules
   nest: with `P:\` → `/mnt/p` and `P:\x` → `/data/x`, the worker path `/mnt/p/x/f` can't be stored as `P:\x\f`,
   which maps forward to `/data/x/f`; no other rule matches, so the value is invalid (step 5). A candidate from a rule whose `source_path_format` is `URI` is a URI, and a
   consumer whose Session has no rule for it receives it as a URI.
5. If no candidate maps back and the Session has path mapping rules, the value is invalid. Most often the path is on
   the worker only, for example under the Session Working Directory, which is deleted when the Session ends.
6. If the Session has no path mapping rules, the value is stored unchanged, as a `PATH` job parameter is used
   unchanged then. This is right when the worker and the submitter share their path context, as with `openjd run` on
   one host. A scheduler that runs Tasks on hosts with another path context must supply rules for `PATH` values to
   work, as it must for `PATH` job parameters.

Constraint properties (`allowedValues`, `minLength`, `maxLength`) are checked against the stored value, and the
stored value is the one compared for [Retries and re-runs](#retries-and-re-runs). A consumer reads a `PATH` value as
`Steps.P.Output.X` in format strings resolved on the worker host, with its own Session's rules applied forward, or as
`Steps.P.RawOutput.X`, the stored value, anywhere else.

#### Committing values

A Task's output values are committed when the Task succeeds. The values of every Task of a Step are retained by the
scheduler for the life of the Job. A scheduler SHOULD make each Task's committed outputs visible to users, for example
beside the Task's parameters, using the declared *unit* and *description*, since they are often useful with no
consumer at all.

Because the runtime processes directives as they are printed, it MAY forward each value to the scheduler as soon as it
is accepted, so that the scheduler has the values when the Task ends. A scheduler MAY show the values of a Task that
is still running, such as a `LAST` counter, if it marks them as not yet committed. Only committed values are visible to
consumers through `Steps.*`.

Output values are stored, logged, and may be displayed, so they are not redacted and MUST NOT be used to pass secrets.

#### Limits

1. A runtime MUST accept a directive line of at least 64 KiB (65,536 bytes of UTF-8). A line that begins with
   `openjd_output_var:` and is longer than the runtime's line limit is invalid: the runtime MUST NOT truncate a
   directive and parse what remains, and MUST NOT recognize the discarded remainder of an over-long line as a
   directive. Larger values use `@<FilePath>`.
2. A runtime MUST accept at least 1 MiB (1,048,576 bytes) of output values per Task attempt (per Task of a chunk), and
   MAY accept more. The size counted is that of the values the Task currently holds, as the bytes that reported them
   (the inline `<JSON>` text without the `<WS>` around it, or the whole content of an `@` file): the `ONCE` value, the
   current `LAST` value (a value it replaces stops counting), and every `APPEND` element. For a `PATH` or `LIST[PATH]`
   value, or a `PATH` element under `APPEND`, the larger of its reported bytes and the JSON serialization of its stored
   form counts. A directive that takes the size over the runtime's limit is invalid.
3. A producer's constraint properties bound its values. A scheduler SHOULD encourage authors to declare `maxLength` on
   list outputs that feed a parameter space, and on `APPEND` outputs.
4. A scheduler MAY impose a maximum number of Tasks per Step. A Step whose materialized parameter space exceeds it
   fails at step input binding, before any of its Tasks runs, with a message that names the limit.
5. Expressions over output values are evaluated under EXPR's existing memory and operation limits.

#### Retries and re-runs

* **Retry of a failed attempt.** A failed attempt committed nothing, so a retry starts with no values.
* **Re-run of a succeeded Task.** An output of a producer is **consumed** once a consumer that references it (as
  found by symbol collection) has been bound or has started a Task. A scheduler that allows re-running a Task that
  already succeeded MUST apply these rules to each of the re-run Task's own output values:
    1. If the output has not been consumed, a successful re-run's value replaces the earlier one.
    2. If the output has been consumed and the re-run succeeds with the same value, nothing changes.
    3. If the output has been consumed and the re-run succeeds with a different value, the re-run attempt is
       reported as failed, with a message that names the output and a consumer that has consumed it. The Task keeps
       its earlier succeeded state and all of its committed values, so the Step and its consumers are unaffected.
    4. If the re-run fails, it commits nothing, as any failed attempt, and the committed values are unchanged.

  The check is all or nothing: if any output falls under rule 3, nothing from the re-run is committed. Values are the
  same if their converted, typed values are equal.

  So a metric that no Step reads never blocks a re-run, but a consumed value that varies from run to run, such as a
  measured frame rate that a later Step reduces, makes every differing re-run of its Task fail once it is consumed.

> **TODO (deferred): cascade recompute.** A scheduler could instead invalidate and re-bind every Step that consumed
> the changed values, transitively. That has a large blast radius and is scheduler policy, so this draft only defines
> the safe default above. A scheduler capability for cascade recompute can be added later.

### Modifications to the Expression Language

CONTROL_FLOW adds no types, functions, or operators to EXPR. It adds:

* The `Steps` root symbol and its `Output`, `RawOutput`, `TaskParam`, and `TaskRawParam` members, with the types and
  scopes of
  [The `Steps.*` scope](#the-steps-scope).
* The empty-list rule of [Empty parameter spaces](#empty-parameter-spaces) for Task parameter ranges.
* A validation warning. EXPR discards a `null` silently in three places (§1.3.2 of the Expression Language): an
  interpolation with text around it becomes the empty string, so `"--size {{Steps.P.Output.X}}"` becomes
  `"--size "`; a list item (such as an entry of `args`) is dropped; and an optional field is omitted. An
  implementation of template validation MUST emit a warning (not an error) for every interpolated expression in one
  of those three positions whose static type includes `nulltype` and that references, directly or through a `let`
  binding, a `Steps.*` symbol of a conditional Step. The warning names the expression and suggests a conditional
  expression or an `or` default. An expression whose type has no `nulltype`, such as
  `'frames only' if x == null else string(x)`, draws no warning, and neither does *enableIf*, where `null` is an
  error.

Symbol collection (RFC 0005) is how an implementation finds a Step's `Steps.*` references, both to check them and to
decide whether the Step needs step input binding.

> **TODO (deferred): functions for correlating results.** Finding the Task that produced the best value currently
> needs a comprehension over indices (see the sweep example). `argmin(list[T]) -> int` and `argmax`, or a list
> `index(list, item)`, would make it direct. They are a change to RFC 0006 and don't depend on this RFC.

### Step input binding

#### Materialization expressions

A Step's **materialization expressions** are its format strings that the scheduler must evaluate to know whether the
Step runs and what its Tasks are. They are, in evaluation order:

1. *enableIf*;
2. *let*;
3. *hostRequirements*;
4. the `range` of each Task parameter in *parameterSpace*, in declaration order, then the other format strings of
   *parameterSpace* (with `TASK_CHUNKING`, the `chunks` fields such as `defaultTaskCount`);
5. with `TASK_DEPENDENCIES`, the subspace ranges of its *dependencies* (see
   [Composition with `TASK_DEPENDENCIES`](#composition-with-task_dependencies));
6. the TEMPLATE-scope format strings of each of its *stepEnvironments*, in order, each starting with its *enableIf*;
7. the TEMPLATE-scope format strings of its *script*, such as an action's `timeout` and `cancelation` fields.

These are all of the Step's format strings that are resolved at Job creation today (those not annotated
`@fmtstring[host]`), plus the new *enableIf* fields. A format string that a future extension adds to a Step and that
is resolved at Job creation is a materialization expression too, evaluated after those above. Format strings resolved on the worker host are not
materialization expressions, even if they reference `Steps.*`: a Step's `dependencies` already guarantee that the
values they read are concrete before any of its Tasks runs.

#### When a Step is bound

A Step is bound once, at the earliest moment every `Steps.*` symbol referenced by its materialization expressions is
concrete:

* If its materialization expressions reference no `Steps.*` symbol, or only symbols that are concrete at Job
  creation, it is bound at Job creation, as every Step is today.
* Otherwise it enters `PENDING_INPUTS` at Job creation and is bound when the last of those symbols becomes concrete.
* If a Step whose `Steps.*.Output` symbols it references fails, or can never run because one of its own dependencies
  failed, the Step can never be bound. It stays blocked and never runs, as a Step whose dependency failed does today.

#### What binding does

Binding evaluates the Step's materialization expressions in the order above, with the referenced `Steps.*` symbols
concrete, and then:

1. If *enableIf* is `false`, the Step becomes `SKIPPED`. Nothing after *enableIf* is evaluated, so a disabled Step's
   *let* bindings may assume the Step runs (for example, `min(...)` of a list that is only non-empty when it does).
2. Otherwise the Step's parameter space is materialized from the evaluated ranges and its `combination`, and the
   scheduler checks it, and with `TASK_DEPENDENCIES` the Task graph, against its limits.
3. If every evaluation and check succeeds, the Step becomes `READY` with its Tasks enumerable. Its Tasks are then
   scheduled as today: when every dependency is satisfied and a Worker Host meets its host requirements. A Step with
   zero Tasks completes successfully when its dependencies are satisfied.
4. If any evaluation fails (including a `fail(...)` call and a `null` where a value is required) or any check fails
   (a Task count over the scheduler's limit, or with `TASK_DEPENDENCIES` a cycle or an out-of-range index), the Step
   becomes `FAILED` before any of its Tasks is dispatched. The failure message names the field and the expression.

Binding is atomic: either every materialization expression succeeds and the Step's Tasks exist, or none of its Tasks
runs.

#### Implementation strategy

A scheduler may create a `PENDING_INPUTS` Step's Tasks only when it is bound (deferred creation), or create the Step
first and fill in its Tasks when it is bound (create-then-update). The observable contract is the same: a Step's
Tasks are enumerable if and only if it has been bound and is not `SKIPPED`. A scheduler or tool that reports a
`PENDING_INPUTS` Step SHOULD report its Task count as not yet known and name the outputs it is waiting for, for
example "determined at run time from `Steps.VideoToFrames.Output.FrameCount`".

> **TODO (deferred): loop Steps.** A loop extension would add Steps whose Tasks are created one at a time after the
> Step is bound, so it would amend this contract for those Steps: a loop Step's Tasks are enumerable only up to the
> last one created, and its Task count is reported as not yet known until the loop ends. See
> [Loop Steps](#loop-steps).

### Conditional execution: `enableIf`

* **Static or dynamic.** An *enableIf* that references no `Steps.*` symbol is decided at Job creation. One that does
  is decided at the Step's binding. A statically disabled Step is still part of the Job and is shown as `SKIPPED`, so
  every Job created from a template has the same Steps, whatever its parameter values.
* **A skipped Step** runs no Tasks and enters no Environments. It satisfies the `dependsOn` of the Steps that depend
  on it once its own dependencies are satisfied, and its `Steps.*.Output` and `Steps.*.TaskParam` symbols are
  `null`. Its dependents run, and those that read its outputs must handle `null`, which the `T?` type of the symbols
  makes visible:

  ```yaml
  let:
    - video_bytes = Steps.EncodeVideo.Output.VideoBytes or 0
  ```

  A consumer that should not run when its producer was skipped says so:

  ```yaml
  enableIf: "{{Steps.EncodeVideo.Output.VideoBytes != null}}"
  ```

* **Only the skipped Step's own symbols are `null`.** A Step that ran after a skipped Step has ordinary outputs.
* **A disabled Environment** is left out of the Session's Environment stack: it is not entered, its `variables` are
  not set, and Sessions are matched to Tasks as if it were not defined. A disabled Job Environment is disabled for
  every Session of the Job; a disabled Step Environment for every Session that runs that Step's Tasks.
* **Failure is not skipping.** A Step that fails, or can never run because a dependency failed, does not satisfy
  `dependsOn`, as today. A Step that is `SKIPPED` but one of whose own dependencies fails never satisfies `dependsOn`
  either. Its `null` symbols are concrete as soon as it is skipped, so a consumer may be bound with them, but the
  consumer stays blocked behind the failure, as any Step behind a failed dependency does.

### Reductions

A multi-Task producer's output is a list with one element per Task, and the Steps that read it reduce it with the
existing EXPR function library:

```yaml
let:
  - total = sum(Steps.Transcode.Output.OutputBytes)
  - best = min(Steps.Sweep.Output.mpjpe_mm)
  - best_i = [i for i in range(len(Steps.Sweep.Output.mpjpe_mm)) if Steps.Sweep.Output.mpjpe_mm[i] == best][0]
  - best_window = Steps.Sweep.TaskParam.Window[best_i]
```

1. **Order.** The list is in the producer's parameter-space iteration order: the order in which the producer's
   `combination` expression enumerates its Tasks (or the default combination when it has none), independent of the
   order in which Tasks ran, finished, or were retried. A multi-dimensional space is flattened in that order (CWL's
   `flat_crossproduct`). For example, with `combination: "A * B"` the last parameter varies fastest.
2. **Alignment.** Element `i` of every `Steps.P.Output.*` list and every `Steps.P.TaskParam.*` list belongs to the
   same Task.
3. **Completeness.** The list is concrete only when every Task of the producer has succeeded, so there are no partial
   values. A producer with zero Tasks has empty lists.
4. **Nesting.** A per-Task `LIST[T]` output becomes `list[list[T]]`. A per-Task `LIST[LIST[INT]]` output is not
   allowed on a multi-Task producer (see [`<OutputVariableDefinition>`](#outputvariabledefinition)).
5. **Cost.** An implementation may assemble a list when it is first used rather than when the producer finishes.

> **TODO (deferred): partial reductions.** Values from the Tasks of a producer that have finished, before all of them
> have, would let a consumer start early. "Iteration order" is not well defined for an incomplete set, and such a
> value would make a consumer's structure depend on scheduling. A separate symbol can be added later.

### Job outputs

A Job's outputs are what it reports back to the submitter and to any other caller, as its parameters are what the
caller gives it: a frame count, the winning configuration of a sweep, the directory that holds the results. Each is
an expression over the Job's parameters and its Steps' outputs. A Task never writes a job output directly.

1. **When an output is evaluated.** Each job output is evaluated on its own, as soon as every `Steps.*` symbol its
   *value* references is concrete (see [The `Steps.*` scope](#the-steps-scope)), by the same rule that decides when a
   Step is bound. An output that references no `Steps.*` symbol, or only `TaskParam` and `TaskRawParam` symbols of
   Steps bound at Job creation, is evaluated at Job creation. An output that reads `Output` or `RawOutput` of a Step
   is evaluated when that Step succeeds or is skipped, because its values are concrete only then
   ([Reductions](#reductions) item 3). A Job does not succeed until every job output whose inputs are concrete has
   been evaluated, so a Job never reports success and then fails on a job output, except after a re-run (item 5). A user interface can therefore show each output as soon as it exists, before
   the Job finishes.
2. **Commit and display.** An evaluated output is committed at once and retained by the scheduler for the life of
   the Job. A scheduler SHOULD make job outputs visible to users and return them to callers, using *description* and
   *unit*, and SHOULD show an output that is not yet evaluated as pending and name what it waits for.
3. **Failure.** If evaluating an output fails (including a `fail(...)` call, a value that doesn't convert to *type*,
   or a violated constraint property), the scheduler MUST fail the Job: it marks the Job failed with a message that
   names the output, the expression, and the problem, dispatches no further Tasks, and SHOULD cancel Tasks that are
   running. A job output evaluated at Job creation that fails makes Job creation fail, as an invalid job parameter
   value does. Failing at once puts the error next to its cause and doesn't spend the rest of the Job's run time on
   results that can't be reported.
4. **Missing producers.** If a Step that an output references fails, or can never run because one of its
   dependencies failed, the output is not evaluated while that remains so (if a requeue later makes the Step
   succeed, the output is evaluated then), and a scheduler SHOULD show it as unavailable, naming the
   Step, rather than as pending. If the Step is `SKIPPED`, its symbols are `null` and the output is evaluated with
   them; a `null` result means the output has no value, which is not an error. A Step with zero Tasks gives empty
   lists, so `min(Steps.P.Output.X)` fails the Job when the space is empty; write
   `min(Steps.P.Output.X) if len(Steps.P.Output.X) > 0 else null` where emptiness is expected (see
   [Design Choices](#9-an-empty-computed-parameter-space-is-a-zero-task-success)).
5. **Re-runs.** Job outputs do not count as consumers for [Retries and re-runs](#retries-and-re-runs): a caller
   outside the Job can't be tracked. When a re-run commits new values for a Task, the scheduler re-evaluates every job
   output that references the Task's Step and replaces its value. If a re-evaluation fails, the re-run's values stay
   committed, the job output keeps its earlier value and is marked stale, and the Job fails as in item 3, including a
   Job that had already succeeded.
6. **Paths.** A `PATH` or `LIST[PATH]` job output built from `PATH` values is in the submission path context,
   because every `PATH` value it can read (`RawParam.*`, `Steps.*.RawOutput.*`, `Steps.*.TaskRawParam.*`) is. A value
   built from a `STRING`, which EXPR converts implicitly to `path`, is whatever that string holds, so a path that a
   Task reports belongs in a `PATH` output variable. A caller can use it directly
   on the submitting host, for example to download or open the results.
7. **Limits.** A scheduler MUST accept job output values totaling at least 1 MiB per Job, measured as their JSON
   serialization, and MAY accept more. Exceeding its limit fails the Job, as in item 3.
8. **Secrets.** Job outputs are stored and displayed, so they are not redacted and MUST NOT be used to pass secrets.

```yaml
jobOutputDefinitions:
  - name: FrameCount
    type: INT
    value: "{{Steps.VideoToFrames.Output.FrameCount}}"     # evaluated when VideoToFrames succeeds
  - name: OutputVideo
    type: PATH
    objectType: FILE
    value: "{{RawParam.OutputVideo}}"                       # evaluated at Job creation
```

### Modifications to How Jobs Are Run

> A modification to [How Jobs Are Run](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run)

#### Step states

A scheduler tracks, for each Step of a Job, a state from the diagram in [Step states](#step-states). The states this
RFC adds to the model are:

* `PENDING_INPUTS` — The Step's materialization expressions reference a `Steps.*` symbol that is not concrete. Its
  Tasks are not enumerable and none can be scheduled.
* `SKIPPED` — The Step's *enableIf* evaluated to `false`. It is terminal and runs nothing. A dependency on it is
  satisfied once the skipped Step's own dependencies are satisfied, so ordering stays transitive.

A Step that has been bound and is not `SKIPPED` is gated by its dependencies exactly as today. A Step whose binding
fails is `FAILED`, which, like any failed Step, does not satisfy `dependsOn`.

#### Job output evaluation

> An addition to [How Jobs Are Run](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run)

A scheduler evaluates each job output as soon as every Step value it reads is known, or at Job creation if it reads
none, keeps its value for the life of the Job, and returns it to callers. A job output that fails to evaluate fails
the Job. See [Job outputs](#job-outputs).

#### Path mapping of output values

> A modification to [Path Mapping](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run#path-mapping)

A `PATH` output value that a Task reports is mapped in reverse, from the worker's form back to the submission path
context, before it is stored, and mapped forward again in each Session that reads it. See [Path values](#path-values).

#### Stdout messages

> A modification to [Stdout/Stderr Messages](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run#stdoutstderr-messages)

Add the following item to the list of special messages:

* `openjd_output_var: <name>=<json>` or `openjd_output_var: <name>=@<path>` where `<name>` is the name of an output
  variable that the Step declares, `<json>` is a JSON value on one line, and `<path>` names a file inside the
  **Session**'s **Working Directory** that holds one JSON value. This reports a value of an output variable of the
  **Task** whose `onRun` **Action** printed it, and can only be emitted by that **Action**. The runtime reads a `@` file
  as soon as it reads the line. An invalid message, or one emitted by any other **Action** of the Job Template, cancels
  the **Action** at once and fails it, and stays in the log with the error. Under WRAP_ACTIONS, a line from a wrap hook
  counts as output of the **Action** it runs in place of. Requires the CONTROL_FLOW extension; without it, and for
  **Actions** of an Environment Template, the line is ordinary output. See [Output emission](#output-emission).

No environment variable is added.

### Composition with `TASK_DEPENDENCIES`

`TASK_DEPENDENCIES` is a separate extension, not yet specified, for dependencies between individual Tasks (#82). It
does not depend on `CONTROL_FLOW`, and `CONTROL_FLOW` does not depend on it. When a template lists both:

1. The subspace ranges of a Step's *dependencies* are materialization expressions of that Step, evaluated at its
   binding after its parameter space is materialized, once per Task of the Step. They may reference the `Steps.*`
   symbols of the Step's direct whole-Step dependencies.
2. At binding the scheduler builds the Step's Task-level dependency graph from those ranges and checks it. A cycle,
   or an index outside the depended-upon Step's parameter space where `TASK_DEPENDENCIES` makes that an error, fails
   the Step before any of its Tasks runs. An empty subspace selects no Tasks.
3. A dependency with a subspace does not satisfy rule 1 of [The `Steps.*` scope](#the-steps-scope), because it lets
   the Step's Tasks start before every Task of the producer has finished.

> **TODO (depends on `TASK_DEPENDENCIES`):** the subspace field's name and schema, the scope of `Task.Param.*` inside
> it, and the treatment of out-of-range values belong to the `TASK_DEPENDENCIES` RFC. Whichever of the two RFCs is
> accepted second states these rules unconditionally.

### Validation

Every rule this extension adds, and when it is checked.

**At template validation** (by the model library, before submission):

1. `CONTROL_FLOW` is listed only together with `EXPR`.
2. *outputVariables*: 1 to 32 elements; names are unique `<Identifier>`s; types are `<OutputVariableType>`s; constraint
   properties are valid for the type; *collect* is an `<OutputCollectMode>`, and `APPEND` only with a `LIST[...]`
   type; *unit* is 1 to 32 characters with no control characters; no `LIST[LIST[INT]]` on a Step with a
   `parameterSpace`.
3. Every `Steps.<StepName>.Output.<VarName>` and `RawOutput.<VarName>` names a Step with an `<Identifier>` name
   that declares output `<VarName>`. Every `Steps.<StepName>.TaskParam.<ParamName>` and `TaskRawParam.<ParamName>`
   names a Step that has Task parameter `<ParamName>`.
4. A Step that references `Steps.<StepName>.*` lists `<StepName>` directly in its `dependencies` as a whole-Step
   dependency, and `<StepName>` is not the Step itself.
5. `Steps.*` appears only in the fields listed in [The `Steps.*` scope](#the-steps-scope), and a `Steps.*.Output.*`
   or `Steps.*.TaskParam.*` whose type is or contains `path` only in format strings resolved on the worker host.
6. *enableIf* references only the symbols its location allows, and does not appear in an Environment Template.
7. Every expression type-checks with `Steps.*` symbols as `unresolved` values of their types; *enableIf* has type
   `bool`.
8. The warning of [Modifications to the Expression Language](#modifications-to-the-expression-language) for an
   optional-typed expression that reads a conditional Step's `Steps.*` symbol where EXPR discards `null`.
9. The Step dependency graph is acyclic, as today.
10. *jobOutputDefinitions*: 1 to 50 elements; names are unique `<Identifier>`s; types and constraint properties as
    for *outputVariables*; each *value* type-checks against its type, made optional, and references only the symbols
    [`<JobOutputDefinition>`](#joboutputdefinition) allows, with any Step's `Steps.*` symbols, values whose type
    is or contains `path` only through `RawOutput` and `TaskRawParam`.

**At step input binding** (by the scheduler, before any Task of the Step runs): every materialization expression
evaluates; ranges are valid (a range string may not be empty; a list may be); the Task count is within the
scheduler's limit; and with `TASK_DEPENDENCIES`, the Task graph is acyclic.

**At job output evaluation** (by the scheduler, as each job output's inputs become concrete): the *value* evaluates,
converts to the declared type, satisfies its constraint properties, and fits the size limit; a failure fails the Job.

**As Actions run** (by the runtime, as it reads each line of an Action's output in a Session of a `CONTROL_FLOW` Job): the rules of
[Processing a directive](#processing-a-directive), including type conversion, constraint properties, collect modes,
`@` file paths, reverse mapping of `PATH` values, and the limits; and when `onRun` exits successfully, that every `ONCE` and `LAST` output has a value
and every `APPEND` list meets its `minLength`.

**On re-run** (by the scheduler): the rules of [Retries and re-runs](#retries-and-re-runs).

## Prior Art

Comparable workflow systems split into two families. **Fixed graph, late-bound width** systems (Argo Workflows,
Apache Airflow, GitHub Actions, Tekton, AWS Step Functions, CWL) fix the set of units when a run is created, and let
only the number of instances of a fanned-out unit, and whether a unit is skipped, depend on run-time data. **Graph
re-evaluation** systems (Snakemake checkpoints, HTCondor DAGMan `SUBDAG EXTERNAL`, Slurm self-submission, Nextflow
streaming) let running work generate new structure, at the cost of static validation and of any view of total work
before the run. CONTROL_FLOW is in the first family, which is the line discussion #22 drew for OpenJD.

| System | Outputs declared and typed | Emitted through | Referenced as | Fan-out from data | Dependents of a skipped unit | Fan-in |
|---|---|---|---|---|---|---|
| Argo Workflows | `outputs.parameters`, strings | file (`valueFrom.path`) or stdout (`result`) | `{{tasks.A.outputs.parameters.p}}` | `withParam` over a JSON array | run; outputs resolve to `""` | looped outputs as a JSON array |
| Apache Airflow | implicit return value, untyped | return value → XCom | `XComArg` argument | `.expand()`, `max_map_length` 1024 fails the producer | skip cascades; joins need trigger rules | lazy sequence of results |
| GitHub Actions | `jobs.<id>.outputs`, strings | file `$GITHUB_OUTPUT` (stdout `::set-output` deprecated) | `${{ needs.A.outputs.x }}`, needs a `needs` edge | `matrix` from `fromJSON(...)`, max 256 | skip cascades unless `always()` | none; outputs collide |
| Tekton | `results[]` typed `string`/`array`/`object` | file `$(results.r.path)` | `$(tasks.A.results.r)` | `matrix` over array results, max 256 | ordering dependents run; result consumers skip | `results.r[*]` array |
| AWS Step Functions | none; JSON state | task result JSON | JSONPath / JSONata | `Map` over an array or S3 | n/a (branching is a transition) | ordered array |
| CWL | typed, including optional `T?`, arrays, records | `outputBinding` glob or `cwl.output.json` | `source: stepA/out` | `scatter` over array inputs | outputs are `null` | arrays; `pickValue` for conditional sources |
| Snakemake | file patterns | files | input patterns | `checkpoint` re-evaluates the DAG | n/a | aggregation rule |
| Slurm | none | files | convention | `--array` fixed at submission | n/a | `afterok:<array>`; `aftercorr` is per-element |
| HTCondor DAGMan | none | files, exit codes | `VARS` macros | `SUBDAG EXTERNAL` generated at run time | `PRE_SKIP` | `PARENT ... CHILD` |

What this RFC takes from them:

* **Typed outputs** (Tekton, CWL). Declared types let a validator check every reference, every range built from it,
  and every condition before the run. Systems with string outputs (Argo, GitHub Actions) find those errors at run
  time.
* **An explicit edge to read outputs** (GitHub Actions `needs`, Argo `dependencies`). Airflow, Tekton, and CWL derive
  edges from references instead; OpenJD already has explicit `dependencies`.
* **`null` for a skipped unit's outputs** (CWL). CWL's `pickValue` is the explicit answer to "which of these
  possibly-skipped producers supplies the value"; in EXPR the `or` operator and conditional expressions do the same.
  The surveyed alternatives each have a pitfall: Airflow and GitHub Actions cascade skips and need special trigger
  rules at joins, Argo and GitHub Actions turn missing outputs into an empty string, and Tekton skips consumers of a
  skipped task along an edge that is implicit in its expressions.
* **Fail the producer** on a missing output (CWL, Nextflow, Snakemake, Argo without a default) and on an oversized
  fan-out (Airflow's `max_map_length`). Tekton fails the consumer on a missing result, which is harder to diagnose.
* **Ordered lists for fan-in** (Step Functions `Map`, Tekton `[*]`, Argo loop outputs, CWL scatter), with a defined
  flattening of multi-dimensional fan-outs (CWL's `flat_crossproduct`).
* **Workflow-level outputs as expressions over step outputs** (GitHub Actions reusable workflows'
  `on.workflow_call.outputs.<name>.value`, Tekton Pipeline `results`, CWL workflow `outputs` with `outputSource`, Argo
  workflow `outputs`), for job outputs.
* **Path remapping in render managers.** Thinkbox Deadline's path mapping and Pixar Tractor's dirmaps translate paths
  between submitter and worker forms. This survey found no system that specifies the reverse direction for values a
  task produces; this RFC defines it in terms of the existing rules.
* **Limits with a guaranteed minimum.** GitHub Actions allows 1 MB of outputs per job, Step Functions 256 KiB per
  state, Tekton 4 KB per TaskRun by default. This RFC guarantees at least 1 MiB per Task.

What this RFC does differently:

* **A stdout directive rather than a file.** GitHub Actions, Tekton, Argo, and CWL emit outputs through a file, and
  GitHub Actions moved from its `::set-output` stdout command to the `$GITHUB_OUTPUT` file because untrusted text in a
  log could set outputs that steered more privileged workflow steps. OpenJD Actions already report everything else on
  stdout, and code in `onRun` runs as the Job's user, so a printed value grants nothing new. This RFC keeps stdout,
  requires each directive to name a declared output, and makes a repeated `ONCE` output an error, which turns an
  accidental directive into a failed Task (see
  [Design Choices](#2-outputs-are-reported-with-a-stdout-directive-with-file-for-large-values)). Argo's `result` (a
  step's stdout as an output) and TeamCity's service messages are precedents for stdout.

* **An empty fan-out succeeds** rather than being skipped (Airflow, Argo, Tekton) or failing (GitHub Actions), so a
  computed parameter space never makes a Step's outputs optional (see
  [Design Choices](#9-an-empty-computed-parameter-space-is-a-zero-task-success)).
* **Re-runs are specified.** Render farms routinely requeue Tasks that succeeded, and the surveyed DAG systems mostly
  treat a run as one-shot (Airflow's "clear downstream" is the exception). This RFC defines a safe default.

For Task metrics, the precedents are:

* **TeamCity service messages.** A build step prints `##teamcity[buildStatisticValue key='<name>' value='<number>']`,
  and TeamCity charts the value across builds. This is the closest match: a stdout line, a name, and a value.
* **Amazon SageMaker training jobs.** A job declares `MetricDefinitions`, each a name and a regular expression, and
  SageMaker extracts the values from the job's logs. This shows the demand, and the cost of making every user write a
  regular expression for their own log format, which a fixed directive avoids.
* **Kubeflow Pipelines.** Components write a metrics file that the UI shows and compares across runs.
* **Experiment trackers** such as MLflow (`log_metric`) and Weights & Biases (`log`). These are library calls to a
  tracking server and need network access and credentials in the Task. A directive needs neither, and a scheduler or
  a bridge can still forward the values to such a tracker.

The full survey, with per-system notes and references, is in
[02-external-prior-art.md](../docs/control-flow-investigation/02-external-prior-art.md).

Within OpenJD, the draft `SERVICE` RFC (0009, discussion #145) is the closest precedent for a late-bound symbol scope
(`Service.*`, `unresolved` until a later stage), and RFC 0008's `WrappedStep` root is the precedent for naming a Step
other than the current one without overloading `Step`.

## Implementation Impact

- **Python (openjd-model)**: New schema fields (`outputVariables` with `collect` and `unit`, `enableIf` on Steps and
  Environments) gated on the extension, and the `Steps` symbol family in the template-validation symbol table with types derived from the
  producer. The validation rules are graph checks over existing data (direct dependency, identifier names, field
  scopes) plus the format-string warning. Job creation must stop treating every TEMPLATE-scope field as resolvable at
  creation: it reports which Steps need binding and exposes a function that binds one Step given the concrete
  `Steps.*` values, which schedulers and the CLI call. The parameter-space iterator already defines the order
  reductions need. Job outputs add the `jobOutputDefinitions` schema and a function that evaluates one job output
  given the concrete `Steps.*` values it reads, used in the same way. The `Raw` symbols are typed like the others,
  and `PATH` values of `Steps.*.Output` and `TaskParam` are passed to the session marked as paths, so that it maps
  them forward in host-scope format strings as it does `PATH` job parameters.
- **Rust (openjd-rs)**: The same model changes. Two existing behaviors change under the extension: a computed range
  may be empty, and the 1024-element cap that `crates/openjd-model/src/job/create_job/ranges.rs` applies to computed list ranges
  (`max_task_param_range_len`) applies only to YAML list literals, with a scheduler-supplied Task count limit taking
  its place.
- **Sessions (openjd-sessions, Python and Rust)**: Add `openjd_output_var:` to the action filter beside `openjd_env`,
  recognized only in a Session of a `CONTROL_FLOW` Job. For each directive: parse it and check its name and collect
  mode; for `@`, open the file with the Action user's access, check that the opened file is inside the Session's
  Working Directory, and read it up to the remaining limit; convert and check the value against the Step's
  declarations; and report the value through a new callback so that the application running
  the Session can forward it at once. At exit, check for missing values and return the collected values with the Task
  result. The Session must be given the Step's output declarations along with its script, whether the Job lists
  `CONTROL_FLOW`, and for each Environment it enters, whether the Job Template or an Environment Template defines it. A runtime that runs
  Actions as another user reads `@` files through its cross-user helper. One existing behavior must change: openjd-rs
  truncates over-long lines silently in four places (`truncate_line` in `crates/openjd-sessions/src/subprocess.rs`,
  and `LineFramer::accumulate`, `decode_and_truncate`, and `fit_out_payload` in
  `crates/openjd-sessions/src/helper/src/framer.rs`, the last of which can cut a line under 64 KiB that grows when
  escaped), and each must instead mark the line as truncated so that
  an over-long directive is invalid. No file is created before the Action runs. For `PATH` outputs, the session
  applies its path mapping rules in reverse, which needs the rules' source and destination paths it already holds.
- **CLI (`openjd run`)**: Runs Steps in dependency order already, so it binds each Step just before running it, passes
  the collected values in, and prints each Task's outputs, with their units, and each job output as soon as it is
  evaluated. When it runs only some Steps (`--step`), a job output that reads a Step it doesn't run is reported as not
  evaluated. `openjd summary` reports a Step that would need binding as
  having a Task count determined at run time.
- **Schedulers**: The largest work: storing committed output values for the life of the Job, evaluating job outputs as
  their inputs complete and returning them to callers, the `PENDING_INPUTS`
  and `SKIPPED` states, triggering binding when producers finish, displaying outputs (as metric columns beside each
  Task's parameters) and pending Steps, and the re-run rule. A scheduler that creates every Task at submission today must either defer creating a pending Step's Tasks or
  support adding them later.

No part of the design is harder in one language than another: the parsing is JSON, the evaluation is EXPR, and the
graph checks are the ones implementations already run on Step dependencies. Output values are small by design, and a
reduction is evaluated once per consumer, at binding or at Task start.

## Rejected Ideas

### A session-supplied output file

An earlier draft gave each Task attempt a file, `Task.OutputFile` (also `OPENJD_TASK_OUTPUT_FILE`), to append
`<Name>=<JSON>` lines to, read after `onRun` exits, as GitHub Actions does with `$GITHUB_OUTPUT`. Rejected because it
adds a second channel beside the stdout messages Actions already use, with its own lifecycle, permission, and
container questions; because its values arrive only after the Action exits; and because the security argument for it
does not hold in OpenJD (see [Design Choices](#2-outputs-are-reported-with-a-stdout-directive-with-file-for-large-values)).
`@<FilePath>` keeps the one thing a file does better, carrying large values.

### Reading `@` files when the Action exits

Reading every `@` file after `onRun` exits would let a script print the directive before it writes the file. Rejected
because it delays every file value until the Task ends, so a runtime could neither validate nor forward it promptly,
and because it would make the value depend on whatever the file held at exit rather than when it was reported.

### Failing an invalid directive when the Action exits

Letting the Action run to the end and failing it then would let a long render finish after its outputs were already
known to be unusable. Rejected: the runtime cancels at once, which saves the rest of the run and makes the outcome the
same in every runtime.

### A separate `TASK_METRICS` extension with an `openjd_metric:` directive

Proposed before this RFC for per-Task measurements: numbers only, an optional declaration, last value wins, and
malformed lines ignored with a warning. Rejected in favor of treating a metric as an output variable that no Step
reads, so the system has one reporting mechanism with one set of rules (see
[Design Choices](#4-task-metrics-are-output-variables)). Its declaration properties, `unit` and `description`, and
its last-value-wins behavior, as `collect: LAST`, are part of this RFC.

### Tasks that write job outputs directly

A directive such as `openjd_job_output:` from any Task. Rejected because a Job usually has many Tasks, so the value
would need a rule for which Task wins, and because a second directive would duplicate `openjd_output_var:`. A job
output that one Task computes is an output variable of its Step read by the job output's *value*.

### Recording an errored job output and continuing

Marking a job output that fails to evaluate as errored and letting the Job run on. Rejected in favor of failing the
Job: the Job can no longer report what it declared, and finding that out at the end wastes the rest of its run.

### Storing `PATH` outputs as the worker wrote them

Rejected because a worker path is meaningless to a consumer on another host and to the submitter. Storing the
submission form, as `PATH` job parameters are stored, lets every reader map it forward with its own rules.

### Lenient handling of malformed or undeclared values

The metrics proposal ignored a malformed or undeclared line with a warning, as `openjd_progress` does. Rejected for
every output: a value dropped with a warning can't be read by a consumer, and a Task that reports a bad measurement
has produced a bad result. Failing the Task points the author at the line that caused it.

### Last value wins by default

Rejected in favor of `collect: ONCE` as the default. A second directive for an output computed once is almost always
a mistake, and with last-value-wins a stray directive would silently replace the real value. `collect: LAST` is
available where repeated reports are intended.

### Concatenating list values

A `collect` mode that splices each directive's list into the Task's value, so that `[1, 2]` then `[3]` gives
`[1, 2, 3]`. Rejected because it is ambiguous for a `LIST[LIST[INT]]` output, where a list could be either one
element or a batch, and because `APPEND` with one directive per value covers the same cases (see
[Design Choices](#3-repeated-reports-follow-a-declared-collect-mode)).

### `Step.<StepName>.<VarName>`

Proposed in #111. Rejected because `Step` already means the current Step, a Step named `Name` would be unreachable,
and every future `Step.<Property>` would be a breaking change (see
[Design Choices](#5-a-new-steps-root-with-output-and-taskparam-levels)).

### Transitive access to outputs

Letting any downstream Step read `Steps.P.*` saves a `dependsOn` line in long chains but hides data flow from readers
and tools. Rejected in favor of direct dependents only.

### References that create dependency edges

Airflow, Tekton, and CWL infer an edge from a data reference. OpenJD Steps declare their edges, and having two
sources of edges would make it harder to read a template's structure. A reference without an edge is an error that
names the fix.

### Skipping the consumers of a skipped Step (Tekton's rule)

Rejected because enablement would flow along an edge that exists only in expression text, and because the explicit
form is one line (see [Design Choices](#8-a-skipped-step-satisfies-ordering-and-yields-null-data)).

### Treating an empty computed parameter space as skipped

Rejected because every Step with a computed parameter space would then have optional outputs (see
[Design Choices](#9-an-empty-computed-parameter-space-is-a-zero-task-success)).

### Declared reductions

`reduce: MAX | MIN | SUM | LIST` on each output was implied by #111, to reduce across a producer's Tasks. Rejected
because the full list plus EXPR functions does everything a declared reduction does, lets consumers reduce
differently, and keeps the correlation between results and configurations that the sweep needs. The same reasoning
applies to reducing the values one Task reports: `collect` keeps them (`APPEND`) or picks the last (`LAST`), and never
sums or averages them.

### An outputs-only extension first

folouiseAWS's two-part plan, which would also have been the natural home for Task metrics. Rejected as a split of the
specification, though not of implementation work (see
[Design Choices](#1-variable-data-flow-is-part-of-control_flow-not-a-standalone-extension)).

### Task-task dependencies inside CONTROL_FLOW

Rejected in favor of a separate `TASK_DEPENDENCIES` extension that composes with this one (see
[Design Choices](#12-task-task-dependencies-are-a-separate-extension)).

### A record or map type for structured outputs

Passing a Task's three metrics and its configuration as one record would make the sweep read more naturally. RFC 0007
rejected map types for EXPR, and parallel lists aligned by index cover the use cases with no new type.

### Tasks that add Steps to a running Job

Houdini PDG-style dynamic work (U15) needs the step graph itself to change. Every surveyed system that allows this
loses static validation and the view of total work at the dynamic boundary. Discussion #22 pointed that use case at a
separate mechanism, standalone StepTemplates submitted into a running Job, which this RFC neither replaces nor
precludes. A loop Step (U17, [Loop Steps](#loop-steps)) is different: the step graph stays fixed and only one Step's
width grows, within a declared bound.

## Open Questions

These are to be raised on discussion #111 before this RFC leaves draft. Each has the answer this draft adopts. The
reasoning is in [04-design-space.md §6](../docs/control-flow-investigation/04-design-space.md#6-open-questions-for-the-discussion-thread),
and for the emission channel, collect modes, and metrics in [§8](../docs/control-flow-investigation/04-design-space.md#8-addendum-emission-channel-collect-modes-and-metrics).

1. **Emission channel.** The `openjd_output_var:` stdout directive, with `@<FilePath>` for large values, read as soon
   as the directive is read. Is a 64 KiB floor for a directive line right, and is limiting `@` paths to the Session's
   Working Directory too strict for any runtime?
2. **Namespace.** `Steps.<Name>.Output.<Var>` and `Steps.<Name>.TaskParam.<Param>`, and their unmapped forms
   `RawOutput` and `TaskRawParam`, with referenced Steps required to
   have `<Identifier>` names. Alternative: drop `.Output` and reserve a short list of property names.
3. **Direct dependents only.** Adopted.
4. **Output shape.** Scalar for a producer without `parameterSpace`, a list in iteration order otherwise; reductions
   are ordinary EXPR.
5. **Correlating a reduced value with its Task.** `Steps.*.TaskParam` lists now, and a separate RFC 0006 addendum for
   `argmin`/`argmax`, or a list `index`, or `zip`. Which is preferred?
6. **Fields that may reference outputs.** The materialization expressions plus host-scope script fields, and job
   output *value*s, which may read any Step; not `combination`, other job-level fields, or Environment Templates.
7. **Emission rules.** Only a Task's `onRun`; a directive anywhere else cancels and fails its Action; `collect: ONCE` by
   default, with `LAST` and `APPEND` declared; missing, malformed, undeclared, mistyped, repeated `ONCE`, or
   out-of-constraint values fail the Task, and an invalid directive cancels the Action at once; failed attempts commit
   nothing; outputs are not secrets. Does anyone need outputs from an Environment's `onEnter`?
8. **Chunked producers.** Each directive names its Task with a `[<value>]` selector on the output name. Should the
   selector name the parameter as well (`[Frame=12]`) in case a Step ever allows more than one `CHUNK[INT]`
   parameter? Committing the outputs of some Tasks of a failed chunk is out of scope here; it belongs with the
   per-Task progress of #169.
9. **Limits.** At least 1 MiB per Task, constraints checked at the producer, an optional scheduler Task-count limit
   checked at binding. Is 1 MiB the right floor?
10. **Range length.** This draft lifts the 1024-element cap for ranges given as format strings in `CONTROL_FLOW`
    templates, in favor of the scheduler's Task-count limit, as `<IntRangeExpr>` already does. Should the relaxation
    apply only to ranges that read `Steps.*`?
11. **Empty parameter spaces.** A zero-Task success; range strings still can't be empty. Separately from this RFC,
    the specification and implementations disagree on a descending range such as `1-0`: §3.4.1.1.1 defines it as
    `[1]`, and openjd-rs rejects it. That should be settled on its own.
12. **`enableIf`.** `SKIPPED` satisfies `dependsOn` once the skipped Step's own dependencies are satisfied; a skipped
    Step's symbols are `null` and typed `T?`; evaluated before step `let`; on Steps and Environments; shipped in
    `CONTROL_FLOW`. Is a warning enough where EXPR silently discards a conditional Step's `null`, or should
    it be an error? Do issue #80 users need static `enableIf` sooner than `CONTROL_FLOW` can ship?
13. **Re-runs.** A differing re-run after consumption fails; cascade recompute is a possible scheduler capability.
    Is "fail the re-run" acceptable to render-farm operators?
14. **`TASK_DEPENDENCIES` beside, not inside.** Adopted.
15. **`@` file encoding.** UTF-8 only, with a leading BOM ignored. Should UTF-16 with a BOM be accepted, given
    Windows PowerShell 5.1's defaults?
16. **Coercion strictness.** This draft converts JSON values with EXPR's implicit coercion, so `S=123` is the string
    `"123"` for a `STRING` output. Should output conversion be stricter than EXPR's?
17. **Number of outputs per Step.** This draft allows up to 32. Is a limit needed, and is that the right one?
18. **Standalone StepTemplates (#22).** Not replaced by this RFC. May a Step appended to a running Job reference
    `Steps.*`?
19. **Collect modes.** `ONCE`, `LAST`, and `APPEND`, named `collect`. Are they enough for metrics and for data flow,
    and is `collect` the right name?
20. **Uncommitted values.** A scheduler MAY show values of a running Task, marked as not committed. Should it also
    keep the values of failed attempts for display, as the metrics proposal asked, given that they are never
    visible to consumers?
21. **Unmapped paths.** A `PATH` output that no path mapping rule covers is invalid when the Session has rules. Is
    that too strict for farms whose workers share some paths with submitters without a rule for them?
22. **Job outputs.** Evaluated as soon as their inputs are known; a failure fails the Job; up to 50 per Job; not
    symbols in the template. Should a later Job be able to take an earlier Job's outputs as parameters, and if so is
    that a scheduler feature or part of OpenJD?

## Future Work

Each of these was deferred from this draft and can be added later without breaking templates written against it.
Each has a **TODO** block at the place in the specification it would change.

| Deferred | Why it can wait |
|---|---|
| Producer-declared `default` | `or` and conditional expressions cover the cases. |
| Partial reductions | Iteration order is ill-defined for an incomplete set. |
| Cascade recompute on re-run | Large blast radius; scheduler policy. |
| `enableIf` on individual host requirements | Computed amounts and attribute lists cover U7. |
| `argmin`/`argmax`, list `index`, `zip`/`enumerate` | Comprehensions work; belongs to RFC 0006. |
| Outputs from Environment actions | No use case yet. |
| Task-task dependencies | A separate `TASK_DEPENDENCIES` RFC, from #82. |
| A reducing `collect` mode (sum, min, max, count) | Consumers reduce with EXPR; only needed if collected lists become too large. |
| Passing one Job's outputs to another Job's parameters | Job outputs share the parameter type vocabulary for this; a chaining mechanism is a separate design (Open Question 22). |
| Standalone StepTemplates appended to a running Job (#22) | A different mechanism, for a separate RFC. |
| Loop Steps (U17) | Breaks "Task count known once bound" and needs sequential, incremental Task creation; a separate extension. See [Loop Steps](#loop-steps). |

### Loop Steps

A loop Step runs its Tasks one after another, and each Task's parameters are computed from the Task that ran before
it, until an expression says to stop. It would be a separate extension, provisionally `STEP_LOOPS`, that depends on
`CONTROL_FLOW`, for the same kind of reasons that task-task dependencies are separate (see
[Design Choices](#12-task-task-dependencies-are-a-separate-extension)). Everything else in this RFC assumes that a
bound Step's Tasks are all known; a loop Step's are not, and a scheduler needs a new capability to create them one at
a time and run them strictly in sequence.

A sketch, to test that this RFC doesn't block it:

```yaml
- name: Refine
  dependencies:
    - dependsOn: Setup
  parameterSpace:
    taskParameterDefinitions:
      - name: Iter
        type: INT
        initial: "{{Steps.Setup.Output.StartIter}}"
        next: "{{Task.Param.Iter + 1 if Task.Output.Residual > Param.Tolerance else null}}"
    maxTaskCount: 50
  outputVariables:
    - name: Residual
      type: FLOAT
  script:
    # ...
```

* **`initial` and `next` instead of `range`.** `initial` is the first Task's value, a materialization expression that
  may read `Steps.*`. `next` is evaluated after each Task succeeds; its value is the next Task's value, and `null` ends
  the loop. The loop parameter may be any Task parameter type, so a `STRING` continuation token works as well as an
  index.
* **The scope of `next`.** The Task that just ran: its `Task.Param.*` and its committed outputs, as a new
  `Task.Output.<Var>` symbol that exists only in `next`. That is a new evaluation point, after a Task and before the
  next one, which this RFC's stage table would gain.
* **A required bound.** Termination can't be checked statically, so `maxTaskCount` is required, in keeping with
  [Technical Requirements](#technical-requirements) item 7. Reaching it fails the Step; a loop that should stop
  quietly says so in `next`.
* **Parameters.** In the simplest form, a loop Step's parameter space has only loop parameters. A product of a loop
  with static parameters, one independent loop per combination ("refine each configuration until it converges"),
  overlaps with `TASK_DEPENDENCIES` and is a later step.
* **Consumers** read `Steps.<Loop>.Output.*` as lists in the order the Tasks ran, and are bound once the loop ends,
  under this RFC's completeness rule ([Reductions](#reductions) item 3). `collect: LAST` and `APPEND` already cover a
  single Task's final value and history.
* **Re-runs.** Each Task's outputs are consumed by the next Task's `next`, so under
  [Retries and re-runs](#retries-and-re-runs) a re-run of a Task other than the last that reports different values
  fails. The rule needs no change.
* **Sessions.** Because a loop runs in sequence, a scheduler would do well to keep its Tasks in one Session, reusing
  its Environments.
* **Not covered.** A loop over one Step can't repeat a group of Steps, such as render, check, and re-render until
  clean. Looping over part of the step graph is a much larger change, close to the dynamic work of U15.

Prior art: CWL's loop feature, whose loop condition and output method (the last iteration's value, or all of them)
correspond to `next` and to reading the last element or the whole list; Argo Workflows' recursive templates with a `when`
condition; and AWS Step Functions' Choice state that transitions back to an earlier state. Apache Airflow, GitHub
Actions, and Tekton have no loops, because their graphs are acyclic.

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is more permissive.
