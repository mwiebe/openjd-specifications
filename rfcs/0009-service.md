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
signal that gates the scheduling of the Tasks in its scope. The template names the ports; the
scheduler chooses the actual port numbers and addresses when it places the Service, and every entity
in the Service's scope reads them through a new `Service.*` format-string scope.

A Service can also be supplied from outside the Job: an Environment Template may define a
`services:` list alongside or instead of its `environment:`, so that a scheduler can start per-Job
Services for every Job submitted through a queue, exactly as it applies queue Environments today.
An Environment in the same document can reference the Services' endpoints to configure Tasks for
them, for example by setting environment variables, so Job Templates use such **external Services**
the same way they use queue Environments, without declaring them. Services run inside the
Environments of their scope, and a new `runScope` property on `<Environment>` says which kinds of
Session an Environment applies to. This RFC also adds to Environment
Templates the `extensions` key that Job Templates already have.

The RFC also reserves a new standard attribute capability, `attr.worker.preemptible`, so that a
Service (or any Step) can require placement on capacity that will not be reclaimed mid-scope.

## Basic Examples

### A Valkey store shared across a Job's Tasks

This job starts a single [Valkey](https://valkey.io/) in-memory data store before scheduling any
Task, and every Task in the Job connects to it to cache and coordinate intermediate results. The
template declares one named port, `main`, but no port number: the scheduler picks a free port on
whatever host it places the Service on, and hands the number to both sides. The service process
reads it as `Service.Cache.main.port` (with the interface to bind as
`Service.Cache.main.bindAddress`), and Tasks, which may be on other hosts, read the same number as
`Service.Cache.main.port` alongside the address to connect to, `Service.Cache.main.connectAddress`.
An author who needs a specific port number can request one with `port:` on the port entry. If the
service process dies the scheduler relaunches it, and because a cache can be repopulated, completed
Tasks keep their results (`completedTasks: KEEP`).

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

### A queue-supplied cache with client configuration

A scheduler can supply Services to every Job submitted through a queue by attaching an Environment
Template that defines a `services:` list. The same document can also define an `environment:` that
references the Services' endpoints, so that Tasks are configured for them without any Job Template
knowing they exist. Here a queue administrator attaches a Valkey store and an Environment
that publishes its address through environment variables; the `extensions` key on the Environment
Template is new in this RFC.

```yaml
specificationVersion: "environment-2023-09"
extensions: [SERVICE, FEATURE_BUNDLE_1]
parameterDefinitions:
  - name: CacheMemoryMiB
    type: INT
    default: 8192
services:
  - name: Cache
    description: "A per-Job Valkey store shared by all of the Job's Tasks."
    hostRequirements:
      amounts:
        - name: amount.worker.memory
          min: "{{ Param.CacheMemoryMiB }}"
    ports:
      - name: main
    restartPolicy:
      maxAttempts: 3
      completedTasks: KEEP
    script:
      actions:
        onRun:
          command: valkey-server
          args:
            - "--port"
            - "{{ Service.Cache.main.port }}"
            - "--bind"
            - "{{ Service.Cache.main.bindAddress }}"
            - "--maxmemory"
            - "{{ Param.CacheMemoryMiB }}mb"
environment:
  name: CacheClient
  description: "Tells every Task where the Job's Valkey store is."
  # This Environment references the Service, so it applies to Task Sessions only;
  # it would be meaningless in the Service's own Session.
  runScope: [TASK]
  variables:
    VALKEY_HOST: "{{ Service.Cache.main.connectAddress }}"
    VALKEY_PORT: "{{ Service.Cache.main.port }}"
```

Every Task in every Job submitted through the queue starts with `VALKEY_HOST` and `VALKEY_PORT` set,
whether or not its Job Template has heard of the Service. (`FEATURE_BUNDLE_1` is declared for the
format string in the amount requirement's `min`.) A Job Template
consumes the Service the same way it consumes any queue Environment: through the effects the
Environment has on its Tasks. This Job Template does not use the `SERVICE` extension at all.

```yaml
specificationVersion: "jobtemplate-2023-09"
name: "Frame Processing With Queue Cache"

steps:
  - name: ProcessFrames
    parameterSpace:
      taskParameterDefinitions:
        - name: Frame
          type: INT
          range: "1-100"
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
            # VALKEY_HOST and VALKEY_PORT are set by the queue's CacheClient environment.
            process-frame --frame {{ Task.Param.Frame }} \
              --valkey-host "$VALKEY_HOST" --valkey-port "$VALKEY_PORT"
```

### Execution order

For a Job with one `jobService` `S`, one `jobEnvironment` `E`, and one Step with Tasks `T1..Tn`
running in two Sessions on two Worker Hosts, the scheduler proceeds:

```
Service host                   Worker host A            Worker host B
------------                   -------------            -------------
E.onEnter
  S.onEnter
  S.onRun (starts)
  S readiness probe -> READY
                               E.onEnter                E.onEnter
                                 T1.onRun                 T3.onRun
                                 T2.onRun                 T4.onRun
                               E.onExit                 E.onExit
  S.onRun (canceled)
  S.onExit
E.onExit
```

The Service runs in a Session of its own on the service host, inside the same Environment `E` that
the Tasks run in. No Task in `S`'s scope is scheduled until `S` is READY. `S` outlives every
Session that uses it and is stopped only when the Job has no remaining Tasks that could run.

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
   Template author having to know how it is provisioned. An Environment in the same attachment
   can publish the endpoint to Tasks as environment variables or a configuration file, so even
   Job Templates written before the Service existed can use it. This is the Service analogue of
   the queue Environments that schedulers already apply from Environment Templates.
6. **Rendezvous for distributed compute.** Frameworks such as MPI and Dask consist of a set of
   worker processes that must find each other and a small coordinator they find each other
   through: a PMIx server for MPI ranks, the scheduler process for Dask workers. The coordinator
   is a Service; the workers are the Tasks of a Step, each connecting to the coordinator's
   endpoint and taking its identity (its rank) from a Task parameter. Running these workloads
   well additionally requires that the Step's Tasks be scheduled concurrently, which is outside
   this RFC; see [Future Work](#future-work).

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
| **Service Session** | The Session on the service host in which a Service's actions run: a working directory, the Environments of the Service's scope entered around the Service's actions, and the Service's actions themselves. It is a Session as described in *How Jobs Are Run*, containing one Service instead of Tasks. |
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
+ jobServices: [ <Service>, ... ] # @optional @extension SERVICE
  steps: [<StepTemplate>, ...]
```

New property:

* *jobServices* — An ordered list of the Services required by Tasks in the Jobs created by this
  Job Template. Each Service is started, in list order, before any Task of the Job is scheduled and
  is stopped, in reverse order, once no Task of the Job remains to be run. See
  [`<Service>`](#service). Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. Maximum number of elements: 10.
    3. No two Services in this list may have the same value for the `name` property.
    4. The Services defined in this list must not have the same `name` as a Service defined in any
       Step within the same Job Template.

Extension list addition to item 3 (*extensions*): `SERVICE` (this RFC).

#### Environment Template

> A modification to [`1.2. Environment Template`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#12-environment-template)

```diff
  specificationVersion: environment-2023-09
+ $schema: <string> # @optional
+ extensions: [ <ExtensionName>, ... ] # @optional
  parameterDefinitions: [ <JobParameterDefinition>, ... ] # @optional
- environment: <Environment>
+ environment: <Environment> # @optional
+ services: [ <Service>, ... ] # @optional @extension SERVICE
```

New and changed properties:

* *$schema* — Ignored, as on the Job Template. Allowed for compatibility with JSON-editing IDEs.
* *extensions* — If provided, a non-empty list of extensions that this document uses, with the
  same available names as the Job Template. An extension listed here applies to the content of
  this document only; a Job Template does not need to list the extensions used by the Environment
  Templates a scheduler applies to it, and vice versa. Environment Templates have
  used this key informally since RFC 0002 (RFC 0008's example declares `WRAP_ACTIONS` and `EXPR`
  on one); this RFC makes it part of the schema.
* *environment* — Now optional. The definition of an Environment that the Environment Template
  defines. When *services* is also provided, format strings within the Environment may reference
  those Services' endpoints via `Service.<name>.<port>.port` and
  `Service.<name>.<port>.connectAddress`.
* *services* — An ordered list of Services that the Environment Template defines, with the same
  constraints as a Job Template's `jobServices`: at least one element, at most 10, unique `name`s,
  and a Service may reference only Services earlier in the list. At least one of *environment* or
  *services* must be provided.

**Services from Environment Templates.** An Environment Template that defines *services* defines
Services that the scheduler applies to every Job Template submitted through it, just as its
*environment* defines an Environment applied to every submission. From the Job Template's point of
view these are **external Services**. External Services are applied per Job: each Job gets its own
instance of each, with Job scope, exactly as if the Services had appeared in the Job Template's
`jobServices`. A Service defined this way does not outlive the Job; a process shared across many
Jobs is infrastructure outside the scope of this specification.

When a document defines both, the entities compose as they would in a Job Template: the Services
are Job-scoped and are READY before any Session starts, and the Environment is Session-scoped and
may reference any of the Services' `port` and `connectAddress` values in its *variables*, actions,
and embedded files. This is how an attachment publishes a Service to Tasks that do not reference
`Service.*` themselves — by setting environment variables or writing a configuration file that the
Tasks' tools already read.

A `Service.*` reference within an Environment Template MUST resolve to a Service in the same
document's *services* list (and, within that list, to an earlier Service, as in `jobServices`).
References to Services defined in other documents are not permitted, so every `Service.*`
reference in every template is checkable without knowledge of the scheduler's configuration.

A Job Template never references an external Service. Its `Service.*` references MUST resolve to
Services it declares itself; an external Service reaches a Job's Tasks only through the effects of
the Environment defined alongside it (environment variables, files written to the Session working
directory, and so on), exactly as a queue Environment reaches them today. This keeps every
`Service.*` reference in a Job Template checkable by `openjd check` without knowledge of the queue,
and means a Job Template needs neither the `SERVICE` extension nor any declaration to benefit from
a queue-supplied Service.

When a submission combines a Job Template with one or more Environment Templates:

1. The external Services are ordered as the scheduler orders the Environment Templates, and within
   one template in its *services* order, and are placed before every Service in the Job Template's
   `jobServices`; the attached Environments are placed in `jobEnvironments` as today. The combined
   `jobServices` list is started and stopped as a single list.
2. The `name` of an external Service MUST NOT equal the `name` of any other external Service, nor
   of any Service in the Job Template's `jobServices` or any Step's `stepServices`. The submission
   MUST be rejected on a collision. Queue operators SHOULD give external Services names that Job
   Template authors are unlikely to choose (for example, a studio prefix), since a collision makes
   an existing Job Template unsubmittable.

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

#### `<Environment>`

> A modification to [`4. <Environment>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#4-environment)

```diff
  name: <EnvironmentName>
  description: <Description> # @optional
+ runScope: [ <RunScopeName>, ... ] # @optional @extension SERVICE
  script: <EnvironmentScript> # @optional
  variables: <EnvironmentVariables> # @optional
```

New property:

* *runScope* — The kinds of Session this Environment is entered in. `<RunScopeName>` is one of:
    * `TASK` — Sessions that run Tasks. This is the only kind of Session that exists without the
      `SERVICE` extension.
    * `SERVICE` — Service Sessions, in which a Service's actions run (see
      [Services run inside Environments](#services-run-inside-environments)).

  If not provided, the Environment is entered in every kind of Session (`[TASK, SERVICE]`), so
  existing Environments that provision software apply to Services without modification.
  Constraints:
    1. Minimum number of elements: 1. Maximum: one of each defined name; no duplicates.
    2. An Environment whose *runScope* includes `SERVICE` MUST NOT reference any `Service.*` value:
       a Service Session may begin before any Service other than its own has an endpoint, and its
       own endpoint is not meaningful to an Environment that provisions it. Only Environments with
       `runScope: [TASK]` may reference `Service.*`, and the Environment that configures Tasks to
       use a Service (see the [queue cache example](#a-queue-supplied-cache-with-client-configuration))
       is exactly such an Environment.

  Future extensions may define additional names; a scheduler MUST reject a name it does not
  recognize.

> A modification to [`4.3. <EnvironmentActions>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#43-environmentactions)

```diff
  onEnter: <Action> # @optional
  onWrapEnvEnter: <Action>    # @optional @extension WRAP_ACTIONS
  onWrapTaskRun: <Action>     # @optional @extension WRAP_ACTIONS
  onWrapEnvExit: <Action>     # @optional @extension WRAP_ACTIONS
+ onWrapServiceEnter: <Action>          # @optional @extension WRAP_ACTIONS SERVICE
+ onWrapServiceRun: <Action>            # @optional @extension WRAP_ACTIONS SERVICE
+ onWrapServiceReadinessCheck: <Action> # @optional @extension WRAP_ACTIONS SERVICE
+ onWrapServiceExit: <Action>           # @optional @extension WRAP_ACTIONS SERVICE
  onExit: <Action> # @optional
```

New properties, available only when both `WRAP_ACTIONS` (RFC 0008) and `SERVICE` are in use:

* *onWrapServiceEnter*, *onWrapServiceRun*, *onWrapServiceReadinessCheck*, *onWrapServiceExit* — When
  this Environment is a wrapping Environment in a Service Session, these run instead of the
  Service's *onEnter*, *onRun*, *onReadinessCheck*, and *onExit* respectively, exactly as
  `onWrapTaskRun` runs instead of a Task's `onRun`. Each receives the wrapped action's fields
  through the same `WrappedAction.*` variables RFC 0008 defines, plus `WrappedService.Name`, the
  `name` of the Service being wrapped (the analogue of `WrappedEnv.Name` and `WrappedStep.Name`).

RFC 0008's rules extend to the new group as follows:

1. *All-or-nothing, per group.* A wrapping Environment (one that defines `onWrapEnvEnter`,
   `onWrapTaskRun`, and `onWrapEnvExit`) whose *runScope* includes `SERVICE` MUST also define all
   four `onWrapService*` hooks, since it will wrap Service Sessions. One with `runScope: [TASK]`
   MUST NOT define any of them. An Environment MUST NOT define an `onWrapService*` hook without
   also defining the three RFC 0008 hooks.
2. *Nothing to replace.* `onWrapServiceEnter`, `onWrapServiceReadinessCheck`, and
   `onWrapServiceExit` run only for a Service that defines the corresponding action. Every Service
   defines *onRun*, so `onWrapServiceRun` runs for every wrapped Service.
3. *Single layer.* A Service Session contains at most one wrapping Environment, as any Session
   does.
4. *Stdout.* As in RFC 0008, the runtime scans the wrap script's stdout, not the wrapped
   process's. A wrapped *onEnter*'s `openjd_env` messages and a wrapped *onRun*'s
   `openjd_service_ready` message are therefore honored when the wrap script forwards the wrapped
   process's stdout verbatim, which RFC 0008 already requires.
5. *Failure.* A failed `onWrapServiceRun` is an instance failure, a failed `onWrapServiceEnter` is a
   start failure, and a failed `onWrapServiceExit` is an *onExit* failure, exactly as the wrapped
   action's failure would be.

A wrapping Environment's own *onEnter* and *onExit* are never wrapped, in a Service Session as in
any other. The four hooks are the price of RFC 0008's one-hook-per-action-kind pattern; the
unified `onWrapAction` hook that RFC 0008 lists as future work would replace all seven with one.

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
3. *let* — An ordered list of expression bindings evaluated once, at job creation, like a
   `<StepTemplate>`'s *let*. Bindings may reference `Param.*`, `RawParam.*`, `Job.Name`, and (for a
   Step Service) `Step.Name` and the Step's bindings, but not `Session.*` or `Service.*`, which are
   not known until the Service is placed. Bound names are available in *hostRequirements*,
   *variables*, and *script*. Available with the `EXPR` extension.
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
7. *restartPolicy* — What the scheduler does when the Service's `onRun` action exits before the
   scope ends. If not provided, defaults to `{ maxAttempts: 0, completedTasks: RERUN }`: the
   Service is never relaunched, and its exit fails the scope. See
   [`<ServiceRestartPolicy>`](#servicerestartpolicy).
8. *variables* — A set of environment variable name/value pairs, with the values being Format
   Strings resolved when the Service is started, that are set in the process environment of every
   action of the Service's *script*, including *onReadinessCheck*. This is the declarative
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
   refers to the Service Session's own working directory on the service host, not to the Session of
   any Task.
3. `Service.File.<name>` — The filesystem location of an embedded file defined within this
   Service's *script*. See [`<ServiceScript>`](#servicescript).
4. `Service.<name>.<port>.*` — The endpoint of this Service's own ports, and of any Service
   earlier in the same list (for a Step Service, also any Job Service). See
   [The `Service.*` scope](#the-service-scope).
5. `Step.Name` — Available in a Step Service only. Requires the `EXPR` extension.
6. Names bound by `let` in the enclosing `<StepTemplate>` (for a Step Service), in the `<Service>`,
   and in the `<ServiceScript>`. Available with the `EXPR` extension.

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
      host to the allocated `port` on the loopback interface, or on `bindAddress` when it is not a
      wildcard address (`0.0.0.0` or `::`, which cannot be connected to on every operating
      system), and is closed immediately. The scheduler retries at an implementation-defined
      interval (recommended: 1 second) until success or timeout.
    * `COMMAND` — The Service is READY once the Service's *onReadinessCheck* action (see
      [`<ServiceActions>`](#serviceactions)) exits with status 0. The Service MUST define
      *onReadinessCheck* when this type is used. The action is run in the Service Session like
      every other Service action, and is re-run every *intervalSeconds* (default: 5) until it
      succeeds, the timeout elapses, or `onRun` exits. Its own `timeout` (default: 30 seconds)
      bounds one invocation. An exit status other than 0 is "not yet ready", never a failure of
      the Service.
    * `STDOUT` — The Service is READY once its `onRun` action writes a line matching the regular
      expression `^openjd_service_ready(: .*)?$` to stdout. The optional message after the colon
      has no functional purpose but MAY be surfaced in UIs. This is consistent with the existing
      `openjd_*` stdout protocol and is the mechanism of choice for services whose readiness is
      not observable from outside the process.
2. *ports* (`TCP_CONNECT` only) — The names of the ports to probe. Each must be declared in the
   Service's *ports*. Defaults to all of them.
3. *intervalSeconds* (`COMMAND` only) — Seconds to wait between the end of one *onReadinessCheck*
   invocation and the start of the next. Default: 5.
4. *timeoutSeconds* — The maximum time, measured from the start of the `onRun` action, that the
   scheduler waits for the Service to become READY. If exceeded, the instance is treated as failed
   (see [Failure and restart](#failure-and-restart)). Default: 300 seconds.

Readiness applies to every instance: after a restart, the new `onRun` must pass the readiness check
before Tasks in the scope are (re)scheduled.

##### `<ServiceRestartPolicy>`

A `<ServiceRestartPolicy>` is the object:

```yaml
maxAttempts: <nonnegativeinteger> # @optional
completedTasks: enum("KEEP", "RERUN") # @optional
```

Where:

1. *maxAttempts* — The maximum number of times the scheduler will relaunch the Service after an
   instance failure or a start failure (see [Failure and restart](#failure-and-restart)). The
   initial launch is not counted, so the default of 0 means the Service is launched exactly once
   and never relaunched.
2. *completedTasks* — What happens, when a new instance is launched, to Tasks in the Service's
   scope that completed successfully against a previous instance. (Tasks that were running when
   the previous instance failed are always canceled and requeued; Tasks not yet started are simply
   scheduled once the new instance is READY. Only completed Tasks are a choice.)
    * `KEEP` — Completed Tasks remain complete. Use this when the Service holds no state that
      Tasks depend on, or state that Tasks can regenerate (a cache).
    * `RERUN` — Completed Tasks are returned to the scheduling queue and run again. Use this when
      the Service holds state that makes prior results unreliable once lost (a coordinator).
    * Default: `RERUN`. This is the safe default: an author must opt in to keeping results.

If *maxAttempts* is exhausted, or the policy is not provided, an `onRun` exit fails the scope
regardless of *completedTasks*.

##### `<ServiceScript>`

A `<ServiceScript>` is the object:

```yaml
let: <LetBindings> # @optional @extension EXPR
actions: <ServiceActions>
embeddedFiles: [ <EmbeddedFile>, ... ] # @optional
```

1. *let* — An ordered list of expression bindings evaluated once, on the service host, when the
   Service is started; this is the host-context counterpart of the `<Service>`'s *let*, and may
   reference `Session.*`, `Service.File.*`, and in-scope `Service.<name>.<port>.*`. Bound names are
   available in *actions* and *embeddedFiles*.
2. *actions* — The actions to run at the different stages of the Service's lifecycle.
3. *embeddedFiles* — Files embedded into the Service that are materialized to the Service
   Session's working directory before each of the Service's actions runs. The path of an embedded
   file is available as `Service.File.<name>`.

##### `<ServiceActions>`

A `<ServiceActions>` is the object:

```yaml
onEnter: <Action> # @optional
onRun: <Action>
onReadinessCheck: <Action> # @optional
onExit: <Action> # @optional
```

Where:

1. *onEnter* — A one-time setup action run before the first `onRun` of the Service in a Service
   Session. It is an ordinary action, like an Environment's `onEnter`: it runs to completion, its
   `timeout` and `cancelation` apply as for any `<Action>`, and the scheduler cancels it with its
   cancelation method if the scope ends while it is running. A non-zero exit or a timeout is a
   start failure (see [Failure and restart](#failure-and-restart)). Use *onEnter* for setup that
   must survive a relaunch of *onRun* (for example, initializing an on-disk database in the
   Service Session's working directory); it runs once per Service Session, not once per instance.
2. *onRun* — The long-lived action whose process *is* the service. The scheduler starts it, watches
   it for readiness, and expects it to run until canceled. If *onRun* exits for any reason — zero or
   non-zero exit status — before the scheduler cancels it, the Service instance has failed; see
   [Failure and restart](#failure-and-restart). The scheduler stops the Service by canceling *onRun*
   according to its `cancelation` method.
3. *onReadinessCheck* — A short action the scheduler runs repeatedly to decide whether the Service
   is READY, when the Service's `readinessCheck` has type `COMMAND`. Exit status 0 means ready; any
   other status means not yet. MUST be defined when the readiness check type is `COMMAND`, and MUST
   NOT be defined otherwise. See [`<ServiceReadinessCheck>`](#servicereadinesscheck).
4. *onExit* — A cleanup action run after the Service's other actions have stopped for the last
   time in a Service Session, whether the Service stopped normally, failed, or was canceled, and
   whether or not *onRun* was ever launched. It runs to completion. A non-zero exit is reported but
   does not change the outcome of the scope.

All four actions run in the Service Session with the same format-string scopes: `Param.*` and
`RawParam.*`, `Session.*`, `Service.File.*`, the `Service.<name>.<port>.*` values in scope for the
Service, and the names bound by the `<Service>`'s and the `<ServiceScript>`'s `let`. Embedded
files are materialized before each of them runs.

Default timeouts for these actions (extending the table in `<Action>`):

| Action container | Action | Default |
|---|---|---|
| `<ServiceActions>` | `onEnter` | *no timeout* |
| `<ServiceActions>` | `onRun` | *no timeout* (a timeout, if given, is measured from launch and its expiry is an instance failure) |
| `<ServiceActions>` | `onReadinessCheck` | 30 seconds |
| `<ServiceActions>` | `onExit` | 300 seconds (five minutes) |

Implementations MUST watch the stdout of every `<ServiceActions>` action for the standard
`openjd_status`, `openjd_progress`, and `openjd_fail` messages, and the stdout of *onRun* for
`openjd_service_ready` when the readiness check's *type* is `STDOUT`.

**Environment variables within a Service.** Implementations MUST additionally watch the stdout of
*onEnter* for `openjd_env`, `openjd_redacted_env`, and `openjd_unset_env`, with the same syntax
and redaction rules as for an Environment's `onEnter`. A variable set this way is set in the
process environment of every subsequent action of the same Service Session — every instance of
*onRun*, *onReadinessCheck*, and *onExit* — and is retained across relaunches of *onRun* within the
Session. This is how *onEnter* hands values it computes (a generated credential, a discovered
device, a path it created) to the service process, and it is the reason those values survive a
relaunch when *onEnter* is not re-run. Variables set by *onEnter* take precedence over the
Service's declarative *variables*. These messages are ignored when emitted by *onRun*,
*onReadinessCheck*, or *onExit*, and nothing set within a Service is ever propagated to the entities
in the Service's scope.

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

This attribute is not gated by the `SERVICE` extension; it is a reserved standard name that any Step
may require. Workers SHOULD advertise it, since a requirement of `anyOf: ["false"]` matches no host
that does not. Note that the values must be written as strings: in YAML, `anyOf: [false]` is a
boolean, not the string `"false"`. This RFC introduces it because a long-lived Service is the first
entity in the specification whose correctness depends on not being preempted mid-scope.

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
| `Service.File.<name>` | `path` | The filesystem location to which the Service embedded file with key `<name>` has been written. | Within the Service Script actions and embedded files of the declaring Service. |

`Service.<name>.<port>.*` for a Service `<name>` is in scope in:

1. The Service `<name>` itself (all three values).
2. Any Service later in the same `jobServices` or `stepServices` list, and — for a Job Service —
   any Step Service in the Job. A Service cannot reference a Service later in its own list, nor a
   Step Service of a different Step; this ordering rule guarantees the referenced Service is READY
   before the referencing one starts.
3. For a Job Service: every `jobEnvironments` entry whose `runScope` is `[TASK]`, and every
   Step's `stepEnvironments` with `runScope: [TASK]`, `stepServices`, and `script`.
4. For a Step Service: that Step's `stepEnvironments` with `runScope: [TASK]`, later
   `stepServices`, and `script`.

An Environment whose `runScope` includes `SERVICE` is never in scope for any `Service.*` value; see
[`<Environment>`](#environment).

`Service.*` is never in scope in a `hostRequirements` object, neither a Step's nor a Service's.
Host requirements are resolved when the scheduler chooses a host, which for a Step is at job
creation and for a Service is before the Service starts; a Service endpoint is not known at either
point.

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
at the earlier stages. A `<Service>`'s *let* is evaluated at job creation alongside
`<StepTemplate>` bindings; a `<ServiceScript>`'s *let* is evaluated at service execution on the
service host, alongside `<EnvironmentScript>` and `<StepScript>` bindings at task execution.

### Modifications to How Jobs Are Run

> Additions to [How Jobs Are Run](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run)

#### Service lifecycle

A Service is in exactly one of three states, tracked by the scheduler:

| State | Meaning |
|---|---|
| **UNREADY** | No instance is READY. Either no instance has been launched yet, an instance is launched but has not passed its readiness check, or the previous instance failed and a relaunch is pending or in progress. |
| **READY** | The current instance has passed its readiness check and has not exited. |
| **FAILED** | The Service will not be relaunched. Its scope has failed. |

A Job Service is UNREADY from Job creation. A Step Service is UNREADY from the time its Step's
dependencies are satisfied. A Service's actions run in a **Service Session** on a service host: a
working directory, the Environments of the Service's scope whose `runScope` includes `SERVICE`,
and the Service's own actions. Starting a Service means opening a Service Session and, within it,
entering those Environments in order, running *onEnter* if defined, launching *onRun*, and applying
the readiness check. When the check passes the Service is READY.

Rather than prescribe the scheduler's sequencing in detail, this specification states the
constraints a scheduler MUST satisfy; how it satisfies them is its own concern.

**Ordering.**

1. All of a Service's ports are allocated, and `bindAddress` and `connectAddress` determined,
   before any action of its Service Session runs. The Service's own `Service.<name>.*` values are
   therefore resolvable throughout its Session.
2. No action of a Service Session (including the `onEnter` of an Environment it enters) begins
   until every Service earlier in the same list is READY, and, for a Step Service, until every Job
   Service is READY. This is what makes forward-only references sound: a referenced Service is
   READY before the referencing Service's Session starts.
3. No Task of a Step is scheduled until every Job Service and every Step Service of that Step is
   READY. Once they are, Tasks are scheduled exactly as today, with no change to how Sessions are
   formed; in particular Step Services do not affect which Tasks may share a Session.
4. The order in which Services are *stopped* is the reverse of the order in which they were
   started.

**Sessions.**

5. A Service has at most one live Service Session at a time, and at most one instance (a launched
   *onRun*) within it.
6. A Service Session ends when the Service's scope completes (the Job or Step has no Task that
   could still run, whether because every Task completed or because the scope failed or was
   canceled), when the Service is relocated, or when the Session fails to start (see below). A
   Service whose scope completes MUST have its Session ended whatever its state, including UNREADY
   and FAILED, so that partial state is cleaned up.
7. Before a Service Session ends: any running action is canceled with its own `cancelation`
   method; *onExit* runs if it is defined and any action of the Service has run; and every
   Environment entered is exited in reverse order, as at the end of any Session. Then the working
   directory is deleted and the host's allocated amounts and ports are released.
8. Constraint 7 does not apply to a host the scheduler has lost (see below). Nothing is run or
   awaited there.
9. A Service that is started again after its Session has ended — after relocation, after a start
   failure, or because a Job Service `RERUN` returned its Step to pending — begins a new Service
   Session: new host selection, new ports, new working directory, Environments re-entered, *onEnter*
   re-run.

**Timing.**

10. A scheduler MAY start a Service at any time consistent with the ordering constraints, including
    lazily, when it is first prepared to schedule a Task in the Service's scope. It MAY decline to
    start a Service whose scope will schedule no Task (for example, a Job canceled before any Task
    could be scheduled). Once started, a Service is kept READY until its scope completes or it
    fails.

#### Services run inside Environments

A Service Session enters the Environments of the Service's scope whose `runScope` includes
`SERVICE`, in the same order a Session for a Task in that scope would enter them: a Job Service
enters the Job's `jobEnvironments` (including any attached from Environment Templates, in the
scheduler's order); a Step Service enters the Job's `jobEnvironments` followed by its Step's
`stepEnvironments`. Environments with `runScope: [TASK]` are skipped. The Environments are entered
before the Service's *onEnter* and exited after its *onExit*. Environment variables set by an
Environment's `variables` or `openjd_env` apply to every action of the Service, with the Service's
own *variables* taking precedence over them and variables set by the Service's *onEnter* taking
precedence over both. This is what lets a Service reuse the same Conda, Rez, or container
Environments that provision software for Tasks, without those Environments knowing about
Services at all.

Because an Environment entered in a Service Session cannot reference `Service.*` (see
[`<Environment>`](#environment)), nothing in a Service Session depends on the endpoint of any
Service other than those the Service itself references, and those are READY before the Session
starts (ordering constraint 2).

Under the `WRAP_ACTIONS` extension, a wrapping Environment whose `runScope` includes `SERVICE`
wraps the Service's actions with its `onWrapService*` hooks (see
[`<Environment>`](#environment)). The Service's process then runs inside the container or under
the remote launcher that the wrapping Environment provides, without the Service knowing it.

An Environment entered for a Service runs its `onEnter` once per Service Session, on the service
host, in addition to once per Task Session. Authors of Environments with expensive `onEnter`
actions should expect this, as they already do for Sessions on many hosts.

#### Failure and restart

Two kinds of failure lead to the restart decision:

* An **instance failure** occurs when *onRun* exits (with any status) before the scheduler cancels
  it, when the readiness check times out, or when the scheduler loses the service host.
* A **start failure** occurs when a Service Session fails before *onRun* is launched: an
  Environment's `onEnter` fails, or the Service's *onEnter* exits non-zero or times out. The
  Session ends (constraint 7 applies: *onExit* runs if *onEnter* ran, Environments are exited).

On either failure the Service becomes UNREADY and the scheduler:

1. Cancels every Task in the Service's scope that is currently running. Each such Task is
   returned to the queue and will run again once the Service is READY; this is not counted as a
   Task failure.
2. Cancels *onRun* if it is still running (the readiness timeout case).
3. If the number of relaunches so far is less than `restartPolicy.maxAttempts`:
    1. If `restartPolicy.completedTasks` is `RERUN`, returns every successfully completed Task in
       the scope to the queue.
    2. Relaunches the Service. After an instance failure on a host that is still available, the
       scheduler MAY relaunch *onRun* within the existing Service Session (same working directory,
       same ports, *onEnter* not re-run), which preserves the state *onEnter* established; or it
       MAY relocate. After a start failure or host loss it MUST begin a new Service Session
       (constraint 9). The new instance must pass the readiness check before the Service is READY
       again.
4. Otherwise, the Service becomes FAILED and its scope fails: a failed Job Service fails the Job,
   a failed Step Service fails the Step. The Service Session is then ended per constraint 6.

**Relocation.** Relaunching in a new Service Session on a different host is relocation. It counts as
one relaunch, is governed by `restartPolicy` including `completedTasks`, and changes every
`Service.<name>.*` value. Because entities in the scope resolve `Service.*` at task execution time
on the worker host, a Task scheduled after relocation sees the new endpoint automatically; a Task
that was running during the relocation was canceled in step 1 and re-resolves when it runs again.
When relocating away from a host that is still available, constraint 7 applies to the old Session
in full.

**Host loss.** A scheduler loses a service host when it determines, by its own means (a missed
heartbeat, a preemption notice, an instance termination), that the host is gone or unreachable.
Host loss is an instance failure. Nothing further can run on the lost host, so the scheduler MUST
NOT wait for *onExit*, Environment exits, or working-directory cleanup there; it releases the
host's allocations and proceeds directly to the restart decision above. If the host later
reappears, the scheduler SHOULD terminate any surviving service process and remove the working
directory, but the Service's state is not affected either way. Because a lost host takes the
Service's state with it, authors of stateful Services SHOULD require non-preemptible capacity with
`attr.worker.preemptible`, and SHOULD choose `completedTasks: RERUN` unless the state can be
regenerated.

**`RERUN` and Step dependencies.** A Step Service's scope is a single Step, and a Step Service
cannot fail after its Step completes, so `RERUN` on a Step Service requeues only that Step's Tasks
and never affects another Step. A Job Service's scope is the whole Job, so `RERUN` on a Job
Service returns every completed Task in the Job to the queue: every Step returns to a pending
state, Step dependencies are resolved again from scratch, and a Step Service that had been stopped
because its Step completed is started again, in a new Service Session, when its Step next becomes
schedulable. Steps whose Tasks were running are handled by step 1 above.

**Interaction with Task failures.** A Task failure never fails a Service. A Task that fails
because it could not reach a Service that the scheduler still considers READY is an ordinary
Task failure. Authors SHOULD prefer the `COMMAND` or `STDOUT` readiness check when a plain TCP
connect is not a sufficient indicator that the service can do useful work.

#### Validation

Everything this RFC adds is checkable when a template is validated on its own, with one
exception. At template validation, an implementation MUST check:

1. Every `Service.*` reference resolves to a Service and port declared in the same document, is
   used only where the [scope rules](#the-service-scope) permit, and (for a reference from a
   Service) refers to that Service or one earlier in the start order.
2. No `Service.*` value appears in any `hostRequirements`, in a `<Service>`'s `let`, or in an
   Environment whose `runScope` includes `SERVICE`.
3. `runScope` contains only recognized names, without duplicates.
4. `readinessCheck` is consistent with `<ServiceActions>`: `onReadinessCheck` is defined if and
   only if the type is `COMMAND`, and every port a `TCP_CONNECT` check names is declared.
5. Service and port names are valid `<Identifier>`s, not `File`, and unique within their lists.
6. Wrap hooks are complete per group and consistent with `runScope`, per
   [`<Environment>`](#environment).

The one check that requires the combined Job is the external-Service name collision rule in
[Environment Template](#environment-template), because it compares names across documents that
only the scheduler sees together. It is performed at submission.

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
  action of a Service whose readiness check *type* is `STDOUT`. Indicates that the service is
  accepting connections on all of its declared ports. Emitting it more than once has no additional
  effect; emitting it from any other action is ignored. When `onRun` is wrapped by
  `onWrapServiceRun`, the line is recognized on the wrap script's stdout, as for every `openjd_*`
  message under `WRAP_ACTIONS`.

## Design Choice Rationale

### A new entity rather than extending `<Environment>`

An Environment is defined by *where* it runs: on the host of the Session, entered and exited around
that Session's Tasks. Every one of the four differences in [Motivation](#motivation) — cross-host
placement, scope-long lifetime, endpoint publication, readiness — contradicts that model. Adding
`ports:` and a long-lived `onRun:` to `<Environment>` would produce an entity whose lifecycle
depends on which optional fields are present, which is hard to read and harder to implement. A
distinct `<Service>` that *reuses* Environment's sub-schemas (`variables`, `let`, embedded files,
`<Action>`) gets the familiarity without the ambiguity. (The Environment Template *document* is
reused as a container for an external Service, see below, but the `<Service>` entity itself is
distinct from `<Environment>`.)

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
lists, before anyone could use the feature.

### One document may define Services and an Environment together

Supplying a Service to a queue is usually two things: the process, and the client-side
configuration that lets existing tools find it. Allowing one Environment Template to define both
mirrors how a Job Template composes `jobServices` with `jobEnvironments`, and lets the Environment
publish the Services' endpoints as environment variables or a configuration file, so Job Templates
that were written before the Services existed can use them unchanged. It also removes the need for
a Service reference to cross documents: the Environment that configures clients for a Service lives
next to that Service, and a Service that needs another Service lives in the same list. Requiring
same-document resolution keeps every `Service.*` reference statically checkable, and removes any
dependence of template validity on the order in which a scheduler happens to attach templates.

### Services run inside the scope's Environments

The alternative — a Service is responsible for all of its own setup — was drafted first for
simplicity, but it means a Service cannot reuse the Conda, Rez, or container Environments that
provision software for Tasks, which is the whole reason those Environments exist; every Service
author would reimplement activation inside *onEnter*. Entering the scope's Environments in the
Service Session makes a Service Session a Session in the ordinary sense, so every existing rule
about Environments (ordering, `openjd_env`, cleanup on failure, wrap hooks) applies unchanged, and
an Environment Template written for Tasks works for Services.

### `runScope` rather than an inferred exclusion rule

Two kinds of Environment exist once Services do: those that provision a process (Conda, a
container) and belong in every Session, and those that configure Tasks to *use* a Service and
belong only in Task Sessions. An earlier draft told them apart by inference — an Environment that
referenced a Service was excluded from the Sessions of that Service and of earlier Services — but
that rule depended on the start order, which for attached templates is the scheduler's attachment
order and not knowable from the template. `runScope` makes the distinction a declaration. The
default of "every kind of Session" keeps existing Environments working for Services with no
edits, an Environment that references `Service.*` must say `runScope: [TASK]`, and the check is
local to the document. It also names the concept that future work needs: the set of Sessions an
Environment applies to, to which user-defined names can later be added so that Steps opt in and
out of Environments.

### `onReadinessCheck` is an action of the Service

The readiness command was first modeled as an `<Action>` inside `readinessCheck`. That left it
outside `script:`, with no principled claim on `let` bindings, embedded files, or the same
format-string scopes as the other actions, and the RFC and wiki drifted in describing what it
could reference. Making it a fourth `<ServiceActions>` entry gives every action of a Service one
scope rule and lets probe scripts use embedded files, which they will want to. The `readinessCheck`
object keeps the policy (type, interval, timeout); the action keeps the command.

### Four `onWrapService*` hooks

RFC 0008 defines one wrap hook per kind of wrapped action and an all-or-nothing rule within the
group, so that a wrapper has a complete, explicit picture of what it intercepts. Wrapping a
Service's actions with `onWrapTaskRun` would have been shorter but would have wrapped an `onEnter`
with a hook named for a Task's `onRun`, made `WrappedStep.Name` meaningless, and broken the
one-hook-per-kind pattern. Following the pattern costs four hooks, tied to `runScope` so a
wrapper that never sees a Service Session need not define them. RFC 0008's future unified
`onWrapAction` would collapse all seven hooks into one and is the right long-term answer.

### Start failures consume a restart attempt

A failure before *onRun* launches (an Environment's `onEnter`, or the Service's *onEnter*) is
usually transient in the same way an *onRun* crash is: a package mirror hiccup, a container pull
timeout. Treating it as unconditionally terminal while retrying an *onRun* crash would fail a Job
for exactly the kind of fault `restartPolicy` exists to absorb. So a start failure is handled by
the same restart decision, with the one difference that the relaunch must begin a new Service
Session, because the failed one never reached a usable state.

### `services` is a list while `environment` is singular

An Environment Template has always carried exactly one Environment, and that has been sufficient
because two Environments can be merged into one: their `onEnter` scripts concatenate. Services
cannot be merged. Each is a separate process with its own ports, readiness check, restart policy,
and host requirements, and may need a different host. A queue capability that needs two processes
(a data store and a coordinator, for example) therefore needs a list, and with same-document
reference resolution the list is also the only way for one external Service, or the client
Environment, to reference more than one of them. `environment` stays singular because it is
existing schema and the limitation is workable.

The document's name becomes slightly misleading. That is cosmetic and can be addressed when the
specification version next rolls; a second root element would be permanent surface area.

### Per-Job instances of external Services

An external Service produces one instance per Job rather than one shared by every Job in the
queue. A shared instance would outlive any Job, so `restartPolicy.completedTasks`, the working
directory, host-requirement allocation, and "stop when the scope completes" would all need
different definitions. Keeping external Services Job-scoped means every rule in this RFC applies
to them unchanged, and leaves genuinely queue-long processes where they belong: in infrastructure.

### Job Templates do not reference external Services

Every `Service.*` reference resolves within the document that contains it: a Job Template references
its own `jobServices` and `stepServices`, and an Environment Template references its own `services`.
No template of either kind contains a reference whose validity depends on the scheduler's
configuration, so `openjd check` can validate every reference, and a reader can tell what a template
depends on from the template alone.

A Job Template therefore consumes an external Service the way it consumes a queue Environment today:
through the effects on its Tasks. The Environment defined alongside the Service publishes the
endpoint as environment variables or a file. This is deliberately the same contract queue
Environments already have, and it means Job Templates written before a queue Service existed use it
without modification, and without adopting the `SERVICE` extension. An explicit, typed declaration
of a dependency on an external Service (a stub entry in `jobServices` naming the Service and its
ports) was drafted and removed; see [Rejected
Ideas](#a-typed-reference-to-external-services-from-job-templates).

## Open Questions

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
  that pattern. Slurm's `srun` also demonstrates the direct-launch model for MPI that the
  co-scheduled Step in [Future Work](#future-work) relies on: every rank is a process the scheduler
  launched itself, wired up through a PMI server rather than through `mpirun` and ssh.
* **Nomad** — `service` stanzas with `check` blocks (`tcp`, `script`, `http`) and `restart`
  stanzas with `attempts`. The `KEEP`/`RERUN` distinction has no direct analogue; it is specific to
  batch systems where completed work has a value that a service restart can invalidate.

## Implementation Impact

- **Python (openjd-model)**: New models for `Service`, `ServicePort`, `ServiceReadinessCheck` (a
  discriminated union), `ServiceRestartPolicy`, `ServiceScript`, and `ServiceActions`; new
  `jobServices` / `stepServices` fields gated on the `SERVICE` extension; `$schema`,
  `extensions`, and an optional `services` list (with `environment` now optional) on the
  Environment Template model;
  name-uniqueness validators mirroring the environment ones; a `Service.*` symbol scope with the
  scoping rules in [The `Service.*` scope](#the-service-scope); `Service.File.*` alongside
  `Env.File.*` and `Task.File.*`; `runScope` on `Environment` with the no-`Service.*` check; the
  four `onWrapService*` hooks and `WrappedService.Name` when `WRAP_ACTIONS` is also enabled; and a
  submission-time merge step that orders external Services and checks name collisions. Moderate.
- **Python (openjd-sessions)**: A Service Session variant of the existing session that enters the
  `SERVICE`-scoped Environments, runs `onEnter`, launches `onRun` without awaiting exit, runs
  `onReadinessCheck` on an interval, exposes an "instance exited" callback, and runs `onExit`
  before exiting Environments. Reuses the existing action runner, stdout scanner, cancelation, and
  wrap-hook logic. Moderate.
- **Rust (openjd-rs)**: Same shape as Python across `openjd-model` and `openjd-sessions`.
  Moderate.
- **Schedulers**: This is where most of the work lives. A scheduler must place Services, allocate
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

Rejected in favor of a `services:` property on the existing Environment Template; see
[Design Choice Rationale](#external-services-via-the-environment-template-not-a-new-template-type).

### A typed reference to external Services from Job Templates

An earlier draft let a Job Template declare an `<ExternalServiceRequirement>` in `jobServices` — a
stub with `external: true`, a Service name, and port names — so that the Job Template could use
`Service.<name>.*` for a queue-supplied Service with static validation, and the submission would be
rejected if the queue did not supply a match. It was removed once Environment Templates could carry
a `services:` list and an `environment:` together: the Environment can publish the endpoint to Tasks
by the same means queue Environments already use, no other queue-provided entity requires a
declaration in the Job Template, and the stub added a discriminated union to `jobServices`, a merge
rule, and an exception to the name-collision and forward-reference rules. Unchecked `Service.*`
references resolved at submission time were also considered and rejected, because they would make
`openjd check` unable to catch a typo and hide the dependency from readers.

## Future Work

* **User-defined run scopes.** `runScope` names the kinds of Session an Environment applies to.
  Letting a Step or Service declare additional scope names for its Sessions, and an Environment
  list them in `runScope`, would let Steps opt in to or out of specific Environments — the
  generalization this RFC deliberately stops short of. The grammar here (a list of names, unknown
  names rejected) is chosen so that extension is additive.
* **UDP ports and Unix domain sockets.** A `protocol:` field on `<ServicePort>` is the natural
  extension.
* **Co-scheduled Steps, for systems like MPI and Dask.** With this RFC, an MPI or Dask workload is a
  Step whose Tasks are the ranks or workers, plus a Step Service that is their rendezvous point: a
  PMIx server that the ranks' `MPI_Init` connects to, or the Dask scheduler that the workers and the
  client connect to. Each Task takes its rank from a Task parameter and the coordinator's address
  from `Service.*`; the ranks run the program directly, so each is an ordinary Task with its own log
  and status, and the Step completes when the program exits. (For Dask, one Task runs the client and
  calls `client.shutdown()` when done, which ends the worker Tasks.) What is missing is a guarantee
  that the Tasks run *concurrently*: an MPI job with a fixed world of 30 ranks deadlocks in
  `MPI_Init` if only 10 are scheduled, and a Dask client needs at least some workers alive while it
  runs. A follow-up RFC should add a Step-level co-scheduling constraint — a minimum number of the
  Step's Tasks (or all of them) that the scheduler must be able to run at once before it starts any,
  with all-or-nothing allocation — together with a gang failure policy (cancel the sibling Tasks
  when one fails, and retry as a group) and a requirement that the Tasks' hosts be mutually
  reachable, since ranks talk peer-to-peer once they have found each other. That primitive is useful
  independently of Services, so it is not part of this RFC.
* **Replicated Services.** A `replicas:` count producing `Service.<name>.<port>.connectAddresses`
  as a `list[string]`, for a pool of identical daemons serving a single driver (a distributed
  cache, for example). Deferred until the single-instance model has been exercised; the
  co-scheduled Step above covers the cases where the replicas are themselves the work.
* **Session-scoped Services.** A service that is started once per Session on the Session's host,
  for per-host sidecars. This may be better served by an Environment whose `onEnter` backgrounds a
  process, as today, and is noted here only to record that it was considered distinct.
* **Liveness monitoring** via the mechanism proposed in Issue #133.

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is
more permissive.
