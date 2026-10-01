# SERVICE Conformance Tests

Conformance tests for the `SERVICE` extension defined in
[RFC 0009 — Services](../../../rfcs/0009-service.md).

## What this extension does

`SERVICE` adds a new entity, the `<Service>` (Template Schemas §9): a long-lived
process that a scheduler starts before scheduling the Tasks in its scope, keeps
running for the lifetime of its scope, and stops once the scope no longer needs
it. A Service publishes one or more named TCP ports whose endpoint the entities
in its scope read through the `Service.*` format-string scope.

```yaml
# Job Template (§1.1) / Environment Template (§1.2)
jobServices: [ <Service>, ... ]      # NEW — Job scope
services:    [ <Service>, ... ]      # NEW — external Services, Job scope
# <StepTemplate> (§3)
stepServices: [ <Service>, ... ]     # NEW — Step scope
# <Environment> (§4)
runScope: [ TASK | SERVICE, ... ]    # NEW — which kinds of Session enter it
# <EnvironmentActions> (§4.3), with WRAP_ACTIONS
onWrapServiceEnter / onWrapServiceRun / onWrapServiceReadinessCheck / onWrapServiceExit
```

```yaml
<Service> ::= the object:
  name: <ServiceName>                         # an <Identifier>, not `File`
  description: <Description>                  # optional
  let: <LetBindings>                          # optional (job creation)
  hostRequirements: <HostRequirements>        # optional
  ports: [ <ServicePort>, ... ]               # 1–10, unique names
  readinessCheck: <ServiceReadinessCheck>     # TCP_CONNECT (default) | COMMAND | STDOUT
  restartPolicy: <ServiceRestartPolicy>       # maxAttempts (0), completedTasks (RERUN | KEEP)
  variables: <EnvironmentVariables>           # optional
  script:
    let: <LetBindings>                        # optional (service execution)
    actions: { onEnter?, onRun, onReadinessCheck?, onExit? }
    embeddedFiles: [ <EmbeddedFile>, ... ]    # optional, Service.File.<name>
```

| Value | Type | In scope |
|---|---|---|
| `Service.<name>.<port>.port` | `int` | the declaring Service; every entity in its scope |
| `Service.<name>.<port>.connectAddress` | `string` | the declaring Service; every entity in its scope |
| `Service.<name>.<port>.bindAddress` | `string` | the declaring Service **only** |
| `Service.File.<name>` | `path` | the declaring Service's script |
| `WrappedService.Name` / `.PortNames` / `.Ports` / `.BindAddresses` | `string` / `list[string]` / `list[int]` / `list[string]` | the four `onWrapService*` hooks |

## RFC rules these tests verify

- **Extension gating**: `jobServices`, `stepServices`, `services`, and `runScope`
  require `SERVICE`; the `onWrapService*` hooks require both `WRAP_ACTIONS` and
  `SERVICE`; `SERVICE` requires `EXPR` (§9.7 item 7).
- **List shape and names** (§1.1 item 8, §1.2 item 6, §3 item 6, §9 item 5,
  §9.1, §9.2): 1–10 elements, unique names, no Job/Step Service name collision
  (different Steps may reuse a name), identifiers that are not `File`.
- **Environment Template root** (§1.2): `$schema` and `extensions` accepted,
  `environment` optional, at least one of `environment` or `services`.
- **`runScope`** (§4 item 3): recognized names only, no duplicates, not empty;
  an Environment whose `runScope` includes `SERVICE` (including the default)
  references no `Service.*` value.
- **Hooks follow `runScope`** (§4.3 constraint 6): a wrapping Environment
  defines `onWrapEnvEnter`/`onWrapEnvExit` always, `onWrapTaskRun` iff `TASK`,
  and all four `onWrapService*` hooks iff `SERVICE`; a hook the `runScope` does
  not call for is rejected.
- **Scope rules** (§7.3.1, §9, §9.7 items 1–2): forward-only references
  between Services, Job Services visible to Step Services and Steps, Step
  Services visible only to their Step, `bindAddress` only in the declaring
  Service, no `Service.*` in `hostRequirements`, `<Service>.let`, a Step's
  `let`, or an action `timeout`; `Task.*` never inside a Service;
  `Service.File.*` only in the declaring Service's script; types `int` /
  `string` / `path`.
