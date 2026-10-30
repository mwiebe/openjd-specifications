# CONTROL_FLOW Conformance Tests

Conformance tests for the draft `CONTROL_FLOW` extension defined in
[RFC 0011 — Control Flow](../../../rfcs/0011-control-flow.md),
[RFC 0012 — Step Output Variables](../../../rfcs/0012-step-output-variables.md), and
[RFC 0013 — Step Loops](../../../rfcs/0013-step-loops.md).

> **Status.** There are 416 tests: 4 in `env_templates`, 199 in `job_templates`, and 213 in
> `jobs`. All 416 pass on Linux against the openjd-rs prototype (branch
> `control-flow-prototype`, commit `6553952`), the only implementation that has run them.
> The tests were written from the RFC text, and the prototype was developed against them.
> If a test and the RFC disagree, the RFC wins: fix the test, or raise the question on the
> RFC.

## What this extension does

`CONTROL_FLOW` (which requires `EXPR`) lets a Step declare typed `outputVariables` that
each Task reports with an `openjd_output_var: <Name>=<JSON>` (or `=@<file>`) stdout
directive. Steps that depend *directly* on the producer read the values as
`Steps.<Step>.Output.<Var>` (plus `TaskParam`, `RawOutput`, and `TaskRawParam`), so a Step
can compute its parameter space (dynamic fan-out), decide whether it runs (`enableIf` on
Steps and Environments), and reduce the values of a multi-Task producer (fan-in). Fields
that read upstream outputs are evaluated at a new stage, *step input binding*, group by
group as the values each group reads arrive (`enableIf` first), and no Task runs until
every group has been evaluated. A Job can
also declare `jobOutputDefinitions`, typed values computed from parameters and Step
outputs. `PATH` outputs are reverse-mapped to the submission path context. A `<StepLoop>`
entry in `steps` runs a body of Steps once per iteration, one iteration after another,
carrying typed loop variables (`Loop.<Name>`, `Loop.Iteration`) from one iteration to the
next with `next`, until `while` or `until` says to stop, within a required `maxIterations`;
Steps after the loop read its declared `LAST` or `APPEND` outputs.

## RFC rules these tests verify

The filename prefix is the [Template Schemas](../../../wiki/2023-09-Template-Schemas.md)
section; each file's header comment cites the RFC (0011, 0012, or 0013) that holds the rule,
and its section.

- **Gating** (`1.1--*`, `env_templates/*`): `CONTROL_FLOW` requires `EXPR`; every new
  field and symbol requires `CONTROL_FLOW`; Environment Templates can't declare
  `CONTROL_FLOW`, use `enableIf`, or reference `Steps.*` or `Loop.*`. An Environment
  Template can't list `CONTROL_FLOW`, so these rules give the same result as extension
  gating; the `jobs/*-in-environment-template-with-control-flow-job` tests check that a
  Job's `CONTROL_FLOW` doesn't make `enableIf` or `Steps.*` available to the Environment
  Templates used with it, with `jobs/4--environment-template-in-control-flow-job-runs` as
  their valid companion.
- **`<OutputVariableDefinition>`** (`job_templates/3.7--*`): each `<OutputVariableType>`,
  case-insensitive type names, 1 to 32 outputs with unique `<Identifier>` names,
  `LIST[LIST[INT]]` only without a `parameterSpace`, no `default`/`userInterface`/`dataFlow`,
  `objectType` on paths, `collect` modes (`APPEND` needs a list type), `unit` length and
  character rules, and constraint properties valid for the type (all 12 types).
- **The directive** (`jobs/3.7.1--*`): every row of the RFC's type conversion table that
  has a defined result, other JSON coercions implied by EXPR §1.2.3, `null`/objects/`NaN`/
  `Infinity`, the `<Directive>` grammar (whitespace placement, case-sensitive names, empty
  payload, line must begin with the prefix), CRLF and stderr, UTF-8, the 64 KiB line
  floor (a line of exactly 65,536 bytes), logging of valid and invalid directives, and cancel-at-once on an invalid directive.
