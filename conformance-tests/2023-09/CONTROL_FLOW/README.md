# CONTROL_FLOW Conformance Tests

Conformance tests for the draft `CONTROL_FLOW` extension defined in
[RFC 0011 — Control Flow](../../../rfcs/0011-control-flow.md).

> **Unverified.** These tests were written from the RFC text before any implementation of
> `CONTROL_FLOW` existed, so none of them has been run. They are meant to be correct
> against the spec so that a prototype can be developed against them. If a test and the RFC
> disagree, the RFC wins: fix the test, or raise the question on the RFC.

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
outputs. `PATH` outputs are reverse-mapped to the submission path context.

## RFC rules these tests verify

The filename prefix is the [Template Schemas](../../../wiki/2023-09-Template-Schemas.md)
section; each file's header comment cites the RFC 0011 section and rule.

- **Gating** (`1.1--*`, `env_templates/*`): `CONTROL_FLOW` requires `EXPR`; every new
  field and symbol requires `CONTROL_FLOW`; Environment Templates can't declare
  `CONTROL_FLOW`, use `enableIf`, or reference `Steps.*`.
- **`<OutputVariableDefinition>`** (`job_templates/3.7--*`): each `<OutputVariableType>`,
  case-insensitive type names, 1 to 32 outputs with unique `<Identifier>` names,
  `LIST[LIST[INT]]` only without a `parameterSpace`, no `default`/`userInterface`/`dataFlow`,
  `objectType` on paths, `collect` modes (`APPEND` needs a list type), `unit` length and
  character rules, and constraint properties valid for the type.
- **The directive** (`jobs/3.7.1--*`): every row of the RFC's type conversion table that
  has a defined result, other JSON coercions implied by EXPR §1.2.3, `null`/objects/`NaN`/
  `Infinity`, the `<Directive>` grammar (whitespace placement, case-sensitive names, empty
  payload, line must begin with the prefix), CRLF and stderr, UTF-8, the 64 KiB line
  floor, logging of valid and invalid directives, and cancel-at-once on an invalid directive.
- **`@<FilePath>` values** (`jobs/3.7.1--file-*`): relative and absolute paths inside the
  Working Directory, BOM and multi-line JSON, trailing-whitespace trimming, escapes via an
  absolute path, `..`, or a symbolic link, missing file, directory, not exactly one JSON
  value, NUL, invalid UTF-8, empty path, one element per `APPEND` directive, the 1 MiB
  floor, and reading the file when the directive is processed.
- **Which Actions may print a directive** (`jobs/3.7.1--directive-*`): only `onRun` of a
  Step with `outputVariables`; Job/Step Environment `onEnter` and other Steps fail; an
  Environment Template's actions and a non-`CONTROL_FLOW` Job see ordinary output; a
  directive forwarded through `onWrapTaskRun` (`WRAP_ACTIONS`) counts for the Task, while
  one forwarded through `onWrapEnvEnter` from a Step Environment's `onEnter` is invalid.
- **Collect modes and constraints** (`jobs/3.7--*`): `ONCE` twice or missing, `LAST`
  takes the last and checks every value, `APPEND` order, `[]` when empty, a list as one
  element, `maxLength`/`minLength`/`item` checks; `minValue`/`maxValue`/`allowedValues`/
  `maxLength` violations fail the producer; only declared constraints apply.
- **Chunked Steps** (`*--chunk*`, with `TASK_CHUNKING`): the `[<value>]` selector is
  required (even for a chunk of one), forbidden without `CHUNK[INT]`, must be in the chunk
  being run (not merely in the range), and is matched by integer value; `ONCE` and the
  exit check apply per Task; duplicate chunk values are rejected in a literal list at
  validation and in a computed list at binding, before any Task runs, and are allowed
  without `outputVariables`; consumers see per-Task lists aligned with `TaskParam`.
