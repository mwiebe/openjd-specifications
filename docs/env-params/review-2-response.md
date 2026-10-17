# ENV_PARAMS RFC — Response to review round 2

Responds to `docs/env-params/review-2.md` (reviewed at 3803a8d). Every Required change and every Suggestion was
applied; nothing was declined. No fixed design decision was reopened.

Files changed: `rfcs/NNNN-environment-parameters.md`, `wiki/2023-09-Template-Schemas.md`, `wiki/How-Jobs-Are-Run.md`,
`wiki/2026-02-Expression-Language.md` (new for this RFC), and `samples/v2023-09/job_templates/video-sfm-composite.yaml`.

The RFC's embedded §3.2–§3.2.2 and §4.5 were edited together with the wiki by one script and re-compared after
normalizing absolute links and heading levels; they are still identical. All intra-document and RFC-to-wiki anchors
resolve (the dangling anchors that remain also dangle on mainline). All YAML in the RFC and samples parses, and every
bash embedded file in the samples passes `bash -n` (except the Docker `Enter` script, whose `{{ repr_sh(...) }}` is
substituted before bash sees it, as before).

## Required changes

### R1. How a supplied value becomes a typed value, and whose Job Parameters it may reference — applied

Schemas §3.2 and the RFC, `<EnvironmentParameterValue>` format-string bullet:

* `RawParam.*` and `Param.*` are now "those defined in the top-level *parameterDefinitions* of the document that
  contains the entry, with their merged values (see §1.2.1)", matching what §4.5 item 4 says for a `default`.
* A format string that is not a single `{{ }}` expression resolves to a string and is "converted to the parameter's
  type as a submitted Job Parameter value of that type would be, except that a relative path is not joined with any
  directory". This now backs the §3.2.2 INT `1` / `"1"` example.
* The specification defines no string form for a list, so a list-typed value must be a list literal or a single
  `{{ }}` expression. The same rule was added to §4.5 item 4 for a non-literal list `default`, together with the
  no-join rule for paths.

A new paragraph after the bullets says that the supplying document's extensions, not the Environment's, decide the
forms a value may take. Without `EXPR` in the supplying document, an entry cannot supply a parameter whose type
requires `EXPR`, and its Steps take the `default`. The paragraph gives the Docker `Mounts` example. Constraint 6 gains
the normative sentence: "An entry in a document that does not list `EXPR` must not supply a parameter whose type
requires `EXPR` (BOOL or a LIST type)." I chose the reading the review proposed. The alternative, letting the
Environment's extensions decide, would make the syntax a Job Template may use depend on the queue it is submitted to.

RFC additions: a rationale subsection "Values follow the supplying document's extensions", and two Validation-index
rows (the `EXPR` restriction of constraint 6, and the single-expression rule for list values and defaults).

### R2. A same-named Environment that declares no parameters — applied, keeping rejection (answers Q1)

The rule is unchanged: constraint 5 rejects an entry that supplies a parameter the resolved Environment lacks, whatever
`required` says. I verified the review's premise against `aws-deadline/deadline-cloud-samples`
`queue_environments/conda_queue_env_from_console.yaml` on mainline. It is named `Conda`, takes `CondaPackages` and
`CondaChannels` as Job Parameters, and lists no extensions. The rule is now stated wherever a reader relies on it:

* **Schemas §3.2.1 item 1** (and the RFC copy) says that this applies whatever *required* says: `required: false`
  tolerates a missing Environment, not one that lacks a supplied parameter. An entry that supplies parameters to an
  attached Environment that declares none, such as one written before `ENV_PARAMS`, causes the submission to be
  rejected, naming the entry and the parameter.
* **How Jobs Are Run, §Environment Parameters** (and the RFC's quoted copy) gains the same sentence in plain terms.
* **RFC §Backward compatibility** has a new bullet. It names the Deadline Cloud console Conda queue environment as the
  case that exists today and gives the remedy: add `ENV_PARAMS` and the recommended `parameterDefinitions`, with
  defaults that read the existing Job Parameters, as `conda-env-params.yaml` does. It also notes that this remedy is
  backward compatible for existing Job Templates.
* **RFC Basic Example 2** has a new paragraph after the example: `required: false` covers a queue with no Environment
  of the name, not a `Conda` Environment without `Packages`, which rejects the submission.
* **RFC §Recommended names** has a new paragraph: an Environment Template that adopts a recommended name should declare
  that name's recommended parameters, which matters most for `Conda`.
* **Rationale**: a new subsection, "An Environment without the supplied parameter is an error". It weighs the
  alternative of ignoring the entry, under which every Step would run with the usually empty job-wide `CondaPackages`
  and fail inside its Tasks after hosts were provisioned. It also explains that the fix belongs to the queue
  administrator.
* **Job sample header** now says that a queue whose Conda or Docker Environment declares no ENV_PARAMS parameters,
  such as the console Conda queue environment, rejects the Job at submission, and how its administrator updates it.
* The Validation-index row for constraint 5 on external Environments notes that this applies even with
  `required: false`.

## Suggestions

* **S1. Residual SERVICE text — applied.** RFC §Composition with SERVICE:
    * Item 6 adds the SERVICE §3.6.2 `<Service>.let` row change (bindings available in *parameters* values).
    * Item 7 gives the replacement second sentence of constraint 7, which limits it to Environments whose `runScope`
      includes `TASK`, and says that §3.2.1's last paragraph is qualified the same way.
    * New item 9 amends SERVICE §1.2.2's "Items 2 and 4 are the only checks that require the combined Job" to include
      the ENV_PARAMS §3.2.1 checks.
* **S2. `Env.Param` row — applied** in Schemas §7.3.1 and the RFC: "except for types PATH and LIST[PATH]".
* **S3. Why `Param.<PATH>` is excluded — applied.** RFC §Values resolved at job creation has a new paragraph:
  `Param.<name>` of a path is the host-mapped value, which does not exist at job creation, and the Environment's
  `Env.Param.*` applies the mapping.
* **S4. Rationale overclaim — applied.** Item 3 of §External Environments are named implicitly is now limited to
  `required: true`. It says that a misspelled `required: false` name is silently ignored, and that a validator given
  the queue's Environment Templates should warn when such an entry resolves to nothing.
* **S5. Step scripts in the job sample — applied.** Every Step script now receives its values as argv (`args`) and
  reads `"$1"`, `"$2"`, and so on. No `Param.*`, `Task.Param.*`, or `Task.File.*` is substituted into bash source. The
  header notes why. Unlike `repr_sh`, this form does not depend on `EXPR`.
* **S6. Expression Language symbol table — applied.** `wiki/2026-02-Expression-Language.md` §1.2.2 has a new
  "Environment Parameter Symbols" table after Embedded File Symbols. It gives `Env.Param.<name>` and
  `Env.RawParam.<name>` the `Param` and `RawParam` types of the Job Parameter table. The RFC has a matching new section,
  "Modifications to the Expression Language".
* **S7. Design Tenets paraphrase — applied.** Motivation now applies the Portable tenet ("portable to many different
  systems", "the technology stack of their choice") rather than quoting a requirement the tenet does not state.
* **S8. Summary line wrap — applied.**

## Questions

* **Q1.** Rejection, documented (see R2). It fails at submission with an error naming the entry and the parameter, and
  it leaves the meaning of an Environment name unambiguous.