- **`@<FilePath>` values** (`jobs/3.7.1--file-*`): relative and absolute paths inside the
  Working Directory, BOM and multi-line JSON, trailing-whitespace trimming, escapes via an
  absolute path, `..`, or a symbolic link, missing file, directory, not exactly one JSON
  value, NUL, invalid UTF-8, empty path, one element per `APPEND` directive, the 1 MiB
  floor (an `@` file of exactly 1,048,576 bytes), and reading the file when the directive
  is processed.
- **Which Actions may print a directive** (`jobs/3.7.1--directive-*`): only `onRun` of a
  Step with `outputVariables`; Job/Step Environment `onEnter` and other Steps fail; an
  Environment Template's actions and a non-`CONTROL_FLOW` Job see ordinary output; a
  directive forwarded through `onWrapTaskRun` (`WRAP_ACTIONS`) counts for the Task, while
  one forwarded through `onWrapEnvEnter` from a Step Environment's `onEnter` is invalid.
- **Collect modes and constraints** (`jobs/3.7--*`): `ONCE` twice or missing, `LAST`
  takes the last and checks every value, `APPEND` order, `[]` when empty, a list as one
  element, `maxLength`/`minLength`/`item` checks; `minValue`/`maxValue`/`allowedValues`/
  `maxLength` violations fail the producer; only declared constraints apply; a `LAST`
  value that a later directive replaces stops counting toward the 1 MiB floor.
- **Chunked Steps** (`*--chunk*`, with `TASK_CHUNKING`): the `[<value>]` selector is
  required (even for a chunk of one), forbidden without `CHUNK[INT]`, must be in the chunk
  being run (not merely in the range), is matched by integer value, may be negative, and
  has no whitespace inside its brackets; `ONCE` and the
  exit check apply per Task; duplicate chunk values are rejected in a literal list at
  validation and in a computed list at binding, before any Task runs, and are allowed
  without `outputVariables`; consumers see per-Task lists aligned with `TaskParam`.
- **The `Steps.*` scope** (`job_templates/7.3.1--*`): direct dependency only (not
  transitive, not self, checked even in an untaken branch); readable in every allowed
  field, including `hostRequirements` and an action `timeout`; known Step, declared
  outputs and Task parameters; a Step whose name isn't an `<Identifier>` is still valid;
  not in the Job name or `jobEnvironments`; `PATH`-typed `Output`/`TaskParam` only on the worker host, `RawOutput`/
  `TaskRawParam` elsewhere; scalar vs. list shape; type errors; `T?` accepted where `T` is
  required (RFC 0011 Design Choice 5).
- **Data flow, reductions, and parameter spaces** (`jobs/7.3.1--*`, `jobs/3.4--*`,
  `job_templates/3.4--*`): iteration order with `combination`, literal-list order,
  `TaskParam` alignment, Step Environment variables from outputs, dynamic fan-out from lists and range strings, empty spaces as
  zero-Task successes (including associated ranges), `[]` outputs of an empty producer,
  min/max/sum/nested lists, metrics with no consumer, and the 1024 cap lifted for computed
  ranges but kept for list literals.