- **Readiness and restart** (§9.3, §9.4, §9.7 item 4): `onReadinessCheck`
  defined iff the type is `COMMAND`; every `TCP_CONNECT` port declared; ranges
  of `port`, `timeoutSeconds`, `intervalSeconds`, `maxAttempts`; `@fmtstring`
  numeric fields resolved at job creation in the `<Service>.let` scope with a
  whole-field `null` meaning "not provided".
- **Expression Language §2.2.4**: `join_host_port`, `split_host_port`,
  `is_ipv4`, `is_ipv6` and their signatures.
- **Submission-time checks** (§1.2.2 items 2–3): external-Service name
  collisions, and a wrapping Environment from a document that does not declare
  `SERVICE` placed in scope of a Service.
- **Lifecycle** (*How Jobs Are Run* § Services): READY before any Task,
  referenced Services READY before the referencing Service's Session starts,
  Service Sessions enter the Environments whose `runScope` includes `SERVICE`,
  `onEnter` once per Session, `openjd_env` from `onEnter` reaches `onRun`,
  `openjd_service_ready` honored only from `onRun`, `completedTasks: RERUN` vs
  `KEEP` on an instance failure, `maxAttempts` exhaustion fails the scope, a
  FAILED or failed-scope Service still runs `onExit`, a failing `onExit` does not
  fail the scope, and wrap hooks run in place of the Service's actions with
  `WrappedService.*` populated.

## How these tests are designed

The RFC motivates Services with Valkey, coordinators, and caches, but none of
those is needed to *test* the mechanism. The execution tests use small python
TCP listeners (`socket`, `http.server`) standing in for a service process,
Tasks that connect to them and print what they received, and sentinel markers
on stdout. Instance-failure tests tell a relaunched `onRun` apart from the
first by a marker file in the Service Session's working directory, which a
relaunch of a READY instance keeps.

Fixtures that carry no `runOn` gate use `python`, the suite's portable
interpreter. The `bash` wrappers (adapted from the `WRAP_ACTIONS` suite) are
gated to `runOn: [posix]`; `service-wrap-service-hooks-python` is the
cross-platform variant.

## Layout

