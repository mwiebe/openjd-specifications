# ENV_PARAMS suite audit

An audit of the `ENV_PARAMS` conformance suite against
[RFC 0010](../../../rfcs/0010-environment-parameters.md) and the
[Template Schemas](../../../wiki/2023-09-Template-Schemas.md), made after the planning, template validation, and job
execution tasks were merged (integration branch at `49045e7`). No implementation of `ENV_PARAMS` exists, so every check
is a reading of the test against the specification text. No test has been run.

## What was checked

1. **Coverage.** Each row of the RFC's Validation table (V1–V30 in the [README](README.md#validation-table-coverage))
   has at least one test whose expected outcome and stage match the row, except the `SERVICE` and `CONTROL_FLOW` rows,
   which are out of scope. Each runtime behavior in the epic's `jobs/` scope (values in `onEnter`, `onExit` and
   `variables`; defaults from merged Job Parameters; external resolution with `required: true` and `false`; ambiguous
   names; `ONLY_REQUESTED`; entry `enableIf` true, false and `null`; `Job.EnvNames` contents and order; path mapping;
   INT `1` and `"1"`; constraint 7 at job creation; host requirement conflicts) has a test.
2. **Correctness.** All 240 existing files, and the 13 added here, were read by independent reviewers in five
   batches. For each `.invalid` file the reviewer named the single rule it violates and checked that nothing else in
   the file is invalid. For each `jobs/*.invalid.test.yaml`, the reviewer checked that the template and each attached
   Environment Template pass template validation alone, so the rejection can only come from job creation. For each
   valid file and `.test.yaml`, the reviewer checked that every field conforms, that every `{{ }}` reference is in
   scope (§3.6.2, §7.3.1, RFC §3.2, §4, §4.5), that each document lists the extensions it uses, and that every
   `expected.output` and `forbidden` line follows from the text. These points got particular attention:
   - `Job.EnvNames` order and contents. Attached-versus-own order is ambiguity 1, and JB-49 and JB-52 accept either.
   - No `Env.Param.*` or `Env.RawParam.*` of PATH or LIST[PATH] appears in an `enableIf` or `hostRequirements` of a
     valid file.
   - Entry `enableIf` is decided before name resolution (JB-45).
   - No valid file has `required: false` on an entry naming a `jobEnvironments` entry. Every `required: false` entry
     is in a Job Template with no Job Environment of that name.
   - No test assumes that amounts add (JB-70).
3. **Mechanical**, by script over every file:
   - every file parses with `yaml.safe_load` and starts with a comment;
   - every filename matches `<section>--<description>[.invalid][.test].yaml` with a known section, `.test` only in
     `jobs/`;
   - every `jobs/` file has `runOn` and no `bash` command with `windows` in `runOn`, and every valid run test has
     `expected`;
   - no file references `SERVICE` or `CONTROL_FLOW`;
   - each document that uses an `ENV_PARAMS` field, an `enableIf`, a BOOL or LIST type, or `Job.EnvNames` lists the
     extensions they need;
   - every file appears in the README matrix and every matrix row names an existing file.

## What was fixed

- **JT-110** `3.2--value-reads-env-scope.invalid.yaml`: the value read `Env.File.Spec`, but no Environment defined an
  embedded file `Spec`, so the file would also fail as an undefined reference. The requested Environment now defines
  it, and the only fault is that `Env.*` is not in scope in a `parameters` value.
- **JB-27** `3.2.1--non-expr-document-supplies-list-parameter.invalid.test.yaml`: the leading comment quoted the
  RFC's rationale heading as if it were §3.2 constraint text. It now quotes the §3.2 sentence.
- **README matrix**: the JB-73 row named parameters `A` and `B` instead of the file's `First` and `Second`. The JB-84
  row now shows that the attached Environment Template lists `FEATURE_BUNDLE_1`, which its format-string `min` needs.
  The JT-110 row describes the corrected file.
