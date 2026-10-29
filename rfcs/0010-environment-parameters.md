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
    whose earlier drafts defined a `runScope` field that this RFC replaces with `activation`
  * RFC 0011, Control Flow (`CONTROL_FLOW`), whose draft also proposes `enableIf` on an Environment, which this RFC now
    defines at Job creation for RFC 0011 to extend

## Summary

External Environments run setup and teardown actions around a Job's Tasks, and are often used to provide
software such as a Conda or Docker environment. They are configured by Job Parameters, which have one value
for the whole Job, so a Job whose Steps need different software has no way to give each Step its own. This
RFC lets each Step configure the Environments that wrap it. An `<Environment>` declares the parameters a
Step may set in a new `envParameterDefinitions` block and reads them as `{{Env.Param.<name>}}`. A Step names
the Environment and supplies values with a new `environment:` entry in its `dependencies`. The Environment
may also be one of the Job Template's own `jobEnvironments`, and a Job Template can be written to run
whether or not an external Environment it names is attached. A parameter's `default` may read a Job
Parameter, so an existing Environment Template can adopt parameters without breaking Job Templates that
predate them.

This RFC also adds three `<Environment>` fields. `activation: ON_REQUEST` makes an
Environment one that only the Steps requesting it enter, so a scheduler can attach several Environments
without every Session entering all of them. `enableIf` lets an Environment leave out the Steps it has
nothing to do for, decided from each Step's values. `hostRequirements` on an Environment are combined with each
entering Step's own, keeping those Steps off hosts where the Environment cannot run. All values are fixed
when the Job is created, and a Session runs Tasks only of Steps whose Environments and values match the
ones it entered, so a scheduler can group and reuse Sessions by those values.

## Basic Examples

### An external Conda Environment that each Step configures

The scheduler attaches this Environment Template to every Job. The template-level `CondaPackages` Job
Parameter keeps working as it does today for Job Templates that know nothing about this extension:
the Environment parameter `Packages` defaults to it.

```yaml
specificationVersion: "environment-2023-09"
extensions: [ENV_PARAMS, EXPR]
parameterDefinitions:
  - name: CondaPackages
    type: STRING
    default: ""
  - name: CondaChannels
    type: STRING
    default: "conda-forge"
environment:
  name: Conda
  envParameterDefinitions:              # new: values a Step may supply
    - name: Packages
      type: STRING
      default: "{{Param.CondaPackages}}"
    - name: Channels
      type: STRING
      default: "{{Param.CondaChannels}}"
  enableIf: "Env.Param.Packages != ''"
  script:
    actions:
      onEnter:
        command: bash
        # Values reach the script as argv, never as script text.
        args: ["{{Env.File.Enter}}", "{{Env.Param.Packages}}", "{{Env.Param.Channels}}"]
    embeddedFiles:
      - name: Enter
        type: TEXT
        data: ...
```

This Environment keeps the default `activation: ALWAYS`, so a Step that asks for nothing gets the job-wide
`CondaPackages`, as before. Its `enableIf` is decided for each Step from that Step's `Packages`: a Step
that ends up with no packages, from its own entry or from `CondaPackages`, does not enter the Environment at all.
Environment Templates for several ways to provision software (Conda, Rez, and Docker, say) that are attached
together can instead each be marked `activation: ON_REQUEST`, so that a Session enters only the Environments its
Step requests.

### A Job Template whose Steps each ask for their own software

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [ENV_PARAMS, EXPR]
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
      - step: DecodeFrames
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
Session for `DecodeFrames` enters it with `ffmpeg=7`. Where the scheduler attaches a Docker
Environment instead, the `Docker` entries take effect and the `Conda` entries are ignored. Where the
software is preinstalled and neither Environment is attached, both entries are ignored and the Job
runs unchanged. The complete pipeline, with both Environment Templates, is in
[`samples/v2023-09`](../samples/v2023-09); there the Docker entries also name the host paths to
mount in the container and, for `SolveSfM`, request the host's GPUs.

