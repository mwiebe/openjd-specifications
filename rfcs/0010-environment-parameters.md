* Feature Name: Environment Parameters
* Author(s): Mark Wiebe <[mwiebe](https://github.com/mwiebe)>
* RFC Tracking Issue: (pending). The number `0010` is a placeholder assigned before a tracking issue exists, and may
  change when one is created.
* Start Date: 2026-10-17
* Specification Version: 2023-09 extension ENV_PARAMS
* Accepted On: (pending)
* Depends On:
  * RFC 0002, Model Extensions (https://github.com/OpenJobDescription/openjd-specifications/issues/57)
* Related:
  * Discussion #198, the idea thread for this RFC (https://github.com/OpenJobDescription/openjd-specifications/discussions/198)
  * RFC 0005, Expression Language (`EXPR`), for typed values and list-typed parameters
  * RFC 0008, Environment Wrap Actions (`WRAP_ACTIONS`)
  * RFC 0009, Service (`SERVICE`), whose `service:` dependency form this RFC's `environment:` form follows, and
    whose earlier drafts defined the `runScope` field that this RFC now owns
  * RFC 0011, Control Flow (`CONTROL_FLOW`), whose `enableIf` on an Environment composes with this RFC's values

## Summary

A queue's Environment, such as a Conda or Docker Environment, is configured once per Job from Job
Parameters, so a Job whose Steps need different software must install everything in every Session or
be split into several Jobs. This RFC lets each Step tell such an Environment what it needs. An
`<Environment>` declares the parameters a Step may set in a `parameterDefinitions` block and reads them
as `{{Env.Param.<name>}}`; a Step requests the Environment and supplies values through an
`environment:` entry in its `dependencies`. The Environment may be one of the Job Template's own or one
the queue attaches, and the same Job Template still runs on a queue that attaches neither. An
Environment may also declare `runScope: ONLY_REQUESTED`, so a queue can offer Environments that only
the Steps that ask for them enter, and `hostRequirements`, so an Environment can keep Steps off hosts
where it cannot run. Values are fixed when the Job is created, and a Session runs Tasks only of Steps
whose values match the ones it entered its Environments with, so a scheduler can group Sessions by
them.

## Basic Examples

### A queue Conda Environment that each Step configures

A queue administrator attaches this Environment Template. The template-level `CondaPackages` Job
Parameter keeps working as it does today for Job Templates that know nothing about this extension:
the Environment parameter `Packages` defaults to it.

```yaml
specificationVersion: "environment-2023-09"
extensions: [ENV_PARAMS]
parameterDefinitions:
  - name: CondaPackages
    type: STRING
    default: ""
  - name: CondaChannels
    type: STRING
    default: "conda-forge"
environment:
  name: Conda
  parameterDefinitions:              # new: values a Step may supply
    - name: Packages
      type: STRING
      default: "{{Param.CondaPackages}}"
    - name: Channels
      type: STRING
      default: "{{Param.CondaChannels}}"
  script:
    actions:
      onEnter:
        command: bash
        # Values reach the script as argv, never as script text.
        args: ["{{Env.File.Enter}}", "{{Env.Param.Packages}}", "{{Env.Param.Channels}}"]
    embeddedFiles:
      - name: Enter
        type: TEXT
        data: |
          #!/usr/bin/env bash
          set -euo pipefail -f
          PACKAGES="$1"
          if [ -z "$PACKAGES" ]; then
            echo "No conda packages requested; using the host's software."
            exit 0
          fi
          CHANNELS=()
          for c in $2; do CHANNELS+=(-c "$c"); done
          conda create -y -p ./.conda-env "${CHANNELS[@]}" $PACKAGES
          echo "openjd_env: PATH=$PWD/.conda-env/bin:$PATH"
          echo "openjd_env: CONDA_PREFIX=$PWD/.conda-env"
```

This Environment keeps the default `runScope: ALL`, so every Session enters it and a Step that asks for
nothing gets the job-wide `CondaPackages`, exactly as before. A queue that offers several ways to
provision software (Conda, Rez, and Docker, say) can instead mark each `runScope: ONLY_REQUESTED`:
then a Session enters only the Environments its Step requests, and the `if [ -z "$PACKAGES" ]`
branch above is taken only by a Step that requests `Conda` without any packages.

### A Job Template whose Steps each ask for their own software

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [ENV_PARAMS]
name: "Video To Composite"
steps:
  - name: DecodeFrames
    dependencies:
      - environment: Conda
        required: false
        parameters:
          Packages: "ffmpeg=7"
      - environment: Docker
        required: false
        parameters:
          Image: "docker.io/linuxserver/ffmpeg:latest"
    script: ...
  - name: SolveSfM
    dependencies:
      - dependsOn: DecodeFrames
      - environment: Conda
        required: false
        parameters:
          Packages: "colmap=3.11"
      - environment: Docker
        required: false
        parameters:
          Image: "docker.io/colmap/colmap:latest"
    script: ...
```

A Session for `SolveSfM` enters `Conda` with `{{Env.Param.Packages}}` equal to `colmap=3.11`, and a
Session for `DecodeFrames` enters it with `ffmpeg=7`. On a queue that attaches a Docker Environment
instead, the `Docker` entries take effect and the `Conda` entries are ignored. On a queue where the
software is preinstalled and neither Environment is attached, both entries are ignored and the Job
runs unchanged. The complete pipeline, with both Environment Templates, is in
[`samples/v2023-09`](../samples/v2023-09); there the Docker entries also name the host paths to
mount in the container and, for `SolveSfM`, request the host's GPUs.

`required: false` covers a queue that attaches no Environment of the name. It does not cover a
queue whose `Conda` Environment declares no `Packages` parameter, such as an Environment Template
written before this extension: there the entry resolves, it supplies a parameter the Environment
does not have, and the submission is rejected, naming the entry. See
[Backward compatibility](#backward-compatibility) for how a queue administrator migrates such an
Environment.

### One parameterized Job Environment serving several Steps

An `environment:` entry can name a Job Environment of the same template, so one Environment
definition replaces several near-identical Step Environments. With `runScope: ONLY_REQUESTED`, only
the Steps that request the Environment enter it.

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [ENV_PARAMS]
name: "Plugin-Selected Renders"
jobEnvironments:
  - name: RenderLicense
    runScope: ONLY_REQUESTED
    parameterDefinitions:
      - name: Product
        type: STRING
        allowedValues: ["arnold", "redshift", "vray"]
      - name: Seats
        type: INT
        default: 1
        minValue: 1
    script:
      actions:
        onEnter:
          command: license-checkout
          args: ["--product", "{{Env.Param.Product}}", "--seats", "{{Env.Param.Seats}}"]
        onExit:
          command: license-release
          args: ["--product", "{{Env.Param.Product}}"]
steps:
  - name: BeautyPass
    dependencies:
      - environment: RenderLicense
        parameters: { Product: "arnold" }
    script: ...
  - name: VolumePass
    dependencies:
      - environment: RenderLicense
        parameters: { Product: "redshift", Seats: 2 }
    script: ...
  - name: Composite
    dependencies:
      - dependsOn: BeautyPass
      - dependsOn: VolumePass
    script: ...
```

`Product` has no default, so every Step that requests `RenderLicense` must supply it. `Composite`
does not request it, so its Sessions never check out a license. The two render Steps supply
different values, so their Tasks never share a Session. Had `RenderLicense` kept the default
`runScope: ALL`, every Session would enter it and every Step, including `Composite`, would have to
supply `Product`.

### Choosing between Environments a queue provides

A queue may attach both `Conda` and `Docker`. The pipeline above would then install COLMAP with
conda inside a COLMAP container, which is wasteful. With `EXPR`, a Step can read `Job.EnvNames`,
the names of the Environments the Job has once the queue's are attached, and make an entry conditional
with `enableIf`:

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [ENV_PARAMS, EXPR]
name: "Video To Composite"
steps:
  - name: SolveSfM
    dependencies:
      - environment: Conda
        required: false
        enableIf: "{{ 'Docker' not in Job.EnvNames }}"
        parameters:
          Packages: "colmap=3.11"
      - environment: Docker
        required: false
        parameters:
          Image: "docker.io/colmap/colmap:latest"
    script: ...
```

On a queue with both, the `Conda` entry is disabled: the Step does not request `Conda`, so if `Conda`
is `ONLY_REQUESTED` the Step's Sessions never enter it, and if it is `ALL` they enter it with its
defaults, as a Step with no entry would. On a queue with `Conda` alone, the entry is enabled and COLMAP
is installed with conda.

### An Environment that states what hosts it needs

An Environment may declare `hostRequirements`, and they may read its parameters. A Session that enters
the Environment must satisfy them in addition to its Step's own requirements, so a Docker Environment
can require a GPU only for Steps that ask for one, and a container-capable host for every Step that
requests it:

```yaml
specificationVersion: "environment-2023-09"
extensions: [ENV_PARAMS, EXPR, FEATURE_BUNDLE_1]   # FEATURE_BUNDLE_1 for a format-string `min`
environment:
  name: Docker
  runScope: ONLY_REQUESTED
  parameterDefinitions:
    - name: Image
      type: STRING
    - name: Gpus
      type: STRING
      default: ""
  hostRequirements:
    attributes:
      - name: attr.worker.os.family
        anyOf: [linux]
    amounts:
      - name: amount.worker.gpu
        min: "{{ 1 if Env.Param.Gpus != '' else 0 }}"
  script: ...
```

A Step that requests `Docker` with `Gpus: "all"` is placed only on a Linux host with at least one GPU; one
that requests it without `Gpus` needs only Linux. A Step that does not request
`Docker` is unaffected. On a fleet where only some hosts run containers, this is what keeps the Steps
that asked for a container off the others.

### Execution

For the Video To Composite Job on a Conda queue, with two Worker Hosts:

```
Worker host A                                Worker host B
-------------                                -------------
Session 1 (DecodeFrames)                     Session 2 (SolveSfM, after DecodeFrames)
  Conda.onEnter  Packages="ffmpeg=7"           Conda.onEnter  Packages="colmap=3.11"
    DecodeFrames task                            SolveSfM task
  Conda.onExit                                 Conda.onExit
```

Two Steps that supply equal values for every Environment, such as `DecodeFrames` and `EncodeVideo`
in the sample, may share a Session: a scheduler may run `EncodeVideo`'s Tasks in Session 1 if it is
still open, since its `Conda` was entered with the same `Packages`. Two Steps that supply different
values cannot share one.

## Motivation

### One Job, several applications

A multi-application pipeline is a natural fit for one Job: decode a video into frames with ffmpeg,
solve structure-from-motion with COLMAP, place 3D objects into the solved scene with Maya, render
the composite, and encode the result with ffmpeg. Each Step needs different software, and the Step
dependencies between them are exactly what a Job expresses well.

Today the software for such a Job usually comes from an Environment Template the queue attaches, for
example a Deadline Cloud Conda or Docker queue environment. That Environment takes its configuration
from a single template-level Job Parameter (`CondaPackages`, `DockerImage`), so it does the same thing
in every Session of the Job. The Job's author has three options, none good:

1. **Install everything everywhere.** List ffmpeg, COLMAP, and Maya in `CondaPackages`. Every Session
   resolves and installs all three, which multiplies Session startup time and disk use, and fails
   outright when the packages' dependencies conflict, as they can when two applications pin
   different versions of the same shared libraries (Qt, OpenEXR, or Python, for example).
2. **Build one image with everything.** The same conflict problem, moved into an image build that the
   Job author may not control, and an image that is large to pull on every host.
3. **Split the pipeline into several Jobs.** Each Job gets its own `CondaPackages` value, but the
   dependencies between Steps are lost, and the submitter must chain Jobs by hand.

A Step Environment cannot help, because it has no way to ask an Environment the queue attached to do
anything differently. The Job Template could stop relying on the queue and install software in its own
Step Environments, but then it hard-codes one provisioning mechanism. That works against the Portable
[Design Tenet](https://github.com/OpenJobDescription/openjd-specifications/wiki/Design-Tenets), which asks
that jobs be portable to many different systems and usable with the technology stack of their choice:
applied here, the same Job Template should run whether a Conda, Rez, Docker, or Apptainer Environment
supplies its software.

With this RFC, each Step states what it needs from each kind of provisioning Environment it may meet,
and the queue's Environment does exactly that for each Step's Sessions.

### Other workflows

1. **Per-Step container images.** A Docker or Apptainer wrapping Environment (RFC 0008) runs every
   action of a Session inside one container. A pipeline with a GPU training Step and a CPU
   preprocessing Step wants a CUDA image for the first and a slim image for the second. With an `Image`
   Environment parameter each Step names its image, and the wrapping Environment is written once.
2. **Per-Step license or plugin selection.** A render Job with a beauty pass in one renderer and a
   volume pass in another must check out a different license for each, or load a different plugin set
   into the same DCC. A Job Environment with a `Product` parameter does this for every Step, where
   today the author writes one Step Environment per Step that differs only in a string.
3. **Per-Step resource tuning of a shared Environment.** A queue Environment that mounts a dataset or
   warms a cache can take a size or a subset per Step (`Subset: "train"` for one Step, `"eval"` for
   another), so each Session prepares only what its Tasks read.
4. **Software version pinning per Step.** A Job that validates a scene across two Maya versions has two
   Steps that differ only in the Maya version, which is an Environment parameter rather than a
   duplicated Job.
5. **A queue that offers a menu of Environments.** A queue administrator wants to make Conda, Rez, and a
   GPU driver setup available without imposing all three on every Session. Today each such Environment
   must be written to do nothing unless a Job Parameter asks for it, and every Session still enters and
   exits it. With `runScope: ONLY_REQUESTED` the queue attaches all of them and each Step enters only the
   ones it requests.
6. **A Job Environment that only some Steps need.** A license checkout, a database connection, or a
   large asset download that two of a Job's five Steps use is today written as a Step Environment on each
   of the two, or as a Job Environment that every Session pays for. An `ONLY_REQUESTED` Job Environment
   is written once and entered twice.
7. **An Environment that only works on some hosts.** A queue's Docker Environment needs a host with a
   container runtime; its Conda Environment, on a mixed fleet, needs Linux. Today an Environment Template
   cannot say so, and the Steps it serves fail at `onEnter` on the wrong host. With `hostRequirements` on
   the Environment, every Session that enters it is placed where it can run, and when the requirement
   depends on a parameter (a GPU only when `Gpus` is set) it applies to exactly the Steps it should.

These patterns are common wherever a farm provisions software per Session: Conda- and Rez-based VFX
farms, containerized HPC and ML batch systems, and CI systems that run matrix jobs. The
[Prior Art](#prior-art) section shows that every comparable workflow system lets a unit of work choose
its own software environment.

### Why the values come from the Step

A Session enters each of its Environments once and then runs Tasks, which may come from one Step or
from several whose Environments are identical. The Step is the unit that knows what software its
Tasks need, so it is the natural place to say what an Environment should do for them. With this
extension, a Session's Environments are entered with the values of the Steps it serves, and a Session
serves several Steps only when their values agree, so one set of values is in force for the whole
Session. Values are fixed per Step rather than per Task: a Session runs many Tasks, and an Environment
is entered once for all of them, which is the amortization Environments exist to provide.

### Backward compatibility

This RFC is additive and gated by the `ENV_PARAMS` extension name declared under RFC 0002.

- A document that declares `parameterDefinitions`, `runScope`, or `hostRequirements` on any `<Environment>`, or that
  uses the `environment:` form of `<StepDependency>`, MUST list `ENV_PARAMS` in its `extensions`. A scheduler
  that does not implement `ENV_PARAMS` rejects such a document under RFC 0002.
- `ENV_PARAMS` does not require `EXPR`. Without `EXPR`, Environment parameters are limited to the
  types available to Job Parameters without it (`STRING`, `INT`, `FLOAT`, `PATH`), as Job Parameters
  are.
- The extension applies per document, as RFC 0009 established for Environment Templates. A Job
  Template that does not list `ENV_PARAMS` runs with an Environment Template that does: every
  Environment parameter takes its default, which is how a queue administrator keeps existing Job
  Templates working while adding per-Step configuration. Such a Job Template cannot request anything,
  so an attached `ONLY_REQUESTED` Environment is never entered by its Sessions; a queue administrator
  who wants existing Job Templates to keep getting an Environment leaves its `runScope` at `ALL`. An
  Environment Template that does not list `ENV_PARAMS` runs with a Job Template that does: its
  Environment has `runScope: ALL` and no parameters, so the Job Template's `environment:` entries that
  name it find no parameters to set, which is an error only if they supply any (see
  [`<StepDependency>`](#32-stepdependency) constraint 5).
- A queue Environment written before this extension under a [recommended name](#recommended-names-non-normative)
  rejects Job Templates that supply its recommended parameters, even with `required: false`, because
  the entry resolves to an Environment that lacks them ([§3.2.1](#321-resolving-external-environments)
  item 1). Deadline Cloud's console Conda queue environment is such a case today: it is named `Conda`
  and takes `CondaPackages` and `CondaChannels` as Job Parameters only, so it rejects the pipeline
  sample. So is its sample Rez queue environment (`rez_queue_env.yaml`), named `Rez`, which takes
  `RezPackages` as a Job Parameter only. The remedy is for the queue administrator to add
  `ENV_PARAMS` and the recommended `parameterDefinitions`, with defaults that read the existing Job
  Parameters, as `conda-env-params.yaml` does. That change is itself backward compatible: Job
  Templates that do not supply the parameters see the same job-wide values as before. The rejection
  names the entry and the parameter, so a submitter can see which queue Environment needs updating. The rationale is in
  [An Environment without the supplied parameter is an error](#an-environment-without-the-supplied-parameter-is-an-error).
- No existing field changes meaning. A Job with no `environment:` entries, no Environment
  `parameterDefinitions`, and no `runScope` behaves exactly as today, including which Environments
  each Session enters and how Sessions are shared between Steps. A Job Template that lists
  `ENV_PARAMS` but no `environment:` entries, run with an Environment Template that declares
  parameters and keeps `runScope: ALL`, also shares Sessions as today: every Step takes the same
  defaults, so every Step's values are equal.

## Specification

### Terminology

| Term | Definition |
|------|------------|
| **Environment parameter** | A value declared in an `<Environment>`'s `parameterDefinitions`, read inside that Environment as `Env.Param.<name>`. Distinct from a Job Parameter, which is declared in a template's top-level `parameterDefinitions` and read as `Param.<name>`. |
| **Request** | A Step (or, with `SERVICE`, a Service) requests an Environment when its `dependencies` contain an enabled `environment:` entry naming it: one whose `enableIf` is absent or `true`. The entry may also supply parameter values. |
| **Requesting entity** | The Step or Service whose `dependencies` contain the `environment:` entry. Its Sessions enter the named Environment with the values the entry supplies. |
| **Run scope** | An Environment's `runScope`: which Sessions enter it. `ALL` means every Session of the Job; `ONLY_REQUESTED` means only the Sessions of the entities that request it. |
| **Combined host requirements** | For one Step (or Service), the conjunction of its own `hostRequirements` and those of every Environment its Sessions enter, each Environment's evaluated with the Step's effective values. A Session running the Step's Tasks must be placed on a host that satisfies all of them. |
| **Effective values** | For one Step (or Service) and one Environment its Sessions enter, the value of each of the Environment's parameters: the value the Step's enabled entry supplies if it supplies one, otherwise the parameter's resolved `default`. Fixed when the Job is created. |
| **Equal values** | Effective values with the same type and the same resolved value before path mapping. |
| **A Session's values** | For one Session and one Environment it has entered, the parameter values it entered the Environment with. They equal the effective values of every Step whose Tasks the Session runs. |
| **External Environment** | An Environment the scheduler attaches to a Job from an Environment Template, as opposed to one of the Job Template's own `jobEnvironments`. |

### Schema modifications

> Changes to [the template schema](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas).

#### Job Template

> A modification to [`1.1. Job Template`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#11-job-template)

The Job Template schema is unchanged. Extension list addition to item 3 (*extensions*): `ENV_PARAMS`
(this RFC). Item 7 (*jobEnvironments*) says the Environments "are entered in the order provided at the start of every
Session for Tasks in the Job"; add after that sentence: "With the `ENV_PARAMS` extension, an Environment whose
`runScope` is `ONLY_REQUESTED` is entered only in the Sessions of the Steps that request it (see
[`<Environment>`](#4-environment))."


#### Environment Template

> A modification to [`1.2. Environment Template`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#12-environment-template)

An Environment Template must be able to declare the extension. This adds the same two keys, with the
same meaning, as RFC 0009; if RFC 0009 is accepted first, this section is already satisfied.

```diff
  specificationVersion: environment-2023-09
+ $schema: <string> # @optional
+ extensions: [ <ExtensionName>, ... ] # @optional
  parameterDefinitions: [ <JobParameterDefinition>, ... ] # @optional
  environment: <Environment>
```

* *$schema* — Ignored, as on the Job Template. Allowed for compatibility with JSON-editing IDEs.
* *extensions* — If provided, a non-empty list of extensions that this document uses, with the same
  available names as the Job Template. An extension listed here applies to the content of this
  document only; a Job Template does not need to list the extensions used by the Environment
  Templates a scheduler applies to it, and vice versa.

> A modification to [`1.2.1. Merging Environment Template Parameter Definitions`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#121-merging-environment-template-parameter-definitions)

Add at the end of the section:

> The merge applies to the top-level `parameterDefinitions` of each template only. The
> `parameterDefinitions` of an `<Environment>` (`ENV_PARAMS`) are never merged: each Environment's
> parameters are a namespace of their own, even when two Environments, or an Environment and a Job
> Parameter, use the same name. An Environment parameter's `default` that references `Param.<name>`
> or `RawParam.<name>` reads the merged Job Parameter's value, so a Job Template that redefines a
> Job Parameter's `default` changes the default of every Environment parameter that refers to it.

#### `<Environment>`

> A modification to [`4. <Environment>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#4-environment)

```diff
  name: <EnvironmentName>
  description: <Description> # @optional
+ parameterDefinitions: [ <EnvironmentParameterDefinition>, ... ] # @optional @extension ENV_PARAMS
+ runScope: <RunScope> # @optional @extension ENV_PARAMS
+ hostRequirements: <HostRequirements> # @optional @extension ENV_PARAMS
  script: <EnvironmentScript> # @optional
  variables: <EnvironmentVariables> # @optional
```

New properties:

* *parameterDefinitions* — The parameters that a Step whose Sessions enter this Environment may supply values for,
  through an `environment:` entry in its *dependencies* (see [`<StepDependency>`](#32-stepdependency)). Their values
  are available within this Environment as `Env.Param.<name>` and `Env.RawParam.<name>`. See
  [`<EnvironmentParameterDefinition>`](#45-environmentparameterdefinition-extension-env_params). Available with the `ENV_PARAMS` extension.
  Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. Maximum number of elements: 50.
    3. No two definitions in this list may have the same value for the `name` property. A name may coincide with a Job
       Parameter name or with a parameter of another Environment; they do not conflict.
    4. Permitted on a Job Template's *jobEnvironments* entries and on an Environment Template's *environment*. Not
       permitted on a *stepEnvironments* entry.
* *runScope* — Which Sessions enter this Environment. Available with the `ENV_PARAMS` extension. `<RunScope>` is one
  of:
    * `ALL` — Every Session of the Job enters this Environment, in its position in the Job's Environment order. This is
      the behavior of every Job Environment and attached Environment without the extension, and the default.
    * `ONLY_REQUESTED` — Only the Sessions of a Step that requests this Environment, through an `environment:` entry in
      its *dependencies* (see [`<StepDependency>`](#32-stepdependency)), enter it. Any other Session does not enter or
      exit it, does not have its *variables* set, and does not have it in its Environment stack. The Environments a
      Session does enter keep their relative order.

  Constraints:
    1. An implementation MUST reject a `<RunScope>` it does not recognize.
    2. Permitted on a Job Template's *jobEnvironments* entries and on an Environment Template's *environment*. Not
       permitted on a *stepEnvironments* entry, which is entered only by its own Step's Sessions.
* *hostRequirements* — Requirements on the Worker Host that every Session entering this Environment must satisfy, in
  addition to those of the Step (or, with `SERVICE`, the Service) whose Sessions they are. Available with the
  `ENV_PARAMS` extension. The object is a [`<HostRequirements>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#33-hostrequirements)
  with the same shape and constraints as on a `<StepTemplate>`, with these differences:
    1. Its format strings may reference `RawParam.*` and `Param.*` (not PATH or LIST[PATH]) of the Job Parameters of
       the document that contains the Environment, with the `EXPR` extension `Job.Name` and `Job.EnvNames`, and
       this Environment's `Env.Param.*` and `Env.RawParam.*` of types other than PATH and LIST[PATH]. No other scope is
       available: in particular not `Step.Name`, a Step's `let` bindings, or `Session.*`. They are resolved when the Job
       is created, once for each Step whose Sessions enter the Environment, with that Step's effective values (see
       [Effective Environment parameter values](#322-effective-environment-parameter-values)), so one Environment may
       impose different requirements on different Steps.
    2. Permitted on a Job Template's *jobEnvironments* entries and on an Environment Template's *environment*. Not
       permitted on a *stepEnvironments* entry, whose Step has *hostRequirements* of its own.

  How the requirements combine with the Step's is specified in
  [Combined host requirements](#333-combined-host-requirements-extension-env_params).

The format string scope `Env.*` within an Environment gains:

```diff
  3. `Env.*` — Scope of the environment entity itself. Values such as the embedded files defined within the Environment
-    entity.
+    entity, and with the `ENV_PARAMS` extension, `Env.Param.*` and `Env.RawParam.*`: the values of this Environment's
+    parameters with which the Session entered it, which are the effective values of every Step whose Tasks the Session
+    runs (see [Effective Environment parameter values](#322-effective-environment-parameter-values)). A Step's own
+    script cannot reference `Env.Param.*`, and no Environment can reference another Environment's parameters.
```

#### `<EnvironmentParameterDefinition>`

> A new section, inserted after [`4.4. <EnvironmentVariables>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#44-environmentvariables) as `4.5`. The text below
> is the wiki text verbatim, with links made absolute.

##### 4.5. `<EnvironmentParameterDefinition>` `@extension ENV_PARAMS`

An `<EnvironmentParameterDefinition>` declares one value that a Step may supply to the Environment that contains it.
It has the shape of a [&lt;JobParameterDefinition&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#2-jobparameterdefinition) of the same `type`, with these
differences:

1. *type* — Any type available to a `<JobParameterDefinition>` in the same document, except `RANGE_EXPR`. As for Job
   Parameters, the types other than `STRING`, `INT`, `FLOAT`, and `PATH` require the `EXPR` extension.
2. *userInterface* — Not permitted. Environment parameters are set by templates, not by a person submitting a Job.
3. *objectType* and *dataFlow* (PATH and LIST[PATH]) — Not permitted. Which files a Job reads and writes is declared
   by its Job Parameters.
4. *default* — Optional. A [Format String](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#73-format-strings) resolved when the Job is created. It may reference
   `RawParam.<name>` for any Job Parameter, and `Param.<name>` for any Job Parameter not of type PATH or LIST[PATH],
   defined in the top-level *parameterDefinitions* of the same document, and no other scope. Those Job Parameters have
   their merged values (see [§1.2.1](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#121-merging-environment-template-parameter-definitions)). With the `EXPR`
   extension, a `default` consisting of a single `{{ }}` expression is evaluated with the parameter's type as its
   target type; otherwise the resolved string is converted to the parameter's type as a submitted Job Parameter value
   of that type would be, except that a relative path is not joined with any directory (see below). A list-typed
   parameter has no string form, so a list `default` that is not a literal must be a single `{{ }}` expression. A
   literal `default` is written as for a `<JobParameterDefinition>` of the same type.
5. *allowedValues*, *minLength*, *maxLength*, *minValue*, *maxValue*, and *item* — As for the
   `<JobParameterDefinition>` of the same type. They constrain every effective value, whether supplied by a Step or
   taken from the resolved `default`.

A `default` is checked against the definition at two points:

1. A literal `default`, one that contains no `{{ }}`, is checked at template validation against the parameter's type
   and constraints, as a Job Parameter's literal `default` is. A literal that violates them is an error even when every
   Step supplies the parameter. For PATH and LIST[PATH], the checks that depend on the submission path format,
   including absoluteness, are made when the Job is created, as below; a relative literal `default` is then an error
   even when every Step supplies the parameter.
2. Any other `default` is resolved when the Job is created and is checked only where it becomes an effective value,
   that is, for each Step that enters the Environment without supplying the parameter. A resolved `default` that would
   violate the constraints is not an error when every such Step supplies its own value, since a `default` that reads a
   Job Parameter is a job-wide fallback, and the Job Parameter's value may be meaningful only to Jobs that use it.

A parameter without a `default` is required: every Step whose Sessions enter the Environment must supply it (see
[&lt;StepDependency&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#32-stepdependency) constraint 7). For an Environment with `runScope: ONLY_REQUESTED`
those are exactly the Steps that request it; for one with `runScope: ALL`, every Step of the Job.

A value of a PATH parameter, and each element of a value of a LIST[PATH] parameter, whether supplied by a Step or taken
from the `default`, literal or resolved, is a path in the Job's submission path format: the path format of the Job's
PATH Job Parameter values, which is the `source_path_format` that path mapping rules for the Job are keyed on (see
[How Jobs Are Run](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run#path-mapping)). When the Job is created, each such path must be absolute in that
format or, with the `EXPR` extension, a URI; a relative path is an error, since an Environment parameter has no
directory to be relative to. Its `allowedValues`, `minLength`, `maxLength`, and `item` constraints are checked at the
same time, against the unmapped value; path mapping on the Worker Host does not re-check them. Template authors should
take path values from PATH Job Parameters, through `RawParam.<name>`, which always have the submission path format. A
literal absolute path is absolute in only one operating system's format, so a literal PATH value is portable only
across submitters that use that format. A host-local path that an Environment Template owns and that needs no path
mapping is better declared as a STRING.

The value of an Environment parameter is referenced in format strings, within the Environment that defines it, as both:

1. `Env.Param.<name>` — the value with which the Session entered the Environment; for type PATH and LIST[PATH], with
   path mapping rules applied; and
2. `Env.RawParam.<name>` — the same value with no path mapping applied.

With the `EXPR` extension, `Env.Param.<name>` has the expression type of the corresponding Job Parameter type, and
`Env.RawParam.<name>` has the same type except that it is `string` for PATH and `list[string]` for LIST[PATH].

Environment parameters were introduced in
[RFC 0010](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0010-environment-parameters.md).

#### `<HostRequirements>`

> A new subsection appended to [`3.3. <HostRequirements>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#33-hostrequirements)
> after `3.3.2. <AttributeRequirement>`. Add to the end of §3.3's introduction: "With the
> `ENV_PARAMS` extension, an Environment may also have *hostRequirements*; see
> [Combined host requirements](#333-combined-host-requirements-extension-env_params)." The text below is the wiki text
> verbatim, with links made absolute.

##### 3.3.3. Combined host requirements `@extension ENV_PARAMS`

With the `ENV_PARAMS` extension, an [`<Environment>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#4-environment) in a Job Template's *jobEnvironments*
or an Environment Template may have *hostRequirements*. The **combined host requirements** of a Step are the
requirements of the Step's own *hostRequirements* together with those of every Environment that the Step's Sessions
enter (see [`<StepDependency>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#32-stepdependency)), each Environment's resolved
with the Step's effective values for that Environment. A Session that runs the Step's Tasks may be scheduled only to a
host that satisfies every requirement in the combined set, and every *amounts* requirement in it is reserved for the
Session.

The combined requirements are computed when the Job is created, after the Environments each Step's Sessions enter are
known. They are a conjunction: a host satisfies them when it satisfies each requirement from each source. Amounts do not
add: a Step that requires one GPU and enters an Environment that also requires one GPU reserves one GPU, not two. For
requirements from different sources that name the same capability (names compare after resolution and ignoring case,
as in §3.3.1.1 and §3.3.2.1), the conjunction is equivalent to one requirement as follows, and an implementation may
present or store it in that form:

1. Two [&lt;AmountRequirement&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#331-amountrequirement)s combine to one whose *min* is the greater of
   the two *min*s (a missing *min* is 0) and whose *max* is the lesser of the two *max*es (a missing *max* is
   unbounded). If the result's *min* exceeds its *max*, no host can satisfy it, and the submission MUST be rejected,
   naming the Step, the conflicting sources, and the capability. The sources may be the Step and an Environment, or
   two Environments the Step's Sessions enter.
2. Two [&lt;AttributeRequirement&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#332-attributerequirement)s that both use *anyOf* combine to one whose
   *anyOf* is the intersection of the two lists, comparing values ignoring case as §3.3.2 does. If the intersection is
   empty, the submission MUST be rejected as in item 1.
3. Two that both use *allOf* combine to one whose *allOf* is the union of the two lists, again ignoring case.
4. One that uses *anyOf* and one that uses *allOf* are not combined; both remain in the set, and a host must satisfy
   both.

The constraints of [§3.3](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#33-hostrequirements) on list length and on duplicate names apply to
each source's *hostRequirements* separately, not to the combined set, so an Environment's requirements are checked on
their own, and combining them adds no error other than the conflicts in items 1 and 2.

An Environment's *hostRequirements* apply only to Sessions that enter it. A Step whose Sessions do not enter an
Environment, because its `runScope` is `ONLY_REQUESTED` and the Step does not request it, because a `required: false`
entry resolved to nothing, or because the entry is disabled, gets none of that Environment's requirements. An
Environment with `runScope: ALL` adds its requirements to every Step of the Job.

#### `<StepTemplate>` and scope lists

> Modifications to [`3. <StepTemplate>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#3-steptemplate),
> [`3.6.2. Let Binding Scope Summary`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#362-let-binding-scope-summary), and
> [`7.3.1. Value References`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#731-value-references).

The new value sources for *parameters* values (`Step.Name` and the Step's `let` bindings) and for
`<EnvironmentScript>.let` (`Env.Param.*`) are added to every list that defines those scopes.

```diff
  3. *let* — An ordered list of expression bindings evaluated once per step. Bound names are available in
-    *stepEnvironments*, *hostRequirements*, *parameterSpace*, and *script* fields. See: [&lt;LetBindings&gt;](#36-letbindings).
+    *stepEnvironments*, *hostRequirements*, *parameterSpace*, and *script* fields, and with the `ENV_PARAMS` extension in
+    the *parameters* values and *enableIf* of *dependencies* entries. See: [&lt;LetBindings&gt;](#36-letbindings).
  4. *dependencies* — A list of the dependencies of this Step. These dependencies must be resolved before the Tasks of the
-    Step may be scheduled. See: [&lt;StepDependency&gt;](#32-stepdependency)
+    Step may be scheduled. With the `ENV_PARAMS` extension, an entry may instead request an Environment that this
+    Step's Sessions enter, and supply values for its parameters; such an entry does not delay scheduling.
+    See: [&lt;StepDependency&gt;](#32-stepdependency)
```

In the §3.6.2 table:

```diff
- | `<StepTemplate>.let` | `Param.*`, `RawParam.*`, `Job.Name`, `Step.Name`, earlier bindings in same `let` | *stepEnvironments*, *hostRequirements*, *parameterSpace*, *script* (including nested `<StepScript>.let` or `<SimpleAction>.let`) |
+ | `<StepTemplate>.let` | `Param.*`, `RawParam.*`, `Job.Name`, `Step.Name`, earlier bindings in same `let`, and with `ENV_PARAMS` `Job.EnvNames` | *stepEnvironments*, *hostRequirements*, *parameterSpace*, *script* (including nested `<StepScript>.let` or `<SimpleAction>.let`), and with `ENV_PARAMS` the *parameters* values and *enableIf* of *dependencies* entries |
- | `<EnvironmentScript>.let` | `Param.*`, `RawParam.*`, `Session.*`, `Env.File.*`, `Job.Name`, earlier bindings in same `let` | *actions*, *embeddedFiles* |
+ | `<EnvironmentScript>.let` | `Param.*`, `RawParam.*`, `Session.*`, `Env.File.*`, `Job.Name`, earlier bindings in same `let`, and with `ENV_PARAMS` `Job.EnvNames` and the enclosing Environment's `Env.Param.*` and `Env.RawParam.*` | *actions*, *embeddedFiles* |
```

In the §7.3.1 table:

```diff
- |`Step.Name`|The name of the current Step. ...|Available within the Step Template scope: `stepEnvironments`, `hostRequirements`, `parameterSpace`, and `script`.|
+ |`Step.Name`|The name of the current Step. ...|Available within the Step Template scope: `stepEnvironments`, `hostRequirements`, `parameterSpace`, and `script`, and with the `ENV_PARAMS` extension the *parameters* values and *enableIf* of `dependencies` entries.|
```

The *let* item of `<EnvironmentScript>` (§4.2) does not enumerate the scopes its bindings may read, so it is
unchanged; the §3.6.2 row is the normative list.

#### `<StepDependency>`

> A replacement of [`3.2. <StepDependency>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#32-stepdependency), which on mainline defines only `dependsOn`, and
> two new subsections, §3.2.1 and §3.2.2. The schema change is:

```diff
- This entity is used by the template author to specify a Step in the same Job that a Step depends upon.
+ This entity is used by the template author to specify a Step in the same Job that a Step depends upon or, with the
+ `ENV_PARAMS` extension, an Environment that the Step requests: one its Sessions enter, with parameter values the Step
+ supplies.

- A `<StepDependency>` is the object:
+ A `<StepDependency>` is one of the objects:

  dependsOn: "<StepName>"

+ # @extension ENV_PARAMS
+ environment: "<EnvironmentName>"
+ required: <bool> # @optional
+ enableIf: <bool> | <FormatString> # @optional @fmtstring @extension EXPR
+ parameters: <EnvironmentParameterValues> # @optional
```

> The complete text of the new §3.2, §3.2.1, and §3.2.2 follows, verbatim from the wiki with links made absolute.
> Composition with RFC 0009 (`SERVICE`) is in [Composition with SERVICE](#composition-with-service).

##### 3.2. `<StepDependency>`

This entity is used by the template author to specify a Step in the same Job that a Step depends upon or, with the
`ENV_PARAMS` extension, an Environment that the Step requests: one its Sessions enter, with parameter values the Step
supplies.

A `<StepDependency>` is one of the objects:

```yaml
dependsOn: "<StepName>"
```

```yaml
# @extension ENV_PARAMS
environment: "<EnvironmentName>"
required: <bool> # @optional
enableIf: <bool> | <FormatString> # @optional @fmtstring @extension EXPR
parameters: <EnvironmentParameterValues> # @optional
```

Where:

1. *dependsOn* — Provides the name of another step in the same job upon which this step depends. The Task(s) of this Step
   may be scheduled only when the depended-upon Step has fully completed successfully.
2. *environment* — The name of an Environment that this Step requests: its Sessions enter it, and the Step may supply
   values for its parameters. Available with the `ENV_PARAMS` extension. When *enableIf* is present and resolves to
   `false`, the entry is **disabled** and has no effect of any kind: the name is not resolved, nothing is requested or
   supplied, and the Step is treated as having no entry for the Environment. Otherwise the name resolves, in order:
    1. to the entry of the same Job Template's *jobEnvironments* with that `name`, if there is one;
    2. otherwise, to an **external Environment**: the `environment` of an Environment Template that the scheduler
       attaches to the submission, matched by `name` when the Job is created (see
       [Resolving external Environments](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#321-resolving-external-environments)).

   The entry declares that the Step depends on the Environment: its Sessions need that Environment, entered with these
   values. If the Environment's `runScope` is `ONLY_REQUESTED`, the entry is what causes this Step's Sessions to enter
   it; if it is `ALL`, the Step's Sessions enter it regardless, and the entry only supplies values. The entry does not
   delay scheduling the Step's Tasks, and it does not change the position of the Environment in the order a Session
   enters its Environments.
3. *required* — Whether the Environment must be present. Default `true`. When `false` and the name resolves to no
   Environment when the Job is created, the entry has no effect: the Step's Sessions run without that Environment. A
   *jobEnvironments* entry is always present, so `required: false` is not permitted on an entry that names one
   (constraint 8).
4. *parameters* — The values this Step supplies, keyed by the names in the Environment's *parameterDefinitions* (see
   [&lt;EnvironmentParameterDefinition&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#45-environmentparameterdefinition-extension-env_params)). A parameter the entry omits takes its
   `default`. If not provided, the entry supplies no values; it still requests the Environment.
5. *enableIf* — Whether this entry is in effect. Available with the `EXPR` extension. A boolean literal, or a
   [Format String](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#73-format-strings) consisting of a single `{{ }}` expression whose
   target type is `bool`, with the same available scopes as a *parameters* value (`RawParam.*`, `Param.*` other than
   PATH and LIST[PATH], `Job.Name`, `Job.EnvNames`, `Step.Name`, and the Step's `let` bindings). It is resolved
   when the Job is created, before the entry's name is resolved; a `null` result is an error. If omitted, the entry is
   enabled. A disabled entry is as if absent (item 2): for an `ONLY_REQUESTED` Environment the Step's Sessions do not
   enter it, and for an `ALL` Environment they enter it with every parameter at its `default`, subject to constraint 7.

`<EnvironmentParameterValues>` is a map from [&lt;Identifier&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#71-identifier) to `<EnvironmentParameterValue>`:

```yaml
<Identifier>: <EnvironmentParameterValue>, # @fmtstring
...
```

An `<EnvironmentParameterValue>` is one of:

* A literal of the parameter's type, written as the `default` of a [&lt;JobParameterDefinition&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#2-jobparameterdefinition)
  of the same type is written: a string, an integer or `<intstring>`, a float or `<floatstring>`, and with the `EXPR`
  extension a boolean or a list.
* A [Format String](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#73-format-strings) resolved when the Job is created. Its available scopes are `RawParam.*` for
  every Job Parameter and `Param.*` for every Job Parameter not of type PATH or LIST[PATH], in both cases those defined
  in the top-level *parameterDefinitions* of the document that contains the entry, with their merged values (see
  [§1.2.1](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#121-merging-environment-template-parameter-definitions)); and, with the `EXPR` extension, `Job.Name`,
  `Job.EnvNames`, `Step.Name`, and the names bound by the Step's `let`. `Task.*`, `Session.*`, and `Env.*` are not available: a value
  is fixed for the whole Step. With the `EXPR` extension, a value consisting of a single `{{ }}` expression is
  evaluated with the parameter's type as its target type. Any other format string resolves to a string, which is
  converted to the parameter's type as a submitted Job Parameter value of that type would be, except that a relative
  path is not joined with any directory (see
  [&lt;EnvironmentParameterDefinition&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#45-environmentparameterdefinition-extension-env_params)). A list-typed
  parameter has no string form, so its value must be a list literal or a single `{{ }}` expression.

An entry is part of the document that contains it, and that document's extensions, not the Environment's, decide which
forms a value may take. Without the `EXPR` extension in the requesting document, a value is a string, integer, or float
literal or a format string of value references, and the entry cannot supply a parameter whose type requires `EXPR`
(constraint 6); every Step of that document takes such a parameter's `default`. For example, a Job Template that does
not list `EXPR` can supply a STRING `Image` to a Docker Environment but not a LIST[PATH] `Mounts`.

A value for a parameter of type PATH or LIST[PATH] is a path in the Job's submission path format; see
[&lt;EnvironmentParameterDefinition&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#45-environmentparameterdefinition-extension-env_params).

Constraints:

1. Exactly one of *dependsOn* or *environment* must be present. *required*, *enableIf*, and *parameters* are permitted
   only with *environment*.
2. *environment* must not name a *stepEnvironments* entry of the same Step.
3. No two *environment* entries of one *dependencies* list may name the same Environment.
4. *parameters*, if provided, must contain at least one entry and at most 50 entries.
5. Every key of *parameters* must name a parameter declared in the named Environment's *parameterDefinitions*. An
   Environment that declares none, including one from a document that does not list `ENV_PARAMS`, accepts an entry with
   no *parameters* and rejects one with any.
6. Every value must be valid for the parameter it supplies: of its type, and within its `allowedValues`, `minLength`,
   `maxLength`, `minValue`, `maxValue`, and `item` constraints. An entry in a document that does not list `EXPR` must
   not supply a parameter whose type requires `EXPR` (BOOL or a LIST type).
7. For every Environment that a Step's Sessions enter, the Step must supply every parameter of that Environment that has
   no `default`. A Step's Sessions enter every Environment of the Job, whether a *jobEnvironments* entry or an attached
   Environment, whose `runScope` is `ALL`, so such a parameter of such an Environment must be supplied by every Step of
   the Job; and they enter every Environment with `runScope: ONLY_REQUESTED` that the Step requests, so such a parameter
   of such an Environment must be supplied by every Step that requests it.
8. *required* must not be `false` on an entry that names a *jobEnvironments* entry.
9. *enableIf* is permitted only in a document that lists `EXPR`, and must be a boolean literal or a single `{{ }}`
   expression of type `bool`.

Constraints 1–4, 8, and 9 are checked at template validation. Constraints 5–7 are checked at template validation for
entries that name a *jobEnvironments* entry (for constraint 6, against literal values only), and when the Job is
created for every entry whose *enableIf* is absent or `true`, once format strings are resolved and external
Environments are known. A check that depends on whether an entry is enabled, which includes constraint 7 and any check
of which Steps request an Environment, is made at template validation only for entries whose *enableIf* is absent or a
boolean literal; for an entry whose *enableIf* is an expression, it is made when the Job is created. Constraint 3
applies to entries as written, enabled or not, so two entries for one Environment with complementary *enableIf*
conditions are an error; write one entry whose values are conditional instead.

For an entry that names no *jobEnvironments* entry, template validation can check only constraints 1–4, 8, and 9, and the
type of a literal value where its form fixes one (a list is not a STRING). A validator that is given the Environment
Templates a scheduler will attach, as `openjd run` accepts them, can resolve the entry as
[§3.2.1](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#321-resolving-external-environments) describes and check constraints 5–7 before the Job is created.

###### 3.2.1. Resolving external Environments

When a Job is created from a Job Template and zero or more Environment Templates, each enabled *environment* entry
(one whose *enableIf* is absent or resolves to `true`) that names no *jobEnvironments* entry of the Job Template is
resolved against the attached Environment Templates. A disabled entry is not resolved and takes no part in what
follows:

1. If exactly one attached Environment Template's `environment` has the entry's name, the entry refers to it, and
   constraints 5 and 6 are checked against that Environment's *parameterDefinitions*. This applies whatever *required*
   says: `required: false` tolerates a missing Environment, not one that lacks a parameter the entry supplies. An entry
   that supplies parameters to an attached Environment that declares none, such as one from an Environment Template
   written before `ENV_PARAMS`, causes the submission to be rejected, naming the entry and the parameter.
2. If none does, then with `required: true` the submission must be rejected, naming the entry and the missing
   Environment; with `required: false` the entry is ignored.
3. If two or more do, the submission must be rejected, naming the entry and the ambiguous name.
4. An attached Environment with the same name as a *jobEnvironments* entry is not reachable by name from the Job
   Template. Implementations should warn when they see one.
5. Likewise, an attached Environment with the same name as a *stepEnvironments* entry of a Step is not reachable by name
   from that Step (constraint 2). Other Steps of the Job reach it as usual.

Each entry that resolves applies independently of the others: when the scheduler attaches several of the Environments
a Step requests, the Step's Sessions enter all of them, each with the values its own entry supplies. An attached
Environment with `runScope: ONLY_REQUESTED` is entered by the Sessions of exactly the Steps whose entries resolve to it;
one with `runScope: ALL` is entered by every Session, as without the extension.

After resolution, constraint 7 is checked over the combined Job: every Step whose Sessions enter an attached Environment
with a parameter that has no `default` must supply it, or the submission is rejected. For an attached Environment with
`runScope: ALL`, every Session enters it, so such a parameter rejects every Job Template that does not supply it from
every Step; an Environment Template intended for existing Job Templates therefore gives every parameter a `default`, or
uses `runScope: ONLY_REQUESTED` so that only the Steps that request it are held to the rule.

###### 3.2.2. Effective Environment parameter values

For each Step and each Environment its Sessions enter, the **effective value** of every parameter of that Environment
is determined when the Job is created:

1. If an enabled *environment* entry of the Step names the Environment and supplies the parameter, the resolved value of
   that entry.
2. Otherwise, the resolved `default` of the parameter.

The effective values are part of the Job, and are checked against the parameter's definition when the Job is created
(constraint 6, and see [&lt;EnvironmentParameterDefinition&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#45-environmentparameterdefinition-extension-env_params) for PATH values and
for `default`). They are also the values with which the Environment's *hostRequirements*, if any, are resolved for the
Step (see [Combined host requirements](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#333-combined-host-requirements-extension-env_params)). Path mapping rules are applied to PATH-typed effective values on the Worker Host, as they are to PATH
Job Parameters.

Two effective values are **equal** when they have the same type and the same value as resolved when the Job is created,
before any path mapping. An INT parameter supplied as `1` by one Step and as `"1"` by another resolves to the INT 1 for
both, so the values are equal; two PATH values are equal when their unmapped strings are equal; and two lists are equal
when they have equal elements in the same order.

A Session enters each of its Environments once, with one set of parameter values, and keeps those values until it exits
the Environment. A Session may run Tasks of a Step only when the set of Environments the Session has entered is the set
the Step's Sessions enter (every `ALL` Environment of the Job, every `ONLY_REQUESTED` Environment the Step requests, and
the Step's own *stepEnvironments*) and, for every Environment in that set, the Step's effective values are equal to the
values the Session entered it with. The first condition is the rule *How Jobs Are Run* already states for Sessions that
run Tasks of several Steps, now applied to a set that `runScope` can make differ between Steps; the second is new. A
Session may therefore run Tasks of several Steps, as before, when their Environment sets and effective values agree; the
set of Steps it will serve need not be known when it starts.
See [How Jobs Are Run](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run#environment-parameters).

#### Composition with SERVICE

> Applies only when RFC 0009 (`SERVICE`) is also accepted. Whichever RFC is merged into the wiki second applies these
> edits to the other's text, and renumbers the `<Environment>` items that both RFCs add (SERVICE's *dependencies*; this
> RFC's *parameterDefinitions*, *runScope*, and *hostRequirements*) and the citations below that use SERVICE's numbers.
> The wiki on this branch is based on mainline, which has no `SERVICE`, so it does not yet contain them.

RFC 0009 adds `service:` entries to `<StepDependency>`, `dependencies` to `<Service>` and to Job Environments, and a
second kind of Session, the Service Session, in which a Service's actions run inside the Job's Environments. Earlier
drafts of RFC 0009 also defined a `runScope` list on `<Environment>` that selected Sessions by kind (`TASK`,
`SERVICE`); that field is replaced by this RFC's `runScope`, and RFC 0009 depends on this RFC for it. With both
extensions:

1. **§3.2 schema.** `<StepDependency>` is one of three objects, `dependsOn:`, `service:`, and `environment:`. ENV_PARAMS
   constraint 1 and SERVICE constraint 4 merge into one: "Exactly one of *dependsOn*, *service*, or *environment* must be
   present; an entry that gives more than one, or none, is a validation error. *required*, *enableIf*, and *parameters*
   are permitted only with *environment*."
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
   values (`<Environment>` *runScope* and *hostRequirements*, §3.3.3, constraints 5–7, §3.2.1, §3.2.2), read "Step or
   Service", and wherever it says that an `ALL` Environment is entered by every Session, or applies to every Step of the
   Job, read "except the Sessions of a Service that the Environment lists" (item 8). A Service's combined host requirements are its own *hostRequirements*, which RFC 0009 gives it, together
   with those of the Environments its Sessions enter, by the rules of §3.3.3. A Service's *parameters* values and
   *enableIf* may use `RawParam.*`, `Param.*` (not PATH or LIST[PATH]), `Job.Name`, `Job.EnvNames`, and the
   Service's own `let` bindings; `Step.Name` is not available. A Service Session's Environment parameter values are the
   Service's effective values. SERVICE's §3.6.2 row for `<Service>.let` gains "and with `ENV_PARAMS` the *parameters*
   values and *enableIf* of *dependencies* entries" in its Bound Names Available In column, as the `<StepTemplate>.let`
   row does.
7. **Which Sessions enter which Environments.** A Service Session enters the combined Job's *jobEnvironments*
   (including attached Environments) by the same rule as a Task Session: every one with `runScope: ALL`, and every one
   with `runScope: ONLY_REQUESTED` that the Service requests. Constraint 7 therefore applies to a Service for each
   Environment its Sessions enter, exactly as to a Step. An Environment that only Services should enter is
   `ONLY_REQUESTED` and requested by each Service; an Environment that only Steps should enter is `ONLY_REQUESTED` and
   requested by each Step. An attached Environment from a document that does not declare `ENV_PARAMS` is `ALL`, so
   every Service Session enters it, as every Task Session does. The one exception to "every Service Session" is item
   8: a Service's Sessions do not enter an `ALL` Environment that lists that Service.
8. **Environments that configure Service clients.** Earlier drafts of SERVICE kept an Environment that lists a
   Service in its *dependencies* out of every Service Session, because such an Environment reads the Service's
   endpoint, which does not exist until the Service is READY. That rule is replaced by one that follows from the
   dependency graph, plus one exception:
    * Every entity whose Sessions enter an Environment that lists Service `X` depends on `X`. A Step is in `X`'s scope
      (SERVICE *Service scope* rule 3, which with `ONLY_REQUESTED` reads "every Step that requests it" rather than
      "every Step"), and a Service `Y` gains the edge `Y` → `X` in the graph of SERVICE constraint 3 (or, within an
      Environment Template, of SERVICE §1.2 item 6 constraint 4). Those graphs must still be acyclic. In particular, an
      `environment:` entry by `X` itself for an Environment that lists `X` is the edge `X` → `X`, a cycle, and is
      rejected naming it, whatever the Environment's `runScope`: the entry says that `X`'s Sessions enter an
      Environment that waits for `X` to be READY. An explicit entry is never silently ignored.
    * The exception: a Service's Sessions do not enter an `ALL` Environment that lists that Service. Without it, every
      Environment that configures clients of a Service would put that Service in a cycle by default, and the common
      case, a Job Environment that sets a cache's address for every Step, could not keep the default `runScope`. The
      exception applies only where no entry is written; it is a default, not an override.

   A cycle whose edges all come from one Job Template is rejected at template validation, except where an edge depends
   on an entry whose *enableIf* is an expression, in which case it is checked when the Job is created (§3.2). A cycle
   through an attached Environment is checked when the Job is created. The remedy for a cycle that is not a mistake is
   to make the Environment `ONLY_REQUESTED`, so that the Service's Sessions do not enter it. A Service may otherwise use
   an Environment that configures clients of another Service, and starts after that Service is READY, exactly as a Step
   would.
9. **Wrap hooks.** SERVICE's rule "the required hooks follow from `runScope`" and RFC 0008's rule that a wrapper
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
   wrapper may define all seven hooks and be attached to any queue. Which kinds of Session enter the wrapper is known
   from `runScope` and, for `ONLY_REQUESTED`, from the entities that request it; the check is made at template
   validation when the wrapper and every requesting entity are in one Job Template and every request's *enableIf* is
   absent or a literal, and at job creation otherwise, naming the Environment and the kind of Session. A validator
   should warn at template validation when an `ALL` wrapper in a Job Template lacks `onWrapTaskRun`, since every Job
   has Task Sessions and the error is certain. The remedy is to define the hooks, or to make the wrapper
   `ONLY_REQUESTED` and have only entities of the kinds it supports request it.

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

#### Composition with CONTROL_FLOW

> Applies only when RFC 0011 (`CONTROL_FLOW`) is also accepted. Whichever RFC is merged into the wiki second applies
> these edits to the other's text.

RFC 0011 adds `enableIf` to `<Environment>`: a `bool` expression, evaluated once at Job creation for a Job Environment,
over `Param.*`, `RawParam.*`, and `Job.Name`. If it is `false`, the Environment is disabled for every Session of the
Job. With both extensions:

1. **`enableIf` may read the Environment's own parameters.** In an Environment that declares *parameterDefinitions*,
   *enableIf* may additionally reference `Env.Param.*` and `Env.RawParam.*` of that Environment (CONTROL_FLOW
   `<Environment>` *enableIf* constraint 3 is relaxed for these names). Such an *enableIf* is evaluated at Job
   creation once per Step (or Service) whose Sessions would otherwise enter the Environment, with that entity's
   effective values (§3.2.2). For each entity for which it is `false`, that entity's Sessions do not enter the Environment, exactly as
   for an `ONLY_REQUESTED` Environment the entity does not request. PATH-typed values are compared unmapped, through
   `Env.RawParam.*`; `Env.Param.*` of a PATH or LIST[PATH] parameter is not available in *enableIf*, since path mapping
   is applied on the Worker Host and *enableIf* is decided before any host is chosen.
2. **Permitted in Environment Templates when it reads only `Env.*`.** CONTROL_FLOW constraint 4 forbids *enableIf* in
   an Environment Template because the template cannot know the Job's parameters. An *enableIf* whose only references
   are `Env.Param.*` and `Env.RawParam.*` of the same Environment reads values the Environment itself declares and
   defaults, so it is permitted there.
3. **Interaction with `runScope`.** The two are independent filters. A Session enters an Environment when its
   `runScope` admits the Session's entity and *enableIf* is `true` for that entity (or absent). An *enableIf* that
   reads only `Param.*` keeps CONTROL_FLOW's job-wide meaning: `false` disables the Environment for every Session, whatever
   its `runScope`. `Job.EnvNames` lists every Job Environment whatever its *enableIf*, so an *enableIf* that reads it,
   directly or through an effective value, sees a list that no *enableIf* changes.
4. **Session sharing.** The set of Environments a Step's Sessions enter, in §3.2.2, excludes Environments disabled for
   that Step by *enableIf*. Two Steps whose Environment sets differ for this reason do not share Sessions, as for any
   other difference in the set.
5. **The two `enableIf`s.** CONTROL_FLOW's *enableIf* on an `<Environment>` and this RFC's *enableIf* on an
   `environment:` entry are the same idea from the two sides: the Environment's author says when it applies, the Step's
   author says when it is wanted. A Step's Sessions enter an Environment only when both agree (and `runScope` admits
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
   Environment is present may well have host requirements that contradict the queue's Environments when it is not.
   For a Step whose *enableIf* is dynamic, those checks are made at step input binding once it is decided `true`, and
   a failure fails the Step, naming the cause, as a binding error does. Template-validation checks are unaffected: they
   have no parameter values to condition on, and apply to every Step as written, exactly as CONTROL_FLOW type-checks
   every expression of a Step it later skips.

A per-entity *enableIf* expresses selection that `runScope` cannot, because it depends on the values rather than on
whether an entry exists. A GPU-setup Environment can be entered only by Steps whose `Gpus` value is non-empty, or a
provisioning Environment only by Steps that end up with something to install:

```yaml
environment:
  name: Conda
  runScope: ALL
  parameterDefinitions:
    - name: Packages
      type: STRING
      default: "{{Param.CondaPackages}}"
  enableIf: "{{Env.Param.Packages != ''}}"   # entered only by Steps that end up with packages to install
```

Here a Step that supplies no `Packages` and whose Job sets no `CondaPackages` skips the Environment entirely, while a
Job Template written before `ENV_PARAMS` that sets `CondaPackages` still enters it in every Session. `runScope:
ONLY_REQUESTED` is the form of this that needs neither `EXPR` nor `CONTROL_FLOW`.

#### Value references

> A modification to [`7.3.1. Value References`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#731-value-references)

```diff
  |`Env.File.<name>`|...|Available within the Environment Script Actions and Embedded Files.|
+ |`Env.Param.<ParamName>`|The value of the Environment parameter `<ParamName>` with which the Session entered the Environment: the effective value for every Step whose Tasks the Session runs, which are equal (see [§3.2.2](#322-effective-environment-parameter-values)). The same as `Env.RawParam.<ParamName>` except for types PATH and LIST[PATH], where path mapping rules are applied. Requires the `ENV_PARAMS` extension.|Available within the Environment that declares the parameter: its `variables`, its `hostRequirements` (types other than PATH and LIST[PATH]), and its Script's `let`, Actions, and Embedded Files. Not available in any other Environment or in a Step.|
+ |`Env.RawParam.<ParamName>`|The value of the Environment parameter `<ParamName>` with which the Session entered the Environment, with no path mapping applied. Requires the `ENV_PARAMS` extension.|As for `Env.Param.<ParamName>`.|
+ |`Job.EnvNames`|The `name` of each Job Environment of the Job: the Job Template's `jobEnvironments` and the Environments attached from Environment Templates, whatever their `runScope`, in the order a Session that entered all of them would enter them. Step Environments are not included. A name appears once per Environment, so it appears twice if two attached Environments share it. A list, so usable only within a `{{ }}` expression. With `CONTROL_FLOW`, an Environment that `enableIf` disables is still listed: the list is read by expressions that feed effective values, which an Environment's `enableIf` may read, so excluding disabled Environments would be circular. Requires the `ENV_PARAMS` and `EXPR` extensions.|Available in every format string of a Job Template or Environment Template that is resolved at Job creation or later, including an Environment's `hostRequirements` and the `let`, actions, and embedded files of its script, and concrete at Job creation once the attached Environments are known. Not available in an Environment parameter `default`, which reads only `Param.*` and `RawParam.*`, nor in the Job Template's `name`.|
```

`Job.EnvNames` lets a Step decide what to ask of the Environments a queue happens to attach, for example to
prefer a container Environment over a Conda one when both are present
([Choosing between Environments a queue provides](#choosing-between-environments-a-queue-provides)). It names what the
Job has, not what a particular Step's Sessions enter, so it is the same for every Step and does not by itself affect
which Steps may share a Session.

A reference to `Env.Param.<name>` or `Env.RawParam.<name>` that does not name a parameter of the enclosing Environment is
a template validation error. A wrapping Environment's wrap hooks (RFC 0008) read the wrapping Environment's own
parameters through `Env.Param.*`; an inner Environment's parameters are not visible to them.

#### Template processing stages

> A modification to [`7.4. Template Processing Stages`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#74-template-processing-stages)

* **Template validation** — `Env.Param.*` and `Job.EnvNames` are added to the Unknown Values. With `EXPR`,
  `Env.Param.*` is type-checked using the declared types of the Environment's *parameterDefinitions*, and
  `Job.EnvNames` is `unresolved[list[string]]`. Literal Environment parameter `default`s, and literal
  *parameters* values for *jobEnvironments* entries, are checked against their definitions.
* **Job creation** — Each Step's Environment parameter effective values are added to the Known Values at the end of the
  stage. The Environment Templates attached to the Job are determined first, and `Job.EnvNames` is concrete
  before any TEMPLATE-scope expression of the Job, including a Step's `let` bindings, is evaluated. Then each
  `environment:` entry's *enableIf* is evaluated; each enabled entry that names no *jobEnvironments* entry is resolved
  against the attached Environments (§3.2.1); the set of Environments each Step's Sessions enter is fixed from each
  Environment's `runScope` and the Step's enabled entries; Environment parameter `default`s and *parameters* values are
  resolved into each Step's effective values and checked; and each Environment's *hostRequirements* are resolved with
  each entering Step's effective values and combined with the Step's own (§3.3.3). Environment parameter PATH and
  LIST[PATH] values are not joined with any directory and must be absolute (see
  [§4.5](#45-environmentparameterdefinition-extension-env_params)).
* **Task execution** — `Env.Param.*` is available in the Environment's host-evaluated format strings, with path mapping
  applied to PATH and LIST[PATH] values.

### Modifications to the Expression Language

> A modification to [`1.2.2. Built-in Symbol Types`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2026-02-Expression-Language#122-built-in-symbol-types)
> of the Expression Language, after *Embedded File Symbols*.

```diff
+ ##### Environment Parameter Symbols
+
+ With the `ENV_PARAMS` extension, an Environment that declares *parameterDefinitions* (see
+ [Template Schemas](2023-09-Template-Schemas#45-environmentparameterdefinition-extension-env_params)) can read their
+ values within that Environment only:
+
+ | Symbol | Type | Description |
+ |--------|------|-------------|
+ | `Env.Param.<name>` | The `Param.<name>` type of the parameter's type in the Job Parameter table above | The value with which the Session entered the Environment, with path mapping applied to PATH and LIST[PATH] |
+ | `Env.RawParam.<name>` | The `RawParam.<name>` type of the parameter's type in the Job Parameter table above | The same value with no path mapping applied |
+
+ With the `ENV_PARAMS` extension, every template can also read:
+
+ | Symbol | Type | Description |
+ |--------|------|-------------|
+ | `Job.EnvNames` | `list[string]` | The names of the Job's Job Environments, including those attached from Environment Templates, in entry order. `unresolved[list[string]]` at template validation; concrete at Job creation. |
```

### Modifications to How Jobs Are Run

> Modifications to [How Jobs Are Run](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run)

```diff
  Once the Environments have been created and entered for a Session, a series of Tasks are
  run within that Session and the Environments. Tasks from any Step in a Job can run within
  the same Session provided that the set of Environments that are required to run those
- Tasks are identical.
+ Tasks are identical (with the ENV_PARAMS extension, that set depends on which Environments
+ each Step requests) and those Steps' effective values for every Environment's parameters
+ are equal to the values the Session entered it with (see Environment Parameters).
```

And a new section, *Environment Parameters*, after *Running a Session*:

> With the ENV_PARAMS extension, introduced in
> [RFC 0010](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0010-environment-parameters.md),
> an **Environment** may declare parameters, and a Step requests the **Environment** and supplies values for them in an
> `environment:` entry of its `dependencies`. The **Environment** reads the values as `{{Env.Param.<name>}}`. An
> **Environment** may also declare a `runScope`: `ALL`, the default, means every **Session** of the Job enters it, as
> every Job Environment does without the extension; `ONLY_REQUESTED` means only the **Sessions** of the Steps that
> request it do, and every other **Session** runs as if it were not defined.
>
> The values are resolved once, when the Job is created. For each Step and each **Environment** that the Step's
> **Sessions** enter, each parameter takes the value the Step supplies, or else the parameter's default, which may refer
> to a Job Parameter. An `environment:` entry can name one of the Job's own Job Environments, or an **Environment** that
> the render management system attaches from an Environment Template; the latter is matched by name when the Job is
> created. If the entry says `required: false` and nothing of that name is attached, the entry has no effect and the
> Step's **Sessions** run without that **Environment**. If it says `required: true` (the default), the submission is
> rejected. If something of that name is attached but does not declare a parameter the entry supplies, for example an
> **Environment** written before the extension, the submission is rejected whatever `required` says. The entries are
> independent of one another: if the render management system attaches several of the **Environments** a Step
> requests, the Step's **Sessions** enter each of them with the values its own entry supplies. For an `ALL`
> **Environment**, an entry only supplies values; for an `ONLY_REQUESTED` **Environment**, the entry is also what makes
> the Step's **Sessions** enter it. An entry may carry an `enableIf` condition, decided when the Job is created; a
> disabled entry is as if it were not written. The **Environments** a **Session** does enter keep the order they have in
> the Job.
>
> An **Environment** may also declare `hostRequirements`, which may read its parameters. A **Session** that enters it
> must be placed on a Worker Host that satisfies them as well as the Step's own, and its `amounts` are reserved for the
> **Session** like the Step's. Because they are resolved with each Step's values, one **Environment** can ask for a GPU
> only for the Steps that asked it to use one.
>
> A **Session** enters each **Environment** once, with one set of parameter values, and keeps those values until it
> exits the **Environment**: the **Environment**'s `onEnter`, every Task the **Session** runs, and the **Environment**'s
> `onExit` all see the same values. A **Session** may run Tasks of a Step only when the **Environments** it has entered
> are exactly the ones the Step's **Sessions** enter, and the Step's effective values for each of them are equal to the
> values the **Session** entered it with. Values are equal when they have the same type and the same resolved value
> before path mapping. A **Session** can therefore serve several Steps, as it can without the extension, whenever their
> **Environment** sets and effective values agree, and a render management system may add a Step's Tasks to a
> **Session** that is already running when they do. Path mapping rules are applied to PATH-
> and LIST[PATH]-typed values on the Worker Host, as for PATH Job Parameters. A Step's own script cannot read
> `Env.Param.*`.
>
> Because the values are fixed per Step and known when the Job is created, a render management system can group
> **Sessions** by them. Two Steps that request the same `ONLY_REQUESTED` **Environments** and whose effective values
> are equal for every **Environment** may share **Sessions**; two Steps whose **Environment** sets or values differ
> never do. A render management system may also use the values to
> prefer a Worker Host that recently entered an **Environment** with the same values, for example one that has a
> matching Conda environment or container image cached. For a pipeline that decodes video with ffmpeg, solves
> structure-from-motion with COLMAP, and renders with Maya, each Step's **Sessions** then install or pull only the
> software that Step needs.

### Recommended names (non-normative)

The portability of a Job Template that writes `environment: Conda` depends on queues using the same Environment and
parameter names. The specification does not mandate names. This RFC recommends the following, which the samples use,
for queue Environment Templates that provision software. Each parameter defaults to the template-level Job Parameter
that Deadline Cloud's sample queue environments already use, so existing Job Templates keep working. Such an
Environment keeps `runScope: ALL` while existing Job Templates rely on the Job Parameter; a queue that offers several of
these Environments to Job Templates that request them can mark them `ONLY_REQUESTED`.

| Environment | Parameter | Type | Default | Meaning |
|-------------|-----------|------|---------|---------|
| `Conda` | `Packages` | STRING | `{{Param.CondaPackages}}` | Space-separated package specs. Empty means use the host's software. |
| `Conda` | `Channels` | STRING | `{{Param.CondaChannels}}` | Space-separated channels. |
| `Rez` | `Packages` | STRING | `{{Param.RezPackages}}` | Space-separated Rez package requests. |
| `Docker` | `Image` | STRING | `{{Param.DockerImage}}` | The container image. |
| `Docker` | `Mounts` | LIST[PATH] | `[]` | Host paths to make visible at the same path in the container. |
| `Docker` | `Gpus` | STRING | `""` | The value for `docker run --gpus`. |

When this RFC is accepted, the table can become a wiki conventions page that queues and Job Template authors refer to.

An Environment Template that adopts a recommended Environment name should declare that name's recommended parameters,
because Job Templates that follow the table supply them, and an Environment that lacks a supplied parameter rejects the
submission (see [Backward compatibility](#backward-compatibility)). This matters most for `Conda`, the name Deadline
Cloud's existing Conda queue environments already use without parameters, and for `Rez`, the name of its sample Rez
queue environment (`rez_queue_env.yaml`), which also declares no parameters.

### Validation

Every rule this extension adds, by the stage that checks it. Constraint numbers refer to
[`<StepDependency>`](#32-stepdependency).

| Rule | Checked at |
|------|-----------|
| `ENV_PARAMS` listed by any document that uses Environment *parameterDefinitions*, `runScope`, or `hostRequirements`, or `environment:` entries | Template validation |
| Environment *parameterDefinitions*: 1–50 entries, unique names, not on *stepEnvironments*, permitted types and fields | Template validation |
| `runScope` is `ALL` or `ONLY_REQUESTED`, and not on *stepEnvironments* | Template validation |
| Environment *hostRequirements*: §3.3 shape and constraints, not on *stepEnvironments*, `Env.*` references only to the enclosing Environment's non-PATH parameters | Template validation |
| An Environment's *hostRequirements*, resolved for a Step, satisfy §3.3.1 and §3.3.2 (a `min` that is a non-negative number, no two names in one source equal after resolution, and so on) | Job creation, per Step |
| Combined host requirements have no empty intersection: `min` ≤ `max` for each amount, non-empty `anyOf` for each attribute (§3.3.3 items 1, 2) | Job creation; with `CONTROL_FLOW`, not for a `SKIPPED` Step, and at step input binding when the Step's *enableIf* is dynamic or its own requirements read `Steps.*` |
| With `CONTROL_FLOW`: an entry's *enableIf* and *parameters* values, and an Environment's *hostRequirements*, do not reference `Steps.*` directly or through a `let` binding ([Composition with CONTROL_FLOW](#composition-with-control_flow) items 5, 6) | Template validation |
| `default` references only `Param.*` / `RawParam.*` of the same document's Job Parameters | Template validation |
| A literal `default` satisfies its parameter's type and constraints, whether or not any Step uses it | Template validation |
| `Env.Param.*` / `Env.RawParam.*` name a parameter of the enclosing Environment, and appear only inside it | Template validation |
| A *parameters* value or *enableIf* references only `RawParam.*`, non-PATH `Param.*`, and with `EXPR` `Job.Name`, `Job.EnvNames`, `Step.Name`, and the Step's `let`; never `Task.*`, `Session.*`, or `Env.*` (§3.2) | Template validation |
| An Environment's *hostRequirements* reference only its document's `RawParam.*` and non-PATH `Param.*`, with `EXPR` `Job.Name` and `Job.EnvNames`, and its own non-PATH `Env.Param.*` / `Env.RawParam.*` (§4 *hostRequirements* item 1) | Template validation |
| `Job.EnvNames` appears only in a document that lists `EXPR`, inside a `{{ }}` expression, and not in an Environment parameter `default` | Template validation |
| `<StepDependency>` has exactly one of *dependsOn* / *environment*; *required*, *enableIf*, and *parameters* only with *environment* (constraint 1) | Template validation |
| *enableIf* only in a document that lists `EXPR`; a boolean literal or a single `bool` expression (constraint 9) | Template validation |
| *enableIf* does not resolve to `null` | Job creation |
| *environment* does not name the Step's own *stepEnvironments* entry; no duplicates per list; 1–50 *parameters* (constraints 2–4) | Template validation |
| `required: false` not used on an entry naming a *jobEnvironments* entry (constraint 8) | Template validation |
| Keys and literal values of *parameters* for a *jobEnvironments* entry (constraints 5, 6) | Template validation |
| Every parameter without a `default` of an `ALL` *jobEnvironments* entry supplied by every Step, and of an `ONLY_REQUESTED` one by every Step that requests it (constraint 7) | Template validation, for Steps whose entries for it have no *enableIf* or a literal one; job creation for Steps with an *enableIf* expression |
| External name resolution: missing with `required: true`, or ambiguous (§3.2.1) | Job creation |
| Keys and values for an external Environment; resolved format-string values (constraints 5, 6). A key the resolved Environment does not declare is an error even with `required: false` (§3.2.1 item 1) | Job creation |
| A document that does not list `EXPR` supplies no parameter of type BOOL or a LIST type (constraint 6) | Job creation (entries naming external Environments; for *jobEnvironments* entries the rule holds by construction) |
| A format-string value or `default` for a list-typed parameter is a single `{{ }}` expression (§3.2, §4.5) | Template validation for `default`s and *jobEnvironments* entries; job creation for external Environments |
| A resolved, non-literal `default` satisfies its parameter's type and constraints, for each Step that takes it (§4.5) | Job creation |
| PATH and LIST[PATH] values absolute in the submission path format, or URIs; their string constraints, against the unmapped value. This includes a literal `default` that no Step uses (§4.5) | Job creation |
| Required parameters of attached `ALL` Environments supplied by every Step of the combined Job, and of attached `ONLY_REQUESTED` Environments by every Step whose entry resolves to one (constraint 7, §3.2.1) | Job creation; with `CONTROL_FLOW`, not for a `SKIPPED` Step, and at step input binding when the Step's *enableIf* is dynamic |
| With `SERVICE`: the SERVICE dependency graph, with an edge from each Service to every Service listed by an Environment its Sessions enter (including a self-edge from a Service's own entry for such an Environment), is acyclic ([Composition with SERVICE](#composition-with-service) item 8) | Template validation for *jobEnvironments* and entries without an *enableIf* expression; job creation otherwise |
| With `SERVICE` and `WRAP_ACTIONS`: every action that runs inside a wrapping Environment has a wrap hook, checked per kind of Session that enters it ([Composition with SERVICE](#composition-with-service) item 9) | Template validation when the wrapper and its requesters are in one Job Template with literal *enableIf*s; job creation otherwise |
| With `CONTROL_FLOW`: an Environment's `enableIf` references `Env.*` only of the enclosing Environment, and `Env.Param.*` only of non-PATH types; in an Environment Template it references `Env.*` only ([Composition with CONTROL_FLOW](#composition-with-control_flow)) | Template validation |

A Session runs Tasks of a Step only when it has entered the Environments the Step's Sessions enter and the Step's
effective values equal the values the Session entered each of them with (§3.2.2). That is a scheduling rule rather than
a validation rule, and has no row here.

## Design Choice Rationale

### Parameters declared on the Environment

The Environment is the only entity that knows what it can be told. Declaring the parameters there
gives each one a type, constraints, and a default in the place they are used, and lets a template
validator check every `Env.Param.*` reference and every supplied value against a definition, exactly
as it checks `Param.*` against Job Parameters. Reusing the `<JobParameterDefinition>` shapes means
there is nothing new to learn: an Environment parameter is a Job Parameter with a narrower audience.
The fields that do not apply (`userInterface`, `objectType`, `dataFlow`) are the ones that describe a
submitter or a Job's data flow, neither of which an Environment parameter has.

### A `default` that refers to a Job Parameter

Queue Environments already take their configuration from Job Parameters, and existing Job Templates
set those parameters. Letting `default: "{{Param.CondaPackages}}"` read the Job Parameter keeps every
such Job Template working with no change: a Step that says nothing gets the job-wide value, which is
today's behavior. It also gives the submitter a way to override the job-wide value at submission,
through the Job Parameter, while Steps that know better override it per Step. Allowing only
`Param.*` and `RawParam.*` of the same document keeps a default resolvable at job creation and
checkable without knowledge of other documents; §1.2.1 merging then decides the Job Parameter's
value, so the Job Template's own `default` for `CondaPackages` flows into the queue Environment as it
does today.

### The `environment:` form of `<StepDependency>`

A Step that needs an Environment configured a certain way depends on it, and `dependencies` is where a
Step says what it depends on. RFC 0009 adds `service:` there for the same reason, and the two forms
read alike: one list, one key per kind of thing depended on. Putting the values beside the name
(`environment: Conda`, `parameters: {...}`) groups everything a Step says about one Environment in one
entry, and the Environment's name is the key that makes the values unambiguous when a Step meets
several Environments that use the same parameter name (see
[Rejected Ideas](#a-flat-per-step-map-of-values)).

An `environment:` entry does not delay scheduling the way `dependsOn:` and `service:` do. It is still
a dependency: with `required: true` the Step cannot run without the Environment, and the scheduler
rejects the submission rather than run it on a queue that lacks it. For an `ONLY_REQUESTED`
Environment it is also what brings the Environment into the Step's Sessions, which is the same
relationship a `service:` entry has to a Service: naming it is what makes it run for you.

### `required` defaults to `true`

Listing something in `dependencies` declares a dependency on it, and the safe reading of an
undeclared choice is that the Step needs what it named. A Job Template that targets several kinds of
queue writes `required: false` on each alternative, which states in the template that the Job runs
without them. The opposite default would let a typo in an Environment name silently do nothing.

### `runScope` lives in this RFC

Earlier drafts of RFC 0009 defined `runScope` as a list of Session kinds, `TASK` and `SERVICE`, to
keep an Environment that configures a Service's clients out of the Service's own Session. Looking at
how it would be used, selection by kind was mostly a stand-in for selection by who asked: an
Environment "for Services only" is one that each Service wants and no Step does, and an Environment
"for Tasks only" is one the Steps want. The `environment:` entry gives every Step and Service a way
to say that directly, and it is defined here. So this RFC owns the field, with two values that need
no Session-kind vocabulary, and RFC 0009 keeps the one rule that is really about kinds (an
Environment that reads a Service's endpoint cannot run in that Service's Session) as a rule rather
than a default for a list.

The move also fixes an awkwardness in the first draft of this RFC. A required parameter on a Job
Environment had to be supplied by every Step, because every Session entered every Job Environment,
so a Step with no use for the Environment still had to configure it. With `ONLY_REQUESTED`, the
Steps that must supply a required parameter are exactly the Steps that asked for the Environment.

### Two values rather than a kind list

`ALL` and `ONLY_REQUESTED` are the two cases that need no further information: everyone, or those
who asked. A list of Session kinds would reintroduce a vocabulary that grows with every RFC that adds
a kind of Session, and its interesting combinations (`[TASK]` for Steps only, `[SERVICE]` for
Services only) are each one `ONLY_REQUESTED` Environment that the right entities request. Selection
on anything finer, such as which Steps have GPUs, is a predicate over the Step's values, which is
what `enableIf` with `Env.Param.*` expresses under `CONTROL_FLOW`
([Composition with CONTROL_FLOW](#composition-with-control_flow)). `ONLY_REQUESTED` is the part of
that which needs neither `EXPR` nor `CONTROL_FLOW`, and the part a queue administrator reaches for
first.

### `ALL` is the default

Every Job Environment and attached Environment is entered by every Session today, so a template that
says nothing must keep that behavior. It is also what an auditing, logging, or credential
Environment wants: a Step cannot opt out of it by not asking.

### An Environment without the supplied parameter is an error

`required: false` makes an absent Environment acceptable. It does not make an entry that supplies
a parameter the resolved Environment lacks acceptable, and this RFC rejects the submission in that
case. The case arises in practice during migration: a queue's `Conda` Environment written before
this extension has the recommended name but no `Packages` parameter.

The alternative is to ignore such an entry, perhaps with a warning. On that queue the Job would
then run every Step's Sessions with the job-wide `CondaPackages`, which a Job Template written for
per-Step values usually leaves empty, and each Step would fail inside its Tasks for want of its
software, after Worker Hosts had been provisioned and Sessions started. Rejecting the submission
fails before anything runs, with an error naming the entry and the parameter, and the fix belongs
to the queue administrator, who adds the parameters with defaults that keep existing Job Templates
working. An Environment of the same name that does not understand the request is also not the
Environment the Job Template asked for; treating it as present but ignoring what the Step asked of
it would make the name mean two things.

### Names resolve to the Job Template first

An `environment:` entry is checkable at template validation whenever it names one of the template's
own `jobEnvironments`, which is the case a template author controls completely. Resolving the
template's own names first means an attached Environment can never change what such an entry refers
to.

### External Environments are named implicitly

An entry that names no Job Environment refers to an external one, with no separate declaration. This
differs from RFC 0009, which requires a Job Template to declare each external Service it uses in
`externalServices`, so that "every `Service.*` reference in every template can be validated without
knowledge of the scheduler's configuration". Under this RFC, the keys and values of an entry that
names an external Environment cannot be checked until job creation.

The difference is acceptable here for three reasons:

1. RFC 0009 needs the declaration because a Job Template *reads* `Service.*` values, and their names
   and types must be known to type-check the reading expressions. A Job Template never reads
   `Env.Param.*`; it only writes values, so no expression in it depends on the external Environment's
   shape.
2. The motivating use is `required: false` entries for Environments that a given queue may not attach
   at all. A declaration would be a second place to write the same name, add no information, and
   still could not be checked without the queue.
3. Every mistake an implicit name allows in a `required: true` entry is caught at job creation, with
   an error that names the entry: a misspelled name, an unknown key, or a value of the wrong type. A
   `required: false` entry is different: a misspelled name resolves to nothing on every queue and the
   entry is silently ignored, which is the cost of letting the same Job Template run where the
   Environment is absent. A validator given the queue's Environment Templates should warn when a
   `required: false` entry resolves to no Environment, so that the mistake is visible before
   submission.

What a template validator does with an external entry is specified in
[`<StepDependency>`](#32-stepdependency): it checks constraints 1–4, 8, and 9 and the form of literal values.
A validator given the Environment Templates the scheduler will attach, as `openjd run` already accepts
them, performs the job-creation checks before submission, which is what the Tooling parseable tenet
asks for.

### Values fixed per Step

An Environment is entered once per Session and serves every Task of that Session. Values that varied
per Task would force one Session per Task, which defeats the Environment's purpose. Values fixed per
Step are known at job creation, so tooling can display them, a scheduler can group Sessions by them,
and two Steps with equal values can still share Sessions as today.

### Sessions that serve several Steps

How Jobs Are Run already lets a Session run Tasks of several Steps whose Environments are identical.
This RFC keeps that rule, which `runScope` gives new work to do, since two Steps that request
different `ONLY_REQUESTED` Environments have different Environment sets, and adds one condition: the
Steps' effective values must be equal to the values the Session entered each Environment with. The rule is stated as a condition on adding a Step's Tasks to a
Session, not as a property of a set of Steps fixed when the Session starts, for two reasons. A scheduler
that today adds Tasks of a newly ready Step to a running Session can keep doing so, with one more
comparison. And the condition is all a Session needs: its Environments' values were fixed at `onEnter`,
and every Task it runs belongs to a Step whose values match them, so `Env.Param.*` has one meaning for
the whole Session.

Equality compares typed values as resolved at job creation, before path mapping. Comparing after
mapping would let two hosts with different mapping rules disagree about whether two Steps may share a
Session, and comparing untyped strings would make an INT `1` differ from `"1"` depending on how the
template wrote it.

### `Step.Name` in values

`Step.Name` and the Step's `let` bindings are available to *parameters* values because they are
available everywhere else in the Step's scope that is resolved at job creation, and a Step `let`
binding is how a Step uses one value both in an `environment:` entry and in its own script. A value
that uses `Step.Name` differs for every Step, so Steps that use it never share Sessions. That suits
values that are meant to be per Step, such as a cache key or a label for an Environment's log output;
it is not the common case.

### `Job.EnvNames`

A Job Template that writes `required: false` entries for both `Conda` and `Docker` says it can use
either, but not which it prefers when a queue attaches both. The template needs to see what the Job
has. A list of names is the smallest thing that answers that, and EXPR already has `in` to test it and
list comprehensions to filter it, so no new function is needed. The name follows RFC 0009's
`WrappedService.PortNames`, a `list[string]` of declared names in declaration order; uses `Env` as the
existing `Env.File.*` and `Env.Param.*` scopes do; and is in the `Job.*` scope because the list is a
property of the Job, not of any Step or Session. It is concrete at job
creation, when the attached Environments are known, which is the same stage as `Job.Name`, so it adds
no new stage and no new state. A set of per-Environment booleans (`Job.HasEnvironment.Conda`) would
need a name per Environment and still could not be enumerated; a list can do both.

The list names every Job Environment whatever its `runScope`, including `ONLY_REQUESTED` ones the Step
does not request, because its purpose is to show what is available to request. It excludes Step
Environments, which no Step can request and which the Step that owns them already knows.

### `enableIf` on the entry

`Job.EnvNames` tells a Step what the queue attached; `enableIf` on the entry is how the Step
acts on it. Without it, a Step that prefers Docker when both are attached could only request `Conda`
with an empty value, which enters an Environment to do nothing and, for an `ONLY_REQUESTED`
Environment, defeats the point of opting in. Disabling the entry is the same as not writing it, which
is the one meaning a reader can predict without consulting the Environment: nothing is requested,
nothing is supplied, and an `ALL` Environment behaves as it does for any Step without an entry.

The field takes the name RFC 0011 uses for the same idea on Steps and Environments, and the same
shape: a `bool` literal or a single `bool` expression, `null` an error. It is evaluated at job
creation with the scopes a *parameters* value has, so a disabled entry costs nothing at run time and
the Environment set of every Step is known when the Job is created, as `runScope` already requires.
It is evaluated before the entry's name is resolved, so a `required: true` entry for an Environment
the queue lacks is not an error when its condition is `false`; that lets one entry say "I need `Conda`
unless `Docker` is here".

`required: false` and `enableIf` answer different questions. `required` is about what the queue
attaches, which the template cannot know; `enableIf` is about what the template wants, which it
states. A template that only wants to tolerate a missing Environment writes `required: false`; one
that wants to choose among present ones writes `enableIf`.

### `hostRequirements` on the Environment

An Environment that provisions software knows what it needs from a host: a container runtime, a
Linux kernel, a GPU when asked for one. Today only a Step can state host requirements, so a queue
Environment Template that works on part of a fleet has no way to say so, and the Steps it serves fail
at `onEnter` on the rest. Putting `hostRequirements` on the Environment puts the statement with the
author who knows it.

The parameters are what make this belong here. A static requirement could be its own small RFC, but
the requirement a provisioning Environment most wants to state depends on what the Step asked for:
`amount.worker.gpu` matters only when `Gpus` is set. This RFC already resolves effective values per
Step at job creation, which is the stage at which a Step's own `hostRequirements` are resolved, so
resolving an Environment's requirements with those values costs no new stage and gives a per-Step
answer for free.

Conjunction is the only combination that respects both authors. The Step's author and the
Environment's author each state what they need; a host must satisfy both, and neither can loosen the
other's. The normalization rules (greatest `min`, least `max`, intersected `anyOf`, united `allOf`)
restate the conjunction as one requirement per capability where that is possible, so that tooling can
show it and a conflict can be detected before anything runs. A conflict (an empty intersection) is
rejected at job creation rather than left to never schedule, because it is static, and the error can
name the two authors who disagree. The mixed `anyOf`/`allOf` case is left as two requirements because
the specification makes the two keys exclusive in one requirement, and nothing is lost: a host either
satisfies both or it does not.

The requirements follow entering. An `ONLY_REQUESTED` Environment constrains only the Steps that
request it, so a queue with some container-capable hosts pulls exactly the Steps that asked for a
container onto them, and a `required: false` entry that resolves to nothing adds nothing. Session
sharing is unaffected: a host either satisfies the combined requirements of the Steps a Session
serves or the Session is never placed there, as today.

### Values resolved at job creation

`parameters` values and defaults see only the Job Parameters (and, with `EXPR`, the Step's `let`
bindings and names, and `Job.EnvNames`), all known at job creation. That makes the effective values a static property of
the Job, which is what lets a scheduler compare them to group Sessions and cache environments. Values
that depended on `Session.*` or host state would be different on every host, and the grouping would
be meaningless. Path mapping is the one host-side step, and it applies to `PATH` values as it does to
`PATH` Job Parameters.

The same reason excludes `Param.<name>` of a PATH or LIST[PATH] Job Parameter from values and
defaults, although §7.3.1 allows it elsewhere in an Environment. `Param.<name>` of a path is the
host-mapped value, which does not exist until a Worker Host applies its path mapping rules. A value
or default reads the unmapped `RawParam.<name>`, and the Environment's own `Env.Param.<name>`
applies the mapping on the host.

### Values follow the requesting document's extensions

An `environment:` entry is written in the requesting document, so that document's `EXPR`
extension, not the Environment's, decides whether the entry may use list and boolean literals or
expressions. Reading the Environment's extensions instead would make the syntax a Job Template may
use depend on which queue it is submitted to, and a template validator could not check it. The
consequence is that a Job Template without `EXPR` cannot supply a parameter whose type requires
`EXPR`, such as the Docker sample's LIST[PATH] `Mounts`; its Steps take that parameter's `default`.
A format string that is not a single expression is converted as a submitted Job Parameter value of
the parameter's type would be, which is the rule a `default` already follows, so an INT supplied
as `"{{Param.SeatCount}}"` is the same INT as one supplied as a literal.

### Path values use the submission path format

Path mapping rules are keyed on the path format of the system that submitted the Job. A path value
that is not in that format cannot be mapped correctly, and "absolute" depends on the format (`/opt/x`
is not absolute on Windows). This RFC therefore interprets every PATH and LIST[PATH] Environment
parameter value in the Job's submission path format. Values taken from PATH Job Parameters through
`RawParam.*` are always in that format; that is the recommended source, and the samples use it.

The alternative, allowing a PATH value only as a single `{{RawParam.<name>}}`, would guarantee the
format, but it would forbid useful forms such as `{{RawParam.WorkDir}}/cache` and a list of several
Job Parameters' paths, which the Docker sample's `Mounts` needs. Literal PATH values remain allowed and
are portable only between submitters on one operating system, which the specification says. A
host-local path owned by the Environment Template, such as a cache directory on every Worker Host,
needs no mapping and is better a STRING.

### When a `default` is checked

A literal `default` is fixed text, so a literal that violates its own constraints is an author error
and is reported at template validation, as for a Job Parameter. A `default` that reads a Job Parameter
is different: it is the queue's job-wide fallback for Job Templates that say nothing. If every Step
that enters the Environment supplies its own value, the fallback is never used, and the Job Parameter
it reads may be empty or meaningless for this Job. Rejecting such a Job would force the submitter to
set a Job Parameter that changes nothing. The resolved `default` is therefore checked only for the
Steps that take it.

### Environment parameters are not merged with Job Parameters

§1.2.1 merges same-named Job Parameters because a Job has one value per Job Parameter name. An
Environment's parameters are a per-Environment namespace with one value per Step; merging a queue
Conda Environment's `Packages` with a Job Environment's `Packages` would make two independent
Environments share an input they never agreed to share. The names are therefore scoped to the
Environment, and the only link to Job Parameters is the explicit one a `default` writes.

### Not on Step Environments

A Step Environment is entered only by its own Step's Sessions, and its Step writes its actions. The
Step can write any value directly in the Environment, or use a Step `let` binding, so a parameter
block there would add a second way to say the same thing. The same holds for `runScope`, which could
only say `ONLY_REQUESTED` by the one Step that owns it, and for `hostRequirements`, which the Step
already has.

### `RANGE_EXPR` excluded, `LIST[PATH]` permitted

`RANGE_EXPR` describes a parameter space, which belongs to a Step. It can be added later without
changing anything here.

The first draft also excluded `LIST[PATH]`, whose Job Parameter defaults are relative paths joined
with the template directory (§2.12). Once Environment parameter paths are defined to be absolute in the
submission path format, that objection no longer applies, and each element is mapped as a PATH value
is, exactly as for a `LIST[PATH]` Job Parameter. The Docker sample needs it: a container must mount
every host path the Step's Tasks read or write, after mapping, and a `LIST[STRING]` of `RawParam`
values would mount the submitter's paths instead of the Worker Host's.

### Required parameters are allowed

A parameter without a `default` makes the Environment useless unless each Step that enters it
configures it, which is the right contract for a Job Environment like `RenderLicense` whose author
wants every Step that uses it to say which product it uses. With `runScope: ONLY_REQUESTED` the
Steps held to that are exactly the Steps that asked. For an `ALL` Environment in an Environment
Template the same rule would break every Job Template that does not know the parameter, so this RFC
states the consequence and leaves the choice to the queue administrator, who normally gives every
parameter a default or makes the Environment `ONLY_REQUESTED`.

### `required: false` is an error on a Job Environment entry

A Job Environment is always present, so `required: false` on an entry that names one can only be a
mistake: either the author meant an external Environment and the name collides with a Job Environment,
or the author misunderstood the field. Making it a template validation error reports the first case,
which otherwise silently changes which Environment the entry refers to.

### No extension dependency on `EXPR`

The core of the feature, string and numeric values passed to a provisioning script, needs nothing from
the expression language. Requiring `EXPR` would keep this feature from schedulers and templates that
have not adopted it. With `EXPR`, list and boolean parameters and typed expression defaults become
available, as they do for Job Parameters.

## Security Considerations

### Trust model

Environment parameter values are written by the author of the Job Template (in `environment:`
entries) or come from Job Parameters (through a `default`), and the submitter already controls both.
They are executed by Environment scripts that the queue administrator owns. That is the same trust
relationship as a queue Environment that reads `{{Param.CondaPackages}}` today: this extension grants
the submitter no new privilege, it only lets the same submitter-controlled input differ per Step. A
queue administrator who does not want submitters to choose, for example, arbitrary container images
restricts the parameter with `allowedValues`, which a scheduler checks at job creation before any
Worker Host runs anything.

Environment Template authors must still treat `Env.Param.*` as untrusted input in their scripts, as
they should treat `Param.*` today: the values are chosen by whoever submits the Job, which on a shared
queue is not the administrator.

### Do not substitute values into script source

A format string substitutes text, and an embedded file's `data` is script source. A script line such as
`PACKAGES='{{Env.Param.Packages}}'` breaks when the value contains `'`, and a crafted value such as
`x'; curl ... | sh; '` runs arbitrary commands as the job user. Quoting in the template does not help,
since the value can contain any quote character.

Pass values to scripts in one of these ways instead:

* **As `args` of the action.** Each `args` element reaches the process as one argv entry, with no shell
  parsing. The samples and Basic Examples do this:
  `args: ["{{Env.File.Enter}}", "{{Env.Param.Packages}}"]`, and the script reads `"$1"`. With `EXPR`, a
  list-typed parameter such as the Docker sample's `Mounts` expands to one argv entry per element.
* **Through `variables`.** An Environment's `variables` set environment variables before its actions
  run, and a script reads `"$NAME"`, again without parsing the value as code.
* **With `repr_sh`, `repr_cmd`, or `repr_pwsh`** (RFC 0006, requires `EXPR`) when a value must appear in
  a shell command line, as RFC 0008 recommends for wrapped commands.

A script that intentionally word-splits a value, as the Conda sample does with its package list, should
disable pathname expansion (`set -f`), so that a value containing `*` is not expanded against the
Session directory.

### Values in `onExit`

An Environment's `onExit` sees the same values as its `onEnter`, so cleanup driven by a parameter (for
example `license-release --product {{Env.Param.Product}}`) acts on what `onEnter` set up, and a value
cannot change between the two.

## Prior Art

* **Nextflow** — Every `process` can carry its own `conda`, `container`, `module`, and `spack`
  directives, and a pipeline's configuration selects which of them is honored with
  `conda.enabled = true` or `docker.enabled = true`. The same pipeline thus runs with Conda,
  Docker, Singularity, or preinstalled software, and each process gets only its own tools. This is the
  closest precedent: per-unit software requirements, written once per kind of provisioning, with the
  execution platform choosing which kind applies. This RFC's `required: false` entries are the same
  pattern, with the queue's attached Environments playing the role of the enabled directive. Nextflow
  builds the provisioning mechanisms into the engine; OpenJD keeps them in Environment Templates the
  queue owns.
* **Snakemake** — Each `rule` can declare `conda:` (an environment file), `container:`, and
  `envmodules:`, and the command line selects which is used: since Snakemake 8 with
  `--software-deployment-method` (`--sdm conda`, `--sdm apptainer`, `--sdm env-modules`), and before
  it with `--use-conda`, `--use-singularity`, or `--use-envmodules`. Snakemake caches each distinct
  Conda environment by the hash of its definition, which is the reuse a scheduler can achieve by
  grouping Sessions by effective values.
* **Deadline Cloud sample queue environments** — The in-ecosystem baseline. Deadline Cloud's
  sample Conda queue environments take `CondaPackages` and `CondaChannels` Job Parameters, and its
  Rez sample takes `RezPackages`, each applied to every Session of a Job. This RFC generalizes them: the
  same Environment Templates gain a per-Step value that defaults to those Job Parameters, which is
  exactly the shape of this RFC's samples. Deadline Cloud's improved-caching Conda sample names each
  environment by a hash of its package list and reuses it across Sessions on a host, which is the
  reuse that grouping Sessions by effective values makes more frequent.
* **Kubernetes `RuntimeClass`** — A `RuntimeClass` carries `scheduling.nodeSelector` and
  `tolerations` that are merged with the pod's own by conjunction, so a pod that asks for a runtime is
  placed only where that runtime exists. That is the shape of an Environment's `hostRequirements`
  combined with a Step's: the platform-owned layer states its placement needs and the unit of work
  inherits them by using it.
* **Apache Airflow** — With the Kubernetes executor, each task can set
  `executor_config={"pod_override": ...}` to run in its own image or with its own resources, while
  the executor itself is configured once for the deployment. That is a platform-owned provisioning
  layer configured per unit of work, the closest match to an Environment Template that each Step
  configures.
* **Rez** — `rez env <packages> -- <command>` resolves and enters a software environment per
  invocation, and farm integrations run each job under its own request. The request is per job or per
  command line, not a declared input of a farm-owned environment.
* **Argo Workflows** — Every `template` in a workflow specifies its own `container` (image, command,
  resources), and `inputs.parameters` let a step pass values into a template. A parameterized template
  invoked from several steps with different `arguments` is the shape of this RFC's parameterized Job
  Environment. Argo has no notion of a queue-supplied environment around a template, since every step
  is its own container.
* **Slurm and Environment Modules** — A batch script runs `module load colmap/3.11` per job step, or
  `srun --container-image=...` (with Pyxis/Enroot) per step. The selection lives in the script text, so
  no tool can see which software a step needs without parsing shell, which is what the
  [Tooling parseable](https://github.com/OpenJobDescription/openjd-specifications/wiki/Design-Tenets)
  tenet asks OpenJD to avoid.
* **GitHub Actions** — `jobs.<id>.container` and `runs-on` select the execution environment per job,
  and reusable workflows take typed `inputs` with defaults. A job's choice is written where the job is
  defined, as an `environment:` entry is written on the Step.
* **CWL** — `requirements` and `hints` on each `CommandLineTool` (`DockerRequirement`,
  `SoftwareRequirement` with package names and versions) state what a tool needs, and a hint is
  optional while a requirement is not. That distinction is `required: false` versus `required: true`.

Across these systems the unit of work names its own software, and the platform decides how it is
provided. What none of them has is a provisioning layer owned by someone other than the workflow
author, which OpenJD has in Environment Templates; this RFC is what lets a Job Template speak to that
layer per Step.

## Implementation Impact

- **Rust (openjd-rs)**: `openjd-model` gains the `parameterDefinitions`, `runScope`, and
  `hostRequirements` fields on `Environment`, reusing the existing parameter-definition and
  host-requirement types with a validator that rejects the excluded fields and types; the `environment`
  variant of `StepDependency` with its `required`, `enableIf`, and `parameters`; the `Job.EnvNames`
  symbol; and scope rules for `Env.Param.*` and `Env.RawParam.*`. Job creation computes, per Step, the list of Environments its
  Sessions enter (filtering `ONLY_REQUESTED` Environments by the Step's entries) and the effective
  values for each, and stores both on the Job's Steps in place of the single resolved Environment
  list; the external resolution reuses the code that already merges Environment Template parameter
  definitions. The attached Environment Templates are determined, and `Job.EnvNames` made concrete,
  before any Step `let`, entry *enableIf*, or *parameters* value is evaluated; entries are then resolved
  against them. The `hostRequirements` type is reused on `Environment`, and job creation resolves each
  Environment's requirements per entering Step and combines them with the Step's by the §3.3.3 rules,
  storing the combined set on the Job's Step.
  `openjd-sessions` takes the effective values as part of entering an Environment, the way it takes
  Job Parameter values today, and adds them to the Environment's symbol table. `openjd run` passes the
  values of the Step being run. The work is moderate and mostly in validation.
- **Python (openjd-model, openjd-sessions)**: The same changes in the pydantic models and the
  session's symbol table. As RFC 0009 notes, the Python packages are intended to become bindings over
  openjd-rs, in which case they inherit the feature.
- **Schedulers**: Store each Step's Environment list, effective values, and combined host
  requirements, pass the values to the Worker when a Session enters each Environment, and resolve
  external names when attaching Environment Templates. A scheduler that places Sessions by a Step's
  `hostRequirements` reads the combined set instead, which has the same shape. A scheduler that assumes every Session enters every Job Environment must instead use the
  per-Step list, which the Job carries. A scheduler that shares Sessions across Steps whose
  Environments match keeps that comparison, now over per-Step lists, and must also compare the Steps'
  effective values, adding a Step's Tasks to a Session only when they equal the values the Session
  entered its Environments with. A scheduler that never shares a Session across Steps needs no grouping change. A
  scheduler may additionally use the values for host affinity, which is an optimization and not
  required.

Nothing here is harder in one language than the other. The effective values are a small map per
Step and Environment, computed once at job creation, so there is no runtime cost.

## Rejected Ideas

### A `stepOverridable` flag on template-level Job Parameters

The discussion's first sketch marked a Job Parameter such as `CondaPackages` as overridable and let a
Step set it. Rejected for three reasons. First, a Job Parameter has one value per Job by definition,
and every format string that reads `Param.CondaPackages` (in a Step script, a name, a host
requirement) would then have a value that depends on where it is read, which breaks the meaning of
`Param.*` everywhere. Second, the Job Parameter is declared by whichever template wins §1.2.1 merging,
so the flag would have to be merged too, and a Job Template could make a queue's parameter overridable
or not without the queue's agreement. Third, it gives the Environment no say in what a Step may
change: every overridable parameter becomes Step-settable for every Environment that reads it. An
Environment parameter is declared by the Environment, read only by it, and defaults to the Job
Parameter, which gets the same compatibility without any of these effects.

### A flat per-Step map of values

A Step could carry `environmentParameters: { Packages: "colmap=3.11", Image: "..." }`, and each
Environment would read the names it knows. Rejected because names collide: a Conda Environment and a
Pip Environment can both reasonably call their input `Packages`, and a flat map would send the same
value to both. It also cannot express `required`, cannot be checked against a definition (which
Environment does `Image` belong to?), and gives a scheduler no way to tell which Environment a typo
was meant for. Keying the values by Environment name in an `environment:` entry fixes all four.

### Letting a Step's script read `Env.Param.*`

A Step might want to read the value it supplied, for example to log the Conda packages it asked for.
Rejected because the reference is ambiguous: two Environments wrapping the Step may declare the same
parameter name, and `Env.*` inside an Environment already means "this Environment". A Step that needs
the value already has it, since it wrote it; with `EXPR` it can bind it once with a Step `let` and use
the binding in both the `environment:` entry and its script.

### Values per Task

Letting Task parameters feed Environment parameters would make every Task its own Session, defeating
the amortization Environments exist for. A Step whose Tasks need different software is two Steps.

### `required` defaulting to `false`

With an optional default, the common case of a Job Template written for the Environments of its own
`jobEnvironments` would carry no difference, but a misspelled external name would be silently
ignored, and a Job written for a Conda queue would run on a non-Conda queue and fail inside its Tasks
instead of at submission. Listing something as a dependency should mean depending on it.

### Matching external Environments by something other than name

An Environment Template could carry a separate "kind" (`conda`, `docker`) that entries match on, so a
Job Template could say "any Conda-like Environment". Rejected for now: Environment names are already
how a queue's Environments are identified and ordered, and a kind registry would need a vocabulary the
specification does not have. The `required: false` pattern with one entry per expected name covers the
motivating cases, and `Job.EnvNames` lets a template choose among the names a queue attaches. A
future RFC can add a matching key if queues standardize on one.

### `runScope` as a list of Session kinds

RFC 0009's earlier `runScope: [TASK, SERVICE]` selected Sessions by kind. See
[`runScope` lives in this RFC](#runscope-lives-in-this-rfc) and
[Two values rather than a kind list](#two-values-rather-than-a-kind-list). The one thing the kind
list could say that `ONLY_REQUESTED` cannot is "every Step but no Service" without each Step
writing an entry. RFC 0009's own motivation is that Services should get the Tasks' provisioning, so
that case is not one to make easy.

### `runScope` on the entry instead of the Environment

A Step could write `environment: Conda, exclusive: true` to mean "enter this for me and not for
Steps that do not ask". Rejected because whether an Environment is opt-in is a property of the
Environment, decided by its author: an auditing Environment must not be something a Step can turn
off for everyone else, and two Steps could disagree. The Environment's author knows whether it is a
menu item or a mandate.

### Environment parameters on Step Environments

See [Not on Step Environments](#not-on-step-environments). A Step Environment's own Step is the only
possible supplier, so the values would always be written by the same author in the same Step.

## Open Questions

None block acceptance. The following are noted for the review:

1. **Standard names for queue Environments.** The portability of a Job Template that writes
   `environment: Conda` depends on queues naming their Conda Environment `Conda` and its input
   `Packages`. [Recommended names](#recommended-names-non-normative) proposes a table, which the
   samples follow. Whether it should become a wiki conventions page on acceptance, and which further
   Environments it should cover, is open.
2. **Interaction with `WRAP_ACTIONS` beyond the wrapper's own values.** A wrapping Environment reads
   its own parameters (the image to run) but not those of the Environments it wraps. No use case for
   the latter is known; `WrappedEnv.Param.*` is listed under [Future Work](#future-work).
3. **The extension name.** With `runScope`, `ENV_PARAMS` covers more than parameters: it is the
   extension under which a Step requests an Environment. The name is kept for continuity with the
   discussion thread; a reviewer may prefer one that names the request.
4. **`enableIf` over `Env.Param.*`.** [Composition with CONTROL_FLOW](#composition-with-control_flow)
   lets an Environment's `enableIf` read its own parameters and be decided per requesting entity.
   Whether that belongs in this RFC's composition section or in RFC 0011 is for the two reviews to
   settle; the semantics are stated here because they rest on effective values.

## Future Work

* **`WrappedEnv.Param.*` for wrap hooks**, so a wrapping Environment can read the effective values of
  the inner Environment whose action it is wrapping, if a use case appears.
* **Matching external Environments by a standard kind**, as in
  [Rejected Ideas](#matching-external-environments-by-something-other-than-name).
* **Late-bound entries under `CONTROL_FLOW`**, letting an entry's `enableIf` and `parameters` read
  `Steps.*` so that, for example, the packages a Step installs depend on an upstream output. The
  Step's Environment set and effective values would then be fixed at step input binding rather than
  Job creation, which touches the scheduler's Session grouping and is left for RFC 0011's review.
* **A Job-level default for a whole `environment:` entry**, so a Job Template could set a value once
  for all its Steps without a Job Parameter. The pipeline sample shows the need: four Steps carry
  eight near-identical entries, and the Docker `Mounts` value and the Conda `enableIf` are the same in
  all four. Today a YAML anchor writes a repeated value once (the sample does this for both), a Job Parameter plus the
  Environment's `default` covers it for queue Environments, and a `jobEnvironments` entry's own
  `default` covers it for Job Environments.

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is
more permissive.
