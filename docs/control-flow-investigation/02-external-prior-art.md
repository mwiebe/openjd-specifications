# CONTROL_FLOW investigation 02: External prior art

This note surveys how comparable workflow and DAG systems express run-time control flow, so that the
CONTROL_FLOW RFC's Prior Art section is grounded and the design avoids known pitfalls. It feeds the RFC
sections on step output variables, dynamic parameter spaces, conditional steps (#81 `enableIf`),
task-task dependencies (#82 `dependsOnSubspace`), and fan-out/fan-in.

Each system is described against the same seven questions:

1. **Declare/type** — how outputs are declared and typed.
2. **Emit** — how a running unit of work emits an output value.
3. **Reference** — how a downstream consumer refers to it.
4. **Fan-out size** — how the number of parallel units is determined, and when.
5. **Conditions** — how conditional execution is expressed, and what happens to dependents of a skipped unit.
6. **Fan-in** — how results of a fan-out are gathered or reduced.
7. **Static validation** — what can be checked before anything runs.

Terminology: "unit" means the system's schedulable thing (Argo task, Airflow task instance, GHA job,
Tekton TaskRun, ...). The closest OpenJD analog is a Task; the closest analog of a fanned-out group is a Step
with a parameter space.

Sources are the upstream documentation for each project as of October 2026. Statements about Argo,
Tekton, and Airflow behavior (limits, skip and missing-output semantics) were re-checked against the
current upstream docs while writing this note; see [References](#references).

## Summary comparison

| System | Declare / type | Emit | Reference | Fan-out size | Conditions | Fan-in | Static validation |
|---|---|---|---|---|---|---|---|
| Argo Workflows | `outputs.parameters[]` (string) + `outputs.result` (stdout); artifacts separate | File via `valueFrom.path`, or stdout (`result`, 256 KB cap) | `{{tasks.A.outputs.parameters.p}}` template substitution | `withParam` JSON array from upstream output; `withItems`/`withSequence` static | `when:` (govaluate/expr) on task; `depends:` boolean over `A.Succeeded`/`.Skipped`/`.Failed`/... | Looped task's outputs referenced as a JSON array | Schema + reference-name lint; values untyped strings |
| Apache Airflow | Implicit: Python return value → XCom; untyped (JSON-serializable) | Return value, or `xcom_push` | `XComArg` passed as task argument (TaskFlow) or Jinja `ti.xcom_pull` | `.expand()` over upstream list/dict at run time; `max_map_length` (1024) | `@task.branch` returns task IDs to follow, `@task.short_circuit`; `trigger_rule` on joins | Downstream receives lazy sequence of all mapped results | DAG parse checks graph structure only |
| Nextflow | `output:` block: `val`, `path`, `tuple`, `env`, `stdout`, `eval`; `optional: true`; `emit:` names | Files matched by glob in work dir; vals from script vars | Named channels (`proc.out.name`) wired by dataflow | Implicit: one task per channel element; unbounded, streaming | `when:` block in process; channel `filter`/`branch` operators | Channel operators `collect`, `groupTuple`, `reduce` | Limited (Groovy DSL); static typing is newer/opt-in |
| Snakemake | `output:` file patterns with wildcards; `directory()` | Files on disk | Rule `input:` file patterns; DAG inferred backward from targets | `expand()` at parse time; `checkpoint` re-evaluates DAG after a rule runs | Input functions / `branch()` choose inputs; no first-class skip | Aggregation rule whose input function reads checkpoint output | Dry run (`-n`) computes full DAG up to checkpoints |
| GitHub Actions | `jobs.<id>.outputs` map of string expressions | Append `name=value` to `$GITHUB_OUTPUT` file (stdout `::set-output` deprecated) | `${{ needs.A.outputs.x }}` | `strategy.matrix` incl. `fromJSON(needs.A.outputs.m)`; max 256 jobs | `if:` expression on job/step; dependents of skipped job skip unless `always()`/`!cancelled()` | None for matrix jobs (outputs collide); use artifacts | Syntax only on GitHub; `actionlint` adds context type checks |
| Tekton | `results[]` typed `string`/`array`/`object` (object needs `properties`) | Write to `$(results.r.path)` file (4 KB/termination message; sidecar logs for larger) | `$(tasks.A.results.r)` into params or `when` | `matrix` over array params incl. upstream results; default max 256 combinations | `when:` (`input`/`operator`/`values`, or CEL) skips the task only; ordering-dependents run, result-consumers skip | String results of a matrixed task aggregated into array: `results.r[*]` | Admission webhook checks types, references, DAG; result presence is runtime |
| AWS Step Functions | No declarations; state input/output is JSON | Task state's output JSON (filtered via `ResultSelector`/`ResultPath`, or JSONata `Output`/`Assign`) | JSONPath (`$.x`) / JSONata over state input, or workflow variables (`$var`) | `Map` state over a JSON array (`ItemsPath`/`Items`) or S3 (`ItemReader`, Distributed mode) | `Choice` state rules (comparison operators or JSONata `Condition`) with `Default` | Map output is an ordered array (or `ResultWriter` to S3); `Parallel` merges branches | Validates ASL structure; no data typing |
| CWL | Typed `outputs:` (`File`, `Directory`, `int`, `string`, arrays, records, enums, `null` unions) | `outputBinding` (`glob`, `loadContents`, `outputEval`) or `cwl.output.json` | `source: stepA/out` | `scatter:` over array inputs (`ScatterFeatureRequirement`); `scatterMethod` dot/cross product | `when:` JS expression (v1.2); skipped step outputs are `null` | Scatter outputs become arrays; `linkMerge`, `pickValue` (`first_non_null`, `all_non_null`, ...) | Strong: source/sink type compatibility checked by validators |
| Slurm | None (files on shared FS) | Files / exit code | Script convention | `--array=0-999%50` fixed at submit; `MaxArraySize` | `--dependency=afterok/afternotok/afterany`; logic in scripts | Dependent job on whole array (`afterok:<array>`) | Submit-time syntax only |
| HTCondor DAGMan | None (files) | Files / exit codes | `VARS` macros, file conventions | `queue N` / `queue from file` at submit; `SUBDAG EXTERNAL` read when node starts | PRE/POST scripts; `PRE_SKIP`, `ABORT-DAG-ON`; POST exit decides node success | `PARENT a b c CHILD d` | DAG file parse; sub-DAGs validated when reached |

### Size and fan-out limits

Every system with value passing bounds the size of passed values and the width of a fan-out. The limits
and *where the failure surfaces* matter as much as the numbers.

| System | Value-size limit | Fan-out limit | Failure mode on exceeding |
|---|---|---|---|
| Argo | `result` captures up to 256 KB of stdout; whole workflow status bounded by etcd object size (~1 MB, mitigated by node-status offload) | `parallelism` controls concurrency, not count | Large workflows fail on persistence |
| Airflow | XCom backend–dependent (DB column); custom XCom backends for large data | `max_map_length` default 1024 | **Producer** task fails if it returns a longer list |
| GitHub Actions | 1 MB per job output set, 50 MB per workflow run | 256 jobs per matrix | Workflow/job error |
| Tekton | 4096 bytes termination message per TaskRun (shared by all results); sidecar-log mode default 4 KB per result, configurable | `default-max-matrix-combinations-count` 256 | TaskRun fails; PipelineRun fails validation if matrix too big |
| Step Functions | 256 KiB state payload | Inline Map: 40 concurrent iterations; Distributed Map: up to 10,000 concurrent child executions | `States.DataLimitExceeded` |
| Slurm | n/a | `MaxArraySize` (site configured) | Submit rejected |

## Per-system notes

### Argo Workflows

* **Declare/type.** A template declares `outputs.parameters` (name, `valueFrom`, optional `default`) and
  `outputs.artifacts`. Values are strings; structured data is passed as JSON text. Every script/container
  template also has an implicit `outputs.result` holding stdout (up to 256 KB).
* **Emit.** `valueFrom.path` reads a file the container wrote, so emission is file-based. `valueFrom` can
  also take a JSONPath/jq over a resource, a supplied value, or an expression. If the file is absent and no
  `default` is declared, the node fails.
* **Reference.** Template substitution: `{{tasks.A.outputs.parameters.p}}` (DAG) or
  `{{steps.A.outputs.parameters.p}}` (steps), also usable in `when` and `withParam`. An `expr` mode
  (`{{= ...}}`) allows expressions over them.
* **Fan-out.** `withItems` (literal list), `withSequence` (numeric range), and `withParam` (a JSON array,
  typically an upstream output). Size is known only when the upstream completes. An empty `withParam`
  produces no children and the node is marked skipped.
* **Conditions.** `when:` is a govaluate/expr boolean evaluated after substitution — a string-pasting model
  that is prone to quoting bugs. `depends:` is a boolean over upstream states (`.Succeeded`, `.Failed`,
  `.Errored`, `.Skipped`, `.Omitted`, `.Daemoned`, and for loops `.AnySucceeded`/`.AllFailed`). Argo
  distinguishes **Skipped** (own `when` false) from **Omitted** (`depends` not satisfied). A bare
  dependency means `Succeeded || Skipped || Daemoned`, so by default a skipped upstream does *not* block
  dependents. References to outputs of skipped/omitted nodes resolve to `""` unless the output declares
  `valueFrom.default`.
* **Fan-in.** Referencing a looped task's output parameter yields a JSON array of every iteration's value.
* **Static validation.** `argo lint` checks schema and that referenced templates/tasks exist. It cannot
  check value types since everything is a string.

### Apache Airflow

* **Declare/type.** No declaration. A TaskFlow task's Python return value becomes an XCom; typing is
  whatever serializes. `multiple_outputs=True` splits a dict into keyed XComs.
* **Emit.** Return value, or explicit `xcom_push`. Stored in the metadata DB or a pluggable XCom backend
  (commonly object storage for big values).
* **Reference.** Passing an `XComArg` as a function argument builds both the dependency edge and the
  data reference; classic operators use Jinja `{{ ti.xcom_pull(...) }}`.
* **Fan-out.** Dynamic task mapping: `task.partial(const=...).expand(x=upstream())`. Multiple `expand`
  kwargs produce a cross product; `.zip()` produces a dot product; `.map(fn)` transforms items;
  `expand_kwargs` takes a list of dicts. Mapped task groups chain a per-index sub-pipeline (an implicit
  task-to-task dependency). `max_map_length` (default 1024) bounds the width, and exceeding it **fails the
  producing task**. A zero-length input creates no instances and marks the mapped task SKIPPED.
  `TriggerRule.ALWAYS` is rejected at parse time for task-generated mapping because the expansion is
  undefined until upstream finishes.
* **Conditions.** `@task.branch` returns the task ID(s) to follow; the rest are skipped.
  `@task.short_circuit` skips everything downstream on a falsy value. Skips propagate, so a join after a
  branch needs `trigger_rule="none_failed_min_one_success"` — the classic "join never runs" pitfall.
* **Fan-in.** A downstream task receiving a mapped output gets a `LazySelectSequence`. Materializing it
  with `list()` loads every value; Airflow warns when a lazy proxy is pushed to XCom because of the cost.
* **Static validation.** DAG parsing validates graph shape and some mapping rules (e.g. the
  `TriggerRule.ALWAYS` restriction). No data typing.

### Nextflow

* **Declare/type.** A process's `output:` block declares qualifiers `val`, `path` (glob), `tuple`, `env`,
  `stdout`, `eval`, optionally named with `emit:` and marked `optional: true`.
* **Emit.** Files matching the `path` glob in the task work directory, values from script variables or
  captured env/stdout. A non-optional output that is not produced fails the task.
* **Reference.** Dataflow channels: `proc.out.name` is wired to another process's input. There is no
  separate dependency declaration; edges are the channels.
* **Fan-out.** Implicit and streaming: each item arriving on an input channel launches a task. Width is
  unbounded and never known up front; `flatten`, `splitCsv`, etc. produce items.
* **Conditions.** `when:` block on a process, and channel `filter`/`branch` operators route items.
  A filtered-out item simply never produces a task; there is no "skipped" state to propagate.
* **Fan-in.** Channel operators: `collect()` (wait for channel close, emit list), `groupTuple(by:, size:)`,
  `reduce`, `join`. `groupTuple` without `size` blocks until the upstream channel closes, which serializes
  pipelines; supplying the expected group size lets groups be released early.
* **Static validation.** The Groovy-based DSL historically gave little before-run checking; recent
  releases add an opt-in static-typing syntax. Wiring errors often surface at run time.

### Snakemake

* **Declare/type.** Rules declare `output:` file patterns with wildcards (and `directory()` outputs).
  The "type" is a file path.
* **Emit.** Files. A missing declared output raises `MissingOutputException` (after `--latency-wait`
  for shared filesystems) and the job fails.
* **Reference.** Consumers declare `input:` patterns; Snakemake infers the DAG backward from requested
  targets by matching patterns, so references are by file name, not by producer.
* **Fan-out.** `expand()` at parse time for known sets. For data-dependent sets a rule is declared
  `checkpoint`; once it finishes, the scheduler **re-evaluates the DAG**, and an aggregation rule's input
  function calls `checkpoints.<rule>.get(...)` plus `glob_wildcards` to discover the new jobs.
* **Conditions.** Input functions (and `branch()`) choose inputs at DAG-build time; there is no
  first-class run-time skip.
* **Fan-in.** An aggregation rule whose inputs are the expanded list.
* **Static validation.** A dry run computes the whole DAG up to the first unresolved checkpoint.

### GitHub Actions

* **Declare/type.** `jobs.<id>.outputs` maps names to expressions over step outputs. Values are strings;
  structured data is JSON text decoded with `fromJSON`.
* **Emit.** A step appends `name=value` (or a heredoc for multi-line) to the file named by
  `$GITHUB_OUTPUT`. The older stdout workflow command `::set-output` was **deprecated because any process
  writing to the log — including untrusted tool output — could inject or overwrite outputs**. Outputs that
  contain secrets are masked and not passed.
* **Reference.** `${{ needs.A.outputs.x }}` in a job that lists `A` in `needs`.
* **Fan-out.** `strategy.matrix`, which may be `${{ fromJSON(needs.A.outputs.matrix) }}`. Cross product
  of axes plus `include`/`exclude`. Max 256 jobs per matrix. An empty dynamic matrix is a workflow error,
  so authors guard with `if: needs.A.outputs.matrix != '[]'`.
* **Conditions.** `if:` on jobs and steps using the GHA expression language. A dependent of a skipped or
  failed job is skipped unless its `if:` includes a status function (`always()`, `!cancelled()`,
  `failure()`); `needs.A.result` exposes `success|failure|cancelled|skipped`. A referenced output that
  was never set evaluates to the empty string — no error.
* **Fan-in.** No aggregation: matrix jobs that set the same output overwrite each other non-
  deterministically. The idiom is to upload per-job artifacts and download them in a later job.
* **Static validation.** GitHub validates YAML syntax and expression syntax. The third-party `actionlint`
  type-checks contexts, including `needs.*.outputs` names.

### Tekton Pipelines

* **Declare/type.** A Task declares `results[]` with `type: string | array | object`; object results must
  declare `properties`. Pipeline results are expressions over task results.
* **Emit.** A step writes to `$(results.<name>.path)`. Results travel through the Kubernetes termination
  message (4096 bytes per TaskRun, shared by all results); an opt-in sidecar-log mode raises this
  (4 KB per result by default, configurable). Exceeding the limit fails the TaskRun.
* **Reference.** `$(tasks.A.results.r)`, `$(tasks.A.results.arr[*])`, `$(tasks.A.results.obj.key)` in
  params or `when`. A reference also creates the ordering edge.
* **Missing output.** "Tekton does not enforce that results are produced at Task level. If a pipeline
  attempts to consume a result that was declared by a Task, but not produced, it will fail"
  (`InvalidTaskResultReference`). TEP-0048 proposes result defaults. A missing result consumed by a
  pipeline result on a skipped task is simply omitted.
* **Fan-out.** `matrix` on a PipelineTask fans out over array params (literal or from upstream results):
  cross product plus explicit `include` combinations. Default max 256 combinations. An empty array param
  skips the PipelineTask.
* **Conditions.** `when` expressions (`input`, `operator: in|notin`, `values`) or CEL. A false `when`
  skips **only that task**: tasks that depend on it purely for ordering (`runAfter`) still run, while
  tasks that consume its results are skipped for lack of the result (and their dependents with them). To
  guard a task together with its dependents, authors copy the `when` onto each dependent or wrap the group
  in a Pipeline-in-Pipeline. `finally` tasks can inspect `$(tasks.A.status)` and `$(tasks.status)`.
* **Fan-in.** String results of each matrixed TaskRun are aggregated into an array consumed as
  `$(tasks.A.results.r[*])`; consuming a single element or array/object results from a matrix is not
  supported. `$(tasks.A.matrix.length)` and `...matrix.r.length` expose counts.
* **Static validation.** The admission webhook checks param/result types, reference names, and DAG
  acyclicity before anything runs. Whether a result is produced is a run-time property.

### AWS Step Functions

* **Declare/type.** No output declarations. Each state transforms a JSON document; Task states call a
  service and the response becomes output.
* **Emit.** The task's result, shaped by `ResultSelector`/`ResultPath`/`OutputPath` (JSONPath mode) or
  `Output` and `Assign` (JSONata mode, which also introduced workflow-scoped variables).
* **Reference.** JSONPath/JSONata over the current state's input, or `$variable` references.
* **Fan-out.** `Map` iterates over an array from the state input (`ItemsPath`/`Items`) with
  `ItemSelector` to shape each item and `MaxConcurrency`. Distributed mode reads items from S3
  (`ItemReader`), batches them (`ItemBatcher`, analogous to OpenJD task chunking), runs child executions
  at up to 10,000-way concurrency, and supports `ToleratedFailurePercentage`/`Count`.
* **Conditions.** `Choice` state with ordered rules (comparison operators or a JSONata `Condition`) and a
  `Default`; if nothing matches and there is no `Default`, the execution fails with
  `States.NoChoiceMatched`. Branching is a state transition, not a skip flag, so "dependents of a skipped
  state" does not arise.
* **Fan-in.** Map output is an ordered array of iteration results (or written to S3 by `ResultWriter`
  when large). `Parallel` produces an array of branch outputs.
* **Static validation.** ASL structural validation at create time (and by the validator API). No data
  typing; path errors such as a missing field are runtime `States.Runtime` failures.

### Common Workflow Language (CWL)

* **Declare/type.** Every input and output is typed: `File`, `Directory`, `string`, `int`, `float`,
  `boolean`, arrays, records, enums, and optional (`null`-union, written `type?`).
* **Emit.** Tools capture outputs via `outputBinding` (`glob`, `loadContents`, `outputEval` expression) or
  by writing `cwl.output.json`. A required output that is not produced is a `permanentFail`.
* **Reference.** A workflow step input names its `source: stepA/out`; edges are inferred from sources.
  `valueFrom` (with `StepInputExpressionRequirement`) transforms values; expressions are JavaScript under
  `InlineJavascriptRequirement`.
* **Fan-out.** `scatter:` lists array-typed inputs to iterate over (`ScatterFeatureRequirement`);
  `scatterMethod` is `dotproduct`, `nested_crossproduct` (nested output arrays), or `flat_crossproduct`.
  Width is the runtime length of those arrays.
* **Conditions.** `when:` (v1.2) is a boolean expression over step inputs. A skipped step's outputs are
  `null`, so its outputs must be typed optional or merged.
* **Fan-in.** Scattered outputs are arrays shaped by `scatterMethod`. Multiple sources merge with
  `linkMerge` (`merge_nested`, `merge_flattened`), and `pickValue` (`first_non_null`,
  `the_only_non_null`, `all_non_null`) resolves conditional branches — the explicit answer to "which of
  these possibly-skipped producers supplies the value".
* **Static validation.** Strongest in the survey: `cwltool --validate` and others check that each sink's
  type accepts its sources' types, accounting for scatter (adds an array level) and `when` (adds `null`).

### Slurm (traditional HPC baseline)

* **Outputs.** None in the scheduler; data passes through a shared filesystem by convention.
* **Fan-out.** Job arrays, `--array=0-999%50` (`%` caps concurrency); size fixed at submission and bounded
  by the site's `MaxArraySize`. Each element sees `SLURM_ARRAY_TASK_ID`.
* **Dependencies/conditions.** `--dependency=afterok:<id>`, `afternotok`, `afterany`, `singleton`, and
  `aftercorr:<array>` — element *i* of one array waits on element *i* of another, the same idea as #82
  task-task dependencies. Conditional logic lives in scripts.
* **Dynamic structure.** A job computes its fan-out and calls `sbatch` itself. Effective but invisible to
  the scheduler until it happens, so there's no pre-run view of the whole job.

### HTCondor DAGMan (traditional HPC baseline)

* **Outputs.** None; files plus exit codes.
* **Fan-out.** `queue N` or `queue <var> from <file>` in a submit description, expanded when the node is
  submitted. `SUBDAG EXTERNAL` nodes point at a DAG file that is read when the node starts, so an earlier
  node or a PRE script can **generate** the sub-DAG — the classic render-farm way to make structure
  data-dependent. `SPLICE` includes are static.
* **Conditions.** PRE/POST scripts around each node. The POST script's exit code decides node success
  (so it can turn a failure into success or vice versa). `PRE_SKIP <node> <code>` skips a node when the
  PRE script exits with that code; `ABORT-DAG-ON` stops the DAG on a node exit value. `RETRY` handles
  transient failures; rescue DAGs record progress for resubmission.
* **Fan-in.** `PARENT a b c CHILD d`.
* **Static validation.** The DAG file is parsed at submit; generated sub-DAGs are only seen when reached.

## Cross-cutting observations

**Two strategies for dynamic structure.** Systems divide into:

1. *Fixed graph, late-bound width* — Argo, Airflow, GHA, Tekton, Step Functions, CWL. The set of
   units-of-definition (templates, tasks, jobs, steps) is fixed when the run is created. Only the number of
   instances of a fanned-out unit, and whether a unit is skipped, depends on run-time data. The whole graph
   can be drawn and validated up front; some nodes are just "width not yet known".
2. *Graph re-evaluation / generation* — Snakemake checkpoints, DAGMan `SUBDAG EXTERNAL`, Slurm
   self-submission, Nextflow streaming. More expressive, but the full structure is unknowable before
   running, static validation stops at the dynamic boundary, and the scheduler cannot show total work.

**Reference creates the edge, or not.** Airflow (`XComArg`), Tekton, CWL, and Nextflow derive dependency
edges from data references. GHA and Argo require the explicit edge (`needs`, `dependencies`/`depends`)
and a reference without it is an error or empty. OpenJD has explicit `dependencies`; whether
`Step.<Name>.<Var>` implies one is a design choice to make deliberately.

**Values are mostly strings.** Argo and GHA pass strings and push JSON-in-a-string through `fromJSON`
or `withParam`. Tekton and CWL type their values and get earlier, clearer errors.

**Skip semantics are a recurring source of bugs.** Each system picks a different default for dependents
of a skipped unit: Argo lets them run (bare dependency includes `Skipped`) and resolves outputs to `""`;
Airflow and GHA cascade the skip and require special trigger rules / status functions on joins; Tekton
runs ordering-only dependents but skips data dependents (the skip follows the data); CWL turns outputs
into `null` and requires `pickValue`.

**Missing declared outputs.** Three behaviors appear:

* Fail the producer — CWL, Nextflow, Snakemake, Argo (when no `default`). Error surfaces next to its cause.
* Fail the consumer — Tekton (`InvalidTaskResultReference`). The producer reports success, which is
  confusing to diagnose.
* Silently empty — GHA (empty string), Argo for skipped nodes (`""`). Least friction, most latent bugs.

**Producer-side limits are kinder than consumer-side ones.** Airflow's `max_map_length` fails the task that
returned the oversized list, which is where the fix belongs. Most limits (GHA, Tekton, Step Functions)
fail on size of *values*; Tekton also caps fan-out *width* with a validation error.

**Fan-in shape and ordering.** Where reductions exist, results come back as an ordered array keyed by
fan-out index (Step Functions Map, Tekton `[*]`, Argo loop outputs, CWL scatter, Airflow lazy sequence).
Multi-dimensional fan-outs need a defined shape (CWL `nested_crossproduct` vs `flat_crossproduct`).
Airflow's lazy-proxy warnings show that eagerly materializing a large reduction is a real cost.

**Emission channel security.** GHA's move from stdout `::set-output` to the `$GITHUB_OUTPUT` file is the
single most relevant cautionary tale: any tool that echoes attacker-influenced text (file names, logs from
third-party software) could forge outputs. Argo, Tekton, and CWL all use files.

**Task-to-task dependencies are rare but proven.** Slurm `aftercorr` and Airflow mapped task groups
implement "instance *i* waits on instance *i*". Nextflow gets it implicitly from item-wise channel
flow. Neither GHA nor Tekton matrix supports it; users fall back to whole-step barriers.

**Re-running producers is rarely specified.** DAG systems mostly treat a run as one-shot (Airflow "clear
downstream" is the exception). Render farms routinely requeue completed tasks, so OpenJD must define what
happens to outputs and downstream tasks when an upstream task is re-run.

## Lessons for OpenJD

Each lesson names the OpenJD concept it applies to; these are inputs to the RFC, not decisions.

1. **Type step output variables.** Declare `outputVariables` with OpenJD EXPR types (RFC 0005/0007: `INT`,
   `FLOAT`, `STRING`, `PATH`, `BOOL`, `LIST[...]`, ...). Tekton and CWL show that typed outputs let a
   validator check every `Step.<Name>.<Var>` reference, every `range` expression built from it, and every
   condition, at template-validation time. Untyped string outputs (GHA, Argo) push those errors to run time.
2. **Keep the step graph fixed; late-bind only widths and enablement.** Follow strategy 1 above: the
   set of steps and their dependency edges is fixed at job creation; a step's parameter space size and
   `enableIf` value may depend on upstream outputs. This preserves static validation, lets a scheduler show
   the full job, and maps onto a single new state — "not yet evaluable" until upstream outputs exist.
   Leave graph generation (Snakemake checkpoints, `SUBDAG EXTERNAL`) out of scope.
3. **Emit outputs through a file, not only stdout.** OpenJD's `openjd_env`/`openjd_progress` stdout
   protocol is convenient, but GHA's `::set-output` deprecation shows that log-scraped outputs are
   spoofable by any tool output. Prefer a session-provided file path (environment variable) for
   `openjd_output_var`, or at minimum restrict stdout emission to declared names and document the risk.
4. **Fail the producer on a missing declared output.** When a task completes successfully but has not
   emitted a declared output without a default, fail that task (CWL/Nextflow/Snakemake/Argo behavior), so
   the error is reported where it was caused. Allow an optional declared `default`, as Argo's
   `valueFrom.default` and Tekton's TEP-0048 do.
5. **Define limits, and fail at the producer.** Specify a minimum guaranteed output-value size and a
   maximum dynamic fan-out width (as `max_map_length`, Tekton's matrix cap, GHA's 256), and require that an
   oversized output or an over-wide computed range fails the producing task. Today's limits (1024-element
   range lists, 1024-character strings) are a starting point; the range-expression rule that the task count
   is implementation-limited already sets the precedent.
6. **Specify zero-width fan-out.** OpenJD range lists currently require at least one element. When a range
   is computed, an empty result needs a defined meaning; Airflow and Tekton treat it as **skipped**, GHA
   errors, Argo skips. Skipped (with dependents governed by the same rule as `enableIf: false`) is the most
   useful default.
7. **Pick one skip-propagation rule and make it explicit.** Every surveyed system that has skips has a
   pitfall around joins. For `enableIf`, decide whether dependents of a disabled step run or skip.
   Tekton's rule — ordering-only dependents run, dependents that reference its outputs skip — is a good
   candidate. Also define what `Step.<Disabled>.<Var>` evaluates to: an error, a declared default, or
   `null` combined with a CWL `pickValue`-like expression. Avoid silent empty strings (GHA, Argo).
8. **Reductions return ordered, shaped lists.** A reference to an output of a multi-task step from a
   downstream step should be a `LIST[T]` ordered by the producer's parameter-space iteration order, and
   the spec must say how multi-dimensional spaces flatten (CWL's nested vs flat cross product). Reuse the
   existing combination expression (`A * B` cross, `(A, B)` zip) semantics rather than new ones; Airflow
   (`expand` vs `zip`), Tekton (`matrix` + `include`), and CWL (`scatterMethod`) all needed the same two
   operators.
9. **Task-task dependencies have precedent.** Slurm `aftercorr` and Airflow mapped task groups validate
   #82's `dependsOnSubspace`: per-index dependencies avoid whole-step barriers and are a big win for
   render pipelines (render frame *i* → composite frame *i*).
10. **Use EXPR for conditions and computed ranges.** Every system grew or adopted an expression language
    for this (govaluate/expr, GHA expressions, CEL, JSONata, JavaScript). OpenJD already has one (RFC 0005,
    0006); conditions and computed ranges should be EXPR expressions whose free variables determine
    *when* they can be evaluated: job-parameter-only expressions at job creation, output-dependent
    expressions when the referenced steps finish.
11. **Define requeue semantics.** Render-farm users requeue completed tasks. The RFC must say whether
    re-running a producer invalidates the computed values (and thus parameter spaces and enablement) of
    downstream steps, and what happens to downstream tasks that already ran. Surveyed DAG systems mostly do
    not answer this, so OpenJD can't borrow a solution.
12. **Don't make the default reduction eager.** Airflow's lazy-proxy warnings show that loading every
    value from a wide fan-out is costly. Keep reductions bounded by the size limits in lesson 5, and keep
    bulk data as file outputs (job attachments / `PATH` values) rather than output variables.

## References

* Argo Workflows: [Output parameters](https://argo-workflows.readthedocs.io/en/latest/walk-through/output-parameters/),
  [Loops](https://argo-workflows.readthedocs.io/en/latest/walk-through/loops/),
  [Conditionals](https://argo-workflows.readthedocs.io/en/latest/walk-through/conditionals/),
  [Enhanced depends logic](https://argo-workflows.readthedocs.io/en/latest/enhanced-depends-logic/)
* Apache Airflow: [Dynamic task mapping](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/dynamic-task-mapping.html),
  [XComs](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/xcoms.html),
  [Branching and trigger rules](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#control-flow)
* Nextflow: [Process outputs](https://www.nextflow.io/docs/latest/process.html#outputs),
  [Channel operators](https://www.nextflow.io/docs/latest/reference/operator.html)
* Snakemake: [Rules](https://snakemake.readthedocs.io/en/stable/snakefiles/rules.html),
  [Data-dependent conditional execution (checkpoints)](https://snakemake.readthedocs.io/en/stable/snakefiles/rules.html#data-dependent-conditional-execution)
* GitHub Actions: [Passing information between jobs](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/passing-information-between-jobs),
  [Matrix strategies](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow),
  [Deprecating save-state and set-output commands](https://github.blog/changelog/2022-10-11-github-actions-deprecating-save-state-and-set-output-commands/),
  [actionlint](https://github.com/rhysd/actionlint)
* Tekton: [Tasks (results)](https://tekton.dev/docs/pipelines/tasks/#emitting-results),
  [Pipelines (results, when)](https://tekton.dev/docs/pipelines/pipelines/),
  [Matrix](https://tekton.dev/docs/pipelines/matrix/),
  [TEP-0048](https://github.com/tektoncd/community/blob/main/teps/0048-task-results-without-results.md)
* AWS Step Functions: [Map state](https://docs.aws.amazon.com/step-functions/latest/dg/state-map.html),
  [Choice state](https://docs.aws.amazon.com/step-functions/latest/dg/state-choice.html),
  [Quotas](https://docs.aws.amazon.com/step-functions/latest/dg/service-quotas.html)
* CWL v1.2: [Workflow description](https://www.commonwl.org/v1.2/Workflow.html) (scatter, `when`,
  `pickValue`, `linkMerge`)
* Slurm: [Job arrays](https://slurm.schedmd.com/job_array.html),
  [`sbatch --dependency`](https://slurm.schedmd.com/sbatch.html#OPT_dependency)
* HTCondor DAGMan: [DAGMan workflows](https://htcondor.readthedocs.io/en/latest/automated-workflows/index.html)
