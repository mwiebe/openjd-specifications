# SERVICE Conformance Tests

Conformance tests for the `SERVICE` extension defined in
[RFC 0009 — Services](../../../rfcs/0009-service.md).

## What this extension does

`SERVICE` adds a new entity, the `<Service>` (Template Schemas §9): a long-lived
process that a scheduler starts before scheduling the Tasks in its scope, keeps
running for the lifetime of its scope, and stops once the scope no longer needs
it. A Service publishes one or more named ports, each TCP (the default) or UDP,
whose endpoint the entities in its scope read through the `Service.*`
format-string scope.

```yaml
# Job Template (§1.1) / Environment Template (§1.2)
jobServices: [ <Service>, ... ]      # NEW — Job scope
services:    [ <Service>, ... ]      # NEW — external Services, Job scope
# <StepTemplate> (§3)
stepServices: [ <Service>, ... ]     # NEW — Step scope
# <Environment> (§4)
runScope: [ TASK | SERVICE, ... ]    # NEW — which kinds of Session enter it
# <EnvironmentActions> (§4.3), with WRAP_ACTIONS
onWrapServiceEnter / onWrapServiceRun / onWrapServiceHealthCheck / onWrapServiceExit
```

```yaml
<Service> ::= the object:
  name: <ServiceName>                         # an <Identifier>, not `File`
  description: <Description>                  # optional
  let: <LetBindings>                          # optional (job creation)
  hostRequirements: <HostRequirements>        # optional
  ports: [ <ServicePort>, ... ]               # 1–10, unique names; { name, port?, protocol?: TCP | UDP }
  healthCheck: <ServiceHealthCheck>           # TCP_CONNECT (default, TCP ports only) | COMMAND | STDOUT
                                              #   readinessIntervalSeconds, readinessTimeoutSeconds,
                                              #   healthIntervalSeconds, failureThreshold
  restartPolicy: <ServiceRestartPolicy>       # maxAttempts (0), completedTasks (RERUN | KEEP)
  variables: <EnvironmentVariables>           # optional
  script:
    let: <LetBindings>                        # optional (service execution)
    actions: { onEnter?, onRun, onHealthCheck?, onExit? }
    embeddedFiles: [ <EmbeddedFile>, ... ]    # optional, Service.File.<name>
```

| Value | Type | In scope |
|---|---|---|
| `Service.<name>.<port>.port` | `int` | the declaring Service; every entity in its scope |
| `Service.<name>.<port>.connectAddress` | `string` | the declaring Service; every entity in its scope |
| `Service.<name>.<port>.bindAddress` | `string` | the declaring Service **only** |
| `Service.File.<name>` | `path` | the declaring Service's script |
| `WrappedService.Name` / `.PortNames` / `.Ports` / `.BindAddresses` / `.Protocols` | `string` / `list[string]` / `list[int]` / `list[string]` / `list[string]` | the four `onWrapService*` hooks |

## RFC rules these tests verify

- **Extension gating**: `jobServices`, `stepServices`, `services`, `runScope`,
  and the string functions `join_host_port`, `split_host_port`, `is_ipv4`, and
  `is_ipv6` require `SERVICE`; the `onWrapService*` hooks require both
  `WRAP_ACTIONS` and `SERVICE`; `SERVICE` requires `EXPR` (§9.7 item 7).
- **List shape and names** (§1.1 item 8, §1.2 item 6, §3 item 6, §9 item 5,
  §9.1, §9.2): 1–10 elements, unique names, no Job/Step Service name collision
  (different Steps may reuse a name), identifiers that are not `File`. A
  `<Service>` has a closed set of properties; in particular it has no
  `serviceEnvironments` list, and one is rejected as an unknown property.
- **Environment Template root** (§1.2): `$schema` and `extensions` accepted,
  `environment` optional, at least one of `environment` or `services`.
- **`runScope`** (§4 item 3): recognized names only, no duplicates, not empty;
  an Environment whose `runScope` includes `SERVICE` (including the default)
  references no `Service.*` value.
