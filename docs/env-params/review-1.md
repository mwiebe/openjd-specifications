# ENV_PARAMS RFC — Review round 1

Reviewed at `rfc-env-params` 2009d70, applying `skills/openjd-rfc-review/SKILL.md`.

Scope:

* `rfcs/NNNN-environment-parameters.md` (cited below as "RFC Lnnn").
* `git diff mainline -- wiki/`, covering `wiki/2023-09-Template-Schemas.md` ("Schemas Lnnn", line numbers in the
  post-change file) and `wiki/How-Jobs-Are-Run.md` ("HJAR Lnnn").
* `samples/v2023-09/environment_templates/conda-env-params.yaml`, `.../docker-env-params.yaml`,
  `samples/v2023-09/job_templates/video-sfm-composite.yaml` ("job sample Lnnn"), and `samples/README.md`.
* The `service:` form of `<StepDependency>` in `~/openjd-specifications`, branch `rfc-service` (71fae98),
  `wiki/2023-09-Template-Schemas.md` §3, §3.2, §1.2, §1.2.2, §4, and §9 ("SERVICE §x").

The design decisions in the epic description are treated as fixed. This review covers how they are specified.

## 1. Summary assessment

**Not ready.** The draft is complete in structure and unusually thorough: every template section is present and
substantive, and the Specification is written as drop-in text with a validation index. A Validation section,
Implementation Impact, and Future Work are also included. The core mechanism (Environment-declared parameters,
an `environment:` dependency entry, per-Step effective values fixed at job creation) is specified with care.

The draft has one foundational error. It rests on the premise that "a Session runs Tasks from a single Step"
(RFC L23, L237; HJAR L99). The specification it modifies says the opposite at HJAR L27: Tasks from any Step may
share a Session when the Steps' Environments are identical. The draft's own Session-sharing rule depends on that
sentence. The result is a spec that contradicts itself within one wiki page and leaves `Env.Param.*` undefined
for a Session that serves two Steps. The seed discussion makes the same claim, but the fixed design decisions do
not depend on it, so it can be corrected without reopening them.

Beyond that, the RFC's `<StepDependency>` text and the wiki's §3.2 have drifted apart, and the composition with
SERVICE's §3.2 is not specified. Several scope tables were not updated for the new value sources. PATH-typed
Environment parameters have an unspecified source path format. The Docker sample cannot run the pipeline it
ships with. The samples also demonstrate an unsafe interpolation pattern that the RFC never discusses.

## 2. Strengths

* **Clear problem statement.** RFC §Motivation (L183–L211) lays out the three bad options available today
  (install everything, one image, split Jobs). It then explains why a Step Environment cannot help. The four
  additional workflows in L213–L233 are concrete and distinct.
* **Backward compatibility is reasoned per document.** RFC L242–L260 covers all four combinations of Job
  Template and Environment Template with and without `ENV_PARAMS`. The rule "an Environment parameter defaults
  to the Job Parameter" keeps existing Job Templates working with a queue that adopts the feature.
* **Reuse of `<JobParameterDefinition>`.** `<EnvironmentParameterDefinition>` (RFC L348–L398, Schemas §4.5)
  reuses existing shapes and names the excluded fields and types with a reason for each. Readers learn nothing
  new beyond the differences.
* **Static effective values.** Resolving all values at job creation (RFC §Effective values L521–L534,
  rationale L676–L683) makes the values a property of the Job. Tooling can display them, and a scheduler can
  group and cache by them.
* **Validation index.** The table at RFC L596–L612 lists every new rule with the stage that checks it. This is
  what an implementer needs and what RFC 0008 lacks.
* **Rejected Ideas are well argued.** The `stepOverridable` rejection (L783–L794) names three independent
  failures (it breaks `Param.*` meaning, interacts with §1.2.1 merging, and gives the Environment no say). The
  flat-map rejection (L796–L803) names four.
* **Prior Art is relevant.** Nextflow and Snakemake per-process directives, which the platform enables
  selectively, are exactly the `required: false` pattern, and the RFC says so (L723–L735).

## 3. Tenet alignment