```
SERVICE/
├── README.md                       (this file)
├── job_templates/                  # Job template validation tests
│   │  # §1.1 jobServices, §3 stepServices, extension gating
│   ├── 1.1--job-services-minimal.yaml
│   ├── 1.1--job-services-ten.yaml
│   ├── 1.1--schema-field-accepted.yaml
│   ├── 1.1--job-services-without-service-extension.invalid.yaml
│   ├── 1.1--service-without-expr.invalid.yaml
│   ├── 1.1--service-without-expr-no-services.invalid.yaml
│   ├── 1.1--job-services-empty-list.invalid.yaml
│   ├── 1.1--job-services-more-than-ten.invalid.yaml
│   ├── 1.1--job-services-duplicate-name.invalid.yaml
│   ├── 1.1--job-and-step-service-name-collision.invalid.yaml
│   ├── 3--step-services-minimal.yaml
│   ├── 3--step-services-same-name-in-different-steps.yaml
│   ├── 3--step-services-without-service-extension.invalid.yaml
│   ├── 3--step-services-empty-list.invalid.yaml
│   ├── 3--step-services-more-than-ten.invalid.yaml
│   ├── 3--step-services-duplicate-name.invalid.yaml
│   ├── 3.3.2--attr-worker-preemptible-on-step.yaml
│   ├── 3.3.2--attr-worker-preemptible-on-service.yaml
│   ├── 3.6--step-let-available-in-step-services.yaml
│   │  # §4 runScope
│   ├── 4--run-scope-task.yaml
│   ├── 4--run-scope-service.yaml
│   ├── 4--run-scope-both.yaml
│   ├── 4--run-scope-on-step-environment.yaml
│   ├── 4--run-scope-task-references-service-value.yaml
│   ├── 4--run-scope-without-service-extension.invalid.yaml
│   ├── 4--run-scope-empty.invalid.yaml
│   ├── 4--run-scope-duplicate.invalid.yaml
│   ├── 4--run-scope-unknown-name.invalid.yaml
│   ├── 4--run-scope-case-sensitive.invalid.yaml
│   ├── 4--run-scope-service-references-service-value.invalid.yaml
│   ├── 4--run-scope-default-references-service-value.invalid.yaml
│   │  # §4.3 hooks follow runScope; §4.3.1 WrappedService.*
│   ├── 4.3--wrap-default-run-scope-seven-hooks.yaml
│   ├── 4.3--wrap-run-scope-task-three-hooks.yaml
│   ├── 4.3--wrap-run-scope-service-six-hooks.yaml
│   ├── 4.3--wrap-run-scope-both-seven-hooks.yaml
│   ├── 4.3--non-wrapping-environment-not-subject-to-rule.yaml
│   ├── 4.3--wrap-default-run-scope-missing-service-hooks.invalid.yaml
│   ├── 4.3--wrap-default-run-scope-missing-task-hook.invalid.yaml
│   ├── 4.3--wrap-run-scope-task-with-service-hook.invalid.yaml
│   ├── 4.3--wrap-run-scope-service-with-task-hook.invalid.yaml
│   ├── 4.3--wrap-run-scope-service-missing-one-service-hook.invalid.yaml
│   ├── 4.3--wrap-run-scope-service-missing-env-hooks.invalid.yaml
│   ├── 4.3--service-hooks-without-service-extension.invalid.yaml
│   ├── 4.3--service-hooks-without-wrap-actions.invalid.yaml
│   ├── 4.3.1--wrapped-service-in-service-hooks.yaml
│   ├── 4.3.1--wrapped-service-lists-type-check.yaml
│   ├── 4.3.1--wrapped-action-in-service-hooks.yaml
│   ├── 4.3.1--wrapped-service-ports-not-strings.invalid.yaml
│   ├── 4.3.1--wrapped-service-in-task-hook.invalid.yaml
│   ├── 4.3.1--wrapped-service-in-env-enter-hook.invalid.yaml
│   ├── 4.3.1--wrapped-service-in-own-on-enter.invalid.yaml
│   ├── 4.3.1--wrapped-step-in-service-hook.invalid.yaml
│   ├── 4.3.1--wrapped-env-in-service-hook.invalid.yaml
│   │  # §7.3.1 / §9 scope rules
│   ├── 7.3.1--step-script-sees-job-service.yaml
│   ├── 7.3.1--bind-address-in-own-service.yaml
│   ├── 7.3.1--service-references-earlier-service.yaml
│   ├── 7.3.1--step-service-references-job-service.yaml
│   ├── 7.3.1--step-environment-task-scoped-sees-step-service.yaml
│   ├── 7.3.1--service-let-sees-params-and-job-name.yaml
│   ├── 7.3.1--service-file-in-declaring-service.yaml
│   ├── 7.3.1--service-value-types.yaml
│   ├── 7.3.1--bind-address-in-step-script.invalid.yaml
│   ├── 7.3.1--bind-address-in-later-service.invalid.yaml
│   ├── 7.3.1--bind-address-in-task-scoped-environment.invalid.yaml
│   ├── 7.3.1--service-references-later-service.invalid.yaml
│   ├── 7.3.1--step-services-forward-only.invalid.yaml
│   ├── 7.3.1--job-service-references-step-service.invalid.yaml
│   ├── 7.3.1--step-script-references-other-steps-service.invalid.yaml
│   ├── 7.3.1--job-environment-references-step-service.invalid.yaml
│   ├── 7.3.1--step-environment-default-scope-references-step-service.invalid.yaml
│   ├── 7.3.1--service-in-step-host-requirements.invalid.yaml
│   ├── 7.3.1--service-in-service-host-requirements.invalid.yaml
│   ├── 7.3.1--service-in-service-let.invalid.yaml
│   ├── 7.3.1--session-in-service-let.invalid.yaml
│   ├── 7.3.1--step-let-references-service.invalid.yaml
│   ├── 7.3.1--service-in-step-action-timeout.invalid.yaml
│   ├── 7.3.1--task-values-in-service.invalid.yaml
│   ├── 7.3.1--task-param-in-step-service.invalid.yaml
│   ├── 7.3.1--undeclared-service.invalid.yaml
│   ├── 7.3.1--undeclared-port.invalid.yaml
│   ├── 7.3.1--service-file-of-other-service.invalid.yaml
│   ├── 7.3.1--service-file-in-step-script.invalid.yaml
│   ├── 7.3.1--service-port-plus-string.invalid.yaml
│   ├── expr2.2.4--join-host-port-with-service.yaml
│   ├── expr2.2.4--join-host-port-swapped-arguments.invalid.yaml
│   ├── expr2.2.4--join-host-port-string-port.invalid.yaml
│   ├── expr2.2.4--is-ipv6-on-int.invalid.yaml
│   │  # §9–§9.6 <Service> structure
│   ├── 9--service-all-fields.yaml
│   ├── 9--ports-ten.yaml
│   ├── 9--service-missing-on-run.invalid.yaml
│   ├── 9--service-missing-ports.invalid.yaml
│   ├── 9--ports-empty.invalid.yaml
│   ├── 9--ports-more-than-ten.invalid.yaml
│   ├── 9--ports-duplicate-name.invalid.yaml
│   ├── 9--service-unknown-field.invalid.yaml
│   ├── 9.1--service-name-underscore-identifier.yaml
│   ├── 9.1--service-name-file.invalid.yaml
│   ├── 9.1--service-name-not-identifier.invalid.yaml
│   ├── 9.1--service-name-leading-digit.invalid.yaml
│   ├── 9.2--port-explicit-number.yaml
│   ├── 9.2--numeric-fields-from-param.yaml
│   ├── 9.2--port-name-file.invalid.yaml
│   ├── 9.2--port-name-not-identifier.invalid.yaml
│   ├── 9.2--port-zero.invalid.yaml
│   ├── 9.2--port-above-range.invalid.yaml
│   ├── 9.2--port-not-integer.invalid.yaml
│   ├── 9.2--numeric-field-references-session.invalid.yaml
│   ├── 9.2--numeric-field-references-service.invalid.yaml
│   ├── 9.2--numeric-field-constant-out-of-range.invalid.yaml
│   ├── 9.3--readiness-tcp-connect-ports.yaml
│   ├── 9.3--readiness-tcp-connect-default-ports.yaml
│   ├── 9.3--readiness-command.yaml
│   ├── 9.3--readiness-stdout.yaml
│   ├── 9.3--readiness-tcp-connect-undeclared-port.invalid.yaml
│   ├── 9.3--readiness-tcp-connect-empty-ports.invalid.yaml
│   ├── 9.3--readiness-tcp-connect-with-interval.invalid.yaml
│   ├── 9.3--readiness-command-without-on-readiness-check.invalid.yaml
│   ├── 9.3--readiness-command-interval-zero.invalid.yaml
│   ├── 9.3--readiness-stdout-with-on-readiness-check.invalid.yaml
│   ├── 9.3--readiness-stdout-with-ports.invalid.yaml
│   ├── 9.3--readiness-default-with-on-readiness-check.invalid.yaml
│   ├── 9.3--readiness-timeout-zero.invalid.yaml
│   ├── 9.3--readiness-unknown-type.invalid.yaml
│   ├── 9.3--readiness-missing-type.invalid.yaml
│   ├── 9.4--restart-policy-keep.yaml
│   ├── 9.4--restart-policy-rerun-explicit-defaults.yaml
│   ├── 9.4--max-attempts-negative.invalid.yaml
│   ├── 9.4--max-attempts-not-integer.invalid.yaml
│   ├── 9.4--completed-tasks-unknown.invalid.yaml
│   ├── 9.4--completed-tasks-lowercase.invalid.yaml
│   ├── 9.5--script-let-requires-expr.yaml
│   ├── 9.5--embedded-files-empty.invalid.yaml
│   ├── 9.5--script-let-duplicate-name.invalid.yaml
│   ├── 9.5--service-let-shadows-script-let.invalid.yaml
│   ├── 9.6--service-actions-all-four.yaml
│   ├── 9.6--service-action-empty-command.invalid.yaml
│   └── 9.6--service-actions-unknown-action.invalid.yaml
├── env_templates/                  # Environment template validation tests
│   │  # §1.2 root, §1.2.2 Services from Environment Templates
│   ├── 1.2--services-only.yaml
│   ├── 1.2--environment-only-no-extensions.yaml
│   ├── 1.2--services-and-environment.yaml
│   ├── 1.2--schema-field-accepted.yaml
│   ├── 1.2--schema-field-with-services.yaml
│   ├── 1.2--service-references-earlier-service.yaml
│   ├── 1.2--service-references-job-name.yaml
│   ├── 1.2--neither-environment-nor-services.invalid.yaml
│   ├── 1.2--neither-environment-nor-services-no-extensions.invalid.yaml
│   ├── 1.2--services-without-service-extension.invalid.yaml
│   ├── 1.2--service-without-expr.invalid.yaml
│   ├── 1.2--services-empty-list.invalid.yaml
│   ├── 1.2--services-empty-list-with-environment.invalid.yaml
│   ├── 1.2--services-duplicate-name.invalid.yaml
│   ├── 1.2--services-more-than-ten.invalid.yaml
│   ├── 1.2--service-references-later-service.invalid.yaml
│   ├── 1.2--service-references-step-name.invalid.yaml
│   ├── 1.2--service-in-environment-template-missing-on-run.invalid.yaml
│   ├── 1.2--environment-default-run-scope-references-service.invalid.yaml
│   ├── 1.2--environment-service-run-scope-references-service.invalid.yaml
│   ├── 1.2--environment-references-bind-address.invalid.yaml
│   ├── 1.2--environment-references-undeclared-service.invalid.yaml
│   ├── 1.2--environment-only-references-service.invalid.yaml
│   │  # §4 runScope
│   ├── 4--run-scope-service-environment.yaml
│   ├── 4--run-scope-unknown-name.invalid.yaml
│   ├── 4--run-scope-duplicate.invalid.yaml
│   ├── 4--run-scope-without-service-extension.invalid.yaml
│   │  # §4.3 / §4.3.1 wrap hooks
│   ├── 4.3--wrap-run-scope-service-six-hooks.yaml
│   ├── 4.3--wrap-run-scope-task-three-hooks.yaml
│   ├── 4.3--wrap-default-run-scope-seven-hooks.yaml
│   ├── 4.3--rfc0008-three-hooks-without-service-unchanged.yaml
│   ├── 4.3--services-and-wrapping-environment-compose.yaml
│   ├── 4.3--wrap-default-run-scope-missing-service-hooks.invalid.yaml
│   ├── 4.3--wrap-run-scope-service-with-task-hook.invalid.yaml
│   ├── 4.3--wrap-run-scope-task-with-service-hooks.invalid.yaml
│   ├── 4.3--wrap-run-scope-service-missing-service-exit.invalid.yaml
│   ├── 4.3--service-hooks-without-service-extension.invalid.yaml
│   ├── 4.3--service-hooks-without-wrap-actions.invalid.yaml
│   ├── 4.3.1--wrapped-service-in-service-hooks.yaml
│   ├── 4.3.1--wrapped-service-in-service-hook-timing-fields.yaml
│   ├── 4.3.1--wrapped-service-in-env-exit-hook.invalid.yaml
│   ├── 4.3.1--wrapped-service-in-embedded-file.invalid.yaml
│   └── 4.3.1--wrapped-step-in-service-hook-timeout.invalid.yaml
└── jobs/                           # End-to-end execution tests
    │  # Readiness checks and the Service.* scope
    ├── service-job-tcp-connect.test.yaml
    ├── service-step-stdout-readiness.test.yaml
    ├── service-command-readiness.test.yaml
    ├── service-ready-message-only-from-on-run.test.yaml
    ├── service-reference-chain.test.yaml
    ├── service-step-services-per-step.test.yaml
    ├── service-file-embedded.test.yaml
    ├── service-join-host-port-url.test.yaml
    ├── service-let-bindings-resolve.test.yaml
    ├── service-numeric-fields-from-param.test.yaml
    │  # The Service Session
    ├── service-session-is-its-own-session.test.yaml
    ├── service-environments-follow-run-scope.test.yaml
    ├── service-on-enter-openjd-env-reaches-on-run.test.yaml
    ├── service-on-exit-runs-after-task-failure.test.yaml
    ├── service-on-exit-failure-does-not-fail-scope.test.yaml
    │  # External Services (§1.2.2)
    ├── service-external-task-scoped-client-environment.test.yaml
    ├── service-external-parameter-override.test.yaml
    ├── service-external-services-only-attachment.test.yaml
    │  # Failure and restart
    ├── service-rerun-relaunch.test.yaml
    ├── service-keep-relaunch.test.yaml
    ├── service-max-attempts-exhausted-task-never-runs.test.yaml
    │  # WRAP_ACTIONS composition
    ├── service-wrap-service-scoped-hooks.test.yaml
    ├── service-wrap-env-hooks-wrap-inner-environment-in-service-session.test.yaml
    ├── service-wrap-service-hooks-python.test.yaml
    ├── service-wrapper-task-scoped-with-service.test.yaml
    │  # Runs the runner must reject
    ├── service-max-attempts-exhausted.invalid.test.yaml
    ├── service-without-extension.invalid.test.yaml
    ├── service-without-expr.invalid.test.yaml
    ├── service-port-from-param-out-of-range.invalid.test.yaml
    ├── service-port-from-string-param-not-integer.invalid.test.yaml
    ├── service-max-attempts-from-param-negative.invalid.test.yaml
    ├── service-external-name-collision.invalid.test.yaml
    ├── service-external-name-collision-step-service.invalid.test.yaml
    ├── service-external-two-attachments-collision.invalid.test.yaml
    ├── service-wrapper-without-service-declaration.invalid.test.yaml
    └── service-job-wrapper-with-external-service.invalid.test.yaml
```

