# ENV_PARAMS RFC — Review round 4

Reviewed at `rfc-env-params` d474de3, applying `skills/openjd-rfc-review/SKILL.md`.

Scope:

* `rfcs/0010-environment-parameters.md` ("RFC Lnnn").
* `git diff origin/mainline -- wiki/`: `wiki/2023-09-Template-Schemas.md` ("Schemas Lnnn", post-change line numbers),
  `wiki/How-Jobs-Are-Run.md` ("HJAR"), and `wiki/2026-02-Expression-Language.md` ("ExprLang").
* `samples/v2023-09/environment_templates/conda-env-params.yaml` ("Conda sample"), `.../docker-env-params.yaml`
  ("Docker sample"), `samples/v2023-09/job_templates/video-sfm-composite.yaml` ("job sample"), and
  `samples/README.md`.
* For the composition sections: RFC 0009 at `mwiebe/rfc-service:rfcs/0009-service.md` ("SERVICE Lnnn"), and the
  control flow RFC at `mwiebe/rfc-control-flow:rfcs/0010-control-flow.md` ("CONTROL_FLOW Lnnn"), which this RFC cites
  as RFC 0011.
* `docs/env-params/review-3.md` and `review-3-response.md`, for context only.

The new material since round 3 (`runScope`, Environment `hostRequirements`, `enableIf` on the entry,
`Job.EnvironmentNames`, the rewritten SERVICE composition, the new CONTROL_FLOW composition, the Summary, and the
renumbering) was read with the most care. Its design decisions are treated as fixed; the findings are about how they
are specified.

## Mechanical checks

* **Verbatim sections.** After replacing absolute wiki links with relative ones and normalizing heading levels:
  * RFC §3.2–§3.2.2 (RFC L697–L875) and Schemas §3.2–§3.2.2 (Schemas L945–L1123): identical.
  * RFC §3.3.3 (RFC L596–L629) and Schemas §3.3.3 (Schemas L1288–L1321): identical.
  * RFC §4.5 (RFC L524–L586) and Schemas §4.5 (Schemas L1992–L2054): identical.
  * RFC's quoted How Jobs Are Run section (RFC L1097–L1143), after removing `> `, and HJAR §Environment Parameters
    (HJAR L85–L133): identical.
  * RFC §1.2.1 paragraph (RFC L446–L451) differs from Schemas L140–L145 in "The merge" / "This merge" and in
    "`<Environment>` (`ENV_PARAMS`)" / "[&lt;Environment&gt;](#4-environment) (`@extension ENV_PARAMS`)". Wording only.
  * The diff blocks that are not claimed verbatim were also compared. The §3.6.2 rows, §7.3.1 rows, and ExprLang
    tables match the wiki. Two differences: the RFC's `<Environment>` *hostRequirements* item says "the Step (or, with
    `SERVICE`, the Service)" (RFC L494–L495) where Schemas L1787–L1788 says "the Step"; and the RFC's §7.4 Job-creation
    bullet has an ordering sentence (RFC L1051–L1052) that the Schemas §7.4 row (Schemas L2394) lacks (see R4).