- **Hooks follow `runScope`** (§4.3 constraint 6): a wrapping Environment
  defines `onWrapEnvEnter`/`onWrapEnvExit` always, `onWrapTaskRun` iff `TASK`,
  and all four `onWrapService*` hooks iff `SERVICE`; a hook the `runScope` does
  not call for is rejected. A Service Session's stack (the scope's
  `SERVICE`-scoped Environments) holds at most one wrap layer.
- **Scope rules** (§7.3.1, §9, §9.7 items 1–2): forward-only references
  between Services, Job Services visible to Step Services and Steps, Step
  Services visible only to their Step, `bindAddress` only in the declaring
  Service, no `Service.*` in `hostRequirements`, `<Service>.let`, a Step's
  `let`, or an action `timeout`; `Task.*` never inside a Service;
  `Service.File.*` only in the declaring Service's script; types `int` /
  `string` / `path`.
- **Port protocol** (§9 item 5 constraint 4, §9.2 item 3, §9.7 item 8):
  `protocol` is `TCP` (default) or `UDP`, a case-sensitive literal that is not
  `@fmtstring`; two ports may share a `port` number only when their `protocol`
  differs.
- **Health check and restart** (§9 item 6, §9.3, §9.4, §9.7 item 4):
  `onHealthCheck` defined iff the type is `COMMAND`; every `TCP_CONNECT` port
  declared and TCP, defaulting to every TCP port; a Service none of whose
  ports is TCP must give a `STDOUT` or `COMMAND` check, so omitting
  `healthCheck` or giving `TCP_CONNECT` on it is rejected; a `STDOUT` check
  rejects `readinessIntervalSeconds`, and accepts `failureThreshold` only with
  `healthIntervalSeconds`; `readinessIntervalSeconds` is accepted on
  `TCP_CONNECT` as well as `COMMAND`; the keys `readinessCheck` and
  `timeoutSeconds` are unknown and rejected; ranges of `port`,
  `readinessIntervalSeconds`, `readinessTimeoutSeconds`, `healthIntervalSeconds`,
  `failureThreshold`, `maxAttempts`; `@fmtstring` numeric fields (all four
  health-check fields included) resolved at job creation in the
  `<Service>.let` scope with a whole-field `null` meaning "not provided";
  `completedTasks` is a literal enum, not `@fmtstring`, so a format string
  over a parameter is rejected at validation.
- **Extensions are per document** (§1.2 item 3): an Environment Template's
  `extensions` apply to that document only, so an attachment that declares
  `SERVICE` and `EXPR` may use `services`, `runScope`, and the §2.2.4 host/port
  functions while the Job Template it is applied to declares no extensions.
- **Expression Language §2.2.4**: `join_host_port`, `split_host_port`,
  `is_ipv4`, `is_ipv6` and their signatures.
- **Service names are scoped to their document** (§1.2.2 item 2): an external
  Service may share its name with a Service in the Job Template's `jobServices`
  or any Step's `stepServices`, or in another attached Environment Template; the
  submission is not rejected for it, every `Service.*` reference resolves within
  its own document, and the runner keeps same-named Services from different
  documents distinct.
- **Submission-time check** (§1.2.2 item 3): a wrapping Environment from a
  document that does not declare `SERVICE` placed in scope of a Service — the
  only check that needs the combined Job.