- **Added tests** for rules that had no test, or that were tested in one form only:

  | ID | File | Rule |
  |---|---|---|
  | JT-117 | `job_templates/4--host-requirements-empty.invalid.yaml` | `hostRequirements: {}` on a Job Environment (§3.3 constraint 1) |
  | JT-118 | `job_templates/4--host-requirements-env-raw-path-param.invalid.yaml` | `Env.RawParam.<PATH>` in Environment `hostRequirements` (§4 item 5.1) |
  | JT-119 | `job_templates/4--host-requirements-env-list-path-param.invalid.yaml` | `Env.Param.<LIST[PATH]>` in Environment `hostRequirements` |
  | JT-120 | `job_templates/4.5--list-path-object-type.invalid.yaml` | `objectType` on LIST[PATH] (§4.5 item 3) |
  | JT-121 | `job_templates/4.5--default-reads-list-path-param.invalid.yaml` | `default` reading `Param.<LIST[PATH]>` (§4.5 item 4) |
  | JT-122 | `job_templates/3.2--enable-if-reads-path-param.invalid.yaml` | `enableIf` reading `Param.<PATH>` (§3.2 item 5) |
  | JT-123 | `job_templates/7.3.1--env-raw-param-in-step-script.invalid.yaml` | `Env.RawParam.*` in a Step script (§7.3.1) |
  | JB-87 | `jobs/3.2.1--external-required-false-present-receives-values.test.yaml` | A `required: false` entry that resolves applies normally |
  | JB-88 | `jobs/3.3.3--any-of-intersection-ignores-case.test.yaml` | `anyOf` values intersect ignoring case (§3.3.3 item 2) |
  | JB-89 | `jobs/3.3.3--capability-name-compared-ignoring-case.invalid.test.yaml` | Capability names compare ignoring case, so a conflict is found |
  | JB-90 | `jobs/3.3.3--disabled-entry-requirements-not-applied.test.yaml` | A disabled entry contributes no host requirements |
  | JB-91 | `jobs/4--resolved-host-requirements-negative-min.invalid.test.yaml` | A resolved `min` of `-1` is rejected at job creation (V5) |
  | JB-92 | `jobs/3.2.1--attached-all-required-entry-disabled.invalid.test.yaml` | Constraint 7 for an attached `ALL` Environment when the only supplying entry is disabled |

  A second reviewer checked the added files. The README matrix, the Validation coverage table, and the runtime summary
  list them. The suite now has 123 `job_templates/`, 38 `env_templates/`, and 92 `jobs/` files.

No other defect was found. The earlier tasks' own audits had already corrected the `jobs/` assertions and citations.

## What remains ambiguous in the spec

The README's [Rules without a definite test](README.md#rules-without-a-definite-test) lists every item. Items 1–13
predate this audit. Items 14–16 are new, and item 12 is narrowed:

- Item 1: Where attached Environments fall in `Job.EnvNames` relative to the Job Template's own. The base specification does
  not give the entry order of attached Environments.
- Item 2: Whether a validator may reject a relative literal PATH `default` at template validation.
- Item 6: What the template-validation literal type check can reject for an entry naming an external Environment.
- Item 7: The "`Job.EnvNames` inside `{{ }}`" rule forbids nothing the format-string grammar allows.
- Item 8: Whether `runScope` values are case-sensitive.
- Item 9: Whether a submission may attach two Environment Templates with the same Environment name.
- Item 10: Which document's `EXPR` admits a URI as a PATH value.
- Item 11: Whether `enableIf` coerces to `bool` ("target type `bool`" in §3.2 item 5, "of type `bool`" in constraint 9).
- Item 12: Whether `hostRequirements: {}` is rejected in an Environment Template. The Job Template case is now tested
  (JT-117).
- Item 13: Whether an `enableIf` of static type `bool?` passes constraint 9, which decides the stage at which JB-46 is
  rejected.
- Item 14: **New.** Whether §4 item 5.1's "`RawParam.*` and `Param.*` (not PATH or LIST[PATH])" forbids a Job Parameter's
  `RawParam.<PATH>` in an Environment's `hostRequirements`. The Validation row permits it.
- Item 15: **New.** Whether constraints 2, 5 and 6 apply at template validation to an entry disabled by a literal
  `enableIf: false`.
- Item 16: **New.** Whether a missing `min` in §3.3.3 is 0, or the capability's own minimum from §3.3.1 item 2.

Items 3–5 are scheduling and warning behavior that `openjd run` cannot observe, not ambiguities.

## Not added

These were considered and left out. The reasons are in the README's
[What the suite does not test](README.md#what-the-suite-does-not-test).

- Repeat variants of rules that are already tested: a literal `default` against `maxLength`, a value reading
  `Param.<LIST[PATH]>`, `env_templates/` copies of the new `hostRequirements` scope tests, and an `enableIf` reading
  `Session.*`.
- A resolved attribute value that breaks the §3.3.2.2 pattern. V5 is already covered by JB-73 and JB-91.
