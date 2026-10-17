# ENV_PARAMS RFC — Response to review round 1

Responds to `docs/env-params/review-1.md`. Section references are to `rfcs/NNNN-environment-parameters.md` ("RFC"),
`wiki/2023-09-Template-Schemas.md` ("Schemas"), and `wiki/How-Jobs-Are-Run.md` ("HJAR") as revised. None of the
changes reopens a design decision fixed in the epic description, and none of the Required changes conflicted with one.

## Required changes

### R1. Single-Step Session premise — changed

* The premise is replaced throughout. RFC Summary, RFC §Motivation "Why the values come from the Step", and HJAR
  §Environment Parameters now say that a Session enters each Environment once with one set of values, and may run
  Tasks of several Steps when their values agree.
* Schemas §3.2.2 (and the RFC, which carries the same text) defines **equal** values and the Session rule: a Session may
  run Tasks of a Step only when, for every Environment it has entered, the Step's effective values equal the values it
  entered with. The HJAR L27 sentence is amended to the same rule.
* The `Env.Param.*` / `Env.RawParam.*` rows (Schemas §7.3.1, RFC §Value references), the §4 `Env.*` scope item, and the
  RFC Terminology (new rows "Equal values" and "A Session's values") now refer to the values the Session entered the
  Environment with, which equal the effective values of every Step it serves.
* RFC §Implementation Impact now says a scheduler that shares Sessions across Steps must compare effective values;
  only a scheduler that never shares across Steps needs no change.
* RFC Basic Examples §Execution now uses DecodeFrames/EncodeVideo from the sample to illustrate sharing.
* Q1 is answered in the rule itself (see Q1 below).

### R2. RFC and wiki `<StepDependency>` text, composition with SERVICE — changed

* The RFC's §`<StepDependency>` now gives the schema change as a diff against mainline (no `service:` lines) and then
  the complete new §3.2, §3.2.1, and §3.2.2 verbatim from the wiki, with only links made absolute and headings demoted.
  The same is done for §4.5. A script check confirmed the embedded text matches the wiki. All constraint
  cross-references in the RFC now use the wiki numbering (constraints 1–8).
* The RFC's former constraint 2 (no `environment:` on an Environment's `dependencies`) only makes sense with SERVICE,
  so it moved to the new RFC §Composition with SERVICE.
* That section gives the merged rules: one "exactly one of *dependsOn*, *service*, or *environment*" constraint; SERVICE
  constraint 2 and ENV_PARAMS constraint 3 stay separate rules about separate keys; `environment:` entries are not nodes
  or edges of SERVICE constraint 3's graph; and SERVICE's `<StepTemplate>` item 4 gains the third alternative. The wiki
  on this branch is based on mainline, which has no SERVICE, so the composition edits are in the RFC only, and
  whichever RFC is merged into the wiki second applies them. The RFC says so.
* The "checked at" statement now reads the same in both documents: constraints 1–4 and 8 at template validation;
  5–7 at template validation for Job Environment entries and at job creation for all.

### R3. Scope lists — changed

* Schemas §3 item 3 (*let*): bound names are also available in the *parameters* values of *dependencies* entries.
* Schemas §3.6.2 table: the `<StepTemplate>.let` row gains the same; the `<EnvironmentScript>.let` "Can Reference" cell
  gains `Env.Param.*` and `Env.RawParam.*`.
* Schemas §7.3.1 `Step.Name` row gains the *parameters* values of `dependencies` entries.
* The `<EnvironmentScript>` *let* item (§4.2) does not enumerate scopes, so it is unchanged; the RFC says the §3.6.2
  row is the normative list.
* The RFC mirrors these as diffs in the new §`<StepTemplate>` and scope lists.

### R4. Services subsection — changed

Replaced by RFC §Composition with SERVICE, items 5–8:

* `runScope` check: a template validation error for a Job Environment entry, a job-creation error for an external one
  (item 8).
* Environment Template Services: `environment:` entries are **not** permitted there. SERVICE §1.2 item 6 constraint 5
  stays unchanged, and the RFC gives the reason (one author writes both the Service and the document's Environment, as
  for Step Environments) (item 5). `environment:` entries are permitted on Job Template Services.