* **Portable — mostly satisfied, with one gap.** Optional `environment:` entries let one Job Template run on
  Conda, Docker, or preinstalled queues, which directly serves this tenet. Portability across queues, however,
  depends entirely on queues agreeing on Environment names (`Conda`) and parameter names (`Packages`). The RFC
  knows this (Open Question 1, L840–L845) and defers it to a possible wiki page. That is acceptable as
  non-blocking, but see Suggestion S12. Separately, PATH-typed Environment parameters break portability as
  specified: a literal absolute `default` in a queue Environment Template has no defined source path format for
  path mapping (Required R6).
* **Expressive — satisfied.** The mechanism is general: any Environment, any parameter, any Step. It is not tied
  to Conda or Docker, and the license and dataset examples show non-software uses.
* **Human readable — satisfied.** `environment: Conda` / `parameters: {Packages: ...}` reads naturally next to
  `dependsOn:`, and the Environment's `parameterDefinitions` reads like a Job Template's.
* **Human writable — partially satisfied.** The pipeline sample repeats two near-identical four-line entries on
  every Step (8 entries for 4 Steps). That is tolerable, but it grows linearly with the number of provisioning
  mechanisms a template supports. See Suggestion S7.
* **Tooling parseable — partially satisfied.** Within a Job Template everything is checkable. For an external
  Environment, `openjd check` cannot validate any key or value, because resolution happens only at job creation.
  RFC 0009 deliberately avoided this for Services by requiring an explicit `externalServices` declaration. The
  RFC claims to match RFC 0009 here (L665–L666), which is inaccurate (Required R5).

## 4. Specification compatibility