## Semantics tested by the execution fixtures

- **Readiness gating** (*How Jobs Are Run* constraint 3): no Task runs before
  every Job Service and Step Service of its Step is READY, under each of the
  three checks — `TCP_CONNECT` (`service-job-tcp-connect`), `STDOUT`
  (`service-step-stdout-readiness`), and `COMMAND` with `onReadinessCheck`
  running concurrently with `onRun` (`service-command-readiness`).
  `openjd_service_ready` is honored only from `onRun`
  (`service-ready-message-only-from-on-run`).
- **Start ordering** (constraint 2): a Service that references another's
  endpoint starts only once the referenced Service is READY
  (`service-reference-chain`).
- **Step scope** (§3 item 6): Step Services of different Steps with the same
  name are distinct Services, each stopped when its Step completes
  (`service-step-services-per-step`).
- **`Service.File.*` and `let`**: embedded files materialize into the Service
  Session and resolve through `<ServiceScript>.let`; `<Service>.let` resolves
  `Param.*`, `Job.Name`, `Step.Name`, and the Step's `let`
  (`service-file-embedded`, `service-let-bindings-resolve`).
- **`join_host_port`** composes a URL authority from `connectAddress` and
  `port` (`service-join-host-port-url`).
- **Numeric `@fmtstring` fields** resolve at job creation; a whole-field `null`
  leaves the port to the runtime; out-of-range or non-integer resolutions are
  rejected at job creation (`service-numeric-fields-from-param`,
  `service-port-from-param-out-of-range`,
  `service-port-from-string-param-not-integer`,
  `service-max-attempts-from-param-negative`).