- **The `Steps.*` scope** (`job_templates/7.3.1--*`): direct dependency only (not
  transitive, not self, checked even in an untaken branch); readable in every allowed
  field, including `hostRequirements` and an action `timeout`; known Step, declared
  outputs and Task parameters; a Step whose name isn't an `<Identifier>` is still valid;
  not in the Job name or `jobEnvironments`; `PATH`-typed `Output`/`TaskParam` only on the worker host, `RawOutput`/
  `TaskRawParam` elsewhere; scalar vs. list shape; type errors; `T?` accepted where `T` is
  required (Design Choice 8).
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
  creation and after a Step fail the run, and valid or `null` job outputs don't.

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
- **Markers are built at run time** (`print('C_' + 'RAN')`) so that a forbidden or
  expected marker never appears literally in the template, in case a runner logs the
  command line. Directive lines that are asserted in the log are built the same way.
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

- Re-runs and retries, committed-value display, forwarding, and `PENDING_INPUTS`/`SKIPPED`
  state reporting, including whether a pending Step's Tasks and host requirements are known:
  scheduler behavior that `openjd run` can't show portably.
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
- `Steps.*` in `parameterDefinitions`: no field there is a format string.
- A directive in a Job Environment's `onExit`, because whether a failed `onExit` fails a
  run whose Tasks succeeded is base-spec behavior the RFC doesn't settle.
- `"1-{{N}}"` with `N = 0`: the RFC notes implementations disagree on `1-0`.
- `combination` may not reference `Steps.*` (§Task parameter ranges): `combination` is not
  a format string, so `{{Steps.P.Output.X}}` there is a syntax error like any other
  malformed `combination`. A template can't fail this rule without failing an existing
  base-spec rule first, so a test would not show the rule is enforced.
- A referenced Step must have an `<Identifier>` name (§Validation item 3): EXPR parses
  `Steps.Find-Frames.Output.N` as `Steps.Find - Frames.Output.N`, which fails as an unknown
  Step `Find` and an unknown root `Frames`, rules that are already tested. No reference can
  name a non-`<Identifier>` Step, so the rule can't be violated distinctly.
  (`7.3.1--non-identifier-step-name-unreferenced` checks that such a Step is still valid.)

## Ambiguities found while writing these tests

1. **Path reverse mapping (§Path values).** When the Session has rules, there are two
   MUSTs: the stored value MUST be in the submission path context, and forward mapping it
   MUST give back the reported path; "If the runtime can find no such form, the value is
   invalid." A worker-only path under no rule's `destination_path` round-trips if stored
   unchanged, but it has no submission-context form, so these tests read it as invalid.
   `3.7.1--path-not-covered-by-rules` and `3.7.1--path-nested-rules-no-inverse` take that
   reading, which agrees with the reference inverse (steps 4 and 5) and its nested-rules
   example. The RFC could state it more directly, for example by saying that a value under
   no rule's `destination_path` has no submission-context form. The stored forms asserted
   in the other path tests are those of the reference inverse.
2. **Type of `RawOutput` for `PATH`.** The RFC says `RawOutput` has the same type as
   `Output` (`path`), while EXPR types `RawParam` of a `PATH` parameter as `string`
   because a submission path may not parse on the local OS.
3. **Float formatting of output values.** EXPR §1.3.4 preserves a float's original text
   when it is copied; the RFC doesn't say whether that applies to a reported `3` or `3.50`.
4. **Iteration order of a chunked list range** such as `[5, 1, 3, 2]`: list order or the
   sorted order chunks are formed in. `3.7.1--chunk-list-range-alignment` checks alignment only.
5. **Job output failure timing.** Whether Tasks that depend on the producer may be
   dispatched before the job output is evaluated. `2.17--job-output-fails-after-step` only
   asserts that the run fails.
6. **Directive coercion vs. parameter parsing.** BOOL job parameters accept `1`; EXPR
   coercion has no `int` to `bool`. The tests follow the RFC's EXPR coercion rule (`B=1`
   is invalid; `N="12"` is the INT 12).
7. **The 64 KiB line floor**: whether it includes the line terminator. The test uses a
   65,023-byte line.
8. **`@` file read time**: no bound on how soon a runtime must process a flushed line.

## Running the tests

```bash
uv run conformance-tests/run_openjd_cli_tests.py 2023-09/CONTROL_FLOW
uv run conformance-tests/run_openjd_cli_tests.py 2023-09/CONTROL_FLOW/jobs
uv run conformance-tests/run_openjd_cli_tests.py '*/CONTROL_FLOW/*/*chunk*'
```