- **Lifecycle** (*How Jobs Are Run* § Services): READY before any Task,
  referenced Services READY before the referencing Service's Session starts
  and stopped after it is stopped, a Job Service's Session spans every Step of
  the Job while a Step Service's ends when its Step completes, Service Sessions
  enter the Environments whose `runScope` includes `SERVICE`, `onEnter` once
  per Session, `openjd_env` and `openjd_redacted_env` from `onEnter` reach
  `onRun`, `onHealthCheck`, and `onExit` (the latter setting the variable
  only with `REDACTED_ENV_VARS`, masked in the log regardless),
  `openjd_service_ready` honored only from `onRun`, `completedTasks: RERUN` vs
  `KEEP` on an instance failure, a ready timeout is an instance failure that
  cancels an in-flight `onHealthCheck`, the health check keeps probing after
  READY and `failureThreshold` consecutive failures (a `COMMAND` exiting
  non-zero, a `TCP_CONNECT` refused, a `STDOUT` heartbeat missed) make the
  instance UNHEALTHY, which is an instance failure taking the ordinary restart
  decision, while fewer failures change nothing and a `STDOUT` check without
  `healthIntervalSeconds` expects no heartbeat, a start failure consumes an
  attempt and relaunches in a new Service Session, `maxAttempts` exhaustion
  fails the scope whether before or after the first Task, a FAILED or
  failed-scope Service still runs `onExit` if any of its actions ran and not
  otherwise, a failing `onExit` does not fail the scope, and wrap hooks run in
  place of the Service's actions with `WrappedService.*` populated.

## How these tests are designed