* **The Session model is contradicted** (R1). HJAR L27 allows Sessions to span Steps, and the draft extends
  that rule. Meanwhile RFC L23 and L237, HJAR L99, the "Effective values" terminology (RFC L269, "for one supplying
  entity"), and the `Env.Param.*` definition (Schemas L2201, "for the Step whose Session is entering") all assume
  one Step per Session.
* **`<StepDependency>` composition with SERVICE is underspecified** (R2). On `rfc-service`, SERVICE §3.2 has its
  own constraint list (constraints 1–4). Its constraint 4 says "exactly one of the two keys", its constraint 2
  forbids duplicate Step/Service entries, and its constraint 3 defines an acyclic graph over `dependencies`. The
  RFC's diff (L415–L425) marks SERVICE's `service:` lines as `+` additions of this RFC. It then gives a separate
  1–8 list that does not say how it merges with SERVICE's 1–4, or whether `environment:` entries are nodes of
  SERVICE's graph. The `environment:` form is otherwise consistent with `service:`: same list, one key per kind
  of dependency, exactly-one-key rule, no duplicates, not permitted on `stepEnvironments`. SERVICE puts
  Environment `dependencies` under the `service` key only, and RFC constraint 2 (L470–L472) correctly keeps
  `environment:` off Environments.
* **Environment Template Services conflict** (R4). SERVICE §1.2 item 6 constraint 5 requires every `dependencies`
  entry of an Environment Template's Service to use the `service` key. RFC §Services (L511–L519) permits
  `environment:` entries on any `<Service>` and does not address this.
* **Scope tables not updated** (R3). The new value sources (`Step.Name` and Step `let` in `dependencies`,
  `Env.Param.*` in `<EnvironmentScript>.let`) are added in prose but not in the tables and lists that define
  scope (§3 item 3, §3.6.2, §7.3.1 `Step.Name` row).
* **Fits existing concepts well otherwise.** Putting values in `dependencies`, keeping `Env.*` meaning "this
  Environment", and resolving at job creation alongside Step `let` all reuse existing mechanisms. The §1.2.1
  note (Schemas L140–L145) correctly isolates Environment parameters from Job Parameter merging.

## 5. Checklist

| Item | Status | Notes |
|------|--------|-------|
| **Completeness** | | |
| All template sections present and substantive | ✅ | Plus Validation, Implementation Impact, Future Work. A Security Considerations section is missing (R10). |
| Metadata fields filled correctly | ✅ | Tracking issue and Accepted On are placeholders as instructed. |
| Tracking issue exists and linked | ⚠️ | `(pending)` and `NNNN`, per the task instructions. The user must open the issue before the PR; not actionable by a revise task. |
| Examples are runnable and demonstrate the feature | ❌ | The Docker path of the pipeline sample cannot run (R8). The Conda sample interpolates unsafely (R10). |
| **Clarity** | | |
| Summary captures the essence in one paragraph | ⚠️ | Yes, but it states the incorrect single-Step premise (R1). |
| Motivation explains WHY | ✅ | |
| Examples succinctly demonstrate the proposal | ✅ | Basic Example 3 declares an unused `SceneFile` (S11). |
| Specification is precise enough to implement | ❌ | R1, R2, R6, R7. |
| Design rationale explains key decisions | ⚠️ | The rationale for name resolution misstates RFC 0009 (R5). |
| **Tenet Alignment** | | |
| Portable | ⚠️ | PATH source format (R6); naming conventions (S12). |
| Expressive | ✅ | |
| Human readable | ✅ | |
| Human writeable | ⚠️ | Repetition per Step (S7); acceptable. |
| Tooling parseable | ⚠️ | External entries are unvalidatable at template validation and not acknowledged (R5). |
| **Specification Compatibility** | | |
| Proposed language fits naturally | ⚠️ | R2, R3. |
| High-level ideas fit existing concepts | ❌ | The Session model contradiction (R1). |
| New concepts apply orthogonally | ⚠️ | SERVICE composition (R2, R4). |
| Combined work is consistent and whole | ❌ | RFC and wiki diverge (R2); HJAR contradicts itself (R1). |
| **Technical Quality** | | |
| Specification diff is syntactically correct | ❌ | The §3.2 diff marks SERVICE lines as additions (R2). |
| Examples validate against proposed schema | ⚠️ | All YAML parses. Checked by hand: every `environment:` key matches a declared parameter, defaults reference only same-document Job Parameters, extensions are declared, no Step reads `Env.Param.*`, and Basic Example 3 satisfies constraint 8. Comments in the job sample contradict the Session rule (R9). |
| Edge cases addressed | ⚠️ | Unused invalid defaults (R7), PATH (R6), Sessions spanning Steps (R1). |
| Backward compatibility considered | ✅ | |
| Extension naming follows conventions | ✅ | `ENV_PARAMS` matches `[A-Z_0-9]{3,128}` and is listed in §1.1 item 3. |
| **Process** | | |
| Prior art shows research | ✅ | Some refinements in S8. |
| Rejected ideas explain why | ✅ | |
| No unresolved open questions blocking acceptance | ✅ | Both listed questions are non-blocking. Q1 below is not. |

## 6. Required changes

### R1. Correct the single-Step Session premise and define `Env.Param.*` for a Session that serves several Steps

RFC L23 ("Because a Session runs Tasks from a single Step…"), RFC L237 (§Why the values come from the Step),
and HJAR L99 state that a Session serves one Step. HJAR L27, which the draft itself edits, says "Tasks from any
Step in a Job can run within the same Session provided that the set of Environments … are identical". The RFC's
Execution example (L177–L178) and the job sample (L259–L261) also rely on cross-Step sharing. So the spec
currently says both things.

Fix:

* Restate the premise as: every Session enters its Environments once, for Tasks of one or more Steps, and with
  `ENV_PARAMS` a Session may serve several Steps only when their effective values agree for every Environment
  it enters. Therefore one set of values is in force for the whole Session.
* Define effective values per Step (as now). Then define a Session's Environment parameter values as the
  common effective values of the Steps it serves. Update the `Env.Param.*` row (RFC L542, Schemas L2201), the
  §4 scope item 3 (Schemas L1670–L1675), the Terminology row (RFC L269), and HJAR L99–L102 accordingly. "For
  the Step whose Session is entering it" is undefined when there are two.
* Fix RFC L774–L776 (Implementation Impact). A scheduler that today shares Sessions across Steps whose
  Environments match must now also compare effective values. Only a scheduler that never shares across Steps
  needs no change.

### R2. Make the RFC's `<StepDependency>` text identical to the wiki, and specify its composition with SERVICE

The two texts differ:

* The RFC has 8 constraints (L466–L487); the wiki has 7 (Schemas L1002–L1014). The RFC's constraint 2
  (Service/Environment placement) has no wiki counterpart. As a result, every cross-reference differs: the RFC
  refers to constraints 6 and 8 at L258, L381, L489, L503, L506, and L530, while the wiki refers to 5 and 7.
* The RFC says only "Constraints 6 and 7 are checked at template validation for … jobEnvironments" (L489–L490),
  yet constraint 8 is also checked at template validation (L485–L487). The wiki's L1016–L1018 has a different,
  more complete statement.
* The RFC diff (L415–L425) shows `service:` and the `# or, with the SERVICE extension` comment as `+` lines,
  so the diff is wrong against mainline and wrong against `rfc-service`.

Fix:

* Write the RFC's §3.2 change as a diff against mainline that matches the wiki byte for byte.
* Add a "Composition with SERVICE" note that gives the merged §3.2 text for when both extensions are present.
  It should cover: SERVICE constraint 4 becomes "exactly one of *dependsOn*, *service*, or *environment*";
  SERVICE constraint 2's duplicate rule extends to Environments (or stays separate, but say which); and
  `environment:` entries are not nodes or edges in SERVICE constraint 3's acyclic graph. SERVICE's item 4 under
  `<StepTemplate>` ("Each names a Step … or … a Service that must be READY") also needs the third alternative.

### R3. Update every scope list and table for the new value sources

The RFC lets `parameters` values use `Step.Name` and the Step's `let` bindings (RFC L459–L462, Schemas
L996–L997). It also lets `<EnvironmentScript>.let` read `Env.Param.*` (RFC L542–L543). The normative scope lists
were not changed:

