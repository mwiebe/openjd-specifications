* Feature Name: step_output_variables
* Author(s): Mark Wiebe <[mwiebe](https://github.com/mwiebe)>
* RFC Tracking Issue: (pending). The numbers `0011`, `0012`, and `0013` are placeholders assigned before tracking
  issues exist, and may change when they are created.
* Start Date: 2026-10-15
* Specification Version: 2023-09 extension CONTROL_FLOW
* Accepted On: (pending)
* Part of: the `CONTROL_FLOW` extension, which this RFC defines together with
  [RFC 0011, Control Flow](0011-control-flow.md) and [RFC 0013, Step Loops](0013-step-loops.md). A template that
  lists `CONTROL_FLOW` gets all three.
* Depends On:
  * RFC 0011, Control Flow, which defines the extension name, the `Steps.*` scope through which output values are
    read, and step input binding
  * RFC 0005, Expression Language (EXPR) (https://github.com/OpenJobDescription/openjd-specifications/issues/112)
  * RFC 0007, Extended Parameter Types, whose job parameter types output variables reuse
* Related:
  * Discussion #111, variable data flow along step dependencies, which proposed the `openjd_output_var:` directive
    (https://github.com/OpenJobDescription/openjd-specifications/discussions/111)
  * The Task metrics proposal (`TASK_METRICS`, `openjd_metric:`), which this RFC replaces with output variables
  * Investigation notes behind this draft: [`docs/control-flow-investigation/`](../docs/control-flow-investigation/)

> **Draft status.** This is the first draft. It was split out of RFC 0011 so that the producer side of
> `CONTROL_FLOW` can be read and reviewed on its own. Blocks marked **TODO** are deferred (see
> [Future Work](#future-work)), and [Open Questions](#open-questions) lists the decisions to confirm on discussion
> #111.

## Summary

[RFC 0011](0011-control-flow.md) lets the Steps of a Job read values that other Steps produce while the Job runs, and
use them to compute their parameter spaces, decide whether they run, and reduce the results of a fan-out. This RFC
defines how those values are produced. A Step declares typed **output variables** in an `outputVariables` list,
typed with the job parameter type vocabulary and constrained with the same properties, and each of its Tasks reports
them by printing an **`openjd_output_var: <Name>=<JSON>` stdout directive** from its `onRun` action, or
`openjd_output_var: <Name>=@<file>` for a value too large for one line. The runtime validates each directive as soon
as it reads it, so an undeclared name, a malformed value, or a value that violates its constraints fails the Task
that printed it, and a declared **collect mode** says whether a Task reports an output once, updates it, or appends
to a list. Output variables are also how a Task reports **metrics**, such as an error score or a frame rate, for a
scheduler to show beside the Task's parameters: a metric is an output variable that no Step reads. `PATH` values are
stored in the submission path context, by applying the Session's path mapping rules in reverse, so a path that a Task
reports on a Linux worker reaches a Windows submitter in a form it can use.

## Overview

### What this RFC adds

* **Declaring outputs.** An `outputVariables` list on `<StepTemplate>`, with an optional display `unit` and a
  `collect` mode (`ONCE`, `LAST`, or `APPEND`) for repeated reports:
  [`<OutputVariableDefinition>`](#outputvariabledefinition).
* **Reporting outputs.** The `openjd_output_var:` directive, printed by a Task's `onRun` and validated strictly as soon
  as it is read, with a `[<value>]` selector naming the Task in a chunked Step: [Output emission](#output-emission).
* **Paths.** `PATH` and `LIST[PATH]` outputs, reverse-mapped to the submission path context:
  [Path values](#path-values).
* **Storage and re-runs.** When values are committed, how long they are kept, and what happens when a Task whose
  values were already read is run again: [Committing values](#committing-values) and
  [Retries and re-runs](#retries-and-re-runs).

### How it fits with RFC 0011

RFC 0011 defines the extension name, the `Steps.<StepName>.Output.<VarName>` symbols through which other Steps and job
outputs read the values this RFC produces, the types those symbols have, and the evaluation stages at which the values
are read (see [RFC 0011 §Evaluation stages](0011-control-flow.md#evaluation-stages)). A Task's committed values are its
contribution to those symbols: for a Step with a `parameterSpace`, `Steps.<StepName>.Output.<VarName>` is the list of
its Tasks' values in parameter-space iteration order (see [RFC 0011 §Reductions](0011-control-flow.md#reductions)).
Nothing in this RFC depends on whether any Step reads an output. [RFC 0013](0013-step-loops.md)'s loop variables and
loop outputs use the types and constraint properties defined here, and its loop outputs the `LAST` and `APPEND` collect
modes.

## Basic Examples

Every example in [RFC 0011 §Basic Examples](0011-control-flow.md#basic-examples) reports its outputs with
`openjd_output_var:` directives. The example here has no consumer at all, and uses each of the three collect modes.
The example in [Chunked Steps](#chunked-steps) shows the selector for a Step whose `onRun` runs a chunk of Tasks.

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
        maxLength: 20000        # keeps the list well under the guaranteed 1 MiB per Task attempt
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
* A later Step could read the same outputs as lists and pick the winner, as in
  [RFC 0011 §Parameter sweep, then run the winner](0011-control-flow.md#parameter-sweep-then-run-the-winner).

## Motivation

The use cases are those of [RFC 0011 §Motivation](0011-control-flow.md#motivation). Each of U1–U4 needs a Task to
report a value that a later Step reads, and U16, Task metrics, needs a Task to report values that a scheduler stores
and shows with no Step reading them at all. Today a Task can report only progress, a status message, a failure
message, and environment variables, so authors pack numbers into `openjd_status` messages or write files at
agreed-upon locations, which no scheduler or tool can see.

What a Task reports has to be declared, typed, and checked where it is produced, so that a template can be validated
before submission and a bad value fails the Task that produced it rather than a consumer that reads it later. The
[Technical Requirements of RFC 0011](0011-control-flow.md#technical-requirements) apply to the whole extension.
This RFC is where items 5 (fail at the cause), 7 (bounded), 8 (strict emission), 10 (composable, with a selector for
`TASK_CHUNKING`), and 11 (prompt delivery) are met.

## Design Choices

### 1. Outputs are reported with a stdout directive, with `@file` for large values

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
  size up to the per-Task-attempt limit goes through a file, and the directive that names it is short.

An earlier draft of this RFC used a session-supplied output file instead (`Task.OutputFile`, like GitHub Actions'
`$GITHUB_OUTPUT`), on the grounds that text printed by third-party tools could forge an output. That argument does
not carry over to OpenJD. Code in `onRun` already runs as the Job's user, with everything that user can do, so a value
it prints grants no capability it didn't have. GitHub's case differs: untrusted pull-request text in a log could steer
workflows that hold more privilege than its author. What remains is a correctness problem. A tool that echoes data,
such as an object key, at the start of a line might print something shaped like a directive. The rules in
[Output emission](#output-emission) handle it: a directive must name a declared output and be well formed and well
typed, and by default may appear only once per output (see
[Design Choices](#2-repeated-reports-follow-a-declared-collect-mode)). An accidental directive therefore fails the
Task instead of silently changing a value.

`@file` has one security rule. The runtime may run as a different operating-system user from the Action, with access
the Action's user lacks, so it must read the file with no more access than the Action's user has. Otherwise a Task
could copy a file it can't read into outputs that are stored and displayed. Separately, for portability, a `@` path
must resolve inside the Session's Working Directory, which also makes relative paths work the same inside a container
under `WRAP_ACTIONS`.

### 2. Repeated reports follow a declared collect mode

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
[RFC 0011 Design Choice 7](0011-control-flow.md#7-reductions-are-ordinary-expr-over-ordered-lists)), and keeping every value is what a Task that
reports measurements wants. A reducing mode can be added later if the size of collected lists becomes a problem.

### 3. Task metrics are output variables

Reporting per-Task measurements, such as an error score, coverage, frames per second, or render time, was first
proposed as a separate extension with its own directive, `openjd_metric: <name>=<number>`. In this RFC a metric is an
output variable that no Step reads. It is declared with a type and constraints, has an optional `unit` and
`description` for display, and is reported with `openjd_output_var:`. A scheduler stores and can show every Task's
committed outputs whether or not a Step reads them (see [Committing values](#committing-values)), so the same
declaration gives a UI its columns and, if a later Step needs it, gives that Step its input (see
[RFC 0011 §Parameter sweep, then run the winner](0011-control-flow.md#parameter-sweep-then-run-the-winner)).

One mechanism instead of two means one directive, one set of validation rules, and one way of storing values per Task.
A metric that later feeds a Step needs no change to the producer. The costs:

* **Metrics follow the strict rules.** An undeclared name or a malformed value fails the Task, where the metrics
  proposal ignored it with a warning. A Task whose result is a measurement, as in a sweep, has produced a bad result
  if its measurement is bad, and the author fixes the script once. A lenient mode for outputs that no Step reads would
  be a second set of rules.
* **Metrics need the whole extension.** A scheduler that wants only metrics must still accept `CONTROL_FLOW`, which
  also brings step input binding. As
  [RFC 0011 Design Choice 1](0011-control-flow.md#1-variable-data-flow-is-part-of-control_flow-not-a-standalone-extension) notes, it can build output
  collection and display first and advertise `CONTROL_FLOW` once binding works.

How results are read, such as whether lower or higher is better, a threshold such as "coverage at least 75", or
averaging over one Task parameter, stays out of the template. Those choices describe how a person reads the results,
often change after the Job has run, and may reasonably differ between schedulers.

### 4. Limits are enforced at the producer

Output values are bounded per Task attempt, and a producer's declared constraints (`maxLength: 5000`, `minValue: 1`) are
checked as its Tasks report values. A discovery Step that finds a million keys fails itself, next to the cause, with
a message naming the constraint, rather than producing a consumer that fails later with a less obvious error.
Airflow's `max_map_length`, which fails the producing task, is the precedent.

### 5. `PATH` values are stored in the submission path context

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

## Specification

Every field, symbol, and behavior in this section is part of the `CONTROL_FLOW` extension (see
[RFC 0011 §Extension name and dependencies](0011-control-flow.md#extension-name-and-dependencies)), and is
available only in a Job Template whose `extensions` list includes `CONTROL_FLOW`.

### Terminology

The terms of [RFC 0011 §Terminology](0011-control-flow.md#terminology) apply. This RFC adds:

| Term | Definition |
|------|------------|
| **Output directive** | A line `openjd_output_var: <Name>=<JSON>` or `openjd_output_var: <Name>=@<Path>` that a Task's `onRun` action prints to stdout or stderr to report an output value. |
| **Collect mode** | The `collect` property of an output variable, `ONCE`, `LAST`, or `APPEND`, which says how a Task's directives for that name become its value. |
| **Submission path context** | The form in which `PATH` job parameter values are given when a Job is submitted, before any Session's path mapping rules are applied. `PATH` output values are stored in this form. |

### Schema modifications

> Changes to [the template schema](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas).

#### `<StepTemplate>`

> A modification to [`3. <StepTemplate>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#3-steptemplate)

```diff
  name: <StepName>
  description: <Description> # @optional
  enableIf: <bool> | <FormatString> # @optional @fmtstring @extension CONTROL_FLOW
  let: <LetBindings> # @optional @extension EXPR
  dependencies: [ <StepDependency>, ... ] # @optional
  stepEnvironments: [ <Environment>, ... ] # @optional
  hostRequirements: <HostRequirements> # @optional
  parameterSpace: <StepParameterSpaceDefinition> # @optional
+ outputVariables: [ <OutputVariableDefinition>, ... ] # @optional @extension CONTROL_FLOW
  script: <StepScript>
```

New property (*enableIf* is defined in [RFC 0011 §`<StepTemplate>`](0011-control-flow.md#steptemplate)):

* *outputVariables* — The output variables that every Task of the Step reports with `openjd_output_var:` directives.
  See [`<OutputVariableDefinition>`](#outputvariabledefinition) and [Output emission](#output-emission). Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. Maximum number of elements: 32.
    3. No two elements in this list may have the same value for the `name` property.

  In a Step whose parameter space contains a `CHUNK[INT]` Task parameter, one `onRun` runs a whole chunk of Tasks,
  and each directive names the Task it reports for (see [Chunked Steps](#chunked-steps)).

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
   nesting, and a multi-Task producer's outputs are collected into a list (see [RFC 0011 §Reductions](0011-control-flow.md#reductions)).
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

### Output emission

#### The `openjd_output_var` directive

A Task of a Step that declares *outputVariables* reports an output value by printing a line to stdout or stderr from
its `onRun` action, in one of two forms:

```
openjd_output_var: <Name>=<JSON>
openjd_output_var: <Name>=@<FilePath>
openjd_output_var: <Name>[<Int>]=<JSON>        # either form, in a Step with a CHUNK[INT] parameter
openjd_output_var: <Name>[<Int>]=@<FilePath>
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
  parameter. `<Int>` is a decimal integer, an optional `-` then digits, with no whitespace inside the brackets, and
  is matched by its integer value. It is
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
      set -euo pipefail
      for frame in {{repr_sh([string(f) for f in list(Task.Param.Frame)])}}; do
        start=$(date +%s.%N)
        render --frame "$frame"
        seconds=$(awk -v a="$start" -v b="$(date +%s.%N)" 'BEGIN { printf "%.3f", b - a }')
        echo "openjd_output_var: RenderTimeS[$frame]=$seconds"
      done
```

1. **Which Tasks.** The `CHUNK[INT]` parameter's values must be distinct in a Step that declares *outputVariables*, so
   that a value names exactly one Task; a duplicate is an error when its parameter space is materialized, which fails
   Job creation if that happens at Job creation and fails the Step at step input binding otherwise. The selector must
   be a value in the chunk being run. A scheduler may form chunks differently from one run to the next, for example
   under `targetRuntimeSeconds`, so the check is against the chunk the runtime was given, not the parameter's whole
   range.
2. **Per Task.** Collect modes, constraint properties, and the size limit of [Limits](#limits) apply to each Task's
   values separately: under `ONCE`, `RenderTimeS[12]` may be reported once and `RenderTimeS[13]` once. When the
   Action exits successfully, every Task in the chunk must have each of its `ONCE` and `LAST` outputs, and each of its
   `APPEND` outputs meets its `minLength`.
3. **The chunk is one attempt.** If the Action fails, is canceled, or times out, a directive is invalid, or the exit
   check of item 2 fails for any Task of the chunk, the whole chunk fails and no Task in it commits any output, as no
   Task in a failed chunk succeeds under `TASK_CHUNKING`; a Task of the chunk that had already succeeded keeps its
   earlier state and values (see [Retries and re-runs](#retries-and-re-runs)). The selector is required whatever the
   chunk's size, including chunks of one Task.
4. **Consumers see Tasks, not chunks.** `Steps.<Name>.Output.*` and `Steps.<Name>.TaskParam.*` have one element per
   Task in parameter-space iteration order, however the Tasks were chunked.

A value, not a position within the chunk, identifies the Task, so a directive keeps its meaning however the scheduler
splits or merges chunks. A selector on each line, rather than a directive that sets a current Task for the lines
after it, can't be attributed to the wrong Task when stdout and stderr interleave.

#### Processing a directive

The runtime processes each directive when it reads the line:

1. It checks that the Action may print directives, parses the directive, and checks that the name is a declared
   output of the Step, and that a selector is present exactly when the Step is chunked and names a Task of the
   chunk.
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
   form (see [RFC 0011 §Limits](0011-control-flow.md#limits)) counts. A directive that takes the size over the runtime's
   limit is invalid.
3. A producer's constraint properties bound its values. A scheduler SHOULD encourage authors to declare `maxLength` on
   list outputs that feed a parameter space, and on `APPEND` outputs.

#### Retries and re-runs

A re-run of a Task that has already succeeded, whether or not its Step declares outputs, leaves the Task `SUCCEEDED`
for every purpose while the re-run is pending or running, and after it fails (including cancellation or timeout) or
is rejected: for `dependsOn`, for completeness ([RFC 0011 §Reductions](0011-control-flow.md#reductions) item 3), for
Step and Job success, and for loop iteration completion ([RFC 0013 §Iterations](0013-step-loops.md#iterations)). A
scheduler MAY show the re-run attempt's own status separately. Only a re-run that succeeds and commits changes
anything, under the rules below and [RFC 0011 §Job outputs](0011-control-flow.md#job-outputs) item 5.

* **Retry of a failed attempt.** A failed attempt committed nothing, so a retry starts with no values.
* **Re-run of a succeeded Task.** An output of a producer is **consumed** once a consumer that references it (as found
  by symbol collection) has evaluated a materialization group whose inputs include it, or has started a Task.
  Because a scheduler SHOULD, not MUST, evaluate a group as soon as its inputs are concrete, an eager and a lazy
  scheduler may disagree on whether an output has been consumed when a re-run commits. A
  scheduler that allows re-running a Task that already succeeded MUST apply these rules to each of the re-run Task's
  own output values:
    1. If the output has not been consumed, a successful re-run's value replaces the earlier one.
    2. If the output has been consumed and the re-run succeeds with the same value, nothing changes.
    3. If the output has been consumed and the re-run succeeds with a different value, the re-run attempt is
       reported as failed, with a message that names the output and a consumer that has consumed it. The Task keeps
       its earlier succeeded state and all of its committed values, so the Step and its consumers are unaffected.
    4. If the re-run fails (including cancellation or timeout) or is rejected, it commits nothing, as any failed
       attempt, and the committed values are unchanged. The Task keeps its earlier succeeded state and all of its
       committed values, as in rule 3.

  The check is all or nothing: if any output falls under rule 3, nothing from the re-run is committed, and in a
  chunked Step the check covers every Task of the re-run chunk. A chunk that mixes succeeded and not-yet-succeeded
  Tasks can only run before any of the Step's outputs is consumed, so rule 3 never applies to it. Values are the
  same if their converted, typed values are equal.

  So a metric that no Step reads never blocks a re-run, but a consumed value that varies from run to run, such as a
  measured frame rate that a later Step reduces, makes every differing re-run of its Task fail once it is consumed.

> **TODO (deferred): cascade recompute.** A scheduler could instead invalidate and re-bind every Step that consumed
> the changed values, transitively. That has a large blast radius and is scheduler policy, so this draft only defines
> the safe default above. A scheduler capability for cascade recompute can be added later.

### Modifications to How Jobs Are Run

> A modification to [How Jobs Are Run](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run)

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
  **Task** whose `onRun` **Action** printed it, and can only be emitted by that **Action**. In a Step with a
  `CHUNK[INT]` Task parameter, the name is followed by `[<value>]`, the chunk parameter's value for the **Task** of
  the chunk that the value is for, as in `openjd_output_var: <name>[<value>]=<json>`. The runtime reads a `@` file
  as soon as it reads the line. An invalid message, or one emitted by any other **Action** of the Job Template, cancels
  the **Action** at once and fails it, and stays in the log with the error. Under WRAP_ACTIONS, a line from a wrap hook
  counts as output of the **Action** it runs in place of. Requires the CONTROL_FLOW extension; without it, and for
  **Actions** of an Environment Template, the line is ordinary output. See [Output emission](#output-emission).

No environment variable is added.

### Validation

The rules this RFC adds, and when each is checked, in the form of
[RFC 0011 §Validation](0011-control-flow.md#validation).

**At template validation** (by the model library, before submission):

1. *outputVariables*: 1 to 32 elements; with a `CHUNK[INT]` parameter whose `range` is a literal list, its integer
   elements are distinct (a range expression cannot repeat a value, since its sub-ranges may not overlap, and
   format-string elements and computed ranges are checked when materialized); names are unique `<Identifier>`s; types
   are `<OutputVariableType>`s; constraint properties are valid for the type, and there is no `default`,
   `userInterface`, or `dataFlow`; *collect* is an `<OutputCollectMode>`, and `APPEND` only with a `LIST[...]` type;
   *unit* is 1 to 32 characters with no control characters; no `LIST[LIST[INT]]` on a Step with a `parameterSpace`.

**At step input binding** (by the scheduler, when the Step's Task enumeration group is evaluated): `CHUNK[INT]` values
are distinct in a Step that declares *outputVariables*. A failure fails the Step at once, or Job creation if the group
is evaluated then.

**As Actions run** (by the runtime, as it reads each line of an Action's output in a Session of a `CONTROL_FLOW` Job): the rules of
[Processing a directive](#processing-a-directive), including type conversion, constraint properties, collect modes,
`@` file paths, reverse mapping of `PATH` values, and the limits; and when `onRun` exits successfully, that every `ONCE` and `LAST` output has a value
and every `APPEND` list meets its `minLength` (for each Task of a chunk).

**On re-run** (by the scheduler): the rules of [Retries and re-runs](#retries-and-re-runs).

## Prior Art

The survey of comparable systems, and a table comparing how each declares, emits, and reads outputs, is in
[RFC 0011 §Prior Art](0011-control-flow.md#prior-art). What this RFC takes from them:

* **Fail the producer** on a missing output (CWL, Nextflow, Snakemake, Argo without a default) and on an oversized
  fan-out (Airflow's `max_map_length`). Tekton fails the consumer on a missing result, which is harder to diagnose.
* **Path remapping in render managers.** Thinkbox Deadline's path mapping and Pixar Tractor's dirmaps translate paths
  between submitter and worker forms. This survey found no system that specifies the reverse direction for values a
  task produces; this RFC defines it in terms of the existing rules.
* **Limits with a guaranteed minimum.** GitHub Actions allows 1 MB of outputs per job, Step Functions 256 KiB per
  state, Tekton 4 KB per TaskRun by default. This RFC guarantees at least 1 MiB per Task attempt.

What this RFC does differently:

* **A stdout directive rather than a file.** GitHub Actions, Tekton, Argo, and CWL emit outputs through a file, and
  GitHub Actions moved from its `::set-output` stdout command to the `$GITHUB_OUTPUT` file because untrusted text in a
  log could set outputs that steered more privileged workflow steps. OpenJD Actions already report everything else on
  stdout, and code in `onRun` runs as the Job's user, so a printed value grants nothing new. This RFC keeps stdout,
  requires each directive to name a declared output, and makes a repeated `ONCE` output an error, which turns an
  accidental directive into a failed Task (see
  [Design Choices](#1-outputs-are-reported-with-a-stdout-directive-with-file-for-large-values)). Argo's `result` (a
  step's stdout as an output) and TeamCity's service messages are precedents for stdout.

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

## Implementation Impact

- **Python (openjd-model)** and **Rust (openjd-rs)**: The `outputVariables` schema, with `collect`, `unit`, and the
  constraint properties, gated on the extension, and the output types from which RFC 0011 derives the types of
  `Steps.*` symbols. The model passes a Step's output declarations to the Session along with its script.
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
- **CLI (`openjd run`)**: Passes each Step's output declarations to the Session, collects the values each Task
  commits, and prints each Task's outputs, with their units.
- **Schedulers**: Storing committed output values for the life of the Job, displaying them (as metric columns beside
  each Task's parameters), optionally showing the uncommitted values of a running Task, and the re-run rule.

The parsing is JSON, and the checks are the type conversion and constraint checks that job parameters already have,
so no part of this RFC is harder in one language than another. Output values are small by design.

## Rejected Ideas

### A session-supplied output file

An earlier draft gave each Task attempt a file, `Task.OutputFile` (also `OPENJD_TASK_OUTPUT_FILE`), to append
`<Name>=<JSON>` lines to, read after `onRun` exits, as GitHub Actions does with `$GITHUB_OUTPUT`. Rejected because it
adds a second channel beside the stdout messages Actions already use, with its own lifecycle, permission, and
container questions; because its values arrive only after the Action exits; and because the security argument for it
does not hold in OpenJD (see [Design Choices](#1-outputs-are-reported-with-a-stdout-directive-with-file-for-large-values)).
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
[Design Choices](#3-task-metrics-are-output-variables)). Its declaration properties, `unit` and `description`, and
its last-value-wins behavior, as `collect: LAST`, are part of this RFC.

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
[Design Choices](#2-repeated-reports-follow-a-declared-collect-mode)).

### A record or map type for structured outputs

Passing a Task's three metrics and its configuration as one record would make the sweep read more naturally. RFC 0007
rejected map types for EXPR, and parallel lists aligned by index cover the use cases with no new type.

## Open Questions

These are to be raised on discussion #111 before this RFC leaves draft. Each has the answer this draft adopts. The
reasoning is in [04-design-space.md §6](../docs/control-flow-investigation/04-design-space.md#6-open-questions-for-the-discussion-thread),
for the emission channel, collect modes, and metrics in [§8](../docs/control-flow-investigation/04-design-space.md#8-addendum-emission-channel-collect-modes-and-metrics),
and for `PATH` outputs in [§9](../docs/control-flow-investigation/04-design-space.md#9-addendum-path-outputs-and-job-outputs).

1. **Emission channel.** The `openjd_output_var:` stdout directive, with `@<FilePath>` for large values, read as soon
   as the directive is read. Is a 64 KiB floor for a directive line right, and is limiting `@` paths to the Session's
   Working Directory too strict for any runtime?
2. **Emission rules.** Only a Task's `onRun`; a directive anywhere else cancels and fails its Action; `collect: ONCE` by
   default, with `LAST` and `APPEND` declared; missing, malformed, undeclared, mistyped, repeated `ONCE`, or
   out-of-constraint values fail the Task, and an invalid directive cancels the Action at once; failed attempts commit
   nothing; outputs are not secrets. Does anyone need outputs from an Environment's `onEnter`?
3. **Chunked producers.** Each directive names its Task with a `[<value>]` selector on the output name. Should the
   selector name the parameter as well (`[Frame=12]`) in case a Step ever allows more than one `CHUNK[INT]`
   parameter? Committing the outputs of some Tasks of a failed chunk is out of scope here; it belongs with the
   per-Task progress of #169.
4. **Limits.** At least 1 MiB per Task attempt, constraints checked at the producer, and optional scheduler per-Step
   and per-Job Task limits checked when the Step's Tasks become known
   ([RFC 0011 §Limits](0011-control-flow.md#limits) item 1). Is 1 MiB the right floor?
5. **Re-runs.** A differing re-run after consumption fails; cascade recompute is a possible scheduler capability.
   Is "fail the re-run" acceptable to render-farm operators?
6. **`@` file encoding.** UTF-8 only, with a leading BOM ignored. Should UTF-16 with a BOM be accepted, given
   Windows PowerShell 5.1's defaults?
7. **Coercion strictness.** This draft converts JSON values with EXPR's implicit coercion, so `S=123` is the string
   `"123"` for a `STRING` output. Should output conversion be stricter than EXPR's?
8. **Number of outputs per Step.** This draft allows up to 32. Is a limit needed, and is that the right one?
9. **Collect modes.** `ONCE`, `LAST`, and `APPEND`, named `collect`. Are they enough for metrics and for data flow,
   and is `collect` the right name?
10. **Uncommitted values.** A scheduler MAY show values of a running Task, marked as not committed. Should it also
    keep the values of failed attempts for display, as the metrics proposal asked, given that they are never
    visible to consumers?
11. **Unmapped paths.** A `PATH` output that no path mapping rule covers is invalid when the Session has rules. Is
    that too strict for farms whose workers share some paths with submitters without a rule for them?

## Future Work

Each of these was deferred from this draft and can be added later without breaking templates written against it.
Each has a **TODO** block at the place in the specification it would change, except the reducing `collect` mode.

| Deferred | Why it can wait |
|---|---|
| Producer-declared `default` | `or` and conditional expressions cover the cases. |
| Cascade recompute on re-run | Large blast radius; scheduler policy. |
| Outputs from Environment actions | No use case yet. |
| A reducing `collect` mode (sum, min, max, count) | Consumers reduce with EXPR; only needed if collected lists become too large. |

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is more permissive.
