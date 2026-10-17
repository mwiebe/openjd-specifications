# ENV_PARAMS RFC — Review round 3

Reviewed at `rfc-env-params` 07ce3ee, applying `skills/openjd-rfc-review/SKILL.md`.

Scope:

* `rfcs/NNNN-environment-parameters.md` ("RFC Lnnn").
* `git diff mainline -- wiki/`: `wiki/2023-09-Template-Schemas.md` ("Schemas Lnnn", post-change line numbers),
  `wiki/How-Jobs-Are-Run.md` ("HJAR"), and `wiki/2026-02-Expression-Language.md` ("ExprLang").
* `samples/v2023-09/environment_templates/conda-env-params.yaml`, `.../docker-env-params.yaml`,
  `samples/v2023-09/job_templates/video-sfm-composite.yaml` ("job sample Lnnn"), and `samples/README.md`.
* `docs/env-params/review-2.md` and `docs/env-params/review-2-response.md`.
* For the SERVICE composition: `~/openjd-specifications`, branch `rfc-service`, `wiki/2023-09-Template-Schemas.md`
  §1.2.2 and §3.6.2.
* For prior art and the Recommended names: `aws-deadline/deadline-cloud-samples` mainline,
  `queue_environments/conda_queue_env_improved_caching.yaml` and `queue_environments/rez_queue_env.yaml`.

The design decisions in the epic description are treated as fixed. The whole document was read fresh, not only the
round-2 diff.

Mechanical checks performed:

* The RFC's embedded §3.2–§3.2.2 and §4.5 were compared to the wiki after normalizing absolute links and heading
  levels. Both are identical. The RFC's quoted How Jobs Are Run section is identical to the wiki after removing the
  quote markers. The RFC's §1.2.1 paragraph differs from the wiki only in "The merge" / "This merge".
* Every intra-document anchor in the RFC, and every link from the RFC into the three wiki pages, resolves. The
  dangling anchors in the wiki pages (`#29-jobboolparameterdefinition`, `#data-string`,
  `#expression-evaluation-types`, `#121-types`) all occur on mainline as well.
* Every YAML block in the RFC and all three samples parse. Every bash embedded file passes `bash -n`. The Docker
  `Enter` script passes once `{{ repr_sh(Session.WorkingDirectory) }}` is substituted, which happens before bash sees
  it.
* SERVICE citations added in round 2 were checked against `rfc-service`: the §1.2.2 sentence "Items 2 and 4 are the
  only checks that require the combined Job" and the §3.6.2 `<Service>.let` row ("*hostRequirements*, *variables*,
  *script*") both exist as quoted.
* The prior-art claim about Deadline Cloud's improved-caching Conda sample holds: it passes `--persist-envs-hashed`.
  The Rez sample is named `Rez` and takes `RezPackages`, matching the Recommended names table.

## 1. Summary assessment

**Ready for acceptance**, subject to opening the tracking issue, which the instructions defer.

Both round-2 Required changes were resolved in substance (see §6). The specification now says how every supplied value
and every `default` becomes a typed value, whose Job Parameters each may read, and which document's extensions govern
a value's form. The one behavior that surprises a reader, rejection by a same-named Environment that declares no
parameters, is stated in every place a reader relies on: the normative text, How Jobs Are Run, Backward compatibility,
Basic Example 2, Recommended names, a rationale subsection, and the job sample header. RFC, wiki, and samples agree.

The suggestions below are wording improvements. None changes behavior, and none blocks acceptance.

## 2. Strengths

* **Typed values are specified once and consistently.** The format-string bullet of `<EnvironmentParameterValue>`
  (RFC L564–L574) and §4.5 item 4 (RFC L398–L406) now use the same conversion sentence and the same no-join rule for
  paths, and the §3.2.2 equality example (`1` and `"1"`) follows from normative text.
* **The extension-governs-form rule is stated, justified, and checkable.** RFC L576–L580 states it with an example,
  constraint 6 makes it normative, the Validation index has a row for it (RFC L858), and the rationale (RFC L1015–L1024)
  explains why reading the Environment's extensions would make template validity depend on the queue.
* **The migration story is honest.** RFC L272–L284 names the case that exists today, gives the remedy, and notes that
  the remedy is itself backward compatible. The rationale (RFC L913–L931) weighs the alternative concretely: run and
  fail inside Tasks after hosts are provisioned.
* **The samples now follow the RFC's own security advice.** Every Step script takes its values as argv (job sample
  L127, L160, L202–L209, L312), as do both Environment Templates.