* Schemas L908–L909 (`<StepTemplate>` item 3 *let*): "Bound names are available in *stepEnvironments*,
  *hostRequirements*, *parameterSpace*, and *script*". This needs *dependencies* (the `parameters` values of
  `environment:` entries).
* Schemas §3.6.2 table (L1606–L1611): the `<StepTemplate>.let` "Bound Names Available In" cell needs the same
  addition. The `<EnvironmentScript>.let` "Can Reference" cell needs `Env.Param.*` and `Env.RawParam.*`.
* Schemas L2205 (`Step.Name` row): "Available within the Step Template scope: `stepEnvironments`,
  `hostRequirements`, `parameterSpace`, and `script`" needs `dependencies`.
* The `<EnvironmentScript>` item 1 *let* text (Schemas L1721) should list `Env.Param.*` if it enumerates
  scopes.

Mirror these edits in the RFC's Specification section.

### R4. Fix the Services subsection

* RFC L518–L519 says an entry naming an Environment whose `runScope` excludes `SERVICE` "is a template
  validation error". For an external Environment, `runScope` is not known until job creation. State that it is
  a template validation error for a `jobEnvironments` entry and a job-creation error for an external one.
* SERVICE §1.2 item 6 constraint 5 requires every `dependencies` entry of an Environment Template's Service to
  use `service`. State whether an Environment Template's Service may list `environment:` for that document's own
  `environment`. If it may, amend that constraint in the composition note. If it may not, restrict RFC L470
  ("permitted in the `dependencies` … of a `<Service>`") to Job Template Services.
* State that an attached Environment with the default `runScope` (every kind of Session) is entered by Service
  Sessions. Constraint 8 therefore applies to every Service of the Job for such an Environment's parameters
  without a `default`. The text at L506–L510 implies this but does not say it.

### R5. Correct the name-resolution rationale and acknowledge the validation gap

RFC L659–L666 says resolving external names at job creation "matches how RFC 0009 matches `externalServices`".
It does not. RFC 0009 requires the Job Template to declare each external Service in `externalServices`, and
SERVICE §1.2.2 states the goal "every `Service.*` reference in every template can be validated without knowledge
of the scheduler's configuration". Here an `environment:` entry names an external Environment implicitly, and
its keys and values cannot be checked until job creation (RFC L489–L490).

Decision 3 fixes the implicit form, so do not add a declaration. Do the following instead:

* Rewrite the rationale to state the difference and why it is acceptable. Possible reasons: `required: false`
  entries are meant to be ignored on queues that lack the Environment, and a declaration would add a second
  place to write the name without adding information.
* State, under Tooling parseable, what a template validator does with an external entry. It checks the
  structure (constraints 1–5 in the wiki numbering) and the literal types where possible. A validator given the
  Environment Templates the scheduler will attach (as `openjd run` already accepts them) can perform the
  job-creation checks early.

### R6. Specify the path format of PATH-typed Environment parameter values

RFC L383–L389 and Schemas L1045–L1048 require a PATH value to be absolute or a URI, and apply path mapping on
the Worker Host "as for PATH Job Parameters". Path mapping rules are keyed on the *submitting* OS's
`source_path_format` (HJAR §Path Mapping). A PATH value from `{{RawParam.X}}` has the submitter's format. A
*literal* PATH `default` written in a queue administrator's Environment Template, or a literal PATH value in a
Job Template, has the format of whoever wrote it. "Absolute" is also OS-dependent: `/opt/x` is not absolute on
Windows.

