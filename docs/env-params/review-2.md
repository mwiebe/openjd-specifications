# ENV_PARAMS RFC — Review round 2

Reviewed at `rfc-env-params` 3803a8d, applying `skills/openjd-rfc-review/SKILL.md`.

Scope:

* `rfcs/NNNN-environment-parameters.md` ("RFC Lnnn").
* `git diff mainline -- wiki/`, covering `wiki/2023-09-Template-Schemas.md` ("Schemas Lnnn", post-change line numbers)
  and `wiki/How-Jobs-Are-Run.md` ("HJAR").
* `samples/v2023-09/environment_templates/conda-env-params.yaml`, `.../docker-env-params.yaml`,
  `samples/v2023-09/job_templates/video-sfm-composite.yaml` ("job sample Lnnn"), and `samples/README.md`.
* `docs/env-params/review-1.md` and `docs/env-params/review-1-response.md`.
* For the SERVICE composition: `~/openjd-specifications`, branch `rfc-service` (71fae98),
  `wiki/2023-09-Template-Schemas.md` §1.2, §1.2.2, §3, §3.2, §3.6.2, §4 ("SERVICE §x").
* For the backward-compatibility case: Deadline Cloud's console Conda queue environment,
  `aws-deadline/deadline-cloud-samples` `queue_environments/conda_queue_env_from_console.yaml` (mainline).

The design decisions in the epic description are treated as fixed. The whole document was read fresh, not only the
round-1 diff.

Mechanical checks performed:

* The RFC's embedded §3.2/§3.2.1/§3.2.2 and §4.5 were compared to the wiki after normalizing absolute links and heading
  levels. Both are identical.
* Every intra-document anchor in the RFC and both wiki pages, and every absolute link from the RFC into the wiki,
  resolves. The only dangling anchors are `#36-letbindings` inside a quoted diff block (not rendered as a link) and two
  that already dangle on mainline (`#29-jobboolparameterdefinition`, `#data-string`).
* All YAML blocks in the RFC and all three samples parse. `set -euo pipefail -f` (Basic Example 1) was run in bash and
  sets `-f` and `pipefail` as intended.
* The SERVICE citations in RFC §Composition with SERVICE were checked against `rfc-service`: §3.2 constraints 2, 3, 4,
  `<Environment>` item 3 constraint 2, §1.2 item 6 constraint 5, the `<StepTemplate>` item 4 quotation, and the
  existence of `<Service>.let` all match.
* `list[string]` → `list[path]` coercion, which the job sample's `Mounts` value relies on, is permitted by RFC 0005
  ("`string` → `path`" and the element-wise `list[T]` → `list[U]` rule).

## 1. Summary assessment

**Close, not yet ready.** All ten round-1 Required changes were resolved in substance, not just acknowledged (see §6).
The RFC, wiki, and samples now agree with one another, the SERVICE composition is specified in one place, and the
Session model is consistent with How Jobs Are Run.

Two Required changes remain, both small. First, the specification never says how a format-string *value* in an
`environment:` entry becomes a typed value, although it says so for a `default`. The equality example in §3.2.2
depends on that rule. Second, the most common Conda queue in the ecosystem today, the Deadline Cloud console Conda
queue environment, is named `Conda` and declares no Environment parameters. Under constraint 5 it rejects the
pipeline sample and every Job Template that follows the Recommended names, yet the RFC and the sample header say the
same Job Template runs on any Conda queue. The behavior is defensible; it must be stated where readers will rely on the
opposite.

## 2. Strengths

* **Round-1 changes were done thoroughly.** The Session rule (Schemas §3.2.2, HJAR §Environment Parameters) is now a
  condition on adding a Step's Tasks to a Session. It answers round-1 Q1 cleanly and keeps today's cross-Step sharing.
* **One source of truth.** The RFC carries §3.2–§3.2.2 and §4.5 verbatim, and a script confirms it. Constraint numbers
  are now the same in both documents.
* **Composition with SERVICE is precise.** RFC §Composition with SERVICE (L624–L669) merges the exactly-one-key rule,
  keeps the duplicate rules separate with a reason, excludes `environment:` from the dependency graph, and closes the
  Environment Template Service question in the direction that leaves SERVICE unchanged. Every citation checks out.