- **Step input binding** (`jobs/7.4--*`, `jobs/3.2--*`): `let` and `hostRequirements`
  bound from outputs; `fail(...)` or a required `null` fails the Step before any Task runs;
  a consumer that reads a failed producer's output in its `range` never runs (the failed
  attempt's value is not committed); a skipped Step in a chain keeps ordering. Progressive
  binding: a `false` `enableIf` skips a parameter space that would fail; no other group is
  evaluated while `enableIf` is undecided, at Job creation or later; a group whose inputs
  are concrete at Job creation is evaluated then and its failure fails Job creation before
  any Step runs (and the same template with a valid value runs, so rejecting the template
  doesn't pass); a group that fails once `enableIf` is `true` fails the Step while another
  group still waits for its inputs (the failure only, not its timing); a consumer's range
  that reads `Steps.P.TaskParam.*` is evaluated at Job creation when `P`'s Tasks are known
  then, even though `P` is pending because its `hostRequirements` read another Step's output
  (`7.4--taskparam-concrete-before-bound-*`), and so is a job output that reads it
  (`2.17--job-output-taskparam-before-bound-*`).
- **`enableIf`** (`job_templates/3--*`, `job_templates/4--*`, `jobs/3--*`, `jobs/4--*`):
  allowed and forbidden symbols per location, `bool` type, static and dynamic decisions,
  `null` outputs of a skipped Step (`== null`, `or`), skipped Steps don't evaluate their
  `let`, a `null` `enableIf` fails (on a Step or a Step Environment), disabled Job and
  Step Environments aren't entered, and a skipped Step whose own dependency failed never
  satisfies `dependsOn` (failure is not skipping).
- **Job outputs** (`*2.17--*`): count and name rules, separate namespace from parameters,
  `LIST[LIST[INT]]`, any Step readable without a dependency, path rules (`RawParam`,
  `RawOutput`, and `TaskRawParam` allowed; `Param` of `PATH` and `LIST[PATH]` parameters
  and `TaskParam` of a `PATH` Task parameter forbidden), other forbidden symbols, type and `collect`; at run time, failures at Job
  creation and after a Step fail the run, valid or `null` job outputs don't, and job
  outputs totaling exactly 1 MiB are accepted.
- **Loop schema** (`1.1--loop-*`, `job_templates/3.8--*`, `3.8.1--*`, `3.8.2--*`, `3.8.3--*`):
  gating; no `script` or `parameterSpace` on a loop; names unique across top-level Steps,
  loops, and body Steps (also across two loops, and a loop and its own body Step); top-level
  Steps depend only on top-level Steps; exactly one of `while` and `until`; `maxIterations`
  required, 1 to 1,000, from `Param.*` but not from `Steps.*` or `Loop.*`; no nested loops;
  a non-empty, acyclic body whose dependencies stay inside it; `while` can't read body Steps;
  `until` is `bool`; 1 to 32 variables, unique, not named `Iteration`, with no `unit`,
  `initial` without `Loop.*` and only from the loop's dependencies, `next` with the
  variable's type; constraint properties valid for the type (all 12 types; none on `BOOL`,
  none from another type, `minValue` not above `maxValue`); `PATH` variables through
  `RawLoop.*` in TEMPLATE scope; 1 to 32 loop outputs, which need `value` and `collect`
  (`LAST` or `APPEND`, not `ONCE`, `APPEND` only for a list, its value of the element
  type), with unique names, `LAST` scalar and `APPEND` list types, the `unit` rules, and
  constraint properties valid for the type.
- **Loop scopes** (`job_templates/3.8.4--*`): a body Step reads the loop's dependencies, but
  not other outside Steps nor body Steps it doesn't depend on; `until`/`next`/`value` read
  any body Step; outside Steps and job outputs read loop outputs, never body Steps, and
  need a dependency on the loop; `Loop.*` only in the loop and its body (not in the loop's
  `enableIf` or `maxIterations`, a top-level Step, a job output, or a Job Environment), and
  only the loop's own variables; no `TaskParam` or undeclared outputs of a loop.