Choose one and specify it:

* Literal PATH values and defaults are interpreted in the Job's submission path format, and the RFC states
  that authors should use them only on single-OS farms; or
* PATH values and defaults must be a single `{{RawParam.<name>}}` (or, with `EXPR`, an expression over
  `RawParam`/`Param` paths), so every Environment PATH value originates from a submitted Job Parameter. A
  literal path then uses type `STRING`.

Also say whether `allowedValues`, `minLength`, and `maxLength` on a PATH Environment parameter are checked
before or after resolution. §2.2 checks Job PATH parameters at both stages (Schemas L286–L297).

### R7. Specify when a `default` is validated, and whether an unused invalid default is an error

RFC L376–L378 and Schemas L1869–L1871 say the constraints "constrain every effective value, whether supplied by
a Step or taken from the resolved `default`". That leaves two cases open:

* A `default` that resolves to a value outside the parameter's constraints, when every supplying entity
  overrides it. For example, `Image` with `minLength: 1` and `default: "{{Param.DockerImage}}"` where
  `DockerImage` is empty, but every Step supplies `Image`. A `<JobParameterDefinition>` requires its `default`
  to satisfy its constraints (§1.2.1: "the value of the `default` property must adhere to the constraints").
  Say whether an Environment parameter follows that rule at job creation or checks only effective values.
* A literal `default` (no format-string reference). Say whether it is checked at template validation, as a Job
  Parameter's literal default is.

Add the chosen rows to the Validation index (RFC L596–L612).

### R8. Make the Docker path of the pipeline sample runnable

`docker-env-params.yaml` bind-mounts only `Session.WorkingDirectory`. Every Step of `video-sfm-composite.yaml`
reads or writes `{{Param.InputVideo}}`, `{{Param.WorkDir}}`, `{{Param.MayaScene}}`, or `{{Param.OutputVideo}}`.
These are host paths outside the Session directory, so inside the container they do not exist and the first
Step fails. SolveSfM also requests `amount.worker.gpu` (job sample L126–L129), but the container is started
without `--gpus`, so COLMAP runs on CPU or fails depending on the image's build.

ENV_PARAMS itself offers a clean fix. Give the Docker Environment a `Mounts` parameter (a space-separated
`STRING`, or a `LIST[STRING]` with `EXPR`), and have each Step supply it from `{{RawParam.WorkDir}}` and the
other PATH parameters. Alternatively, have Steps declare a `Gpus` parameter. Either change also strengthens the
sample as a demonstration. If the sample deliberately supports only Conda end to end, say so in the job
template's header instead of claiming (L15–L17) that "on a Docker queue the Docker entries apply".

### R9. Fix the job sample's comment about Session sharing

Job sample L19–L21 says "the scheduler never puts two of these Steps in the same Session". DecodeFrames and
EncodeVideo supply identical values, and L259–L261 says a scheduler may reuse a host for exactly that reason.
Under the HJAR L27 rule as amended, those two Steps may share a Session. Reword L19–L21 to say that Steps with
different values never share a Session.

### R10. Add Security Considerations and stop interpolating Environment parameters into shell source

Environment parameter values are written by the Job Template author and executed by Environment scripts that
the queue administrator owns. The RFC should state the trust model. Environment parameters carry the same
trust as Job Parameters, which the submitter already controls, so no new privilege is granted. Environment
Template authors must still treat `Env.Param.*` as untrusted input in their scripts.

The samples show the unsafe form. The RFC's Basic Example 1 (L66) and `conda-env-params.yaml` (`PACKAGES='{{Env.Param.Packages}}'`,
`CHANNELS='{{Env.Param.Channels}}'`) substitute the value into bash source inside single quotes. A value
containing `'` breaks the script or runs arbitrary code. Pass values as `args` (as `docker-env-params.yaml`
already does for `Image`) or through `variables`, or use `repr_sh` with `EXPR`. RFC 0008 has a Security
Considerations section with the same concern for wrapped commands. Mirror it here.

## 7. Suggestions

* **S1. Template processing stages table.** Schemas L2217–L2219: add `Env.Param.*` to the Unknown Values of
  Template validation, and list "Environment parameter effective values" among the Known Values of Job
  creation. Today `Env.Param.*` first appears only in the Task execution row, although the values are fixed at
  job creation.