* Service Sessions enter every attached Environment with the default `runScope`, so constraint 7 applies to every
  Service for such an Environment's parameters without a `default` (item 7).

### R5. Name-resolution rationale — changed

The RFC rationale is split. "Names resolve to the Job Template first" keeps only the first point. A new rationale,
"External Environments are named implicitly", states the difference from RFC 0009's `externalServices` and gives
three reasons it is acceptable. The main reason is that a Job Template never reads `Env.Param.*`, so no expression in
it depends on the external Environment's shape. Schemas §3.2 (closing paragraph) now states what a template validator
checks for an external entry: constraints 1–4 and 8, and the form of literal values. It also says a validator that is
given the Environment Templates can run the job-creation checks early.

### R6. PATH format — changed (first option, with a recommendation)

Schemas §4.5 and the RFC now say every PATH and LIST[PATH] Environment parameter value is a path in the Job's
submission path format, the `source_path_format` path mapping is keyed on. It must be absolute in that format, or a
URI with EXPR, when the Job is created. `allowedValues`, `minLength`, `maxLength`, and `item` are checked then,
against the unmapped value, and are not re-checked after mapping. The text recommends taking paths from
`RawParam.*`, says a literal PATH value is portable only between submitters on one OS, and recommends STRING for
host-local paths. The RFC rationale "Path values use the submission path format" explains why the
`{{RawParam.X}}`-only option was not chosen: it would forbid `{{RawParam.WorkDir}}/cache` and the sample's list of
paths. New Validation-index row.

As a consequence of R6 and R8, **LIST[PATH] is now permitted** as an Environment parameter type. It was excluded in
the draft because LIST[PATH] Job Parameter defaults are joined with the template directory. Once paths are defined as
absolute in the submission format, that reason no longer applies, and the Docker sample needs a mapped list of host
paths. See the RFC rationale "`RANGE_EXPR` excluded, `LIST[PATH]` permitted". This is a change in the specified type
set, so it is flagged here.

### R7. When a `default` is validated — changed

Schemas §4.5 and the RFC: a literal `default` (no `{{ }}`) is checked at template validation, even if every Step
overrides it. Any other `default` is resolved at job creation and checked only for the Steps that take it, so an
invalid job-wide fallback is not an error when every Step supplies its own value. Rationale "When a `default` is
checked"; two new Validation-index rows; the §7.4 Template validation row mentions literal defaults.

### R8. Docker path of the pipeline sample — changed

* `docker-env-params.yaml` declares `Mounts` (LIST[PATH], default `[]`) and `Gpus` (STRING, default `""`). Its `onEnter`
  bind-mounts the Session directory and every listed path at the same path in the container. For a file, or a path
  that does not exist yet, it mounts the parent directory, creating it if needed. Duplicate mounts are skipped. A
  non-empty `Gpus` adds `--gpus`. All values reach the script as argv.
* `video-sfm-composite.yaml` now lists EXPR. Every Docker entry supplies
  `Mounts: "{{ [RawParam.InputVideo, RawParam.MayaScene, RawParam.WorkDir, RawParam.OutputVideo] }}"`, written once
  with a YAML anchor, and SolveSfM supplies `Gpus: "all"`. Path mapping applies on the host because the parameter is
  LIST[PATH].
* Verified by running the `onEnter` script with a stub `docker`. It produced the expected `--gpus` and `--mount`
  arguments for an existing file, an existing directory, a not-yet-existing output file (whose parent it created),
  and a duplicate. All three YAML files parse, and every `environment:` key matches a declared parameter.

### R9. Session-sharing comment in the job sample — changed

The header now says Steps whose values differ never share a Session, and that DecodeFrames and EncodeVideo supply
equal values and may share one. The EncodeVideo comment now says the same.

### R10. Security Considerations and unsafe interpolation — changed

* New RFC §Security Considerations covers the trust model (the same as `Param.*` today, no new privilege, use
  `allowedValues` to restrict, treat `Env.Param.*` as untrusted). It explains why substituting values into script
  source is unsafe and gives the safe forms: argv `args`, `variables`, and `repr_sh` / `repr_cmd` / `repr_pwsh`. It
  also recommends `set -f` when word-splitting on purpose, and notes that `onExit` sees the same values as `onEnter`.