`required: false` covers a Job to which no Environment of the name is attached. It does not cover an
attached `Conda` Environment that declares no `Packages` parameter, such as an Environment Template
written before this extension: there the entry resolves, it supplies a parameter the Environment
does not have, and the submission is rejected, naming the entry. See
[Backward compatibility](#backward-compatibility) for how the Environment Template's author migrates
such an Environment.

### One parameterized Job Environment serving several Steps

An `environment:` entry can name a Job Environment of the same template, so one Environment
definition replaces several near-identical Step Environments. With `activation: ON_REQUEST`, only
the Steps that request the Environment enter it.

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [ENV_PARAMS, EXPR]
name: "Plugin-Selected Renders"
jobEnvironments:
  - name: RenderLicense
    activation: ON_REQUEST
    envParameterDefinitions:
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
      - step: BeautyPass
      - step: VolumePass
    script: ...
```

`Product` has no default, so every Step that requests `RenderLicense` must supply it. `Composite`
does not request it, so its Sessions never check out a license. The two render Steps supply
different values, so their Tasks never share a Session. Had `RenderLicense` kept the default
`activation: ALWAYS`, every Session would enter it and every Step, including `Composite`, would have to
supply `Product`.

### Choosing between attached Environments

A scheduler may attach both `Conda` and `Docker`. The pipeline above would then install COLMAP with
conda inside a COLMAP container, which is wasteful. A Step can read `Job.EnvNames`,
the names of the Environments the Job has once external ones are attached, and make an entry conditional
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
        enableIf: "'Docker' not in Job.EnvNames"
        parameters:
          Packages: "colmap=3.11"
      - environment: Docker
        required: false
        parameters:
          Image: "docker.io/colmap/colmap:latest"
    script: ...
```

When both are attached, the `Conda` entry is disabled: the Step does not request `Conda`, so if `Conda`
is `ON_REQUEST` the Step's Sessions never enter it, and if it is `ALWAYS` they enter it with its
defaults, as a Step with no entry would. When `Conda` alone is attached, the entry is enabled and COLMAP
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
  activation: ON_REQUEST
  envParameterDefinitions:
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
`Docker` is unaffected. On a fleet where only some hosts run containers, the Steps that asked for a
container are placed only on those hosts.

### Execution

For the Video To Composite Job with the `Conda` Environment attached, on two Worker Hosts:

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

Consider a Job that decodes a video into frames with ffmpeg, solves structure-from-motion with
COLMAP, places 3D objects into the solved scene with Maya, renders the composite, and encodes the
result with ffmpeg. Each Step uses different software and consumes the output of the Step before
it, so the pipeline belongs in one Job with dependencies between its Steps.

Today the software for such a Job usually comes from an external Environment, attached to the Job from
an Environment Template; Deadline Cloud's Conda and Docker queue environments are examples. That Environment takes its configuration
from a single template-level Job Parameter (`CondaPackages`, `DockerImage`), so it does the same thing
in every Session of the Job. The Job's author has three options:

1. Install everything in every Session. List ffmpeg, COLMAP, and Maya in `CondaPackages`. Every Session
   resolves and installs all three, which multiplies Session startup time and disk use, and fails
   outright when the packages' dependencies conflict, as they can when two applications pin
   different versions of the same shared libraries (Qt, OpenEXR, or Python, for example).
2. Build one image with everything. This has the same conflict problem, moved into an image build that
   the Job author may not control, and the image is large to pull on every host.
3. Split the pipeline into several Jobs. Each Job gets its own `CondaPackages` value, but the
   dependencies between Steps are lost, and the submitter must chain Jobs by hand.

A Step Environment cannot help, because it has no way to ask an external Environment to do anything
differently. The Job Template could stop relying on external Environments and install software in its own
Step Environments, but then it hard-codes one provisioning mechanism. That works against the Portable
[Design Tenet](https://github.com/OpenJobDescription/openjd-specifications/wiki/Design-Tenets), which asks
that jobs be portable to many different systems and usable with the technology stack of their choice:
applied here, the same Job Template should run whether a Conda, Rez, Docker, or Apptainer Environment
supplies its software.

With this RFC, each Step states what it needs from each kind of provisioning Environment it may meet,
and the external Environment provides that in the Step's Sessions.

### Other workflows

1. **Per-Step container images.** A Docker or Apptainer wrapping Environment (RFC 0008) runs every
   action of a Session inside one container. A pipeline with a GPU training Step and a CPU
   preprocessing Step wants a CUDA image for the first and a slim image for the second. With an `Image`
   Environment parameter each Step names its image, and the wrapping Environment is written once.
2. **Per-Step license or plugin selection.** A render Job with a beauty pass in one renderer and a
   volume pass in another must check out a different license for each, or load a different plugin set
   into the same DCC. A Job Environment with a `Product` parameter does this for every Step, where
   today the author writes one Step Environment per Step that differs only in a string.
3. **Per-Step resource tuning of a shared Environment.** An external Environment that mounts a dataset or
   warms a cache can take a size or a subset per Step (`Subset: "train"` for one Step, `"eval"` for
   another), so each Session prepares only what its Tasks read.
4. **Software version pinning per Step.** A Job that validates a scene across two Maya versions has two
   Steps that differ only in the Maya version, which is an Environment parameter rather than a
   duplicated Job.
5. **A menu of external Environments.** A render farm's operator wants to make Conda, Rez, and a
   GPU driver setup available without imposing all three on every Session. Today each such Environment
   must be written to do nothing unless a Job Parameter asks for it, and every Session still enters and
   exits it. With `activation: ON_REQUEST` the scheduler attaches all of them and each Step enters only the
   ones it requests.
6. **A Job Environment that only some Steps need.** A license checkout, a database connection, or a
   large asset download that two of a Job's five Steps use is today written as a Step Environment on each
   of the two, or as a Job Environment that every Session pays for. An `ON_REQUEST` Job Environment
   is written once and entered only by the Sessions of those two Steps.
7. **An Environment that only works on some hosts.** An external Docker Environment needs a host with a
   container runtime; its Conda Environment, on a mixed fleet, needs Linux. Today an Environment Template
   cannot say so, and the Steps it serves fail at `onEnter` on the wrong host. With `hostRequirements` on
   the Environment, every Session that enters it is placed where it can run, and when the requirement
   depends on a parameter (a GPU only when `Gpus` is set) it applies only to the Steps that set it.

These patterns arise wherever a farm provisions software per Session, such as Conda- and Rez-based
VFX farms, containerized HPC and ML batch systems, and CI systems that run matrix jobs.
[Prior Art](#prior-art) describes workflow systems that let each unit of work choose its own software
environment.

### Why the values come from the Step

A Session enters its Environments and then runs Tasks, which may come from one Step or from several
whose Environments are identical. The Step is the unit that knows what software its
Tasks need, so the Step says what an Environment should do for them. With this
extension, a Session's Environments are entered with the values of the Steps it serves, and a Session
serves several Steps only when their values agree, so one set of values is in force for the whole
Session. Values are fixed per Step rather than per Task, because a Session runs many Tasks under one entry
of each Environment.

### Backward compatibility

This RFC is additive and gated by the `ENV_PARAMS` extension name declared under RFC 0002.

- A document that declares `envParameterDefinitions`, `activation`, `hostRequirements`, or `enableIf` on any
  `<Environment>`, or that uses the `environment:` or `step:` form of `<StepDependency>`, MUST list `ENV_PARAMS` in
  its `extensions`, and with it `EXPR`. A scheduler that does not implement `ENV_PARAMS` rejects such a document under
  RFC 0002.
- `dependsOn:` keeps its meaning and stays valid with or without `ENV_PARAMS`; `step:` is a synonym for it. A Job
  Template that lists `ENV_PARAMS` may use either form, and may mix them. The plan to remove `dependsOn:` applies to a
  future major version of the specification, not to `jobtemplate-2023-09`.
- `ENV_PARAMS` requires `EXPR`: a document that lists `ENV_PARAMS` without `EXPR` is rejected. Every Job Parameter
  type is then available to an Environment parameter, and every value, `default`, and `enableIf` is written in the
  Expression Language. A document that lists neither keeps its current meaning.
- The extension applies per document, as for any extension an Environment Template lists. A Job
  Template that does not list `ENV_PARAMS` runs with an Environment Template that does: every
  Environment parameter takes its default, which is how an Environment Template author keeps existing Job
  Templates working while adding per-Step configuration. Such a Job Template cannot request anything,
  so an attached `ON_REQUEST` Environment is never entered by its Sessions; an author
  who wants existing Job Templates to keep getting an Environment leaves its `activation` at `ALWAYS`. An
  Environment Template that does not list `ENV_PARAMS` runs with a Job Template that does: its
  Environment has `activation: ALWAYS` and no parameters, so the Job Template's `environment:` entries that
  name it find no parameters to set, which is an error only if they supply any (see
  [`<StepDependency>`](#32-stepdependency) constraint 5).
- An Environment Template written before this extension under a [recommended name](#recommended-names-non-normative)
  rejects Job Templates that supply its recommended parameters, even with `required: false`, because
  the entry resolves to an Environment that lacks them ([§3.2.1](#321-resolving-external-environments)
  item 1). Deadline Cloud's console Conda queue environment, an external Environment, is such a case today: it is named `Conda`
  and takes `CondaPackages` and `CondaChannels` as Job Parameters only, so it rejects the pipeline
  sample. So is its sample Rez queue environment (`rez_queue_env.yaml`), named `Rez`, which takes
  `RezPackages` as a Job Parameter only. The remedy is for the Environment Template's author to add
  `ENV_PARAMS` and the recommended `envParameterDefinitions`, with defaults that read the existing Job
  Parameters, as `conda-env-params.yaml` does. That change is itself backward compatible: Job
  Templates that do not supply the parameters see the same job-wide values as before. The rejection
  names the entry and the parameter, so a submitter can see which external Environment needs updating. The rationale is in
  [An Environment without the supplied parameter is an error](#an-environment-without-the-supplied-parameter-is-an-error).
- No existing field changes meaning. A Job with no `environment:` entries, no Environment
  `envParameterDefinitions`, no `activation`, and no Environment `enableIf` behaves as today, including which Environments
  each Session enters and how Sessions are shared between Steps. A Job Template that lists
  `ENV_PARAMS` but no `environment:` entries, run with an Environment Template that declares
  parameters and keeps `activation: ALWAYS`, also shares Sessions as today: every Step takes the same
  defaults, so every Step's values are equal.

## Specification

### Terminology

| Term | Definition |
|------|------------|
| **Environment parameter** | A value declared in an `<Environment>`'s `envParameterDefinitions`, read inside that Environment as `Env.Param.<name>`. Distinct from a Job Parameter, which is declared in a template's top-level `parameterDefinitions` and read as `Param.<name>`. |
| **Request** | A Step requests an Environment when its `dependencies` contain an enabled `environment:` entry naming it: one whose `enableIf` is absent or `true`. The entry may also supply parameter values. |
| **Requesting entity** | The Step whose `dependencies` contain the `environment:` entry. Its Sessions enter the named Environment with the values the entry supplies, unless the Environment's *enableIf* is `false` for the Step. |
| **Activation** | An Environment's `activation`: when a Session enters it. `ALWAYS` means every Session of the Job enters it; `ON_REQUEST` means only the Sessions of the Steps that request it. Either way, the Environment's *enableIf* may exclude a Step's Sessions. |
| **Combined host requirements** | For one Step, the conjunction of its own `hostRequirements` and those of every Environment its Sessions enter, each Environment's evaluated with the Step's effective values. A Session running the Step's Tasks must be placed on a host that satisfies all of them. |
| **Effective values** | For one Step and one Environment its Sessions would enter, the value of each of the Environment's parameters: the value the Step's enabled entry supplies if it supplies one, otherwise the parameter's resolved `default`. Fixed when the Job is created. |
| **Disabled** | An Environment is disabled for a Step when its *enableIf* is `false` for the Step, evaluated at Job creation with the values that are known (§3.2.2). Its Sessions do not enter it, constraint 7 does not apply to it, and the Step's values for it are not checked. |
| **Equal values** | Effective values with the same type and the same resolved value before path mapping. |
| **A Session's values** | For one Session and one Environment it has entered, the parameter values it entered the Environment with. They equal the effective values of every Step whose Tasks the Session runs. |
| **Expression field** | A field whose value is a scalar of its type, or one Expression Language expression written without `{{ }}`, with the target type its `@expr <type>` annotation gives (§7.5). Both *enableIf* fields of this RFC are expression fields. |
| **External Environment** | An Environment the scheduler attaches to a Job from an Environment Template, as opposed to one of the Job Template's own `jobEnvironments`. A Deadline Cloud queue environment is an example. |

### Schema modifications

> Changes to [the template schema](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas).

#### Job Template

> A modification to [`1.1. Job Template`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#11-job-template)

The Job Template schema is unchanged. Extension list addition to item 3 (*extensions*): `ENV_PARAMS` (this RFC), with the sub-item "A document that lists `ENV_PARAMS` MUST also list `EXPR`, and an implementation MUST reject one that does not." Item 7 (*jobEnvironments*) says the Environments "are entered in the order provided at the start of every
Session for Tasks in the Job"; add after that sentence: "With the `ENV_PARAMS` extension, an Environment whose
`activation` is `ON_REQUEST` is entered only in the Sessions of the Steps that request it, and an Environment whose
*enableIf* is `false` for a Step is not entered in that Step's Sessions (see [`<Environment>`](#4-environment))."


#### Environment Template

> A modification to [`1.2. Environment Template`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#12-environment-template)

An Environment Template must be able to declare the extension. This adds the same two keys, with the
same meaning, as the draft of RFC 0009 (`SERVICE`) proposes; whichever of the two is accepted first adds them.

```diff
  specificationVersion: environment-2023-09
+ $schema: <string> # @optional
+ extensions: [ <ExtensionName>, ... ] # @optional
  parameterDefinitions: [ <JobParameterDefinition>, ... ] # @optional
  environment: <Environment>
```

* *$schema* — Ignored, as on the Job Template. Allowed for compatibility with JSON-editing IDEs.
* *extensions* — If provided, a non-empty list of extensions that this document uses, with the same available names and prerequisites as the Job Template. An extension listed here applies to the content of this
  document only; a Job Template does not need to list the extensions used by the Environment
  Templates a scheduler applies to it, and vice versa.

> A modification to [`1.2.1. Merging Environment Template Parameter Definitions`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#121-merging-environment-template-parameter-definitions)

Add at the end of the section:

> The merge applies to the top-level `parameterDefinitions` of each template only. The
> `envParameterDefinitions` of an `<Environment>` (`ENV_PARAMS`) are never merged: each Environment's
> parameters are a namespace of their own, even when two Environments, or an Environment and a Job
> Parameter, use the same name. An Environment parameter's `default` that references `Param.<name>`
> or `RawParam.<name>` reads the merged Job Parameter's value, so a Job Template that redefines a
> Job Parameter's `default` changes the default of every Environment parameter that refers to it.

#### RANGE_EXPR length constraints

> A modification to [`2.10. <JobRangeExprParameterDefinition>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#210-jobrangeexprparameterdefinition-extension-expr)

Add after the list of properties:

> *minLength* and *maxLength* are checked against two strings, and both must satisfy them: the value as written (the
> string submitted for the parameter, or the `default`), and its canonical form. So with `maxLength: 3`, the value
> `1,2,3` is rejected although its canonical form `1-3` fits, and with `minLength: 4`, the value `1-3:2` is rejected
> because its canonical form `1,3` is too short.

This was unstated before. An Environment parameter of type RANGE_EXPR takes the same constraints (§4.5), and its values
compare by canonical form (§3.2.2), so checking only the written value would let two equal values differ in whether they
are accepted, and checking only the canonical form would accept a written value longer than `maxLength`.

#### `<Environment>`

> A modification to [`4. <Environment>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#4-environment)

```diff
  name: <EnvironmentName>
  description: <Description> # @optional
+ envParameterDefinitions: [ <EnvironmentParameterDefinition>, ... ] # @optional @extension ENV_PARAMS
+ activation: <EnvironmentActivation> # @optional @extension ENV_PARAMS
+ hostRequirements: <HostRequirements> # @optional @extension ENV_PARAMS
+ enableIf: <Expression> # @optional @expr bool @extension ENV_PARAMS
  script: <EnvironmentScript> # @optional
  variables: <EnvironmentVariables> # @optional
```

New properties:

* *envParameterDefinitions* — The parameters that a Step whose Sessions enter this Environment may supply values for,
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
* *activation* — Whether every Session of the Job enters this Environment, or only the Sessions that request it.
  Available with the `ENV_PARAMS` extension. `<EnvironmentActivation>` is one of:
    * `ALWAYS` — Every Session of the Job enters this Environment, in its position in the Job's Environment order. This is
      the behavior of every Job Environment and attached Environment without the extension, and the default.
    * `ON_REQUEST` — Only the Sessions of a Step that requests this Environment, through an `environment:` entry in
      its *dependencies* (see [`<StepDependency>`](#32-stepdependency)), enter it. Any other Session does not enter or
      exit it, does not have its *variables* set, and does not have it in its Environment stack. The Environments a
      Session does enter keep their relative order.

  Either way, a Step's Sessions do not enter an Environment whose *enableIf* is `false` for that Step.

  Constraints:
    1. An implementation MUST reject an `<EnvironmentActivation>` it does not recognize.
    2. Permitted on a Job Template's *jobEnvironments* entries and on an Environment Template's *environment*. Not
       permitted on a *stepEnvironments* entry, which is entered only by its own Step's Sessions.
* *hostRequirements* — Requirements on the Worker Host that every Session entering this Environment must satisfy, in
  addition to those of the Step whose Sessions they are. Available with the
  `ENV_PARAMS` extension. The object is a [`<HostRequirements>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#33-hostrequirements)
  with the same shape and constraints as on a `<StepTemplate>`, with these differences:
    1. Its format strings may reference `RawParam.*` of any type and `Param.*` (not PATH or LIST[PATH]) of the Job
       Parameters of the document that contains the Environment, `Job.Name` and `Job.EnvNames`, and this Environment's
       `Env.RawParam.*` of any type and `Env.Param.*` of types other than PATH and LIST[PATH]. They are resolved
       in template scope, without a host, so they cannot read a path-mapped value. No other scope is available: in
       particular not `Step.Name`, a Step's `let` bindings, or `Session.*`. They are resolved when the Job is created,
       once for each Step whose Sessions enter the Environment, with that Step's effective values (see [Effective
       Environment parameter values](#322-effective-environment-parameter-values)), so one Environment may impose
       different requirements on different Steps.
    2. Permitted on a Job Template's *jobEnvironments* entries and on an Environment Template's *environment*. Not
       permitted on a *stepEnvironments* entry, whose Step has *hostRequirements* of its own.

  How the requirements combine with the Step's is specified in
  [Combined host requirements](#333-combined-host-requirements-extension-env_params).
* *enableIf* — Whether a Step's Sessions enter this Environment. Available with the `ENV_PARAMS` extension. An
  [expression
  field](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#75-expression-fields)
  of type `bool`. It is resolved when the Job is created, once for each Step whose Sessions would otherwise enter the
  Environment (by its *activation* and the Step's enabled `environment:` entries, or, for a Step Environment, as one
  of the Step's own), with that Step's effective values (see [Effective Environment parameter
  values](#322-effective-environment-parameter-values)); a `null` result is an error. Like an entry's *enableIf*, it
  is evaluated in template scope, not in a host context: `path` values have POSIX semantics whatever the Job's
  submission path format, and no path mapping applies; a `let` binding it reads, in a Step Environment, has its
  template-scope value. Where it is `false` for a Step, that Step's Sessions do not enter the Environment: they do not
  enter or exit it, do not have its *variables* set, and do not have it in their Environment stack, as for an
  `ON_REQUEST` Environment the Step does not request. If omitted, the Environment is enabled for every such Step.
  Constraints:
   1. In a *jobEnvironments* entry or an Environment Template's *environment*, it may reference exactly the scopes
      that this Environment's *hostRequirements* may (*hostRequirements* item 1): the document's `RawParam.*` of any type and `Param.*` other than
      PATH and LIST[PATH], `Job.Name`, `Job.EnvNames`, and this Environment's own `Env.RawParam.*` of any type and
      `Env.Param.*` of types other than PATH and LIST[PATH].
   2. In a *stepEnvironments* entry, it may reference `RawParam.*` of any type, `Param.*` other than PATH and LIST[PATH],
      `Job.Name`, `Job.EnvNames`, `Step.Name`, and the names bound by the Step's `let`.
   3. No other scope is available: in particular not `Task.*`, `Session.*`, `Env.File.*`, or another Environment's
      parameters.
   4. Permitted only in a document that lists `ENV_PARAMS`.

  `Job.EnvNames` lists an Environment whatever its *enableIf*. An Environment that is disabled for a Step
  ([§3.2.2](#322-effective-environment-parameter-values)) is not held to constraint 7 of
  [`<StepDependency>`](#32-stepdependency) for it, and the Step's effective values for it are not checked; an *enableIf*
  whose result depends on a missing, unresolvable, or unconvertible value is an error, reported at the earliest stage at
  which that is known (§3.2.2). An Environment that is disabled for a Step contributes nothing to that Step's combined
  host requirements ([Combined host requirements](#333-combined-host-requirements-extension-env_params)), and its
  *hostRequirements* are not resolved for it. A check that depends on which Environments a Step's Sessions enter, such
  as the `WRAP_ACTIONS` single-layer rule
  ([`<EnvironmentActions>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#43-environmentactions)
  constraint 2), uses what is known at each stage (§7.5 rule 5). Before the Job is created, a Step's Sessions are known
  to enter an Environment whose *enableIf* is absent or `true` when the Step would otherwise enter it (by `activation:
  ALWAYS`, or through an entry whose *enableIf* is absent or `true`), and known not to enter one whose *enableIf* is
  `false`; any other Environment is undetermined. The check reports an error before the Job is created only when the
  error holds whatever the undetermined Environments turn out to be: two wrapping Environments known to be entered
  violate the single-layer rule whatever a third does. Otherwise it is made when the Job is created, over the Step's
  final set.

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
It has the shape of a [&lt;JobParameterDefinition&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#2-jobparameterdefinition) of the same `type`, with every field that shape
has, as follows:

1. *type* — Any type available to a `<JobParameterDefinition>` in a document that lists `EXPR`, which `ENV_PARAMS`
   requires.
2. *userInterface* — Permitted, with the same shape and constraints as on the `<JobParameterDefinition>` of the same
   type. It describes the parameter to tools, such as a Job Template editor, that present Environment parameters; it
   has no effect on Job creation or how the Job runs, since a Step, not a person submitting the Job, supplies the
   value.
3. *objectType* and *dataFlow* (PATH and LIST[PATH]) — Permitted, as on the `<JobParameterDefinition>` of the same
   type, and informational in the same way as *userInterface*. Which files a Job reads and writes is still declared by
   its Job Parameters; an Environment parameter's *dataFlow* does not add to it.
4. *default* — Optional. A [Format
   String](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#73-format-strings)
   resolved when the Job is created. It may reference `RawParam.<name>` for any Job Parameter, and `Param.<name>` for
   any Job Parameter not of type PATH or LIST[PATH], defined in the top-level *parameterDefinitions* of the same
   document, and no other scope. Those Job Parameters have their merged values (see
   [§1.2.1](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#121-merging-environment-template-parameter-definitions)).
   A `default` consisting of a single `{{ }}` expression is evaluated with the parameter's type as its target type;
   otherwise the resolved string is converted to the parameter's type as a submitted Job Parameter value of that type
   would be, except that a relative path is not joined with any directory (see below). A list-typed parameter has no
   string form, so a list `default` that is not a literal must be a single `{{ }}` expression. A literal `default` is
   written as for a `<JobParameterDefinition>` of the same type.
5. *allowedValues*, *minLength*, *maxLength*, *minValue*, *maxValue*, and *item* — As for the
   `<JobParameterDefinition>` of the same type. They constrain every effective value, whether supplied by a Step or
   taken from the resolved `default`, for an Environment that is not disabled for the Step (§3.2.2).
   For RANGE_EXPR, the value as written that *minLength* and *maxLength* check, besides its canonical form, is the
   literal's string, a format string's resolved string, or the string result of a single `{{ }}` expression. A single
   expression whose result is a `range_expr`, such as `{{Param.Frames}}`, has no other spelling, so only its canonical
   form is checked.

A `default` is checked against the definition at two points:

1. A literal `default`, one that contains no `{{ }}`, is checked at template validation against the parameter's type
   and constraints, as a Job Parameter's literal `default` is. A literal that violates them is an error even when every
   Step supplies the parameter, and even when the Environment's *enableIf* disables it for every Step that would take
   the `default`: the check is of the definition, and no expression affects a literal or its constraints. For PATH and
   LIST[PATH], the checks that depend on the submission path format,
   including absoluteness, are made when the Job is created, as below; a relative literal `default` is then an error
   even when every Step supplies the parameter.
2. Any other `default` is resolved when the Job is created and is checked only where it becomes an effective value,
   that is, for each Step that enters the Environment without supplying the parameter. A resolved `default` that would
   violate the constraints is not an error when every such Step supplies its own value, since a `default` that reads a
   Job Parameter is a job-wide fallback, and the Job Parameter's value may be meaningful only to Jobs that use it.

A parameter without a `default` is required: every Step whose Sessions would enter the Environment, and for which it
is not disabled (§3.2.2), must supply it (see
[&lt;StepDependency&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#32-stepdependency)
constraint 7). For an Environment with `activation: ON_REQUEST` those are exactly the Steps that request it; for one
with `activation: ALWAYS`, every Step of the Job.

A value of a PATH parameter, and each element of a value of a LIST[PATH] parameter, whether supplied by a Step or
taken from the `default`, literal or resolved, is a path in the Job's submission path format: the path format of the
Job's PATH Job Parameter values, which is the `source_path_format` that path mapping rules for the Job are keyed on
(see [How Jobs Are
Run](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run#path-mapping)). When the Job is
created, each such path, for an Environment that is not disabled for the Step (§3.2.2), must be absolute in that
format or a URI; a relative path is an error, since an Environment parameter has no directory to be relative to. Its
`allowedValues`, `minLength`, `maxLength`, and `item` constraints are checked at the same time, against the unmapped
value; path mapping on the Worker Host does not re-check them. Template authors should take path values from PATH Job
Parameters, through `RawParam.<name>`, which always have the submission path format. A literal absolute path is
absolute in only one operating system's format, so a literal PATH value is portable only across submitters that use
that format. A host-local path that an Environment Template owns and that needs no path mapping is better declared as
a STRING.

Within the Environment that defines it, an Environment parameter is referenced as `Env.Param.<name>` and
`Env.RawParam.<name>`. These read the value with which the Session entered the Environment exactly as `Param.<name>`
and `RawParam.<name>` read a Job Parameter of the same type, with the same types and path mapping (see
[`<JobParameterDefinition>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#2-jobparameterdefinition)).

Environment parameters were introduced in
[RFC 0010](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0010-environment-parameters.md).

#### `<HostRequirements>`

> A new subsection appended to [`3.3. <HostRequirements>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#33-hostrequirements)
> after `3.3.2. <AttributeRequirement>`. Add to the end of §3.3's introduction: "With the
> `ENV_PARAMS` extension, an Environment may also have *hostRequirements*; see
> [Combined host requirements](#333-combined-host-requirements-extension-env_params)." After the table of attribute
> capabilities in §3.3.2.1, add: "A host has exactly one value of each of these two capabilities. A custom attribute
> capability may have several." This was always so, and item 2.1 below depends on it. The text below is the wiki text
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
known. They are a conjunction: a host satisfies them when it satisfies each requirement from each source. Amounts do
not add: a Step that requires one GPU and enters an Environment that also requires one GPU reserves one GPU, not two.
For requirements from different sources that name the same capability (names compare after resolution and ignoring
case, as in §3.3.1.1 and §3.3.2.1), the conjunction is equivalent to the requirements below, and an implementation may
present or store it in that form. Each form depends only on the set of requirements from all sources, not on the order
in which they are combined:

1. The [&lt;AmountRequirement&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#331-amountrequirement)s that name one capability combine to one whose *min* is the greatest of
   their *min*s (a missing *min* is 0) and whose *max* is the least of their *max*es (a missing *max* is
   unbounded). If the result's *min* exceeds its *max*, no host can satisfy it, and the submission MUST be rejected,
   naming the Step, the conflicting sources, and the capability. The sources may be the Step and an Environment, or
   two Environments the Step's Sessions enter.
2. The
   [&lt;AttributeRequirement&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#332-attributerequirement)s
   that name one attribute combine together, whatever their number. A host has a set of values of an attribute
   (§3.3.2), and values compare ignoring case. An *anyOf* with one value is the same requirement as an *allOf* with
   that value. Let *B* be the union of every *allOf* list and the value of every *anyOf* with one value; *B* is the
   set of values a host must have.
    1. For `attr.worker.os.family` and `attr.worker.cpu.arch`, of which a host has exactly one value, the requirements
       are equivalent to one *anyOf* whose values are those in every *anyOf* and every *allOf* list. If no value is in
       all of them, no host can satisfy the requirements, and the submission MUST be rejected, naming the Step, the
       sources that name the attribute, and the attribute. No smaller set of sources need be at fault: *anyOf*
       `[a, b]`, `[b, c]`, and `[a, c]` share no value, though every two of them do.
    2. For any other attribute, the requirements are equivalent to one *allOf* of the values of *B*, if *B* is not
       empty, together with each *anyOf* of two or more values that contains no value of *B*; an *anyOf* that does
       contain one is implied by the *allOf*. A host may have any set of values, so these requirements can always be
       satisfied, and combining them is never an error. For example, *anyOf* `[a]` from one source and *anyOf* `[b]`
       from another are *allOf* `[a, b]`, which a host with both values satisfies.

The constraints of [§3.3](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#33-hostrequirements) on list length and on duplicate names apply to
each source's *hostRequirements* separately, not to the combined set, so an Environment's requirements are checked on
their own, and combining them adds no error other than the conflicts in items 1 and 2.1.

An Environment's *hostRequirements* apply only to Sessions that enter it. A Step whose Sessions do not enter an
Environment, because its `activation` is `ON_REQUEST` and the Step does not request it, because a `required: false`
entry resolved to nothing, because the entry is disabled, or because the Environment's own *enableIf* is `false`
for the Step, gets none of that Environment's requirements. An
Environment with `activation: ALWAYS` adds its requirements to every Step of the Job for which its *enableIf* is not
`false`.

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
  5. *stepEnvironments* — ... These are entered in the order provided at the start of every Session for Tasks in the
     Step, and exited in the reverse order at the end of those Sessions.
+    With the `ENV_PARAMS` extension, one whose *enableIf* is `false` for the Step is not entered.
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
+ step: "<StepName>"

+ # @extension ENV_PARAMS
+ environment: "<EnvironmentName>"
+ required: <bool> # @optional
+ enableIf: <Expression> # @optional @expr bool
+ parameters: <EnvironmentParameterValues> # @optional
```

> The complete text of the new §3.2, §3.2.1, and §3.2.2 follows, verbatim from the wiki with links made absolute.

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
step: "<StepName>"
```

```yaml
# @extension ENV_PARAMS
environment: "<EnvironmentName>"
required: <bool> # @optional
enableIf: <Expression> # @optional @expr bool
parameters: <EnvironmentParameterValues> # @optional
```

Where:

1. *dependsOn* — Provides the name of another step in the same job upon which this step depends. The Task(s) of this Step
   may be scheduled only when the depended-upon Step has fully completed successfully.
    * *step* — A synonym of *dependsOn*, available with the `ENV_PARAMS` extension. An entry `step: X` means exactly
      what `dependsOn: X` means, and every rule that applies to a *dependsOn* entry applies to it; in particular, a
      *step* entry and a *dependsOn* entry that name the same Step are two entries naming one Step. The name matches
      the other forms, which are keyed by the kind of thing they name (*environment* here). A future major version of this specification is planned to remove *dependsOn* in favor of
      *step*; a Job Template that lists `ENV_PARAMS` should prefer *step*.
2. *environment* — The name of an Environment that this Step requests: its Sessions enter it, and the Step may supply
   values for its parameters. Available with the `ENV_PARAMS` extension. When *enableIf* is present and resolves to
   `false`, the entry is **disabled** and has no effect of any kind: the name is not resolved, nothing is requested or
   supplied, and the Step is treated as having no entry for the Environment. Otherwise the name resolves, in order:
    1. to the entry of the same Job Template's *jobEnvironments* with that `name`, if there is one;
    2. otherwise, to an **external Environment**: the `environment` of an Environment Template that the scheduler
       attaches to the submission, matched by `name` when the Job is created (see
       [Resolving external Environments](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#321-resolving-external-environments)).

   The entry declares that the Step depends on the Environment: its Sessions need that Environment, entered with these
   values. If the Environment's `activation` is `ON_REQUEST`, the entry is what causes this Step's Sessions to enter
   it; if it is `ALWAYS`, the Step's Sessions enter it regardless of the entry, and the entry only supplies values. In
   either case the Environment's own *enableIf* may still exclude the Step's Sessions (see [&lt;Environment&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#4-environment) item 6). The entry does not
   delay scheduling the Step's Tasks, and it does not change the position of the Environment in the order a Session
   enters its Environments.
3. *required* — Whether the Environment must be present. Default `true`. When `false` and the name resolves to no
   Environment when the Job is created, the entry has no effect: the Step's Sessions run without that Environment. A
   *jobEnvironments* entry is always present, so `required: false` is not permitted on an entry that names one
   (constraint 8).
4. *parameters* — The values this Step supplies, keyed by the names in the Environment's *envParameterDefinitions* (see
   [&lt;EnvironmentParameterDefinition&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#45-environmentparameterdefinition-extension-env_params)). A parameter the entry omits takes its
   `default`. If not provided, the entry supplies no values; it still requests the Environment.
5. *enableIf* — Whether this entry is in effect. An [expression
   field](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#75-expression-fields)
   of type `bool`, with the same available scopes as a *parameters* value (`RawParam.*`, `Param.*` other than PATH and
   LIST[PATH], `Job.Name`, `Job.EnvNames`, `Step.Name`, and the Step's `let` bindings). It is resolved when the Job is
   created, before the entry's name is resolved; a `null` result is an error. It is evaluated in template scope, not
   in a host context: `path` values have POSIX semantics whatever the Job's submission path format, and no path
   mapping applies ([Expression Language
   §1.2.1](https://github.com/OpenJobDescription/openjd-specifications/wiki/2026-02-Expression-Language#121-expression-types));
   a `let` binding it reads has its template-scope value. If omitted, the entry is enabled. A disabled entry is as if
   absent (item 2): for an `ON_REQUEST` Environment the Step's Sessions do not enter it, and for an `ALWAYS`
   Environment they enter it with every parameter at its `default`, subject to constraint 7 and to the Environment's
   own *enableIf*.

`<EnvironmentParameterValues>` is a map from [&lt;Identifier&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#71-identifier) to `<EnvironmentParameterValue>`:

```yaml
<Identifier>: <EnvironmentParameterValue>, # @fmtstring
...
```

An `<EnvironmentParameterValue>` is one of:

* A literal of the parameter's type, written as the `default` of a
  [&lt;JobParameterDefinition&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#2-jobparameterdefinition)
  of the same type is written: a string, an integer or `<intstring>`, a float or `<floatstring>`, a boolean, or a
  list.
* A [Format
  String](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#73-format-strings)
  resolved when the Job is created. Its available scopes are `RawParam.*` for every Job Parameter and `Param.*` for
  every Job Parameter not of type PATH or LIST[PATH], in both cases those defined in the top-level
  *parameterDefinitions* of the document that contains the entry, with their merged values (see
  [§1.2.1](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#121-merging-environment-template-parameter-definitions));
  and `Job.Name`, `Job.EnvNames`, `Step.Name`, and the names bound by the Step's `let`. `Task.*`, `Session.*`, and
  `Env.*` are not available: a value is fixed for the whole Step. A value consisting of a single `{{ }}` expression is
  evaluated with the parameter's type as its target type. Any other format string resolves to a string, which is
  converted to the parameter's type as a submitted Job Parameter value of that type would be, except that a relative
  path is not joined with any directory (see
  [&lt;EnvironmentParameterDefinition&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#45-environmentparameterdefinition-extension-env_params)).
  A list-typed parameter has no string form, so its value must be a list literal or a single `{{ }}` expression.



A value for a parameter of type PATH or LIST[PATH] is a path in the Job's submission path format; see
[&lt;EnvironmentParameterDefinition&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#45-environmentparameterdefinition-extension-env_params).

Constraints:

1. Exactly one of *dependsOn*, *step*, or *environment* must be present. *required*, *enableIf*, and *parameters* are
   permitted only with *environment*.
2. *environment* must not name a *stepEnvironments* entry of the same Step.
3. No two *environment* entries of one *dependencies* list may name the same Environment.
4. *parameters*, if provided, must contain at least one entry and at most 50 entries.
5. Every key of *parameters* must name a parameter declared in the named Environment's *envParameterDefinitions*. An
   Environment that declares none, including one from a document that does not list `ENV_PARAMS`, accepts an entry with
   no *parameters* and rejects one with any.
6. Every value must be valid for the parameter it supplies: of its type, and within its `allowedValues`, `minLength`,
   `maxLength`, `minValue`, `maxValue`, and `item` constraints.
7. For every Environment that a Step's Sessions would enter by its `activation` and the Step's enabled entries, except
   one that is disabled for the Step (see §3.2.2), the Step must supply every parameter of that Environment that has
   no `default`. A Step's Sessions enter every Environment of the Job, whether a *jobEnvironments* entry or an
   attached Environment, whose `activation` is `ALWAYS`, so such a parameter of such an Environment must be supplied
   by every Step of the Job for which it is not disabled; and they enter every Environment with `activation:
   ON_REQUEST` that the Step requests, so such a parameter of such an Environment must be supplied by every Step that
   requests it and for which it is not disabled.
8. *required* must not be `false` on an entry that names a *jobEnvironments* entry.
9. *enableIf* is an expression field of type `bool` (§7.5).

Constraints 1–4, 8, and 9 are checked at template validation. Constraints 5–7 are checked at template validation for
entries that name a *jobEnvironments* entry (for constraint 6, against literal values only; constraints 6 and 7 do not
apply to an Environment that is disabled for the Step, but constraint 5 does, see §3.2.2), and when the Job is created
for every entry whose *enableIf* is absent or `true`, once format strings are resolved and external Environments are
known. A check that depends on whether an entry is enabled or an Environment is disabled, which includes constraint 7
and any check of which Steps request an Environment, uses at template validation only what is known then (§7.5 rule 5):
an entry whose *enableIf* is absent or `true` is enabled, one whose *enableIf* is `false` is not, and any other entry,
including one whose *enableIf* is a constant expression such as `"true"`, is undetermined. Template validation reports
the check's error only when it holds however the undetermined entries resolve; otherwise the check is made when the Job
is created. Constraint 3 applies to entries as written, enabled or not, so two entries for one Environment with
complementary *enableIf* conditions are an error; write one entry whose values are conditional instead.

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
   constraints 5 and 6 are checked against that Environment's *envParameterDefinitions*. This applies whatever *required*
   says: `required: false` tolerates a missing Environment, not one that lacks a parameter the entry supplies. An entry
   that supplies parameters to an attached Environment that declares none, such as one from an Environment Template
   written before `ENV_PARAMS`, causes the submission to be rejected, naming the entry and the parameter.
2. If none does, then with `required: true` the submission must be rejected, naming the entry and the missing
   Environment; with `required: false` the entry is ignored.
3. If two or more do, the submission must be rejected, naming the entry and the ambiguous name.
4. An attached Environment with the same name as a *jobEnvironments* entry is not reachable by name from the Job
   Template. Implementations should warn when they see one. With `activation: ALWAYS`, its parameters without a
   `default` reject the submission only if some Step's Sessions enter it (constraint 7 below); one that its *enableIf*
   disables for every Step needs no values.
5. Likewise, an attached Environment with the same name as a *stepEnvironments* entry of a Step is not reachable by name
   from that Step (constraint 2). Other Steps of the Job reach it as usual.

Each entry that resolves applies independently of the others: when the scheduler attaches several of the Environments
a Step requests, the Step's Sessions enter all of them, each with the values its own entry supplies. An attached
Environment with `activation: ON_REQUEST` is entered by the Sessions of exactly the Steps whose entries resolve to it,
less any for which its *enableIf* is `false`;
one with `activation: ALWAYS` is entered by every Session, as without the extension, except the Sessions of a Step
for which its *enableIf* is `false`.

After resolution, constraint 7 is checked over the combined Job: every Step whose Sessions would enter an attached
Environment that is not disabled for the Step, with a parameter that has no `default`, must supply it, or the
submission is rejected. For an attached Environment with `activation: ALWAYS`, every Session would enter it, so such a
parameter rejects every Job Template that does not supply it from every Step; an Environment Template intended for
existing Job Templates therefore gives every parameter a `default`, or uses `activation: ON_REQUEST` so that only the
Steps that request it are held to the rule.

###### 3.2.2. Effective Environment parameter values

For each Step and each Environment its Sessions would enter, the **effective value** of every parameter of that Environment
is determined when the Job is created:

1. If an enabled *environment* entry of the Step names the Environment and supplies the parameter, the resolved value of
   that entry.
2. Otherwise, the resolved `default` of the parameter.

The effective values are part of the Job, and are checked against the parameter's definition when the Job is created
(constraint 6, and see
[&lt;EnvironmentParameterDefinition&gt;](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#45-environmentparameterdefinition-extension-env_params)
for PATH values and for `default`), for each Environment that is not disabled for the Step (below). They are also the
values with which the Environment's *hostRequirements*, if any, are resolved for the Step (see [Combined host
requirements](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#333-combined-host-requirements-extension-env_params)).
Path mapping rules are applied to PATH-typed effective values on the Worker Host, as they are to PATH Job Parameters.

An Environment is **disabled** for a Step when its *enableIf* is `false` for the Step. This is decided before constraint
7 and the checks of the Step's effective values for the Environment: for a literal `false`, already at template
validation (§7.5 rule 5); for an expression, when the Job is created, by evaluating it with what is known then. An
effective value that the Step supplies, or that comes from a `default`, and that converts to its parameter's type is
known, whether or not it meets the parameter's other constraints. A parameter without a `default` that the Step does not
supply, a value whose expression fails to resolve (an index out of range, say), and a value that does not convert, are
unknown: each is `unresolved[T]` of its parameter's type, and the expression is evaluated as the Expression Language's
progressive evaluation defines, so its result is `true`, `false`, or `unresolved[bool]`. Where the result is `false`,
the Environment is disabled: the Step's Sessions do not enter it, constraint 7 does not apply to it, and the Step's
effective values for it are not checked against their parameters' type and constraints. That covers constraint 6 of
`<StepDependency>`, and the rule that a value for a list-typed parameter is a list literal or a single `{{ }}`
expression, which is part of its type. A disabled Environment is still held to the checks of an entry as written:
constraint 5, and the rule that each value is one of the forms of an `<EnvironmentParameterValue>`, so a `null` or an
object is an error. Where the result is `true`, the Environment is not disabled, and constraint 7 and the checks of its
effective values apply. Where the result is unresolved, it depends on a missing, unresolvable, or unconvertible value,
and that is an error. The error is reported at the earliest stage at which it is known: at template validation when the
value is known to be missing (no `default`, and no entry supplies it whatever an undetermined entry *enableIf* resolves
to) and the result depends on it whatever every value still unknown at that stage resolves to (§7.5 rule 5), and
otherwise when the Job is created.

Two effective values are **equal** when they have the same type and the same value as resolved when the Job is created,
before any path mapping. An INT parameter supplied as `1` by one Step and as `"1"` by another resolves to the INT 1 for
both, so the values are equal; two PATH values are equal when their unmapped strings are equal; two lists are equal when
they have equal elements in the same order; and two RANGE_EXPR values are equal when they have the same canonical form
([§3.4.1.1.1](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#34111-intrangeexpr)),
which is when they denote the same integers. `1-3` and `1,2,3` are equal, so Steps that supply them can share a Session.

Each time a Session enters an Environment, it enters it with one set of parameter values and keeps those values until
it exits the Environment. A Session may run Tasks of a Step only when the set of Environments the Session has entered,
and not yet exited, is the set
the Step's Sessions enter (every `ALWAYS` Environment of the Job, every `ON_REQUEST` Environment the Step requests, and
the Step's own *stepEnvironments*, less any whose *enableIf* is `false` for the Step) and, for every Environment in that set, the Step's effective values are equal to the
values the Session entered it with. The first condition is the rule *How Jobs Are Run* already states for Sessions that
run Tasks of several Steps, now applied to a set that `activation` and *enableIf* can make differ between Steps; the second is new. A
Session may therefore run Tasks of several Steps, as before, when their Environment sets and effective values agree; the
set of Steps it will serve need not be known when it starts. Between Tasks of different Steps, a Session may also exit
Environments, in the reverse of the order it entered them, and enter others, or enter an Environment again with
another Step's values, as it already may for Step Environments.
See [How Jobs Are Run](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run#environment-parameters).

#### Extensions specified after this one (non-normative)

This RFC is specified against mainline. Every value it adds is fixed when the Job is created: effective values, each
Step's Environment set, each Environment's *enableIf*, and combined host requirements. An RFC accepted after this one
that adds a kind of Session or a later stage of evaluation specifies how it extends these rules. Points such an RFC
will need to decide include:

* For RFC 0009 (`SERVICE`): whether a Service may request Environments and supply values as a Step does; how an
  Environment that configures a Service's clients avoids a Service depending on itself; how the wrap-hook rules of
  RFC 0008 apply when Service Sessions enter a wrapping Environment; and the merged constraint 1 of
  `<StepDependency>` once `service:` is another form.
* For RFC 0011 (`CONTROL_FLOW`): whether a Step Environment's *enableIf* may read `Steps.*` and so be decided at step
  input binding; that an entry's *enableIf* and *parameters* values and an Environment's *hostRequirements* stay fixed
  at Job creation or move with it; and that a Step skipped by its own *enableIf* contributes nothing to these checks.
  RFC 0011's own fields whose value is one typed expression (a Step's *enableIf*, and a loop's *initial*, *next*,
  *while*, *until*, and output *value*) would be written as expression fields (§7.5) rather than single-expression
  Format Strings, so that every such field reads the same way.

#### Value references

> A modification to [`7.3.1. Value References`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#731-value-references)

```diff
  |`Env.File.<name>`|...|Available within the Environment Script Actions and Embedded Files.|
+ |`Env.Param.<ParamName>`|The value of the Environment parameter `<ParamName>` with which the Session entered the Environment: the effective value for every Step whose Tasks the Session runs, which are equal (see [§3.2.2](#322-effective-environment-parameter-values)). Read as `Param.<ParamName>` reads a Job Parameter of its type. Requires the `ENV_PARAMS` extension.|Available within the Environment that declares the parameter: its `variables`, its `hostRequirements` and `enableIf` (types other than PATH and LIST[PATH]), and its Script's `let`, Actions, and Embedded Files. Not available in any other Environment or in a Step.|
+ |`Env.RawParam.<ParamName>`|The same value, read as `RawParam.<ParamName>` reads a Job Parameter of its type. Requires the `ENV_PARAMS` extension.|As for `Env.Param.<ParamName>`.|
+ |`Job.EnvNames`|The `name` of each Job Environment of the Job: the Job Template's `jobEnvironments` and the Environments attached from Environment Templates, whatever their `activation`, in the order a Session that entered all of them would enter them. Step Environments are not included. A name appears once per Environment, so it appears twice if two attached Environments share it. A list, so usable only within a `{{ }}` expression or an expression field. An Environment that its `enableIf` disables for some or every Step is still listed: the list is read by expressions that feed effective values, which an Environment's `enableIf` may read, so excluding disabled Environments would be circular. Requires the `ENV_PARAMS` extension.|Available in every format string or expression field of a Job Template or Environment Template that is resolved at Job creation or later, including an Environment's `hostRequirements` and `enableIf` and the `let`, actions, and embedded files of its script, and concrete at Job creation once the attached Environments are known. Not available in an Environment parameter `default`, which reads only `Param.*` and `RawParam.*`, nor in the Job Template's `name`.|
```

`Job.EnvNames` lets a Step decide what to ask of the external Environments that happen to be attached, for example to
prefer a container Environment over a Conda one when both are present
([Choosing between attached Environments](#choosing-between-attached-environments)). It names what the
Job has, not what a particular Step's Sessions enter, so it is the same for every Step and does not by itself affect
which Steps may share a Session.

A reference to `Env.Param.<name>` or `Env.RawParam.<name>` that does not name a parameter of the enclosing Environment is
a template validation error. A wrapping Environment's wrap hooks (RFC 0008) read the wrapping Environment's own
parameters through `Env.Param.*`; an inner Environment's parameters are not visible to them.

#### Expression fields

> A new section of the Template Schemas, inserted after [`7.4. Template Processing Stages`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#74-template-processing-stages)
> as `7.5`; a sentence added to §7.4 after the description of the `@fmtstring` annotations: "An expression field,
> marked `@expr <type>` (see §7.5), states its stage in its own definition."; and a bullet added to the Notations list:
> "`@expr <type>` - The value of the annotated property is an expression field with target type `<type>`. See
> Expression Fields." Both *enableIf* fields of this RFC are
> expression fields; no existing field changes kind. The text follows, verbatim from the wiki with links made absolute.

##### 7.5. Expression Fields

An **expression field** holds one [Expression Language](https://github.com/OpenJobDescription/openjd-specifications/wiki/2026-02-Expression-Language) expression, written without
`{{ }}`. This specification marks one with `@expr <type>`, where `<type>` is the field's **target type** in the
Expression Language's type notation, for example `@expr bool`. A target type may be a choice of types or `any`, as in
`@expr int | string`, where a field means to accept more than one. An expression field is not a Format String: no text
surrounds the expression and nothing is interpolated. Expression fields are available only in a document that lists the
`EXPR` extension.

The value of an expression field, in YAML or JSON, is one of:

1. A boolean, integer, or float scalar whose type fits the target type (rule 2), which is used as written: `true` or
   `false` for `bool`, `3` for `int`, `3` or `3.0` for `float`. A string is never read this way, so `"true"` is the expression `true`, not the
   literal; for a target type of `string`, every value is an expression, and a constant string is written as a string
   literal inside it (`"'abc'"`).
2. A string, parsed as a single expression. In YAML, quote it where YAML requires: an expression that begins with a
   quote, a bracket, or another YAML indicator character, or that contains `: ` or ` #`, must be enclosed in quotes, as
   in `"'Docker' not in Job.EnvNames"`.

A null value (`null`, `~`, or a key with no value in YAML) is an error; it is not read as an omitted field.

Rules:

1. The expression must parse. `{{` is not part of the expression grammar, so a value written as a Format String
   (`"{{x}}"`) is a parse error; an implementation should say in its message that an expression field takes an
   expression without `{{ }}`.
2. The expression's static type must **fit** the target type, as
   [Expression Language §1.2.3](https://github.com/OpenJobDescription/openjd-specifications/wiki/2026-02-Expression-Language#123-implicit-type-coercion) step 1 defines it: it is the target
   type, or one of the target's choices, or the target is `any`, with list element types fitting element-wise. Of the
   conversions of §1.2.3 step 2, only the widening of `int` to `float` applies (element-wise for lists), as it does for
   arithmetic operands (§2.1.1): an `int` result for a `float` field becomes that `float`. No other conversion applies,
   so a `string` expression is an error for a `bool` field, whatever its text, and a field that accepts unrelated types
   says so in its target type, as in `int | string`.

   Where the target type does not admit `nulltype`, an expression of an optional type `U?` whose `U` fits is also
   accepted, and a `null` result is an error when the expression is evaluated. This holds when the field itself is
   optional, too: a `null` result never means the field was omitted. An expression whose static type is `nulltype` is
   an error unless the target admits it. At template validation, `unresolved[U]` is checked as `U`.
3. Each field's definition gives the scopes the expression may reference and the stage at which it is evaluated. The
   `@fmtstring` annotations do not apply to expression fields.
4. Parse errors, type errors, and references to names that are not available are reported at template validation.
5. Until an expression field is evaluated, its value is known only when it is a boolean, integer, or float scalar
   (item 1); any other value, a constant expression such as `"true"` included, is unknown at an earlier stage. A check
   made at an earlier stage that depends on such a field's value reports an error only when the error holds for every
   value the unknown fields could take; otherwise the check is made again at the stage where the fields are evaluated,
   with the values they then have. An implementation should report an error at the earliest stage at which it holds
   for every such value.

#### Template processing stages

> A modification to [`7.4. Template Processing Stages`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#74-template-processing-stages)

* **Template validation** — `Env.Param.*` and `Job.EnvNames` are added to the Unknown Values. `Env.Param.*` is
  type-checked using the declared types of the Environment's *envParameterDefinitions*, and `Job.EnvNames` is
  `unresolved[list[string]]`. Literal Environment parameter `default`s, and literal *parameters* values for
  *jobEnvironments* entries, are checked against their definitions.
* **Job creation** — Each Step's Environment parameter effective values are added to the Known Values at the end of
  the stage. The Environment Templates attached to the Job are determined first, and `Job.EnvNames` is concrete before
  any TEMPLATE-scope expression of the Job, including a Step's `let` bindings, is evaluated. Then each `environment:`
  entry's *enableIf* is evaluated; each enabled entry that names no *jobEnvironments* entry is resolved against the
  attached Environments (§3.2.1); the set of Environments each Step's Sessions would enter is fixed from each
  Environment's `activation` and the Step's enabled entries; Environment parameter `default`s and `environment:`
  values are resolved into each Step's effective values; then each Environment's *enableIf* is resolved for each such
  Step, with the effective values that are known (§3.2.2), and removes the Environment from the Step's set where it is
  `false` (§4); constraint 7 and the other checks of the effective values are made for the Environments that remain in
  each Step's set (§3.2.2); and each Environment's *hostRequirements* are resolved with each entering Step's effective
  values and combined with the Step's own (§3.3.3). Environment parameter PATH and LIST[PATH] values are not joined
  with any directory and must be absolute or a URI (see [§4.5](#45-environmentparameterdefinition-extension-env_params)).
* **Task execution** — `Env.Param.*` is available in the Environment's host-evaluated format strings, with path mapping
  applied to PATH and LIST[PATH] values.

### Modifications to the Expression Language

> A modification to [`1.2.2. Built-in Symbol Types`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2026-02-Expression-Language#122-built-in-symbol-types)
> of the Expression Language, after *Embedded File Symbols*.

```diff
+ ##### Environment Parameter Symbols
+
+ With the `ENV_PARAMS` extension, an Environment that declares *envParameterDefinitions* (see
+ [Template Schemas](2023-09-Template-Schemas#45-environmentparameterdefinition-extension-env_params)) can read their
+ values within that Environment only:
+
+ | Symbol | Type | Description |
+ |--------|------|-------------|
+ | `Env.Param.<name>` | As `Param.<name>` for a Job Parameter of its type (table above) | The value with which the Session entered the Environment, read as `Param.<name>` reads a Job Parameter; in the Environment's *hostRequirements* and *enableIf*, which are resolved at Job creation, the Step's effective value |
+ | `Env.RawParam.<name>` | As `RawParam.<name>` for a Job Parameter of its type (table above) | The same value, read as `RawParam.<name>` reads a Job Parameter |
+
+ With the `ENV_PARAMS` extension, templates can also read:
+
+ | Symbol | Type | Description |
+ |--------|------|-------------|
+ | `Job.EnvNames` | `list[string]` | The names of the Job's Job Environments, including those attached from Environment Templates; `unresolved[list[string]]` at template validation. See [Template Schemas §7.3.1](2023-09-Template-Schemas#731-value-references) for its order and where it is available. |
```

And in [`1.3.2. Evaluation Within Template Schemas`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2026-02-Expression-Language#132-evaluation-within-template-schemas),
before the paragraph on a format string that is exactly `"{{<expr>}}"`:

```diff
+ Some fields of the Template Schemas are **expression fields**, marked `@expr <type>`
+ ([Template Schemas §7.5](2023-09-Template-Schemas#75-expression-fields)): their value is one expression written without
+ `{{ }}`, the target type is the field's `<type>`, and the result must already fit it as step 1 of
+ [section 1.2.3](#123-implicit-type-coercion) defines, except that an `int` widens to `float`; no other conversion
+ of step 2 applies. A field that accepts unrelated types says so in its target type, such as `int | string`. This holds when the field is optional,
+ too: an expression of an optional type is accepted, and a `null` result is an error rather than omitting the field.
```

And in [`Progressive Expression Evaluation`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2026-02-Expression-Language#progressive-expression-evaluation),
at the end of item 2, "Job parameter binding":

```diff
+    With the `ENV_PARAMS` extension, an Environment's *enableIf* is the exception: a Step's effective value that is
+    missing, fails to resolve, or does not convert is `unresolved[T]` of its type, and a result that stays unresolved
+    is an error ([Template Schemas §3.2.2](2023-09-Template-Schemas#322-effective-environment-parameter-values)).
```

### Modifications to How Jobs Are Run

> Modifications to [How Jobs Are Run](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run)

```diff
  Once the Environments have been created and entered for a Session, a series of Tasks are
  run within that Session and the Environments. Tasks from any Step in a Job can run within
  the same Session provided that the set of Environments that are required to run those
- Tasks are identical.
+ Tasks are identical (with the ENV_PARAMS extension, that set depends on which Environments
+ each Step requests and on each Environment's enableIf for that Step) and those Steps' effective values for every Environment's parameters
+ are equal to the values the Session entered it with (see Environment Parameters).
```

And a new section, *Environment Parameters*, after *Running a Session*:

> With the ENV_PARAMS extension, introduced in
> [RFC 0010](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0010-environment-parameters.md),
> an **Environment** may declare parameters, and a Step requests the **Environment** and supplies values for them in an
> `environment:` entry of its `dependencies`. The **Environment** reads the values as `{{Env.Param.<name>}}`. An
> **Environment** may also declare an `activation`: `ALWAYS`, the default, means every **Session** of the Job enters it
> (subject to the **Environment**'s `enableIf`, below), as every Job Environment does without the extension; `ON_REQUEST` means only the **Sessions** of the Steps that
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
> requests, the Step's **Sessions** enter each of them with the values its own entry supplies. For an `ALWAYS`
> **Environment**, an entry only supplies values; for an `ON_REQUEST` **Environment**, the entry is also what makes
> the Step's **Sessions** enter it. An entry may carry an `enableIf` condition, decided when the Job is created; a
> disabled entry is as if it were not written. An **Environment** may carry its own `enableIf`, also decided when the
> Job is created, once per Step and with that Step's values; a Step for which it is false does not enter it. The **Environments** a **Session** does enter keep the order they have in
> the Job.
>
> An **Environment** may also declare `hostRequirements`, which may read its parameters. A **Session** that enters it
> must be placed on a Worker Host that satisfies them as well as the Step's own, and its `amounts` are reserved for the
> **Session** like the Step's. Because they are resolved with each Step's values, one **Environment** can ask for a GPU
> only for the Steps that asked it to use one.
>
> Each time a **Session** enters an **Environment**, it uses one set of parameter values and keeps them until it exits
> the **Environment**: the **Environment**'s `onEnter`, every Task run while it is entered, and its `onExit` all see the
> same values. A **Session** may run Tasks of a Step only when the **Environments** it has entered and not yet exited
> are exactly the ones the Step's **Sessions** enter, and the Step's effective values for each of them are equal to the
> values the **Session** entered it with. Values are equal when they have the same type and the same resolved value
> before path mapping. A **Session** can therefore serve several Steps, as it can without the extension, whenever their
> **Environment** sets and effective values agree, and a render management system may add a Step's Tasks to a
> **Session** that is already running when they do. Path mapping rules are applied to PATH-
> and LIST[PATH]-typed values on the Worker Host, as for PATH Job Parameters. A Step's own script cannot read
> `Env.Param.*`.
>
> Because the values are fixed per Step and known when the Job is created, a render management system can group
> **Sessions** by them. Two Steps that request the same `ON_REQUEST` **Environments** and whose effective values
> are equal for every **Environment** may share **Sessions**; two Steps whose **Environment** sets or values differ
> never do. A render management system may also use the values to
> prefer a Worker Host that recently entered an **Environment** with the same values, for example one that has a
> matching Conda environment or container image cached. For a pipeline that decodes video with ffmpeg, solves
> structure-from-motion with COLMAP, and renders with Maya, each Step's **Sessions** then install or pull only the
> software that Step needs.

### Recommended names (non-normative)

The portability of a Job Template that writes `environment: Conda` depends on Environment Templates using the same Environment and
parameter names. The specification does not mandate names. This RFC recommends the following, which the samples use,
for Environment Templates that provision software. Each parameter defaults to the template-level Job Parameter
that Deadline Cloud's sample queue environments (which are Environment Templates) already use, so existing Job Templates keep working. Such an
Environment keeps `activation: ALWAYS` while existing Job Templates rely on the Job Parameter; when several of these
Environments are attached together for Job Templates that request them, they can be marked `ON_REQUEST`.

| Environment | Parameter | Type | Default | Meaning |
|-------------|-----------|------|---------|---------|
| `Conda` | `Packages` | STRING | `{{Param.CondaPackages}}` | Space-separated package specs. Empty means use the host's software. |
| `Conda` | `Channels` | STRING | `{{Param.CondaChannels}}` | Space-separated channels. |
| `Rez` | `Packages` | STRING | `{{Param.RezPackages}}` | Space-separated Rez package requests. |
| `Docker` | `Image` | STRING | `{{Param.DockerImage}}` | The container image. |
| `Docker` | `Mounts` | LIST[PATH] | `[]` | Host paths to make visible at the same path in the container. |
| `Docker` | `Gpus` | STRING | `""` | The value for `docker run --gpus`. |

When this RFC is accepted, the table can become a wiki conventions page that Environment Template and Job Template authors refer to.

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
| `ENV_PARAMS` listed by any document that uses Environment *envParameterDefinitions*, `activation`, `hostRequirements`, or `enableIf`, or `environment:` or `step:` entries | Template validation |
| A document that lists `ENV_PARAMS` also lists `EXPR` | Template validation |
| Environment *envParameterDefinitions*: 1–50 entries, unique names, not on *stepEnvironments*, permitted types and fields | Template validation |
| `activation` is `ALWAYS` or `ON_REQUEST`, and not on *stepEnvironments* | Template validation |
| Environment *hostRequirements*: §3.3 shape and constraints, not on *stepEnvironments*, `Env.*` references only to the enclosing Environment's parameters, and `Env.Param.*` only to its non-PATH ones | Template validation |
| Environment *enableIf*: an expression field of type `bool` (§7.5); for a Job Environment or Environment Template, references only the scopes of its *hostRequirements*; for a Step Environment, only `RawParam.*`, non-PATH `Param.*`, `Job.Name`, `Job.EnvNames`, `Step.Name`, and the Step's `let` (§4 *enableIf* constraints 1–4) | Template validation |
| A check over a Step's Environment set, such as the `WRAP_ACTIONS` single-layer rule, where an Environment's *enableIf* decides membership | Job creation; before then only when the violation holds whatever every undetermined *enableIf* resolves to (§7.5 rule 5) |
| Environment *enableIf* does not resolve to `null`, for each Step whose Sessions would enter the Environment | Job creation, per Step |
| An Environment's *hostRequirements*, resolved for a Step, satisfy §3.3.1 and §3.3.2 (a `min` that is a non-negative number, no two names in one source equal after resolution, and so on) | Job creation, per Step |
| Combined host requirements are satisfiable: `min` ≤ `max` for each amount, and for `attr.worker.os.family` and `attr.worker.cpu.arch` a value in every list (§3.3.3 items 1, 2.1) | Job creation |
| `default` references only `Param.*` / `RawParam.*` of the same document's Job Parameters | Template validation |
| A literal `default` satisfies its parameter's type and constraints, whether or not any Step uses it | Template validation |
| `Env.Param.*` / `Env.RawParam.*` name a parameter of the enclosing Environment, and appear only inside it | Template validation |
| An entry's *parameters* value or *enableIf* references only `RawParam.*`, non-PATH `Param.*`, `Job.Name`, `Job.EnvNames`, `Step.Name`, and the Step's `let`; never `Task.*`, `Session.*`, or `Env.*` (§3.2) | Template validation |
| An Environment's *hostRequirements* reference only its document's `RawParam.*` of any type and non-PATH `Param.*`, `Job.Name` and `Job.EnvNames`, and its own `Env.RawParam.*` of any type and non-PATH `Env.Param.*` (§4 *hostRequirements* item 1) | Template validation |
| `Job.EnvNames` appears only inside a `{{ }}` expression or an expression field, and not in an Environment parameter `default` | Template validation |
| An expression field (§7.5): parses as one expression, with no `{{ }}`; its static type fits its target type (§1.2.3 step 1), or is that made optional, with no implicit conversion other than widening `int` to `float`; a boolean, integer, or float scalar that fits is a literal | Template validation; a `null` result at evaluation |
| `<StepDependency>` has exactly one of *dependsOn* / *step* / *environment*; *required*, *enableIf*, and *parameters* only with *environment* (constraint 1) | Template validation |
| A *step* entry obeys every rule of the *dependsOn* entry it is a synonym of; a *step* and a *dependsOn* entry naming one Step are duplicates (§3.2 item 1) | Template validation |
| An entry's *enableIf* is an expression field of type `bool` (constraint 9, §7.5) | Template validation |
| An entry's *enableIf* does not resolve to `null` | Job creation |
| *environment* does not name the Step's own *stepEnvironments* entry; no duplicates per list; 1–50 *parameters* (constraints 2–4) | Template validation |
| `required: false` not used on an entry naming a *jobEnvironments* entry (constraint 8) | Template validation |
| Keys and literal values of *parameters* for a *jobEnvironments* entry (constraints 5, 6) | Template validation |
| Every parameter without a `default` of an `ALWAYS` *jobEnvironments* entry supplied by every Step, and of an `ON_REQUEST` one by every Step that requests it, except Steps for which the Environment is disabled (constraint 7) | Job creation; template validation only when constraint 7 fails whatever every undetermined entry or Environment *enableIf* (§7.5 rule 5) resolves to |
| External name resolution: missing with `required: true`, or ambiguous (§3.2.1) | Job creation |
| Keys and values for an external Environment; resolved format-string values (constraints 5, 6). A key the resolved Environment does not declare is an error even with `required: false` (§3.2.1 item 1) | Job creation |
| A format-string value or `default` for a list-typed parameter is a single `{{ }}` expression (§3.2, §4.5) | Template validation for `default`s and *jobEnvironments* entries; job creation for external Environments |
| A resolved, non-literal `default` satisfies its parameter's type and constraints, for each Step that takes it (§4.5) | Job creation |
| An Environment disabled for a Step (its *enableIf* is `false` with the values that are known) is not held to constraint 7, and the Step's effective values for it are not checked (§3.2.2) | Template validation for a literal `false`; job creation otherwise |
| An Environment *enableIf* whose result depends on a missing, unresolvable, or unconvertible value is an error (§3.2.2) | The earliest stage at which it is known: template validation for a value known then to be missing, whatever every value still unknown resolves to; job creation otherwise |
| PATH and LIST[PATH] values absolute in the submission path format, or URIs; their string constraints, against the unmapped value. This includes a literal `default` that no Step uses (§4.5) | Job creation |
| Required parameters of attached `ALWAYS` Environments supplied by every Step of the combined Job, and of attached `ON_REQUEST` Environments by every Step whose entry resolves to one, except Steps for which the Environment is disabled (constraint 7, §3.2.1) | Job creation |

A Session runs Tasks of a Step only when it has entered the Environments the Step's Sessions enter and the Step's
effective values equal the values the Session entered each of them with (§3.2.2). That is a scheduling rule rather than
a validation rule, and has no row here.

## Design Choice Rationale

### Parameters declared on the Environment

The Environment's author decides what the Environment accepts, so the parameters are declared on the Environment. That
gives each one a type, constraints, and a default where it is used, and lets a template validator check every
`Env.Param.*` reference and every supplied value against a definition, as it checks `Param.*` against Job Parameters.
The definitions reuse the `<JobParameterDefinition>` shapes, so an author who can write a Job Parameter can write an
Environment parameter. Every field of those shapes is permitted, including `userInterface`, `objectType`, and
`dataFlow`, so an Environment parameter definition is exactly a `<JobParameterDefinition>` of its type. Those three
have no effect on Job creation or execution here, since a Step supplies the value and the Job's Job Parameters declare
its data flow, but a tool such as a Job Template editor can use them to present Environment parameters as it presents
Job Parameters.

The key is `envParameterDefinitions` rather than `parameterDefinitions`. Everywhere in the
specification the bare name declares Job Parameters, read as `Param.*`, and the other kind of
parameter already carries its scope in the key: `taskParameterDefinitions` declares what
`Task.Param.*` reads. `envParameterDefinitions` declares what `Env.Param.*` reads. It also keeps an
Environment Template from having one key at two levels with two meanings, one merged with the Job's
Job Parameters and one never merged.

### A `default` that refers to a Job Parameter

External Environments already take their configuration from Job Parameters, and existing Job Templates
set those parameters. Letting `default: "{{Param.CondaPackages}}"` read the Job Parameter keeps every
such Job Template working with no change: a Step that says nothing gets the job-wide value, which is
today's behavior. It also gives the submitter a way to override the job-wide value at submission,
through the Job Parameter, while Steps that know better override it per Step. Allowing only
`Param.*` and `RawParam.*` of the same document keeps a default resolvable at job creation and
checkable without knowledge of other documents; §1.2.1 merging then decides the Job Parameter's
value, so the Job Template's own `default` for `CondaPackages` flows into the external Environment as it
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
rejects the submission rather than run it without the Environment. For an `ON_REQUEST`
Environment, the entry also causes the Step's Sessions to enter it, as a `service:` entry causes its
Service to run.

### `step:` as a synonym of `dependsOn:`

With `environment:` (and `service:` under RFC 0009), every other form of `<StepDependency>` is keyed by the kind of thing
it names, and `dependsOn:` is the one keyed by the relationship instead. Inside a list named `dependencies`, "depends on"
says nothing the list does not already say, and it leaves the form for a Step as the odd one out:

```yaml
dependencies:
  - step: DecodeFrames
  - environment: Conda
    parameters: {Packages: "colmap=3.11"}
```

Renaming the key outright would break every existing Job Template, so this RFC adds `step:` as a synonym and keeps
`dependsOn:` with its meaning unchanged. The two are interchangeable rather than distinct, so no rule has to say which
one wins, and an implementation can normalize one to the other at parse time. `dependsOn:` is planned for removal in
the next major version of the specification, when a breaking change is allowed anyway; until then both are valid, and
templates that already list `ENV_PARAMS` can adopt the new form.

`step:` is gated by `ENV_PARAMS` because this is the RFC that adds it, not because it depends on Environment
parameters. It could be its own extension, but a separate name for a one-key synonym would cost more in extension
bookkeeping than it buys; a document that wants it is almost always one that also uses `environment:` entries.

### `required` defaults to `true`

Listing something in `dependencies` declares a dependency on it, and the safe reading of an
undeclared choice is that the Step needs what it named. A Job Template that targets several kinds of
provisioning Environment writes `required: false` on each alternative, which states in the template that the Job runs
without them. The opposite default would let a typo in an Environment name silently do nothing.

### `activation` lives in this RFC

Earlier drafts of RFC 0009 defined `runScope` as a list of Session kinds, `TASK` and `SERVICE`, to
keep an Environment that configures a Service's clients out of the Service's own Session. Looking at
how it would be used, selection by kind was mostly a stand-in for selection by who asked: an
Environment "for Services only" is one that each Service wants and no Step does, and an Environment
"for Tasks only" is one the Steps want. The `environment:` entry gives every Step (and, if RFC 0009 adopts it, every
Service) a way to say that directly, and it is defined here. So this RFC owns the field, with two values that need
no Session-kind vocabulary, leaving RFC 0009 to state the one rule that is really about kinds (an
Environment that reads a Service's endpoint cannot run in that Service's Session) as a rule rather
than a default for a list.

Defining `activation` here also changes a rule from the first draft of this RFC. A required parameter on a Job
Environment had to be supplied by every Step, because every Session entered every Job Environment,
so a Step with no use for the Environment still had to configure it. With `ON_REQUEST`, the
Steps that must supply a required parameter are the Steps that request the Environment.

The field is named for what it decides, when a Session enters the Environment, rather than kept as
`runScope`: an Environment is entered, not run, and "scope" already names the value-reference scopes
(`Param.*`, `Env.Param.*`) throughout the specification. The two values answer the same question,
`ALWAYS` or `ON_REQUEST`, and `ON_REQUEST` names the mechanism: an `environment:` entry is the
request.

### Two values rather than a kind list

`ALWAYS` and `ON_REQUEST` are the two cases that need no further information: in every Session, or
in the Sessions of those who asked. A list of Session kinds would reintroduce a vocabulary that grows with every RFC that adds
a kind of Session, and its interesting combinations (`[TASK]` for Steps only, `[SERVICE]` for
Services only) are each one `ON_REQUEST` Environment that the right entities request. Selection
on anything finer, such as which Steps have GPUs, is a predicate over the Step's values, which is
what the Environment's *enableIf* with `Env.Param.*` expresses (see
[`enableIf` on the Environment](#enableif-on-the-environment)). `ON_REQUEST` is the part of that which needs no expression, and the part an Environment Template author reaches for first.

### `ALWAYS` is the default

Every Job Environment and attached Environment is entered by every Session today, so a template that
says nothing must keep that behavior. It is also what an auditing, logging, or credential
Environment wants: a Step cannot opt out of it by not asking.

### `enableIf` on the Environment

An entry's *enableIf* lets the Step's author say when an Environment is wanted. The Environment's *enableIf* lets its
own author say when it applies, and with *envParameterDefinitions* that is often a question about the values a Step
supplied: a Conda Environment has nothing to do for a Step with no packages, and a GPU-setup Environment nothing for a
Step that asked for no GPUs. `activation: ON_REQUEST` answers the coarser question of whether the Step asked at all;
*enableIf* answers it per Step from the effective values, and keeps an `ALWAYS` Environment that is the default for
existing Job Templates out of the Sessions that would only enter it to do nothing.

Every input it reads is fixed when the Job is created, so it is decided then, once per Step, and every other rule of
this RFC (Session sharing, combined host requirements) sees a fixed Environment set. Its scopes are ones this RFC
already defines: for a Job Environment or an Environment Template, those of the Environment's *hostRequirements*,
which is the other field resolved per Step from the Environment's own values; for a Step Environment, those the Step's
other fields have at Job creation. An Environment Template may use it, unlike a field that reads the Job Template's
parameters, because it reads only the Environment Template's own Job Parameters and its own Environment parameters.

A disabled Environment is not held to constraint 7, and its values are not checked. A Step that never enters an
Environment has no reason to configure it, and a value the Environment never receives cannot break it; two Environments
that are never enabled for the same Step may each require a parameter that only their own Steps supply. An *enableIf*
may read the very parameters those rules are about, so it is decided with the values that are known, and one whose
result depends on a missing, unresolvable, or unconvertible value is an error, so the mistake is reported rather than
hidden. It is reported as soon as it is known, as a reference to an undefined name is: a parameter without a `default`
that a Step's entries do not supply is known to be missing for that Step at template validation. A value that converts
but breaks another constraint is known, so an *enableIf* that reads it decides with it, and the value is checked only if
the Environment is not disabled. `Job.EnvNames` keeps listing disabled Environments because an *enableIf* may read
effective values computed from it.

RFC 0011 (`CONTROL_FLOW`) also proposes *enableIf* on `<Environment>`, over `Param.*`. Nothing in that form needs
step input binding, which is what `CONTROL_FLOW` adds, so this RFC defines it, with the scopes above. A Step
Environment whose *enableIf* reads the outputs of other Steps does need step input binding; that is an extension for
`CONTROL_FLOW` to make ([Extensions specified after this one](#extensions-specified-after-this-one-non-normative)).

### The name `enableIf`

The most common name for this kind of field elsewhere is `when:`. Argo Workflows, Tekton, CWL, Nextflow, Ansible,
CircleCI, and Jenkins all use it to mean "run this unit only if the condition holds". GitHub Actions, Buildkite, and
GitLab CI use `if:`, Azure Pipelines uses `condition:`, and Helm charts conventionally use an `enabled:` value. This RFC
uses `enableIf`, the name discussion #81 proposed and RFC 0011 uses for Steps, for three reasons:

1. **It says what `false` does.** A disabled Environment is not entered and a disabled entry is as if absent; nothing
   waits. In OpenJD, Steps do wait, on their dependencies and, under RFC 0011, for step input binding, so
   `when: "Steps.QualityCheck.Output.Noisy"` invites the reading "wait until this is true". The systems that use
   `when:` accept that ambiguity; `enableIf` avoids it.
2. **It reads on any mapping.** `if:` works on a CI job, which plainly either runs or does not, but on a `dependencies`
   entry or an `<Environment>` it does not say what the condition controls. `enabled:` reads well with a literal but
   suggests a fixed setting, and a literal is the rare case for this field.
3. **It is one name across the specification.** The same field on Steps (RFC 0011), Environments, and
   `environment:` entries gives authors one word for one idea.

The cost is unfamiliarity to authors who know `when:` or `if:`, and a literal `enableIf: false` that reads less
naturally than `enabled: false`.

### Expression fields rather than format strings

Both *enableIf* fields hold one boolean expression and nothing else. Written as a Format String, that is
`"{{Env.Param.Packages != ''}}"`: the braces say "interpolate here" in a field where there is nothing to
interpolate into, the form admits values that are errors only because of the braces (`"x{{true}}"`), and the field
inherits the Format String's implicit conversions, under which `"{{Param.Mode}}"` for a STRING `Mode` is accepted and
`"{{'yes'}}"` becomes `true`. An expression field (§7.5) is `"Env.Param.Packages != ''"`: the value is the expression, its
type must fit the field's, and nothing is converted except that an `int` widens to `float`; a field that accepts
unrelated types says so, as in `@expr int | string`. `let` already reads expressions this way,
since a binding is not a Format String either.

The annotation carries the type (`@expr bool`) because the type is what a reader and a validator need to know about
the field, in the place they look for it. Existing fields that take a single-expression Format String, such as
`timeout`, stay as they are: changing their kind would break accepted templates, and some of them, like entry
*parameters* values, legitimately interpolate text. New fields whose value is one typed expression should be
expression fields, and [Extensions specified after this one](#extensions-specified-after-this-one-non-normative) asks
RFC 0011 to use them for its loop fields and a Step's *enableIf*.

Expression fields do need YAML quoting: an expression that begins with a quote or a bracket must be enclosed in quotes, as in
`"'Docker' not in Job.EnvNames"`. That is no worse than the Format String form, which must always be quoted because
`{{` begins a YAML flow mapping.

### An Environment without the supplied parameter is an error

`required: false` makes an absent Environment acceptable. It does not make an entry that supplies
a parameter the resolved Environment lacks acceptable, and this RFC rejects the submission in that
case. The case arises in practice during migration: an external `Conda` Environment written before
this extension has the recommended name but no `Packages` parameter.

The alternative is to ignore such an entry, perhaps with a warning. With that Environment the Job would
then run every Step's Sessions with the job-wide `CondaPackages`, which a Job Template written for
per-Step values usually leaves empty, and each Step would fail inside its Tasks for want of its
software, after Worker Hosts had been provisioned and Sessions started. Rejecting the submission
fails before anything runs, with an error naming the entry and the parameter, and the fix belongs
to the Environment Template's author, who adds the parameters with defaults that keep existing Job Templates
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
2. The motivating use is `required: false` entries for Environments that a given scheduler may not attach
   at all. A declaration would be a second place to write the same name, add no information, and
   still could not be checked without the Environment Templates.
3. Every mistake an implicit name allows in a `required: true` entry is caught at job creation, with
   an error that names the entry: a misspelled name, an unknown key, or a value of the wrong type. A
   `required: false` entry is different: a misspelled name resolves to nothing wherever the Job runs and the
   entry is silently ignored, which is the cost of letting the same Job Template run where the
   Environment is absent. A validator given the Environment Templates to attach should warn when a
   `required: false` entry resolves to no Environment, so that the mistake is visible before
   submission.

What a template validator does with an external entry is specified in
[`<StepDependency>`](#32-stepdependency): it checks constraints 1–4, 8, and 9 and the form of literal values.
A validator given the Environment Templates the scheduler will attach, as `openjd run` already accepts
them, performs the job-creation checks before submission, which is what the Tooling parseable tenet
asks for.

### Values fixed per Step

An Environment, once a Session enters it, serves every Task the Session runs until it exits. Values that varied
per Task would force one Session per Task, which defeats the Environment's purpose. Values fixed per
Step are known at job creation, so tooling can display them, a scheduler can group Sessions by them,
and two Steps with equal values can still share Sessions as today.

### Sessions that serve several Steps

How Jobs Are Run already lets a Session run Tasks of several Steps whose Environments are identical.
This RFC keeps that rule, which `activation` and an Environment's *enableIf* give new work to do, since two Steps
that request different `ON_REQUEST` Environments, or for which an Environment's *enableIf* differs, have different
Environment sets, and adds one condition: the
Steps' effective values must be equal to the values the Session entered each Environment with. The rule is stated as a condition on adding a Step's Tasks to a
Session, not as a property of a set of Steps fixed when the Session starts, for two reasons. A scheduler
that today adds Tasks of a newly ready Step to a running Session can keep doing so, with one more
comparison. And the condition is all a Session needs: its Environments' values were fixed at `onEnter`,
and every Task it runs belongs to a Step whose values match them, so `Env.Param.*` has one meaning for
as long as the Environment is entered. A Session that moves on to a Step with other values can exit and
re-enter the Environment rather than end.

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
either, but not which it prefers when both are attached. The template needs to see what the Job
has. A list of names answers that, and EXPR already has `in` to test it and
list comprehensions to filter it, so no new function is needed. The name follows RFC 0009's
`WrappedService.PortNames`, a `list[string]` of declared names in declaration order; uses `Env` as the
existing `Env.File.*` and `Env.Param.*` scopes do; and is in the `Job.*` scope because the list is a
property of the Job, not of any Step or Session. It is concrete at job
creation, when the attached Environments are known, which is the same stage as `Job.Name`, so it adds
no new stage and no new state. A set of per-Environment booleans (`Job.HasEnvironment.Conda`) would
need a name per Environment and still could not be enumerated; a list can do both.

The list names every Job Environment whatever its `activation`, including `ON_REQUEST` ones the Step
does not request, because its purpose is to show what is available to request. It excludes Step
Environments, which no Step can request and which the Step that owns them already knows.

### `enableIf` on the entry

`Job.EnvNames` tells a Step which external Environments are attached; `enableIf` on the entry is how the Step
acts on it. Without it, a Step that prefers Docker when both are attached could only request `Conda`
with an empty value, which enters an Environment to do nothing and, for an `ON_REQUEST`
Environment, defeats the point of opting in. A disabled entry is the same as no entry, so a
reader can tell what it does without consulting the Environment: nothing is requested,
nothing is supplied, and an `ALWAYS` Environment behaves as it does for any Step without an entry.

The field takes the name this RFC uses on `<Environment>` (and RFC 0011 on Steps) for the same idea, and the same
type, `bool`, with `null` an error; here it is an expression field (§7.5), as the Environment's is. It is evaluated at job
creation with the scopes a *parameters* value has, so a disabled entry costs nothing at run time and
the Environment set of every Step is known when the Job is created, as `activation` already requires.
It is evaluated before the entry's name is resolved, so a `required: true` entry for an Environment
that is not attached is not an error when its condition is `false`; that lets one entry say "I need `Conda`
unless `Docker` is here".

`required: false` and `enableIf` answer different questions. `required` is about what the scheduler
attaches, which the template cannot know; `enableIf` is about what the template wants, which it
states. A template that only wants to tolerate a missing Environment writes `required: false`; one
that wants to choose among present ones writes `enableIf`.

### `hostRequirements` on the Environment

An Environment that provisions software knows what it needs from a host: a container runtime, a
Linux kernel, a GPU when asked for one. Today only a Step can state host requirements, so an
Environment Template that works on part of a fleet has no way to say so, and the Steps it serves fail
at `onEnter` on the rest. With `hostRequirements` on the Environment, its author can state them.

The parameters are why this belongs in this RFC. A static requirement could be its own small RFC, but
the requirement a provisioning Environment most wants to state depends on what the Step asked for:
`amount.worker.gpu` matters only when `Gpus` is set. This RFC already resolves effective values per
Step at job creation, which is the stage at which a Step's own `hostRequirements` are resolved, so
resolving an Environment's requirements with those values adds no new stage and gives a per-Step
result.

The requirements combine by conjunction. The Step's author and the Environment's author each state what they need; a
host must satisfy both, and neither can loosen the other's. The normalization rules restate that conjunction exactly,
so that tooling can show it and a conflict can be detected before anything runs, and they are defined over the whole
set of requirements on a capability, so the result cannot depend on the order in which sources are combined. Amounts
take the greatest `min` and the least `max`. For attributes the rule follows from what a host has. A host has a set of
values of an attribute, so `anyOf [a]` and `anyOf [b]` are both satisfied by a host with `a` and `b`; intersecting
them would reject Jobs that some host can run. Only for the two standard attributes a host has exactly one value of,
`attr.worker.os.family` and `attr.worker.cpu.arch`, is the conjunction an intersection, and only there can attribute
requirements conflict. A conflict is rejected at job creation rather than left to never schedule, because it is
static, and the error can name the authors who disagree.

The requirements follow entering. An `ON_REQUEST` Environment constrains only the Steps that
request it, so on a fleet with some container-capable hosts, only the Steps that asked for a
container are placed on them, and a `required: false` entry that resolves to nothing adds nothing. Session
sharing is unaffected: a host either satisfies the combined requirements of the Steps a Session
serves or the Session is never placed there, as today.

### Values resolved at job creation

`parameters` values and defaults see only the Job Parameters (and the Step's `let` bindings and names, and
`Job.EnvNames`), all known at job creation. That makes the effective values a static property of the Job, which is
what lets a scheduler compare them to group Sessions and cache environments. Values that depended on `Session.*` or
host state would be different on every host, and the grouping would be meaningless. Path mapping is the one host-side
step, and it applies to `PATH` values as it does to `PATH` Job Parameters.

The same reason excludes `Param.<name>` of a PATH or LIST[PATH] Job Parameter from values and
defaults, although §7.3.1 allows it elsewhere in an Environment. `Param.<name>` of a path is the
host-mapped value, which does not exist until a Worker Host applies its path mapping rules. A value
or default reads the unmapped `RawParam.<name>`, and the Environment's own `Env.Param.<name>`
applies the mapping on the host.

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
is different: it is the Environment Template's job-wide fallback for Job Templates that say nothing. If every Step
that enters the Environment supplies its own value, the fallback is never used, and the Job Parameter
it reads may be empty or meaningless for this Job. Rejecting such a Job would force the submitter to
set a Job Parameter that changes nothing. The resolved `default` is therefore checked only for the
Steps that take it.

### Environment parameters are not merged with Job Parameters

§1.2.1 merges same-named Job Parameters because a Job has one value per Job Parameter name. An
Environment's parameters are a per-Environment namespace with one value per Step; merging an external
Conda Environment's `Packages` with a Job Environment's `Packages` would make two independent
Environments share an input they never agreed to share. The names are therefore scoped to the
Environment, and the only link to Job Parameters is the explicit one a `default` writes.

### Not on Step Environments

A Step Environment is entered only by its own Step's Sessions, and its Step writes its actions. The
Step can write any value directly in the Environment, or use a Step `let` binding, so a parameter
block there would add a second way to say the same thing. The same holds for `activation`, which could
only say `ON_REQUEST` by the one Step that owns it, and for `hostRequirements`, which the Step
already has.

### Every Job Parameter type is permitted

An Environment parameter may have any type a Job Parameter may, so an Environment Template author has one set of
types to learn. That includes `RANGE_EXPR`. A range expression is how a Step's parameter space is written, but it
is also a compact, validated set of integers that an Environment can use for its own purposes, such as the frames to
pre-fetch into a cache or the device indices to expose to a container, and it needs nothing new: its literal form is
the `<IntRangeExpr>` string, and `Env.Param.*` reads it as a `range_expr`.

The first draft excluded `LIST[PATH]`, whose Job Parameter defaults are relative paths joined
with the template directory (§2.12). Once Environment parameter paths are defined to be absolute in the
submission path format, that objection no longer applies, and each element is mapped as a PATH value
is, as for a `LIST[PATH]` Job Parameter. The Docker sample needs it: a container must mount
every host path the Step's Tasks read or write, after mapping, and a `LIST[STRING]` of `RawParam`
values would mount the submitter's paths instead of the Worker Host's.

### Required parameters are allowed

A parameter without a `default` makes the Environment useless unless each Step that enters it
configures it, which is the right contract for a Job Environment like `RenderLicense` whose author
wants every Step that uses it to say which product it uses. With `activation: ON_REQUEST`, only
the Steps that request the Environment are held to that. For an `ALWAYS` Environment in an Environment
Template the same rule would break every Job Template that does not know the parameter, so this RFC
states the consequence and leaves the choice to the Environment Template's author, who normally gives every
parameter a default or makes the Environment `ON_REQUEST`.

### `required: false` is an error on a Job Environment entry

A Job Environment is always present, so `required: false` on an entry that names one can only be a
mistake: either the author meant an external Environment and the name collides with a Job Environment,
or the author misunderstood the field. Making it a template validation error reports the first case,
which otherwise silently changes which Environment the entry refers to.

### `ENV_PARAMS` requires `EXPR`

Environment parameters are typed values that flow into expressions: an *enableIf* reads them, `Job.EnvNames` and a
Step's `let` bindings feed them, and LIST, BOOL, and RANGE_EXPR parameters exist only with `EXPR`. Without the
requirement the extension would have two dialects. A document without `EXPR` would get a subset of the parameter types,
no *enableIf* of either kind, no `Job.EnvNames`, and Format Strings of value references only, and every rule would
need a case for a requesting document and an Environment that disagree about `EXPR` (which types a Step may supply, and
which values an *enableIf* can know). Requiring `EXPR` removes those cases, so a Job Template and the Environment
Templates attached to it always read values the same way. RFC 0008 (`WRAP_ACTIONS`) makes the same requirement. The
cost is that a scheduler must implement `EXPR` to offer `ENV_PARAMS`, and a Job Template that adopts per-Step
configuration lists both; a Job Template that lists neither is unaffected.

## Security Considerations

### Trust model

Environment parameter values are written by the author of the Job Template (in `environment:`
entries) or come from Job Parameters (through a `default`), and the submitter already controls both.
They are executed by Environment scripts that the Environment Template's owner controls. That is the
same trust relationship as an external Environment that reads `{{Param.CondaPackages}}` today: this extension grants
the submitter no new privilege, it only lets the same submitter-controlled input differ per Step. An
Environment Template owner who does not want submitters to choose, for example, arbitrary container images
restricts the parameter with `allowedValues`, which a scheduler checks at job creation before any
Worker Host runs anything.

Environment Template authors must still treat `Env.Param.*` as untrusted input in their scripts, as
they should treat `Param.*` today: the values are chosen by whoever submits the Job, which on a shared
render farm is not the owner.

### Do not substitute values into script source

A format string substitutes text, and an embedded file's `data` is script source. A script line such as
`PACKAGES='{{Env.Param.Packages}}'` breaks when the value contains `'`, and a crafted value such as
`x'; curl ... | sh; '` runs arbitrary commands as the job user. Quoting in the template does not help,
since the value can contain any quote character.

Pass values to scripts in one of these ways instead:

* **As `args` of the action.** Each `args` element reaches the process as one argv entry, with no shell parsing. The
  samples and Basic Examples do this: `args: ["{{Env.File.Enter}}", "{{Env.Param.Packages}}"]`, and the script reads
  `"$1"`. A list-typed parameter such as the Docker sample's `Mounts` expands to one argv entry per element.
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
  pattern, with the attached external Environments playing the role of the enabled directive. Nextflow
  builds the provisioning mechanisms into the engine; OpenJD keeps them in Environment Templates the
  render farm's operator owns.
* **Snakemake** — Each `rule` can declare `conda:` (an environment file), `container:`, and
  `envmodules:`, and the command line selects which is used: since Snakemake 8 with
  `--software-deployment-method` (`--sdm conda`, `--sdm apptainer`, `--sdm env-modules`), and before
  it with `--use-conda`, `--use-singularity`, or `--use-envmodules`. Snakemake caches each distinct
  Conda environment by the hash of its definition, which is the reuse a scheduler can achieve by
  grouping Sessions by effective values.
* **Deadline Cloud sample queue environments** — Deadline Cloud's
  sample Conda queue environments, which are Environment Templates attached as external Environments, take `CondaPackages` and `CondaChannels` Job Parameters, and its
  Rez sample takes `RezPackages`, each applied to every Session of a Job. This RFC generalizes them: the
  same Environment Templates gain a per-Step value that defaults to those Job Parameters, which is
  the shape of this RFC's samples. Deadline Cloud's improved-caching Conda sample names each
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
  layer configured per unit of work, like an Environment Template that each Step
  configures.
* **Rez** — `rez env <packages> -- <command>` resolves and enters a software environment per
  invocation, and farm integrations run each job under its own request. The request is per job or per
  command line, not a declared input of a farm-owned environment.
* **Argo Workflows** — Every `template` in a workflow specifies its own `container` (image, command,
  resources), and `inputs.parameters` let a step pass values into a template. A parameterized template
  invoked from several steps with different `arguments` is the shape of this RFC's parameterized Job
  Environment. Argo has no notion of an externally supplied environment around a template, since every step
  is its own container.
* **Slurm and Environment Modules** — A batch script runs `module load colmap/3.11` per job step, or
  `srun --container-image=...` (with Pyxis/Enroot) per step. The selection lives in the script text, so
  no tool can see which software a step needs without parsing shell, which the
  [Tooling parseable](https://github.com/OpenJobDescription/openjd-specifications/wiki/Design-Tenets)
  tenet asks OpenJD to avoid.
* **GitHub Actions** — `jobs.<id>.container` and `runs-on` select the execution environment per job,
  and reusable workflows take typed `inputs` with defaults. A job's choice is written where the job is
  defined, as an `environment:` entry is written on the Step.
* **CWL** — `requirements` and `hints` on each `CommandLineTool` (`DockerRequirement`,
  `SoftwareRequirement` with package names and versions) state what a tool needs, and a hint is
  optional while a requirement is not. That distinction is `required: false` versus `required: true`.

In each of these systems the unit of work names its own software, and the platform decides how it is
provided. In OpenJD the provisioning layer is an Environment Template, which the render farm's
operator owns rather than the Job's author, and this RFC lets a Job Template configure that layer per
Step.

## Implementation Impact

- **Rust (openjd-rs)**: `openjd-model` gains the `envParameterDefinitions`, `activation`,
  `hostRequirements`, and `enableIf` fields on `Environment`, reusing the existing parameter-definition and
  host-requirement types with a validator that rejects the excluded fields and types; the `environment`
  variant of `StepDependency` with its `required`, `enableIf`, and `parameters`; the `Job.EnvNames`
  symbol; and scope rules for `Env.Param.*` and `Env.RawParam.*`. Job creation computes, per Step, the list of Environments its
  Sessions enter (filtering `ON_REQUEST` Environments by the Step's entries, then every Environment by
  its *enableIf* resolved with the Step's effective values) and the effective values for each, and stores both on the Job's Steps in place of the single resolved Environment
  list; the external resolution reuses the code that already merges Environment Template parameter
  definitions. The attached Environment Templates are determined, and `Job.EnvNames` made concrete,
  before any Step `let`, entry *enableIf*, or *parameters* value is evaluated; entries are then resolved
  against them. The `hostRequirements` type is reused on `Environment`, and job creation resolves each
  Environment's requirements per entering Step and combines them with the Step's by the §3.3.3 rules,
  storing the combined set on the Job's Step. Template-validation checks over a Step's Environment
  set, such as the `WRAP_ACTIONS` single-layer rule, move to job creation where an *enableIf* is an
  expression.
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

The effective values are a small map per Step and Environment, computed once at job creation. At run
time they are passed when a Session enters an Environment and compared when a scheduler shares a
Session.

## Rejected Ideas

### A `stepOverridable` flag on template-level Job Parameters

The discussion's first sketch marked a Job Parameter such as `CondaPackages` as overridable and let a
Step set it. Rejected for three reasons. First, a Job Parameter has one value per Job by definition,
and every format string that reads `Param.CondaPackages` (in a Step script, a name, a host
requirement) would then have a value that depends on where it is read, which breaks the meaning of
`Param.*` everywhere. Second, the Job Parameter is declared by whichever template wins §1.2.1 merging,
so the flag would have to be merged too, and a Job Template could make an Environment Template's parameter overridable
or not without that template's agreement. Third, it gives the Environment no say in what a Step may
change: every overridable parameter becomes Step-settable for every Environment that reads it. An
Environment parameter is declared by the Environment, read only by it, and defaults to the Job
Parameter, which gets the same compatibility without any of these effects.

### A flat per-Step map of values

A Step could carry `environmentParameters: { Packages: "colmap=3.11", Image: "..." }`, and each
Environment would read the names it knows. Rejected because names collide: a Conda Environment and a
Pip Environment can both reasonably call their input `Packages`, and a flat map would send the same
value to both. It also cannot express `required`, cannot be checked against a definition (which
Environment does `Image` belong to?), and gives a scheduler no way to tell which Environment a typo
was meant for. An `environment:` entry keys the values by Environment name, so values do not collide,
`required` has a place, and each value, typos included, is checked against the named Environment's
definition.

### Letting a Step's script read `Env.Param.*`

A Step might want to read the value it supplied, for example to log the Conda packages it asked for. Rejected because
the reference is ambiguous: two Environments wrapping the Step may declare the same parameter name, and `Env.*` inside
an Environment already means "this Environment". A Step that needs the value already has it, since it wrote it; it can
bind it once with a Step `let` and use the binding in both the `environment:` entry and its script.

### Values per Task

Letting Task parameters feed Environment parameters would require a Session per Task, so each
Environment would be entered once per Task rather than once for many. Tasks that need different
software belong in different Steps.

### `required` defaulting to `false`

With an optional default, the common case of a Job Template written for the Environments of its own
`jobEnvironments` would carry no difference, but a misspelled external name would be silently
ignored, and a Job written for a `Conda` Environment would run without one and fail inside its Tasks
instead of at submission.

### Matching external Environments by something other than name

An Environment Template could carry a separate "kind" (`conda`, `docker`) that entries match on, so a
Job Template could say "any Conda-like Environment". Rejected for now: Environment names are already
how external Environments are identified and ordered, and a kind registry would need a vocabulary the
specification does not have. The `required: false` pattern with one entry per expected name covers the
motivating cases, and `Job.EnvNames` lets a template choose among the names that are attached. A
future RFC can add a matching key if Environment Templates standardize on one.

### `activation` as a list of Session kinds

RFC 0009's earlier `runScope: [TASK, SERVICE]` selected Sessions by kind. See
[`activation` lives in this RFC](#activation-lives-in-this-rfc) and
[Two values rather than a kind list](#two-values-rather-than-a-kind-list). The one thing the kind
list could say that `ON_REQUEST` cannot is "every Step but no Service" without each Step
writing an entry. RFC 0009's own motivation is that Services should get the Tasks' provisioning, so
this RFC adds no shortcut for excluding them.

### `activation` on the entry instead of the Environment

A Step could write `environment: Conda, exclusive: true` to mean "enter this for me and not for
Steps that do not ask". Rejected because whether an Environment is opt-in is a property of the
Environment, decided by its author: an auditing Environment must not be something a Step can turn
off for everyone else, and two Steps could disagree.

### Environment parameters on Step Environments

See [Not on Step Environments](#not-on-step-environments). A Step Environment's own Step is the only
possible supplier, so the values would always be written by the same author in the same Step.

## Open Questions

None block acceptance. The following are noted for the review:

1. **Standard names for external Environments.** The portability of a Job Template that writes
   `environment: Conda` depends on Environment Templates naming their Conda Environment `Conda` and its input
   `Packages`. [Recommended names](#recommended-names-non-normative) proposes a table, which the
   samples follow. Whether it should become a wiki conventions page on acceptance, and which further
   Environments it should cover, is open.
2. **Interaction with `WRAP_ACTIONS` beyond the wrapper's own values.** A wrapping Environment reads
   its own parameters (the image to run) but not those of the Environments it wraps. No use case for
   the latter is known; `WrappedEnv.Param.*` is listed under [Future Work](#future-work).
3. **The extension name.** With `activation`, `ENV_PARAMS` covers more than parameters: it is the
   extension under which a Step requests an Environment. The name is kept for continuity with the
   discussion thread; a reviewer may prefer one that names the request.
4. **`enableIf` on `<Environment>` and RFC 0011.** This RFC defines *enableIf* on every `<Environment>`,
   decided at Job creation, which RFC 0011 (`CONTROL_FLOW`) also defines in its current draft. If this RFC
   is accepted first, RFC 0011 should drop its definition and extend this one with what needs its own
   stages, such as a Step Environment's *enableIf* reading `Steps.*`
   ([Extensions specified after this one](#extensions-specified-after-this-one-non-normative)).

## Future Work

* **`WrappedEnv.Param.*` for wrap hooks**, so a wrapping Environment can read the effective values of
  the inner Environment whose action it is wrapping, if a use case appears.
* **Matching external Environments by a standard kind**, as in
  [Rejected Ideas](#matching-external-environments-by-something-other-than-name).
* **Late-bound entries under `CONTROL_FLOW`**, letting an entry's `enableIf` and `parameters` read
  `Steps.*` so that, for example, the packages a Step installs depend on an upstream output. The
  Step's Environment set and effective values would then be fixed at step input binding rather than
  Job creation, which touches the scheduler's Session grouping and is left for RFC 0011's review (see
  [Extensions specified after this one](#extensions-specified-after-this-one-non-normative)).
* **A Job-level default for a whole `environment:` entry**, so a Job Template could set a value once
  for all its Steps without a Job Parameter. The pipeline sample shows the need: four Steps carry
  eight near-identical entries, and the Docker `Mounts` value and the Conda `enableIf` are the same in
  all four. Today a YAML anchor writes a repeated value once (the sample does this for both), a Job Parameter plus the
  Environment's `default` covers it for external Environments, and a `jobEnvironments` entry's own
  `default` covers it for Job Environments.
* **Literal `default` checks under expression-valued constraints.** A literal `default` is checked unconditionally
  (§4.5), because neither it nor its constraints can depend on anything. If constraints such as `allowedValues` or
  `maxLength` could be expressions, a literal `default` might be valid only where an *enableIf* holds, and it would be
  checked like any other `default`, only where it becomes an effective value.

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is
more permissive.