* **Composition with SERVICE is complete.** Items 6, 7, and 9 (RFC L691–L715) cover the three residual SERVICE
  sentences, and the citations match `rfc-service`.
* **The Expression Language page gained the symbol table** (ExprLang L858), so the type of `Env.Param.<name>` is
  stated where implementers of type checking look.

## 3. Tenet alignment

* **Portable — satisfied.** Paths use the submission path format, values are fixed before path mapping, and the
  same Job Template runs on Conda, Docker, and preinstalled queues. The one portability limit, queues whose same-named
  Environment predates the extension, is stated with its remedy.
* **Expressive — satisfied.** One mechanism serves queue Environments, Job Environments, and (with SERVICE) Services,
  for any parameter type except `RANGE_EXPR`.
* **Human readable — satisfied.** An `environment:` entry reads as "this Step needs Conda with these packages". The job
  sample is long, but each Step's entries are self-explanatory.
* **Human writable — satisfied.** The per-Step repetition is acknowledged in Future Work and mitigated with a YAML
  anchor in the sample.
* **Tooling parseable — satisfied.** Every rule has a stage in the Validation index. RFC L937–L966 states what a
  validator can check without the queue's Environment Templates, and that it should warn when a `required: false`
  entry resolves to nothing.

## 4. Specification compatibility

* **Fits existing concepts.** Values live in `dependencies`, `Env.*` keeps meaning "this Environment", values resolve
  at job creation alongside Step `let`, and parameter definitions reuse `<JobParameterDefinition>` shapes. Every scope
  table (§3 item 3, §3.6.2, §7.3.1, §7.4, ExprLang §1.2.2) is updated.
* **Orthogonal.** The rules for `EXPR`, `WRAP_ACTIONS`, and `SERVICE` are each stated once, and no rule special-cases
  a particular Environment.
* **Consistent and whole.** No contradictions remain between RFC, wiki, and samples. S1 and S2 are places where
  pre-existing wiki wording, read next to the new text, could mislead.

## 5. Checklist

| Item | Status | Notes |
|------|--------|-------|
| **Completeness** | | |
| All template sections present and substantive | ✅ | Plus Terminology, Validation, Security Considerations, Implementation Impact, Future Work. |
| Metadata fields filled correctly | ✅ | Tracking issue and Accepted On are placeholders as instructed. |
| Tracking issue exists and linked | ⚠️ | `(pending)` / `NNNN` by instruction. The user must open the issue and renumber before the PR. Not counted against the verdict, as in rounds 1 and 2. |
| Examples are runnable and demonstrate the feature | ✅ | Runnable once ENV_PARAMS is implemented. The Docker path mounts every path it needs; the console Conda case is documented in the sample header (job sample L22–L28). |
| **Clarity** | | |
| Summary captures the essence in one paragraph | ✅ | |
| Motivation explains WHY | ✅ | |
| Examples succinctly demonstrate the proposal | ✅ | |
| Specification is precise enough to implement | ✅ | S1–S3 are wording refinements. |
| Design rationale explains key decisions | ✅ | |
| **Tenet Alignment** | | |
| Portable | ✅ | |
| Expressive | ✅ | |
| Human readable | ✅ | |
| Human writeable | ✅ | |
| Tooling parseable | ✅ | |
| **Specification Compatibility** | | |
| Proposed language fits naturally | ✅ | |
| High-level ideas fit existing concepts | ✅ | |
| New concepts apply orthogonally | ✅ | |
| Combined work is consistent and whole | ✅ | |
| **Technical Quality** | | |
| Specification diff is syntactically correct | ✅ | Embedded text matches the wiki; diffs are against mainline. |
| Examples validate against proposed schema | ✅ | Checked by hand: every supplied key is declared, every document lists the extensions its types and expressions need, the `Mounts` value is a single expression coercing `list[string]` to LIST[PATH], and every RawParam path is absolute after Job Parameter resolution. |
| Edge cases addressed | ✅ | |
| Backward compatibility considered | ✅ | Per document and for deployed queue Environments. S4 suggests naming Rez as well. |
| Extension naming follows conventions | ✅ | |
| **Process** | | |
| Prior art shows research | ✅ | Claims spot-checked against deadline-cloud-samples. |
| Rejected ideas explain why | ✅ | |
| No unresolved open questions blocking acceptance | ✅ | Both Open Questions are non-blocking. |

## 6. Round-2 Required changes: verification