The RFC motivates Services with Valkey, coordinators, and caches, but none of
those is needed to *test* the mechanism. The execution tests use small python
TCP listeners (`socket`, `http.server`), and one UDP datagram echo, standing in
for a service process,
Tasks that connect to them and print what they received, and sentinel markers
on stdout. Listeners probed by `TCP_CONNECT` or by an `onHealthCheck`
tolerate a connection that sends nothing and may close abortively, replying
only to a client that sent a request. The health-phase fixtures make a probe
start failing, or a heartbeat stop, from a Task or from the service itself
once Task 1 has been answered, and keep Task 1 running long enough for
`failureThreshold` probes at `healthIntervalSeconds: 1` to elapse before
Task 2 could be scheduled. The few fixtures that need state to
survive a Service Session, or to pass between Steps, use a file in the host's
temp directory (`tempfile.gettempdir()`) keyed by `Job.Name`, deleted by the
Service's `onExit` or by the consuming Task, and written so that a stale file
from an aborted run makes the fixture fail rather than pass; this assumes the
Service Sessions and Task Sessions of one run share a host, as the reference
runner's do. Nothing relies on a relaunch reusing the Service Session, which
the RFC leaves to the scheduler.

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
│   ├── 7.3.1--service-port-int-arithmetic-type-checks.yaml
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
│   ├── 7.3.1--service-port-is-int-string-method-type-error.invalid.yaml
│   ├── 7.3.1--service-connect-address-is-string-plus-int-type-error.invalid.yaml
│   ├── expr2.2.4--join-host-port-with-service.yaml
│   ├── expr2.2.4--join-host-port-swapped-arguments.invalid.yaml
│   ├── expr2.2.4--join-host-port-string-port.invalid.yaml
│   ├── expr2.2.4--is-ipv6-on-int.invalid.yaml
│   ├── expr2.2.4--join-host-port-requires-service-extension.invalid.yaml
│   ├── expr2.2.4--split-host-port-requires-service-extension.invalid.yaml
│   ├── expr2.2.4--is-ipv4-requires-service-extension.invalid.yaml
│   ├── expr2.2.4--is-ipv6-requires-service-extension.invalid.yaml
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
│   ├── 9.2--port-arithmetic-on-literals-in-range.yaml
│   ├── 9.2--port-name-file.invalid.yaml
│   ├── 9.2--port-name-not-identifier.invalid.yaml
│   ├── 9.2--port-zero.invalid.yaml
│   ├── 9.2--port-above-range.invalid.yaml
│   ├── 9.2--port-not-integer.invalid.yaml
│   ├── 9.2--numeric-field-references-session.invalid.yaml
│   ├── 9.2--numeric-field-references-service.invalid.yaml
│   ├── 9.2--numeric-field-constant-out-of-range.invalid.yaml
│   ├── 9.2--port-arithmetic-on-literals-out-of-range.invalid.yaml
│   ├── 9.2--port-let-bound-out-of-range.invalid.yaml
│   ├── 9.2--port-protocol-tcp-explicit.yaml
│   ├── 9.2--port-protocol-udp-with-stdout-health-check.yaml
│   ├── 9.2--port-protocol-mixed-default-health-check.yaml
│   ├── 9.2--port-protocol-mixed-tcp-connect-names-tcp-port.yaml
│   ├── 9.2--port-protocol-same-number-tcp-and-udp.yaml
│   ├── 9.2--port-protocol-tcp-connect-names-udp-port.invalid.yaml
│   ├── 9.2--port-protocol-all-udp-default-health-check.invalid.yaml
│   ├── 9.2--port-protocol-all-udp-tcp-connect.invalid.yaml
│   ├── 9.2--port-protocol-same-number-same-protocol.invalid.yaml
│   ├── 9.2--port-protocol-same-number-udp-twice.invalid.yaml
│   ├── 9.2--port-protocol-unknown-value.invalid.yaml
│   ├── 9.2--port-protocol-lowercase.invalid.yaml
│   ├── 9.2--port-protocol-not-string.invalid.yaml
│   ├── 9.3--health-tcp-connect-ports.yaml
│   ├── 9.3--health-tcp-connect-default-ports.yaml
│   ├── 9.3--health-tcp-connect-with-readiness-interval.yaml
│   ├── 9.3--health-command.yaml
│   ├── 9.3--health-stdout.yaml
│   ├── 9.3--health-stdout-heartbeat-interval.yaml
│   ├── 9.3--health-numeric-fields-from-param.yaml
│   ├── 9.3--health-tcp-connect-undeclared-port.invalid.yaml
│   ├── 9.3--health-tcp-connect-empty-ports.invalid.yaml
│   ├── 9.3--health-command-without-on-health-check.invalid.yaml
│   ├── 9.3--health-command-readiness-interval-zero.invalid.yaml
│   ├── 9.3--health-stdout-with-on-health-check.invalid.yaml
│   ├── 9.3--health-stdout-with-ports.invalid.yaml
│   ├── 9.3--health-stdout-readiness-interval.invalid.yaml
│   ├── 9.3--health-stdout-failure-threshold-without-interval.invalid.yaml
│   ├── 9.3--health-default-with-on-health-check.invalid.yaml
│   ├── 9.3--health-readiness-timeout-zero.invalid.yaml
│   ├── 9.3--health-old-readiness-check-key.invalid.yaml
│   ├── 9.3--health-old-timeout-seconds-key.invalid.yaml
│   ├── 9.3--health-unknown-type.invalid.yaml
│   ├── 9.3--health-missing-type.invalid.yaml
│   ├── 9.4--restart-policy-keep.yaml
│   ├── 9.4--restart-policy-rerun-explicit-defaults.yaml
│   ├── 9.4--max-attempts-negative.invalid.yaml
│   ├── 9.4--max-attempts-not-integer.invalid.yaml
│   ├── 9.4--completed-tasks-unknown.invalid.yaml
│   ├── 9.4--completed-tasks-lowercase.invalid.yaml
│   ├── 9.4--completed-tasks-format-string.invalid.yaml
│   ├── 9.5--script-let-requires-expr.yaml
│   ├── 9.5--embedded-files-empty.invalid.yaml
│   ├── 9.5--script-let-duplicate-name.invalid.yaml
│   ├── 9.5--service-let-shadows-script-let.invalid.yaml
│   ├── 9.6--service-actions-all-four.yaml
│   ├── 9.6--service-action-empty-command.invalid.yaml
│   ├── 9.6--service-actions-unknown-action.invalid.yaml
│   └── 9--service-environments-not-a-property.invalid.yaml
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
│   ├── 4.3.1--wrapped-step-in-service-hook-timeout.invalid.yaml
│   │  # Expression Language §2.2.4 host/port functions gated on SERVICE
│   └── expr2.2.4--join-host-port-requires-service-extension.invalid.yaml
└── jobs/                           # End-to-end execution tests
    │  # Health checks before READY and the Service.* scope
    ├── service-job-tcp-connect.test.yaml
    ├── service-step-stdout-health-check.test.yaml
    ├── service-command-health-check.test.yaml
    ├── service-udp-port-echo.test.yaml
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
    ├── service-on-enter-openjd-env-reaches-health-check.test.yaml
    ├── service-on-enter-openjd-redacted-env-with-extension.test.yaml
    ├── service-on-enter-openjd-redacted-env-without-extension.test.yaml
    ├── service-on-exit-runs-after-task-failure.test.yaml
    ├── service-on-exit-failure-does-not-fail-scope.test.yaml
    ├── service-job-service-state-persists-across-steps.test.yaml
    ├── service-step-service-stopped-before-dependent-step.test.yaml
    ├── service-stop-order-referencing-service-stopped-first.test.yaml
    │  # External Services (§1.2.2)
    ├── service-external-task-scoped-client-environment.test.yaml
    ├── service-external-parameter-override.test.yaml
    ├── service-external-services-only-attachment.test.yaml
    ├── service-external-same-name-as-job-service.test.yaml
    ├── service-external-same-name-as-step-service.test.yaml
    ├── service-external-same-name-in-two-attachments.test.yaml
    ├── service-external-same-name-both-consumed.test.yaml
    ├── service-external-client-environment-uses-service-functions.test.yaml
    │  # Failure and restart
    ├── service-rerun-relaunch.test.yaml
    ├── service-keep-relaunch.test.yaml
    ├── service-keep-max-attempts-exhausted-mid-scope.test.yaml
    ├── service-max-attempts-exhausted-task-never-runs.test.yaml
    ├── service-readiness-timeout-fails-scope.test.yaml
    ├── service-start-failure-relaunches-in-new-session.test.yaml
    ├── service-start-failure-no-attempts-on-exit-not-run.test.yaml
    │  # Health checks after READY
    ├── service-health-command-fails-after-ready-relaunches.test.yaml
    ├── service-health-threshold-tolerates-blips.test.yaml
    ├── service-health-tcp-port-closed-while-process-hangs.test.yaml
    ├── service-health-stdout-heartbeat-missed.test.yaml
    ├── service-health-stdout-no-heartbeat-configured.test.yaml
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
    ├── service-wrapper-without-service-declaration.invalid.test.yaml
    └── service-job-wrapper-with-external-service.invalid.test.yaml
