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
ports that a scheduler starts *before* any Task in its scope is scheduled, keeps running while the
scope has work, and stops once the scope no longer needs it. A Job Template declares Services in a
new `services` list alongside its Environments. A Step that uses a Service declares the dependency
in its `dependencies`, as it declares a dependency on another Step, and a Service's scope is the set
of Steps that depend on it, so a Service one Step depends on lives as long as that Step and a
Service every Step depends on lives as long as the Job. The template names the ports; the scheduler
chooses the port numbers and addresses when it places the Service, which may be on a different host
than the Tasks that use it. Every entity that depends on the Service reads them through a new
`Service.*` format-string scope. A health check gates Task scheduling in the scope until the service
is ready and detects an instance that has hung, and a restart policy says what happens when the
service process dies or hangs, including whether Tasks that completed against the old instance keep
their results. Services run inside the Job's Environments, so the same Conda, Rez, or container
Environments provision them and the Tasks. A queue can supply Services to every Job submitted
through it by defining them in an Environment Template, and a Job Template that needs such a Service
declares the requirement, with the ports it uses, in `requiresServices`.

## Overview

The `SERVICE` extension adds the following, each specified in the sections named:

* **The `<Service>` entity** and the `services` list that holds it. A Service has named ports, a
  health check, a restart policy, host requirements, `dependencies` on Steps and other Services, and
  the same `variables`, `let`, embedded files, and `<Action>` model as an Environment, with actions
  `onEnter`, `onRun`, `onHealthCheck`, and `onExit`. See [`<Service>`](#service).
* **Declared dependencies.** A Step depends on a Service by listing `service:<name>` in its
  `dependencies`, beside the Steps it depends on, and a Service depends on Steps and on other
  Services the same way. The `dependencies` lists of a template's Steps and Services form one
  acyclic graph. A Service's scope is the set of Steps that depend on it, directly or through
  another Service, computed at template validation; a Job Environment that references a Service
  puts every Step in its scope, and a Service nothing depends on is rejected. See
  [`<StepTemplate>`](#steptemplate) and [Service scope](#service-scope).
* **The `Service.*` format-string scope**, through which the service process learns what to bind
  and every entity that depends on it learns where to connect: `Service.<name>.<port>.port`,
  `bindAddress`, and `connectAddress`, plus `Service.File.*` for the Service's embedded files.
  `SERVICE` requires `EXPR`, and adds the `join_host_port` family of functions so that endpoints
  compose correctly on IPv6 networks. See [The `Service.*` scope](#the-service-scope) and
  [Modifications to the Expression Language](#modifications-to-the-expression-language).
* **Lifecycle, failure, and restart**, stated as constraints a scheduler must satisfy: ordering
  among Services, when Tasks may be scheduled, what happens when an instance fails or its host is
  lost, and the `completedTasks` choice between keeping and rerunning completed work. See
  [Modifications to How Jobs Are Run](#modifications-to-how-jobs-are-run).
* **Services run inside Environments.** A Service Session enters the Job's Environments before the
  Service's own actions. A new `runScope` property on `<Environment>` names the kinds of Session an
  Environment applies to; an Environment that references a Service defaults to Task Sessions only,
  so that an Environment which configures Tasks to *use* a Service stays out of Service Sessions.
  See [`<Environment>`](#environment).
* **External Services.** An Environment Template may define a `services:` list alongside or
  instead of its `environment:`, so a scheduler can start per-Job Services for every Job submitted
  through a queue. A Job Template that reads such a Service's endpoint directly declares what it
  needs in `requiresServices`, naming the Service and its ports, so that `openjd check` validates
  every reference and the scheduler checks the match at submission; one that uses it through the
  queue's Environment declares nothing. The Environment Template schema also gains the `extensions`
  and `$schema` keys. See [Environment Template](#environment-template) and
  [`<ServiceRequirement>`](#servicerequirement).
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
`Service.Cache.main.connectAddress`. If the service process dies, or stops accepting connections,
the scheduler relaunches it, and because a cache can be repopulated, completed Tasks keep their
results (`completedTasks: KEEP`). After the store is READY the same TCP probe that proved it ready
runs every 10 seconds, and three consecutive refused connections mark the instance UNHEALTHY, so a
Valkey that has hung without exiting is replaced as a crashed one would be. The Valkey binary comes
from a Conda environment the Tasks do not need, so the Service creates it in its own *onEnter*,
which runs once per Service Session before *onRun*, and puts it on `PATH` with an `openjd_env`
message. The Job's only Step lists `service:Cache` in its `dependencies`, which is what lets it
reference `Service.Cache.*`; the Service's scope is that Step, which here is the whole Job.

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [SERVICE, EXPR]
name: "Frame Processing With Shared Valkey Store"
parameterDefinitions:
  - name: FrameEnd
    type: INT
    default: 100

services:
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
    healthCheck:
      type: TCP_CONNECT
      readinessTimeoutSeconds: 60
      healthIntervalSeconds: 10
      failureThreshold: 3
    restartPolicy:
      maxAttempts: 3
      completedTasks: KEEP
    script:
      actions:
        # onEnter runs once per Service Session; the Tasks never see this Conda environment.
        onEnter:
          command: bash
          args: ["-c", "conda create -y -p ./valkey-env -c conda-forge valkey && echo openjd_env: PATH=$PWD/valkey-env/bin:$PATH"]
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
    dependencies:
      - dependsOn: service:Cache
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

### A coordinator that owns one Step's state

This Job prepares a scene in one Step and renders it as tiles in a second, where a work-distribution
service hands tiles to the render Tasks. `RenderTiles` lists two dependencies: the Step
`PrepareScene`, which must complete first, and `service:Coordinator`, which must be READY. Only
`RenderTiles` depends on the coordinator, so its scope is that one Step: the scheduler starts it
when `RenderTiles` becomes schedulable, not while `PrepareScene` is running, and stops it when the
last tile is done. The coordinator keeps the assignment state in memory, so if it dies and is
relaunched, the Tasks that already completed cannot be trusted: `completedTasks: RERUN` tells the
scheduler to requeue them. The service signals readiness itself on stdout, and does its one-time
database initialization in `onEnter` so that a restart of `onRun` does not wipe the schema.

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [SERVICE, EXPR]
name: "Tiled Render With Coordinator"

services:
  - name: Coordinator
    ports:
      - name: api
      - name: metrics
    healthCheck:
      type: STDOUT
      readinessTimeoutSeconds: 120
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

steps:
  - name: PrepareScene
    script:
      actions:
        onRun:
          command: prepare-scene
          args: ["--out", "{{ Session.WorkingDirectory }}/scene"]
  - name: RenderTiles
    dependencies:
      - dependsOn: PrepareScene
      - dependsOn: service:Coordinator
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
knowing they exist. Here a queue administrator attaches a Valkey store and an Environment that
publishes its address through environment variables; the `extensions` key on the Environment
Template is new in this RFC. The Environment references `Service.Cache.*`, so its `runScope`
defaults to `[TASK]`: it is entered in Task Sessions and not in the Service's own Session, where it
would be meaningless. The default is written out here; the other examples leave it implicit.

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

A Job Template can instead read the queue's Service directly. It declares what it needs in
`requiresServices`: the Service's name and the ports it uses. `Service.Cache.main.port` and
`Service.Cache.main.connectAddress` are then available to the template under the same rule as an
inline Service's, `openjd check` verifies every reference against the declaration, and the
scheduler rejects a submission to a queue that does not attach a Service named `Cache` with a TCP
port `main`. The Service is not declared in `services`, because the queue runs it; the template
only states the contract. A required Service has every Step in its scope and is READY before any
Task runs, so the Step's `dependsOn: service:Cache` below changes neither when its Tasks are
scheduled nor how long the Service lives; the Step lists it because that is what lets its script
reference `Service.Cache.*`, exactly as for a Service declared in `services`.

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [SERVICE, EXPR]
name: "Frame Processing With Required Queue Cache"

requiresServices:
  - name: Cache
    ports:
      - name: main

steps:
  - name: ProcessFrames
    dependencies:
      - dependsOn: service:Cache
    parameterSpace:
      taskParameterDefinitions:
        - name: Frame
          type: INT
          range: "1-100"
    script:
      actions:
        onRun:
          command: process-frame
          args:
            - "--frame"
            - "{{ Task.Param.Frame }}"
            - "--valkey-host"
            - "{{ Service.Cache.main.connectAddress }}"
            - "--valkey-port"
            - "{{ Service.Cache.main.port }}"
```

### A metrics sink with a UDP ingest port

This Job collects per-frame timings in a StatsD-style metrics sink that every Task reports to over
UDP, and exposes the aggregated results through a TCP HTTP API that the final Step reads through a
small authenticating proxy. The `ingest` port declares `protocol: UDP`, so a scheduler that
publishes or forwards ports between hosts forwards it as UDP; the `api` port is TCP by default. The
sink gives no `healthCheck`, so the default `TCP_CONNECT` probes its TCP ports only: once `api`
accepts a connection the sink is up and `ingest` is bound with it. The UDP port carries the same
three values as a TCP port, and the Tasks use `connectAddress` and `port` without caring which it
is. The proxy depends on the sink (`dependsOn: service:Metrics`), which is what lets it reference
`Service.Metrics.api.*`: it starts only once the sink is READY and is stopped before it. `Report`
depends on the proxy, so by the transitive rule it is in the sink's scope too, and the sink is
stopped only after `Report` has read the summary; `RenderFrames` depends on the sink directly. Both
Services therefore live for the whole Job.

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [SERVICE, EXPR]
name: "Frame Render With Metrics Sink"

services:
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
  - name: Proxy
    description: "Fronts the sink's HTTP API with authentication."
    dependencies:
      - dependsOn: service:Metrics
    ports:
      - name: http
    script:
      actions:
        onRun:
          command: auth-proxy
          args:
            - "--listen"
            - "{{ join_host_port(Service.Proxy.http.bindAddress, Service.Proxy.http.port) }}"
            - "--upstream"
            - "http://{{ join_host_port(Service.Metrics.api.connectAddress, Service.Metrics.api.port) }}"

steps:
  - name: RenderFrames
    dependencies:
      - dependsOn: service:Metrics
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
      - dependsOn: service:Proxy
    script:
      actions:
        onRun:
          command: curl
          args:
            - "-sf"
            - "http://{{ join_host_port(Service.Proxy.http.connectAddress, Service.Proxy.http.port) }}/summary"
```

### Execution order

For a Job with one Service `S`, one `jobEnvironment` `E`, and one Step, which depends on `S`, with
Tasks `T1..Tn` running in two Sessions on two Worker Hosts, the scheduler proceeds:

```
Service host                   Worker host A            Worker host B
------------                   -------------            -------------
E.onEnter
  S.onEnter
  S.onRun (starts)
  S health probe -> READY
                               E.onEnter                E.onEnter
                                 T1.onRun                 T3.onRun
                                 T2.onRun                 T4.onRun
                               E.onExit                 E.onExit
  S.onRun (canceled)
  S.onExit
E.onExit
```

The Service runs in a Session of its own on the service host, inside the same Environment `E` that
the Tasks run in. No Task of a Step that depends on `S` is scheduled until `S` is READY. `S`
outlives every Session that uses it and is stopped only when its scope has no remaining Tasks that
could run.

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
   main action runs for the *whole* lifetime of its scope, which is a set of Steps rather than one
   Session, and the scheduler must keep it alive across all of the scope's Sessions.
3. **Endpoint publication.** A Service exposes one or more ports. The runtime allocates a concrete
   address and port and makes them discoverable, on any host, both to the service process (so it
   knows what to bind) and to every entity in its scope (so they know where to connect).
4. **Readiness, health, and failure.** A scheduler must know when the service is accepting
   connections before it schedules the Tasks in the Service's scope, must notice when a running
   service stops answering, and must react when it dies or hangs: relaunch it, or fail the scope.
   Because a service can hold state that Tasks depend on, a service author must be able to say
   whether Tasks that completed against the old instance are still valid.

A Service borrows the rest of its shape from existing entities: `description`, `variables`, `let`,
embedded files, and the `<Action>` model with its cancelation methods from `<Environment>`, and
`hostRequirements` from `<StepTemplate>`.

A Service also needs what the Tasks need. The Valkey binary, the coordinator's Python environment,
or the renderer in server mode is provisioned today by the Job's Environments (a Conda or Rez
Environment, or a container under `WRAP_ACTIONS`), and a Service that could not use them would force
every Service author to reimplement that provisioning. Services therefore run inside the Job's
Environments, and wrapping Environments wrap Service actions as they wrap Task actions. An
Environment that configures Tasks to *use* a Service must then stay out of the Service's own
Session; `runScope` provides that, defaulting to Task Sessions for any Environment that references a
Service.

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
- A template that uses `services`, `requiresServices`, `runScope`, the `Service.*` format-string
  scope, or the string functions `join_host_port`, `split_host_port`, `is_ipv4`, or `is_ipv6` MUST
  list `SERVICE` in `extensions:`, and a scheduler MUST reject a template that uses any of these
  without declaring the extension.
- No existing field changes meaning. A template with no `SERVICE` extension is unaffected on its
  own; in particular the constraint that a Step's `name` not contain `:`, which lets `dependsOn`
  name a Service as `service:<name>`, applies only to templates that declare `SERVICE` (see
  [`<StepTemplate>`](#steptemplate)). One combination is newly rejected at submission: a wrapping
  Environment (RFC 0008) whose document does not declare `SERVICE`, placed in the `jobEnvironments`
  of a Job that has any Service; see [Validation](#validation). Either document can be the one at
  fault: an existing queue wrapper template meets a Job Template that declares Services, or a queue
  gains an external Service and every existing Job Template that contains a Job-level wrapper
  becomes unsubmittable to it. In both cases that document must declare `SERVICE` and either define
  the `onWrapService*` hooks or declare a `runScope` that excludes `SERVICE`. Queue operators SHOULD
  account for this before attaching a Service to a queue whose Job Templates use wrappers; running
  the Service outside the wrapper instead would silently defeat what the wrapper enforces.
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
| **Service** | An entity declared in a `services` list: a long-lived process that publishes one or more network ports and is kept alive by the scheduler for the lifetime of its scope. |
| **Scope** | The set of Steps whose Tasks depend on a Service: the Steps that list it in their `dependencies`, directly or through another Service, determined at template validation (see [Service scope](#service-scope)). A Service is started before any Task of a Step in its scope and stopped once no such Task remains to be run. |
| **Service Session** | The Session on the service host in which a Service's actions run: a working directory, the Job's Environments whose `runScope` includes `SERVICE` entered around the Service's actions, and the Service's actions themselves. It is a Session as described in *How Jobs Are Run*, containing one Service instead of Tasks. |
| **Service host** | The Worker Host the scheduler places a Service on. It may or may not be a host that also runs Tasks. |
| **UNREADY / READY / UNHEALTHY / FAILED** | The four Service states the scheduler tracks. See [Service lifecycle](#service-lifecycle). |
| **Instance** | One launch of a Service's `onRun` action. A Service has one instance at a time; a restart creates a new instance. |
| **External Service** | A Service supplied to a Job by the scheduler from an Environment Template, rather than declared in the Job Template. A Job Template that reads an external Service's endpoint declares a **requirement** for it in `requiresServices`. |
| **Inline Service** | A Service declared in the Job Template's own `services` list, as opposed to an external Service. |

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
+ services: [ <Service>, ... ] # @optional @extension SERVICE
+ requiresServices: [ <ServiceRequirement>, ... ] # @optional @extension SERVICE
  steps: [<StepTemplate>, ...]
```

New properties:

* *services* — The Services that the Jobs created from this Job Template run. Each Service is
  started once every entry of its `dependencies` is satisfied (its Steps completed, its Services
  READY) and before any Task of a Step in its scope is scheduled, and is stopped once no Task of a
  Step in its scope remains to be run, before any Service it depends on. A Service's scope is the
  set of Steps that depend on it, directly or through another Service (see
  [Service scope](#service-scope)); the order of this list carries no meaning. See
  [`<Service>`](#service). Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. Maximum number of elements: 10.
    3. No two Services in this list may have the same value for the `name` property.
    4. The `dependencies` of the Job Template's Steps and Services form one graph, which must be
       acyclic (see [`<StepDependency>`](#stepdependency)).
    5. No Service in this list may have the same `name` as an entry of *requiresServices*.
    6. Every Service in this list has a non-empty scope: some Step depends on it, directly or
       through another Service, or some `jobEnvironments` entry references it.
* *requiresServices* — The external Services (see
  [Environment Template](#environment-template)) whose endpoints this Job Template reads directly,
  each with the ports it uses. A requirement makes `Service.<name>.<port>.port` and
  `Service.<name>.<port>.connectAddress` available, for each port it declares, under the same rule
  as an inline Service's: to the Steps and Services that list `service:<name>` in their
  `dependencies`, and to `jobEnvironments` entries whose `runScope` excludes `SERVICE` (see
  [The `Service.*` scope](#the-service-scope)). The scheduler matches it at submission to exactly
  one Service of that name among the attached Environment Templates' `services`. A Job Template
  that uses an external Service only through the effects of the Environment defined alongside it
  declares nothing. See [`<ServiceRequirement>`](#servicerequirement). Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. Maximum number of elements: 10.
    3. No two requirements in this list may have the same value for the `name` property.
    4. No requirement in this list may have the same `name` as a Service in *services*: a Service
       is either declared or required.

  This property is permitted only in a Job Template. A requirement from one attached Environment
  Template on a Service of another is out of scope for this RFC.

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
* *services* — The Services that the Environment Template defines, with the same constraints as a
  Job Template's `services`: at least one element, at most 10, unique `name`s, and acyclic
  `dependencies`. A Service in this list may depend only on other Services in the same list
  (`dependsOn: service:<name>`), since the document has no Steps, and is never rejected for an
  empty scope, since every Step of every Job is in it. At least one of *environment* or *services*
  must be provided.

**Services from Environment Templates.** An Environment Template that defines *services* defines
Services that the scheduler applies to every Job Template submitted through it, just as its
*environment* defines an Environment applied to every submission. From the Job Template's point of
view these are **external Services**. Each Job gets its own instance of each, and every Step of the
Job is in its scope, whether or not the Job Template requires it; a process shared across many Jobs
is infrastructure outside the scope of this specification.

A `Service.*` reference within an Environment Template MUST resolve to a Service in the same
document's *services* list. A `Service.*` reference within a Job Template MUST resolve to a Service
in its own *services* list or to a port declared by one of its *requiresServices* entries. No
reference crosses a document boundary except through a requirement, so every `Service.*` reference
in every template is checkable without knowledge of the scheduler's configuration.

A Job Template that reads an external Service's endpoint declares a requirement for it in
`requiresServices`, naming the Service and the ports it uses, and then references
`Service.<name>.<port>.port` and `Service.<name>.<port>.connectAddress` as it would an inline
Service's; `bindAddress` of an external Service is never in scope. A Job Template that uses an
external Service only through the effects of the Environment defined alongside it (environment
variables, files written to the Session working directory, and so on) declares nothing, exactly as
it uses a queue Environment today.

When a submission combines a Job Template with one or more Environment Templates:

1. Every attached Service is provided to the Job, whether or not a requirement names it, with
   every Step in its scope; the attached Environments are placed in `jobEnvironments` as today. The
   limit of 10 applies to each document's list; the scheduler bounds the total through how many
   Environment Templates it attaches.
2. Each entry of `requiresServices` MUST match exactly one Service with the same `name` among the
   attached Environment Templates' *services*, and that Service MUST declare every port the
   requirement lists, with the same `protocol`. If no attached Service has the name, if two or more
   do, if the matched Service lacks a listed port, or if a port's protocol differs, the submission
   MUST be rejected, naming the requirement and the cause. Matching uses the requirement alone; the
   scheduler reads nothing else from the Job Template.
3. `Service.*` references in a Job Template resolve to its inline Services first. An attached
   Service with the same `name` as an inline Service is not visible to that Job Template and still
   runs, reaching the Job's Tasks through the Environment defined alongside it. Two attached
   Services with the same `name` are an error only when a requirement names it. An Environment
   Template's own Environment and Services reference only that document's *services*. A scheduler
   MUST keep same-named Services from different documents distinct (for example, by qualifying each
   with its document).

#### `<StepTemplate>`

> A modification to
> [`3. <StepTemplate>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#3-steptemplate)
> and [`3.1. <StepName>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#31-stepname)

The `<StepTemplate>` object is unchanged. Two of its items change under `SERVICE`:

* *dependencies* — The Steps and, when using the `SERVICE` extension, the Services that this Step
  depends on. Each entry names a Step (`dependsOn: <StepName>`), as today, or a Service
  (`dependsOn: service:<ServiceName>`) that the Job Template declares in `services` or requires in
  `requiresServices`. The Step's Tasks may be scheduled only when every Step it depends on has
  completed successfully and every Service it depends on is READY. Listing a Service puts the Step
  in that Service's scope, and is what makes `Service.<name>.*` available to the Step's `script` and
  `stepEnvironments`; see [Service scope](#service-scope) and
  [`<StepDependency>`](#stepdependency).
* *name* — When the Job Template declares `SERVICE`, a Step's `name` MUST NOT contain `:`, so that
  a `dependsOn` value beginning `service:` can only name a Service. The definition of
  `<StepName>` is otherwise unchanged, and this constraint applies only to templates that declare
  the extension; in any other template `service:Cache` is an ordinary Step name.

##### `<StepDependency>`

> A modification to
> [`3.2. <StepDependency>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#32-stepdependency)

A `<StepDependency>` names a Step or, with `SERVICE`, a Service that the entity listing it depends
on. It appears in the `dependencies` of a `<StepTemplate>` and of a [`<Service>`](#service).

```diff
- dependsOn: "<StepName>"
+ dependsOn: "<StepName>" | "service:<ServiceName>" # the service: form @extension SERVICE
```

Where:

1. *dependsOn* — The name of a Step in the same Job Template, or, when using the `SERVICE`
   extension, the literal prefix `service:` followed by the name of a Service that the Job Template
   declares in `services` or requires in `requiresServices`. A dependency on a Step is satisfied
   when that Step has completed successfully, as today. A dependency on a Service is satisfied when
   the Service is READY; the Step's Tasks are not scheduled, or the dependent Service is not
   started, until it is (see [Service lifecycle](#service-lifecycle)). A Step that depends on a
   Service is in that Service's scope (see [Service scope](#service-scope)). A dependency on a
   required external Service is satisfied when that Service is READY, which the scheduler
   guarantees before any Task of the Job runs, and does not change the Service's scope, which is
   every Step; listing it is nonetheless how the Step or Service gains access to the required
   Service's `Service.<name>.*` values, as it is for an inline Service. Constraints:
    1. A Step name MUST name a Step of the same Job Template other than the entity listing it. A
       `service:` name MUST name a Service of the same Job Template's `services` or
       `requiresServices`, other than the entity listing it.
    2. No two entries of one `dependencies` list may name the same Step or the same Service.
    3. The `dependencies` lists of a Job Template's Steps and Services form one graph whose nodes
       are the Steps and Services and whose edges run from each entity to each entry in its list
       (Step to Step, Step to Service, Service to Step, and Service to Service). That graph MUST be
       acyclic. A template with a cycle MUST be rejected at template validation, naming the cycle.
    4. In an Environment Template, which has no Steps, an entry MUST use the `service:` form and
       name a Service of the same document's `services`.

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

  If not provided, the default follows from the Environment's own text: an Environment any of
  whose format strings (*variables*, actions, embedded files, `let`) references a `Service.*`
  value is entered in Task Sessions only, as if `runScope: [TASK]` were given, and any other
  Environment is entered in every kind of Session, including kinds that a later extension defines,
  so existing Environments that provision software apply to Services without modification. An
  explicit list is exhaustive: the Environment is entered in those kinds and no others. An
  implementation MUST reject a `<RunScopeName>` it does not recognize. An extension that defines a
  new kind of Session MUST give it a `<RunScopeName>`, state which `Service.*`-style values (if
  any) are in scope for Environments entered in it, and, under `WRAP_ACTIONS`, name the wrap hooks
  a wrapping Environment must define for it.
  Constraints:
    1. Minimum number of elements: 1. Maximum: one of each defined name; no duplicates.
    2. An Environment whose *runScope* includes `SERVICE` MUST NOT reference any `Service.*` value:
       a Service Session may begin before any Service other than its own has an endpoint, and its
       own endpoint is not meaningful to an Environment that provisions it. An Environment that
       references `Service.*` and gives an explicit *runScope* must therefore exclude `SERVICE`.

> A modification to [`4.3. <EnvironmentActions>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#43-environmentactions)

```diff
  onEnter: <Action> # @optional
  onWrapEnvEnter: <Action>    # @optional @extension WRAP_ACTIONS
  onWrapTaskRun: <Action>     # @optional @extension WRAP_ACTIONS
  onWrapEnvExit: <Action>     # @optional @extension WRAP_ACTIONS
+ onWrapServiceEnter: <Action>       # @optional @extension WRAP_ACTIONS SERVICE
+ onWrapServiceRun: <Action>         # @optional @extension WRAP_ACTIONS SERVICE
+ onWrapServiceHealthCheck: <Action> # @optional @extension WRAP_ACTIONS SERVICE
+ onWrapServiceExit: <Action>        # @optional @extension WRAP_ACTIONS SERVICE
  onExit: <Action> # @optional
```

New properties, available only when both `WRAP_ACTIONS` (RFC 0008) and `SERVICE` are in use:

* *onWrapServiceEnter*, *onWrapServiceRun*, *onWrapServiceHealthCheck*, *onWrapServiceExit* — When
  this Environment is a wrapping Environment in a Service Session, these run instead of the
  Service's *onEnter*, *onRun*, *onHealthCheck*, and *onExit* respectively, exactly as
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
   *runScope* includes `SERVICE`. A template that defines a hook its *runScope* does not call for,
   or omits one it does, MUST be rejected at template validation. For a wrapping Environment in a
   document that does not declare `SERVICE`, see the submission-time check in
   [Validation](#validation).
2. *Nothing to replace.* `onWrapServiceEnter`, `onWrapServiceHealthCheck`, and
   `onWrapServiceExit` run only for a Service that defines the corresponding action. Every Service
   defines *onRun*, so `onWrapServiceRun` runs for every wrapped Service.
3. *Single layer.* A Service Session contains at most one wrapping Environment, as any Session
   does.
4. *Stdout.* As in RFC 0008, the runtime scans the wrap script's stdout, not the wrapped
   process's. A wrapped *onEnter*'s `openjd_env` messages and a wrapped *onRun*'s
   `openjd_service_ready` message are therefore honored when the wrap script forwards the wrapped
   process's stdout verbatim, which RFC 0008 already requires. `onWrapServiceHealthCheck` runs
   concurrently with `onWrapServiceRun`; wrap scripts MUST tolerate this, and the output of both
   wrap scripts is subject to the log attribution rule (see
   [Concurrency with `onRun`](#concurrency-with-onrun)).
5. *Failure.* A failed `onWrapServiceRun` is an instance failure, a failed `onWrapServiceEnter` is a
   start failure, and a failed `onWrapServiceExit` is an *onExit* failure, exactly as the wrapped
   action's failure would be. The exit status of an `onWrapServiceHealthCheck` invocation is one
   probe result, as the wrapped *onHealthCheck*'s would be: 0 is a successful probe, anything else
   (or exceeding its timeout) a failed one, and never by itself a failure of the Service.

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
entity that depends on the Service can discover the resulting endpoint via the `Service.*`
format-string scope.

A `<Service>` is the object:

```yaml
name: <ServiceName>
description: <Description> # @optional
let: <LetBindings> # @optional @extension EXPR
dependencies: [ <StepDependency>, ... ] # @optional
hostRequirements: <HostRequirements> # @optional
ports: [ <ServicePort>, ... ]
healthCheck: <ServiceHealthCheck> # @optional
restartPolicy: <ServiceRestartPolicy> # @optional
variables: <EnvironmentVariables> # @optional
script: <ServiceScript>
```

Where:

1. *name* — An identifier for the Service that is unique within the `services` list that declares
   it and, in a Job Template, differs from every `requiresServices` name. It is the first component
   of `Service.*` references to this Service. See [`<ServiceName>`](#servicename).
2. *description* — A description to apply to the Service. It has no functional purpose, but may
   appear in UI elements.
3. *let* — An ordered list of expression bindings evaluated once, at job creation, like a
   `<StepTemplate>`'s *let*. Bindings may reference `Param.*`, `RawParam.*`, and `Job.Name`, but
   not `Session.*` or `Service.*`, which are not known until the Service is placed. Bound names are
   available in *hostRequirements*, *variables*, and *script*, as a Step's bindings are in its
   `stepEnvironments`. Available with the `EXPR` extension.
4. *dependencies* — The Steps and Services this Service depends on, with the same shape as a
   Step's *dependencies*: each entry names a Step (`dependsOn: <StepName>`) or a Service
   (`dependsOn: service:<ServiceName>`); see [`<StepDependency>`](#stepdependency). The Service is
   started only after every listed Step has completed and every listed Service is READY, in
   addition to its other start conditions, and is stopped before any Service it lists. List a Step
   for a Service that consumes the output of a batch Step, so that it does not hold a host while
   the batch runs; list a Service for one whose endpoint this Service reads, which is also what
   makes `Service.<name>.*` of that Service available here. A Service that depends on another puts
   its own scope inside the other's (see [Service scope](#service-scope)). Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. Each `dependsOn` MUST name a Step of the same Job Template, or a Service, other than this
       one, of the same document's `services` or, in a Job Template, of its `requiresServices`.
    3. The `dependencies` of a document's Steps and Services form one graph, which MUST be acyclic.
       A Service that lists a Step in its own scope, directly or through other Services, is a cycle:
       the Step could not run until the Service was READY, and the Service could not start until the
       Step completed. A template with a cycle MUST be rejected at template validation.
    4. In an Environment Template's `services`, every entry MUST use the `service:` form, since that
       document has no Steps.
5. *hostRequirements* — Requirements on the Worker Host's capabilities that must be satisfied for
   the Service to be placed on the host. Amount capabilities are allocated to the Service Session
   for the lifetime of the Service. This is independent of the *hostRequirements* of any Step
   whose Tasks use the Service.
6. *ports* — The named ports that the Service exposes. See [`<ServicePort>`](#serviceport).
    1. Minimum number of elements: 1.
    2. Maximum number of elements: 10.
    3. No two ports may have the same `name`.
    4. No two ports with the same `protocol` may have the same `port` number. Two ports MAY have
       the same number when their `protocol` differs: a service that speaks TCP and UDP on one
       number gives that number explicitly on both ports. Ports whose `port` is not provided are
       allocated independently, each in its own protocol's space, and may or may not coincide
       across protocols.
7. *healthCheck* — The probe by which the scheduler determines that an instance of the Service is
   ready to accept traffic and, once it is, that it remains healthy. If not provided, defaults to
   `{ type: TCP_CONNECT }` applied to every TCP port. A Service none of whose ports is TCP MUST
   provide a *healthCheck* of type `STDOUT` or `COMMAND`; a `TCP_CONNECT` check, given or
   defaulted, would have no port to probe. See [`<ServiceHealthCheck>`](#servicehealthcheck).
8. *restartPolicy* — What the scheduler does when the Service's `onRun` action exits before the
   scope ends. If not provided, defaults to `{ maxAttempts: 0, completedTasks: RERUN }`: the
   Service is never relaunched, and its exit fails the scope. See
   [`<ServiceRestartPolicy>`](#servicerestartpolicy).
9. *variables* — A set of environment variable name/value pairs, with the values being Format
   Strings resolved when the Service is started, that are set in the process environment of every
   action of the Service's *script*, including *onHealthCheck*. It has the same schema as the
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
4. `Service.<name>.<port>.*` — The endpoint of this Service's own ports and of any Service this
   Service lists in its *dependencies*, inline or, in a Job Template, required (`port` and
   `connectAddress` only for a listed Service). A reference to a Service this Service does not
   list is invalid; see [The `Service.*` scope](#the-service-scope) and
   [Service scope](#service-scope).
5. `Job.Name`.
6. Names bound by `let` in the `<Service>` and in the `<ServiceScript>`.

`Task.*` and `Step.*` values are never available within a Service, since a Service belongs to no
Task and to no Step.

##### Service scope

The scope of a Service is the set of Steps whose Tasks depend on it. For a Service declared in a
Job Template it is determined at template validation from the template's `dependencies` lists:

1. A Step that lists `service:X` in its `dependencies` is in the scope of Service `X`.
2. When Service `Y` lists `service:X`, every Step in `Y`'s scope is in `X`'s scope. This applies
   transitively: a Step is in `X`'s scope when it depends on `X` through any chain of Services.
3. When any `jobEnvironments` entry references `Service.X.*`, every Step is in `X`'s scope. This is
   the one way a Step enters a scope without listing a dependency: a Job Environment configures the
   Tasks of every Step, and has no `dependencies` of its own in which to say so.
4. A Service in whose scope no Step falls — one that no Step and no Service lists and no Job
   Environment references — is unused, and the template MUST be rejected at template validation,
   naming the Service.

The `dependencies` lists of a template's Steps and Services together form one dependency graph, with
Step-to-Step, Step-to-Service, Service-to-Step, and Service-to-Service edges, which MUST be acyclic
(see [`<StepDependency>`](#stepdependency)). A Service's `Service.*` values are available exactly to
the entities that depend on it, so a format string that references a Service the enclosing Step or
Service does not list is a validation error, not an implicit dependency (see
[The `Service.*` scope](#the-service-scope)).

An external Service has every Step of the Job in its scope, whether or not any Step lists it;
listing a required Service does not change its scope, only which Steps and Services may reference
it (see [The `Service.*` scope](#the-service-scope)). A Service is started before any Task of a
Step in its scope is scheduled, and stopped once no Task of a Step in its scope remains to be run
(see [Service lifecycle](#service-lifecycle)); a Service whose scope is one Step therefore lives as
long as that Step, and one whose scope is every Step lives as long as the Job. A Service belongs to
no Step: its Session enters the Job's `jobEnvironments` and never a Step's `stepEnvironments`.

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

Numeric fields marked `@fmtstring` — `<ServicePort>.port`; the `<ServiceHealthCheck>` fields
`readinessIntervalSeconds`, `readinessTimeoutSeconds`, `healthIntervalSeconds`, and
`failureThreshold`; and `<ServiceRestartPolicy>.maxAttempts` — may be given as a format string whose
result is the integer, so that a Job Parameter or a `<Service>.let` binding can supply them. They
are resolved at job creation, with the scope of the `<Service>`'s *let* (item 3). When the value is
a single whole-field expression (`"{{ ... }}"` with no surrounding text) its target type is `int?`:
a `null` result is treated as if the field were not provided, and a non-null result MUST satisfy the
field's range.

Every port, TCP or UDP, is bound and published on the same service host. Unix domain sockets are
out of scope for this RFC (see [Future Work](#future-work)).

##### `<ServiceHealthCheck>`

A `<ServiceHealthCheck>` is one of the following objects, discriminated by *type*:

```yaml
type: "TCP_CONNECT"
ports: [ <Identifier>, ... ] # @optional
readinessIntervalSeconds: <posinteger> | <posintstring> # @optional @fmtstring
readinessTimeoutSeconds: <posinteger> | <posintstring> # @optional @fmtstring
healthIntervalSeconds: <posinteger> | <posintstring> # @optional @fmtstring
failureThreshold: <posinteger> | <posintstring> # @optional @fmtstring
```

```yaml
type: "COMMAND"
readinessIntervalSeconds: <posinteger> | <posintstring> # @optional @fmtstring
readinessTimeoutSeconds: <posinteger> | <posintstring> # @optional @fmtstring
healthIntervalSeconds: <posinteger> | <posintstring> # @optional @fmtstring
failureThreshold: <posinteger> | <posintstring> # @optional @fmtstring
```

```yaml
type: "STDOUT"
readinessTimeoutSeconds: <posinteger> | <posintstring> # @optional @fmtstring
healthIntervalSeconds: <posinteger> | <posintstring> # @optional @fmtstring
failureThreshold: <posinteger> | <posintstring> # @optional @fmtstring
```

A health check is one probe mechanism applied in two phases. Before the instance is READY the probe
decides readiness: the first probe runs as soon as `onRun` is launched, a probe runs every
*readinessIntervalSeconds* after it, and the first success makes the instance READY. After the
instance is READY the probe decides health: a probe runs every *healthIntervalSeconds*, and
*failureThreshold* consecutive failures make the instance UNHEALTHY, which is an instance failure
(see [Failure and restart](#failure-and-restart)). A single successful probe resets the count. While
the count is below the threshold the instance stays READY and nothing changes. In both phases an
interval is measured from the end of the previous probe.

Where:

1. *type* — The probe mechanism:
    * `TCP_CONNECT` — A probe succeeds when a TCP connection to each of the listed ports (by
      default, every TCP port the Service declares) succeeds. The connection is made from the
      service host to the allocated `port` on the loopback interface, or on `bindAddress` when it
      is not a wildcard address (`0.0.0.0` or `::`, which cannot be connected to on every operating
      system), and is closed immediately. This type is usable only on a Service with at least one
      TCP port; see *healthCheck* in [`<Service>`](#service).
    * `COMMAND` — A probe is one invocation of the Service's *onHealthCheck* action (see
      [`<ServiceActions>`](#serviceactions)), and succeeds when the invocation exits with status 0
      while `onRun` is still running. The Service MUST define *onHealthCheck* when this type is
      used. The action is run in the Service Session like every other Service action, concurrently
      with `onRun`; see [Concurrency with `onRun`](#concurrency-with-onrun). Invocations are
      sequential. The action's own `timeout` (default: 30 seconds) bounds one invocation; an
      invocation that exceeds it is canceled and is one failed probe, as is any exit status other
      than 0. Neither is by itself a failure of the Service.
    * `STDOUT` — A probe is a line of the form `openjd_service_ready: <message>` written to stdout
      by the `onRun` action, with the same syntax as the other `openjd_*` messages. The first such
      line makes the instance READY; `<message>` has no functional purpose but MAY be surfaced in
      UIs. After READY, when *healthIntervalSeconds* is given the line is a heartbeat: each interval
      in which no such line arrives is one failed probe. With `healthIntervalSeconds: 10` and
      `failureThreshold: 3`, a service that prints the line every five seconds stays READY, and one
      that falls silent is UNHEALTHY thirty seconds after its last line. When it is omitted, the
      health of a `STDOUT` instance is that its `onRun` process is still running, and further lines
      have no effect. *readinessIntervalSeconds* does not apply to this type: the ready line arrives
      when it arrives.
2. *ports* (`TCP_CONNECT` only) — The names of the ports to probe. Each must be declared in the
   Service's *ports* and have `protocol: TCP`; naming a UDP port is a validation error, since a
   UDP port cannot accept a connection. Defaults to every TCP port the Service declares.
3. *readinessIntervalSeconds* (`TCP_CONNECT` and `COMMAND` only) — Seconds between probes before
   the instance is READY. Default: 1 for `TCP_CONNECT`, 5 for `COMMAND`. A `STDOUT` check that
   gives this field MUST be rejected at template validation.
4. *readinessTimeoutSeconds* — The maximum time, measured from the launch of the `onRun` action,
   that the scheduler waits for the instance to become READY. It runs continuously, including while
   a probe is in progress. If exceeded, the instance has failed (see [Failure and
   restart](#failure-and-restart)). Default: 300.
5. *healthIntervalSeconds* — Seconds between probes after the instance is READY. Default: 30 for
   `TCP_CONNECT` and `COMMAND`. For `STDOUT` there is no default: when given, it is the heartbeat
   interval described under *type*; when omitted, no heartbeat is expected.
6. *failureThreshold* — The number of consecutive failed probes after READY (for a `STDOUT`
   heartbeat, consecutive intervals without a line) that make the instance UNHEALTHY. Default: 3.
   A `STDOUT` check that gives this field without *healthIntervalSeconds* MUST be rejected at
   template validation, since there is no probe for it to count.

The health check applies to every instance: after a restart, the new `onRun` must pass a probe
before Tasks in the scope are (re)scheduled, and is monitored in the same way once it has.

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
onHealthCheck: <Action> # @optional
onExit: <Action> # @optional
```

Where:

1. *onEnter* — A one-time setup action run before the first `onRun` of the Service in a Service
   Session; it runs once per Service Session, not once per instance. It is an ordinary action, like
   an Environment's `onEnter`: it runs to completion and its `timeout` and `cancelation` apply as
   for any `<Action>`. A non-zero exit or a timeout is a start failure (see [Failure and
   restart](#failure-and-restart)). Use *onEnter* for setup that must survive a relaunch of *onRun*
   (for example, initializing an on-disk database in the Service Session's working directory).
2. *onRun* — The long-lived action whose process *is* the service. The scheduler starts it, probes
   it for readiness and then for health, and expects it to run until canceled. If *onRun* exits for
   any reason — zero or non-zero exit status — before the scheduler cancels it, the Service
   instance has failed; see [Failure and restart](#failure-and-restart). The scheduler stops the
   Service by canceling *onRun* according to its `cancelation` method.
3. *onHealthCheck* — A short action the scheduler runs repeatedly, while *onRun* is running, as the
   probe of a `healthCheck` of type `COMMAND`: before the instance is READY, to decide whether it
   is; afterwards, to decide whether it still is. Exit status 0 is a successful probe; any other
   status, or exceeding its `timeout`, is a failed probe. MUST be defined when the health check
   type is `COMMAND`, and MUST NOT be defined otherwise. It SHOULD be read-only with respect to the
   Service Session's working directory, which it shares with *onRun*. See
   [`<ServiceHealthCheck>`](#servicehealthcheck) and
   [Concurrency with `onRun`](#concurrency-with-onrun).
4. *onExit* — A cleanup action run after the Service's other actions have stopped for the last
   time in a Service Session, whether the Service stopped normally, failed, or was canceled, and
   whether or not *onRun* was ever launched. It runs to completion. A non-zero exit is reported but
   does not change the outcome of the scope.

###### Concurrency with `onRun`

*onHealthCheck* is the only action in the specification that runs while another action of the
same Session, *onRun*, is running. Several existing rules silently assume one action at a time; the
following rules say where that assumption holds and where it does not:

1. **Embedded files.** Materializing embedded files for one action MUST NOT modify any file that a
   still-running action of the same Session was given. Implementations may materialize each
   invocation's files to a distinct location (`Service.File.<name>` may resolve to a different path
   in each action), or not rewrite a file whose content is unchanged, which it always is because a
   Service Session's format-string values are constant for its lifetime.
2. **Stdout.** The stdout of *onHealthCheck* is captured to the log but no `openjd_*` message on
   it is honored: its result is its exit status. The `openjd_status`, `openjd_progress`, and
   `openjd_fail` messages of the Service come from *onEnter*, *onRun*, and *onExit* only.
3. **Log attribution.** Every line of stdout or stderr captured in a Service Session MUST be
   attributable to the action that produced it. How attribution is recorded is
   implementation-defined: a structured log may carry the action name as a field, and an
   implementation producing a single plain-text log SHOULD tag each *onHealthCheck* line with the
   action name (for example, `[onHealthCheck] connection refused`) while leaving *onRun*'s lines
   untagged. The same applies under `WRAP_ACTIONS`, where the wrap script's own output is also part
   of each stream. The runtime adds the tag; a Service's processes need not prefix their own output.
   An implementation MAY suppress or collapse the output of invocations that succeed, provided the
   full output of every invocation that fails or is canceled for exceeding its `timeout` is
   retained.
4. **At most one invocation at a time.** *onHealthCheck* invocations never overlap one another and
   never run while *onEnter* or *onExit* is running. They continue for as long as *onRun* runs:
   every *readinessIntervalSeconds* before the instance is READY and every *healthIntervalSeconds*
   after.
5. **`onRun` exit wins.** An instance is READY only if *onRun* is still running when the scheduler
   observes a successful invocation. If *onRun* exits while an invocation is in progress, the
   invocation is canceled with its `cancelation` method and its result is discarded (lifecycle
   constraint 11); the instance has failed. Likewise, an invocation in progress when the Service
   Session ends is canceled before *onExit* runs.
6. **Wrap hooks.** Under `WRAP_ACTIONS`, `onWrapServiceHealthCheck` runs while `onWrapServiceRun`
   is running. Wrap scripts used in a Service Session MUST tolerate concurrent invocation. Wrappers
   that execute into an existing context (`docker exec`, `ssh`) do; a wrapper that creates a
   per-invocation exclusive resource does not.

Default timeouts for these actions (extending the table in `<Action>`):

| Action container | Action | Default |
|---|---|---|
| `<ServiceActions>` | `onEnter` | *no timeout* |
| `<ServiceActions>` | `onRun` | *no timeout* (a timeout, if given, is measured from launch and its expiry is an instance failure) |
| `<ServiceActions>` | `onHealthCheck` | 30 seconds |
| `<ServiceActions>` | `onExit` | 300 seconds (five minutes) |

Implementations MUST watch the stdout of *onEnter*, *onRun*, and *onExit* for the standard
`openjd_status`, `openjd_progress`, and `openjd_fail` messages, and the stdout of *onRun* for
`openjd_service_ready` when the health check's *type* is `STDOUT`. Messages on the stdout of
*onHealthCheck* are not honored (see [Concurrency with `onRun`](#concurrency-with-onrun)).
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
subsequent action of the same Service Session — every instance of *onRun*, *onHealthCheck*, and
*onExit* — and is retained across relaunches of *onRun* within the Session. Variables set by
*onEnter* take precedence over the Service's declarative *variables*. These messages are ignored
when emitted by *onRun*, *onHealthCheck*, or *onExit*, and nothing set within a Service is ever
propagated to the entities in the Service's scope.

##### `<ServiceRequirement>`

> A new section under `9. <Service>`, with `<ServiceRequirementPort>` as its subsection.

Available when using the `SERVICE` extension, in a Job Template's `requiresServices` only.

A `<ServiceRequirement>` declares that the Job Template reads the endpoint of an external Service
(see [Environment Template](#environment-template)): a Service that an attached Environment
Template defines, named `name`, with at least the ports listed. It is the Job Template's half of a
contract the scheduler checks at submission, and it makes the Service's endpoint available to the
template's format strings without the template declaring the Service itself.

A `<ServiceRequirement>` is the object:

```yaml
name: <ServiceName>
ports: [ <ServiceRequirementPort>, ... ]
```

Where:

1. *name* — The `name` of the external Service required. It MUST NOT equal the `name` of any
   Service in the Job Template's `services`. See [`<ServiceName>`](#servicename).
2. *ports* — The ports of the Service that the Job Template uses. Each makes
   `Service.<name>.<port>.port` and `Service.<name>.<port>.connectAddress` available under the same
   rule as an inline Service's ports: in the `script` and `stepEnvironments` of a Step that lists
   `service:<name>` in its `dependencies`, in an inline Service that lists it, and in every
   `jobEnvironments` entry whose `runScope` excludes `SERVICE`. A reference from a Step or Service
   that does not list the dependency is a validation error (see
   [The `Service.*` scope](#the-service-scope)). `bindAddress` of a required Service is never in
   scope. See [`<ServiceRequirementPort>`](#servicerequirementport). Constraints:
    1. Minimum number of elements: 1.
    2. Maximum number of elements: 10.
    3. No two ports may have the same `name`.

At submission, the scheduler matches each requirement to exactly one attached Service with the same
`name`, which MUST declare every listed port with the same `protocol`; see
[Environment Template](#environment-template) for the rule and the rejections. A required Service
has every Step of the Job in its scope, as every external Service does. A requirement that no Step,
Service, or Job Environment uses is not an error, unlike an inline Service that nothing depends on:
the requirement may exist to be matched for a consumer that reaches the Service by other means.

###### `<ServiceRequirementPort>`

A `<ServiceRequirementPort>` is the object:

```yaml
name: <Identifier>
protocol: enum("TCP", "UDP") # @optional
```

Where:

1. *name* — The `name` of a port the external Service declares. It is the second component of
   `Service.<service>.<port>.*` references. Must not be `File`.
2. *protocol* — The transport protocol the Job Template expects the port to carry. The attached
   Service's port MUST declare the same protocol. One of `TCP` or `UDP`. Default: `TCP`.

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
| `Service.<name>.<port>.port` | `int` | The port number allocated (or requested) for port `<port>` of Service `<name>`, in the space of that port's `protocol`. The same number is used for binding and for connecting. | Within the declaring Service, and within every entity that depends on it, per the rules below. |
| `Service.<name>.<port>.bindAddress` | `string` | The interface address the service process MUST bind to so that entities in the Service's scope can reach it. Typically `0.0.0.0` (or `::`) for a distributed scheduler and `127.0.0.1` for a single-host runner. | Within the declaring Service only; never for a required external Service. |
| `Service.<name>.<port>.connectAddress` | `string` | The hostname or IP address that entities in the Service's scope use to reach it. See [Address forms](#address-forms). | Within every entity that depends on the Service, per the rules below. Also within the declaring Service, for self-reference (e.g. to print its own URL). |
| `Service.File.<name>` | `path` | The filesystem location to which the Service embedded file with key `<name>` has been written. | Within the Service Script actions and embedded files of the declaring Service. |

`Service.<name>.<port>.*` for a Service `<name>` declared in the document's `services` or, in a
Job Template, declared by an entry of `requiresServices`, is in scope in:

1. The Service `<name>` itself (all three values), when it is declared in `services`.
2. Any Service in the same `services` list that lists `service:<name>` in its `dependencies`
   (`port` and `connectAddress` only): its actions, *variables*, embedded files, and
   `<ServiceScript>.let`.
3. In a Job Template, any Step that lists `service:<name>` in its `dependencies`: its `script`
   (actions, embedded files, `let`) and each of its `stepEnvironments` entries whose `runScope`
   excludes `SERVICE` (*variables*, actions, embedded files, `let`). A Step Environment follows its
   Step's dependencies.
4. In a Job Template, every `jobEnvironments` entry whose `runScope` excludes `SERVICE` (the default
   for an Environment that references `Service.*`), whether `<name>` is inline or required. This is
   the one exception to the dependency rule: a Job Environment has no `dependencies` in which to
   list a Service. A reference from one to an inline Service puts every Step in that Service's
   scope (see [Service scope](#service-scope)).
5. In an Environment Template: the document's `environment` when its `runScope` excludes `SERVICE`.

A reference from a Step or Service that does not list the dependency is a validation error, whether
the Service is inline or required, and the message SHOULD name the fix: for example, *Step 'Render'
references `Service.Cache.main.port` but does not list `service:Cache` in `dependencies`*. The
dependency is the author's statement that the Step needs the Service; the reference alone is not
taken as one.

For a Service `<name>` and port `<port>` that a Job Template's `requiresServices` declares, only
`port` and `connectAddress` are in scope; `bindAddress` of a required Service is never in scope.
Listing a required Service in `dependencies` is what grants a Step or Service access to its values,
and nothing more: the Service has every Step in its scope and is READY before any Task runs whether
or not any entity lists it.

`Service.*` is never in scope in a `hostRequirements` object, neither a Step's nor a Service's, nor
in a `<Service>`'s or a `<StepTemplate>`'s `let`.

A template that references a `Service.*` value outside the scopes above, or that references a
Service or port name that it neither declares nor requires, is invalid and MUST be rejected before
the Job is created.

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

A Service is in exactly one of four states, tracked by the scheduler:

| State | Meaning |
|---|---|
| **UNREADY** | No instance is READY. Either no instance has been launched yet, an instance is launched but has not yet passed a health-check probe, or the previous instance failed and a relaunch is pending or in progress. |
| **READY** | The current instance has passed a health-check probe, has not since failed `failureThreshold` consecutive probes, and has not exited. |
| **UNHEALTHY** | The current instance has failed `failureThreshold` consecutive health-check probes and is being stopped: the scheduler has canceled *onRun* and is waiting for it to exit, after which the restart decision applies. A transient state; for every other purpose, including Task scheduling, the Service is not READY. |
| **FAILED** | The Service will not be relaunched. Its scope has failed. |

A Service is UNREADY from Job creation. A Service's actions run in a **Service Session** on a
service host. Starting a Service means opening a Service Session and, within it, entering the Job's
Environments whose `runScope` includes `SERVICE` in order, running *onEnter* if defined, launching
*onRun*, and probing it with its health check. When a probe succeeds the Service is READY, and the
health check keeps probing it for as long as *onRun* runs.

A Service Session is always a Session of its own, never one that also runs Tasks.

A scheduler MUST satisfy the following constraints; how it satisfies them is its own concern.

**Ordering.**

1. All of a Service's ports are allocated, and `bindAddress` and `connectAddress` determined,
   before any action of its Service Session runs. The Service's own `Service.<name>.*` values are
   therefore resolvable throughout its Session.
2. No action of a Service Session (including the `onEnter` of an Environment it enters) begins until
   every entry in the Service's `dependencies` is satisfied: every listed Step has completed and
   every listed Service is READY. Services neither of which depends on the other, directly or
   transitively, MAY start concurrently. A Service that must start after another it does not
   otherwise use lists `service:<name>` in its `dependencies` like any other dependency.
3. No Task of a Step is scheduled until every Service whose scope includes that Step is READY. Once
   they are, Tasks are scheduled exactly as today, with no change to how Sessions are formed; in
   particular Services do not affect which Tasks may share a Session.
4. A Service is *stopped* before any Service it depends on is stopped. The dependency graph is
   acyclic, so stopping Services in reverse topological order satisfies this; Services neither of
   which depends on the other MAY be stopped concurrently. Services whose scopes complete at
   different times are stopped independently, each when its own scope completes, and a Service
   whose scope lies inside another's is always stopped no later than that one.

**Sessions.**

5. A Service has at most one live Service Session at a time, and at most one running *onRun*
   within it. Before *onRun* is launched again in the same Session, the previous *onRun* process
   MUST have exited, whether on its own or because the scheduler canceled it.
6. A Service Session ends when the Service's scope completes (no Step in the scope has a Task that
   could still run, whether because every Task completed or because the scope failed or was
   canceled), when the Service is relocated, or when the Session fails to start (see below). A
   Service whose scope completes MUST have its Session ended whatever its state, including UNREADY,
   UNHEALTHY, and FAILED, so that partial state is cleaned up.
7. Before a Service Session ends: any running action is canceled with its own `cancelation`
   method; *onExit* runs if it is defined and any action of the Service has run; and every
   Environment entered is exited in reverse order, as at the end of any Session. Then the working
   directory is deleted and the host's allocated amounts and ports are released.
8. Constraint 7 does not apply to a host the scheduler has lost (see below). Nothing is run or
   awaited there.
9. A Service that is started again after its Session has ended — after relocation, after a start
   failure, or because another Service's `RERUN` returned a Step in its scope to pending — begins a
   new Service Session: new host selection, new ports, new working directory, Environments
   re-entered, *onEnter* re-run.

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

**Health.**

11. A health-check probe in flight when *onRun* exits is canceled, with its `cancelation` method
    when it is an action, and its result is discarded, whether the instance was READY or not. No
    probe result observed after *onRun* has exited makes an instance READY or keeps it READY; the
    instance has failed (see [Failure and restart](#failure-and-restart)). An instance that becomes
    UNHEALTHY is stopped the same way an instance is stopped at scope end: *onRun* is canceled with
    its `cancelation` method, and constraint 5 holds, so no relaunch begins until it has exited.

#### Services run inside Environments

A Service Session enters the Job's `jobEnvironments` whose `runScope` includes `SERVICE`, including
any attached from Environment Templates in the scheduler's order, in the same order a Task Session
enters them. A Service belongs to no Step, so no Step's `stepEnvironments` are entered, whatever the
Service's scope. All are entered before the Service's *onEnter* and exited, in reverse, after its
*onExit*. Environment variables set by an Environment's `variables` or `openjd_env` apply to every
action of the Service, later Environments taking precedence over earlier ones, with the Service's
own *variables* taking precedence over all of them and variables set by the Service's *onEnter*
taking precedence over both.

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
  scheduler canceled it; when `readinessTimeoutSeconds` elapses before the instance is READY; when
  the instance becomes UNHEALTHY, having failed `failureThreshold` consecutive health-check probes
  after READY; or when the scheduler loses the service host. An exit observed after the scope has
  completed is not a failure, whether or not the scheduler's cancelation had yet reached the
  process; the Service Session simply ends.
* A **start failure** occurs when a Service Session fails before *onRun* is launched: a requested
  *port* cannot be allocated on the chosen host, an Environment's `onEnter` fails, or the Service's
  *onEnter* exits non-zero or times out. The Session ends (constraint 7 applies: *onExit* runs if
  *onEnter* ran, Environments are exited).

On either failure the Service becomes UNREADY (an UNHEALTHY instance, once its *onRun* has exited)
and the scheduler:

1. If `restartPolicy.completedTasks` is `RERUN`, cancels every running Task in the Service's scope;
   each is returned to the queue to run again once the Service is READY, and this is not counted as
   a Task failure. If it is `KEEP`, running Tasks continue. A Task that needs the Service while it
   is not READY, or that holds the endpoint of a relocated instance, fails on its own and is retried
   under the scheduler's ordinary Task retry rules, re-resolving `Service.*` when it runs again; a
   Task that no longer needs the Service completes normally.
2. Cancels *onRun* if it is still running (the readiness-timeout and UNHEALTHY cases) and waits
   for it to exit; constraint 5 forbids a second *onRun* in the Session before then.
3. If the number of relaunches so far is less than `restartPolicy.maxAttempts`:
    1. If `restartPolicy.completedTasks` is `RERUN`, returns every successfully completed Task in
       the scope to the queue.
    2. Relaunches the Service. After a start failure or host loss it MUST begin a new Service
       Session (constraint 9). After an instance failure on a host that is still available, it MAY
       relaunch *onRun* within the existing Service Session (same working directory, same ports,
       *onEnter* not re-run), preserving the state *onEnter* established; it SHOULD begin a new
       Service Session, which allocates new ports, when the failure may be a port conflict (*onRun*
       exited without ever becoming READY); or it MAY relocate.
4. Otherwise, the Service becomes FAILED and its scope fails: every Step in the scope fails, and
   with it the Job when the scope is every Step. The Service Session is then ended per constraint 6.

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

**`RERUN` and Step dependencies.** `RERUN` returns the completed Tasks of every Step in the
Service's scope to the queue, so each of those Steps returns to a pending state. A Step that depends
on a returned Step, directly or transitively, returns to pending as well, since the work it consumed
is produced again; Step dependencies are then resolved again from scratch. A Service that had been
stopped because its scope completed, and whose scope includes a returned Step, is started again, in
a new Service Session, when that Step next becomes schedulable. A Service cannot fail after its own
scope completes, so a Service whose scope is one Step never affects a Step outside it unless that
Step depends on a Step in it. Steps whose Tasks were running are handled by step 1 above.

**Interaction with Task failures.** A Task failure never fails a Service. A Task that fails
because it could not reach a Service that the scheduler still considers READY is an ordinary
Task failure. Authors SHOULD prefer a `COMMAND` or `STDOUT` health check when a plain TCP connect
is not a sufficient indicator that the service can do useful work.

#### Validation

Everything this RFC adds is checkable when a template is validated on its own, except the
`@fmtstring` numeric fields, which are checked when resolved at job creation (item 8), and one check
that needs the combined Job (below). At template validation, an implementation MUST check:

1. Every `Service.*` reference resolves to a Service and port declared in the same document's
   `services` or, in a Job Template, to a port declared by an entry of `requiresServices`, and is
   used only where the [scope rules](#the-service-scope) permit; in particular `bindAddress` is
   referenced only by the declaring Service, and a Step or Service that references an inline or
   required Service other than itself lists `service:<name>` in its `dependencies`, the message
   otherwise naming the missing entry.
2. No `Service.*` value appears in any `hostRequirements`, in a `<Service>`'s or a
   `<StepTemplate>`'s `let`, or in an Environment whose explicit `runScope` includes `SERVICE`.
3. `runScope` contains only recognized names, without duplicates.
4. `healthCheck` is consistent with `<ServiceActions>`, with `ports`, and with its own `type`:
   `onHealthCheck` is defined if and only if the type is `COMMAND`; every port a `TCP_CONNECT`
   check names is declared and has `protocol: TCP`; a Service none of whose ports is TCP has a
   `healthCheck` of type `STDOUT` or `COMMAND`; and a `STDOUT` check gives no
   `readinessIntervalSeconds`, and gives `failureThreshold` only together with
   `healthIntervalSeconds`.
5. Service, requirement, and port names are valid `<Identifier>`s, not `File`, and unique within
   their lists; no requirement has the same `name` as an inline Service.
6. The wrap hooks an Environment defines are exactly those its `runScope` calls for, per
   [`<Environment>`](#environment).
7. A template that lists `SERVICE` also lists `EXPR`.
8. No two ports of a Service with the same `protocol` have the same `port` number. For a `port`
   given as a format string this is checked when the value is resolved at job creation, as its
   range is.
9. Each `dependsOn` in a Step's or a Service's `dependencies` names a Step of the same Job
   Template or, as `service:<name>`, a Service of the same document's `services` or the Job
   Template's `requiresServices`, other than the entity listing it, with no Step or Service listed
   twice in one list; in an Environment Template every entry uses the `service:` form.
10. The graph formed by the `dependencies` of the document's Steps and Services is acyclic (see
    [`<StepDependency>`](#stepdependency)).
11. Every Service in a Job Template's `services` has a non-empty scope: some Step lists it, directly
    or through other Services, or some `jobEnvironments` entry references it (see
    [Service scope](#service-scope)).
12. No Step's `name` contains `:` (see [`<StepTemplate>`](#steptemplate)).
13. `requiresServices` appears only in a Job Template.

These checks appear in the wiki as §9.9. Two checks require the combined Job, because they relate
documents that only the scheduler sees together, and are performed at submission:

1. A wrapping Environment defined in a document that does not declare `SERVICE` — an attached
   Environment Template, or the Job Template itself — has the default `runScope` (every kind of
   Session) and cannot define the `onWrapService*` hooks. Service Sessions enter the Job's
   `jobEnvironments` only, so if such an Environment is in the combined Job's `jobEnvironments` and
   the combined Job has any Service, the submission MUST be rejected, naming the document that
   defines the Environment as the cause; see [Backward compatibility](#backward-compatibility) for
   how this arises and the remedy. A wrapping Environment in a Step's `stepEnvironments` is never
   entered by a Service Session and is not subject to this check.
2. Each entry of the Job Template's `requiresServices` matches exactly one attached Service with
   its `name`, declaring every listed port with the same `protocol`; see
   [Environment Template](#environment-template).

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
  by the `onRun` action of a Service whose health check *type* is `STDOUT`. Before the instance is
  READY it indicates that the service is accepting traffic on all of its declared ports, and makes
  the instance READY. After READY it is a heartbeat when the health check gives
  `healthIntervalSeconds`, and has no additional effect otherwise.

## Design Choice Rationale

### A new entity rather than extending `<Environment>`

An Environment is defined by *where* it runs: on the host of the Session, entered and exited around
that Session's Tasks. Every one of the four differences in [Motivation](#motivation) — cross-host
placement, scope-long lifetime, endpoint publication, health — contradicts that model. Adding
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

### Health is declared, not inferred

A scheduler cannot know when an arbitrary process is ready, nor whether a process that still holds
its port is answering. The three probe types cover the three places either is observable: the
network (`TCP_CONNECT`, the zero-configuration default), an external check (`COMMAND`, for
protocol-level health such as a `PING` or an HTTP `/healthz`), and the process itself (`STDOUT`, for
services whose authors can add one print statement).

One mechanism serves both phases because a probe that proves readiness is exactly the probe that
proves continued health; two mechanisms would double the schema for no new information. The two
phases get separate intervals because they trade probe cost against different things. Before READY
the cost is startup latency: a TCP connect is free, so it is retried every second, while a `COMMAND`
launches a process, so every five. After READY the cost is hang-detection latency, where three
failures thirty seconds apart is ample. A threshold rather than a single failed probe because one
failure is often a blip (a garbage-collection pause, a saturated accept queue) and a false restart
under `RERUN` is expensive. The `STDOUT` heartbeat is opt-in because the common `STDOUT` case is a
third-party binary wrapped with a one-line ready echo that has no way to repeat it, and *onRun*
exiting already covers its crash case. UNHEALTHY is an instance failure rather than a policy of its
own because a hung process is indistinguishable in effect from a crashed one, and `restartPolicy`
already encodes what the author wants done about either.

`healthCheck` is the Service-specific form of the monitoring that
[Issue #133](https://github.com/OpenJobDescription/openjd-specifications/issues/133) proposes for
Environment actions in general. #133 should adopt the same `healthIntervalSeconds` and
`failureThreshold` vocabulary so that the two do not diverge.

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

### Dependencies are declared, and one list names both kinds

A Step that uses a Service depends on it, and the template says so where it says everything else
about what a Step waits for: in `dependencies`, with `dependsOn: service:<name>` beside
`dependsOn: <StepName>`. A Service's own `dependencies` names the Steps and Services it waits for
the same way. Three things follow from making the dependency a declaration rather than inferring it
from `Service.*` references. The graph is visible in one place: a reader, a UI, or a scheduler
learns which Steps need which Services, and which Services need which, from the `dependencies`
lists alone, without walking every format string in the template. An opaque consumer is
expressible: a Step whose Tasks reach a Service through a configuration file, a wrapper script, or
an environment variable set outside the template never references `Service.*`, yet it depends on the
Service all the same, and a declaration can say so where an inference could not. And a reference
without a dependency is a diagnosable mistake instead of an implicit edge: a template that reads
`Service.Cache.main.port` in a Step that does not list `service:Cache` is rejected with a message
that names the missing entry, where inference would have silently extended the Service's lifetime
to that Step. Starting is gated by the declared edges, stopping runs them backwards, a cycle is a
validation error exactly as a cycle among Step dependencies is, and list order carries no meaning,
so reordering a `services` list or merging two never changes a Job.

`dependsOn` is reused with a prefix rather than given a second list (`dependsOnServices`, say)
because a dependency on a Service is the same kind of fact as a dependency on a Step, differing
only in what satisfies it: completion for a Step, READY for a Service. One vocabulary means one
place to look, one cycle check over one graph with every kind of edge, and no rule for how two lists
interact. The alternative model in [Services as a kind of Step](#services-as-a-kind-of-step) would
express this as `dependsOn` with a `when: READY` condition; a dependency on a running Service is
exactly what that condition means, and the `service:` prefix says it where the condition would.

The constraint that a Step's `name` not contain `:` makes the prefix unambiguous, and it is gated
on `SERVICE` because the base specification should not change for an extension: a template that
does not declare `SERVICE` may still name a Step `service:Cache`, and `dependsOn: service:Cache`
in such a template names that Step as it always has.

### External Services are declared with their ports

A Job Template that reads a queue Service's address from an environment variable has an undeclared
dependency: `openjd check` passes, and the Job fails at run time on a queue that lacks the Service.
A `requiresServices` entry names the dependency where tools can see it, and listing the ports makes
every `Service.<name>.<port>.*` reference in the template checkable against the declaration alone,
exactly as references to an inline Service are. The two declarations answer different questions: a
requirement declares the contract with the queue, and a Step's `dependsOn: service:<name>` declares
which Steps use it, as it does for an inline Service. The ports also let the scheduler check the
match from the stub alone: it reads the requirement, finds the attached Service, and compares port
names and protocols, without reading the template's format strings. A requirement that named the
Service but not its ports would leave every port reference unchecked until submission and the stub
insufficient for the scheduler, which would then have to walk the template to learn what was used.

### External Services via the Environment Template, not a new template type

A separate `servicetemplate-2023-09` root element was rejected for its deployment surface. Every
scheduler that supports Environment Templates already has the plumbing to attach a document to a
queue, order the attachments, merge their parameter definitions, and apply them to each submission.
Expressing a Service as a variant of that same document means Services work everywhere Environment
Templates do with no new integration; a second root element would require every implementation to
add a second attachment type, UI, and API, and a rule for interleaving the two lists.

### Inline Services shadow external ones

Requiring an external Service's name to differ from every Service in the Job Template would let a
queue operator's choice of name break a Job Template that works everywhere else. Instead a Job
Template's `Service.*` references resolve to its own `services` first, and an attached Service with
the same name is simply not visible to that template; it still runs, and still reaches the Job's
Tasks through its own Environment, so neither party's template changes meaning because of the
other's name. The one place a name must be unique is where the Job Template asks for it: a
requirement names an external Service, so two attachments that both supply it are an error for that
submission, and only for one that requires it.

### One document may define Services and an Environment together

Supplying a Service to a queue is usually two things: the process, and the client-side configuration
that lets existing tools find it. Allowing one Environment Template to define both mirrors how a Job
Template composes `services` with `jobEnvironments`. It also keeps a Job Template's requirement
optional: the Environment that configures clients for a Service lives next to that Service, so a Job
Template written before the Service existed uses it through environment variables, and only a
template that reads the endpoint itself declares a requirement. Within an Environment Template every
`Service.*` reference resolves in the same document, so the validity of an attachment never depends
on the order in which a scheduler happens to attach templates.

### Services run inside the Job's Environments

Entering the Job's Environments in the Service Session makes a Service Session a Session in the
ordinary sense, so every existing rule about Environments (ordering, `openjd_env`, cleanup on
failure, wrap hooks) applies unchanged, and an Environment Template written for Tasks works for
Services. A Service enters no Step's `stepEnvironments` because it belongs to no Step: its scope may
be one Step, several, or all of them, and a Service shared by two Steps with different Step
Environments could not enter both sets. A Service that needs provisioning beyond the Job's
Environments does it in its own *onEnter*.

### `runScope` is an explicit field whose default follows the reference

Two kinds of Environment exist once Services do: those that provision a process (Conda, a container)
and belong in every Session, and those that configure Tasks to *use* a Service and belong only in
Task Sessions. The alternative to a field is pure inference: an Environment that references a
Service is excluded from the Sessions of that Service and of the Services started before it. But
that rule depends on the start order, which for attached templates is the scheduler's attachment
order, unknowable from the template, and it gives an author no way to say that a provisioning
Environment should stay out of Service Sessions for a reason of their own. `runScope` makes the
distinction a declaration that any reader can see.

The default is decided from the Environment's own text rather than being fixed, because the common
case is unambiguous: an Environment that references `Service.*` cannot be entered in a Service
Session, where the value it references may not exist yet, so the only runScope it can have excludes
`SERVICE`, and `[TASK]` is the one that exists today. Writing `runScope: [TASK]` on every such
Environment would be a tax on the pattern this RFC exists to make easy. Because the default is
computed from one document, attachment order is never involved, and an Environment Template's
Environment has the same runScope on every queue.

### `onHealthCheck` is an action of the Service

The alternative is an `<Action>` inside `healthCheck`. That leaves it outside `script:`, with no
principled claim on `let` bindings, embedded files, or the same format-string scopes as the other
actions. Making it a fourth `<ServiceActions>` entry gives every action of a Service one scope rule
and lets check scripts use embedded files. The `healthCheck` object keeps the policy (type,
intervals, timeout, threshold); the action keeps the command.

The check runs while *onRun* is running, which no other action in the specification does. The
alternative, dropping `COMMAND` and having authors wrap their binary in a script that starts it in
the background, polls it, prints `openjd_service_ready`, and keeps polling to repeat it, pushes onto
every third-party binary the boilerplate OpenJD exists to remove, hides the health policy in a shell
script, and does not work for a bare-binary *onRun* without a shell. The rules in [Concurrency with
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
process with its own ports, health check, restart policy, and host requirements, and may need a
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

## Prior Art

* **Kubernetes** — `Service` + `Deployment` with `readinessProbe` and `livenessProbe`
  (`tcpSocket`, `exec`, `httpGet`; `periodSeconds`, `failureThreshold`) and `restartPolicy`. Our
  three probe types map directly onto `tcpSocket`, `exec`, and a stdout signal in place of `httpGet`
  (which would require HTTP semantics in the specification); one `healthCheck` with two intervals
  does the work of Kubernetes' two probes. Kubernetes services are not scoped to a batch of work,
  which is the gap this RFC fills.
* **Docker Compose** — `depends_on: { condition: service_healthy }` plus `healthcheck` with
  `interval`, `retries`, and `start_period`, which `healthIntervalSeconds`, `failureThreshold`, and
  `readinessTimeoutSeconds` correspond to. Its start-until-healthy ordering along the `depends_on`
  graph is the model for ours, declared in `dependencies` as Compose declares it in `depends_on`.
* **GitHub Actions service containers** — `services:` in a job, started before steps and stopped
  after, with ports published to the runner. Scoped to one job on one host; the closest
  batch-oriented precedent for a Service whose scope is one Step.
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
  is developed against the `SERVICE` conformance suite that accompanies this RFC and the rest of
  the 2023-09 suite. It implements the schema, validation, and job-creation rules in
  `openjd-model`, the host and port functions in `openjd-expr`, the Service Session in
  `openjd-sessions`, and a single-host scheduler for `openjd run` in `openjd-cli`. The prototype's
  largest single piece was the Service Session: the existing Session held one current action, one
  log stream, and one cancellation target, and running `onHealthCheck` alongside `onRun` needed a
  second action slot with its own cancellation and per-action attribution on captured output. The
  health phase adds a probe loop that outlives readiness, counts consecutive failures, and turns
  the threshold into a cancellation of `onRun`.
- **Python (openjd-model, openjd-sessions)**: The Python packages are intended to become bindings
  over openjd-rs. The `v0` releases of `openjd-model-for-python` and `openjd-sessions-for-python`
  would therefore not gain this extension; a `v1` built on the bindings would inherit it, exposing
  the new models, the Service Session, and the `Service.*` scope.
- **Schedulers**: This is where most of the work lives. A scheduler must place Services, allocate
  ports, guarantee cross-host reachability over each port's protocol, gate Task scheduling on
  readiness, keep probing after READY, track instance failures, implement `KEEP`/`RERUN`, and stop
  Services at scope end. A
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

A Service could be a Step whose single long-running Task other Steps depend on. Three things are
missing from a Step for that, and each can be supplied by a Step feature: a dependency that is
satisfied when the upstream Step is *running and ready* rather than finished (`dependsOn` with a
`when: READY` condition and a health check beside `onRun`); a `ports:` list on the Step so that its
Task can publish an endpoint through `Step.<name>.<port>.*`; a completion condition so that the
scheduler stops the Task once the Steps that depend on it finish; and a `providedTo` list, or a
rule, that says which Steps that is. With those four, a Service is a Step with a health check,
ports, a stop condition, and a consumer list.

This RFC rejects that for five reasons. First, each of the four changes the Step and Task state
machines that every scheduler and every UI already implements: a Step that is "ready" but not
"completed", a Task that is stopped on purpose without failing, a dependency satisfied by a state
other than completion. A new entity is extension surface that an implementation without `SERVICE`
never sees, and leaves the base state machines alone. Second, four orthogonal knobs yield
combinations that need their own validation rules (a `when: READY` dependency on a Step with no
health check; ports on a Step with ten thousand Tasks; a completion condition on a Step nothing
depends on), and a reader must assemble "this is a service" from parts, where the entity name says
it. Third, a health check beside `onRun` lands in every Task Session's hot path, a probe loop that
every Task of every Step would have to be able to run, where a new Session kind keeps it to the
Sessions that need it. Fourth, readiness for a multi-Task Step would have to be "READY when every
Task has passed its probe", which couples one Task's relaunch to the completed Tasks of every
dependent Step; a Service has one instance, and `completedTasks` says what its relaunch means.
Fifth, the sub-features that are valuable on Steps in their own right deserve their own proposals:
a health check on batch Tasks for hang detection, or ports on the Tasks of a worker Step so that
peers can find one another (see [Future Work](#future-work)). When they land, `<Service>` should
adopt their vocabulary rather than the other way around.

Of the cases the Step model would cover, this RFC covers a Service shared by some Steps and not
others (declared dependencies, which give a Service its scope), a Service that starts only after a
batch Step has finished (a Service's `dependencies` on a Step), and the queue case (`services` in
an Environment Template with `requiresServices` in the Job Template). It does not cover a Step that
runs after a Service's *onExit*, for example to collect a report the Service wrote on shutdown; a
Step's dependency on a Service is satisfied by the Service being READY, not by its having stopped,
and the Service's *onExit* must do that work itself.

### Scope inferred from `Service.*` references

A Service's scope could be computed from the template's format strings: a Step that references
`Service.X.*` is in `X`'s scope, a Service that references another depends on it, and a Service
nothing references serves the whole Job, with no `dependencies` entry to write. Rejected because
it hides the dependency graph in format strings, where only a tool that evaluates every expression
in the template can recover it; because it cannot express an opaque consumer, a Step that reaches
the Service through a file or a wrapper and never mentions `Service.*`, which would have to
reference the endpoint pointlessly to be placed in the scope; and because a reference without a
dependency cannot be diagnosed, since the reference *is* the dependency, so a stray
`Service.Cache.*` in a Step that should not need the cache silently extends the cache's lifetime
instead of being reported. Declared dependencies cost one line per edge and make all three visible.

### `serviceEnvironments`, a Service-scoped Environment list

A `<Service>` could carry its own list of Environments, the analogue of `stepEnvironments`, entered
only in its Session. Rejected because it offers nothing *onEnter* does not. Within a Job Template,
Environments are defined inline, so a Service-scoped list gives no reuse: the `conda create` that
provisions a Valkey is the same command in either place. `stepEnvironments` earn their place by
amortizing setup across the many Tasks a Session runs, but a Service Session already runs one
*onEnter*, once per Session, before *onRun*, so there is nothing to amortize. Wrapping one Service
in a container inside the template is simply an *onRun* that invokes the container; the wrap-hook
machinery exists for queue-supplied wrappers, which enter every Service Session through the Job's
Environments and never needed a per-Service list. A per-Service list could return as future work if
a per-Service *external* wrapper is ever needed.

## Future Work

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
