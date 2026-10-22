# ENV_PARAMS Conformance Tests

Conformance tests for the `ENV_PARAMS` extension defined in
[RFC 0010 — Environment Parameters](../../../rfcs/0010-environment-parameters.md).

This README is also the plan for the suite: the [coverage matrix](#coverage-matrix) lists every test file,
and [Validation table coverage](#validation-table-coverage) maps each row of the RFC's
[Validation](../../../rfcs/0010-environment-parameters.md#validation) table to the tests that verify it. Rules for
which no definite test can be written are listed in
[Rules without a definite test](#rules-without-a-definite-test), and what the suite leaves out on purpose in
[What the suite does not test](#what-the-suite-does-not-test). [AUDIT.md](AUDIT.md) records the audit of the suite
against the specification.

## What this extension does

`ENV_PARAMS` lets each Step tell an Environment what it needs. An `<Environment>` declares parameters, a Step requests
the Environment and supplies values through a new `environment:` form of `<StepDependency>`, and the Environment reads
the values as `{{Env.Param.<name>}}`. The Environment may be one of the Job Template's own `jobEnvironments` or one
that the scheduler attaches from an Environment Template, matched by name when the Job is created.

```yaml
# Environment Template (§1.2)
extensions: [ <ExtensionName>, ... ]                  # NEW — per document; may list ENV_PARAMS

# <Environment> (§4), on jobEnvironments and an Environment Template's environment only
envParameterDefinitions: [ <EnvironmentParameterDefinition>, ... ]  # NEW — 1–50, unique names (§4.5)
activation: ALWAYS | ON_REQUEST                         # NEW — default ALWAYS
hostRequirements: <HostRequirements>                   # NEW — may read Env.Param.* (§3.3.3)

# <StepDependency> (§3.2), in a Step's dependencies
dependsOn: "<StepName>"
# or, NEW, a synonym of dependsOn (dependsOn is planned for removal in the next major version):
step: "<StepName>"
# or, NEW:
environment: "<EnvironmentName>"   # a jobEnvironments entry first, else an attached Environment (§3.2.1)
required: <bool>                   # optional, default true; not false for a jobEnvironments entry
enableIf: <bool> | <FormatString>  # optional, requires EXPR; resolved at job creation
parameters:                        # optional, 1–50 keys
  <Identifier>: <EnvironmentParameterValue>   # literal or format string, resolved at job creation
```

```yaml
<EnvironmentParameterDefinition> ::= a <JobParameterDefinition> of the same type, except:
  type:           # not RANGE_EXPR; BOOL and LIST[*] require EXPR, as for Job Parameters
  userInterface:  # not permitted
  objectType:     # not permitted (PATH, LIST[PATH])
  dataFlow:       # not permitted (PATH, LIST[PATH])
  default:        # optional; may read only Param.* (not PATH) / RawParam.* of the same document
                  # a parameter without a default is required (constraint 7)
```

| Value | Type (with `EXPR`) | Available in |
|---|---|---|
| `Env.Param.<name>` | the `Param.<name>` type of the parameter's type | the declaring Environment only: `variables`, script `let`, actions, embedded files, and `hostRequirements` (not PATH / LIST[PATH]); PATH values are path-mapped |
| `Env.RawParam.<name>` | as `Env.Param.<name>`, but `string` / `list[string]` for PATH / LIST[PATH] | as `Env.Param.<name>`; never path-mapped |
| `Job.EnvNames` | `list[string]` | every format string resolved at job creation or later, in a document that lists `ENV_PARAMS` and `EXPR`; not an Environment parameter `default` or the Job `name` |

Values are fixed per Step when the Job is created. For each Step and each Environment its Sessions enter, a parameter's
**effective value** is the value the Step's enabled entry supplies, or else the parameter's resolved `default`
(§3.2.2). A Session enters an Environment with `activation: ALWAYS` always, and one with `activation: ON_REQUEST` only
when its Step requests it.

## RFC rules these tests verify

- **Extension gating** (RFC §Backward compatibility, §4, §3.2, §7.3.1): Environment `envParameterDefinitions`,
  `activation`, `hostRequirements`, and `environment:` and `step:` entries require `ENV_PARAMS` in the document that uses them;
  `enableIf`, BOOL and LIST values and types, `Step.Name`, and `Job.EnvNames` also require `EXPR`; `Job.EnvNames`
  requires `ENV_PARAMS` too. `ENV_PARAMS` does not require `EXPR`.
- **Extensions are per document** (§1.2 item 3): a Job Template without `ENV_PARAMS` runs with an Environment
  Template that lists it, taking every default and never entering its `ON_REQUEST` Environments; an Environment
  Template without `ENV_PARAMS` is an `ALWAYS` Environment with no parameters.
- **Environment `envParameterDefinitions`** (§4 item 3, §4.5): 1–50 entries, unique names that may coincide with Job
  Parameters and other Environments' parameters, not on `stepEnvironments`; permitted types and fields; `default`
  scope; a literal `default` checked at template validation, a non-literal one only where a Step takes it.
- **`activation`** (§4 item 4): `ALWAYS` or `ON_REQUEST`, not on `stepEnvironments`; an `ON_REQUEST` Environment
  is entered, and its `variables` set, only in the Sessions of Steps that request it, and entered Environments keep
  their order.
- **Environment `hostRequirements`** (§4 item 5, §3.3.3): §3.3 shape per source, restricted scope, resolved per Step
  with its effective values, combined with the Step's by conjunction, conflicts (empty intersection) rejected at job
  creation, and applied only to Sessions that enter the Environment.
- **`<StepDependency>` `environment:` entries** (§3.2 constraints 1–9): one key per entry; no entry for the Step's own
  Step Environment; no duplicates, even with complementary `enableIf`; 1–50 `parameters`; keys declared; values valid
  for their parameter; required parameters supplied by every Step that enters the Environment; `required: false` not
  on a `jobEnvironments` entry; `enableIf` a `bool` literal or single `bool` expression, never `null`.
- **Value scope** (§3.2): a `parameters` value or `enableIf` reads `RawParam.*`, non-PATH `Param.*`, and with `EXPR`
  `Job.Name`, `Job.EnvNames`, `Step.Name`, and the Step's `let`; never `Task.*`, `Session.*`, or `Env.*`.
- **`Env.Param.*` / `Env.RawParam.*` scope** (§4, §7.3.1): only inside the declaring Environment, only for declared
  names; a Step's own script cannot read them; a wrapping Environment's hooks read its own parameters.
- **External resolution** (§3.2.1): a name resolves to the Job Template's `jobEnvironments` first, else to exactly
  one attached Environment; missing with `required: true` and ambiguous names are rejected; `required: false`
  tolerates a missing Environment but not one that lacks a supplied parameter; each resolved entry applies
  independently.
- **Effective values** (§3.2.2, §1.2.1): supplied value or `default`; a `default` reads the merged Job Parameter
  value; format strings convert to the parameter's type; Environment parameters are never merged with Job
  Parameters; values reach `onEnter`, Tasks (through `variables`), and `onExit` unchanged.
- **Path values** (§4.5): PATH and LIST[PATH] values are absolute in the submission path format (or URIs with
  `EXPR`), never joined with a directory, checked unmapped, path-mapped in `Env.Param.*` and not in `Env.RawParam.*`.
- **`Job.EnvNames`** (§7.3.1): the names of every Job Environment, own and attached, whatever its `activation`, in
  entry order, excluding Step Environments, with a name repeated when two attached Environments share it.
- **`enableIf` on the entry** (§3.2 item 5): resolved before the name, so a disabled entry is as if absent: nothing
  is requested or supplied, a missing `required: true` Environment is not an error, and an `ALWAYS` Environment is still
  entered with its defaults.

## How these tests are designed

The RFC motivates the extension with Conda, Docker, and license Environments, but none of them is needed to test the
mechanism. The tests use Environments whose `onEnter` and `onExit` print sentinel lines built from their parameters
(`ENTER Conda Packages=ffmpeg=7`), and Steps whose Tasks print a marker (`TASK A`). A value reached an action when its
sentinel appears; an Environment was not entered when a sentinel it would print with its `default` is `forbidden`.
Each sentinel is `echo` with several arguments (`args: [ENTER, Conda, "Packages={{Env.Param.Packages}}"]`), so the
line that `echo` prints never appears in the template text, and a forbidden line cannot match a template that a
runner happens to log. A Task that reports an environment variable builds its line by concatenation
(`print('TASK_PKG='+os.environ['PKG'])`) for the same reason.

`openjd run` runs each Step of the Job in its own Session, one after another, so a test with two Steps observes each
Step's Environment entry separately, and the output of both Steps is in one log. Output assertions are substring
matches with no order, so order is observed within one line: `Job.EnvNames` is printed as
`{{ Job.EnvNames.join(',') }}`, and the order in which a Session enters Environments is observed through a variable
that each `onEnter` appends to with `openjd_env` and the Task prints (POSIX `bash` only).

Rules about which Sessions may share Steps (§3.2.2 equality) are scheduling rules. `openjd run` never puts two Steps
in one Session, so equality is observable only as "both values are accepted" (see
[Rules without a definite test](#rules-without-a-definite-test)).

Directory rules, from the task plan:

- A rule checked at template validation over one document is a `job_templates/` or `env_templates/` test, run with
  `openjd check`.
- A rule checked at job creation or later is a `jobs/` test; `.invalid.test.yaml` when the run must be rejected.
  Every rule that needs both a Job Template and an attached Environment Template is a `jobs/` test.
- The `template` of a `jobs/*.invalid.test.yaml` file MUST pass `openjd check` on its own, and so MUST each
  Environment Template in `environments`, so that the rejection can only come from job creation. The runner cannot
  distinguish a validation failure from a job-creation failure, so a test author verifies this by hand when writing
  the test.

Commands: tests that only print values use `echo` with `runOn: [posix, windows]`. Tests that inspect exact argv or
build a variable across Environments use `bash` and `runOn: [posix]`. Tests that print an environment variable from
a Task use `python -c`, the suite's portable interpreter. PATH tests have `-posix` and `-windows` variants, because
`openjd run` takes the host's path format as the submission path format and a literal path is absolute in only one.

Each test file starts with a comment naming the RFC or wiki section and the one rule it checks.

## Layout

```
ENV_PARAMS/
├── README.md        (this file)
├── AUDIT.md         (audit of the suite against the specification)
├── job_templates/   # Job Template validation (openjd check)
├── env_templates/   # Environment Template validation (openjd check)
└── jobs/            # Job creation and execution (openjd run, with --env for each attached Environment)
```

Filenames follow `<section>--<description>[.invalid][.test].yaml`, where `<section>` is a section of
[2023-09-Template-Schemas](../../../wiki/2023-09-Template-Schemas.md): `1.1`/`1.2` (document roots and per-document
extensions), `1.2.1` (merging), `3.2`/`3.2.1`/`3.2.2` (`<StepDependency>`, resolution, effective values), `3.3.3`
(combined host requirements), `4` (`<Environment>` `envParameterDefinitions`, `activation`, `hostRequirements`), `4.5`
(`<EnvironmentParameterDefinition>`), and `7.3.1` (`Env.Param.*`, `Env.RawParam.*`, `Job.EnvNames`).

## Coverage matrix

Columns: **ID** is the identifier used in [Validation table coverage](#validation-table-coverage); **File** is the
planned filename in that directory (`.invalid` in the name means the document or run must be rejected); **Exts**
lists the extensions the document under test declares, where it matters (`EP` = `ENV_PARAMS`, `EX` = `EXPR`,
`WA` = `WRAP_ACTIONS`, `FB1` = `FEATURE_BUNDLE_1`, `—` = none; `env` marks the attached Environment Template's list where it differs); **Assertion** is the one thing the test checks.

### `job_templates/` — Job Template validation

#### Extension gating

| ID | File | Exts | Assertion |
|---|---|---|---|
| JT-01 | `4--parameter-definitions-without-extension.invalid.yaml` | — | A `jobEnvironments` entry with `envParameterDefinitions` is rejected without `ENV_PARAMS`. |
| JT-02 | `4--activation-without-extension.invalid.yaml` | — | `activation` is rejected without `ENV_PARAMS`. |
| JT-03 | `4--host-requirements-without-extension.invalid.yaml` | — | `hostRequirements` on an Environment is rejected without `ENV_PARAMS`. |
| JT-04 | `3.2--environment-entry-without-extension.invalid.yaml` | — | An `environment:` entry is rejected without `ENV_PARAMS`. |
| JT-05 | `3.2--enable-if-without-expr.invalid.yaml` | EP | `enableIf: true` (a literal) is rejected without `EXPR` (constraint 9). |
| JT-06 | `3.2--list-value-without-expr.invalid.yaml` | EP | A list literal in `parameters` is rejected without `EXPR`, even for an entry naming an external Environment. |
| JT-07 | `3.2--bool-value-without-expr.invalid.yaml` | EP | A boolean literal in `parameters` is rejected without `EXPR`, even for an external entry. |
| JT-08 | `3.2--step-name-value-without-expr.invalid.yaml` | EP | A `parameters` value reading `Step.Name` is rejected without `EXPR`. |
| JT-09 | `7.3.1--env-names-without-expr.invalid.yaml` | EP | `Job.EnvNames` is rejected without `EXPR`. |
| JT-10 | `7.3.1--env-names-without-env-params.invalid.yaml` | EX | `Job.EnvNames` is rejected without `ENV_PARAMS`. |
| JT-11 | `4.5--bool-type-without-expr.invalid.yaml` | EP | An Environment parameter of type BOOL is rejected without `EXPR`. |
| JT-12 | `4.5--list-type-without-expr.invalid.yaml` | EP | An Environment parameter of type LIST[STRING] is rejected without `EXPR`. |
| JT-13 | `1.1--minimal-env-params.yaml` | EP | A Job Environment with one parameter and a Step entry that supplies it is valid. |
| JT-14 | `4.5--base-types-without-expr.yaml` | EP | STRING, INT, FLOAT, and PATH Environment parameters are valid without `EXPR`. |

#### `<Environment>` `envParameterDefinitions` (§4 item 3) and `<EnvironmentParameterDefinition>` (§4.5)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JT-15 | `4--parameter-definitions-empty.invalid.yaml` | EP | An empty `envParameterDefinitions` list is rejected. |
| JT-16 | `4--parameter-definitions-51.invalid.yaml` | EP | 51 parameters are rejected. |
| JT-17 | `4--parameter-definitions-50.yaml` | EP | 50 parameters are valid. |
| JT-18 | `4--parameter-definitions-duplicate-name.invalid.yaml` | EP | Two parameters of one Environment with the same name are rejected. |
| JT-19 | `4--parameter-name-same-as-job-parameter.yaml` | EP | An Environment parameter may share its name with a Job Parameter. |
| JT-20 | `4--parameter-name-in-two-environments.yaml` | EP | Two Environments may each declare a parameter of the same name. |
| JT-21 | `4--parameter-definitions-on-step-environment.invalid.yaml` | EP | `envParameterDefinitions` on a `stepEnvironments` entry is rejected. |
| JT-22 | `4.5--expr-types.yaml` | EP, EX | BOOL and every LIST type available to Job Parameters, including LIST[PATH], are valid with `EXPR`. |
| JT-23 | `4.5--range-expr-type.invalid.yaml` | EP, EX | Type RANGE_EXPR is rejected. |
| JT-24 | `4.5--user-interface.invalid.yaml` | EP | `userInterface` on an Environment parameter is rejected. |
| JT-25 | `4.5--path-object-type.invalid.yaml` | EP | `objectType` on a PATH Environment parameter is rejected. |
| JT-26 | `4.5--path-data-flow.invalid.yaml` | EP | `dataFlow` on a PATH Environment parameter is rejected. |
| JT-27 | `4.5--default-reads-job-parameter.yaml` | EP | `default: "{{Param.X}}"` for a STRING Job Parameter of the same template is valid. |
| JT-28 | `4.5--default-reads-path-param.invalid.yaml` | EP | `default: "{{Param.P}}"` for a PATH Job Parameter is rejected. |
| JT-29 | `4.5--default-reads-raw-path-param.yaml` | EP | `default: "{{RawParam.P}}"` for a PATH Job Parameter is valid. |
| JT-30 | `4.5--default-reads-undefined-param.invalid.yaml` | EP | A `default` reading an undefined Job Parameter is rejected. |
| JT-31 | `4.5--default-reads-job-name.invalid.yaml` | EP, EX | A `default` reading `Job.Name` is rejected. |
| JT-32 | `4.5--default-reads-env-param.invalid.yaml` | EP | A `default` reading another parameter's `Env.Param.*` is rejected. |
| JT-33 | `4.5--default-reads-env-names.invalid.yaml` | EP, EX | A `default` reading `Job.EnvNames` is rejected. |
| JT-34 | `4.5--literal-default-below-min-value.invalid.yaml` | EP | An INT literal `default: 0` with `minValue: 1` is rejected even though every Step supplies the parameter. |
| JT-35 | `4.5--literal-default-not-in-allowed-values.invalid.yaml` | EP | A STRING literal `default` outside `allowedValues` is rejected even though every Step supplies it. |
| JT-36 | `4.5--literal-default-wrong-type.invalid.yaml` | EP | An INT literal `default: "abc"` is rejected. |
| JT-37 | `4.5--list-default-not-single-expression.invalid.yaml` | EP, EX | A LIST[STRING] `default` that is a format string other than one `{{ }}` expression is rejected. |
| JT-38 | `4.5--list-default-single-expression.yaml` | EP, EX | A LIST[STRING] `default` that is one `{{ }}` expression is valid. |
| JT-39 | `4.5--typed-expression-default.yaml` | EP, EX | An INT `default: "{{ Param.N + 1 }}"` is valid. |
| JT-120 | `4.5--list-path-object-type.invalid.yaml` | EP, EX | `objectType` on a LIST[PATH] Environment parameter is rejected. |
| JT-121 | `4.5--default-reads-list-path-param.invalid.yaml` | EP, EX | A `default: "{{ Param.Dirs }}"` reading a LIST[PATH] Job Parameter is rejected. |

#### `activation` (§4 item 4)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JT-40 | `4--activation-always.yaml` | EP | `activation: ALWAYS` on a Job Environment is valid. |
| JT-41 | `4--activation-on-request.yaml` | EP | `activation: ON_REQUEST` on a Job Environment is valid. |
| JT-42 | `4--activation-unknown.invalid.yaml` | EP | `activation: SOMETIMES` is rejected. |
| JT-43 | `4--activation-on-step-environment.invalid.yaml` | EP | `activation` on a `stepEnvironments` entry is rejected. |

#### Environment `hostRequirements` (§4 item 5)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JT-44 | `4--host-requirements-on-job-environment.yaml` | EP | Literal `hostRequirements` on a Job Environment are valid. |
| JT-45 | `4--host-requirements-read-env-param.yaml` | EP | An attribute `anyOf: ["{{Env.Param.Flavor}}"]` reading the Environment's STRING parameter is valid. |
| JT-46 | `4--host-requirements-read-job-name.yaml` | EP, EX | Environment `hostRequirements` may read `Job.Name`. |
| JT-47 | `4--host-requirements-read-env-names.yaml` | EP, EX | Environment `hostRequirements` may read `Job.EnvNames`. |
| JT-48 | `4--host-requirements-on-step-environment.invalid.yaml` | EP | `hostRequirements` on a `stepEnvironments` entry are rejected. |
| JT-49 | `4--host-requirements-env-path-param.invalid.yaml` | EP | Environment `hostRequirements` reading `Env.Param.<PATH>` are rejected. |
| JT-50 | `4--host-requirements-param-path.invalid.yaml` | EP | Environment `hostRequirements` reading `Param.<PATH>` are rejected. |
| JT-51 | `4--host-requirements-step-name.invalid.yaml` | EP, EX | Environment `hostRequirements` reading `Step.Name` are rejected. |
| JT-52 | `4--host-requirements-session.invalid.yaml` | EP | Environment `hostRequirements` reading `Session.WorkingDirectory` are rejected. |
| JT-53 | `4--host-requirements-other-env-param.invalid.yaml` | EP | Environment `hostRequirements` reading a parameter of a different Environment are rejected. |
| JT-54 | `4--host-requirements-duplicate-amount-name.invalid.yaml` | EP | Two `amounts` of one Environment's `hostRequirements` with the same literal name are rejected (§3.3 applies per source). |
| JT-117 | `4--host-requirements-empty.invalid.yaml` | EP | `hostRequirements: {}` on a Job Environment is rejected (§3.3 constraint 1, in a Job Template). |
| JT-118 | `4--host-requirements-env-raw-path-param.invalid.yaml` | EP | Environment `hostRequirements` reading `Env.RawParam.<PATH>` are rejected (item 5.1 excludes both forms). |
| JT-119 | `4--host-requirements-env-list-path-param.invalid.yaml` | EP, EX | Environment `hostRequirements` reading `Env.Param.<LIST[PATH]>` are rejected. |

#### `Env.Param.*` / `Env.RawParam.*` and `Job.EnvNames` scope (§4, §7.3.1)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JT-55 | `7.3.1--env-param-in-variables.yaml` | EP | `Env.Param.X` in the declaring Environment's `variables` is valid. |
| JT-56 | `7.3.1--env-param-in-let.yaml` | EP, EX | `Env.Param.X` in the declaring Environment's script `let` is valid. |
| JT-57 | `7.3.1--env-param-in-embedded-file.yaml` | EP | `Env.Param.X` in the declaring Environment's embedded file `data` is valid. |
| JT-58 | `7.3.1--env-raw-param-in-on-exit.yaml` | EP | `Env.RawParam.X` in the declaring Environment's `onExit` is valid. |
| JT-59 | `7.3.1--env-param-undeclared.invalid.yaml` | EP | `Env.Param.Y` where the Environment declares no `Y` is rejected. |
| JT-60 | `7.3.1--env-param-of-other-environment.invalid.yaml` | EP | Environment A reading Environment B's parameter is rejected. |
| JT-61 | `7.3.1--env-param-in-step-script.invalid.yaml` | EP | A Step's `onRun` reading `Env.Param.X` of an Environment it requests is rejected. |
| JT-62 | `7.3.1--env-param-in-step-environment.invalid.yaml` | EP | A Step Environment reading `Env.Param.X` is rejected. |
| JT-63 | `7.3.1--env-param-in-step-host-requirements.invalid.yaml` | EP | A Step's own `hostRequirements` reading `Env.Param.X` are rejected. |
| JT-64 | `7.3.1--env-param-int-type-error.invalid.yaml` | EP, EX | `{{ Env.Param.Seats.upper() }}` on an INT parameter is a type error. |
| JT-65 | `7.3.1--env-names-in-step-script.yaml` | EP, EX | `{{ Job.EnvNames.join(',') }}` in a Step's `onRun` is valid. |
| JT-66 | `7.3.1--env-names-in-step-let.yaml` | EP, EX | A Step `let` binding reading `Job.EnvNames` is valid. |
| JT-67 | `7.3.1--env-names-in-job-name.invalid.yaml` | EP, EX | `Job.EnvNames` in the Job Template's `name` is rejected. |
| JT-68 | `7.3.1--env-names-is-list.invalid.yaml` | EP, EX | `{{ Job.EnvNames.upper() }}` is a type error (`list[string]`). |
| JT-123 | `7.3.1--env-raw-param-in-step-script.invalid.yaml` | EP | A Step's `onRun` reading `Env.RawParam.X` of an Environment it requests is rejected. |

#### `<StepDependency>` `environment:` entries (§3.2)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JT-69 | `3.2--environment-and-depends-on.invalid.yaml` | EP | An entry with both `environment` and `dependsOn` is rejected (constraint 1). |
| JT-70 | `3.2--entry-without-key.invalid.yaml` | EP | An entry with only `parameters` is rejected (constraint 1). |
| JT-71 | `3.2--required-with-depends-on.invalid.yaml` | EP | `required` on a `dependsOn` entry is rejected (constraint 1). |
| JT-72 | `3.2--parameters-with-depends-on.invalid.yaml` | EP | `parameters` on a `dependsOn` entry is rejected (constraint 1). |
| JT-73 | `3.2--enable-if-with-depends-on.invalid.yaml` | EP, EX | `enableIf` on a `dependsOn` entry is rejected (constraint 1). |
| JT-74 | `3.2--depends-on-and-environment-entries.yaml` | EP | One `dependencies` list may hold `dependsOn` and `environment` entries. |
| JT-75 | `3.2--names-own-step-environment.invalid.yaml` | EP | An entry naming the Step's own Step Environment is rejected (constraint 2). |
| JT-76 | `3.2--duplicate-environment-entries.invalid.yaml` | EP | Two entries naming one Environment are rejected (constraint 3). |
| JT-77 | `3.2--duplicate-entries-complementary-enable-if.invalid.yaml` | EP, EX | Two entries naming one Environment with `enableIf` true and false are still rejected (constraint 3). |
| JT-78 | `3.2--parameters-empty.invalid.yaml` | EP | `parameters: {}` is rejected (constraint 4). |
| JT-79 | `3.2--parameters-51-keys-external.invalid.yaml` | EP | 51 `parameters` keys on an entry naming an external Environment are rejected (constraint 4). |
| JT-80 | `3.2--parameters-50-keys-external.yaml` | EP | 50 `parameters` keys on an entry naming an external Environment are valid at template validation. |
| JT-81 | `3.2--entry-without-parameters.yaml` | EP | An entry with no `parameters` is valid; every parameter takes its `default`. |
| JT-82 | `3.2--unknown-parameter-key.invalid.yaml` | EP | A key the named Job Environment does not declare is rejected (constraint 5). |
| JT-83 | `3.2--parameters-to-environment-without-definitions.invalid.yaml` | EP | Supplying a parameter to a Job Environment that declares none is rejected (constraint 5). |
| JT-84 | `3.2--no-parameters-to-environment-without-definitions.yaml` | EP | An entry without `parameters` naming a Job Environment that declares none is valid (constraint 5). |
| JT-85 | `3.2--literal-wrong-type.invalid.yaml` | EP | A STRING literal `"abc"` for an INT parameter is rejected (constraint 6). |
| JT-86 | `3.2--literal-not-in-allowed-values.invalid.yaml` | EP | A literal outside `allowedValues` is rejected (constraint 6). |
| JT-87 | `3.2--literal-below-min-value.invalid.yaml` | EP | An INT literal below `minValue` is rejected (constraint 6). |
| JT-88 | `3.2--literal-exceeds-max-length.invalid.yaml` | EP | A STRING literal longer than `maxLength` is rejected (constraint 6). |
| JT-89 | `3.2--int-literal-as-intstring.yaml` | EP | `"1"` for an INT parameter is valid (an `<intstring>`). |
| JT-90 | `3.2--list-literal-value.yaml` | EP, EX | A list literal for a LIST[STRING] Job Environment parameter is valid. |
| JT-91 | `3.2--list-value-not-single-expression.invalid.yaml` | EP, EX | A format string other than one `{{ }}` expression for a LIST[STRING] Job Environment parameter is rejected. |
| JT-92 | `3.2--required-false-on-job-environment.invalid.yaml` | EP | `required: false` on an entry naming a Job Environment is rejected (constraint 8). |
| JT-93 | `3.2--required-true-on-job-environment.yaml` | EP | `required: true` on an entry naming a Job Environment is valid. |
| JT-94 | `3.2--required-false-external.yaml` | EP | `required: false` on an entry naming no Job Environment is valid. |
| JT-95 | `3.2--enable-if-literal.yaml` | EP, EX | `enableIf: false` is valid. |
| JT-96 | `3.2--enable-if-expression.yaml` | EP, EX | `enableIf: "{{ 'Docker' not in Job.EnvNames }}"` is valid. |
| JT-97 | `3.2--enable-if-not-bool.invalid.yaml` | EP, EX | `enableIf: "{{ Param.Count }}"` for an INT Job Parameter is rejected (constraint 9); there is no implicit `int` → `bool` conversion (see [ambiguity 11](#rules-without-a-definite-test)). |
| JT-98 | `3.2--enable-if-not-single-expression.invalid.yaml` | EP, EX | `enableIf: "x{{ true }}"` is rejected (constraint 9). |

#### `<StepDependency>` `step:` entries (§3.2 item 1)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JT-124 | `3.2--step-entry.yaml` | EP | A `step:` entry naming an earlier Step is valid. |
| JT-125 | `3.2--step-entry-without-extension.invalid.yaml` | — | A `step:` entry is rejected without `ENV_PARAMS`. |
| JT-126 | `3.2--step-and-depends-on.invalid.yaml` | EP | An entry with both `step` and `dependsOn` is rejected, even naming one Step (constraint 1). |
| JT-127 | `3.2--step-and-environment.invalid.yaml` | EP | An entry with both `step` and `environment` is rejected (constraint 1). |
| JT-128 | `3.2--parameters-with-step.invalid.yaml` | EP | `parameters` on a `step:` entry is rejected (constraint 1). |
| JT-129 | `3.2--required-with-step.invalid.yaml` | EP | `required` on a `step:` entry is rejected (constraint 1). |
| JT-130 | `3.2--step-and-depends-on-duplicate.invalid.yaml` | EP | A `step:` and a `dependsOn:` entry naming one Step are a duplicate and rejected. |
| JT-131 | `3.2--step-unknown.invalid.yaml` | EP | A `step:` entry naming no Step is rejected. |
| JT-132 | `3.2--step-self.invalid.yaml` | EP | A Step naming itself with `step:` is rejected. |
| JT-133 | `3.2--step-and-depends-on-cycle.invalid.yaml` | EP | A cycle of one `step:` and one `dependsOn:` edge is rejected. |
| JT-134 | `3.2--step-depends-on-and-environment-entries.yaml` | EP | One `dependencies` list may mix `step:`, `dependsOn:`, and `environment:` entries. |

#### Constraint 7 for Job Environments (§3.2)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JT-99 | `3.2--always-required-param-not-supplied.invalid.yaml` | EP | A Step with no entry for an `ALWAYS` Job Environment with a parameter without `default` is rejected. |
| JT-100 | `3.2--always-required-param-supplied-by-every-step.yaml` | EP | The same Environment is valid when every Step supplies the parameter. |
| JT-101 | `3.2--on-request-required-param-non-requesting-step.yaml` | EP | A Step that does not request an `ON_REQUEST` Job Environment need not supply its required parameter. |
| JT-102 | `3.2--on-request-required-param-not-supplied.invalid.yaml` | EP | A Step that requests an `ON_REQUEST` Job Environment without supplying its required parameter is rejected. |
| JT-103 | `3.2--always-required-param-entry-disabled-literal.invalid.yaml` | EP, EX | An entry with `enableIf: false` for an `ALWAYS` Job Environment does not supply its required parameter, and is rejected at validation. |
| JT-104 | `3.2--on-request-entry-disabled-literal.yaml` | EP, EX | An entry with `enableIf: false` for an `ON_REQUEST` Job Environment requests nothing, so its required parameter is not needed. |

#### Value and `enableIf` scope (§3.2)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JT-105 | `3.2--value-reads-job-parameter.yaml` | EP | A value `"{{Param.X}}"` for a STRING Job Parameter is valid. |
| JT-106 | `3.2--value-reads-path-param.invalid.yaml` | EP | A value `"{{Param.P}}"` for a PATH Job Parameter is rejected. |
| JT-107 | `3.2--value-reads-raw-path-param.yaml` | EP | A value `"{{RawParam.P}}"` for a PATH Job Parameter is valid. |
| JT-108 | `3.2--value-reads-task-param.invalid.yaml` | EP | A value reading `Task.Param.*` is rejected. |
| JT-109 | `3.2--value-reads-session.invalid.yaml` | EP | A value reading `Session.WorkingDirectory` is rejected. |
| JT-110 | `3.2--value-reads-env-scope.invalid.yaml` | EP | A value reading `Env.File.Spec`, an embedded file the requested Environment defines, is rejected. |
| JT-111 | `3.2--value-reads-step-name.yaml` | EP, EX | A value reading `Step.Name` is valid. |
| JT-112 | `3.2--value-reads-step-let.yaml` | EP, EX | A value reading a Step `let` binding is valid. |
| JT-113 | `3.2--value-reads-job-name.yaml` | EP, EX | A value reading `Job.Name` is valid. |
| JT-114 | `3.2--value-reads-env-names.yaml` | EP, EX | A value reading `Job.EnvNames` is valid. |
| JT-115 | `3.2--enable-if-reads-task-param.invalid.yaml` | EP, EX | An `enableIf` reading `Task.Param.*` is rejected. |
| JT-116 | `3.2--enable-if-reads-step-name-and-let.yaml` | EP, EX | An `enableIf` reading `Step.Name` and a Step `let` binding is valid. |
| JT-122 | `3.2--enable-if-reads-path-param.invalid.yaml` | EP, EX | An `enableIf` reading `Param.<PATH>` is rejected. |

### `env_templates/` — Environment Template validation

| ID | File | Exts | Assertion |
|---|---|---|---|
| ET-01 | `1.2--extensions-env-params.yaml` | EP | An Environment Template with `extensions: [ENV_PARAMS]` and Environment parameters is valid. |
| ET-02 | `1.2--extensions-empty.invalid.yaml` | — | `extensions: []` is rejected (§1.2 item 3, non-empty). |
| ET-03 | `4--parameter-definitions-without-extension.invalid.yaml` | — | `envParameterDefinitions` on the `environment` is rejected without `ENV_PARAMS`. |
| ET-04 | `4--activation-without-extension.invalid.yaml` | — | `activation` is rejected without `ENV_PARAMS`. |
| ET-05 | `4--host-requirements-without-extension.invalid.yaml` | — | `hostRequirements` is rejected without `ENV_PARAMS`. |
| ET-06 | `4.5--bool-type-without-expr.invalid.yaml` | EP | A BOOL Environment parameter is rejected without `EXPR`. |
| ET-07 | `4--parameter-definitions-empty.invalid.yaml` | EP | An empty `envParameterDefinitions` list is rejected. |
| ET-08 | `4--parameter-definitions-51.invalid.yaml` | EP | 51 parameters are rejected. |
| ET-09 | `4--parameter-definitions-duplicate-name.invalid.yaml` | EP | Duplicate parameter names are rejected. |
| ET-10 | `4--parameter-name-same-as-job-parameter.yaml` | EP | A parameter may share its name with the template's own Job Parameter. |
| ET-11 | `4.5--range-expr-type.invalid.yaml` | EP, EX | Type RANGE_EXPR is rejected. |
| ET-12 | `4.5--user-interface.invalid.yaml` | EP | `userInterface` is rejected. |
| ET-13 | `4--activation-on-request.yaml` | EP | `activation: ON_REQUEST` is valid. |
| ET-14 | `4--activation-unknown.invalid.yaml` | EP | An unknown `activation` value is rejected. |
| ET-15 | `4--host-requirements-read-env-param.yaml` | EP | `hostRequirements` reading the Environment's STRING parameter are valid. |
| ET-16 | `4--host-requirements-env-path-param.invalid.yaml` | EP | `hostRequirements` reading `Env.Param.<PATH>` are rejected. |
| ET-17 | `4--host-requirements-step-name.invalid.yaml` | EP, EX | `hostRequirements` reading `Step.Name` are rejected. |
| ET-18 | `4.5--default-reads-own-job-parameter.yaml` | EP | A `default` reading a Job Parameter the Environment Template defines is valid. |
| ET-19 | `4.5--default-reads-undefined-param.invalid.yaml` | EP | A `default` reading a Job Parameter the Environment Template does not define is rejected, even if a Job Template might define it ("of the same document"). |
| ET-20 | `4.5--default-reads-path-param.invalid.yaml` | EP | A `default` reading `Param.<PATH>` is rejected. |
| ET-21 | `4.5--default-reads-raw-path-param.yaml` | EP | A `default` reading `RawParam.<PATH>` is valid. |
| ET-22 | `4.5--default-reads-env-names.invalid.yaml` | EP, EX | A `default` reading `Job.EnvNames` is rejected. |
| ET-23 | `4.5--literal-default-violates-constraint.invalid.yaml` | EP | A literal `default` outside `allowedValues` is rejected. |
| ET-24 | `4.5--list-default-not-single-expression.invalid.yaml` | EP, EX | A list `default` that is a format string other than one `{{ }}` expression is rejected. |
| ET-25 | `4.5--list-path-default-empty.yaml` | EP, EX | `type: LIST[PATH]` with `default: []` is valid, as in the Docker sample. |
| ET-26 | `4.5--required-parameter-activation-always.yaml` | EP | A parameter without `default` on an `ALWAYS` Environment is valid in the template alone; constraint 7 is a job-creation check (JB-32). |
| ET-27 | `7.3.1--env-param-undeclared.invalid.yaml` | EP | `Env.Param.Y` for an undeclared `Y` is rejected. |
| ET-28 | `7.3.1--env-param-in-variables-let-and-files.yaml` | EP, EX | `Env.Param.X` in `variables`, script `let`, and embedded files is valid. |
| ET-29 | `7.3.1--env-names-in-on-enter.yaml` | EP, EX | `Job.EnvNames` in `onEnter` args is valid. |
| ET-30 | `7.3.1--env-names-without-expr.invalid.yaml` | EP | `Job.EnvNames` is rejected without `EXPR`. |
| ET-31 | `7.3.1--wrap-hook-reads-own-param.yaml` | EP, WA, EX | A wrapping Environment's `onWrapTaskRun` reading its own `Env.Param.Image` is valid. |
| ET-32 | `1.2--schema-field.yaml` | EP | An Environment Template with a `$schema` property is valid (§1.2 item 2). |
| ET-33 | `1.2--unknown-extension.invalid.yaml` | — | An extension name the specification does not define is rejected (§1.2 item 3, same names as the Job Template). |
| ET-34 | `4.5--path-object-type.invalid.yaml` | EP | `objectType` on a PATH Environment parameter is rejected. |
| ET-35 | `4.5--path-data-flow.invalid.yaml` | EP | `dataFlow` on a PATH Environment parameter is rejected. |
| ET-36 | `4.5--list-type-without-expr.invalid.yaml` | EP | A LIST[PATH] Environment parameter is rejected without `EXPR`. |
| ET-37 | `4--parameter-definitions-50.yaml` | EP | 50 parameters are valid. |
| ET-38 | `4--host-requirements-session.invalid.yaml` | EP | `hostRequirements` reading `Session.WorkingDirectory` are rejected. |

### `jobs/` — job creation and execution

Attached Environment Templates are given in the test's `environments` list.

#### Values reach the Environment (§4.5, §3.2.2, §1.2.1)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JB-01 | `4.5--value-reaches-on-enter-args.test.yaml` | EP | A supplied value appears in `onEnter` argv (`echo`). |
| JB-02 | `4.5--value-reaches-on-exit-args.test.yaml` | EP | The same value appears in `onExit` argv. |
| JB-03 | `4.5--value-argv-exact.test.yaml` | EP | `runOn: [posix]`. A value with spaces, quotes, and `$` arrives as one argv element, byte for byte (`printf '[%s]\n' "$@"`). |
| JB-04 | `4.5--value-reaches-task-through-variables.test.yaml` | EP | `variables: {PKG: "{{Env.Param.Packages}}"}`; the Task prints `PKG` with the supplied value. |
| JB-05 | `4.5--omitted-parameter-takes-default.test.yaml` | EP | An entry that omits a parameter yields its literal `default`. |
| JB-06 | `4.5--default-reads-merged-job-parameter.test.yaml` | EP | Attached `default: "{{Param.CondaPackages}}"`; the Job Template redefines `CondaPackages`'s `default`; `onEnter` sees the Job Template's default. |
| JB-07 | `4.5--default-reads-submitted-job-parameter.test.yaml` | EP | As JB-06 with `parameters: {CondaPackages: ...}`; `onEnter` sees the submitted value. |
| JB-08 | `1.2.1--env-parameters-not-merged.test.yaml` | EP | A Job Parameter `Packages` and two Environments' `Packages` parameters keep three distinct values. |
| JB-09 | `3.2.2--format-string-value-converted-to-int.test.yaml` | EP | `"{{Param.N}}"` for an INT parameter with `minValue: 1`, `N=3`, prints `3`. |
| JB-10 | `3.2.2--int-one-and-string-one-both-accepted.test.yaml` | EP | One Step supplies INT `1`, another `"1"`; both run and both print `1` (a second parameter `Tag` tells the two lines apart). |
| JB-11 | `4.5--resolved-default-unused-violates-constraint.test.yaml` | EP | A `default` reading a Job Parameter that resolves outside `allowedValues` is not an error when every Step supplies the parameter. |
| JB-12 | `4.5--resolved-default-taken-violates-constraint.invalid.test.yaml` | EP | The same `default` is rejected when one Step takes it. |
| JB-13 | `3.2--format-string-value-violates-constraint.invalid.test.yaml` | EP | A Job Environment value `"{{Param.N}}"` that resolves below `minValue` is rejected at job creation. |
| JB-14 | `3.2--value-from-step-let-and-step-name.test.yaml` | EP, EX | Two Steps supply `"{{ Step.Name + '-' + tag }}"` from a Step `let`; each `onEnter` prints its own Step's value. |

#### External resolution (§3.2.1)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JB-15 | `3.2.1--external-environment-receives-values.test.yaml` | EP | An entry naming an attached Environment supplies its value to that Environment's `onEnter`. |
| JB-16 | `3.2.1--external-required-missing.invalid.test.yaml` | EP | An entry (default `required: true`) naming an Environment nothing attaches is rejected. |
| JB-17 | `3.2.1--external-required-false-missing-ignored.test.yaml` | EP, EX | The same entry with `required: false` is ignored; the Step's Task runs, and `Job.EnvNames` lists only the Job Environment `J`. |
| JB-18 | `3.2.1--external-lacks-parameter.invalid.test.yaml` | EP | A `required: false` entry supplying a parameter the attached Environment does not declare is rejected. |
| JB-19 | `3.2.1--external-without-env-params-supplied.invalid.test.yaml` | EP | Attached Environment Template without `ENV_PARAMS` (the pre-extension `Conda` case); a `required: false` entry with `parameters` is rejected. |
| JB-20 | `3.2.1--external-without-env-params-no-parameters.test.yaml` | EP | Attached Environment Template without `ENV_PARAMS`; an entry without `parameters` is valid, and the Environment is entered. |
| JB-21 | `3.2.1--external-ambiguous-name.invalid.test.yaml` | EP | Two attached Environments with the entry's name are rejected. |
| JB-22 | `3.2.1--job-environment-shadows-attached.test.yaml` | EP | A Job Environment and an attached Environment share name `X`; the entry's value reaches the Job Environment, and the attached one (which declares no parameter) does not cause a rejection. |
| JB-23 | `3.2.1--attached-reachable-from-other-step.test.yaml` | EP | Step S has a Step Environment `X`; Step T's entry for `X` resolves to the attached `X` and its value reaches it. |
| JB-24 | `3.2.1--external-value-wrong-type.invalid.test.yaml` | EP | A STRING `"abc"` for an attached INT parameter is rejected. |
| JB-25 | `3.2.1--external-value-not-allowed.invalid.test.yaml` | EP | A value outside the attached parameter's `allowedValues` is rejected. |
| JB-26 | `3.2.1--external-list-literal-for-string.invalid.test.yaml` | EP, EX | A list literal for an attached STRING parameter is rejected. |
| JB-27 | `3.2.1--non-expr-document-supplies-list-parameter.invalid.test.yaml` | EP | A Job Template without `EXPR` supplies a string to an attached LIST[PATH] parameter (the Environment Template lists `EXPR`); rejected. `runOn: [posix]`, so the value is absolute. The value also breaks the single-`{{ }}` rule for list values, which a document without `EXPR` cannot avoid. |
| JB-28 | `3.2.1--non-expr-document-supplies-bool-parameter.invalid.test.yaml` | EP | A Job Template without `EXPR` supplies `"true"` to an attached BOOL parameter; rejected. |
| JB-29 | `3.2.1--external-list-value-not-single-expression.invalid.test.yaml` | EP, EX | A format string other than one `{{ }}` expression for an attached LIST[STRING] parameter is rejected. |
| JB-30 | `3.2.1--several-attached-each-own-values.test.yaml` | EP | One Step requests two attached Environments; each `onEnter` prints the value of its own entry. |
| JB-87 | `3.2.1--external-required-false-present-receives-values.test.yaml` | EP | A `required: false` entry that resolves to an attached `ON_REQUEST` Environment enters it with the entry's value. |

#### Constraint 7 over the combined Job (§3.2, §3.2.1)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JB-31 | `3.2.1--attached-always-required-supplied-by-every-step.test.yaml` | EP | An attached `ALWAYS` Environment with a parameter without `default`, supplied by every Step, runs. |
| JB-32 | `3.2.1--attached-always-required-not-supplied.invalid.test.yaml` | EP | The same Environment with one Step not supplying it is rejected. |
| JB-33 | `3.2.1--attached-on-request-required-not-requested.test.yaml` | EP | An attached `ON_REQUEST` Environment with a required parameter, requested by no Step, runs. |
| JB-34 | `3.2.1--attached-on-request-required-not-supplied.invalid.test.yaml` | EP | The same Environment requested without the parameter is rejected. |
| JB-35 | `3.2--required-param-entry-disabled-by-expression.invalid.test.yaml` | EP, EX | An `ALWAYS` Job Environment's only supplying entry has `enableIf: "{{ 'Nope' in Job.EnvNames }}"`, which resolves `false`; constraint 7 fails at job creation. |
| JB-92 | `3.2.1--attached-always-required-entry-disabled.invalid.test.yaml` | EP, EX | The same for an attached `ALWAYS` Environment: the disabled entry supplies nothing, and constraint 7 fails at job creation. |

#### `activation` at run time (§4 item 4)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JB-36 | `4--on-request-entered-by-requesting-step.test.yaml` | EP | The requesting Step's Session prints the `ON_REQUEST` Environment's `onEnter` sentinel with its value. |
| JB-37 | `4--on-request-not-entered-by-other-step.test.yaml` | EP | A second Step that does not request it never prints the sentinel with the `default` value (`forbidden`). |
| JB-38 | `4--on-request-variables-not-set-for-other-step.test.yaml` | EP | The Environment's `variables` are not set in the non-requesting Step's Task. |
| JB-39 | `4--always-entered-by-step-without-entry.test.yaml` | EP | A Step with no entry still enters an `ALWAYS` Environment, with its `default`. |
| JB-40 | `4--attached-on-request-not-entered-without-request.test.yaml` | EP | An attached `ON_REQUEST` Environment that no Step requests is never entered. |
| JB-41 | `4--on-request-keeps-order.test.yaml` | EP | `runOn: [posix]`. `ALWAYS` J1, `ON_REQUEST` J2, `ALWAYS` J3; a requesting Step's Task prints `J1J2J3`, built by each `onEnter` appending to a variable. |

#### `enableIf` on the entry (§3.2 item 5)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JB-42 | `3.2--enable-if-true-enters.test.yaml` | EP, EX | An entry whose `enableIf` resolves `true` causes its `ON_REQUEST` Environment to be entered with the value. |
| JB-43 | `3.2--enable-if-false-not-entered.test.yaml` | EP, EX | An entry whose `enableIf` resolves `false` leaves its `ON_REQUEST` Environment unentered. |
| JB-44 | `3.2--enable-if-false-always-takes-defaults.test.yaml` | EP, EX | A disabled entry for an `ALWAYS` Environment supplies nothing; `onEnter` prints the `default`, not the entry's value. |
| JB-45 | `3.2--enable-if-false-skips-missing-required.test.yaml` | EP, EX | A disabled `required: true` entry naming an Environment nothing attaches is not an error; the Task runs. |
| JB-46 | `3.2--enable-if-null.invalid.test.yaml` | EP, EX | An `enableIf` that resolves to `null` at job creation is rejected. |

#### `Job.EnvNames` (§7.3.1)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JB-47 | `7.3.1--env-names-job-environments-in-order.test.yaml` | EP, EX | No attached Environments; Job Environments `B`, `A`; the Task prints `B,A`. |
| JB-48 | `7.3.1--env-names-excludes-step-environments.test.yaml` | EP, EX | A Step Environment's name does not appear. |
| JB-49 | `7.3.1--env-names-includes-attached.test.yaml` | EP, EX | Attached `Q` and Job Environments `J1`, `J2`; the line is `anyOf` `Q,J1,J2` or `J1,J2,Q` (see [ambiguity 1](#rules-without-a-definite-test)). |
| JB-50 | `7.3.1--env-names-includes-unrequested-on-request.test.yaml` | EP, EX | An `ON_REQUEST` Environment the Step does not request is listed. |
| JB-51 | `7.3.1--env-names-repeats-shared-attached-name.test.yaml` | EP, EX | Two attached Environments named `X` and no entry naming `X`; the line is `X,X`. |
| JB-52 | `7.3.1--env-names-in-environment-script.test.yaml` | EP, EX | An attached Environment's `onEnter` prints `Job.EnvNames`, including the Job Template's Environments. |
| JB-53 | `7.3.1--env-names-in-step-let-feeds-value.test.yaml` | EP, EX | A Step `let` reading `Job.EnvNames` feeds a `parameters` value that `onEnter` prints. |
| JB-86 | `7.3.1--env-names-expands-to-argv.test.yaml` | EP, EX | `runOn: [posix]`. An `args` item `"{{Job.EnvNames}}"` in a Step script expands to one argv element per name (Expression Language §1.3.2 flattens a list result in an `args` item). |

#### Path values (§4.5)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JB-54 | `4.5--path-value-mapped-in-env-param-posix.test.yaml` | EP | `runOn: [posix]`. A PATH value from `"{{RawParam.Dir}}"` is path-mapped in `Env.Param.Dir`. |
| JB-55 | `4.5--path-value-mapped-in-env-param-windows.test.yaml` | EP | `runOn: [windows]`. As JB-54 with Windows paths. |
| JB-56 | `4.5--path-value-unmapped-in-env-raw-param-posix.test.yaml` | EP | `runOn: [posix]`. A PATH parameter whose `default` is `"{{RawParam.Dir}}"`, taken by a Step with no entry, is unmapped in `Env.RawParam.Dir`. |
| JB-57 | `4.5--path-value-unmapped-in-env-raw-param-windows.test.yaml` | EP | `runOn: [windows]`. As JB-56 with Windows paths. |
| JB-58 | `4.5--list-path-elements-mapped-posix.test.yaml` | EP, EX | `runOn: [posix]`. Each element of a LIST[PATH] value is mapped in `Env.Param.*`. |
| JB-59 | `4.5--list-path-elements-mapped-windows.test.yaml` | EP, EX | `runOn: [windows]`. As JB-58 with Windows paths. |
| JB-60 | `4.5--relative-path-value.invalid.test.yaml` | EP | A supplied PATH value `relative/dir` for an attached Environment's parameter is rejected at job creation. The Environment is attached so that template validation cannot know the parameter is a PATH. |
| JB-61 | `4.5--relative-literal-path-default-unused.invalid.test.yaml` | EP | A literal PATH `default: relative/dir` is rejected even though every Step supplies an absolute value (from a PATH Job Parameter whose relative `default` is joined with the template directory, so it is absolute on every host). The Environment Template may already fail `openjd check` (ambiguity 2); the run is rejected either way. |
| JB-62 | `4.5--path-allowed-values-checked-unmapped-posix.test.yaml` | EP | `runOn: [posix]`. `allowedValues` lists the unmapped path; a mapping rule changes it on the host; the Job runs and `Env.Param` prints the mapped path. |
| JB-63 | `4.5--path-uri-value.test.yaml` | EP, EX | A PATH value `s3://bucket/key` is accepted, and `Env.RawParam` prints it unchanged. Both documents list `EXPR` (see ambiguity 10). |

#### Host requirements (§3.3.3, §4 item 5)

Capability names use a custom prefix (`conformance:attr.flavor`, `conformance:amount.widgets`) so that no runner
matches them against the local host.

| ID | File | Exts | Assertion |
|---|---|---|---|
| JB-64 | `3.3.3--non-conflicting-requirements-run.test.yaml` | EP | Step `anyOf: [a, b]`, Environment `anyOf: ["{{Env.Param.Flavor}}"]` with `Flavor: a`; the Job runs. |
| JB-65 | `3.3.3--amount-min-exceeds-max.invalid.test.yaml` | EP | Step amount `max: 1`, Environment amount `min: 2`; rejected. |
| JB-66 | `3.3.3--attribute-any-of-empty-intersection.invalid.test.yaml` | EP | Step `anyOf: [a]`, Environment `anyOf: [b]`; rejected. |
| JB-67 | `3.3.3--conflict-resolved-from-env-param.invalid.test.yaml` | EP | As JB-64 with `Flavor: c`; the conflict appears only after resolution with the Step's value; rejected. |
| JB-68 | `3.3.3--conflict-between-two-environments.invalid.test.yaml` | EP | Two Environments the Step enters require disjoint `anyOf`; rejected. |
| JB-69 | `3.3.3--attached-environment-conflict.invalid.test.yaml` | EP | An attached Environment's requirements conflict with the Step's; rejected. |
| JB-70 | `3.3.3--same-capability-in-step-and-environment.test.yaml` | EP | Step and Environment each require amount `min: 1` of the same capability; not a duplicate-name error; the Job runs. |
| JB-71 | `3.3.3--any-of-and-all-of-not-combined.test.yaml` | EP | Step `anyOf` and Environment `allOf` on one attribute; not an error; the Job runs. |
| JB-72 | `3.3.3--on-request-requirements-not-applied.test.yaml` | EP | An `ON_REQUEST` Environment's requirements conflict with a non-requesting Step's own; the Job runs. |
| JB-73 | `4--resolved-host-requirements-duplicate-name.invalid.test.yaml` | EP | Two attribute names `"{{Env.Param.First}}"` and `"{{Env.Param.Second}}"` resolve to the same name for a Step; rejected (§3.3 constraint 4, within one source, per Step). |
| JB-88 | `3.3.3--any-of-intersection-ignores-case.test.yaml` | EP | Step `anyOf: [A]` and Environment `anyOf: [a]` intersect, since values compare ignoring case; the Job runs. |
| JB-89 | `3.3.3--capability-name-compared-ignoring-case.invalid.test.yaml` | EP | Step `conformance:attr.Flavor` `anyOf: [a]` and Environment `conformance:attr.flavor` `anyOf: [b]` are one capability; rejected. |
| JB-90 | `3.3.3--disabled-entry-requirements-not-applied.test.yaml` | EP, EX | A disabled entry for an `ON_REQUEST` Environment whose requirements conflict with the Step's; no conflict, the Job runs, the Environment is not entered. |
| JB-91 | `4--resolved-host-requirements-negative-min.invalid.test.yaml` | EP; env EP, FB1 | An attached Environment's amount `min: "{{Env.Param.N}}"` resolves to `-1` for the Step; rejected (§3.3.1). |

#### Extensions per document and backward compatibility (§1.2, RFC §Backward compatibility)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JB-74 | `1.2--job-template-without-extension-takes-defaults.test.yaml` | — | A Job Template without `ENV_PARAMS` runs with an attached `ALWAYS` Environment that lists it; `onEnter` prints the `default`. |
| JB-75 | `1.2--job-template-without-extension-skips-on-request.test.yaml` | — | The same with `ON_REQUEST`; the Environment is never entered. |

#### With `WRAP_ACTIONS` (§4, §7.3.1; RFC Composition with SERVICE item 9 for the non-SERVICE case)

| ID | File | Exts | Assertion |
|---|---|---|---|
| JB-76 | `7.3.1--wrap-hook-reads-wrapper-env-param.test.yaml` | EP, WA, EX | A wrapping Environment's `onWrapTaskRun` prints its own `Env.Param.Image` supplied by the Step. |
| JB-77 | `4--on-request-wrapper-not-applied-to-other-step.test.yaml` | EP, WA, EX | An `ON_REQUEST` wrapping Environment wraps the requesting Step's Task and not the other Step's (the wrap sentinel is absent from the other Step's output). |

#### Basic Examples (RFC §Basic Examples)

Each example becomes one or two runnable tests that replace `conda`, `docker`, and `license-checkout` with `echo`
sentinels.

| ID | File | Exts | Assertion |
|---|---|---|---|
| JB-78 | `4.5--example-queue-conda-step-configures.test.yaml` | EP | *A queue Conda Environment that each Step configures*: a Step with no entry gets `CondaPackages`; a Step with an entry gets its own `Packages`. |
| JB-79 | `3.2.1--example-steps-own-software-conda-queue.test.yaml` | EP | *A Job Template whose Steps each ask for their own software*, with only `Conda` attached: each Step's `onEnter` prints its own `Packages`; the `Docker` entries are ignored. |
| JB-80 | `3.2.1--example-steps-own-software-no-queue-environment.test.yaml` | EP | The same Job Template with nothing attached runs unchanged. |
| JB-81 | `4--example-parameterized-job-environment.test.yaml` | EP | *One parameterized Job Environment serving several Steps*: `BeautyPass` prints `arnold 1`, `VolumePass` prints `redshift 2`, `Composite` never enters `RenderLicense`. |
| JB-82 | `3.2--example-choose-docker-when-both.test.yaml` | EP, EX | *Choosing between Environments*: with `Conda` and `Docker` attached (both `ON_REQUEST`), the `Conda` entry is disabled and `Conda` is not entered. |
| JB-83 | `3.2--example-choose-conda-when-alone.test.yaml` | EP, EX | The same Job Template with only `Conda` attached enters it with `colmap=3.11`. |
| JB-84 | `3.3.3--example-environment-host-requirements.test.yaml` | EP; env EP, EX, FB1 | *An Environment that states what hosts it needs*: one Step requests `Docker` with `Gpus: all` and one without; both run (requirements resolve per Step without conflict). |
| JB-85 | `3.2.2--example-execution-per-step-values.test.yaml` | EP | *Execution*: `DecodeFrames` and `SolveSfM` each enter `Conda` once with their own `Packages`, and `onExit` follows each. |

## Validation table coverage

One row per row of the RFC's [Validation](../../../rfcs/0010-environment-parameters.md#validation) table, in table
order.

| # | Validation rule (abridged) | Checked at | Tests |
|---|---|---|---|
| V1 | `ENV_PARAMS` listed by any document using Environment `envParameterDefinitions`, `activation`, `hostRequirements`, or `environment:` or `step:` entries | Template validation | JT-01–JT-04, JT-125, ET-03–ET-05; per-document: ET-01, JB-74, JB-75, JB-19, JB-20; Environment Template `extensions` and `$schema` keys: ET-01, ET-02, ET-32, ET-33 |
| V2 | Environment `envParameterDefinitions`: 1–50, unique names, not on `stepEnvironments`, permitted types and fields | Template validation | JT-11, JT-12, JT-14–JT-26, JT-120, ET-06–ET-12, ET-34–ET-37 |
| V3 | `activation` is `ALWAYS` or `ON_REQUEST`, not on `stepEnvironments` | Template validation | JT-40–JT-43, ET-13, ET-14; run time: JB-36–JB-41 |
| V4 | Environment `hostRequirements`: §3.3 shape, not on `stepEnvironments`, `Env.*` only of the enclosing Environment's non-PATH parameters | Template validation | JT-44, JT-45, JT-48, JT-49, JT-53, JT-54, JT-117–JT-119, ET-15, ET-16 |
| V5 | An Environment's resolved `hostRequirements` satisfy §3.3.1 and §3.3.2 per Step | Job creation | JB-73, JB-91 |
| V6 | Combined host requirements have no empty intersection (§3.3.3 items 1, 2) | Job creation | JB-65–JB-69, JB-89; non-conflicts: JB-64, JB-70–JB-72, JB-84, JB-88, JB-90 |
| V7 | With `CONTROL_FLOW`: no `Steps.*` in entries or Environment `hostRequirements` | Template validation | Out of scope (`CONTROL_FLOW` is not on mainline) |
| V8 | `default` references only `Param.*` / `RawParam.*` of the same document's Job Parameters | Template validation | JT-27–JT-33, JT-121, ET-18–ET-22; merged value: JB-06, JB-07 |
| V9 | A literal `default` satisfies its type and constraints, whether or not any Step uses it | Template validation | JT-34–JT-36, ET-23 |
| V10 | `Env.Param.*` / `Env.RawParam.*` name a parameter of the enclosing Environment, and appear only inside it | Template validation | JT-55–JT-64, JT-123, ET-27, ET-28, ET-31; run time: JB-01–JB-04, JB-76 |
| V11 | A `parameters` value or `enableIf` references only `RawParam.*`, non-PATH `Param.*`, and with `EXPR` `Job.Name`, `Job.EnvNames`, `Step.Name`, Step `let`; never `Task.*`, `Session.*`, `Env.*` | Template validation | JT-08, JT-105–JT-116, JT-122; run time: JB-14, JB-53 |
| V12 | An Environment's `hostRequirements` reference only its document's `RawParam.*`, non-PATH `Param.*`, with `EXPR` `Job.Name`, `Job.EnvNames`, and its own non-PATH `Env.*` | Template validation | JT-45–JT-47, JT-49–JT-53, JT-118, JT-119, ET-15–ET-17, ET-38 |
| V13 | `Job.EnvNames` only with `EXPR`, inside `{{ }}`, not in an Environment parameter `default` | Template validation | JT-09, JT-10, JT-33, JT-65–JT-68, ET-22, ET-29, ET-30; contents: JB-47–JB-53, JB-86 (see ambiguity 7 for "inside `{{ }}`") |
| V14 | `<StepDependency>` has exactly one of `dependsOn` / `step` / `environment`; `required`, `enableIf`, `parameters` only with `environment` (constraint 1) | Template validation | JT-69–JT-74, JT-126–JT-129, JT-134 |
| V15 | `enableIf` only with `EXPR`; a `bool` literal or single `bool` expression (constraint 9) | Template validation | JT-05, JT-95–JT-98 |
| V16 | `enableIf` does not resolve to `null` | Job creation | JB-46 |
| V17 | Not the Step's own Step Environment; no duplicates; 1–50 `parameters` (constraints 2–4) | Template validation | JT-75–JT-80 |
| V18 | `required: false` not on an entry naming a Job Environment (constraint 8) | Template validation | JT-92–JT-94 |
| V19 | Keys and literal values of `parameters` for a Job Environment (constraints 5, 6) | Template validation | JT-81–JT-91; resolved format strings at job creation: JB-13 |
| V20 | Required parameters of `ALWAYS` Job Environments supplied by every Step, of `ON_REQUEST` ones by every requester (constraint 7) | Template validation for literal or absent `enableIf`; job creation otherwise | JT-99–JT-104, JB-35 |
| V21 | External resolution: missing with `required: true`, or ambiguous (§3.2.1) | Job creation | JB-16, JB-21; accepted forms: JB-15, JB-17, JB-22, JB-23, JB-30, JB-45, JB-87 |
| V22 | Keys and values for an external Environment; a key the resolved Environment lacks is an error even with `required: false` | Job creation | JB-18, JB-19, JB-24–JB-26; accepted: JB-20 |
| V23 | A document without `EXPR` supplies no BOOL or LIST parameter (constraint 6) | Job creation | JB-27, JB-28; literal forms at validation: JT-06, JT-07 |
| V24 | A format-string value or `default` for a list-typed parameter is one `{{ }}` expression | Template validation for `default`s and Job Environment entries; job creation for external | JT-37, JT-38, JT-91, ET-24, JB-29 |
| V25 | A resolved, non-literal `default` satisfies its type and constraints for each Step that takes it | Job creation | JB-11, JB-12 |
| V26 | PATH / LIST[PATH] values absolute in the submission path format or URIs; string constraints on the unmapped value; including an unused literal `default` | Job creation | JB-60–JB-63; mapping: JB-54–JB-59 |
| V27 | Required parameters of attached `ALWAYS` Environments supplied by every Step, of attached `ON_REQUEST` ones by every requester (constraint 7, §3.2.1) | Job creation | JB-31–JB-34, JB-92 |
| V28 | With `SERVICE`: the dependency graph with Environment edges is acyclic | Template validation / job creation | Out of scope (`SERVICE` is not on mainline) |
| V29 | With `SERVICE` and `WRAP_ACTIONS`: every action inside a wrapping Environment has a wrap hook, per kind of Session | Template validation / job creation | Out of scope for the Service-Session case. The Task-Session case is RFC 0008's existing rule, covered by the `WRAP_ACTIONS` suite; its interaction with `activation` is JB-77 |
| V30 | With `CONTROL_FLOW`: an Environment's `enableIf` reads only its own `Env.*` | Template validation | Out of scope (`CONTROL_FLOW` is not on mainline) |
| V31 | A `step:` entry obeys every rule of the `dependsOn:` entry it is a synonym of; a `step:` and a `dependsOn:` entry naming one Step are duplicates (§3.2 item 1). Listed after V30 so that existing row numbers are stable; in the RFC table it follows V14 | Template validation | JT-124, JT-130–JT-133 |

The RFC's closing note, that a Session runs Tasks of a Step only when its Environments and values match (§3.2.2), is a
scheduling rule with no Validation row. Its observable parts are JB-10 and JB-85.

Runtime behaviors in the suite's scope that are not Validation rows are covered as follows: values in `onEnter`,
`onExit`, and `variables` (JB-01–JB-04); defaults from merged Job Parameters (JB-05–JB-08); `ON_REQUEST` entry
(JB-36–JB-41); entry `enableIf` true, false, and `null` (JB-42–JB-46); `Job.EnvNames` contents and order (JB-47–JB-53, JB-86);
path mapping in `Env.Param.*` and not `Env.RawParam.*` (JB-54–JB-59); INT `1` vs `"1"` (JB-10); per-document
extensions (JB-74, JB-75); wrap hooks (JB-76, JB-77); host requirements of a disabled entry and case-insensitive combination (JB-88–JB-90). The Basic Examples are JB-78–JB-85, and the pre-extension
`Conda` rejection in the second example is JB-19.

## Running the tests

From the repo root:

```bash
# Run just the ENV_PARAMS extension tests
uv run conformance-tests/run_openjd_cli_tests.py 2023-09/ENV_PARAMS

# Run only the job execution tests
uv run conformance-tests/run_openjd_cli_tests.py 2023-09/ENV_PARAMS/jobs
```

## Writing your own runner

See the top-level [conformance README](../../README.md) for the standard runner contract. The behavior specific to
`ENV_PARAMS`:

1. Each entry of a test's `environments` list is an Environment Template the scheduler attaches to the Job, in list
   order. The runner must resolve `environment:` entries against them at job creation as §3.2.1 describes, and reject
   the run (non-zero exit) for every job-creation error the RFC lists.
2. The runner's submission path format is the host's path format, and `pathMapping` rules apply to PATH and
   LIST[PATH] Environment parameter values in `Env.Param.*`.
3. The runner need not place Sessions by host requirements, but it must compute the combined host requirements of
   each Step at job creation and reject a conflict.

## Rules without a definite test

These rules, or parts of rules, have no planned test with a definite expected outcome. Each says why.

1. **Where attached Environments fall in `Job.EnvNames`, relative to the Job Template's own.** §7.3.1 orders the list
   "in the order a Session that entered all of them would enter them", but neither the Template Schemas nor *How Jobs
   Are Run* says whether attached Environments are entered before or after the Job Template's `jobEnvironments`
   (§1.2.1 says only that Environment Templates are processed "in an order defined by the render management system
   or submission"). JB-49 therefore accepts either placement with `anyOf`, and asserts only the relative order within
   each group. A definite test needs the base specification to state the entry order of attached Environments.
2. **Whether a template validator may reject a relative literal PATH `default` early.** §4.5 says the checks that
   depend on the submission path format, "including absoluteness, are made when the Job is created". A string such as
   `relative/dir` is relative in every path format, so a validator could know at template validation; the text does
   not say whether rejecting it then is permitted or forbidden. JB-61 tests rejection at job creation only, which is
   definite either way; no `job_templates/` or `env_templates/` test asserts that `openjd check` accepts it.
3. **§3.2.2 equality and Session sharing.** That two Steps with equal values may share a Session, that two with
   different values never do, and that a Session's values are fixed until it exits are scheduling properties.
   `openjd run` runs each Step in its own Session, so the suite can show only that `1` and `"1"` are both accepted for
   an INT and print the same value (JB-10), not that they are treated as equal.
4. **Placement and reservation by combined host requirements.** That a Session is placed only on a host satisfying the
   combined set, that `amounts` are reserved for the Session, and that amounts "do not add" (one GPU, not two) are
   scheduler behaviors with no output in `openjd run`. Only conflict and resolution rejection (JB-65–JB-69, JB-73,
   JB-89, JB-91) and acceptance of non-conflicts (JB-64, JB-70–JB-72, JB-88, JB-90) are tested.
5. **Warnings.** §3.2.1 item 4 says implementations should warn when an attached Environment shares a name with a Job
   Environment, and the rationale says a validator should warn when a `required: false` entry resolves to nothing.
   The test format has no way to assert a warning, and both are "should". JB-22 tests the resolution itself.
6. **The external-entry literal type check at template validation.** §3.2 says that for an entry naming no Job
   Environment, template validation can check "the type of a literal value where its form fixes one (a list is not a
   STRING)". Without the attached Environment, a validator does not know that the parameter is a STRING, so it is
   unclear what this check can reject. The only definite template-validation case is a list or boolean literal in a
   document without `EXPR` (JT-06, JT-07), which is invalid for a different reason. With `EXPR`, the list-for-STRING
   case is tested at job creation (JB-26).
7. **"`Job.EnvNames` ... inside a `{{ }}` expression".** The Validation row requires `Job.EnvNames` to appear inside a
   `{{ }}` expression, and §7.3.1 says it is "usable only within a `{{ }}` expression". Every value reference is
   inside `{{ }}`, so the rule forbids nothing that the base format-string grammar allows, and no invalid test is
   planned for it. The `args` case is definite: Expression Language §1.3.2 flattens a list result in an `args` item,
   so `args: ["{{Job.EnvNames}}"]` passes one argument per name (JB-86). Interpolating the list into surrounding text
   (`"x{{Job.EnvNames}}"`) is also defined (§1.3.2 converts it to its string form), so it is valid, not an error.
8. **`activation` case.** "An implementation MUST reject an `<EnvironmentActivation>` it does not recognize" does not say whether
   `always` or `On_Request` is recognized. Enum values elsewhere in the 2023-09 schema are case-sensitive, but the
   section does not say so. Only an unrelated value (`SOMETIMES`) is tested (JT-42, ET-14).
9. **Two attached Environment Templates with one Environment name.** §3.2.1 item 3 (ambiguous names) and the
   `Job.EnvNames` row ("it appears twice if two attached Environments share it") imply a submission may attach two
   Environments with the same name, but the base specification does not say whether attaching them is itself
   allowed. JB-21 passes either way. JB-51 assumes the attachment is accepted when no entry names the shared name; a
   runner that rejects duplicate attached names outright would fail it.
10. **Which document's `EXPR` admits a URI PATH value.** §4.5 says a PATH value must be absolute "or, with the `EXPR`
    extension, a URI", without saying whether the requesting document or the Environment's document must list `EXPR`.
    The RFC's rule that the requesting document's extensions decide a value's forms suggests the requester, but a URI
    is a path value, not a form. JB-63 lists `EXPR` in both documents, which is definite; the one-sided cases are not
    planned.
11. **Whether `enableIf` coerces to `bool`.** §3.2 item 5 describes a format-string `enableIf` as a single `{{ }}`
    expression "whose target type is `bool`", and the Expression Language (§1.2.3) converts `string` to `bool` toward
    a `bool` target (`"yes"` becomes `true`). Constraint 9 and the Validation row say instead "of type `bool`". So
    `enableIf: "{{ 'yes' }}"`, or one reading a STRING Job Parameter, is valid under the first reading and invalid
    under the second. JT-97 uses an INT expression, which no conversion turns into a `bool`, so it is invalid under
    both; the `string` case is not tested.
12. **An empty `hostRequirements` on an Environment Template.** §3.3 constraint 1 requires at least one of *amounts* or
    *attributes* "if this object is provided in a Job Template", and §4 item 5 gives an Environment's
    *hostRequirements* "the same shape and constraints as on a `<StepTemplate>`". In a Job Template the rule applies, so
    `hostRequirements: {}` on a Job Environment is rejected (JT-117). Whether `hostRequirements: {}` in an Environment
    Template is rejected depends on whether "in a Job Template" carries over, so no `env_templates/` test is written
    for it.
13. **An `enableIf` that can resolve to `null`.** §3.2 item 5 says "a `null` result is an error" at job creation, which
    requires an `enableIf` whose static type admits `null` (such as `bool?`, from `null if ... else true`) to pass
    template validation. Constraint 9 says the expression must be "of type `bool`", which a strict reading would apply
    to `bool?` and reject at template validation. JB-46 uses `{{ null if Param.Mode == 'off' else true }}` and is an
    `.invalid` test, so it passes under both readings, but under the strict reading the rejection comes from
    template validation, not job creation, and the test does not then exercise the job-creation rule.

14. **`RawParam.<PATH>` of a Job Parameter in an Environment's `hostRequirements`.** §4 item 5.1 lists "`RawParam.*`
    and `Param.*` (not PATH or LIST[PATH]) of the Job Parameters", which can be read with the parenthetical applying to
    both. The Validation row says "`RawParam.*` and non-PATH `Param.*`", which permits `RawParam.<PATH>`, as §4.5 item 4
    and §3.2 do for `default`s and values. The suite tests only `Param.<PATH>` (JT-50, rejected under both readings).
15. **Constraints 2, 5 and 6 on an entry disabled by a literal `enableIf: false`.** §3.2 says constraints 5–7 are
    checked at template validation for entries that name a Job Environment, with no exception for disabled entries,
    while item 2 says a disabled entry "has no effect of any kind" and only constraint 3 is said to apply "enabled or
    not". So `{environment: Conda, enableIf: false, parameters: {Bogus: x}}` may or may not be rejected. No test depends
    on the answer: JT-95, JT-103 and JT-104 supply only declared keys with valid values.
16. **A missing `min` in §3.3.3 versus a capability's defined minimum.** §3.3.3 item 1 says "a missing *min* is 0",
    while §3.3.1 item 2 defaults a missing `min` to the capability's own minimum (1 for `amount.worker.vcpu`). For such a
    capability the two give different combined `min`s and can disagree on whether another source's `max` conflicts. The
    suite uses only custom `conformance:` capabilities, whose minimum is 0, so no test depends on it.

## What the suite does not test

Deliberately, and why:

- **Composition with `SERVICE` and `CONTROL_FLOW`.** Every rule under the RFC's *Composition with SERVICE* and
  *Composition with CONTROL_FLOW* sections (Validation rows V7, V28, V29 for Service Sessions, V30, and the review-4
  resolutions Q1–Q3) is out of scope by the suite's charter, since neither extension is on mainline. No test uses
  either extension.
- **Scheduling behavior.** Session sharing and §3.2.2 equality, placement on a host that satisfies the combined host
  requirements, reservation of `amounts`, and "amounts do not add" have no output in `openjd run`, which runs each Step
  in its own Session and does not place Sessions (items 3 and 4 above). The suite tests only what job creation decides:
  conflicts are rejected and non-conflicts run.
- **Warnings** (item 5 above), since the test format cannot assert one.
- **The forms listed above as ambiguous** (items 1, 2 and 6–16) whose outcome the text does not decide. Where a form is
  definite under every reading, it is tested and the item names the test.
- **Repeat variants of a tested rule.** A rule that applies to several types or fields is tested where an
  implementation could plausibly differ, for example `objectType` on both PATH and LIST[PATH] (JT-25, JT-120). A literal
  `default` against `maxLength` is not tested separately from `minValue`, `allowedValues` and type (JT-34–JT-36), and a
  value reading `Param.<LIST[PATH]>` is not tested separately from `Param.<PATH>` (JT-106) and the `default` form
  (JT-121).
- **Running under an ENV_PARAMS implementation.** None exists yet, so no test has been run; see [AUDIT.md](AUDIT.md).
