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

This RFC proposes a new `<Service>` entity: a long-lived process with one or more named network
ports that a scheduler starts *before* any Task in its scope is scheduled, keeps running for the
lifetime of its scope (the whole Job, or a single Step), and stops once the scope no longer needs
it. A Job Template declares Services in two new lists, `jobServices` and `stepServices`, alongside
its Environments. The template names the ports; the scheduler chooses the port numbers and addresses
when it places the Service, which may be on a different host than the Tasks that use it. Every
entity in the scope reads them through a new `Service.*` format-string scope. A readiness check
gates Task scheduling in the scope, and a restart policy says what happens when the service process
dies, including whether Tasks that completed against the old instance keep their results. Services
run inside the Environments of their scope, so the same Conda, Rez, or container Environments
provision them and the Tasks. A queue can supply Services to every Job submitted through it by
defining them in an Environment Template.

## Overview

The `SERVICE` extension adds the following, each specified in the sections named:

* **The `<Service>` entity** and the `jobServices` and `stepServices` lists that hold it. A
  Service has named ports, a readiness check, a restart policy, host requirements, and the same
  `variables`, `let`, embedded files, and `<Action>` model as an Environment, with actions
  `onEnter`, `onRun`, `onReadinessCheck`, and `onExit`. See [`<Service>`](#service).
* **The `Service.*` format-string scope**, through which the service process learns what to bind
  and every entity in the scope learns where to connect: `Service.<name>.<port>.port`,
  `bindAddress`, and `connectAddress`, plus `Service.File.*` for the Service's embedded files.
  `SERVICE` requires `EXPR`, and adds the `join_host_port` family of functions so that endpoints
  compose correctly on IPv6 networks. See [The `Service.*` scope](#the-service-scope) and
  [Modifications to the Expression Language](#modifications-to-the-expression-language).
* **Lifecycle, failure, and restart**, stated as constraints a scheduler must satisfy: ordering
  among Services, when Tasks may be scheduled, what happens when an instance fails or its host is
  lost, and the `completedTasks` choice between keeping and rerunning completed work. See
  [Modifications to How Jobs Are Run](#modifications-to-how-jobs-are-run).
* **Services run inside Environments.** A Service Session enters the Environments of its scope, then
  the Service's own `serviceEnvironments` (the analogue of `stepEnvironments`, for provisioning one
  Service differently from its Tasks). A new `runScope` property on `<Environment>` names the kinds
  of Session an Environment applies to, so that an Environment which configures Tasks to *use* a
  Service stays out of Service Sessions. See [`<Environment>`](#environment).
* **External Services.** An Environment Template may define a `services:` list alongside or
  instead of its `environment:`, so a scheduler can start per-Job Services for every Job submitted
  through a queue. Job Templates use such Services the way they use queue Environments, without
  declaring them. The Environment Template schema also gains the `extensions` and `$schema` keys.
  See [Environment Template](#environment-template).
* **Wrap hooks for Services.** Under `WRAP_ACTIONS`, four `onWrapService*` hooks and the
  `WrappedService.*` variables let a wrapping Environment run a Service inside the container or
  launcher it provides. See [`<Environment>`](#environment).
* **`attr.worker.preemptible`**, a standard attribute capability so that a Service, or any Step, can
  require capacity that will not be reclaimed mid-scope. See
  [Host requirements](#host-requirements).

## Basic Examples

### A Valkey store shared across a Job's Tasks

This job starts a single [Valkey](https://valkey.io/) in-memory data store before scheduling any
Task, and every Task in the Job connects to it to cache and coordinate intermediate results. The
template declares one named port, `main`, but no port number: the scheduler picks a free port on
whatever host it places the Service on, and hands the number to both sides. The service process
reads it as `Service.Cache.main.port` and binds `Service.Cache.main.bindAddress`; Tasks, which may
be on other hosts, read the same `Service.Cache.main.port` and connect to
`Service.Cache.main.connectAddress`. If the service process dies the scheduler relaunches it, and
because a cache can be repopulated, completed Tasks keep their results (`completedTasks: KEEP`). The
Valkey binary comes from a Conda environment the Tasks do not need, so the Service declares it as a
`serviceEnvironments` entry, entered only in the Service's own Session.

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [SERVICE, EXPR]
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
    serviceEnvironments:
      - name: ValkeyConda
        script:
          actions:
            onEnter:
              command: bash
              args: ["-c", "conda create -y -p ./valkey-env -c conda-forge valkey && echo openjd_env: PATH=$PWD/valkey-env/bin:$PATH"]
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
trusted: `completedTasks: RERUN` tells the scheduler to requeue them. The service signals readiness
itself on stdout, and does its one-time database initialization in `onEnter` so that a restart of
`onRun` does not wipe the schema.

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [SERVICE, EXPR]
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
                - "{{ join_host_port(Service.Coordinator.api.bindAddress, Service.Coordinator.api.port) }}"
                - "--metrics"
                - "{{ join_host_port(Service.Coordinator.metrics.bindAddress, Service.Coordinator.metrics.port) }}"
                # The coordinator prints "openjd_service_ready: listening" to stdout
                # once its listeners are accepting connections.
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
            - "http://{{ join_host_port(Service.Coordinator.api.connectAddress, Service.Coordinator.api.port) }}"
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
extensions: [SERVICE, EXPR, FEATURE_BUNDLE_1]
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
format string in the amount requirement's `min`.) This Job Template does not use the `SERVICE`
extension at all.

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

### A metrics sink with a UDP ingest port

This Job collects per-frame timings in a StatsD-style metrics sink that every Task reports to over
UDP, and exposes the aggregated results through a TCP HTTP API that the final Step reads. The
`ingest` port declares `protocol: UDP`, so a scheduler that publishes or forwards ports between
hosts forwards it as UDP; the `api` port is TCP by default. The Service gives no `readinessCheck`,
so the default `TCP_CONNECT` probes its TCP ports only: once `api` accepts a connection the sink is
up and `ingest` is bound with it. The UDP port carries the same three values as a TCP port, and the
Tasks use `connectAddress` and `port` without caring which it is.

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [SERVICE, EXPR]
name: "Frame Render With Metrics Sink"

jobServices:
  - name: Metrics
    description: "Collects frame timings over UDP; serves the aggregate over HTTP."
    ports:
      - name: ingest
        protocol: UDP
      - name: api
    restartPolicy:
      maxAttempts: 3
      completedTasks: KEEP
    script:
      actions:
        onRun:
          command: metrics-sink
          args:
            - "--udp-listen"
            - "{{ join_host_port(Service.Metrics.ingest.bindAddress, Service.Metrics.ingest.port) }}"
            - "--http-listen"
            - "{{ join_host_port(Service.Metrics.api.bindAddress, Service.Metrics.api.port) }}"

steps:
  - name: RenderFrames
    parameterSpace:
      taskParameterDefinitions:
        - name: Frame
          type: INT
          range: "1-100"
    script:
      actions:
        onRun:
          command: render-frame
          args:
            - "--frame"
            - "{{ Task.Param.Frame }}"
            - "--statsd-host"
            - "{{ Service.Metrics.ingest.connectAddress }}"
            - "--statsd-port"
            - "{{ Service.Metrics.ingest.port }}"
  - name: Report
    dependencies:
      - dependsOn: RenderFrames
    script:
      actions:
        onRun:
          command: curl
          args:
            - "-sf"
            - "http://{{ join_host_port(Service.Metrics.api.connectAddress, Service.Metrics.api.port) }}/summary"
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
stand up a host, run the daemon, and hard-code (or pass as a Job Parameter) its address. This puts a
dependency of the Tasks outside the job template, which is then no longer a self-contained, [tooling
parseable](https://github.com/OpenJobDescription/openjd-specifications/wiki/Design-Tenets)
description of the work. It also cannot be started and stopped with the Job, so it either runs
permanently or requires bespoke orchestration around every submission.

An `<Environment>` almost fits but differs in four ways that a schema needs to make first-class:

1. **Placement.** An Environment is entered on the host that runs the Tasks. A Service may run on a
   different host and be shared by Tasks spread across many hosts and Sessions.
2. **Lifetime.** An Environment's actions run to completion at Session start and end. A Service's
   main action runs for the *whole* lifetime of its scope, and the scheduler must keep it alive
   across all of the scope's Sessions.
3. **Endpoint publication.** A Service exposes one or more ports. The runtime allocates a concrete
   address and port and makes them discoverable, on any host, both to the service process (so it
   knows what to bind) and to every entity in its scope (so they know where to connect).
4. **Readiness and failure.** A scheduler must know when the service is accepting connections
   before it schedules the Tasks in the Service's scope, and must react when the service dies:
   relaunch it, or fail the scope. Because a service can hold state that Tasks depend on, a
   service author must be able to say whether Tasks that completed against the old instance are
   still valid.

Everything else about a Service — `name`, `description`, `variables`, `let`, embedded files, the
`<Action>` model and its cancelation methods — is deliberately identical to `<Environment>`.

A Service also needs what the Tasks need. The Valkey binary, the coordinator's Python environment,
or the renderer in server mode is provisioned today by the Job's Environments (a Conda or Rez
Environment, or a container under `WRAP_ACTIONS`), and a Service that could not use them would force
every Service author to reimplement that provisioning. Services therefore run inside the
Environments of their scope, and wrapping Environments wrap Service actions as they wrap Task
actions. An Environment that configures Tasks to *use* a Service must then stay out of the Service's
own Session; `runScope` provides that.

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
   Template author having to know how it is provisioned. An Environment in the same attachment can
   publish the endpoint to Tasks as environment variables or a configuration file, so even Job
   Templates written before the Service existed can use it.
6. **Rendezvous for distributed compute.** Frameworks such as MPI and Dask consist of worker
   processes that find each other through a small coordinator: a PMIx server for MPI ranks, the
   scheduler process for Dask workers. The coordinator is a Service; the workers are the Tasks of a
   Step, each connecting to the coordinator's endpoint and taking its identity (its rank) from a
   Task parameter. Running these workloads well also requires that the scheduler launch the Step's
   Tasks only once a threshold number of hosts is available for them, and that the Tasks' hosts can
   reach one another; both are outside this RFC (see [Future Work](#future-work)).

### Backward compatibility

This RFC is additive and gated by the `SERVICE` extension name declared under RFC 0002:

- Schedulers that do not implement `SERVICE` MUST reject templates that list it in `extensions:`.
- A template that uses `jobServices`, `stepServices`, `services`, `runScope`, the `Service.*`
  format-string scope, or the string functions `join_host_port`, `split_host_port`, `is_ipv4`, or
  `is_ipv6` MUST list `SERVICE` in `extensions:`, and a scheduler MUST reject a template that uses
  any of these without declaring the extension.
- No existing field changes meaning. A template with no `SERVICE` extension is unaffected on its
  own. One combination is newly rejected at submission: a wrapping Environment (RFC 0008) whose
  document does not declare `SERVICE`, in a Job that has a Service in that Environment's scope; see
  [Validation](#validation). Either document can be the one at fault: an existing queue wrapper
  template meets a Job Template that declares Services, or a queue gains an external Service and
  every existing Job Template that contains a wrapper becomes unsubmittable to it. In both cases
  that document must declare `SERVICE` and either define the `onWrapService*` hooks or declare a
  `runScope` that excludes `SERVICE`. Queue operators SHOULD account for this before attaching a
  Service to a queue whose Job Templates use wrappers; running the Service outside the wrapper
  instead would silently defeat what the wrapper enforces.
- The `$schema` and `extensions` keys are new to the Environment Template schema (RFC 0002 added
  `extensions` to the Job Template only) and are not gated by `SERVICE`; they formalize what
  `openjd-model` already accepts and what RFC 0008's example assumes. Environment Templates that
  omit both keys are unaffected.
- `SERVICE` requires `EXPR`, as `WRAP_ACTIONS` does: a template that lists `SERVICE` in
  `extensions:` MUST also list `EXPR`, and a scheduler MUST reject one that does not. The
  `Service.*` symbols carry the types listed in [The `Service.*` scope](#the-service-scope), and
  composing an endpoint into a URL relies on the `join_host_port` function (see
  [Address forms](#address-forms)).

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
  Job Template. Each Service is started before any Task of the Job is scheduled and after any
  Service it references is READY, and is stopped once no Task of the Job remains to be run, before
  any Service it references. See
  [`<Service>`](#service). Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. Maximum number of elements: 10.
    3. No two Services in this list may have the same value for the `name` property.
    4. The Services defined in this list must not have the same `name` as a Service defined in any
       Step within the same Job Template.
    5. A Service in this list may reference, through `Service.*`, only itself and Services earlier
       in this list.

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
  Templates a scheduler applies to it, and vice versa.
* *environment* — Now optional. The definition of an Environment that the Environment Template
  defines. When *services* is also provided, format strings within the Environment may reference
  those Services' endpoints via `Service.<name>.<port>.port` and
  `Service.<name>.<port>.connectAddress`.
* *services* — An ordered list of Services that the Environment Template defines, with the same
  constraints as a Job Template's `jobServices`: at least one element, at most 10, unique `name`s,
  and a Service may reference only itself and Services earlier in the list. At least one of
  *environment* or *services* must be provided.

**Services from Environment Templates.** An Environment Template that defines *services* defines
Services that the scheduler applies to every Job Template submitted through it, just as its
*environment* defines an Environment applied to every submission. From the Job Template's point of
view these are **external Services**. Each Job gets its own instance of each, with Job scope,
exactly as if they had appeared in the Job Template's `jobServices`; a process shared across many
Jobs is infrastructure outside the scope of this specification.

A `Service.*` reference within an Environment Template MUST resolve to a Service in the same
document's *services* list (and, within that list, to itself or an earlier Service, as in
`jobServices`). References to Services defined in other documents are not permitted, so every
`Service.*` reference in every template is checkable without knowledge of the scheduler's
configuration.

A Job Template never references an external Service. Its `Service.*` references MUST resolve to
Services it declares itself; an external Service reaches a Job's Tasks only through the effects of
the Environment defined alongside it (environment variables, files written to the Session working
directory, and so on), exactly as a queue Environment reaches them today.

When a submission combines a Job Template with one or more Environment Templates:

1. The external Services are ordered as the scheduler orders the Environment Templates, each
   template's in its *services* order, and placed before every Service in the Job Template's
   `jobServices`; the attached Environments are placed in `jobEnvironments` as today. The combined
   list is started and stopped as one list. The limit of 10 applies to each document's list; the
   scheduler bounds the combined list through how many Environment Templates it attaches.
2. Service names are scoped to the document that declares them. An external Service MAY have the
   same `name` as a Service in another attached Environment Template or in the Job Template; the
   submission is not rejected for it, because every `Service.*` reference resolves within its own
   document. A scheduler MUST keep same-named Services from different documents distinct (for
   example, by qualifying each with its document).

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

* *stepServices* — An ordered list of the Services required by the Tasks of this Step. Each Service
  is started after the Step's dependencies are satisfied, after any Service it references is READY,
  and before any Task of the Step is scheduled, and is stopped once no Task of the Step remains to
  be run, before any Service it references. A Step Service is available only to the Step that
  declares it; it is not available to other Steps, including Steps that depend on this one. See
  [`<Service>`](#service). Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. No two Services in this list may have the same value for the `name` property.
    3. Maximum number of elements: 10.
    4. The Services defined in this list must not have the same `name` as a Job Service defined in
       the same Job Template.
    5. A Service in this list may reference, through `Service.*`, only itself, Services earlier in
       this list, and the Job Services of the Job Template.

  Note: as with Step Environments, the scope of a Step Service's `name` is the Step that defines
  it. Different Steps may each define a Step Service with the same `name`.

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

* *runScope* — The kinds of Session this Environment is entered in. Every Session has exactly one
  kind, and an Environment is entered in a Session if and only if that Session's kind is in the
  Environment's *runScope*. `<RunScopeName>` is one of:
    * `TASK` — Sessions that run Tasks. This is the only kind of Session that exists without the
      `SERVICE` extension.
    * `SERVICE` — Service Sessions, in which a Service's actions run (see
      [Services run inside Environments](#services-run-inside-environments)).

  If not provided, the Environment is entered in every kind of Session, including kinds that a
  later extension defines, so existing Environments that provision software apply to Services
  without modification. An explicit list is exhaustive: the Environment is entered in those kinds
  and no others. An implementation MUST reject a `<RunScopeName>` it does not recognize. An
  extension that defines a new kind of Session MUST give it a `<RunScopeName>`, state which
  `Service.*`-style values (if any) are in scope for Environments entered in it, and, under
  `WRAP_ACTIONS`, name the wrap hooks a wrapping Environment must define for it.
  Constraints:
    1. Minimum number of elements: 1. Maximum: one of each defined name; no duplicates.
    2. An Environment whose *runScope* includes `SERVICE` MUST NOT reference any `Service.*` value:
       a Service Session may begin before any Service other than its own has an endpoint, and its
       own endpoint is not meaningful to an Environment that provisions it. An Environment that
       references `Service.*` must therefore have a *runScope* that excludes `SERVICE`, for example
       `runScope: [TASK]`. This rule does not apply to a Service's `serviceEnvironments`, which are
       entered in exactly one Service Session and may reference that Service (see
       [`<Service>`](#service)).

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
  through the same `WrappedAction.*` variables RFC 0008 defines, plus a `WrappedService.*` group
  (the analogue of `WrappedEnv.Name` and `WrappedStep.Name`):

  | Variable | Type | Description |
  |---|---|---|
  | `WrappedService.Name` | `string` | The `name` of the Service being wrapped. |
  | `WrappedService.PortNames` | `list[string]` | The `name` of each of the Service's ports, in declaration order. |
  | `WrappedService.Ports` | `list[int]` | The allocated port number of each port, in the same order. |
  | `WrappedService.BindAddresses` | `list[string]` | The `bindAddress` of each port, in the same order. |
  | `WrappedService.Protocols` | `list[string]` | The `protocol` of each port (`TCP` or `UDP`), in the same order. |

  The four lists are parallel: index *i* of each describes the Service's *i*-th declared port. A
  wrapper that launches the service process in its own network namespace (a container without
  host networking) uses them to forward every port with its protocol; a Docker wrapper passes
  `{{ flatten([['-p', string(WrappedService.Ports[i]) + ':' + string(WrappedService.Ports[i]) + '/' + lower(WrappedService.Protocols[i])] for i in range(len(WrappedService.Ports))]) }}`
  in its `args`, and one that restricts the host-side binding to the allocated interface passes
  `{{ flatten([['-p', join_host_port(WrappedService.BindAddresses[i], WrappedService.Ports[i]) + ':' + string(WrappedService.Ports[i]) + '/' + lower(WrappedService.Protocols[i])] for i in range(len(WrappedService.Ports))]) }}`.

RFC 0008's rules extend to the new hooks as follows:

1. *The required hooks follow from `runScope`.* A wrapping Environment is one that defines any wrap
   hook. Inner Environments are entered in every kind of Session, so every wrapping Environment MUST
   define `onWrapEnvEnter` and `onWrapEnvExit`. It MUST define `onWrapTaskRun` if and only if its
   *runScope* includes `TASK`, and MUST define all four `onWrapService*` hooks if and only if its
   *runScope* includes `SERVICE`. A Service's `serviceEnvironments` have an effective *runScope* of
   `[SERVICE]`. A template that defines a hook its *runScope* does not call for, or omits one it
   does, MUST be rejected at template validation. For a wrapping Environment in a document that does
   not declare `SERVICE`, see the submission-time check in [Validation](#validation).
2. *Nothing to replace.* `onWrapServiceEnter`, `onWrapServiceReadinessCheck`, and
   `onWrapServiceExit` run only for a Service that defines the corresponding action. Every Service
   defines *onRun*, so `onWrapServiceRun` runs for every wrapped Service.
3. *Single layer.* A Service Session contains at most one wrapping Environment, as any Session
   does.
4. *Stdout.* As in RFC 0008, the runtime scans the wrap script's stdout, not the wrapped
   process's. A wrapped *onEnter*'s `openjd_env` messages and a wrapped *onRun*'s
   `openjd_service_ready` message are therefore honored when the wrap script forwards the wrapped
   process's stdout verbatim, which RFC 0008 already requires. `onWrapServiceReadinessCheck` runs
   concurrently with `onWrapServiceRun`; wrap scripts MUST tolerate this, and the output of both
   wrap scripts is subject to the log attribution rule (see
   [Concurrency with `onRun`](#concurrency-with-onrun)).
5. *Failure.* A failed `onWrapServiceRun` is an instance failure, a failed `onWrapServiceEnter` is a
   start failure, and a failed `onWrapServiceExit` is an *onExit* failure, exactly as the wrapped
   action's failure would be.

6. *Networking.* `bindAddress` and `connectAddress` are determined for the service host's network
   namespace. A wrapper that runs the service process in that namespace (a launcher, `ssh`, or a
   container with host networking) needs no port handling. A wrapper that gives the process its own
   network namespace MUST forward each port in `WrappedService.Ports` with the protocol in
   `WrappedService.Protocols`, so that a connection to `connectAddress`:`port` from any host in the
   scope reaches the process, and MUST ensure the process can bind `bindAddress` inside that
   namespace. A wildcard `bindAddress` (`0.0.0.0`, `::`) works in either case; a loopback
   `bindAddress` reaches a forwarded port only with host networking.

A wrapping Environment's own *onEnter* and *onExit* are never wrapped, in a Service Session as in
any other.

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
serviceEnvironments: [ <Environment>, ... ] # @optional
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
   *serviceEnvironments*, *variables*, and *script*, as a Step's bindings are in its
   `stepEnvironments`. Available with the `EXPR` extension.
4. *hostRequirements* — Requirements on the Worker Host's capabilities that must be satisfied for
   the Service to be placed on the host. Amount capabilities are allocated to the Service Session
   for the lifetime of the Service. This is independent of the *hostRequirements* of any Step
   whose Tasks use the Service.
5. *serviceEnvironments* — An ordered list of Environments entered only in this Service's Session,
   the analogue of a Step's `stepEnvironments`. They are entered, in order, after the Environments
   of the Service's scope (the Job's, and for a Step Service the Step's) and before the Service's
   *onEnter*, and exited in reverse order after its *onExit*. Use them to provision the Service
   differently from the Tasks that use it: a Valkey from one Conda environment while the Tasks run
   in another, or a container that wraps this one Service (see
   [`<Environment>`](#environment)). Because a Service Environment is entered only in the
   declaring Service's Session, its format strings have the Service's own scope: they may reference
   the Service's ports, including `bindAddress`, and the ports of Services earlier in the start
   order, which are READY before the Session begins. Constraints:
    1. No two Environments in this list may have the same `name`, and none may have the `name` of
       a Job Environment or, for a Step Service, of the declaring Step's Step Environments.
    2. *runScope* MUST NOT be provided on a Service Environment; its scope is fixed to the
       declaring Service's Session. For the wrap-hook rule it is treated as `runScope: [SERVICE]`.
6. *ports* — The named ports that the Service exposes. See [`<ServicePort>`](#serviceport).
    1. Minimum number of elements: 1.
    2. Maximum number of elements: 10.
    3. No two ports may have the same `name`.
    4. No two ports with the same `protocol` may have the same `port` number. Two ports MAY have
       the same number when their `protocol` differs: a service that speaks TCP and UDP on one
       number gives that number explicitly on both ports. Ports whose `port` is not provided are
       allocated independently, each in its own protocol's space, and may or may not coincide
       across protocols.
7. *readinessCheck* — How the scheduler determines that the Service is ready to accept traffic. If
   not provided, defaults to `{ type: TCP_CONNECT }` applied to every TCP port. A Service none of
   whose ports is TCP MUST provide a *readinessCheck* of type `STDOUT` or `COMMAND`; a `TCP_CONNECT`
   check, given or defaulted, would have no port to probe. See
   [`<ServiceReadinessCheck>`](#servicereadinesscheck).
8. *restartPolicy* — What the scheduler does when the Service's `onRun` action exits before the
   scope ends. If not provided, defaults to `{ maxAttempts: 0, completedTasks: RERUN }`: the
   Service is never relaunched, and its exit fails the scope. See
   [`<ServiceRestartPolicy>`](#servicerestartpolicy).
9. *variables* — A set of environment variable name/value pairs, with the values being Format
   Strings resolved when the Service is started, that are set in the process environment of every
   action of the Service's *script*, including *onReadinessCheck*. It has the same schema as the
   *variables* property of
   [`<Environment>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#4-environment).
   A Service's *variables* are **not** propagated to the entities in its scope; they receive the
   endpoint through `Service.*` only. Values computed at *onEnter* time are passed to *onRun* with
   `openjd_env` instead (see [`<ServiceActions>`](#serviceactions)).
10. *script* — The actions that the Service runs on its host. See
    [`<ServiceScript>`](#servicescript).

The format string scopes available to format strings within a `<Service>` are:

1. `Param.*` and `RawParam.*` — Values of Job Parameters. `PATH`-typed `Param.*` values are
   available in the Service's *variables* and *script* (actions, embedded files, and
   `<ServiceScript>.let`), where they carry the service host's path mapping applied, exactly as in
   an Environment or Step Script; they are not available in `<Service>.let` or *hostRequirements*,
   which are resolved at job creation before a host is chosen.
2. `Session.*` — Values such as the Service Session's working directory. `Session.WorkingDirectory`
   refers to the Service Session's own working directory on the service host, not to the Session of
   any Task. A Service Session receives path mapping rules like any Session:
   `Session.HasPathMappingRules` and `Session.PathMappingRulesFile` describe the rules for the
   service host, and the rules file is written to the Service Session's working directory.
3. `Service.File.<name>` — The filesystem location of an embedded file defined within this
   Service's *script*. See [`<ServiceScript>`](#servicescript).
4. `Service.<name>.<port>.*` — The endpoint of this Service's own ports, and of any Service
   earlier in the same list (for a Step Service, also any Job Service). See
   [The `Service.*` scope](#the-service-scope).
5. `Job.Name`, and in a Step Service also `Step.Name`.
6. Names bound by `let` in the enclosing `<StepTemplate>` (for a Step Service), in the `<Service>`,
   and in the `<ServiceScript>`.

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
port: <posinteger> | <posintstring> # @optional @fmtstring
protocol: enum("TCP", "UDP") # @optional
```

Where:

1. *name* — The name of the port. It is the second component of `Service.<service>.<port>.*`
   references. Must not be `File`.
2. *port* — If provided, the Service requires this specific port number on its host, in the space
   of the port's *protocol*. If the scheduler cannot provide that port on the chosen host (for
   example, because it is in use), that is a start failure (see
   [Failure and restart](#failure-and-restart)), and relocation to another host may succeed. If not
   provided, the runtime allocates an available port of the port's *protocol*. Range: 1–65535.
   Authors SHOULD omit this and let the runtime allocate, so that multiple Services and Sessions can
   share a host. An allocated port is reserved only in the runtime's own bookkeeping; on a host
   shared with unrelated processes, one of them may bind the port between allocation and *onRun*'s
   bind. When that happens *onRun* exits with an error or never becomes READY, either of which is an
   instance failure, and the restart decision applies (see
   [Failure and restart](#failure-and-restart)).
3. *protocol* — The transport protocol the service process binds the port with, and which the
   scheduler uses for any publishing or forwarding it performs to make the port reachable (a
   container port mapping, a firewall rule, a NAT entry). One of `TCP` or `UDP`. Default: `TCP`.

Numeric fields marked `@fmtstring` — `<ServicePort>.port`, `<ServiceReadinessCheck>.timeoutSeconds`
and `intervalSeconds`, and `<ServiceRestartPolicy>.maxAttempts` — may be given as a format string
whose result is the integer, so that a Job Parameter or a `<Service>.let` binding can supply them.
They are resolved at job creation, with the scope of the `<Service>`'s *let* (item 3). When the
value is a single whole-field expression (`"{{ ... }}"` with no surrounding text) its target type is
`int?`: a `null` result is treated as if the field were not provided, and a non-null result MUST
satisfy the field's range.

Every port, TCP or UDP, is bound and published on the same service host. Unix domain sockets are
out of scope for this RFC (see [Future Work](#future-work)).

##### `<ServiceReadinessCheck>`

A `<ServiceReadinessCheck>` is one of the following objects, discriminated by *type*:

```yaml
type: "TCP_CONNECT"
ports: [ <Identifier>, ... ] # @optional
timeoutSeconds: <posinteger> | <posintstring> # @optional @fmtstring
```

```yaml
type: "COMMAND"
intervalSeconds: <posinteger> | <posintstring> # @optional @fmtstring
timeoutSeconds: <posinteger> | <posintstring> # @optional @fmtstring
```

```yaml
type: "STDOUT"
timeoutSeconds: <posinteger> | <posintstring> # @optional @fmtstring
```

Where:

1. *type* — The probe mechanism:
    * `TCP_CONNECT` — The Service is READY once a TCP connection to each of the listed ports (by
      default, every TCP port the Service declares) succeeds. The connection is made from the
      service host to the allocated `port` on the loopback interface, or on `bindAddress` when it
      is not a wildcard address (`0.0.0.0` or `::`, which cannot be connected to on every operating
      system), and is closed immediately. The scheduler retries at an implementation-defined
      interval (recommended: 1 second) until success or timeout. This type is usable only on a
      Service with at least one TCP port; see *readinessCheck* in [`<Service>`](#service).
    * `COMMAND` — The Service is READY once the Service's *onReadinessCheck* action (see
      [`<ServiceActions>`](#serviceactions)) exits with status 0 while `onRun` is still running.
      The Service MUST define *onReadinessCheck* when this type is used. The action is run in the
      Service Session like every other Service action, concurrently with `onRun`; see
      [Concurrency with `onRun`](#concurrency-with-onrun). Invocations are sequential: the first
      begins once `onRun` has been launched, and each subsequent one begins *intervalSeconds*
      (default: 5) after the previous one ends. Invocations continue until one succeeds, the
      readiness timeout elapses, or `onRun` exits. The action's own `timeout` (default: 30
      seconds) bounds one invocation; an invocation that exceeds it is canceled and counts as
      "not yet ready", as does any exit status other than 0. Neither is ever a failure of the
      Service. Once the instance is READY the action is not run again; readiness is not a
      liveness check (see [Open Questions](#health-monitoring-after-ready)).
    * `STDOUT` — The Service is READY once its `onRun` action writes a line of the form
      `openjd_service_ready: <message>` to stdout, with the same syntax as the other `openjd_*`
      messages. The message has no functional purpose but MAY be surfaced in UIs.
2. *ports* (`TCP_CONNECT` only) — The names of the ports to probe. Each must be declared in the
   Service's *ports* and have `protocol: TCP`; naming a UDP port is a validation error, since a
   UDP port cannot accept a connection. Defaults to every TCP port the Service declares.
3. *intervalSeconds* (`COMMAND` only) — Seconds to wait between the end of one *onReadinessCheck*
   invocation and the start of the next. Default: 5.
4. *timeoutSeconds* — The maximum time, measured from the start of the `onRun` action, that the
   scheduler waits for the Service to become READY. It runs continuously, including while an
   *onReadinessCheck* invocation is in progress. If exceeded, the instance is treated as failed
   (see [Failure and restart](#failure-and-restart)). Default: 300 seconds.

Readiness applies to every instance: after a restart, the new `onRun` must pass the readiness check
before Tasks in the scope are (re)scheduled.

##### `<ServiceRestartPolicy>`

A `<ServiceRestartPolicy>` is the object:

```yaml
maxAttempts: <integer> | <intstring> # @optional @fmtstring
completedTasks: enum("KEEP", "RERUN") # @optional
```

Where:

1. *maxAttempts* — The maximum number of times the scheduler will relaunch the Service after an
   instance failure or a start failure (see [Failure and restart](#failure-and-restart)). The
   initial launch is not counted, so the default of 0 means the Service is launched exactly once
   and never relaunched. Must be 0 or greater.
2. *completedTasks* — What happens, when a new instance is launched, to Tasks in the Service's
   scope that completed successfully against a previous instance, and to Tasks that were running
   at the time. (Tasks not yet started are simply scheduled once the new instance is READY.)
    * `KEEP` — Completed Tasks remain complete, and running Tasks continue, failing and retrying
      on their own if the outage affects them. Use this when the Service holds no state that Tasks
      depend on, or state that Tasks can regenerate (a cache).
    * `RERUN` — Completed Tasks are returned to the scheduling queue and run again, and running
      Tasks are canceled and requeued without counting as a Task failure. Use this when the Service
      holds state that makes prior results unreliable once lost (a coordinator).
    * Default: `RERUN`.

   A `KEEP` Service may be suspended while no Task in its scope can run (timing constraint 10).

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
   Session; it runs once per Service Session, not once per instance. It is an ordinary action, like
   an Environment's `onEnter`: it runs to completion and its `timeout` and `cancelation` apply as
   for any `<Action>`. A non-zero exit or a timeout is a start failure (see [Failure and
   restart](#failure-and-restart)). Use *onEnter* for setup that must survive a relaunch of *onRun*
   (for example, initializing an on-disk database in the Service Session's working directory).
2. *onRun* — The long-lived action whose process *is* the service. The scheduler starts it, watches
   it for readiness, and expects it to run until canceled. If *onRun* exits for any reason — zero or
   non-zero exit status — before the scheduler cancels it, the Service instance has failed; see
   [Failure and restart](#failure-and-restart). The scheduler stops the Service by canceling *onRun*
   according to its `cancelation` method.
3. *onReadinessCheck* — A short action the scheduler runs repeatedly, while *onRun* is running, to
   decide whether the Service is READY, when the Service's `readinessCheck` has type `COMMAND`.
   Exit status 0 means ready; any other status means not yet. MUST be defined when the readiness
   check type is `COMMAND`, and MUST NOT be defined otherwise. It SHOULD be read-only with respect
   to the Service Session's working directory, which it shares with *onRun*. See
   [`<ServiceReadinessCheck>`](#servicereadinesscheck) and
   [Concurrency with `onRun`](#concurrency-with-onrun).
4. *onExit* — A cleanup action run after the Service's other actions have stopped for the last
   time in a Service Session, whether the Service stopped normally, failed, or was canceled, and
   whether or not *onRun* was ever launched. It runs to completion. A non-zero exit is reported but
   does not change the outcome of the scope.

###### Concurrency with `onRun`

*onReadinessCheck* is the only action in the specification that runs while another action of the
same Session, *onRun*, is running. Several existing rules silently assume one action at a time; the
following rules say where that assumption holds and where it does not:

1. **Embedded files.** Materializing embedded files for one action MUST NOT modify any file that a
   still-running action of the same Session was given. Implementations may materialize each
   invocation's files to a distinct location (`Service.File.<name>` may resolve to a different path
   in each action), or not rewrite a file whose content is unchanged, which it always is because a
   Service Session's format-string values are constant for its lifetime.
2. **Stdout.** The stdout of *onReadinessCheck* is captured to the log but no `openjd_*` message on
   it is honored: its result is its exit status. The `openjd_status`, `openjd_progress`, and
   `openjd_fail` messages of the Service come from *onEnter*, *onRun*, and *onExit* only.
3. **Log attribution.** Every line of stdout or stderr captured in a Service Session MUST be
   attributable to the action that produced it. How attribution is recorded is
   implementation-defined: a structured log may carry the action name as a field, and an
   implementation producing a single plain-text log SHOULD tag each *onReadinessCheck* line with the
   action name (for example, `[onReadinessCheck] connection refused`) while leaving *onRun*'s lines
   untagged. The same applies under `WRAP_ACTIONS`, where the wrap script's own output is also part
   of each stream. The runtime adds the tag; a Service's processes need not prefix their own output.
   An implementation MAY suppress or collapse the output of invocations that succeed, provided the
   full output of every invocation that fails or is canceled for exceeding its `timeout` is
   retained.
4. **At most one invocation at a time.** *onReadinessCheck* invocations never overlap one another,
   never run while *onEnter* or *onExit* is running, and never run after the instance is READY.
5. **`onRun` exit wins.** An instance is READY only if *onRun* is still running when the scheduler
   observes a successful invocation. If *onRun* exits while an invocation is in progress, the
   invocation is canceled with its `cancelation` method and its result is discarded; the instance
   has failed. Likewise, an invocation in progress when the Service Session ends is canceled before
   *onExit* runs.
6. **Wrap hooks.** Under `WRAP_ACTIONS`, `onWrapServiceReadinessCheck` runs while `onWrapServiceRun`
   is running. Wrap scripts used in a Service Session MUST tolerate concurrent invocation. Wrappers
   that execute into an existing context (`docker exec`, `ssh`) do; a wrapper that creates a
   per-invocation exclusive resource does not.

Default timeouts for these actions (extending the table in `<Action>`):

| Action container | Action | Default |
|---|---|---|
| `<ServiceActions>` | `onEnter` | *no timeout* |
| `<ServiceActions>` | `onRun` | *no timeout* (a timeout, if given, is measured from launch and its expiry is an instance failure) |
| `<ServiceActions>` | `onReadinessCheck` | 30 seconds |
| `<ServiceActions>` | `onExit` | 300 seconds (five minutes) |

Implementations MUST watch the stdout of *onEnter*, *onRun*, and *onExit* for the standard
`openjd_status`, `openjd_progress`, and `openjd_fail` messages, and the stdout of *onRun* for
`openjd_service_ready` when the readiness check's *type* is `STDOUT`. Messages on the stdout of
*onReadinessCheck* are not honored (see [Concurrency with `onRun`](#concurrency-with-onrun)).
`openjd_fail` has the meaning it has in a Task's *onRun*: it supplies the human-readable reason
reported when the action fails, and does not itself decide success or failure, which the exit status
does. The message accompanies the start failure, instance failure, or *onExit* failure (see [Failure
and restart](#failure-and-restart)) that the action's exit produces.

The environment variables that *How Jobs Are Run* defines for every Session, currently
`OPENJD_SESSION_WORKING_DIR`, are set in the process environment of every action of a Service
Session with their usual meanings; the working directory is the Service Session's own.

**Environment variables within a Service.** Implementations MUST additionally watch the stdout of
*onEnter* for `openjd_env`, `openjd_redacted_env`, and `openjd_unset_env`, with the same syntax and
redaction rules as for an Environment's `onEnter`, including that `openjd_redacted_env` sets the
variable only when the document declares the `REDACTED_ENV_VARS` extension, while its value is
redacted from the log regardless. A variable set this way is set in the process environment of every
subsequent action of the same Service Session — every instance of *onRun*, *onReadinessCheck*, and
*onExit* — and is retained across relaunches of *onRun* within the Session. Variables set by
*onEnter* take precedence over the Service's declarative *variables*. These messages are ignored
when emitted by *onRun*, *onReadinessCheck*, or *onExit*, and nothing set within a Service is ever
propagated to the entities in the Service's scope.

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
may require, and Workers SHOULD advertise it. The values are strings: in YAML, `anyOf: [false]` is a
boolean, not the string `"false"`. This RFC introduces it because a long-lived Service is the first
entity in the specification whose correctness depends on not being preempted mid-scope.

#### Format strings

> A modification to
> [`7.3.1. Value References`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#731-value-references)

##### The `Service.*` scope

New value references, all annotated `@fmtstring[host]` because a Service's endpoint is not known
until the scheduler places the Service, and may change if the Service is relocated:

| Value | Type | Description | Scope |
|---|---|---|---|
| `Service.<name>.<port>.port` | `int` | The port number allocated (or requested) for port `<port>` of Service `<name>`, in the space of that port's `protocol`. The same number is used for binding and for connecting. | Within the declaring Service; within every entity in the Service's scope (see below). |
| `Service.<name>.<port>.bindAddress` | `string` | The interface address the service process MUST bind to so that entities in the Service's scope can reach it. Typically `0.0.0.0` (or `::`) for a distributed scheduler and `127.0.0.1` for a single-host runner. | Within the declaring Service only. |
| `Service.<name>.<port>.connectAddress` | `string` | The hostname or IP address that entities in the Service's scope use to reach it. See [Address forms](#address-forms). | Within every entity in the Service's scope. Also within the declaring Service, for self-reference (e.g. to print its own URL). |
| `Service.File.<name>` | `path` | The filesystem location to which the Service embedded file with key `<name>` has been written. | Within the Service Script actions and embedded files of the declaring Service. |

`Service.<name>.<port>.*` for a Service `<name>` is in scope in:

1. The Service `<name>` itself (all three values), including its `serviceEnvironments`.
2. Any Service later in the same `jobServices` or `stepServices` list, and — for a Job Service — any
   Step Service in the Job, including those Services' `serviceEnvironments` (`port` and
   `connectAddress` only). A Service cannot reference a Service later in its own list, nor a Step
   Service of a different Step.
3. For a Job Service: every `jobEnvironments` entry whose `runScope` excludes `SERVICE`, and every
   Step's `stepEnvironments` whose `runScope` excludes `SERVICE`, `stepServices`, and `script`.
4. For a Step Service: that Step's `stepEnvironments` whose `runScope` excludes `SERVICE`, later
   `stepServices`, and `script`.

`Service.*` is never in scope in a `hostRequirements` object, neither a Step's nor a Service's.

A template that references a `Service.*` value outside the scopes above, or that references a
Service or port name that is not declared, is invalid and MUST be rejected before the Job is
created.

###### Address forms

`bindAddress` and `connectAddress` are each a hostname, an IPv4 literal, or an IPv6 literal, in bare
form: an IPv6 literal is never bracketed. Schedulers SHOULD provide a hostname for `connectAddress`
when one resolves from every host in the Service's scope. Wherever an address and a port are joined
into one string (a URL authority, a `--listen host:port` flag) an IPv6 literal must be bracketed, so
templates MUST compose such strings with `join_host_port` (see [Modifications to the Expression
Language](#modifications-to-the-expression-language)) rather than `{{ addr }}:{{ port }}`.

##### Template processing stages

> A modification to
> [`7.4. Template Processing Stages`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#74-template-processing-stages)

`Service.*` values are unknown at template validation and at job creation, and known at task
execution (and at service execution, for the Service's own actions). They type-check as
`unresolved[int]`, `unresolved[string]`, and `unresolved[path]` at the earlier stages. A
`<Service>`'s *let* is evaluated at job creation, with `<StepTemplate>` bindings; a
`<ServiceScript>`'s *let* is evaluated at service execution on the service host, as
`<EnvironmentScript>` and `<StepScript>` bindings are at task execution.

### Modifications to the Expression Language

> Additions to
> [Expression Language](https://github.com/OpenJobDescription/openjd-specifications/wiki/2026-02-Expression-Language),
> §1.2.2 (built-in symbol types) and §2.2.4 (string functions)

**Service symbols.** `Service.File.<name>` (`path`) joins the file-location symbols, and a "Service
Symbols" table lists `Service.<name>.<port>.port` (`int`), `bindAddress` (`string`), and
`connectAddress` (`string`), all resolved at task or service execution and typed `unresolved[...]`
at earlier stages, with the scopes in [The `Service.*` scope](#the-service-scope).

**Host and port functions.** Four string functions are added. They are available only in documents
that declare `SERVICE`; `EXPR` alone does not provide them:

| Signature | Description |
|---|---|
| `join_host_port(host: string, port: int) -> string` | Join a host and a port as `host:port`, enclosing `host` in square brackets when it is an IPv6 literal (contains a colon and is not already bracketed). Suitable as a URL authority or a `host:port` flag value. |
| `split_host_port(s: string) -> list[string]?` | The inverse: split `host:port` or `[host]:port` into `[host, port]`, with the brackets removed from an IPv6 literal. Returns `null` when `s` has no port, including an unbracketed IPv6 literal such as `2001:db8::5`. Malformed brackets (`[::1`, `[::1]x:80`) are an evaluation error. The port is a string; convert it with `int()`. |
| `is_ipv4(s: string) -> bool` | True if `s` is an IPv4 literal. |
| `is_ipv6(s: string) -> bool` | True if `s` is an IPv6 literal, bracketed or not, with or without a zone identifier (`fe80::1%eth0`). |

`join_host_port("cache.example", 6379)` is `"cache.example:6379"`;
`join_host_port("2001:db8::5", 6379)` is `"[2001:db8::5]:6379"`;
`split_host_port("[2001:db8::5]:6379")` is `["2001:db8::5", "6379"]`. A zone identifier is carried
through verbatim: `join_host_port("fe80::1%eth0", 80)` is `"[fe80::1%eth0]:80"`. These follow Go's
`net.JoinHostPort` and `net.SplitHostPort`, except that a bare IPv6 literal splits to `null` rather
than an error and an already-bracketed host is not bracketed again. Under method syntax,
`addr.join_host_port(port)`.

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
dependencies are satisfied. A Service's actions run in a **Service Session** on a service host.
Starting a Service means opening a Service Session and, within it, entering the Environments of the
Service's scope whose `runScope` includes `SERVICE` in order, running *onEnter* if defined,
launching *onRun*, and applying the readiness check. When the check passes the Service is READY.

A Service Session is always a Session of its own, never one that also runs Tasks.

A scheduler MUST satisfy the following constraints; how it satisfies them is its own concern.

**Ordering.**

1. All of a Service's ports are allocated, and `bindAddress` and `connectAddress` determined,
   before any action of its Service Session runs. The Service's own `Service.<name>.*` values are
   therefore resolvable throughout its Session.
2. No action of a Service Session (including the `onEnter` of an Environment it enters) begins until
   every Service it references through `Service.*` is READY. Services that do not reference one
   another MAY start concurrently. A Service that must start after another it does not otherwise use
   can reference that Service's endpoint anywhere in its *variables* or *script* to express the
   dependency.
3. No Task of a Step is scheduled until every Job Service and every Step Service of that Step is
   READY. Once they are, Tasks are scheduled exactly as today, with no change to how Sessions are
   formed; in particular Step Services do not affect which Tasks may share a Session.
4. A Service is *stopped* before any Service it references is stopped, and every Step Service is
   stopped before any Job Service. Within one list — the combined `jobServices`, or a Step's
   `stepServices` — stopping in the reverse of list order satisfies this, since references are
   forward-only; Services that do not reference one another MAY be stopped concurrently. Step
   Services of different Steps are stopped independently, as their Steps complete.

**Sessions.**

5. A Service has at most one live Service Session at a time, and at most one running *onRun*
   within it. Before *onRun* is launched again in the same Session, the previous *onRun* process
   MUST have exited, whether on its own or because the scheduler canceled it.
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
    lazily, when it is first prepared to schedule a Task in the Service's scope, and MAY decline to
    start a Service whose scope will schedule no Task (for example, a Job canceled before any Task
    could be scheduled). Once started, a Service is kept READY until its scope completes or it
    fails, with one exception. A scheduler MAY **suspend** a Service whose `completedTasks` is
    `KEEP` while no Task in its scope is running or can be scheduled (a paused Job, or a Job waiting
    on a resource). A suspension is not a failure: the Service Session ends as constraint 7
    requires, the Service becomes UNREADY, no restart attempt is consumed, completed Tasks keep
    their results, and the Service is started again in a new Service Session (constraint 9) when the
    scheduler is next prepared to schedule a Task in the scope. A Service whose `completedTasks` is
    `RERUN` MUST NOT be suspended.

#### Services run inside Environments

A Service Session enters the Environments of the Service's scope whose `runScope` includes
`SERVICE`, in the same order a Session for a Task in that scope would enter them: a Job Service
enters the Job's `jobEnvironments` (including any attached from Environment Templates, in the
scheduler's order); a Step Service enters the Job's `jobEnvironments` followed by its Step's
`stepEnvironments`. After those it enters its own `serviceEnvironments`, in order. All are entered
before the Service's *onEnter* and exited, in reverse, after its *onExit*. Environment variables set
by an Environment's `variables` or `openjd_env` apply to every action of the Service, later
Environments taking precedence over earlier ones, with the Service's own *variables* taking
precedence over all of them and variables set by the Service's *onEnter* taking precedence over
both.

Under the `WRAP_ACTIONS` extension, a wrapping Environment whose `runScope` includes `SERVICE`
wraps the Service's actions with its `onWrapService*` hooks, and the inner Environments of the
Service Session with `onWrapEnvEnter` and `onWrapEnvExit` as in any Session (see
[`<Environment>`](#environment)). The Service's process then runs inside the container or under
the remote launcher that the wrapping Environment provides, without the Service knowing it.

An Environment entered for a Service runs its `onEnter` once per Service Session, on the service
host, in addition to once per Task Session.

#### Failure and restart

Two kinds of failure lead to the restart decision:

* An **instance failure** occurs when *onRun* exits (with any status) while the Service's scope
  still has work — a Task that has not completed, or that could still run — other than because the
  scheduler canceled it; when the readiness check times out; or when the scheduler loses the
  service host. An exit observed after the scope has completed is not a failure, whether or not the
  scheduler's cancelation had yet reached the process; the Service Session simply ends.
* A **start failure** occurs when a Service Session fails before *onRun* is launched: a requested
  *port* cannot be allocated on the chosen host, an Environment's `onEnter` fails, or the Service's
  *onEnter* exits non-zero or times out. The Session ends (constraint 7 applies: *onExit* runs if
  *onEnter* ran, Environments are exited).

On either failure the Service becomes UNREADY and the scheduler:

1. If `restartPolicy.completedTasks` is `RERUN`, cancels every running Task in the Service's scope;
   each is returned to the queue to run again once the Service is READY, and this is not counted as
   a Task failure. If it is `KEEP`, running Tasks continue. A Task that needs the Service while it
   is UNREADY, or that holds the endpoint of a relocated instance, fails on its own and is retried
   under the scheduler's ordinary Task retry rules, re-resolving `Service.*` when it runs again; a
   Task that no longer needs the Service completes normally.
2. Cancels *onRun* if it is still running (the readiness timeout case) and waits for it to exit;
   constraint 5 forbids a second *onRun* in the Session before then.
3. If the number of relaunches so far is less than `restartPolicy.maxAttempts`:
    1. If `restartPolicy.completedTasks` is `RERUN`, returns every successfully completed Task in
       the scope to the queue.
    2. Relaunches the Service. After a start failure or host loss it MUST begin a new Service
       Session (constraint 9). After an instance failure on a host that is still available, it MAY
       relaunch *onRun* within the existing Service Session (same working directory, same ports,
       *onEnter* not re-run), preserving the state *onEnter* established; it SHOULD begin a new
       Service Session, which allocates new ports, when the failure may be a port conflict (*onRun*
       exited without ever becoming READY); or it MAY relocate.
4. Otherwise, the Service becomes FAILED and its scope fails: a failed Job Service fails the Job,
   a failed Step Service fails the Step. The Service Session is then ended per constraint 6.

**Relocation.** Relaunching in a new Service Session on a different host is relocation. It counts as
one relaunch, is governed by `restartPolicy` including `completedTasks`, and changes every
`Service.<name>.*` value. When relocating away from a host that is still available, constraint 7
applies to the old Session in full.

**Host loss.** A scheduler loses a service host when it determines, by its own means (a missed
heartbeat, a preemption notice, an instance termination), that the host is gone or unreachable. This
is an instance failure. The scheduler MUST NOT wait for *onExit*, Environment exits, or
working-directory cleanup there; it releases the host's allocations and proceeds to the restart
decision above. If the host later reappears, the scheduler SHOULD terminate any surviving service
process and remove the working directory, but the Service's state is not affected either way.
Because a lost host takes the Service's state with it, authors of stateful Services SHOULD require
non-preemptible capacity with `attr.worker.preemptible`, and SHOULD choose `completedTasks: RERUN`
unless the state can be regenerated.

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

Everything this RFC adds is checkable when a template is validated on its own, except the
`@fmtstring` numeric fields, which are checked when resolved at job creation (item 8), and one check
that needs the combined Job (below). At template validation, an implementation MUST check:

1. Every `Service.*` reference resolves to a Service and port declared in the same document, is
   used only where the [scope rules](#the-service-scope) permit, and (for a reference from a
   Service) refers to that Service or one earlier in the start order.
2. No `Service.*` value appears in any `hostRequirements`, in a `<Service>`'s `let`, or in a Job
   or Step Environment whose `runScope` includes `SERVICE`.
3. `runScope` contains only recognized names, without duplicates, and is not provided on a
   Service Environment.
4. `readinessCheck` is consistent with `<ServiceActions>` and with `ports`: `onReadinessCheck` is
   defined if and only if the type is `COMMAND`; every port a `TCP_CONNECT` check names is declared
   and has `protocol: TCP`; and a Service none of whose ports is TCP has a `readinessCheck` of type
   `STDOUT` or `COMMAND`.
5. Service and port names are valid `<Identifier>`s, not `File`, and unique within their lists;
   Service Environment names are unique within their list and distinct from the Job Environments
   and, for a Step Service, the Step's Step Environments.
6. The wrap hooks an Environment defines are exactly those its `runScope` calls for, per
   [`<Environment>`](#environment).
7. A template that lists `SERVICE` also lists `EXPR`.
8. No two ports of a Service with the same `protocol` have the same `port` number. For a `port`
   given as a format string this is checked when the value is resolved at job creation, as its
   range is.

These checks appear in the wiki as §9.7. One check requires the combined Job, because it relates
documents that only the scheduler sees together, and is performed at submission:

1. A wrapping Environment defined in a document that does not declare `SERVICE` — an attached
   Environment Template, or the Job Template itself — has the default `runScope` (every kind of
   Session) and cannot define the `onWrapService*` hooks. If the combined Job places any Service in
   that Environment's scope, the submission MUST be rejected, naming the document that defines the
   Environment as the cause; see [Backward compatibility](#backward-compatibility) for how this
   arises and the remedy.

#### Placement

Placement is entirely up to the scheduler, subject to the Service's *hostRequirements*. A
single-host runner such as `openjd run` starts the Service alongside the Tasks on the same host with
`bindAddress` and `connectAddress` both on loopback.

Whatever the placement, `connectAddress` and `port`, over the port's `protocol`, MUST reach the
service process from every host in the scope. How the scheduler achieves that (a shared network, an
overlay, port forwarding, a proxy) is not visible in the template.

#### Stdout/Stderr messages

> Addition to the list in *Stdout/Stderr Messages*

* `openjd_service_ready: <message>` where `<message>` is any string — Interpreted only when emitted
  by the `onRun` action of a Service whose readiness check *type* is `STDOUT`, where it indicates
  that the service is accepting traffic on all of its declared ports. Emitting it more than once has
  no additional effect.

## Design Choice Rationale

### A new entity rather than extending `<Environment>`

An Environment is defined by *where* it runs: on the host of the Session, entered and exited around
that Session's Tasks. Every one of the four differences in [Motivation](#motivation) — cross-host
placement, scope-long lifetime, endpoint publication, readiness — contradicts that model. Adding
`ports:` and a long-lived `onRun:` to `<Environment>` would produce an entity whose lifecycle
depends on which optional fields are present, which is hard to read and harder to implement. A
distinct `<Service>` that *reuses* Environment's sub-schemas (`variables`, `let`, embedded files,
`<Action>`) gets the familiarity without the ambiguity.

### `onEnter` / `onRun` / `onExit` rather than a single `onRun`

Two things argue for the three-action form over a single long-lived action:

1. **Restart hygiene.** A restart policy needs a clear statement of *what* is relaunched. With
   only `onRun`, any one-time initialization is re-executed on every restart, which is wrong for
   anything with on-disk state. `onEnter` gives authors a place for setup that must survive a
   restart of the service process; `onExit` gives them a place for cleanup that must run whether
   the process exited cleanly or was killed.
2. **Symmetry with `<Environment>` and `<StepScript>`.** Authors already know `onEnter` / `onExit`
   from Environments and `onRun` from Steps. The only new idea is `onRun`'s definition for a
   Service: the process that is the service, expected to run until canceled.

### `onRun` exiting is always a failure

There is no such thing as a service that has "finished" while its scope still has work. Treating a
zero exit as success would silently leave the Tasks in its scope without their endpoint. Making
*any* exit an instance failure keeps the rule simple; authors who want a service to stop early
should let the scope complete. The rule is tied to the scope having work rather than to the
scheduler's cancelation because there is a window between the scheduler deciding the scope is
complete and its cancelation reaching the process; a service that notices its clients are gone and
exits in that window must not fail a Job whose work has all succeeded.

### Suspension is allowed only for `KEEP` Services

A Job can sit for days paused, throttled, or waiting on a dependency, and a Service kept READY the
whole time holds a host for nothing. Suspending it is free under `completedTasks: KEEP`, where
completed Tasks stand and the Service starts fresh when work resumes; under `RERUN` it would throw
away every completed Task, a choice that is the author's, not the scheduler's. `completedTasks`
already carries exactly this information — whether the Service's state matters to completed work —
so no new field is needed.

### Readiness is declared, not inferred

A scheduler cannot know when an arbitrary process is ready. The three probe types cover the three
places readiness is observable: the network (`TCP_CONNECT`, the zero-configuration default), an
external check (`COMMAND`, for protocol-level health such as a `PING` or an HTTP `/healthz`), and
the process itself (`STDOUT`, for services whose authors can add one print statement).

### Why the protocol is declared on the port

A single-host runner needs only a port number: the process binds whatever protocol it likes and a
Task on the same host reaches it with no help. A distributed scheduler must publish or forward the
port (a container port mapping, a firewall rule, a NAT entry), and every one of those is defined per
protocol. A template silent about the protocol therefore works on one host and fails on a farm,
where the scheduler forwards TCP because it cannot know otherwise. The `protocol` field on
`<ServicePort>` gives the scheduler the one fact it lacks, in the place the port is declared. The
default of `TCP` means a template that never mentions the field declares plain TCP ports, which is
what most services want.

`TCP_CONNECT` is scoped to TCP ports rather than forbidden on any Service that has a UDP port. A
service with a TCP API and a UDP telemetry port beside it is as probeable as any other through its
TCP port; requiring it to supply a `COMMAND` check would be busywork. An all-UDP service satisfies
the `STDOUT` or `COMMAND` check it must declare by printing one line once it has bound.

### `restartPolicy.completedTasks` is per-Service, defaulting to `RERUN`

The two motivating examples pull in opposite directions: a cache loses nothing of consequence when
it restarts, while a coordinator that loses its state has invalidated every result produced against
it. No single behavior is correct, so the policy is per-Service. `RERUN` is the default because it
is the conservative choice: it can only cost time, whereas `KEEP` applied to a stateful service can
produce silently incorrect output.

The same distinction decides what happens to Tasks running at the moment of failure. Under `RERUN`
their partial work depends on the lost state as much as completed work does, so they are canceled
and requeued at no cost to their retry budget. Under `KEEP` canceling them would be heavy-handed: a
brief restart of a cache would cancel every in-flight frame, most of which would have finished
without ever noticing. So they continue; one that does need the Service during the outage fails and
consumes a Task retry, which an author who chooses `KEEP` for a Service that restarts often should
budget for.

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

### Endpoint as three scalar values

`port`, `bindAddress`, and `connectAddress` are separate scalars rather than a single `host:port`
string so they compose with any URL scheme (`http://`, `redis://`) or a bare `--host` / `--port`
pair without string surgery. `bindAddress` is separate from `connectAddress` because they differ in
essentially every deployment (a process binds `0.0.0.0`; clients connect to a hostname), and
conflating them is the most common source of "works on one host, not on two" bugs. The one
composition that needs logic, joining an address and a port when the address may be an IPv6 literal,
is the `join_host_port` function rather than a fourth pre-joined value, so a template never inspects
an address, and the same function serves any host and port pair a template handles.

### Ordered lists with forward-only references

Letting a Service reference only Services *earlier* in its list expresses the dependency order with
no new syntax and no possibility of a cycle, as the ordering of `jobEnvironments` does. Starting is
gated by the references themselves rather than by list position, so Services that are independent
start concurrently; the list order is what makes the reference rule checkable and gives
implementations a total order that always satisfies the stop constraint.

### External Services via the Environment Template, not a new template type

A separate `servicetemplate-2023-09` root element was rejected for its deployment surface. Every
scheduler that supports Environment Templates already has the plumbing to attach a document to a
queue, order the attachments, merge their parameter definitions, and apply them to each submission.
Expressing a Service as a variant of that same document means Services work everywhere Environment
Templates do with no new integration; a second root element would require every implementation to
add a second attachment type, UI, and API, and a rule for interleaving the two lists.

### Service names are scoped to their document

Requiring an external Service's name to differ from every Service in the Job Template would let a
queue operator's choice of name break a Job Template that works everywhere else: the portability
failure the same-document reference rule exists to prevent. Because no `Service.*` reference crosses
a document boundary, global uniqueness buys nothing.

### One document may define Services and an Environment together

Supplying a Service to a queue is usually two things: the process, and the client-side configuration
that lets existing tools find it. Allowing one Environment Template to define both mirrors how a Job
Template composes `jobServices` with `jobEnvironments`. It also removes the need for a Service
reference to cross documents: the Environment that configures clients for a Service lives next to
that Service, and a Service that needs another Service lives in the same list. Requiring
same-document resolution removes any dependence of template validity on the order in which a
scheduler happens to attach templates.

### Services run inside the scope's Environments

Entering the scope's Environments in the Service Session makes a Service Session a Session in the
ordinary sense, so every existing rule about Environments (ordering, `openjd_env`, cleanup on
failure, wrap hooks) applies unchanged, and an Environment Template written for Tasks works for
Services.

### `serviceEnvironments`, the analogue of `stepEnvironments`

Entering the scope's Environments gives a Service the provisioning its Tasks have, but not
provisioning of its own: a Valkey from `conda-forge` while the Tasks use a renderer's Conda
environment, or a container for the one Service that needs isolation. Without a Service-scoped list
the only recourse is a Job Environment that every Service in scope also enters, or folding the
provisioning into *onEnter*, which is what Environments exist to avoid.

### `runScope` rather than an inferred exclusion rule

Two kinds of Environment exist once Services do: those that provision a process (Conda, a container)
and belong in every Session, and those that configure Tasks to *use* a Service and belong only in
Task Sessions. The alternative is inference: an Environment that references a Service is excluded
from the Sessions of that Service and of earlier Services. But that rule depends on the start order,
which for attached templates is the scheduler's attachment order, unknowable from the template.
`runScope` makes the distinction a declaration.

### `onReadinessCheck` is an action of the Service

The alternative is an `<Action>` inside `readinessCheck`. That leaves it outside `script:`, with no
principled claim on `let` bindings, embedded files, or the same format-string scopes as the other
actions. Making it a fourth `<ServiceActions>` entry gives every action of a Service one scope rule
and lets check scripts use embedded files. The `readinessCheck` object keeps the policy (type,
interval, timeout); the action keeps the command.

The check runs while *onRun* is running, which no other action in the specification does. The
alternative, dropping `COMMAND` and having authors wrap their binary in a script that starts it in
the background, polls it, prints `openjd_service_ready`, and waits, pushes onto every third-party
binary the boilerplate OpenJD exists to remove, hides the readiness policy in a shell script, and
does not work for a bare-binary *onRun* without a shell. The rules in [Concurrency with
`onRun`](#concurrency-with-onrun) are the cost of keeping it.

### Four `onWrapService*` hooks

RFC 0008 defines one wrap hook per kind of wrapped action and requires a wrapper to define every
hook it could need, so that it has a complete, explicit picture of what it intercepts. A Service has
four kinds of action, so following the pattern costs four hooks. RFC 0008's future unified
`onWrapAction` would collapse all seven hooks into one and is the right long-term answer.

### Start failures consume a restart attempt

A failure before *onRun* launches (an Environment's `onEnter`, or the Service's *onEnter*) is
usually as transient as an *onRun* crash: a package mirror hiccup, a container pull timeout.
Treating it as terminal while retrying an *onRun* crash would fail a Job for exactly the fault
`restartPolicy` exists to absorb. So a start failure takes the same restart decision; the relaunch
begins a new Service Session only because the failed one never reached a usable state.

### `services` is a list while `environment` is singular

An Environment Template has always carried one Environment, which suffices because two Environments
merge into one: their `onEnter` scripts concatenate. Services cannot merge: each is a separate
process with its own ports, readiness check, restart policy, and host requirements, and may need a
different host. A queue capability that needs two processes (a data store and a coordinator)
therefore needs a list, and under same-document resolution the list is also the only way for an
external Service, or the client Environment, to reference more than one of them. `environment` stays
singular because it is existing schema and the limitation is workable.

The document's name becomes slightly misleading. That is cosmetic and can be addressed when the
specification version next rolls.

### Per-Job instances of external Services

An external Service produces one instance per Job rather than one shared by every Job in the queue.
A shared instance would outlive any Job, so `restartPolicy.completedTasks`, the working directory,
host-requirement allocation, and "stop when the scope completes" would each need a new definition;
Job-scoped, every rule in this RFC applies unchanged.

## Open Questions

### Health monitoring after READY

This RFC detects an instance failure only when `onRun` exits; a hung process that still holds its
port is invisible to it.
[Issue #133](https://github.com/OpenJobDescription/openjd-specifications/issues/133) proposes a
general monitoring mechanism for Actions, of which a Service's `onRun` is an obvious consumer, so
this RFC adds no Service-specific probe that could diverge from it. If #133 lands first, this RFC
should say how its monitor applies to `onRun` (most likely: a monitor failure is an instance
failure).

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

- **Rust (openjd-rs)**: A prototype on the [`service-extension`
  branch](https://github.com/mwiebe/openjd-rs/tree/service-extension) of the `mwiebe/openjd-rs` fork
  passes the `SERVICE` conformance suite that accompanies this RFC and the rest of the 2023-09
  suite. It implements the schema, validation, and job-creation rules in `openjd-model`, the host
  and port functions in `openjd-expr`, the Service Session in `openjd-sessions`, and a single-host
  scheduler for `openjd run` in `openjd-cli`. The prototype's largest single piece was the Service
  Session: the existing Session held one current action, one log stream, and one cancellation
  target, and running `onReadinessCheck` alongside `onRun` needed a second action slot with its own
  cancellation and per-action attribution on captured output.
- **Python (openjd-model, openjd-sessions)**: The Python packages are intended to become bindings
  over openjd-rs. The `v0` releases of `openjd-model-for-python` and `openjd-sessions-for-python`
  would therefore not gain this extension; a `v1` built on the bindings would inherit it, exposing
  the new models, the Service Session, and the `Service.*` scope.
- **Schedulers**: This is where most of the work lives. A scheduler must place Services, allocate
  ports, guarantee cross-host reachability over each port's protocol, gate Task scheduling on
  readiness, track instance failures, implement `KEEP`/`RERUN`, and stop Services at scope end. A
  single-host runner can implement all of this with loopback addresses and a process table, as the
  prototype does; a distributed scheduler needs a networking answer.

No feature here is significantly harder in one language than another. There are no evaluation
performance implications; `Service.*` adds a handful of symbols resolved at task execution time.

## Rejected Ideas

### Passing the endpoint through environment variables

Exporting `OPENJD_SERVICE_<NAME>_<PORT>_ADDRESS` into every Session is rejected on the
Tooling-parseable tenet: environment variables hide data flow from tools that read the template,
whereas `{{ Service.Cache.main.connectAddress }}` is visible. This is the same reasoning recorded
for `OPENJD_SESSION_WORKING_DIR` in *How Jobs Are Run*.

### Services as a kind of Step

A Service could be a Step with a single long-running Task that other Steps depend on. This breaks in
three ways: Step dependencies mean "after it *finishes*", not "while it runs"; a Step's Task has no
way to publish an endpoint; and a Step cannot be stopped when the Steps that depend on it finish.

### A typed reference to external Services from Job Templates

A Job Template could declare a dependency on a queue-supplied Service (a stub in `jobServices`
naming the Service and its ports), use `Service.<name>.*` with static validation, and be rejected at
submission if the queue supplied no match. Rejected because the Environment alongside an external
Service already publishes the endpoint by the means queue Environments use, so no declaration is
needed; no other queue-provided entity requires one; and the stub would add a discriminated union to
`jobServices`, a merge rule, a cross-document name requirement, and an exception to the
forward-reference rule. Unchecked `Service.*` references resolved at submission time are rejected as
well, because they would make `openjd check` unable to catch a typo and hide the dependency from
readers.

## Future Work

* **Custom named scopes, for Environments and Services alike.** This RFC gives a Service two
  scopes, the Job and a single Step, which are the two scopes Environments have. Use case 2 (a
  coordinator shared by a scatter Step and its gather Step) wants a middle scope: a Service that
  outlives one Step but not the Job, started when the first of a set of Steps becomes schedulable
  and stopped when the last completes. Environments have the same gap; a Step cannot opt in to or
  out of a specific Job Environment. One extension can close both: a Job Template declares named
  scopes, each Step lists the scopes it belongs to, and an Environment or Service is attached to a
  scope rather than to the Job or a Step, with its lifetime running from the first Step in the
  scope to the last. `runScope` on `<Environment>` is the first appearance of scope names in the
  schema, and its grammar (a list of names, unknown names rejected) is chosen so that user-defined
  names can be added without changing it.
* **Unix domain sockets.** A Unix domain socket is not a port: it is a filesystem path rather than
  a number in a protocol's space, there is no address to publish since it is reachable only from
  the host that holds it, and nothing about it can be forwarded. It therefore fits none of `port`,
  `bindAddress`, or `connectAddress`, and a Service used only from its own host would need a
  distinct kind of endpoint rather than another `protocol` value.
* **Co-scheduled Steps, for systems like MPI and Dask.** For use case 6, Tasks already run
  concurrently when capacity allows; what is missing is a *reservation*: an MPI job with a fixed
  world of 30 ranks deadlocks in `MPI_Init` if the scheduler launches 10 Tasks and the other 20 wait
  for hosts, and a Dask client needs at least some workers alive while it runs. A follow-up RFC
  should add three things: a Step-level reservation (a minimum number of the Step's Tasks, or all of
  them, for which hosts must be available before any launches, allocated all-or-nothing); a gang
  failure policy (cancel the sibling Tasks when one fails, and retry as a group); and a way to
  require network connectivity between the Tasks' hosts, since ranks talk peer-to-peer once they
  have found each other and this RFC guarantees reachability only from a Task's host to a Service
  host. That primitive is useful independently of Services, so it is not part of this RFC.
* **Replicated Services.** A `replicas:` count producing `Service.<name>.<port>.connectAddresses`
  as a `list[string]`, for a pool of identical daemons serving a single driver (a distributed
  cache, for example). Deferred until the single-instance model has been exercised; the
  co-scheduled Step above covers the cases where the replicas are themselves the work.
* **Session-scoped Services.** A service that is started once per Session on the Session's host, for
  per-host sidecars. An Environment whose `onEnter` backgrounds a process already serves this; it is
  noted only to record that it was considered distinct.

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is
more permissive.