- **A Session of its own**: `Session.WorkingDirectory` is the Service
  Session's, distinct from the Task's; `OPENJD_SESSION_WORKING_DIR` is set
  (`service-session-is-its-own-session`).
- **Environments by `runScope`** ("Services run inside Environments"): the
  Service Session enters the Environments whose `runScope` includes `SERVICE`;
  Environment `variables` reach the Service's actions with the Service's own
  `variables` taking precedence (`service-environments-follow-run-scope`).
- **`openjd_env` within a Service** (§9.6): variables set by `onEnter` reach
  `onRun` and `onExit`, override declarative `variables`, and never propagate to
  Tasks (`service-on-enter-openjd-env-reaches-on-run`).
- **Session end** (constraints 6–7): a Service's Session is ended, and its
  `onExit` run, when the scope fails (`service-on-exit-runs-after-task-failure`,
  `service-max-attempts-exhausted-task-never-runs`); a failing `onExit` does not
  fail the scope (`service-on-exit-failure-does-not-fail-scope`).
- **External Services** (§1.2.2): a services-plus-environment attachment
  publishes an endpoint to a Job Template that knows nothing of `SERVICE`; the
  attachment's parameters are Job Parameters; a services-only attachment is
  started and stopped around a Job that never mentions it
  (`service-external-task-scoped-client-environment`,
  `service-external-parameter-override`,
  `service-external-services-only-attachment`).