- **Running loops** (`jobs/3.8.5--*`): `until` after each iteration, with `APPEND`
  history, `LAST` final value, and `Loop.Iteration` from 0; `while` before each iteration,
  including zero iterations (`LAST` is `null`, `APPEND` is `[]`), and an `APPEND` output's
  `minLength` checked against `[]` after zero iterations, failing the loop; a two-Step
  body whose check drives `next`; every `next` reading the old values (a swap); `until`
  true skips `next`; the bound fails a loop that would continue, from a literal or a
  parameter, but not one whose condition ends it in the last allowed iteration; a failed
  body Step, a `fail(...)` in `next` or in a loop output `value`, and a loop variable
  constraint violation, including one from `next` at the final boundary of a `while` loop
  that `while` would have ended, each fail the loop and stop its iterations and its
  dependents; a body fan-out from a loop variable with its own Tasks per iteration; a
  fan-out after the loop over an `APPEND` output; a skipped loop's `null` outputs; a body
  Step skipped in some iterations; body Steps reading the loop's dependency; and a job
  output from a loop output, valid and failing. Loop binding: `maxIterations` from a
  parameter outside 1 to 1,000, and an `initial` that reads only parameters and fails,
  fail Job creation (with a valid companion); an `initial` that fails, is `null`, or
  violates a constraint, and a first `while` that fails or is `null`, fail the loop before
  iteration 0. At the boundary, a `fail(...)` in `until` or `while`, a `null` `until`,
  `while`, `next`, or loop output `value`, a `LAST` output's constraint, an `APPEND`
  output's `item` constraint or `maxLength`, and its `minLength` when `until` ends the loop
  fail the loop. A body Step whose range fails at
  the start of an iteration fails the loop. Size: loop values totaling exactly 1 MiB are
  accepted, with each `APPEND` element counted on its own and replaced loop variable and
  `LAST` output values not counted.

## Conventions in this directory

- **Extensions.** Every template lists `EXPR` and `CONTROL_FLOW` (plus `TASK_CHUNKING`, or
  `FEATURE_BUNDLE_1` for a format-string amount `min` or action `timeout`, where needed)
  unless the test is about gating. Environment templates used under `environments:` list
  only what they need.
- **Observing values portably.** Output values are never asserted from a CLI's own
  printing of outputs. A consumer Step that depends on the producer prints them:

  ```python
  import json; print('GOT:' + json.dumps(json.loads({{repr_py(repr_json(Steps.P.Output.V))}})))
  ```

  `repr_json` renders the value and the `json.loads`/`json.dumps` round trip normalizes
  whitespace, so the expected line (`GOT:[1, 2]`, `GOT:"a=b"`, `GOT:true`) doesn't depend
  on `repr_json`'s spacing. Float results are checked through arithmetic that gives a
  non-integral result (`Steps.P.Output.V + 0.5`), because how an integral float prints
  is not pinned down for output values (see *Ambiguities*). `PATH` values are printed with
  `repr_py`, which accepts both `path` and `string`.
- **Job outputs aren't printed by any template**, so execution tests only check that a
  failing job output fails the run and that valid (or `null`) job outputs don't affect a
  successful one. Their values are left to implementation tests.
- **Failures.** A run that must fail is `.invalid.test.yaml`. Where the test also asserts
  output (a forbidden marker, a logged line), it uses `expected.taskFailure` instead. Where
  a runtime that wrongly ignored the line under test would still fail the Task for a
  missing value, the output is declared `collect: LAST` and the producer first prints a
  valid directive for it, so only the line under test can fail the run. (The
  `indented-directive-not-recognized` and `chunk-task-missing-value` tests rely on the
  missing value on purpose.)
- **Loop failures.** A loop that must fail at run time is asserted with
  `expected.taskFailure` and markers, not `.invalid.`: the iteration markers it must print
  (`IT_0`, ...) show the template was accepted and how far it ran, and the forbidden ones
  show where it stopped. A test of the bound makes its condition true a little after
  `maxIterations` (for example `until: Loop.Iteration >= 4` with `maxIterations: 3`), so a
  runtime that ignored the bound succeeds and prints a forbidden marker instead of hanging.
  Only `3.8.5--job-output-from-loop-output-fails` is `.invalid.`, as the other job output
  failure tests are, with `3.8.5--job-output-from-loop-output-valid` as its companion.
- **Markers are built at run time** (`print('C_' + 'RAN')`) so that a forbidden or
  expected marker never appears literally in the template, in case a runner logs the
  command line. Directive lines that are asserted in the log are built the same way. No
  expected or forbidden marker is a substring of another line the same template prints
  (`X_0`, not `T_0`, beside `IT_0`).