* **PATH semantics are now portable and explicit** (Schemas §4.5, L1915–L1925): submission path format, absolute or
  URI, constraints checked unmapped, plus a recommendation to use `RawParam.*`. Permitting `LIST[PATH]` follows from
  that, and the rationale (RFC §`RANGE_EXPR` excluded, `LIST[PATH]` permitted) explains why `LIST[STRING]` would mount
  the wrong paths.
* **Security Considerations is useful.** The trust model is stated accurately (no new privilege; `allowedValues` as the
  administrator's control), and the safe forms (argv, `variables`, `repr_*`) are concrete. Both Environment samples
  follow them.
* **The Docker sample now demonstrates the feature well.** Per-Step `Image`, `Mounts`, and `Gpus` show the case for
  typed, list-valued Environment parameters better than any prose could.
* **The Validation index** (RFC L765–L790) has a row for every new rule, including the round-1 additions.

## 3. Tenet alignment

* **Portable — mostly satisfied.** Path values are now well defined across operating systems. The remaining gap is
  migration: a Job Template that follows the Recommended names is rejected by a queue that still runs a
  pre-`ENV_PARAMS` Environment of the same name (R2). That is a portability statement the RFC makes and does not yet
  qualify.
* **Expressive — satisfied.** The mechanism is general across Environments, parameters, and Steps, and composes with
  SERVICE and WRAP_ACTIONS without special cases.
* **Human readable — satisfied.** The Docker entries in the job sample read naturally, and the YAML anchor keeps the
  repeated `Mounts` value legible.
* **Human writable — satisfied, with a known cost.** The per-Step repetition is acknowledged in Future Work and
  mitigated with anchors in the sample.
* **Tooling parseable — satisfied.** RFC §External Environments are named implicitly (L844–L868) now states the
  difference from RFC 0009 and what a validator can and cannot check. See S4 for one overclaim in that rationale.

## 4. Specification compatibility

* **Fits existing concepts.** Values live in `dependencies`, `Env.*` keeps meaning "this Environment", and resolution
  happens at job creation alongside Step `let`. The scope tables (§3 item 3, §3.6.2, §7.3.1, §7.4) were all updated.
* **Typed values are underspecified for supplied values** (R1). §4.5 item 4 defines conversion for a `default`; §3.2
  `<EnvironmentParameterValue>` (Schemas L997–L1001) does not, except for a single `{{ }}` expression with `EXPR`.
* **SERVICE composition is consistent**, with three small residual edits to SERVICE text that the composition note does
  not list (S1).
* **Combined work is consistent and whole.** No contradictions remain between RFC, wiki, and samples, apart from the
  claim addressed by R2.

## 5. Checklist

| Item | Status | Notes |
|------|--------|-------|
| **Completeness** | | |
| All template sections present and substantive | ✅ | Plus Terminology, Validation, Security Considerations, Implementation Impact, Future Work. |
| Metadata fields filled correctly | ✅ | Tracking issue and Accepted On are placeholders as instructed. |
| Tracking issue exists and linked | ⚠️ | `(pending)` / `NNNN` by instruction. The user must open the issue before the PR. Not counted against the verdict, as in round 1. |
| Examples are runnable and demonstrate the feature | ⚠️ | Runnable once ENV_PARAMS is implemented, and the Docker path now works. But the job sample is rejected by the console Conda queue environment that most Deadline Cloud queues run, contrary to its header (R2). |
| **Clarity** | | |
| Summary captures the essence in one paragraph | ✅ | |
| Motivation explains WHY | ✅ | One overstatement of the Design Tenets (S7). |
| Examples succinctly demonstrate the proposal | ✅ | |
| Specification is precise enough to implement | ❌ | Value-to-type conversion and the scope of `Param.*` in supplied values (R1). |
| Design rationale explains key decisions | ✅ | Small gaps in S3 and S4. |
| **Tenet Alignment** | | |
| Portable | ⚠️ | Migration from pre-ENV_PARAMS queue Environments (R2). |
| Expressive | ✅ | |
| Human readable | ✅ | |
| Human writeable | ✅ | |
| Tooling parseable | ✅ | |
| **Specification Compatibility** | | |
| Proposed language fits naturally | ✅ | |
| High-level ideas fit existing concepts | ✅ | |
| New concepts apply orthogonally | ✅ | S1 lists residual SERVICE text. |
| Combined work is consistent and whole | ✅ | |
| **Technical Quality** | | |
| Specification diff is syntactically correct | ✅ | Diffs against mainline; embedded text matches the wiki. |
| Examples validate against proposed schema | ✅ | Checked by hand: every `environment:` key is declared, every document lists the extensions its types and expressions need, `Mounts` coerces to `LIST[PATH]`. |
| Edge cases addressed | ⚠️ | R1 (non-`EXPR` and multi-part format-string values), R2 (same-named Environment without parameters). |
| Backward compatibility considered | ⚠️ | Reasoned per document, but the deployed-queue case is not stated where it matters (R2). |
| Extension naming follows conventions | ✅ | |
| **Process** | | |
| Prior art shows research | ✅ | |
| Rejected ideas explain why | ✅ | |
| No unresolved open questions blocking acceptance | ✅ | Both Open Questions are non-blocking. |

## 6. Round-1 Required changes: verification

| Round 1 | Resolved? | Evidence |
|---------|-----------|----------|
| R1 Session premise | ✅ | RFC Summary (L23–L25), §Why the values come from the Step (L235–L243), Terminology rows "Equal values" and "A Session's values", Schemas §3.2.2 last paragraph, §4 `Env.*` scope item (Schemas L1700–L1704), §7.3.1 `Env.Param` row, HJAR L27 and §Environment Parameters, RFC §Implementation Impact (Schedulers). No single-Step claim remains. |
| R2 RFC/wiki identity, SERVICE composition | ✅ | Verbatim match confirmed by script. Mainline diff contains no `service:` lines. RFC §Composition with SERVICE items 1–4 give the merged constraint, duplicate rule, graph exclusion, and `<StepTemplate>` item 4 text. |
| R3 Scope lists | ✅ | Schemas §3 item 3, §3.6.2 rows for `<StepTemplate>.let` and `<EnvironmentScript>.let`, §7.3.1 `Step.Name` row, mirrored in the RFC. |
| R4 Services subsection | ✅ | Composition items 5, 7, 8: `runScope` check stage by entry kind; Environment Template Services may not list `environment:`, with SERVICE §1.2 item 6 constraint 5 unchanged; Service Sessions enter attached Environments with the default `runScope`. |
| R5 Name-resolution rationale | ✅ | RFC L844–L868 states the difference from `externalServices` and why it is acceptable; Schemas §3.2 closing paragraph says what a validator checks. See S4 for one overclaim. |
| R6 PATH format | ✅ | Schemas §4.5 L1915–L1925; RFC rationale "Path values use the submission path format"; Validation-index row. |
| R7 When a `default` is validated | ✅ | Schemas §4.5 "A `default` is checked against the definition at two points"; two Validation-index rows; §7.4 Template validation row. |
| R8 Docker sample runnable | ✅ | `docker-env-params.yaml` declares `Mounts` (LIST[PATH]) and `Gpus`, mounts each path or its parent, adds `--gpus`; the job sample supplies both. Reviewed the script logic: empty `Mounts` and empty `Gpus` both work, nonexistent outputs get their parent created and mounted, duplicates are skipped. |
| R9 Session-sharing comment | ✅ | Job sample L27–L30 and the EncodeVideo comment (L274–L276). |
| R10 Security Considerations | ✅ | RFC §Security Considerations; `conda-env-params.yaml` and Basic Example 1 pass values as argv with `set -f`. See S5 for the Step scripts. |

All round-1 Suggestions S1–S13 were also applied as the response describes (spot-checked: S1 in §7.4, S3 as constraint
8, S4 in §3.2.2, S10 as §3.2.1 item 5, S11, S12 as §Recommended names).

## 7. Required changes

### R1. Specify how a supplied value becomes a typed value, and which Job Parameters it may reference

Schemas §3.2 (L992–L1001, RFC L536–L545) lists two forms of `<EnvironmentParameterValue>`: a literal of the
parameter's type, and a format string resolved at job creation. For a format string it gives a typing rule only for
"a value consisting of a single `{{ }}` expression" with `EXPR`. It does not say what happens to:

* any format-string value without `EXPR`, such as `Seats: "{{Param.SeatCount}}"` for an INT parameter; and
* a multi-part format string with `EXPR`, such as `Seats: "{{Param.Base}}0"`.

Both resolve to a string, and constraint 6 then requires a value "of its type". §4.5 item 4 has the missing rule for
a `default` ("otherwise the resolved string is converted to the parameter's type as a submitted Job Parameter value of
that type would be"). The §3.2.2 equality example ("an INT parameter supplied as `1` by one Step and as `"1"` by
another resolves to the INT 1") assumes the same rule applies to supplied values, but no normative text says so.

Fix, in Schemas §3.2 and the RFC together:

* Add the §4.5 item 4 conversion sentence to the format-string bullet of `<EnvironmentParameterValue>`. For PATH and
  LIST[PATH], keep the §4.5 rule that a relative path is an error, so the conversion does not join it with a
  directory as a submitted value would be.
* State which document's extensions govern the form of a value. An `environment:` entry is part of the supplying
  document, so its `EXPR` (not the Environment's) decides whether list and boolean literals and typed expressions are
  available. Then state the consequence: a Job Template without `EXPR` cannot supply a list-typed parameter such as the
  Docker sample's `Mounts`, and that parameter takes its `default`. (If the other reading is intended, say that
  instead; either way, write it down.)
* "`RawParam.*` for every Job Parameter, `Param.*` for every Job Parameter not of type PATH or LIST[PATH]" should say
  *whose* Job Parameters: those in the supplying template's own top-level `parameterDefinitions`, as §4.5 item 4 says
  for a `default` ("of the same document"). Otherwise a Job Template could appear to reference `Param.CondaPackages`
  defined only by an attached Environment Template, which template validation cannot check.

### R2. State what happens with a same-named Environment that declares no parameters, where readers rely on the opposite

Deadline Cloud's console Conda queue environment (`conda_queue_env_from_console.yaml`), which console onboarding
creates and which most Conda-provisioned queues run today, has `environment.name: Conda`, takes `CondaPackages` and
`CondaChannels` as Job Parameters, and does not list `ENV_PARAMS`. The RFC's Recommended names (L747–L763) and the job
sample use exactly `environment: Conda` with `parameters: {Packages: ...}`. Under constraint 5 ("An Environment that
declares none … rejects one with any") together with §3.2.1 item 1 (the entry resolves, so `required: false` does not
apply), the pipeline sample is **rejected at submission** on every such queue.

The RFC states the rule for this case only in passing (Backward compatibility, L258–L261). The places a reader relies
on say the opposite:

* Job sample L15–L17: "On a Conda queue the Conda entries apply … The Job Template is the same in all three cases."
* RFC Basic Example 2 (L112–L116) and Motivation (L205–L211), which present `required: false` entries as the way one
  Job Template runs on any queue.
* RFC §Recommended names, which recommends the name already in use by Environments without parameters. During
  migration this guarantees a collision rather than a miss.

Rejecting the Job is a defensible outcome. On such a queue the Job would otherwise run with the job-wide
`CondaPackages` (usually empty) and fail inside its Tasks, so rejection fails fast with a clear message. If you keep
the rule:

* Add the case to RFC §Backward compatibility as its own bullet: a Job Template whose entries supply parameters is
  rejected by a queue whose same-named Environment predates `ENV_PARAMS`. The remedy is for the queue administrator to
  add `parameterDefinitions` whose defaults read the existing Job Parameters (as the sample does). That change is
  itself backward compatible for existing Job Templates.
* Qualify the sample header and Basic Example 2: the Conda entries apply on a Conda queue whose Environment declares
  `ENV_PARAMS` parameters, and the Job is rejected, naming the entry, on a Conda queue whose Environment does not.
* Add a sentence to §Recommended names saying that an Environment Template adopting a recommended name should declare
  the recommended parameters. A short note in the rationale "`required` defaults to `true`" or "External
  Environments are named implicitly" should explain why rejection, not silent fallback, is the chosen behavior.

If you instead prefer that `required: false` also tolerate an Environment that lacks the supplied parameters (ignoring
the entry, perhaps with a warning), that changes constraint 5 and §3.2.1. It does not reopen design decision 3, which
covers only the missing-Environment case. It does need a rationale weighing the run-and-fail-in-Tasks outcome above,
and the matching wiki and Validation-index edits.

## 8. Suggestions

* **S1. Residual SERVICE text in the composition note.** Three SERVICE or ENV_PARAMS sentences become inaccurate when
  both extensions are present and are not listed in RFC §Composition with SERVICE:
    * Constraint 7's second sentence ("Every Session enters every *jobEnvironments* entry and every attached
      Environment, so such a parameter must be supplied by every Step of the Job", Schemas L1019–L1020) is false for an
      Environment with `runScope: [SERVICE]`. Item 7 covers the Service side only. Add: "the second sentence of
      constraint 7 reads: every Task Session enters every Environment whose `runScope` includes `TASK`, so …".
    * SERVICE's §3.6.2 row for `<Service>.let` lists "Bound Names Available In: *hostRequirements*, *variables*,
      *script*". Item 6 makes the bindings available in *parameters* values too, so that row needs the same addition
      R3 made for `<StepTemplate>.let`.
    * SERVICE §1.2.2 ends "Items 2 and 4 are the only checks that require the combined Job". ENV_PARAMS adds §3.2.1
      resolution and constraint 7 over the combined Job, so that sentence needs amending.
* **S2. `Env.Param` value row.** Schemas L2253 and RFC L677 say "The same as `Env.RawParam.<ParamName>` except for
  type PATH". §4.5 maps LIST[PATH] too. Say "except for types PATH and LIST[PATH]".
* **S3. Why `Param.*` of PATH type is excluded from values and defaults.** §7.3.1 allows `Param.<PATH>` in any
  Environment format string, so a reader may expect it in an Environment parameter's `default`. One sentence in RFC
  §Values resolved at job creation would help: `Param.<PATH>` is the host-mapped value, which does not exist at job
  creation, and the Environment's own `Env.Param.*` applies the mapping.
* **S4. Rationale overclaim.** RFC L861–L862 ("Every mistake an implicit name allows is caught at job creation") is
  true only for `required: true`. A misspelled name in a `required: false` entry, which is the motivating form, is
  silently ignored on every queue. Say so, and suggest that a validator given the queue's Environment Templates warn
  when a `required: false` entry resolves to nothing.
* **S5. Step scripts in the job sample.** Every Step embeds `'{{Param.WorkDir}}'`, `'{{Param.InputVideo}}'`, etc.
  directly in bash source (job sample L123–L125, L155–L164, L198–L204, L297–L300). That is the pattern RFC §Do not
  substitute values into script source calls unsafe, in a sample shipped with the RFC. The issue is pre-existing
  practice and concerns `Param.*`, not this extension. Still, the template already lists `EXPR`, so passing the paths
  as `args` or using `{{ repr_sh(Param.WorkDir) }}` would make the sample consistent with its own RFC.
* **S6. Expression Language symbol tables.** `wiki/2026-02-Expression-Language.md` §1.2.2 lists every symbol's
  expression type (`Env.File.<name>` is `path`). Consider adding `Env.Param.<name>` and `Env.RawParam.<name>` with the
  type rule from Schemas §4.5. WRAP_ACTIONS did not add `WrappedAction.*` there, so this is optional.
* **S7. Design Tenets paraphrase.** RFC L207–L208 says the Design Tenets "ask that a Job Template run unmodified with a
  Conda, Rez, Docker, or Apptainer Environment supplying its software". The Portable tenet says jobs should be
  "portable to many different systems" and usable "with the technology stack of their choice". Phrase it as an
  application of that tenet rather than a quotation of it.
* **S8. Summary line wrap.** RFC L25 is a single 141-column line where the rest of the paragraph wraps at about 100.
  Cosmetic.

## 9. Questions

* **Q1 (non-blocking).** For R2, do you prefer rejection (current behavior, documented) or tolerance under
  `required: false`? The review recommends documented rejection, since it fails at submission rather than inside
  Tasks. Either choice is consistent with the fixed design decisions.

VERDICT: NEEDS_CHANGES
