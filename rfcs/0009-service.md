* Feature Name: Service
* Author(s): Mark Wiebe <[mwiebe](https://github.com/mwiebe)>
* RFC Tracking Issue: (pending)
* Start Date: 2026-09-17
* Specification Version: 2023-09 extension SERVICE
* Accepted On: (pending)
* Depends On:
  * RFC 0002, Model Extensions (https://github.com/OpenJobDescription/openjd-specifications/issues/57)
* Related:
  * Discussion #145, the idea thread for this RFC (https://github.com/OpenJobDescription/openjd-specifications/discussions/145)
  * Discussion #136 / Issue #133, Environment [Monitor] extension (https://github.com/OpenJobDescription/openjd-specifications/issues/133)

## Summary

This RFC proposes a new `<Service>` entity that a Job Template can declare in two new lists,
`jobServices` and `stepServices`, alongside the existing `jobEnvironments` and `stepEnvironments`. A
Service is a long-lived process that a scheduler starts *before* any Task in its scope is scheduled,
keeps running for the lifetime of its scope (the whole Job, or a single Step), and stops once that
scope no longer needs it. Unlike an Environment, a Service publishes one or more named network
ports, may be placed on a different host than the Tasks that use it, and has an explicit readiness
signal that gates the scheduling of the Tasks in its scope. Every entity in the Service's scope can
discover the Service's endpoint through a new `Service.*` format-string scope.

A Service can also be supplied from outside the Job: an Environment Template may define a `service:`
in place of an `environment:`, so that a scheduler can start a per-Job Service for every Job
submitted through a queue, exactly as it applies queue Environments today. A Job Template declares
its dependency on such an **external Service** with an `<ExternalServiceRequirement>` so that its
`Service.*` references remain statically checkable. This RFC also adds to Environment Templates the
`extensions` key that Job Templates already have.

The RFC also reserves a new standard attribute capability, `attr.worker.preemptible`, so that a
Service (or any Step) can require placement on capacity that will not be reclaimed mid-scope.

## Basic Examples

### A Valkey store shared across a Job's Tasks

This job starts a single [Valkey](https://valkey.io/) in-memory data store before scheduling any
Task, and every Task in the Job connects to it to cache and coordinate intermediate results. The
service may run on a different host from any Task; Tasks discover the endpoint via
`Service.Cache.main.connectAddress`. If the service process dies the scheduler relaunches it, and
because a cache can be repopulated, completed Tasks keep their results (`completedTasks: KEEP`).

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [SERVICE]
name: "Frame Processing With Shared Valkey Store"
parameterDefinitions:
  - name: FrameEnd
    type: INT
    default: 100

jobServices:
  - name: Cache
    description: "A Valkey store the Job's Tasks share to cache and coordinate."
    hostRequirements:
      attributes:
        # Keep the service off capacity that could be reclaimed mid-Job.
        - name: attr.worker.preemptible
          anyOf: ["false"]
      amounts:
        - name: amount.worker.memory
          min: 8192
    ports:
      - name: main
    readinessCheck:
      type: TCP_CONNECT
      timeoutSeconds: 60
    restartPolicy:
      maxAttempts: 3
      completedTasks: KEEP
    script:
      actions:
        # onRun is the long-lived action: its process IS the service.
        onRun:
          command: valkey-server
          args:
            - "--port"
            - "{{ Service.Cache.main.port }}"
            - "--bind"
            - "{{ Service.Cache.main.bindAddress }}"
            - "--save"
            - ""
          cancelation:
            mode: NOTIFY_THEN_TERMINATE
            notifyPeriodInSeconds: 30

steps:
  - name: ProcessFrames
    parameterSpace:
      taskParameterDefinitions:
        - name: Frame
          type: INT
          range: "1-{{ Param.FrameEnd }}"
    script:
      actions:
        onRun:
          command: bash
          args: ["{{ Task.File.Run }}"]
      embeddedFiles:
        - name: Run
          type: TEXT
          data: |
            #!/usr/bin/env bash
            set -euo pipefail
            process-frame \
              --frame {{ Task.Param.Frame }} \
              --valkey-host '{{ Service.Cache.main.connectAddress }}' \
              --valkey-port {{ Service.Cache.main.port }}
```

### A coordinator that owns Step state

This Step runs a work-distribution service for its own Tasks. The coordinator keeps the assignment
state in memory, so if it dies and is relaunched, the Tasks that already completed cannot be
trusted: `completedTasks: RERUN` tells the scheduler to requeue them. Readiness is signaled by the
service itself on stdout, and the service does its one-time database initialization in `onEnter`
so that a restart of `onRun` does not wipe the schema.

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [SERVICE]
name: "Tiled Render With Per-Step Coordinator"

steps:
  - name: RenderTiles
    stepServices:
      - name: Coordinator
        ports:
          - name: api
          - name: metrics
        readinessCheck:
          type: STDOUT
          timeoutSeconds: 120
        restartPolicy:
          maxAttempts: 1
          completedTasks: RERUN
        script:
          actions:
            onEnter:
              command: coordinator
              args: ["init-db", "--data-dir", "{{ Session.WorkingDirectory }}/db"]
            onRun:
              command: coordinator
              args:
                - "serve"
                - "--data-dir"
                - "{{ Session.WorkingDirectory }}/db"
                - "--listen"
                - "{{ Service.Coordinator.api.bindAddress }}:{{ Service.Coordinator.api.port }}"
                - "--metrics"
                - "{{ Service.Coordinator.metrics.bindAddress }}:{{ Service.Coordinator.metrics.port }}"
                # The coordinator prints "openjd_service_ready" to stdout once its
                # listeners are accepting connections.
            onExit:
              command: coordinator
              args: ["export-report", "--data-dir", "{{ Session.WorkingDirectory }}/db"]
    parameterSpace:
      taskParameterDefinitions:
        - name: Tile
          type: INT
          range: "1-64"
    script:
      actions:
        onRun:
          command: render-tile
          args:
            - "--coordinator"
            - "http://{{ Service.Coordinator.api.connectAddress }}:{{ Service.Coordinator.api.port }}"
            - "--tile"
            - "{{ Task.Param.Tile }}"
```

### A queue-supplied license proxy

A scheduler can supply a Service to every Job submitted through a queue by attaching an Environment
Template whose root is `service:`. Here a queue administrator attaches a license proxy; the
`extensions` key on the Environment Template is new in this RFC.

```yaml
specificationVersion: "environment-2023-09"
extensions: [SERVICE]
parameterDefinitions:
  - name: LicenseServer
    type: STRING
    default: "27000@licenses.example.com"
service:
  name: LicenseProxy
  description: "Per-job license proxy so Tasks never contact the license server directly."
  ports:
    - name: flexlm
  restartPolicy:
    maxAttempts: 5
    completedTasks: KEEP
  script:
    actions:
      onRun:
        command: license-proxy
        args:
          - "--upstream"
          - "{{ Param.LicenseServer }}"
          - "--listen"
          - "{{ Service.LicenseProxy.flexlm.bindAddress }}:{{ Service.LicenseProxy.flexlm.port }}"
```

A Job Template that uses the proxy does not define it. It declares an `<ExternalServiceRequirement>`
naming the Service and the ports it uses, which lets `openjd check` validate the `Service.*`
references and tells a reader exactly what the Job expects from its queue. Submission is rejected if
the queue does not supply a matching Service.

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [SERVICE]
name: "Render With Queue License Proxy"

jobServices:
  - name: LicenseProxy
    external: true
    ports: [flexlm]

steps:
  - name: Render
    parameterSpace:
      taskParameterDefinitions:
        - name: Frame
          type: INT
          range: "1-100"
    script:
      actions:
        onRun:
          command: renderer
          args:
            - "--license"
            - "{{ Service.LicenseProxy.flexlm.port }}@{{ Service.LicenseProxy.flexlm.connectAddress }}"
            - "--frame"
            - "{{ Task.Param.Frame }}"
```

### Execution order

For a Job with one `jobService` `S`, one `jobEnvironment` `E`, and one Step with Tasks `T1..Tn`
running in two Sessions on two Worker Hosts, the scheduler proceeds:

```
Service host                   Worker host A            Worker host B
------------                   -------------            -------------
S.onEnter
S.onRun (starts)
S readiness probe -> READY
                               E.onEnter                E.onEnter
                                 T1.onRun                 T3.onRun
                                 T2.onRun                 T4.onRun
                               E.onExit                 E.onExit
S.onRun (canceled)
S.onExit
```

No Task in `S`'s scope is scheduled until `S` is READY. `S` outlives every Session that uses it
and is stopped only when the Job has no remaining Tasks that could run.

## Motivation

Many parallel workloads depend on a shared, long-lived process that is not itself a unit of work:
an in-memory store that Tasks read from and write to, a coordinator that hands out work or
implements a barrier between phases, or an application "server mode" process (a DCC or renderer
listening on a port) that is expensive to start and is reused by many Tasks.

Today the only way to provide such a process is to deploy it as infrastructure outside the job:
stand up a host, run the daemon, and hard-code (or pass as a Job Parameter) its address. This
pulls the dependency the Tasks rely on outside the job template, so the template is no longer a
self-contained, [tooling parseable](https://github.com/OpenJobDescription/openjd-specifications/wiki/Design-Tenets)
description of the work. It also cannot be started and stopped with the Job, so it either runs
permanently or requires bespoke orchestration around every submission.

An `<Environment>` almost fits but differs in four ways that a schema needs to make first-class:

1. **Placement.** An Environment is entered on the host that runs the Tasks. A Service may run on a
   different host and be shared by Tasks spread across many hosts and Sessions.
2. **Lifetime.** An Environment's actions run to completion at Session start and end. A Service's
   main action runs for the *whole* lifetime of its scope, and the scheduler must keep it alive
   across however many Sessions come and go.
3. **Endpoint publication.** A Service exposes one or more ports. The runtime allocates a concrete
   address and port and makes it discoverable both to the service process (so it knows what to
   bind) and to every entity in its scope (so they know where to connect), on any host.
4. **Readiness and health.** A scheduler must know when the service is accepting connections
   before it schedules the Tasks in the Service's scope, and must react when the service dies:
   relaunch it, or fail the scope. Because a service can hold state that Tasks depend on, a
   service author must be able to say whether Tasks that completed against the old instance are
   still valid.

Everything else about a Service — `name`, `description`, `variables`, `let`, embedded files, the
`<Action>` model and its cancelation methods — is deliberately identical to `<Environment>` so the
entity is immediately familiar and reuses the existing machinery.

### Use cases

1. **Shared cache / data store.** A Valkey or Redis-compatible store that Tasks use to memoize
   expensive intermediate results (e.g. shared geometry or texture bakes) within a Job.
2. **Work coordination.** A service that divides a large, irregular work list among the Tasks of a
   Step, collects results, or implements a barrier so that a "gather" phase begins only when all
   "scatter" Tasks have checked in.
3. **Application server mode.** A renderer, DCC, or simulation engine started once in server mode on
   a large host, with many light Tasks submitting commands to it over its port. This amortizes
   scene load across Tasks in the same way an Environment does, but across hosts.
4. **Job-local infrastructure.** A license proxy, a metrics sink, or a small object store that
   exists only for the duration of one Job, so that the whole pipeline is described by the
   template and torn down with it.
5. **Queue-supplied infrastructure.** A studio attaches a Service to a queue so that *every* Job
   submitted there gets, for example, a per-Job license proxy or asset cache, without any Job
   Template author having to know how it is provisioned. This is the Service analogue of the
   queue Environments that schedulers already apply from Environment Templates.

### Backward compatibility

This RFC is additive and gated by the `SERVICE` extension name declared under RFC 0002:

- Schedulers that do not implement `SERVICE` MUST reject templates that list it in `extensions:`.
- A template that uses `jobServices`, `stepServices`, or the `Service.*` format-string scope MUST
  list `SERVICE` in `extensions:`, and a scheduler MUST reject a template that uses any of these
  without declaring the extension.
- No existing field changes meaning. A template with no `SERVICE` extension is unaffected.
- The `$schema` and `extensions` keys added to the Environment Template schema are not gated by
  `SERVICE`. `extensions` has been accepted by implementations on Environment Templates since
  RFC 0002 (RFC 0008's example relies on it); this RFC documents existing behavior. Environment
  Templates that omit both keys are unaffected.
- `SERVICE` does not require `EXPR`. All new format-string values are plain strings and integers
  usable with the base format-string grammar. With `EXPR` enabled, the `Service.*` symbols carry
  the types listed in [The `Service.*` scope](#the-service-scope).

## Specification

### Terminology

| Term | Definition |
|------|------------|
| **Scheduler** | The system that accepts Job and Environment Templates, creates Jobs from them, and dispatches Sessions and Services to Worker Hosts. Distinct from the session runtime that runs actions on a host. Called a "render management system" in some older parts of the specification. |
| **Service** | An entity declared in `jobServices` or `stepServices`: a long-lived process that publishes one or more network ports and is kept alive by the scheduler for the lifetime of its scope. |
| **Scope** | The Job, for a Service in `jobServices`; the Step that declares it, for a Service in `stepServices`. |
| **Service Session** | The runtime context in which a Service's actions run: a working directory on the service's host, created before `onEnter` and deleted after `onExit`. Analogous to a Task Session but containing exactly one Service. |
| **Service host** | The Worker Host the scheduler places a Service on. It may or may not be a host that also runs Tasks. |
| **READY / UNREADY / FAILED** | The three Service states the scheduler tracks. See [Service lifecycle](#service-lifecycle). |
| **Instance** | One launch of a Service's `onRun` action. A Service has one instance at a time; a restart creates a new instance. |
| **External Service** | A Service supplied to a Job by the scheduler from an Environment Template, rather than declared in the Job Template. |

### Schema modifications

> Changes to [the template schema](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas).

#### Job Template

> A modification to [`1.1. Job Template`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#11-job-template)

```diff
  specificationVersion: "jobtemplate-2023-09"
  $schema: <string> # @optional
  extensions: [ <ExtensionName>, ... ] # @optional
  name: <JobName> # @fmtstring
  description: <Description> # @optional
  parameterDefinitions:  [ <JobParameterDefinition>, ... ] # @optional
  jobEnvironments: [ <Environment>, ... ] # @optional
+ jobServices: [ <Service> | <ExternalServiceRequirement>, ... ] # @optional @extension SERVICE
  steps: [<StepTemplate>, ...]
```

New property:

* *jobServices* — An ordered list of the Services required by Tasks in the Jobs created by this
  Job Template. Each Service is started, in list order, before any Task of the Job is scheduled and
  is stopped, in reverse order, once no Task of the Job remains to be run. An element may instead
  be an `<ExternalServiceRequirement>`, declaring that a Service of the given name and ports is
  supplied by the scheduler (see
  [Environment Template](#environment-template)). See [`<Service>`](#service) and
  [`<ExternalServiceRequirement>`](#externalservicerequirement). Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. Maximum number of elements: 10.
    3. No two elements in this list may have the same value for the `name` property.
    4. The elements of this list must not have the same `name` as a Service defined in any
       Step within the same Job Template.
    5. External Services are started before every `<Service>` in this list, so a `<Service>` may
       reference the ports of an `<ExternalServiceRequirement>` regardless of their relative
       positions in the list.

Extension list addition to item 3 (*extensions*): `SERVICE` (this RFC).

#### Environment Template

> A modification to [`1.2. Environment Template`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#12-environment-template)

```diff
  specificationVersion: environment-2023-09
+ $schema: <string> # @optional
+ extensions: [ <ExtensionName>, ... ] # @optional
  parameterDefinitions: [ <JobParameterDefinition>, ... ] # @optional
- environment: <Environment>
+ environment: <Environment> # @optional @incompatible service
+ service: <Service> # @optional @incompatible environment @extension SERVICE
```

New and changed properties:

* *$schema* — Ignored, as on the Job Template. Allowed for compatibility with JSON-editing IDEs.
* *extensions* — If provided, a non-empty list of extensions that this document uses, with the
  same available names as the Job Template. An extension listed here applies to the content of
  this document only; a Job Template does not need to list the extensions used by the Environment
  Templates a scheduler applies to it, and vice versa. Environment Templates have
  used this key informally since RFC 0002 (RFC 0008's example declares `WRAP_ACTIONS` and `EXPR`
  on one); this RFC makes it part of the schema.
* *service* — The definition of a Service that the Environment Template defines in place of an
  Environment. Exactly one of *environment* or *service* must be provided.

**Services from Environment Templates.** An Environment Template whose root is *service* defines a
Service that the scheduler applies to every Job Template submitted through it, just as one whose
root is *environment* defines an Environment applied to every submission. From the Job Template's
point of view this is an **external Service**. External Services are applied per Job: each Job gets
its own instance, with Job scope, exactly as if the Service had appeared in the Job Template's
`jobServices`. A Service defined this way does not outlive the Job; a process shared across many
Jobs is infrastructure outside the scope of this specification.

When a submission combines a Job Template with one or more Environment Templates:

1. The external Services are ordered as the scheduler orders the Environment Templates, and are
   placed before every Service in the Job Template's `jobServices`. The combined list is started
   and stopped as a single `jobServices` list. Consequently a Service in the Job Template's
   `jobServices`, and every Environment, Step, and Task in the Job, may reference the endpoint of
   any external Service, and an external Service may reference the endpoint of one from an
   Environment Template earlier in the scheduler's order.
2. The `name` of an external Service MUST NOT equal the `name` of any Service in the Job Template's
   `jobServices` or any Step's `stepServices`, except an `<ExternalServiceRequirement>` that it
   satisfies. The submission MUST be rejected on any other collision.
3. Every `<ExternalServiceRequirement>` in the Job Template MUST be satisfied by exactly one
   external Service with the same `name` whose *ports* include every port name the requirement
   lists. The submission MUST be rejected if any requirement is unsatisfied.
4. A `Service.*` reference within an Environment Template to a Service the same document does not
   declare is resolved at submission time against the external Services from Environment Templates
   earlier in the scheduler's order, and the submission MUST be rejected if it cannot be resolved. A Job Template MUST NOT make such an undeclared reference; it declares an
   `<ExternalServiceRequirement>` instead.

An external Service has the same effect on a Job as a Service the Job Template declared itself: the
Job's Tasks are not scheduled until it is READY, its host requirements are allocated for the Job's
lifetime, and its restart policy governs whether the Job's completed Tasks are rerun when it
restarts.

#### Step Template

> A modification to [`3. <StepTemplate>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#3-steptemplate)

```diff
  name: <StepName>
  description: <Description> # @optional
  let: <LetBindings> # @optional @extension EXPR
  dependencies: [ <StepDependency>, ... ] # @optional
  stepEnvironments: [ <Environment>, ... ] # @optional
+ stepServices: [ <Service>, ... ] # @optional @extension SERVICE
  hostRequirements: <HostRequirements> # @optional
  parameterSpace: <StepParameterSpaceDefinition> # @optional
  script: <StepScript>
```

New property:

* *stepServices* — An ordered list of the Services required by the Tasks of this Step. Each
  Service is started, in list order, after the Step's dependencies are satisfied and before any
  Task of the Step is scheduled, and is stopped, in reverse order, once no Task of the Step
  remains to be run. A Step Service is available only to the Step that declares it; it is not
  available to other Steps, including Steps that depend on this one. See [`<Service>`](#service).
  Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. Maximum number of elements: 10.
    3. No two Services in this list may have the same value for the `name` property.
    4. The Services defined in this list must not have the same `name` as a Job Service defined
       in the same Job Template.
    5. Note: as with Step Environments, the scope of a Step Service's `name` is the Step that
       defines it. Different Steps may each define a Step Service with the same `name`.

The `let` bindings of a `<StepTemplate>` are additionally available in *stepServices*.

#### `<Service>`

> A new top-level section, inserted after
> [`8. <SimpleAction>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#8-simpleaction),
> renumbering *Additional Information* and *License* to sections 10 and 11.

Available when using the `SERVICE` extension.

A Service is a long-lived process that a scheduler starts before scheduling the Tasks in its
scope, keeps running for the lifetime of its scope, and stops once the scope no longer needs it.
A Service publishes one or more named network ports that the runtime allocates, and every
entity in the Service's scope can discover the resulting endpoint via the `Service.*`
format-string scope.

A `<Service>` is the object:

```yaml
name: <ServiceName>
description: <Description> # @optional
let: <LetBindings> # @optional @extension EXPR
hostRequirements: <HostRequirements> # @optional
ports: [ <ServicePort>, ... ]
readinessCheck: <ServiceReadinessCheck> # @optional
restartPolicy: <ServiceRestartPolicy> # @optional
variables: <EnvironmentVariables> # @optional
script: <ServiceScript>
```

Where:

1. *name* — An identifier for the Service that is unique within its scope (the Job Template for a
   Job Service, the Step Template for a Step Service). It is the first component of `Service.*`
   references to this Service. See [`<ServiceName>`](#servicename).
2. *description* — A description to apply to the Service. It has no functional purpose, but may
   appear in UI elements.
3. *let* — An ordered list of expression bindings evaluated once when the Service is started.
   Bound names are available in *hostRequirements*, *variables*, and *script*. Available with the
   `EXPR` extension.
4. *hostRequirements* — Requirements on the Worker Host's capabilities that must be satisfied for
   the Service to be placed on the host. Amount capabilities are allocated to the Service Session
   for the lifetime of the Service. This is independent of the *hostRequirements* of any Step
   whose Tasks use the Service.
5. *ports* — The named ports that the Service exposes. See [`<ServicePort>`](#serviceport).
    1. Minimum number of elements: 1.
    2. Maximum number of elements: 10.
    3. No two ports may have the same `name`.
6. *readinessCheck* — How the scheduler determines that the Service is ready to accept connections.
   If not provided, defaults to `{ type: TCP_CONNECT }` applied to every port. See
   [`<ServiceReadinessCheck>`](#servicereadinesscheck).
7. *restartPolicy* — What the scheduler does when the Service's `onRun` action exits before the scope
   ends. If not provided, defaults to `{ maxAttempts: 0, completedTasks: RERUN }`: the Service is
   never relaunched, and its exit fails the scope. See
   [`<ServiceRestartPolicy>`](#servicerestartpolicy).
8. *variables* — A set of environment variable name/value pairs, with the values being Format
   Strings resolved when the Service is started, that are set in the process environment of every
   action of the Service's *script* and of the `COMMAND` readiness probe. This is the declarative
   way to configure the service process: a Service's *onRun* is very often a bare binary that takes
   its configuration from environment variables (`VALKEY_*`, `OTEL_*`, and so on), and *variables*
   supplies them without wrapping the command in a shell, in a form that tooling can read. It has
   the same schema as the *variables* property of
   [`<Environment>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#4-environment).
   A Service's *variables* are **not** propagated to the entities in its scope; they receive the
   endpoint through `Service.*` only. Values computed at *onEnter* time are passed to *onRun* with
   `openjd_env` instead (see [`<ServiceActions>`](#serviceactions)).
9. *script* — The actions that the Service runs on its host. See [`<ServiceScript>`](#servicescript).

The format string scopes available to format strings within a `<Service>` are:

1. `Param.*` and `RawParam.*` — Values of Job Parameters.
2. `Session.*` — Values such as the Service Session's working directory. `Session.WorkingDirectory`
   refers to the Service Session's own working directory on the service host, not to any Task
   Session.
3. `Service.File.<name>` — The filesystem location of an embedded file defined within this
   Service's *script*. See [`<ServiceScript>`](#servicescript).
4. `Service.<name>.<port>.*` — The endpoint of this Service's own ports, and of any Service
   earlier in the same list (for a Step Service, also any Job Service). See
   [The `Service.*` scope](#the-service-scope).
5. `Step.Name` — Available in a Step Service only. Requires the `EXPR` extension.
6. Names bound by `let` in the enclosing `<StepTemplate>` (for a Step Service) and in the Service
   itself. Available with the `EXPR` extension.

`Task.*` values are never available within a Service, since a Service is not associated with any
Task.

##### `<ServiceName>`

An [`<Identifier>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#71-identifier)
subject to the additional constraint that it MUST NOT be `File`, which is reserved for the
`Service.File.*` embedded-file references.

Note that, unlike `<EnvironmentName>`, a Service's name is restricted to identifier characters
because it appears as a component of dotted `Service.*` value references.

##### `<ServicePort>`

A `<ServicePort>` is the object:

```yaml
name: <Identifier>
port: <posinteger> # @optional
```

Where:

1. *name* — The name of the port. It is the second component of `Service.<service>.<port>.*`
   references. Must not be `File`.
2. *port* — If provided, the Service requires this specific TCP port number on its host. If the
   scheduler cannot provide that port (for example, because it is in use), the Service fails to
   start. If not provided, the runtime allocates an available port. Range: 1–65535. Authors
   SHOULD omit this and let the runtime allocate, so that multiple Services and Sessions can share
   a host.

All ports are TCP. Every port is bound and published on the same service host. UDP ports and
Unix domain sockets are out of scope for this RFC (see [Future Work](#future-work)).

##### `<ServiceReadinessCheck>`

A `<ServiceReadinessCheck>` is one of the following objects, discriminated by *type*:

```yaml
type: "TCP_CONNECT"
ports: [ <Identifier>, ... ] # @optional
timeoutSeconds: <posinteger> # @optional
```

```yaml
type: "COMMAND"
probe: <Action>
intervalSeconds: <posinteger> # @optional
timeoutSeconds: <posinteger> # @optional
```

```yaml
type: "STDOUT"
timeoutSeconds: <posinteger> # @optional
```

Where:

1. *type* — The probe mechanism:
    * `TCP_CONNECT` — The Service is READY once a TCP connection to each of the listed ports (by
      default, every port the Service declares) succeeds. The connection is made from the service
      host to the allocated `bindAddress` and `port`, and closed immediately. The scheduler retries
      at an implementation-defined interval (recommended: 1 second) until success or timeout.
    * `COMMAND` — The Service is READY once the *probe* action exits with status 0. The probe is
      run on the service host, in the Service Session, with the Service's *variables* set and the
      full `Service.*` scope available. It is re-run every *intervalSeconds* (default: 5) until it
      succeeds, the timeout elapses, or `onRun` exits. The probe's own `timeout` (default: 30
      seconds) bounds one invocation. A probe exit status other than 0 is "not yet ready", never a
      failure of the Service.
    * `STDOUT` — The Service is READY once its `onRun` action writes a line matching the regular
      expression `^openjd_service_ready(: .*)?$` to stdout. The optional message after the colon
      has no functional purpose but MAY be surfaced in UIs. This is consistent with the existing
      `openjd_*` stdout protocol and is the mechanism of choice for services whose readiness is
      not observable from outside the process.
2. *ports* (`TCP_CONNECT` only) — The names of the ports to probe. Each must be declared in the
   Service's *ports*. Defaults to all of them.
3. *probe* (`COMMAND` only) — The action to run as the probe. See
   [`<Action>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#5-action).
4. *intervalSeconds* (`COMMAND` only) — Seconds to wait between the end of one probe invocation
   and the start of the next. Default: 5.
5. *timeoutSeconds* — The maximum time, measured from the start of the `onRun` action, that the
   scheduler waits for the Service to become READY. If exceeded, the instance is treated as failed
   (see [Failure and restart](#failure-and-restart)). Default: 300 seconds.

Readiness applies to every instance: after a restart, the new `onRun` must pass the readiness probe
before Tasks in the scope are (re)scheduled.

##### `<ServiceRestartPolicy>`

A `<ServiceRestartPolicy>` is the object:

```yaml
maxAttempts: <nonnegativeinteger> # @optional
completedTasks: enum("KEEP", "RERUN") # @optional
```

Where:

1. *maxAttempts* — The maximum number of times the scheduler will relaunch the `onRun` action after
   it exits, or fails to become READY, before the scope ends. Default: 0 (never relaunch).
2. *completedTasks* — What happens, when a new instance is launched, to Tasks in the Service's
   scope that completed successfully against a previous instance. (Tasks that were running when
   the previous instance failed are always canceled and requeued; Tasks not yet started are simply
   scheduled once the new instance is READY. Only completed Tasks are a choice.)
    * `KEEP` — Completed Tasks remain complete. Use this when the Service holds no state that
      Tasks depend on, or state that Tasks can regenerate (a cache).
    * `RERUN` — Completed Tasks are returned to the scheduling queue and run again. Use this when
      the Service holds state that makes prior results unreliable once lost (a coordinator).
    * Default: `RERUN`. This is the safe default: an author must opt in to keeping results.

If *maxAttempts* is exhausted, or the policy is not provided, a `onRun` exit fails the scope
regardless of *completedTasks*.

##### `<ServiceScript>`

A `<ServiceScript>` is the object:

```yaml
let: <LetBindings> # @optional @extension EXPR
actions: <ServiceActions>
embeddedFiles: [ <EmbeddedFile>, ... ] # @optional
```

1. *let* — An ordered list of expression bindings evaluated once when the Service is started.
   Bound names are available in *actions* and *embeddedFiles*.
2. *actions* — The actions to run at the different stages of the Service's lifecycle.
3. *embeddedFiles* — Files embedded into the Service that are materialized to the Service
   Session's working directory before each of the Service's actions runs. The path of an embedded
   file is available as `Service.File.<name>`.

##### `<ServiceActions>`

A `<ServiceActions>` is the object:

```yaml
onEnter: <Action> # @optional
onRun: <Action>
onExit: <Action> # @optional
```

Where:

1. *onEnter* — A one-time setup action run before the first `onRun` of the Service on a host. It
   runs to completion. A non-zero exit fails the Service without consuming a restart attempt, and
   the Service becomes FAILED. Use *onEnter* for setup that must survive a restart of *onRun* (for
   example, initializing an on-disk database in the Service Session's working directory). It is
   re-run only when the scheduler relocates the Service to a new host (see
   [Failure and restart](#failure-and-restart)).
2. *onRun* — The long-lived action whose process *is* the service. The scheduler starts it, watches
   it for readiness, and expects it to run until canceled. If *onRun* exits for any reason — zero or
   non-zero exit status — before the scheduler cancels it, the Service instance has failed; see
   [Failure and restart](#failure-and-restart). The scheduler stops the Service by canceling *onRun*
   according to its `cancelation` method.
3. *onExit* — A cleanup action run after *onRun* has stopped for the last time on a host, whether
   the Service stopped normally, failed, or was canceled. It runs to completion. A non-zero exit
   is reported but does not change the outcome of the scope.

Default timeouts for these actions (extending the table in `<Action>`):

| Action container | Action | Default |
|---|---|---|
| `<ServiceActions>` | `onEnter` | *no timeout* |
| `<ServiceActions>` | `onRun` | *no timeout* (a timeout, if given, is measured from launch and its expiry is an instance failure) |
| `<ServiceActions>` | `onExit` | 300 seconds (five minutes) |

Implementations MUST watch the stdout of every `<ServiceActions>` action for the standard
`openjd_status`, `openjd_progress`, and `openjd_fail` messages, and the stdout of *onRun* for
`openjd_service_ready` when the readiness check's *type* is `STDOUT`.

**Environment variables within a Service.** Implementations MUST additionally watch the stdout of
*onEnter* for `openjd_env`, `openjd_redacted_env`, and `openjd_unset_env`, with the same syntax
and redaction rules as for an Environment's `onEnter`. A variable set this way is set in the
process environment of every subsequent action of the same Service — every instance of *onRun*, the
`COMMAND` readiness probe, and *onExit* — and is retained across relaunches of *onRun* on the same
host. This is how *onEnter* hands values it computes (a generated credential, a discovered device,
a path it created) to the service process, and it is the reason those values survive a restart
when *onEnter* is not re-run. Variables set by *onEnter* take precedence over the Service's
declarative *variables*. These messages are ignored when emitted by *onRun*, the probe, or *onExit*,
and nothing set within a Service is ever propagated to the entities in the Service's scope.

##### `<ExternalServiceRequirement>`

An `<ExternalServiceRequirement>` is an element of a Job Template's `jobServices` that declares a
dependency on a Service supplied by the scheduler from an Environment Template, rather than defining
the Service in the Job Template. It makes the dependency visible to tooling and lets the Job
Template's `Service.*` references be validated without knowledge of the scheduler's configuration.

```yaml
name: <ServiceName>
description: <Description> # @optional
external: true
ports: [ <Identifier>, ... ]
```

Where:

1. *name* — The `name` of the external Service that must be supplied. Subject to the same
   uniqueness constraints as a Service `name` in `jobServices`.
2. *description* — What the Job expects from the Service. No functional purpose.
3. *external* — The literal `true`. Distinguishes an `<ExternalServiceRequirement>` from a
   `<Service>`.
4. *ports* — The names of the ports of the external Service that the Job Template references.
   Each must be declared by the supplied Service. Minimum 1, maximum 10 elements.

Within the Job Template, `Service.<name>.<port>.port` and `Service.<name>.<port>.connectAddress`
for each listed port are in scope everywhere the corresponding values of a Job Service would be.
`bindAddress` is not available, since the Job Template does not define the service process.

A submission of a Job Template containing an `<ExternalServiceRequirement>` MUST be rejected unless
the scheduler supplies a Service with the same `name` declaring every listed port.

#### Host requirements

> A modification to the attribute capability table in
> [`3.3.2.1. <AttributeCapabilityName>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#3321-attributecapabilityname)

```diff
  |**Capability Name**|**Values**|**Description**|
  |---|---|---|
  |attr.worker.os.family|linux, windows, macos|The family of operating system on the host.|
  |attr.worker.cpu.arch |x86_64, arm64|The architecture of the CPUs on the host.|
+ |attr.worker.preemptible|true, false|Whether the host's capacity may be reclaimed by the infrastructure provider while work is running on it (for example, EC2 Spot Instances). A host that does not advertise this attribute has an unknown value; a requirement of `anyOf: ["false"]` does not match such a host.|
```

This attribute is not gated by the `SERVICE` extension; it is a reserved standard name that any
Step may require. This RFC introduces it because a long-lived Service is the first entity in the
specification whose correctness depends on not being preempted mid-scope.

#### Format strings

> A modification to
> [`7.3.1. Value References`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#731-value-references)

##### The `Service.*` scope

New value references, all annotated `@fmtstring[host]` because a Service's endpoint is not known
until the scheduler places the Service, and may change if the Service is relocated:

| Value | Type (with EXPR) | Description | Scope |
|---|---|---|---|
| `Service.<name>.<port>.port` | `int` | The TCP port number allocated (or requested) for port `<port>` of Service `<name>`. The same number is used for binding and for connecting. | Within the declaring Service; within every entity in the Service's scope (see below). |
| `Service.<name>.<port>.bindAddress` | `string` | The interface address the service process MUST bind to so that entities in the Service's scope can reach it. Typically `0.0.0.0` (or `::`) for a distributed scheduler and `127.0.0.1` for a single-host runner. | Within the declaring Service only. |
| `Service.<name>.<port>.connectAddress` | `string` | The hostname or IP address that entities in the Service's scope use to reach it. | Within every entity in the Service's scope. Also within the declaring Service, for self-reference (e.g. to print its own URL). |
| `Service.File.<name>` | `path` | The filesystem location to which the Service embedded file with key `<name>` has been written. | Within the Service Script actions, readiness probe, and embedded files of the declaring Service. |

`Service.<name>.<port>.*` for a Service `<name>` is in scope in:

1. The Service `<name>` itself (all three values).
2. Any Service later in the same `jobServices` or `stepServices` list, and — for a Job Service —
   any Step Service in the Job. A Service cannot reference a Service later in its own list, nor a
   Step Service of a different Step; this ordering rule guarantees the referenced Service is READY
   before the referencing one starts.
3. For a Job Service: every `jobEnvironments` entry, every Step's `stepEnvironments`,
   `stepServices`, and `script`, and every Step's `hostRequirements` attribute values.
4. For a Step Service: that Step's `stepEnvironments`, later `stepServices`, `script`, and
   `hostRequirements` attribute values.

`bindAddress` is deliberately *not* available outside the declaring Service: a Task connecting to
the Service has no use for the interface the service bound, and exposing it would tempt authors to
hard-code the assumption that the service and the Task share a host.

A template that references a `Service.*` value outside the scopes above, or that references a
Service or port name that is not declared, is invalid and MUST be rejected before the Job is
created.

##### Template processing stages

> A modification to
> [`7.4. Template Processing Stages`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#74-template-processing-stages)

`Service.*` values are unknown at template validation and at job creation, and known at
task execution (and at service execution, for the Service's own actions). With the `EXPR`
extension they type-check as `unresolved[int]`, `unresolved[string]`, and `unresolved[path]`
at the earlier stages.

### Modifications to How Jobs Are Run

> Additions to [How Jobs Are Run](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run)

#### Service lifecycle

A Service is in exactly one of three states, tracked by the scheduler:

| State | Meaning |
|---|---|
| **UNREADY** | No instance is READY. Either no instance has been launched yet, an instance is launched but has not passed its readiness probe, or the previous instance failed and a restart is pending or in progress. |
| **READY** | The current instance has passed its readiness probe and has not exited. |
| **FAILED** | The Service will not be relaunched. Its scope has failed. |

When a Job is created, each Job Service is UNREADY. When a Step's dependencies are satisfied, each
of its Step Services is UNREADY.

The scheduler starts a Service as follows:

1. Choose a service host satisfying the Service's *hostRequirements* and allocate the Service's
   amount capabilities to it for the lifetime of the Service.
2. Allocate a TCP port on that host for each declared port (or verify the requested port is
   available), and determine `bindAddress` and `connectAddress` for each.
3. Create the Service Session working directory.
4. Run *onEnter*, if defined. A non-zero exit sets the Service FAILED.
5. Launch *onRun*. This creates the first instance.
6. Apply the readiness probe. When it passes, the Service is READY. When it times out, or *onRun*
   exits before it passes, the instance has failed (see below).

Services in a list are started in order: the scheduler MUST NOT launch Service *i+1*'s *onRun*
until Service *i* is READY, since *i+1* may reference *i*'s endpoint. Services in different lists
(a Job Service and a Step Service) are ordered only by scope: all Job Services are READY before
any Step Service is started.

**Gating.** The scheduler MUST NOT schedule any Task of a Step until every Job Service and every
Step Service of that Step is READY. Once READY, Tasks are scheduled exactly as today, with no
change to how Sessions are formed.

**Stopping.** A Service is stopped when its scope completes: for a Job Service, when the Job has
no Tasks that could still run (all Tasks are complete, or the Job has failed or been canceled);
for a Step Service, likewise for the Step. Services are stopped in reverse list order. To stop a
Service the scheduler cancels *onRun* using its `cancelation` method, then runs *onExit* if defined,
then deletes the Service Session working directory and releases the host's allocated amounts.

A Service MUST be stopped when its scope completes even if it is UNREADY or FAILED; *onExit*
still runs in those cases so that partial state can be cleaned up.

#### Failure and restart

An **instance failure** occurs when *onRun* exits (with any status) before the scheduler cancels it,
or when the readiness probe times out. On an instance failure the Service becomes UNREADY and the
scheduler:

1. Cancels every Task in the Service's scope that is currently running. Each such Task is
   returned to the queue and will run again once the Service is READY; this is not counted as a
   Task failure.
2. If *onRun* is still running (readiness timeout case), cancels it.
3. If the number of relaunches so far is less than `restartPolicy.maxAttempts`:
    1. If `restartPolicy.completedTasks` is `RERUN`, returns every successfully completed Task in the
       scope to the queue.
    2. Relaunches *onRun* on the same host, in the same Service Session working directory, with the
       same allocated ports, and applies the readiness probe again. *onEnter* is not re-run.
4. Otherwise, the Service becomes FAILED and its scope fails: a failed Job Service fails the Job,
   a failed Step Service fails the Step. The scheduler then stops the Service as described above.

**Relocation.** A scheduler MAY relocate a Service to a different host as part of a restart, for
example when the original host is lost or preempted. Relocation counts as one relaunch. On
relocation the scheduler repeats the full start sequence on the new host (new ports, new working
directory, *onEnter* re-run) and every `Service.<name>.*` value may change. Because entities in
the scope resolve `Service.*` at task execution time on the worker host, a Task scheduled after
relocation sees the new endpoint automatically; a Task that was running during the relocation was
canceled in step 1 and will re-resolve when it runs again.

**Interaction with Task failures.** A Task failure never fails a Service. A Task that fails
because it could not reach a Service that the scheduler still considers READY is an ordinary
Task failure. Authors SHOULD prefer the `COMMAND` or `STDOUT` readiness probe when a plain TCP
connect is not a sufficient indicator that the service can do useful work.

#### Placement

Placement is entirely up to the scheduler, subject to the Service's *hostRequirements*. A
distributed scheduler might dedicate a host to a Service, co-locate several Services, or run a
Service on a host that also runs Sessions. A single-host runner such as `openjd run` starts the
Service alongside the Tasks on the same host with `bindAddress` and `connectAddress` both on
loopback.

Whatever the placement, the scheduler is responsible for network reachability: `connectAddress`
and `port` as seen from any host in the scope MUST reach the service process. How that is
achieved — a shared network, an overlay, port forwarding, a proxy — is an implementation
concern and is not visible in the template.

#### Stdout/Stderr messages

> Addition to the list in *Stdout/Stderr Messages*

* `openjd_service_ready` or `openjd_service_ready: <message>` — May be emitted only by the `onRun`
  action of a Service whose readiness check *type* is `STDOUT`. Indicates that the service is accepting
  connections on all of its declared ports. Emitting it more than once has no additional effect;
  emitting it from any other action is ignored.

## Design Choice Rationale

### A new entity rather than extending `<Environment>`

An Environment is defined by *where* it runs: on the host of the Session, entered and exited around
that Session's Tasks. Every one of the four differences in [Motivation](#motivation) — cross-host
placement, scope-long lifetime, endpoint publication, readiness — contradicts that model. Adding
`ports:` and a long-lived `onRun:` to `<Environment>` would produce an entity whose lifecycle depends on which
optional fields are present, which is hard to read and harder to implement. A distinct `<Service>`
that *reuses* Environment's sub-schemas (`variables`, `let`, embedded files, `<Action>`) gets the
familiarity without the ambiguity. (The Environment Template *document* is reused as a container
for an external Service, see below, but the `<Service>` entity itself is distinct from
`<Environment>`.)

### `onEnter` / `onRun` / `onExit` rather than a single `onRun`

An earlier sketch had only the long-lived action. Two things argued for the three-action form:

1. **Restart hygiene.** A restart policy needs a clear statement of *what* is relaunched. With
   only `onRun`, any one-time initialization is re-executed on every restart, which is wrong for
   anything with on-disk state. `onEnter` gives authors a place for setup that must survive a
   restart of the service process; `onExit` gives them a place for cleanup that must run whether
   the process exited cleanly or was killed.
2. **Symmetry with `<Environment>` and `<StepScript>`.** Authors already know `onEnter` / `onExit`
   from Environments and `onRun` from Steps. The only new idea is `onRun`'s definition for a
   Service ("the process that is the service, expected to run until canceled"), which is what
   distinguishes a Service; the names are all existing ones.

### `onRun` exiting is always a failure

There is no such thing as a service that has "finished" while its scope still has work. Treating a
zero exit as success would silently leave the Tasks in its scope without their endpoint. Making
*any* exit an instance failure keeps the rule simple; authors who want a service to stop early
should let the scope complete.

### Readiness is declared, not inferred

A scheduler cannot know when an arbitrary process is ready. The three probe types cover the three
places readiness is observable: the network (`TCP_CONNECT`, the zero-configuration default), an
external check (`COMMAND`, for protocol-level health such as a `PING` or an HTTP `/healthz`), and
the process itself (`STDOUT`, for services whose authors can add one print statement). The default
of TCP_CONNECT-on-all-ports means the simplest templates need no `readinessCheck:` block at all.

### `restartPolicy.completedTasks` is per-Service, defaulting to `RERUN`

The two motivating examples pull in opposite directions: a cache loses nothing of consequence when
it restarts, while a coordinator that loses its state has invalidated every result produced against
it. No single behavior is correct, so the policy is per-Service. `RERUN` is the default because it
is the conservative choice: it can only cost time, whereas `KEEP` applied to a stateful service can
produce silently incorrect output. Templates that know their service is stateless opt in to
`KEEP`.

### `variables` and `openjd_env` both, within a Service

A Service has two ways to put a value in its process's environment, for two different kinds of
value. *variables* is for configuration known when the template is written or the Job is created:
it is declarative, visible to tooling, and lets a bare binary be configured without a shell
wrapper. `openjd_env` from *onEnter* is for values that exist only once the Service is on a host:
a generated password, a discovered device, a directory *onEnter* created. Because *onEnter* runs
once per host and *onRun* may be relaunched several times, the variables *onEnter* sets are exactly
the state that has to survive a restart, and carrying them in the process environment means a
relaunched *onRun* needs no code to recover them. Both mechanisms already exist on `<Environment>`
with the same shapes, so a Service adds no new syntax, only the rule that its variables stay
inside the Service.

### Names are `<Identifier>`s

`Service.<name>.<port>.port` is a dotted value reference. Allowing the full `<EnvironmentName>`
character set (which includes spaces and punctuation) in a Service name would make it impossible to
reference. Reserving `File` keeps `Service.File.<name>` unambiguous.

### Endpoint as three scalar values

`port`, `bindAddress`, and `connectAddress` are separate scalars rather than a single `host:port`
string so they compose with any URL scheme (`http://{{...}}:{{...}}`, `redis://`, a bare `--host`
/ `--port` pair) without string surgery. `bindAddress` is separate from `connectAddress` because
they differ in essentially every deployment (a process binds `0.0.0.0`; clients connect to a
hostname), and conflating them is the most common source of "works on one host, not on two" bugs.

### Ordered lists with forward-only references

Requiring that a Service reference only Services *earlier* in its list makes the start order the
dependency order with no additional syntax and no possibility of a cycle. This mirrors the
existing ordered-entry semantics of `jobEnvironments`.

### `attr.worker.preemptible` in this RFC

A Service that dies because its host was reclaimed will, at best, restart and rerun the Tasks in its
scope; at worst it fails the Job. The ability to say "not on Spot" is essential for a stateful
Service, and the attribute is small and generally useful, so it is defined here rather than
deferred.

### External Services via the Environment Template, not a new template type

A separate `servicetemplate-2023-09` root element was considered and rejected on deployment surface.
Every scheduler that supports Environment Templates already has the plumbing to attach a document to
a queue, order the attachments, merge their parameter definitions, and apply them to each
submission. Expressing a Service as a variant of that same document means Services work everywhere
Environment Templates do with no new integration; a second root element would require every
implementation to add a second attachment type, UI, and API, and a rule for interleaving the two
lists, before anyone could use the feature. A single attachment list also gives a total order for
free, which matters when an administrator wants a queue Service and a queue Environment that
references it (for example, setting a legacy `VALKEY_HOST` variable from
`Service.Cache.main.connectAddress`) to travel together.

The document's name becomes slightly misleading. That is cosmetic and can be addressed when the
specification version next rolls; a second root element would be permanent surface area.

### Per-Job instances of external Services

An external Service produces one instance per Job rather than one shared by every Job in the
queue. A shared instance would outlive any Job, so `restartPolicy.completedTasks`, the working
directory, host-requirement allocation, and "stop when the scope completes" would all need
different definitions. Keeping external Services Job-scoped means every rule in this RFC applies
to them unchanged, and leaves genuinely queue-long processes where they belong: in infrastructure.

### An explicit requirement stub rather than deferred validation

A Job Template that uses an external Service must reference `Service.<name>.*` for a Service it does
not declare. The alternative to the `<ExternalServiceRequirement>` stub is to permit unknown
`Service.*` names in a Job Template and check them at submission time. That was rejected because it
makes `openjd check` unable to catch a typo in a Service or port name, and because a reader of the
template could no longer tell what the Job depends on. The stub costs one field (`external: true`)
and restores both properties; it is the same interface/implementation split that a Job Parameter
definition already has with the value supplied at submission. Environment Templates, being
administrator artifacts that the scheduler composes, are allowed the deferred form.

## Open Questions

### Do Services run inside Environments?

This RFC deliberately does **not** enter any `jobEnvironments` on the service host before running
a Service's actions. A Service is responsible for its own setup, in *onEnter* or *onRun*. This is
the simplest rule, but it means a Service cannot reuse an Environment Template that, for example,
activates a Conda environment or starts a container (RFC 0008) — the same software-provisioning
that Tasks get for free.

The obvious fix, "enter `jobEnvironments` before `onRun`", has problems: Job Environments may
themselves reference `Service.*` (a circular dependency), and an Environment that installs
software would run once per service host as well as once per Session. A per-Service
`environments: [<Environment>, ...]` list would avoid the cycle but introduces a third place
Environments are declared.

We suspect the right answer is more general than Services: **a mechanism for a Step (or a
Service) to select, skip, or customize the Environments applied to it.** Steps today have the same
limitation in miniature — every Job Environment applies to every Step, with no opt-out. If such a
mechanism existed, a Service would simply use it. We propose leaving Services environment-free in
this RFC and pursuing that generalization separately; feedback on whether that ordering is
acceptable is requested.

### Health monitoring after READY

This RFC detects an instance failure only when `onRun` exits. A hung process that still holds its
port is invisible to it. [Issue #133](https://github.com/OpenJobDescription/openjd-specifications/issues/133)
proposes a general monitoring mechanism for Actions; a Service's `onRun` is an obvious consumer. We
have not added a Service-specific liveness probe here so that the two proposals do not diverge.
If #133 lands first, this RFC should be updated to say how its monitor applies to `onRun` (most
likely: a monitor failure is an instance failure).

## Prior Art

* **Kubernetes** — `Service` + `Deployment` with `readinessProbe` (`tcpSocket`, `exec`,
  `httpGet`) and `restartPolicy`. Our three readiness types map directly onto `tcpSocket`,
  `exec`, and a stdout signal in place of `httpGet` (which would require HTTP semantics in the
  specification). Kubernetes services are not scoped to a batch of work, which is the gap this RFC
  fills.
* **Docker Compose** — `depends_on: { condition: service_healthy }` plus `healthcheck`. The
  ordered-start-until-healthy behavior is the model for our list ordering.
* **GitHub Actions service containers** — `services:` in a job, started before steps and stopped
  after, with ports published to the runner. Scoped to one job on one host; the closest
  batch-oriented precedent for `stepServices`.
* **Slurm heterogeneous jobs / `srun --overlap`** — Users commonly launch a coordinator (e.g. a Ray
  head or a Dask scheduler) as one component and workers as another, passing the head's address by
  hand through environment variables or a shared file. This RFC is the tooling-parseable version of
  that pattern.
* **Nomad** — `service` stanzas with `check` blocks (`tcp`, `script`, `http`) and `restart`
  stanzas with `attempts`. The `KEEP`/`RERUN` distinction has no direct analogue; it is specific to
  batch systems where completed work has a value that a service restart can invalidate.

## Implementation Impact

- **Python (openjd-model)**: New models for `Service`, `ServicePort`, `ServiceReadinessCheck` (a
  discriminated union), `ServiceRestartPolicy`, `ServiceScript`, `ServiceActions`, and
  `ExternalServiceRequirement` (discriminated from `Service` by `external: true`); new
  `jobServices` / `stepServices` fields gated on the `SERVICE` extension; `$schema`,
  `extensions`, and a `service` alternative to `environment` on the Environment Template model;
  name-uniqueness validators mirroring the environment ones; a `Service.*` symbol scope with the
  scoping rules in [The `Service.*` scope](#the-service-scope); `Service.File.*` alongside
  `Env.File.*` and `Task.File.*`; and a submission-time merge step that orders external Services,
  checks name collisions, and matches `<ExternalServiceRequirement>`s. Moderate.
- **Python (openjd-sessions)**: A new session kind that runs `onEnter`, launches `onRun` without
  awaiting exit, applies a readiness probe, exposes an "instance exited" callback, and runs
  `onExit`. Reuses the existing action runner, stdout scanner, and cancelation logic. Moderate.
- **Rust (openjd-rs)**: Same shape as Python across `openjd-model` and `openjd-sessions`.
  Moderate.
- **Schedulers**: This is where the real cost lies. A scheduler must place Services, allocate
  ports, guarantee cross-host reachability, gate Task scheduling on readiness, track instance
  failures, implement `KEEP`/`RERUN`, and stop Services at scope end. A single-host runner
  (`openjd run`) can implement all of this with loopback addresses and a process table; a
  distributed scheduler needs a networking answer.

No feature here is significantly harder in one language than another. There are no evaluation
performance implications; `Service.*` adds a handful of symbols resolved at task execution time.

## Rejected Ideas

### Passing the endpoint through environment variables

An earlier idea was to export `OPENJD_SERVICE_<NAME>_<PORT>_ADDRESS` into every Session. This was
rejected on the Tooling-parseable tenet: environment variables hide data flow from tools that read
the template, whereas `{{ Service.Cache.main.connectAddress }}` is visible. This is the same
reasoning recorded for `OPENJD_SESSION_WORKING_DIR` in *How Jobs Are Run*.

### Automatic health-check-based liveness in v1

Rejected in favor of coordinating with [Issue #133](https://github.com/OpenJobDescription/openjd-specifications/issues/133);
see [Open Questions](#health-monitoring-after-ready).

### Services as a kind of Step

One could model a Service as a Step with a single long-running Task and have other Steps depend
on it. This breaks in several ways: Step dependencies mean "after it *finishes*", not "while it
runs"; a Step's Task has no way to publish an endpoint; and a Step cannot be stopped when the
Steps that depend on it finish. The Service entity exists precisely because the Step lifecycle is
wrong for it.

### `completedTasks` as a Job-wide setting

Rejected because a Job commonly has one stateless cache and one stateful coordinator; the choice is
inherent to each Service.

### A separate Service Template root element

Rejected in favor of a `service:` alternative on the existing Environment Template; see
[Design Choice Rationale](#external-services-via-the-environment-template-not-a-new-template-type).

### Unchecked `Service.*` references in Job Templates

Rejected in favor of the `<ExternalServiceRequirement>` stub; see
[Design Choice Rationale](#an-explicit-requirement-stub-rather-than-deferred-validation).

## Future Work

* **Environments for Services**, or a general mechanism for selecting Environments per Step /
  Service. See [Open Questions](#do-services-run-inside-environments).
* **UDP ports and Unix domain sockets.** A `protocol:` field on `<ServicePort>` is the natural
  extension once there is demand.
* **Replicated Services.** A `replicas:` count producing `Service.<name>.<port>.connectAddresses`
  as a `list[string]`, for services that scale horizontally (a Dask worker pool, a distributed
  cache). Deferred until the single-instance model has been exercised.
* **Session-scoped Services.** A service that is started once per Session on the Session's host,
  for per-host sidecars. This may be better served by an Environment whose `onEnter` backgrounds a
  process, as today, and is noted here only to record that it was considered distinct.
* **Liveness monitoring** via the mechanism proposed in Issue #133.

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is
more permissive.