| Round 2 | Resolved? | Evidence |
|---------|-----------|----------|
| R1 Typed conversion of supplied values; whose Job Parameters; governing extensions | ✅ | Format-string bullet (RFC L564–L574, Schemas §3.2): conversion "as a submitted Job Parameter value of that type would be, except that a relative path is not joined"; scopes are "those defined in the top-level *parameterDefinitions* of the document that contains the entry, with their merged values"; list values must be a list literal or a single `{{ }}` expression. §4.5 item 4 has the matching list rule (RFC L404–L405). New paragraph RFC L576–L580 and constraint 6's last sentence (RFC L596–L597) state that the supplying document's `EXPR` decides. Validation rows RFC L858–L859. Rationale RFC L1015–L1024. |
| R2 Same-named Environment without parameters | ✅ | Rejection kept, as Q1 recommended. §3.2.1 item 1 (RFC L617–L621); HJAR §Environment Parameters ("rejected whatever `required` says"); Backward compatibility bullet naming the console Conda queue environment and the remedy (RFC L272–L284); Basic Example 2 follow-up (RFC L120–L125); Recommended names paragraph (RFC L834–L837); rationale "An Environment without the supplied parameter is an error" (RFC L913–L931); job sample header L22–L28; Validation row RFC L857. |

Round-2 Suggestions S1–S8 were applied as the response describes. Spot-checked: S1 as Composition items 6, 7, 9
(RFC L691–L715); S2 in the §7.3.1 row (RFC L726); S3 at RFC L1009–L1013; S4 in item 3 of "External Environments are
named implicitly"; S5 in every Step's `args`; S6 at ExprLang L858; S7 at RFC L214–L218; S8 in the Summary.

## 7. Required changes

None.

## 8. Suggestions

Each is a wording change that a human can make without re-reading the review. None changes behavior. Where the text is
in Schemas §3.2–§3.2.2 or §4.5, make the same edit in the RFC's embedded copy so the two stay identical.

* **S1. Job-creation row of §7.4 (Schemas L2288).** The row says "PATH and `LIST[PATH]` parameter defaults must be
  relative and are joined with the job template directory", and the new ENV_PARAMS sentence at the end of the same
  row mentions "Environment parameter defaults". A reader can apply the first rule to the second. Change "PATH and
  `LIST[PATH]` parameter defaults" to "PATH and `LIST[PATH]` Job Parameter defaults", and append to the ENV_PARAMS
  sentence: "Environment parameter PATH and LIST[PATH] values are not joined with any directory and must be absolute
  (see §4.5)." The RFC's §7.4 bullets (RFC L738–L743) can gain the same clause.
* **S2. Zero attached Environment Templates (Schemas L1047, RFC L614).** §3.2.1 opens "When a Job is created from a
  Job Template and one or more Environment Templates". Read literally, it does not cover a Job created from a Job
  Template alone, such as `openjd run` with no `--environment`, although §3.2 item 3 ("Whether the Environment must be
  present") implies the answer. Change "one or more" to "zero or more", so that item 2 rejects a `required: true`
  entry that names nothing in that case too.
* **S3. When a literal PATH `default` is checked (Schemas L1922–L1925 and L1934–L1938, RFC L413–L416 and
  L425–L430).** Point 1 of "A `default` is checked against the definition at two points" checks a literal `default`
  at template validation "against the parameter's type and constraints". The PATH paragraph checks absoluteness, and
  the string constraints of path values, when the Job is created, since the submission path format is not known
  earlier. The two read as overlapping for a literal PATH `default`. Add to point 1: "For PATH and LIST[PATH], the
  checks that depend on the submission path format, including absoluteness, are made when the Job is created, as
  below." Also say whether an unused literal PATH `default` that is relative is an error at job creation. The
  consistent answer, given point 1, is yes.
* **S4. Name Rez alongside Conda (RFC L275–L279, L836–L837).** Deadline Cloud's sample Rez queue environment
  (`rez_queue_env.yaml`) is also named `Rez`, takes `RezPackages` as a Job Parameter, and declares no Environment
  parameters, so it rejects Job Templates that follow the `Rez` row of Recommended names in the same way. One clause in
  the Backward compatibility bullet ("as is its sample Rez queue environment, named `Rez`") and in the Recommended
  names paragraph would make the scope of the migration clear.
* **S5. Validation row for the `EXPR` restriction (RFC L858).** "Template validation for *jobEnvironments*" can never
  fire: a Job Environment's parameters are declared in the same document as the entry, so a BOOL or LIST parameter
  there already requires that document to list `EXPR`. Say "Job creation (entries naming external Environments; for
  *jobEnvironments* entries the rule holds by construction)".
* **S6. Prior Art line length (RFC L1159).** The Snakemake line runs to about 140 columns where the rest of the
  section wraps at about 100. Cosmetic.

## 9. Questions

None.

VERDICT: READY