* **S2. Say which Sessions enter attached Environments.** Constraint 8 over the combined Job (RFC L506–L510,
  Schemas L1033–L1034) depends on the fact that every Session of the Job enters every attached Environment
  (absent SERVICE `runScope`). State it once; the consequence then follows plainly: a parameter without a
  default in a queue Environment breaks every Job Template that does not list it.
* **S3. `required` on a `jobEnvironments` entry.** RFC L447–L449 says the field "has no effect" there.
  `required: false` on an Environment that is always present is almost certainly an author mistake. Make it a
  validation error, or recommend that validators warn.
* **S4. Define equality for Session grouping.** Specify that effective values compare as typed resolved values
  before path mapping. Otherwise two hosts with different mapping rules could disagree, and `1` versus `"1"`
  for an INT is ambiguous without the type.
* **S5. State what `environment:` entries do not affect.** Add one sentence each: they do not change the order
  in which Environments are entered, they do not cause an Environment to be entered that would not otherwise be,
  and (with SERVICE) they are not edges of the dependency graph.
* **S6. Both Conda and Docker attached.** The job sample assumes a queue attaches one of the two. If both are
  attached, the Conda Environment's `onEnter` may run inside the ffmpeg image, which has no `conda`. Add a note
  to the sample header, and consider a sentence in HJAR §Environment Parameters that the entries are
  independent: each applies if its Environment is attached.
* **S7. Repetition across Steps.** Eight near-identical entries for four Steps is the main writability cost.
  The Future Work item "A Job-level default for a whole `environment:` entry" (RFC L857–L860) would address it.
  Cite the sample as motivation there, and mention that YAML anchors are a valid stopgap today.
* **S8. Prior Art refinements.** Snakemake 8 replaced `--use-conda`/`--use-singularity` with
  `--software-deployment-method` (`--sdm`); use the current form or note both. Add the in-ecosystem baseline:
  Deadline Cloud's sample Conda queue environment (`CondaPackages`, `CondaChannels`) is what this RFC
  generalizes. Consider Airflow's per-task `executor_config`/`pod_override` (a per-task image on a shared
  executor) and Rez (`rez env` per job). Each is closer to "a platform-owned provisioning layer configured per
  unit of work" than Argo is.
* **S9. Sample header commands.** Job sample L23–L25 suggests `openjd summary video-sfm-composite.yaml`.
  No released tool accepts `ENV_PARAMS`, so this fails today. Say "once ENV_PARAMS is implemented".
* **S10. Unreachable same-named Environments.** Constraint 3 (RFC L473–L475) means a Step whose Step
  Environment shares a name with a queue Environment cannot address the queue one. Add this as a fifth item to
  §3.2.1, alongside item 4 for `jobEnvironments`.
* **S11. Basic Example 3.** RFC L127–L130 declares `SceneFile` but never uses it. Use it in the Step scripts or
  drop it.
* **S12. Naming conventions.** Since Open Question 1 decides real-world portability, include a short
  non-normative "Recommended names" subsection in the RFC: `Conda`/`Packages`/`Channels`, `Docker`/`Image`,
  each defaulting to `CondaPackages`/`CondaChannels`/`DockerImage`. A wiki conventions page can then be lifted
  directly from it on acceptance.
* **S13. Unsourced claim.** RFC L197–L198 ("COLMAP and Maya pin incompatible versions of shared libraries in
  practice") would be stronger with an example or softened to "can".

## 8. Questions

* **Q1 (could change the specification).** When a Session serves several Steps (R1), may a scheduler start a
  Session for Step A and later run Step B's Tasks in it, provided B's effective values match A's? Or must the
  set of Steps be known when the Session starts? The answer determines whether "a Session's values are fixed by
  the Steps it serves" is a scheduling rule or just a consequence of equality.
* **Q2 (non-blocking).** With SERVICE, a Job Environment such as `RenderLicense` (no `runScope`, required
  `Product`) is entered by Service Sessions too. Every Service must then list it with a `Product`, which is
  likely not what the author wants. Should the RFC recommend `runScope: [TASK]` for Environments with required
  parameters, or is the error the intended nudge?
* **Q3 (non-blocking).** Is `Step.Name` in `parameters` values (RFC L461) intended to be common? It makes every
  Step's values distinct and so disables cross-Step Session sharing. If it exists mainly for logging, the
  rationale could say so.

VERDICT: NEEDS_CHANGES
