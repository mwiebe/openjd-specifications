# ENV_PARAMS RFC — Response to review round 3

Responds to `docs/env-params/review-3.md` (verdict READY, reviewed at 07ce3ee). Review 3 had no Required changes. All
six Suggestions were applied as wording changes; nothing was declined, no behavior changed, and no fixed design
decision was reopened. The RFC's embedded §3.2–§3.2.2 and §4.5 were edited identically with the wiki and re-compared
after normalizing absolute links and heading levels; they are still identical.

## Suggestions

* **S1 — applied.** Schemas §7.4 Job-creation row now says "PATH and `LIST[PATH]` Job Parameter defaults", and its
  ENV_PARAMS sentence ends "Environment parameter PATH and LIST[PATH] values are not joined with any directory and must
  be absolute (see §4.5)." The RFC's Template processing stages Job-creation bullet gains the same sentence.
* **S2 — applied.** Schemas §3.2.1 and the RFC's embedded §3.2.1 open "from a Job Template and zero or more Environment
  Templates".
* **S3 — applied.** Schemas §4.5 and the RFC's embedded §4.5, point 1 of "A `default` is checked against the definition
  at two points", now defer the submission-path-format checks (including absoluteness) for PATH and LIST[PATH] to job
  creation, and state that a relative literal `default` is then an error even when every Step supplies the parameter.
  For consistency, the RFC Validation row for PATH and LIST[PATH] values (Job creation) now says it includes a literal
  `default` that no Step uses.
* **S4 — applied.** The RFC's Backward compatibility bullet names Deadline Cloud's sample Rez queue environment
  (`rez_queue_env.yaml`, named `Rez`, taking `RezPackages` as a Job Parameter only) alongside Conda, and the paragraph
  after the Recommended names table names `Rez` alongside `Conda`.
* **S5 — applied.** The RFC Validation row for the `EXPR` restriction (constraint 6) is now checked at "Job creation
  (entries naming external Environments; for *jobEnvironments* entries the rule holds by construction)".
* **S6 — applied.** The Snakemake bullet in the RFC's Prior Art is rewrapped to about 100 columns, as is the edited
  Backward compatibility bullet.