- **Timing.** `3.7.1--invalid-directive-cancels-at-once` sleeps 30 s after an invalid
  directive and forbids the line printed after the sleep, so a runtime that waits for the
  Action to exit instead of canceling it fails the test by printing that line. A runner
  timeout under 30 s only saves time; it isn't needed to catch the failure. `3.7.1--file-read-when-directive-processed` waits 3 s after
  printing a flushed `@` directive before overwriting the file.
- **Platform.** Actions use `python -c`. Tests that need symbolic links or POSIX path
  mapping are `runOn: [posix]`. Path mapping uses one rule, `/mnt/submit` (submission,
  `source_path`) to `/worker/root` (worker, `destination_path`): a Task reports
  `/worker/root/...`, `RawOutput` is `/mnt/submit/...`, and `Output` maps forward to
  `/worker/root/...` again.
- **Temporary files.** The `@` file escape tests create one small file with
  `tempfile.mkstemp` that the cancelled Action can't remove.
  `3.2--skipped-step-keeps-chain-order` passes a marker through a file in the system temp
  directory, so it requires the Steps to run on one host, as with `openjd run`.

## Not covered

- Re-runs and retries, including the recovery of a loop, which is `FAILED` until every
  failed body Step instance of the iteration in progress has succeeded after a retry or
  requeue of its Tasks and then returns to `RUNNING`, and the rule that no recovery
  follows a final loop failure, committed-value display, forwarding, and
  `PENDING_INPUTS`/`SKIPPED` state reporting, including whether a pending Step's Tasks and
  host requirements are known: scheduler behavior that `openjd run` can't show portably.
- *When* a Step fails at step input binding after Job creation. Evaluating a group as soon as
  its inputs are concrete is a SHOULD, an early and a late failure end the same way (the Step
  never runs and the run fails), and the RFC doesn't say whether Steps independent of a failed
  Step keep running, so no marker can portably show the time.
  `7.4--failure-with-pending-remaining-group` asserts only the failure; the Job-creation case,
  which is a MUST, is `7.4--static-group-fails-job-creation`.
- Which error is reported when several materialization groups of a Step become evaluable at
  once and more than one would fail: the RFC fixes the order (enablement, Task enumeration,
  host selection, Task graph, remaining), but message text is not portable.
- The validation *warning* for a conditional Step's `null` discarded by EXPR: warnings
  don't change `openjd check`'s exit status.
- Line-limit and value-size *rejection* above 64 KiB / 1 MiB: runtimes MAY accept more,
  so only the floors are tested.
- Reading `@` files with the Action user's access (needs a cross-user setup), and FIFOs
  (a runtime that blocks would hang the runner).
- A scheduler's Task-count limit, and `TASK_DEPENDENCIES` composition.
- For loops: Session reuse across iterations, a scheduler's display of iterations, re-runs
  of body Tasks that have already succeeded (the Task stays `SUCCEEDED` under RFC 0012
  §Retries and re-runs, so a re-run never fails the loop and the iteration boundary does
  not wait for it), a
  scheduler's limits on total Tasks or body Step instances, how a loop and its body Steps
  count as Steps of the template (the base specification sets no limit on the number of
  Steps), and chunked body Steps.
- The loop rules of `openjd run`, which are CLI-specific: `--tasks` and `--task-param` are
  rejected whenever the selected Step is a loop, an explicitly given `--maximum-tasks` is
  rejected with `--step` naming a loop, and otherwise `--maximum-tasks` applies to each body
  Step instance, so a body Step instance that does not run all its Tasks stops the loop
  before its boundary.
- `Steps.*` in `parameterDefinitions`: no field there is a format string.
- A directive in a Job Environment's `onExit`, because whether a failed `onExit` fails a
  run whose Tasks succeeded is base-spec behavior the RFC doesn't settle.
- `"1-{{N}}"` with `N = 0`: the RFC notes implementations disagree on `1-0`.
- `combination` may not reference `Steps.*` (RFC 0011 §Task parameter ranges): `combination` is not
  a format string, so `{{Steps.P.Output.X}}` there is a syntax error like any other
  malformed `combination`. A template can't fail this rule without failing an existing
  base-spec rule first, so a test would not show the rule is enforced.