- **Failure and restart**: on an instance failure, `RERUN` cancels the running
  Task without counting a failure, relaunches `onRun` in the same Service
  Session, and reruns completed Tasks against the new instance; `KEEP` lets the
  running Task finish and keeps completed results, gating only not-yet-started
  Tasks on the new instance (`service-rerun-relaunch`, `service-keep-relaunch`).
  Exhausting `maxAttempts` makes the Service FAILED and fails its scope before
  any Task runs (`service-max-attempts-exhausted*`).
- **Wrap hooks** (§4.3 constraint 6, §4.3.1): a SERVICE-scoped wrapping
  Environment's four `onWrapService*` hooks run in place of the Service's
  actions with `WrappedAction.*` and the parallel `WrappedService.*` lists; the
  wrapped `onRun`'s `openjd_service_ready` and the readiness check's exit status
  are recognized through the wrap script; `onWrapEnvEnter`/`onWrapEnvExit` wrap
  the inner Environments the Service Session enters; the wrapper is not entered
  for the Task (`service-wrap-service-scoped-hooks`,
  `service-wrap-env-hooks-wrap-inner-environment-in-service-session`,
  `service-wrap-service-hooks-python`). A TASK-scoped wrapper coexists with a
  Service (`service-wrapper-task-scoped-with-service`).
