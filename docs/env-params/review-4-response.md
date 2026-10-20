# ENV_PARAMS RFC — Response to review round 4

Applied at the commit that adds this file. Items not listed here (S1–S3, S5–S9, Q1, Q3, Q4) are deferred to the next
pass; none blocks the edits below.

## Required changes

* **R1** — Constraint 7 now reads "every Environment of the Job, whether a *jobEnvironments* entry or an attached
  Environment, whose `runScope` is `ALL`". Schemas §1.1 item 7 gains the `ONLY_REQUESTED` sentence, and the RFC's Job
  Template subsection lists that edit.
* **R2** — §3.2's checked-at paragraph gains: a check that depends on whether an entry is enabled (constraint 7, and
  any check of which Steps request an Environment) is made at template validation only for entries whose *enableIf* is
  absent or a boolean literal, otherwise at job creation. SERVICE items 8 and 9 and the constraint-7 and SERVICE-graph
  Validation rows carry the same qualification. The "checks constraints 1–4 and 8" sentence in the rationale now says
  "1–4, 8, and 9".
* **R3** — `<Environment>` *hostRequirements* item 1 now lists its scopes explicitly: the document's `RawParam.*` and
  non-PATH `Param.*`, with `EXPR` `Job.Name` and `Job.EnvironmentNames`, and the Environment's own non-PATH
  `Env.Param.*` / `Env.RawParam.*`; not `Step.Name`, a Step's `let`, or `Session.*`. Mirrored in the wiki.
* **R4** — §7.4 (RFC bullet, Schemas row) and Implementation Impact now distinguish attaching Environment Templates
  from resolving entries: attachment comes first and makes `Job.EnvironmentNames` concrete before any TEMPLATE-scope
  expression, including Step `let`; *enableIf* is evaluated next; enabled entries are then resolved (§3.2.1).
* **R5** — The RFC's Docker example now requires `attr.worker.os.family: [linux]`, as the sample does, instead of the
  undefined `attr.worker.docker`.
* **R6** — SERVICE item 1 includes *enableIf* in the merged constraint; item 6 says a Service's *enableIf* shares the
  scopes of its *parameters* values and adds *enableIf* to the `<Service>.let` row text; item 10's list of
  combined-Job checks now includes `Job.EnvironmentNames`-dependent checks, §3.3.3 conflicts and item 8 cycles through
  attached Environments.
* **R7** — CONTROL_FLOW item 5 says "directly or through a `let` binding that does". Item 6 restates that Environment
  *hostRequirements* read no Step scope (per R3) and so are concrete at Job creation, and says a §3.3.3 conflict found
  at step input binding fails the Step as a binding error does.
* **R8** — Validation table: `hostRequirements` added to the gating row; new rows for the scopes of *parameters*
  values / *enableIf* and of Environment *hostRequirements*, for an Environment's resolved *hostRequirements* satisfying
  §3.3.1/§3.3.2 per Step, and for the CONTROL_FLOW no-`Steps.*` rule; the constraint-7 and §3.3.3-conflict rows carry
  the stage qualifications from R2 and R7.

## Suggestions applied

* **S4** — `<nonnegativefloatstring>` in Schemas §3.3.1 now reads "non-negative" instead of "non-zero and positive",
  which was a copy of `<positivefloatstring>`'s definition under the wrong name. The non-string form, the field's
  default, and openjd-rs all accept 0.

## Questions answered

* **Q2** — Resolved differently from the recommendation. An explicit `environment:` entry by Service `X` for an
  Environment that lists `X` is not a pointless entry to be flagged as such; it is the edge `X` → `X` in the SERVICE
  dependency graph, a cycle, and is rejected as one, whatever the Environment's `runScope`. An explicit entry is never
  silently overridden by an implicit rule. The implicit rule survives only as an exception that applies where no entry
  is written: a Service's Sessions do not enter an `ALL` Environment that lists that Service, since otherwise every
  client-configuring Environment would put its Service in a cycle by default. Item 8 is rewritten in those terms and
  names both SERVICE graphs (constraint 3 and §1.2 item 6 constraint 4).

## Second pass (suggestions and questions)

* **S1** — Summary rewritten as one paragraph that opens with the problem (a queue's Environment is configured once
  per Job) and gives `runScope` and `hostRequirements` one purpose clause each. The inaccurate "all Steps share the
  same Environment" is gone.
* **S2** — §3.3.3 now says amounts do not add, that "the same capability" compares names after resolution ignoring
  case (and `anyOf`/`allOf` values likewise), and that a conflict names "the Step, the conflicting sources, and the
  capability", which may be two Environments.
* **S3** — Reworded as suggested: an Environment's requirements are checked on their own, and combining adds no error
  other than the item 1 and 2 conflicts.
* **S5** — "is taken only by a Step that requests `Conda` without any packages".
* **S6** — Item 6 adds the `ALL` exception to its "read as" list; item 7 points to item 8. Both graphs were already
  named under Q2.
* **S7** — Composition preamble notes the `<Environment>` item renumbering for whichever RFC merges second.
* **S8** — The §7.3.1 row states availability directly: every format string of a Job Template or Environment Template
  resolved at Job creation or later, including an Environment's `hostRequirements` and script; not a parameter
  `default` or the Job Template's `name`.
* **S9** — All four editorial items applied.
* **Rename** — `Job.EnvironmentNames` is now `Job.EnvNames`, using `Env` as `Env.File.*` and `Env.Param.*` already do.
  The rationale notes the choice.
* **Q1** — Includes disabled Environments, for the reason the question anticipated: the list feeds Step `let` and entry
  values, which feed effective values, which an Environment's `enableIf` may read. Stated in the §7.3.1 row and
  CONTROL_FLOW item 3.
* **Q3** — Resolved by progressive evaluation rather than by either answer offered. A Step's own *enableIf* is decided
  first, as CONTROL_FLOW already requires for every other fact about a Step; a `SKIPPED` Step has none of this RFC's
  job-creation checks evaluated (entry *enableIf*, §3.2.1 resolution, effective values, constraint 7 for attached
  Environments, §3.3.3 conflicts), because the condition that skips a Step is often the one under which its
  configuration would be invalid. Template-validation checks still apply to every Step as written. New CONTROL_FLOW
  composition item 8; Validation rows qualified.
* **Q4** — "Only if" is dropped deliberately, and the rule is restated as an invariant that generalizes to future
  action kinds: every action that runs inside a wrapping Environment is wrapped by it. A wrapper MUST define the hook
  for every kind of action that a kind of Session entering it may run; a hook for an action that never runs inside it
  is permitted. RFC 0008's "all three" rule and SERVICE's Environment Template item 4 become instances. Item 9
  rewritten with the action-to-hook table.