- A referenced Step must have an `<Identifier>` name (RFC 0011 §Validation item 2): EXPR parses
  `Steps.Find-Frames.Output.N` as `Steps.Find - Frames.Output.N`, which fails as an unknown
  Step `Find` and an unknown root `Frames`, rules that are already tested. No reference can
  name a non-`<Identifier>` Step, so the rule can't be violated distinctly.
  (`7.3.1--non-identifier-step-name-unreferenced` checks that such a Step is still valid.)

## Ambiguities found while writing these tests

Items 9 and later concern loops (RFC 0013).

1. **Path reverse mapping (RFC 0012 §Path values).** When the Session has rules, there are two
   MUSTs: the stored value MUST be in the submission path context, and forward mapping it
   MUST give back the reported path; "If the runtime can find no such form, the value is
   invalid." A worker-only path under no rule's `destination_path` round-trips if stored
   unchanged, but it has no submission-context form, so these tests read it as invalid.
   `3.7.1--path-not-covered-by-rules` and `3.7.1--path-nested-rules-no-inverse` take that
   reading, which agrees with the reference inverse (steps 4 and 5) and its nested-rules
   example. The RFC could state it more directly, for example by saying that a value under
   no rule's `destination_path` has no submission-context form. The stored forms asserted
   in the other path tests are those of the reference inverse.
2. **Type of `RawOutput` for `PATH` (RFC 0011 §The `Steps.*` scope).** RFC 0011 says
   `RawOutput` has the same type as `Output` (`path`), while EXPR types `RawParam` of a
   `PATH` parameter as `string` because a submission path may not parse on the local OS.
3. **Float formatting of output values (RFC 0012 §Processing a directive).** EXPR §1.3.4
   preserves a float's original text when it is copied; RFC 0012 doesn't say whether that
   applies to a reported `3` or `3.50`.
4. **Iteration order of a chunked list range (RFC 0012 §Chunked Steps)** such as
   `[5, 1, 3, 2]`: list order or the sorted order chunks are formed in.
   `3.7.1--chunk-list-range-alignment` checks alignment only.
5. **Job output failure timing (RFC 0011 §Job outputs).** Whether Tasks that depend on the
   producer may be dispatched before the job output is evaluated.
   `2.17--job-output-fails-after-step` only asserts that the run fails.
6. **Directive coercion vs. parameter parsing (RFC 0012 §Processing a directive).** BOOL
   job parameters accept `1`; EXPR coercion has no `int` to `bool`. The tests follow
   RFC 0012's EXPR coercion rule (`B=1` is invalid; `N="12"` is the INT 12).
7. **The 64 KiB line floor (RFC 0012 §Limits)**: whether it includes the line terminator.
   `3.7.1--directive-line-64kib` uses a line of exactly 65,536 bytes not counting the
   terminator, so a runtime whose 65,536-byte limit includes the terminator fails it. The
   RFC could say which it means.
8. **`@` file read time (RFC 0012 §File values)**: no bound on how soon a runtime must
   process a flushed line.
9. **Other body Steps after a failure (RFC 0013 §Failure).** RFC 0013 lets instances of
   the same iteration that don't depend on a failed one keep running. Tests forbid only
   markers of dependents of the failed instance and of later iterations.
10. **Iteration order is not observed directly (RFC 0013 §Iterations).** That iteration
    `N + 1` starts only after iteration `N` ends is implied by the values the tests check
    (each iteration reads `next` from the one before), not by timing.
11. **A loop's own failure is not a Task failure (RFC 0013 §Failure).** A boundary error or
    the bound fails the loop, not a Task, but `openjd run` reports it as a failed run,
    which `taskFailure` accepts; its exit-code check is not used.

## Running the tests

```bash
uv run conformance-tests/run_openjd_cli_tests.py 2023-09/CONTROL_FLOW
uv run conformance-tests/run_openjd_cli_tests.py 2023-09/CONTROL_FLOW/jobs
uv run conformance-tests/run_openjd_cli_tests.py '*/CONTROL_FLOW/*/*chunk*'
```