* `conda-env-params.yaml` and RFC Basic Example 1 now pass Packages and Channels as `args` and read `"$1"`/`"$2"`,
  with `set -f`.

## Suggestions

* **S1. Accepted.** Schemas §7.4: `Env.Param.*` is added to the Unknown Values of Template validation. Each Step's
  effective values are added to the Known Values of Job creation, at the end of that stage. Task execution now
  mentions path mapping of PATH/LIST[PATH] Environment parameters. The RFC's §Template processing stages matches.
* **S2. Accepted.** Schemas §3.2 constraint 7 and §3.2.1 state that every Session enters every Job Environment and
  every attached Environment, and what follows for a queue Environment with a required parameter. The SERVICE case
  is in Composition item 7.
* **S3. Accepted, as an error.** New constraint 8: `required: false` on an entry that names a Job Environment is a
  template validation error. Item 3 and the Validation index are updated, and there is a rationale.
* **S4. Accepted.** Schemas §3.2.2 defines equality as typed values as resolved at job creation, before path mapping,
  with an INT example. RFC rationale "Sessions that serve several Steps".
* **S5. Accepted.** Schemas §3.2 item 2: an entry does not delay scheduling, cause an Environment to be entered, or
  change entry order. HJAR repeats the last two. The SERVICE graph point is Composition item 3.
* **S6. Accepted.** The job sample header says to attach one of the two Environments and why. Schemas §3.2.1 and HJAR
  say that entries apply independently.
* **S7. Accepted.** RFC Future Work cites the sample (eight entries for four Steps) and mentions YAML anchors, which
  the sample now uses for `Mounts`.
* **S8. Accepted.** Snakemake now cites `--software-deployment-method` (Snakemake 8) alongside the older flags. New
  Prior Art entries cover the Deadline Cloud sample queue environments (`CondaPackages`/`CondaChannels`,
  `RezPackages`, and the hash-cached Conda sample) as the baseline, Airflow `executor_config`/`pod_override`, and Rez.
* **S9. Accepted.** The job sample header now says "Once ENV_PARAMS is implemented in the openjd tools".
* **S10. Accepted.** New Schemas §3.2.1 item 5: an attached Environment with the same name as a Step's Step Environment
  cannot be reached from that Step.
* **S11. Accepted.** `SceneFile` is removed from Basic Example 3.
* **S12. Accepted.** New non-normative RFC §Recommended names, with a table for `Conda`, `Rez`, and `Docker`. Open
  Question 1 now points to it and asks only whether it should become a wiki page.
* **S13. Accepted.** The claim is softened to "can", with Qt, OpenEXR, and Python as examples of shared libraries.

## Questions

* **Q1. Answered in the specification.** No, the set of Steps does not need to be known when the Session starts. The
  rule (Schemas §3.2.2, HJAR) is a condition on adding a Step's Tasks to a Session: the Step's effective values must
  equal the values the Session entered each Environment with. A scheduler may therefore add a newly ready Step's Tasks
  to a running Session. The RFC rationale "Sessions that serve several Steps" explains the choice.
* **Q2. Answered.** The error stays. RFC §Composition with SERVICE recommends `runScope: [TASK]` for a Job Environment
  whose required parameters only Steps supply, and says the error names the Service and the parameter.
* **Q3. Answered.** New RFC rationale "`Step.Name` in values": the name is available for consistency with the rest of
  the Step's job-creation scope. Using it makes a Step's values unique, so Steps that use it never share Sessions.
  That suits per-Step cache keys or log labels, and is not the common case.

## Not addressed

* **Tracking issue** (checklist): still `(pending)` and `NNNN`, per the task instructions. The user must open the issue
  before the PR.
* **Sample CI**: `.github/workflows/check_samples.yml` runs `openjd check` on every sample. The released openjd-cli
  does not implement ENV_PARAMS, so the three ENV_PARAMS samples will fail that job until it does. This was already
  true of the draft, and this revision does not change it.