- **Submission-time checks** (§1.2.2 items 2–3): an external Service colliding
  with a Job Service, a Step Service, or another attachment's Service is
  rejected, as is a wrapping Environment from a document without `SERVICE` —
  attached to a Job declaring a Service, or in a Job submitted alongside an
  external Service (`service-external-*collision`,
  `service-wrapper-without-service-declaration`,
  `service-job-wrapper-with-external-service`).

## Running the tests

From the repo root:

```bash
# Run just the SERVICE extension tests
uv run conformance-tests/run_openjd_cli_tests.py 2023-09/SERVICE

# Run only the job execution tests
uv run conformance-tests/run_openjd_cli_tests.py 2023-09/SERVICE/jobs

# Pattern-match a single scenario
uv run conformance-tests/run_openjd_cli_tests.py '*service*relaunch*'
```

## Writing your own runner

See the top-level [conformance README](../../README.md) for the standard
runner contract. The behavior specific to `SERVICE`:

1. When a template declares `extensions: [SERVICE, EXPR]`, the runner must
   accept `jobServices`, `stepServices`, `services`, and `runScope`, and
   validate every `<Service>` per Template Schemas §9.7. Without `SERVICE` it
   must reject those fields; with `SERVICE` but without `EXPR` it must reject
   the template.
2. `openjd run` with `--environment` must merge an Environment Template's
   `services` into the Job as external Services ahead of the Job Template's
   `jobServices`, apply the two submission-time checks of §1.2.2, and treat
   the attachment's `parameterDefinitions` as Job Parameters.
3. Before any Task of a scope runs, every Service of the scope must be
   started in a Service Session of its own — entering the Environments whose
   `runScope` includes `SERVICE`, allocating its ports, running `onEnter`,
   launching `onRun`, and applying the readiness check — and be READY.
   `Service.<name>.<port>.port` and `.connectAddress` must resolve in the
   Task's actions to an endpoint that reaches the process.
4. An `onRun` exit while the scope has work is an instance failure governed by
   `restartPolicy`: relaunch up to `maxAttempts` times, applying
   `completedTasks`; otherwise fail the scope. In every case the Service Session
   must be ended (`onRun` canceled, `onExit` run, Environments exited) when the
   scope completes or fails.
5. Under `WRAP_ACTIONS`, a wrapping Environment whose `runScope` includes
   `SERVICE` must run its `onWrapService*` hooks in place of the Service's
   actions with `WrappedAction.*` and `WrappedService.*` populated, scanning
   the wrap script's stdout for `openjd_*` messages.