```

## Semantics tested by the execution fixtures

- **Readiness gating** (*How Jobs Are Run* constraint 3): no Task runs before
  every Job Service and Step Service of its Step is READY, under each of the
  three checks — `TCP_CONNECT` (`service-job-tcp-connect`), `STDOUT`
  (`service-step-stdout-health-check`), and `COMMAND` with `onHealthCheck`
  running concurrently with `onRun` (`service-command-health-check`).
  `openjd_service_ready` is honored only from `onRun`
  (`service-ready-message-only-from-on-run`). A `protocol: UDP` port is bound
  by a datagram listener that signals readiness through `STDOUT`, and the Task
  reaches it through the same `connectAddress` and `port` values a TCP port
  carries (`service-udp-port-echo`).
- **Start ordering** (constraint 2): a Service that references another's
  endpoint starts only once the referenced Service is READY
  (`service-reference-chain`).
- **Stop ordering** (constraint 4): a Service that references another is
  stopped first, so its `onExit` can still reach the upstream Service
  (`service-stop-order-referencing-service-stopped-first`).
- **Step scope** (§3 item 6, constraints 4 and 6): Step Services of different
  Steps with the same name are distinct Services, each stopped when its Step
  completes (`service-step-services-per-step`); a Step Service is stopped,
  and its `onExit` run, before a dependent Step's Tasks run, which find its
  port refused (`service-step-service-stopped-before-dependent-step`).
- **Job scope** (constraint 6): a Job Service's instance persists across
  Steps, holding state written by one Step for a dependent Step to read, and is
  stopped only when the Job completes, after Steps that never use it
  (`service-job-service-state-persists-across-steps`).
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
  Tasks (`service-on-enter-openjd-env-reaches-on-run`); they reach every
  `onHealthCheck` invocation of a `COMMAND` check too
  (`service-on-enter-openjd-env-reaches-health-check`).
  `openjd_redacted_env` from `onEnter` sets the variable for `onRun` and
  `onExit` only when the document declares `REDACTED_ENV_VARS`, and its value
  is masked in the log either way
  (`service-on-enter-openjd-redacted-env-with-extension`,
  `service-on-enter-openjd-redacted-env-without-extension`).
- **Session end** (constraints 6–7): a Service's Session is ended, and its
  `onExit` run, when the scope fails (`service-on-exit-runs-after-task-failure`,
  `service-max-attempts-exhausted-task-never-runs`); a failing `onExit` does not
  fail the scope (`service-on-exit-failure-does-not-fail-scope`); `onExit` is
  not run when no action of the Service ran, as when a SERVICE-scoped
  Environment's `onEnter` fails with no attempts left
  (`service-start-failure-no-attempts-on-exit-not-run`).
- **External Services** (§1.2.2): a services-plus-environment attachment
  publishes an endpoint to a Job Template that knows nothing of `SERVICE`; the
  attachment's parameters are Job Parameters; a services-only attachment is
  started and stopped around a Job that never mentions it
  (`service-external-task-scoped-client-environment`,
  `service-external-parameter-override`,
  `service-external-services-only-attachment`); the attachment's `extensions`
  gate the §2.2.4 host/port functions for its own `variables` while the Job
  Template declares none
  (`service-external-client-environment-uses-service-functions`).
- **Service names are scoped to their document** (§1.2.2 item 2): an external
  Service named like a Job Service, a Step Service, or another attachment's
  Service is accepted and both start, each resolving its own
  `Service.<name>.*` (`service-external-same-name-as-job-service`,
  `service-external-same-name-as-step-service`,
  `service-external-same-name-in-two-attachments`); with the RFC's Valkey Job
  Template and queue-cache attachment both declaring `Cache`, the Task's
  `Service.Cache.*` reaches the Job Template's Cache while the attached client
  Environment's `VALKEY_HOST` / `VALKEY_PORT` reach the queue's, on different
  ports, and neither answer leaks to the other consumer
  (`service-external-same-name-both-consumed`).
- **Failure and restart**: on an instance failure, `RERUN` cancels the running
  Task without counting a failure, relaunches `onRun`, and reruns completed
  Tasks against the new instance; `KEEP` lets the running Task finish and keeps
  completed results, gating only not-yet-started Tasks on the new instance
  (`service-rerun-relaunch`, `service-keep-relaunch`). Exhausting
  `maxAttempts` makes the Service FAILED and fails its scope, whether before
  any Task runs (`service-max-attempts-exhausted*`) or after some Tasks have
  completed, in which case the remaining Tasks never start and `onExit` runs
  (`service-keep-max-attempts-exhausted-mid-scope`). A ready timeout is an
  instance failure: the in-flight `onHealthCheck` is canceled, no Task runs,
  and `onExit` runs (`service-readiness-timeout-fails-scope`). A start failure
  — a SERVICE-scoped Environment's `onEnter` exiting non-zero — consumes an
  attempt and the relaunch begins a new Service Session with a new working
  directory, re-entering the Environments and re-running `onEnter`
  (`service-start-failure-relaunches-in-new-session`).
- **Health after READY** (§9.3, *How Jobs Are Run* constraint 11): the probe
  that proved readiness keeps running every `healthIntervalSeconds`, and
  `failureThreshold` consecutive failures make the instance UNHEALTHY, an
  instance failure that takes the ordinary restart decision. A `COMMAND` probe
  that starts exiting non-zero after Task 1 gets instance 1 canceled and, under
  `maxAttempts: 1` and `KEEP`, a second instance that serves the remaining Tasks
  (`service-health-command-fails-after-ready-relaunches`); a probe that fails
  twice and recovers under `failureThreshold: 3` changes nothing
  (`service-health-threshold-tolerates-blips`); a `TCP_CONNECT` probe refused
  by a process that closed its listener but is still running makes the
  instance UNHEALTHY, and with no attempts left the Job fails and `onExit` runs
  (`service-health-tcp-port-closed-while-process-hangs`); a `STDOUT` check with
  `healthIntervalSeconds` treats `openjd_service_ready` as a heartbeat and
  fails the instance when it stops (`service-health-stdout-heartbeat-missed`),
  while one without expects no heartbeat and a single ready line serves the
  whole Job (`service-health-stdout-no-heartbeat-configured`).
- **Wrap hooks** (§4.3 constraint 6, §4.3.1): a SERVICE-scoped wrapping
  Environment's four `onWrapService*` hooks run in place of the Service's
  actions with `WrappedAction.*` and the parallel `WrappedService.*` lists; the
  wrapped `onRun`'s `openjd_service_ready` and the health check's exit status
  are recognized through the wrap script; `onWrapEnvEnter`/`onWrapEnvExit` wrap
  the inner Environments the Service Session enters; the wrapper is not entered
  for the Task (`service-wrap-service-scoped-hooks`,
  `service-wrap-env-hooks-wrap-inner-environment-in-service-session`,
  `service-wrap-service-hooks-python`). A TASK-scoped wrapper coexists with a
  Service (`service-wrapper-task-scoped-with-service`).
- **Submission-time check** (§1.2.2 item 3): a wrapping Environment from a
  document without `SERVICE` is rejected when a Service is in its scope —
  attached to a Job declaring a Service, or in a Job submitted alongside an
  external Service (`service-wrapper-without-service-declaration`,
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
   `jobServices`, apply the submission-time wrapper check of §1.2.2 item 3,
   treat the attachment's `parameterDefinitions` as Job Parameters, and keep
   a Service of one document distinct from a same-named Service of another
   (§1.2.2 item 2): every `Service.*` reference — in a Service, an
   Environment, or a Step — resolves to a Service of the document that made
   it.
3. Before any Task of a scope runs, every Service of the scope must be
   started in a Service Session of its own — entering the Environments whose
   `runScope` includes `SERVICE`, allocating its ports in each port's own
   `protocol` space, running `onEnter`, launching `onRun`, and probing it with
   its health check every `readinessIntervalSeconds` — and be READY.
   `Service.<name>.<port>.port` and `.connectAddress` must resolve in the
   Task's actions to an endpoint that reaches the process.
4. Once READY, the runner must keep probing every `healthIntervalSeconds`
   (`TCP_CONNECT` and `COMMAND`; `STDOUT` only when the field is given, as a
   heartbeat deadline for `openjd_service_ready`), count consecutive failures,
   reset the count on a success, and on `failureThreshold` failures mark the
   instance UNHEALTHY, cancel `onRun` by its cancelation method, wait for it to
   exit, and take the restart decision below.
5. An `onRun` exit while the scope has work, a ready timeout, or an UNHEALTHY
   instance is an instance failure governed by `restartPolicy`: relaunch up to
   `maxAttempts` times, applying `completedTasks`; otherwise fail the scope. A
   failure before `onRun` is launched (an Environment's or the Service's
   `onEnter` exiting non-zero) is a start failure that consumes an attempt and
   must be relaunched in a new Service Session. In every case the Service
   Session must be ended (`onRun` and any in-flight `onHealthCheck` canceled,
   `onExit` run if any action of the Service ran, Environments exited) when the
   scope completes or fails, a Service being stopped before any Service it
   references.
6. Under `WRAP_ACTIONS`, a wrapping Environment whose `runScope` includes
   `SERVICE` must run its `onWrapService*` hooks in place of the Service's
   actions with `WrappedAction.*` and `WrappedService.*` populated, scanning
   the wrap script's stdout for `openjd_*` messages.
