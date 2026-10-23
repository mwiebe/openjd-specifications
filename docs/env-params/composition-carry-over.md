# ENV_PARAMS composition with SERVICE and CONTROL_FLOW (carry-over notes)

RFC 0010 is specified against mainline, which has neither `SERVICE` (RFC 0009) nor `CONTROL_FLOW` (RFC 0011). It
used to carry two sections describing how it composes with them. Those sections were removed from the RFC: an RFC
accepted later is the one that says how it extends what is already specified. The text is kept here, verbatim as it
was last written (field names updated to `activation` / `envParameterDefinitions`), as input for the RFC 0009 and
RFC 0011 branches.

Since the removal, RFC 0010 also defines `enableIf` on every `<Environment>`, evaluated at Job creation, which RFC
0011 previously defined. In the CONTROL_FLOW section below, items 1-4 and 7 described that feature and are
**superseded** by RFC 0010's main text (Template Schemas §4 *enableIf*), which differs from them in three ways that
RFC 0011 should not re-adopt:

* Item 7 exempted a Step from constraint 7 for an Environment disabled for it. RFC 0010 checks constraint 7 before
  the Environment's *enableIf* is applied, so there is no exemption.
* Item 1 allowed `Env.RawParam.*` of a PATH parameter. RFC 0010 gives *enableIf* exactly the scopes of the
  Environment's *hostRequirements*, which exclude PATH and LIST[PATH] `Env.*` of either kind.
* Item 2 limited an Environment Template's *enableIf* to `Env.*`. RFC 0010 also allows the Environment Template's
  own Job Parameters, as its *hostRequirements* and `default`s do.