* **Anchors.** Every intra-document anchor in the RFC and every link from the RFC into the three wiki pages resolves.
  The only dangling anchors are in the wiki pages and occur on mainline too: `#29-jobboolparameterdefinition`,
  `#data-string`, `#121-types`, and `2026-02-Expression-Language#expression-evaluation-types`. RFC L642 and L644 link
  `#36-letbindings` from inside a ```diff block that quotes the wiki, where the anchor exists.
* **YAML and bash.** All nine YAML blocks in the RFC and all three samples parse with `yaml.safe_load`. Every bash
  embedded file (RFC Basic Example 1 `Enter`; Conda `Enter`; Docker `Enter` and `Wrap`; the four job sample `Run`
  files) passes `bash -n` after replacing each `{{ }}` with placeholder text. The job sample's `Composite` file is
  Python and was not checked.
* **Renumbering.** No `NNNN` remains in the RFC, wiki, or samples. Every "RFC 0010" means this RFC and every "RFC 0011"
  means control flow. The control flow branch itself is still `rfcs/0010-control-flow.md` with "The number `0010`"
  in its header (CONTROL_FLOW L3); that is outside this PR but must be renumbered before the two are cross-referenced
  publicly.
* **Contradictions searched for.** Statements that every Session enters every Job Environment: R1. "name" where the
  vocabulary is now "request": none found in normative text. Constraint numbering 1–9 is consistent within §3.2, but
  stale lists remain (R6, R8). `§3.3.3` is cited correctly everywhere; there is no stray `§3.3.1`. Schemas `<Environment>`
  items are numbered 1–8 with *script*, *variables*, and the "at least one" rule at 6, 7, and 8; no wiki text refers to
  the old numbers.
* **Validation table.** Checked row by row against the Specification. Gaps and stale rows are in R8.

## 1. Summary assessment

**Not ready; close.** The round-4 additions are well motivated and mostly well specified, and the verbatim copies,
anchors, and samples are clean. But the new fields were not carried through every rule that depends on them. The most
important case is that template validation checks constraint 7, and several SERVICE rules, from `environment:` entries
as written, although whether an entry requests anything now depends on an *enableIf* that is decided only at Job
creation (R2). Constraint 7's own text still says every Session enters every Job Environment (R1). An Environment's
*hostRequirements* inherit the Step's scopes wholesale, including the Step's `let` bindings and `Step.Name` (R3). The
Job-creation ordering says external Environments are resolved before *enableIf* is evaluated, which contradicts §3.2.1
(R4). The composition sections and the Validation table were not updated for *enableIf* and *hostRequirements* in
several places (R6–R8). Each fix is a sentence or a table row; none reopens a design decision.

## 2. Strengths

* **`runScope` reads as one rule everywhere it matters.** §3.2 item 2 (RFC L721–L734), §3.2.1 (RFC L836–L845), §3.2.2
  (RFC L867–L875), HJAR, and SERVICE composition item 7 all derive which Sessions enter an Environment from the same
  two cases, and the Session-sharing condition now names the Environment set explicitly.
* **Combined host requirements are a conjunction with a stated normal form.** §3.3.3 (RFC L596–L629) defines the
  combination as a conjunction first, then gives a normal form for display and conflict detection, and rejects static
  conflicts at Job creation, naming the sources. The rationale (RFC L1417–L1446) explains why neither author can loosen
  the other's requirements.
* **`enableIf` has one meaning a reader can predict.** A disabled entry is as if absent (RFC L722–L725, L742–L748), it
  is decided before name resolution so that "I need `Conda` unless `Docker` is here" is expressible (RFC L1407–L1410),
  and the rationale separates it from `required` cleanly (RFC L1412–L1415).
* **The SERVICE composition now follows from the dependency graph.** Item 8 (RFC L923–L939) replaces a special-case
  default with two rules that fall out of "entering an Environment that lists `X` depends on `X`", including the
  cycle case and its remedy.
* **The job sample shows the queue-choice pattern end to end.** The `&prefer_docker` anchor (job sample L121, L150,
  L192, L309) keeps four identical conditions identical.

## 3. Tenet alignment

* **Portable — satisfied.** `Job.EnvironmentNames` with `enableIf` lets one Job Template prefer whichever provisioning
  Environment a queue attaches. `hostRequirements` on the Environment keeps Steps off hosts where that Environment
  cannot work, without the Job Template knowing the fleet.
* **Expressive — satisfied.** `runScope` has two values that cover both the queue "menu" and the selective Job
  Environment, and per-entity `enableIf` under CONTROL_FLOW covers finer selection.
* **Human readable — satisfied, with R5.** The examples read naturally. The Docker example's `attr.worker.docker`
  suggests a standard capability that does not exist.
* **Human writable — satisfied.** The job sample is long but regular, and YAML anchors carry the repetition.
* **Tooling parseable — partly.** A validator can no longer tell, from the template alone, which Steps request an
  `ONLY_REQUESTED` Environment when entries carry non-literal *enableIf*, and the specification still asks it to (R2).
  Once R2 states what is deferred to Job creation, this is satisfied.

## 4. Specification compatibility

* **Fits existing concepts.** `runScope` and `hostRequirements` sit naturally on `<Environment>`, and the combined
  requirements reuse `<HostRequirements>` unchanged.
* **Orthogonal, with gaps at the seams.** The new fields compose with ENV_PARAMS's own rules, but not yet with three
  pieces of existing text: Schemas §1.1 item 7, which says every Session enters every Job Environment (R1); the scope
  set of a Step's *hostRequirements*, which is broader than an Environment should see (R3); and the §7.4 ordering of
  `let` evaluation relative to attachment (R4).
* **Consistent and whole: not yet.** The SERVICE composition items 1, 6, and 10 and the Validation table predate
  *enableIf* and Environment *hostRequirements* in places (R6, R8).

## 5. Required changes

* **R1. Constraint 7 and Schemas §1.1 still say every Session enters every Job Environment.**
  Constraint 7 (RFC L797–L801, Schemas L1045–L1049) reads "A Step's Sessions enter every *jobEnvironments* entry and
  every attached Environment whose `runScope` is `ALL`, so such a parameter of such an Environment must be supplied by
  every Step of the Job". As written, every *jobEnvironments* entry is entered by every Step, which contradicts
  `runScope: ONLY_REQUESTED` on a Job Environment (Basic Example 3, RFC L139–L190). Change it to "A Step's Sessions
  enter every Environment of the Job, whether a *jobEnvironments* entry or attached, whose `runScope` is `ALL`".
  Separately, Schemas §1.1 item 7 (Schemas L69–L71), unchanged from mainline, says Job Environments "are entered in the
  order provided at the start of every Session for Tasks in the Job". Add "With the `ENV_PARAMS` extension, an
  Environment whose `runScope` is `ONLY_REQUESTED` is entered only in the Sessions of the Steps that request it (see
  [&lt;Environment&gt;](#4-environment))." The RFC's Job Template subsection (RFC L414–L419), which says the schema
  is unchanged except for the extension name, should list this sentence too.

* **R2. Template-validation checks that depend on which entities request an Environment ignore *enableIf*.**
  RFC L805–L809 (Schemas L1053–L1057) checks constraints 5–7 at template validation for entries that name a
  *jobEnvironments* entry, and the Validation row RFC L1193 checks constraint 7 there unconditionally. When an entry's
  *enableIf* is a non-literal expression, its value is known only at Job creation. For an `ONLY_REQUESTED`
  Environment, a Step whose entry omits a required parameter violates constraint 7 only if the entry is enabled. For
  an `ALL` Environment, a Step whose entry supplies it violates constraint 7 only if the entry is disabled. Template
  validation cannot decide either case. The same applies to three rules in the compositions: SERVICE composition item
  8 (RFC L927–L929) makes "a request by that Service" a template validation error, but a disabled request is "as if
  absent"; item 9 (RFC L940–L949) decides at template validation which kinds of Session enter an `ONLY_REQUESTED`
  wrapper; and SERVICE's Service scope is "determined at template validation" (SERVICE L965–L966), with its rule 4
  rejecting unused Services, while item 8 makes rule 3 read "every Step that requests it". Add to the paragraph at RFC
  L805: "A check that depends on whether an entry is enabled is made at template validation only for entries whose
  *enableIf* is absent or a literal; for other entries it is made when the Job is created." Give item 8, item 9, and
  the Service scope note the same qualification, and split the Validation row RFC L1193 accordingly.

* **R3. An Environment's *hostRequirements* see the Step's scopes.**
  `<Environment>` *hostRequirements* item 1 (RFC L497–L501, Schemas L1791–L1795) allows "besides the scopes available
  to a Step's *hostRequirements*, this Environment's `Env.Param.*` and `Env.RawParam.*`". A Step's *hostRequirements*
  can read `Step.Name` and the Step's `let` bindings, and under CONTROL_FLOW `Steps.*` directly and through `let`. An
  Environment, especially one in an Environment Template, cannot know a Step's `let` names, and the CONTROL_FLOW
  composition (RFC L990–L992) separately forbids `Steps.*` here. Replace the phrase with an explicit list:
  "`Param.*` (not PATH or LIST[PATH]) and `RawParam.*` of the Job Parameters of the document that contains the
  Environment, `Job.Name` and `Job.EnvironmentNames` with the `EXPR` extension, and this Environment's `Env.Param.*` and
  `Env.RawParam.*` of types other than PATH and LIST[PATH]". If the intent is to allow `Step.Name`, say so; but the
  Validation row RFC L1181 already implies the narrower set.

* **R4. Job-creation ordering contradicts §3.2.1 and does not cover Step `let`.**
  The RFC's §7.4 bullet (RFC L1046) and the Schemas §7.4 row (Schemas L2394) both say "external Environments are
  resolved and `Job.EnvironmentNames` becomes concrete; each `environment:` entry's *enableIf* is evaluated". In this
  RFC, resolving an external Environment is what §3.2.1 does to an entry, and §3.2.1 says "A disabled entry is not
  resolved". What must come first is the attachment, not the resolution. The Implementation Impact bullet (RFC
  L1664–L1666) has the same conflation ("the external resolution ... must run before *parameters* values are
  resolved"). Separately, `Job.EnvironmentNames` may be read by a Step's `let` (§3.6.2 row, RFC L656), and the same row
  evaluates Step `let` bindings under EXPR before the ENV_PARAMS sentence, so the order the row implies is wrong for
  that case. The RFC's sentence at RFC L1051–L1052 fixes only *enableIf* and *parameters*, and the wiki row lacks even
  that. Write, in both places: "With the `ENV_PARAMS` extension: the Environment Templates attached to the Job are
  determined first, and `Job.EnvironmentNames` is concrete before any TEMPLATE-scope expression of the Job, including
  Step `let` bindings, is evaluated. Then each `environment:` entry's *enableIf* is evaluated; each enabled entry that
  names no *jobEnvironments* entry is resolved (§3.2.1); ...". Use "attach" and "resolve" consistently in
  Implementation Impact.

* **R5. The RFC's Docker example uses an undefined capability in a reserved namespace.**
  Basic Example "An Environment that states what hosts it needs" (RFC L243–L245, L252–L253) requires
  `attr.worker.docker` with `anyOf: ["true"]`. `worker` is a reserved first identifier for capability names (Schemas
  §3.3.2.1 item 3.2), and §3.3.2 defines no `attr.worker.docker`, so the example asks for a capability the
  specification has not defined, in a namespace that custom capabilities may not use. The Docker sample uses
  `attr.worker.os.family: [linux]` instead (Docker sample L87–L89), so the RFC and the sample also disagree. Either use
  the sample's requirement, or a custom name outside the reserved namespace (for example `attr.container.docker`),
  and say in one clause that it is a queue-defined attribute. The "examples validate against the schema" checklist item
  fails until this is fixed.

* **R6. The SERVICE composition was not updated for *enableIf* and Environment *hostRequirements*.**
  * Item 1 (RFC L890–L892) gives the merged constraint as "*required* and *parameters* are permitted only with
    *environment*", dropping *enableIf*, which ENV_PARAMS constraint 1 now includes (RFC L785–L786). Add it.
  * Item 6 (RFC L912–L915) lists the scopes of a Service's *parameters* values and changes SERVICE's `<Service>.let`
    row to add "the *parameters* values of *dependencies* entries". The `<StepTemplate>.let` row it cites as the model
    adds "the *parameters* values and *enableIf*" (RFC L656). Say that a Service's entry may carry *enableIf* with the
    same scopes as its *parameters* values, and add *enableIf* to the row text.
  * Item 10 (RFC L951–L954) amends SERVICE's "the only checks that require the combined Job" to add §3.2.1 resolution,
    constraints 5–7, and the item 9 wrap check. Other checks now also need the combined Job: the §3.3.3 conflicts
    involving an attached Environment's *hostRequirements*, the item 8 acyclicity check through attached Environments
    (the Validation row RFC L1201 already says Job creation), and any *enableIf* or *parameters* value that reads
    `Job.EnvironmentNames`. Add them, or make the sentence non-exhaustive.

* **R7. The CONTROL_FLOW composition leaves two holes.**
  * Item 5 (RFC L986–L989) says an entry's *enableIf* and *parameters* values "may not reference `Steps.*`". Under
    CONTROL_FLOW a Step `let` binding may read `Steps.*`, and both fields may read Step `let` bindings (RFC L742–L746,
    L762–L765). Say "may not reference `Steps.*`, directly or through a `let` binding". Item 6 (RFC L990–L992) needs the
    same words for Environment *hostRequirements*, unless R3 removes `let` from their scope.
  * Item 6 says an Environment's requirements join the Step's host selection materialization group, so the combined
    requirements are known when the Step's are. When the Step's own *hostRequirements* read `Steps.*`, that is at step
    input binding, after submission. §3.3.3 item 1 says a conflict means "the submission MUST be rejected", which cannot
    happen then. State what happens instead, for example: "a conflict found at step input binding fails the Step, naming
    the sources, as a binding error does".

* **R8. The Validation table has stale and missing rows.**
  * RFC L1178: "`ENV_PARAMS` listed by any document that uses Environment *parameterDefinitions*, `runScope`, or
    `environment:` entries" omits *hostRequirements*, which Backward compatibility (RFC L359–L360) includes.
  * No row checks the scopes of a *parameters* value and of *enableIf* (§3.2 format-string bullet and item 5: no
    `Task.*`, `Session.*`, `Env.*`, nor `Param.*` of a PATH type). The `default` row (RFC L1183) has no counterpart for
    values.
  * No row checks an Environment's *hostRequirements* as resolved for each Step: the §3.3.1/§3.3.2 value and name
    constraints after format strings resolve (for example a `min` that resolves to a negative number or a non-number,
    or two names in one source that resolve equal). This is Job creation, per Step.
  * No row for the CONTROL_FLOW rule that entry fields and Environment *hostRequirements* do not reference `Steps.*`
    (composition items 5 and 6).
  * RFC L1193 checks constraint 7 at template validation only; see R2.
  * Outside the table, RFC L1341 still says a validator "checks constraints 1–4 and 8"; §3.2 now says "1–4, 8, and 9"
    (RFC L805, L811).

## 6. Suggestions

* **S1. The Summary still leads with what, not why.** RFC L20 opens "This RFC lets each Step supply different
  parameter values to the Environments it uses", and the problem appears only in the third sentence. That sentence,
  "Today all Steps share the same Environment, configured once from the Job Parameters" (RFC L22–L23), is also
  inaccurate as written, since Step Environments differ by Step. The Summary is also two paragraphs where the template
  asks for one, and it introduces `runScope` and `hostRequirements` without saying what each is for. Suggested
  opening: "A queue's Environment, such as a Conda or Docker Environment, is configured once per Job from Job
  Parameters, so every Session of a multi-application Job installs every application, or the pipeline is split into
  several Jobs. This RFC lets each Step tell such an Environment what it needs." Then the mechanism, with one clause
  each for `runScope` ("so a queue can offer Environments that only the Steps that ask for them enter") and
  `hostRequirements` ("so an Environment can keep Steps off hosts where it cannot run").

* **S2. State three normalization details in §3.3.3.** In Schemas L1298–L1311 (RFC L606–L619):
  1. Names and `anyOf`/`allOf` values compare case-insensitively (Schemas §3.3.1.1 item 4, §3.3.2 items 2–3), so "the
     same capability" means equal names after resolution, ignoring case, and the intersection and union of item 2 and
     item 3 are case-insensitive.
  2. Amounts do not add. A Step with `amount.worker.gpu` `min: 1` that enters the Docker Environment with `Gpus: all`
     reserves one GPU, not two. Readers who know that amounts are "reserved for the Session" may expect a sum. One
     sentence after item 1 settles it, and the job sample's `SolveSfM` (job sample L159–L162 with L158) is exactly this
     case.
  3. The rejection "naming the Step, the Environment, and the capability" assumes one of the two sources is the Step.
     Two Environments can conflict with each other. Say "naming the Step, the conflicting sources, and the capability".

* **S3. "Never make a valid Step invalid except by the conflicts in items 1 and 2" is too strong.** Schemas
  L1314–L1316 (RFC L622–L624). An Environment's requirements resolved for one Step can be invalid on their own (a `min`
  that resolves to a negative number), and they narrow the hosts the Step can use, which on a given fleet may be none.
  Suggest: "so an Environment's requirements are checked on their own, and combining them adds no error other than the
  conflicts in items 1 and 2."

* **S4. The sample and example rely on a `min` of `0`, which the wiki's string form calls invalid.** Both the RFC
  (RFC L248) and the Docker sample (L94) resolve `min` to `0` for Steps without `Gpus`. Schemas L1186–L1189 defines
  `<nonnegativefloatstring>` as "the string representation of a non-zero and positive floating point or integer value".
  The name, `<nonnegativefloat>`, and the `min` defaults say 0 is allowed, and openjd-rs accepts it (`instantiate.rs`
  "`min` is `<nonnegativefloat>` ... so `0` is" valid). This is a mainline erratum, but this RFC is the first to depend
  on it. Fix "non-zero and positive" to "non-negative" in this PR's wiki diff, or add a sentence to the example.

* **S5. Basic Example 1's "never taken" is not true.** RFC L86–L90 says that with `ONLY_REQUESTED` "the
  `if [ -z "$PACKAGES" ]` branch above is never taken". A Step that requests `Conda` without supplying `Packages`, in a
  Job whose `CondaPackages` is empty, still takes it. Say "is taken only by a Step that requests `Conda` without any
  packages".

* **S6. Carry SERVICE item 8's exception to `ALL` into the text it modifies.** Item 8 (RFC L927–L928) makes an `ALL`
  Environment that lists Service `X` skipped by `X`'s Sessions. The `runScope` `ALL` definition ("Every Session of the
  Job enters this Environment", RFC L482), constraint 7, and §3.3.3's last sentence ("adds its requirements to every
  Step of the Job") then have an exception. Item 6, which lists the ENV_PARAMS text to reread with "Step or Service",
  should also say that `ALL` means "every Session except those of a Service the Environment lists", and item 7 should
  point to item 8. Also, item 8's acyclicity rule names only "the graph of SERVICE constraint 3", which is the Job
  Template's graph. An Environment Template whose *environment* lists its own Service `X`, and whose other Service `Z`
  enters that Environment, adds `Z` → `X` to the graph of SERVICE §1.2 item 6 constraint 4 (SERVICE L642), which can be
  checked when that Environment Template is validated. Name both graphs.

* **S7. Note the `<Environment>` item renumbering for whichever RFC merges second.** Composition item 5 cites
  "SERVICE `<Environment>` item 3 constraint 2" (RFC L903–L904). SERVICE adds *dependencies* to §4, and this PR's wiki
  has already put *parameterDefinitions*, *runScope*, and *hostRequirements* at items 3–5 (Schemas L1763–L1806).
  Once both merge, the item numbers differ from both RFCs' citations. One sentence in the composition preamble
  (RFC L879–L881) suffices.

* **S8. Say whether `Job.EnvironmentNames` is available in Environment Templates.** The §7.3.1 row says "Available
  wherever `Job.Name` is available" (Schemas L2372), and `Job.Name`'s row says "every Format String in the Job
  Template" (Schemas L2374). ExprLang's new table says "every template can also read" it (ExprLang L869–L873), and
  the §3.6.2 `<EnvironmentScript>.let` row lists it (RFC L658). If Environment Templates may read it, which the
  `<EnvironmentScript>.let` row implies, say so in the §7.3.1 row instead of deferring to `Job.Name`.

* **S9. Editorial.**
  * Terminology, "Effective values" (RFC L405): "the value the Step supplies" → "the value the Step's enabled entry
    supplies".
  * §7.4 template-validation bullet (RFC L1042–L1043): "With `EXPR` it is type-checked" now follows two symbols; say
    "`Env.Param.*` is type-checked", as the Schemas row does.
  * Implementation Impact (RFC L1658–L1660) names the new `Environment` fields as `parameterDefinitions` and `runScope`;
    add `hostRequirements`, the entry's `enableIf`, and the `Job.EnvironmentNames` symbol.
  * The job sample header (L38–L41) and Future Work (RFC L1797) mention only the `&mounts` anchor; the sample also
    uses `&prefer_docker`.

## 7. Questions

* **Q1. Does `Job.EnvironmentNames` reach CONTROL_FLOW's Environment `enableIf`, and does it list disabled
  Environments?** It is "available wherever `Job.Name` is available", and CONTROL_FLOW's Environment *enableIf* may
  read `Job.Name` (CONTROL_FLOW L1510–L1511, RFC L961–L962). If both hold, an Environment's *enableIf* can read a list
  that might or might not include Environments that another *enableIf* disables. Recommended answer: the list names
  every Job Environment of the combined Job whatever its *enableIf* (as it already does whatever its `runScope`), so
  there is no circularity; state that in the §7.3.1 row and in CONTROL_FLOW composition item 3.

* **Q2. Under SERVICE, is a Service's entry for an `ALL` Environment that lists that Service an error?** Item 8 makes
  a request for an `ONLY_REQUESTED` one an error, and says the Service's Sessions skip an `ALL` one. An entry for an
  `ALL` one would supply values that are never used. Recommended answer: the same error, since the entry can have no
  effect and is most likely a mistake.

* **Q3. Do constraint 7 and the §3.3.3 conflicts apply to a Step that CONTROL_FLOW disables statically?** A statically
  `SKIPPED` Step "runs no Tasks and enters no Environments" (CONTROL_FLOW L2148). Rejecting the submission because such a
  Step lacks a required parameter, or has conflicting combined host requirements, would be surprising; not rejecting it
  makes the Job's validity depend on Job Parameter values in a new way. Either answer is defensible; the composition
  section should state one.

* **Q4. Does SERVICE's "only if" half of the wrap hook rule survive item 9?** SERVICE's rule (SERVICE L835–L841) requires
  `onWrapTaskRun` "if and only if" its *runScope* includes `TASK`, and makes an extra hook an error. Item 9 (RFC
  L940–L943) keeps only the "if" half. For an `ALL` wrapper in an Environment Template, the kinds of Session are not
  known until Job creation, so dropping "only if" seems intended. If it is, say that a wrapper may define hooks for a
  kind of Session that does not enter it.

VERDICT: NOT READY (8 required changes, all wording or table edits)