RFC 0011 keeps only the extension to step input binding (`Steps.*` in a Step Environment's *enableIf*), item 5's
second half, and items 6 and 8. Section anchors such as `#321-resolving-external-environments` and `#future-work` in
the text below refer to the RFC and the Template Schemas wiki, not to this file.

## Composition with SERVICE

> Applies only when RFC 0009 (`SERVICE`) is also accepted. Whichever RFC is merged into the wiki second applies these
> edits to the other's text, and renumbers the `<Environment>` items that both RFCs add (SERVICE's *dependencies*; this
> RFC's *envParameterDefinitions*, *activation*, and *hostRequirements*) and the citations below that use SERVICE's numbers.
> The wiki on this branch is based on mainline, which has no `SERVICE`, so it does not yet contain them.

RFC 0009 adds `service:` entries to `<StepDependency>`, `dependencies` to `<Service>` and to Job Environments, and a
second kind of Session, the Service Session, in which a Service's actions run inside the Job's Environments. Earlier
drafts of RFC 0009 also defined a `runScope` list on `<Environment>` that selected Sessions by kind (`TASK`,
`SERVICE`); that field is replaced by this RFC's `activation`, and RFC 0009 depends on this RFC for it. With both
extensions:

1. **§3.2 schema.** `<StepDependency>` is one of four objects, `dependsOn:`, its synonym `step:`, `service:`, and
   `environment:`. ENV_PARAMS constraint 1 and SERVICE constraint 4 merge into one: "Exactly one of *dependsOn*, *step*,
   *service*, or *environment* must be present; an entry that gives more than one, or none, is a validation error.
   *required*, *enableIf*, and *parameters* are permitted only with *environment*." SERVICE's rules for *dependsOn*
   entries, including constraint 2 and the dependency graph of constraint 3, apply to *step* entries as to the
   *dependsOn* entries they are synonyms of.
2. **Duplicates.** SERVICE constraint 2 ("No two entries of one *dependencies* list may name the same Step or the same
   Service") and ENV_PARAMS constraint 3 (no two *environment* entries naming the same Environment) stay separate rules
   about separate keys. A Step, a Service, and an Environment may share a name; an entry's key says which it means.
3. **Dependency graph.** *environment* entries are not nodes or edges of the graph of SERVICE constraint 3. They do not
   order anything, so they cannot create a cycle.
4. **`<StepTemplate>` item 4.** SERVICE's text "Each names a Step of the same Job Template or, with the `SERVICE`
   extension, a Service that must be READY" gains "or, with the `ENV_PARAMS` extension, an Environment that this Step
   requests and may supply parameter values to; such an entry does not delay scheduling".
5. **Where *environment* entries are permitted.** In the *dependencies* of a `<StepTemplate>` and of a `<Service>` in a
   Job Template's *services*. Not permitted in:
    * the *dependencies* of an `<Environment>`, which SERVICE limits to `service` entries (SERVICE `<Environment>` item
      3 constraint 2, unchanged); an Environment does not request another Environment.
    * the *dependencies* of a `<Service>` in an Environment Template's *services*, which SERVICE §1.2 item 6 constraint
      5 limits to `service` entries (unchanged). The Environment Template's author writes both the Service and the
      document's *environment*, so, as for a Step Environment, the values can be written directly.
6. **Requesting entities.** Wherever ENV_PARAMS says "Step" for the entity that requests an Environment or supplies
   values (`<Environment>` *activation* and *hostRequirements*, §3.3.3, constraints 5–7, §3.2.1, §3.2.2), read "Step or
   Service", and wherever it says that an `ALWAYS` Environment is entered by every Session, or applies to every Step of the
   Job, read "except the Sessions of a Service that the Environment lists" (item 8). A Service's combined host requirements are its own *hostRequirements*, which RFC 0009 gives it, together
   with those of the Environments its Sessions enter, by the rules of §3.3.3. A Service's *parameters* values and
   *enableIf* may use `RawParam.*`, `Param.*` (not PATH or LIST[PATH]), `Job.Name`, `Job.EnvNames`, and the
   Service's own `let` bindings; `Step.Name` is not available. A Service Session's Environment parameter values are the
   Service's effective values. SERVICE's §3.6.2 row for `<Service>.let` gains "and with `ENV_PARAMS` the *parameters*
   values and *enableIf* of *dependencies* entries" in its Bound Names Available In column, as the `<StepTemplate>.let`
   row does.
7. **Which Sessions enter which Environments.** A Service Session enters the combined Job's *jobEnvironments*
   (including attached Environments) by the same rule as a Task Session: every one with `activation: ALWAYS`, and every one
   with `activation: ON_REQUEST` that the Service requests. Constraint 7 therefore applies to a Service for each
   Environment its Sessions enter, exactly as to a Step. An Environment that only Services should enter is
   `ON_REQUEST` and requested by each Service; an Environment that only Steps should enter is `ON_REQUEST` and
   requested by each Step. An attached Environment from a document that does not declare `ENV_PARAMS` is `ALWAYS`, so
   every Service Session enters it, as every Task Session does. The one exception to "every Service Session" is item
   8: a Service's Sessions do not enter an `ALWAYS` Environment that lists that Service.
8. **Environments that configure Service clients.** Earlier drafts of SERVICE kept an Environment that lists a
   Service in its *dependencies* out of every Service Session, because such an Environment reads the Service's
   endpoint, which does not exist until the Service is READY. That rule is replaced by one that follows from the
   dependency graph, plus one exception:
    * Every entity whose Sessions enter an Environment that lists Service `X` depends on `X`. A Step is in `X`'s scope
      (SERVICE *Service scope* rule 3, which with `ON_REQUEST` reads "every Step that requests it" rather than
      "every Step"), and a Service `Y` gains the edge `Y` → `X` in the graph of SERVICE constraint 3 (or, within an
      Environment Template, of SERVICE §1.2 item 6 constraint 4). Those graphs must still be acyclic. In particular, an
      `environment:` entry by `X` itself for an Environment that lists `X` is the edge `X` → `X`, a cycle, and is
      rejected naming it, whatever the Environment's `activation`: the entry says that `X`'s Sessions enter an
      Environment that waits for `X` to be READY. An explicit entry is never silently ignored.
    * The exception: a Service's Sessions do not enter an `ALWAYS` Environment that lists that Service. Without it, every
      Environment that configures clients of a Service would put that Service in a cycle by default, and the common
      case, a Job Environment that sets a cache's address for every Step, could not keep the default `activation`. The
      exception applies only where no entry is written; it is a default, not an override.

   A cycle whose edges all come from one Job Template is rejected at template validation, except where an edge depends
   on an entry whose *enableIf* is an expression, in which case it is checked when the Job is created (§3.2). A cycle
   through an attached Environment is checked when the Job is created. The remedy for a cycle that is not a mistake is
   to make the Environment `ON_REQUEST`, so that the Service's Sessions do not enter it. A Service may otherwise use
   an Environment that configures clients of another Service, and starts after that Service is READY, exactly as a Step
   would.
9. **Wrap hooks.** SERVICE's rule "the required hooks follow from `activation`" and RFC 0008's rule that a wrapper
   defines all three of its hooks are replaced by one invariant: **every action that runs inside a wrapping
   Environment is wrapped by it.** An action runs inside a wrapping Environment when it runs in a Session that has
   entered the Environment and not yet exited it. Each kind of action has one wrap hook:

   | Action | Wrap hook |
   |--------|-----------|
   | An inner Environment's `onEnter` | `onWrapEnvEnter` |
   | A Task's `onRun` | `onWrapTaskRun` |
   | An inner Environment's `onExit` | `onWrapEnvExit` |
   | A Service's `onEnter` | `onWrapServiceEnter` |
   | A Service's `onRun` | `onWrapServiceRun` |
   | A Service's `onHealthCheck` | `onWrapServiceHealthCheck` |
   | A Service's `onExit` | `onWrapServiceExit` |

   A wrapping Environment that a Session of some kind enters MUST define the hook for every kind of action that kind of
   Session may run inside it: the three Task-Session hooks if any Task Session enters it, the two inner-Environment
   hooks and the four Service hooks if any Service Session enters it. The check is over the actions a kind of Session
   may run, not the actions a particular template defines, so one wrapper is checked once per kind of Session that
   enters it. A hook for a kind of action that never runs inside the wrapper is permitted and wraps nothing, so a
   wrapper may define all seven hooks and be attached to any Job. Which kinds of Session enter the wrapper is known
   from `activation` and, for `ON_REQUEST`, from the entities that request it; the check is made at template
   validation when the wrapper and every requesting entity are in one Job Template and every request's *enableIf* is
   absent or a literal, and at job creation otherwise, naming the Environment and the kind of Session. A validator
   should warn at template validation when an `ALWAYS` wrapper in a Job Template lacks `onWrapTaskRun`, since every Job
   has Task Sessions and the error is certain. The remedy is to define the hooks, or to make the wrapper
   `ON_REQUEST` and have only entities of the kinds it supports request it.

   RFC 0008's "all three" rule is the Task-Session case of this invariant. SERVICE's Environment Template item 4 (a
   wrapper from a document without `SERVICE`, in a Job with Services) is the Service-Session case, and needs no
   separate text. A future RFC that adds a kind of action adds its hook to the table; the invariant does not change.
10. **Checks over the combined Job.** SERVICE §1.2.2 ends "Items 2 and 4 are the only checks that require the combined
    Job". With `ENV_PARAMS` that sentence reads "Items 2 and 4, and the `ENV_PARAMS` checks that need the attached
    Environments: resolving *environment* entries ([§3.2.1](#321-resolving-external-environments)), constraints 5–7 for
    entries that resolve to attached Environments, every check that depends on an *enableIf* or *parameters* value
    that reads `Job.EnvNames`, the §3.3.3 conflicts and the item 8 cycles that involve an attached
    Environment, and the wrap hook check of item 9 for attached wrapping Environments, are the only checks that require
    the combined Job".

## Composition with CONTROL_FLOW

> Applies only when RFC 0011 (`CONTROL_FLOW`) is also accepted. Whichever RFC is merged into the wiki second applies
> these edits to the other's text.

RFC 0011 adds `enableIf` to `<Environment>`: a `bool` expression, evaluated once at Job creation for a Job Environment,
over `Param.*`, `RawParam.*`, and `Job.Name`. If it is `false`, the Environment is disabled for every Session of the
Job. With both extensions:

1. **`enableIf` may read the Environment's own parameters.** In an Environment that declares *envParameterDefinitions*,
   *enableIf* may additionally reference `Env.Param.*` and `Env.RawParam.*` of that Environment (CONTROL_FLOW
   `<Environment>` *enableIf* constraint 3 is relaxed for these names). Such an *enableIf* is evaluated at Job
   creation once per Step (or Service) whose Sessions would otherwise enter the Environment, with that entity's
   effective values (§3.2.2). For each entity for which it is `false`, that entity's Sessions do not enter the Environment, exactly as
   for an `ON_REQUEST` Environment the entity does not request. PATH-typed values are compared unmapped, through
   `Env.RawParam.*`; `Env.Param.*` of a PATH or LIST[PATH] parameter is not available in *enableIf*, since path mapping
   is applied on the Worker Host and *enableIf* is decided before any host is chosen.
2. **Permitted in Environment Templates when it reads only `Env.*`.** CONTROL_FLOW constraint 4 forbids *enableIf* in
   an Environment Template because the template cannot know the Job's parameters. An *enableIf* whose only references
   are `Env.Param.*` and `Env.RawParam.*` of the same Environment reads values the Environment itself declares and
   defaults, so it is permitted there.
3. **Interaction with `activation`.** The two are independent filters. A Session enters an Environment when its
   `activation` admits the Session's entity and *enableIf* is `true` for that entity (or absent). An *enableIf* that
   reads only `Param.*` keeps CONTROL_FLOW's job-wide meaning: `false` disables the Environment for every Session, whatever
   its `activation`. `Job.EnvNames` lists every Job Environment whatever its *enableIf*, so an *enableIf* that reads it,
   directly or through an effective value, sees a list that no *enableIf* changes.
4. **Session sharing.** The set of Environments a Step's Sessions enter, in §3.2.2, excludes Environments disabled for
   that Step by *enableIf*. Two Steps whose Environment sets differ for this reason do not share Sessions, as for any
   other difference in the set.
5. **The two `enableIf`s.** CONTROL_FLOW's *enableIf* on an `<Environment>` and this RFC's *enableIf* on an
   `environment:` entry are the same idea from the two sides: the Environment's author says when it applies, the Step's
   author says when it is wanted. A Step's Sessions enter an Environment only when both agree (and `activation` admits
   the Step). An entry's *enableIf* and *parameters* values may not reference `Steps.*`, directly or through a `let`
   binding that does: a Step's Environment set and effective values are fixed at Job creation, not at step input
   binding. Late-binding them is listed under [Future Work](#future-work).
6. **Host selection.** An Environment's *hostRequirements* read no Step scope (`<Environment>` *hostRequirements* item
   1), so they are resolved for every Step at Job creation. They join the Step's host selection materialization group
   as already-concrete requirements, and the Step's combined host requirements are known when CONTROL_FLOW says its
   host requirements are known. When the Step's own *hostRequirements* read `Steps.*`, or its *enableIf* is dynamic,
   that is at step input binding, after submission; a §3.3.3 conflict found then cannot reject the submission, and
   instead fails the Step, naming the sources, as a binding error does (item 8).
7. **Constraint 7 and an Environment's `enableIf`.** A Step whose Sessions do not enter an Environment, because the
   Environment's *enableIf* is `false` for that Step, need not supply the Environment's required parameters. Since that
   *enableIf* reads effective values, and a required parameter has no `default`, an *enableIf* that reads a required
   parameter can be evaluated for a Step only when the Step supplies it; a Step that does not supply it is rejected
   under constraint 7 before the Environment's *enableIf* is evaluated.
8. **A Step's own `enableIf` comes first.** The checks this RFC makes when the Job is created or later follow
   CONTROL_FLOW's rule that a Step gains no fact until its *enableIf* is decided. For a Step that is `SKIPPED`, none of
   the following is evaluated: its entries' *enableIf*, the resolution of its entries against attached Environments
   (§3.2.1), its effective values, constraint 7 for attached Environments, and its combined host requirements. A
   skipped Step therefore cannot fail Job creation for a missing parameter or a §3.3.3 conflict, and contributes
   nothing to Session grouping or, with `SERVICE`, to a Service's scope. The reason is that the condition that skips a
   Step is often the condition under which its configuration would be invalid: a Step enabled only when a container
   Environment is present may well have host requirements that contradict the attached Environments when it is not.
   For a Step whose *enableIf* is dynamic, those checks are made at step input binding once it is decided `true`, and
   a failure fails the Step, naming the cause, as a binding error does. Template-validation checks are unaffected: they
   have no parameter values to condition on, and apply to every Step as written, exactly as CONTROL_FLOW type-checks
   every expression of a Step it later skips.

A per-entity *enableIf* expresses selection that `activation` cannot, because it depends on the values rather than on
whether an entry exists. A GPU-setup Environment can be entered only by Steps whose `Gpus` value is non-empty, or a
provisioning Environment only by Steps that end up with something to install:

```yaml
environment:
  name: Conda
  activation: ALWAYS
  envParameterDefinitions:
    - name: Packages
      type: STRING
      default: "{{Param.CondaPackages}}"
  enableIf: "Env.Param.Packages != ''"   # entered only by Steps that end up with packages to install
```

Here a Step that supplies no `Packages` and whose Job sets no `CondaPackages` skips the Environment entirely, while a
Job Template written before `ENV_PARAMS` that sets `CondaPackages` still enters it in every Session. `activation:
ON_REQUEST` is the form of this that needs neither `EXPR` nor `CONTROL_FLOW`.

