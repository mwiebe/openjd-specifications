* Feature Name: Environment Parameters
* Author(s): Mark Wiebe <[mwiebe](https://github.com/mwiebe)>
* RFC Tracking Issue: (pending)
* Start Date: 2026-10-17
* Specification Version: 2023-09 extension ENV_PARAMS
* Accepted On: (pending)
* Depends On:
  * RFC 0002, Model Extensions (https://github.com/OpenJobDescription/openjd-specifications/issues/57)
* Related:
  * Discussion #198, the idea thread for this RFC (https://github.com/OpenJobDescription/openjd-specifications/discussions/198)
  * RFC 0005, Expression Language (`EXPR`), for typed values and list-typed parameters
  * RFC 0008, Environment Wrap Actions (`WRAP_ACTIONS`)
  * RFC 0009, Service (`SERVICE`), whose `service:` dependency form this RFC's `environment:` form follows

## Summary

This RFC lets a Step supply parameter values to the Environments that wrap it. An `<Environment>`
gains its own `parameterDefinitions` block, declaring the values a Step may supply, and reads them
as `{{Env.Param.<name>}}` in its actions, embedded files, and variables. A Step supplies values
through a new `environment:` entry in its `dependencies`, which names an Environment, says whether
the Step requires it (`required`, default `true`), and gives the values (`parameters`). The
Environment may be one of the Job Template's own `jobEnvironments` or one attached by the scheduler
from an Environment Template. Because a Session runs Tasks from a single Step, every Environment in
the Session is entered with one known set of values, and a scheduler can group Sessions by those
values. A multi-application Job can then install, or pull a container image for, only the software
each Step needs, and the same Job Template still runs on a queue that attaches no such Environment.

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
        args: ["{{Env.File.Enter}}"]
    embeddedFiles:
      - name: Enter
        type: TEXT
        data: |
          #!/usr/bin/env bash
          set -euo pipefail
          PACKAGES='{{Env.Param.Packages}}'
          if [ -z "$PACKAGES" ]; then
            echo "No conda packages requested; using the host's software."
            exit 0
          fi
          CHANNELS=()
          for c in {{Env.Param.Channels}}; do CHANNELS+=(-c "$c"); done
          conda create -y -p ./.conda-env "${CHANNELS[@]}" $PACKAGES
          echo "openjd_env: PATH=$PWD/.conda-env/bin:$PATH"
          echo "openjd_env: CONDA_PREFIX=$PWD/.conda-env"
```

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
[`samples/v2023-09`](../samples/v2023-09).

### One parameterized Job Environment serving several Steps

An `environment:` entry can name a Job Environment of the same template, so one Environment
definition replaces several near-identical Step Environments.

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [ENV_PARAMS]
name: "Plugin-Selected Renders"
parameterDefinitions:
  - name: SceneFile
    type: PATH
    objectType: FILE
    dataFlow: IN
jobEnvironments:
  - name: RenderLicense
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
```

`Product` has no default, so every Step must supply it: a Step without a `RenderLicense` entry is a
template validation error, because every Session enters every Job Environment.

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

Two Steps that supply identical values for every Environment, such as two Maya Steps, may share a
Session; two Steps that supply different values cannot.

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
   outright when the packages' dependencies conflict (COLMAP and Maya pin incompatible versions of
   shared libraries in practice).
2. **Build one image with everything.** The same conflict problem, moved into an image build that the
   Job author may not control, and an image that is large to pull on every host.
3. **Split the pipeline into several Jobs.** Each Job gets its own `CondaPackages` value, but the
   dependencies between Steps are lost, and the submitter must chain Jobs by hand.

A Step Environment cannot help, because it has no way to ask an Environment the queue attached to do
anything differently. The Job Template could stop relying on the queue and install software in its own
Step Environments, but then it hard-codes one provisioning mechanism, and the
[Design Tenets](https://github.com/OpenJobDescription/openjd-specifications/wiki/Design-Tenets) ask that
a Job Template run unmodified with a Conda, Rez, Docker, or Apptainer Environment supplying its software.

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

These patterns are common wherever a farm provisions software per Session: Conda- and Rez-based VFX
farms, containerized HPC and ML batch systems, and CI systems that run matrix jobs. The
[Prior Art](#prior-art) section shows that every comparable workflow system lets a unit of work choose
its own software environment.

### Why the values come from the Step

A Session runs Tasks from a single Step, so when an Environment's `onEnter` runs, the Step it serves is
known. The Step is therefore the natural place to say what that Environment should do. Values are fixed
per Step rather than per Task: a Session runs many Tasks, and an Environment is entered once for all of
them, which is the amortization Environments exist to provide.

### Backward compatibility

This RFC is additive and gated by the `ENV_PARAMS` extension name declared under RFC 0002.

- A document that declares `parameterDefinitions` on any `<Environment>`, or that uses the
  `environment:` form of `<StepDependency>`, MUST list `ENV_PARAMS` in its `extensions`. A scheduler
  that does not implement `ENV_PARAMS` rejects such a document under RFC 0002.
- `ENV_PARAMS` does not require `EXPR`. Without `EXPR`, Environment parameters are limited to the
  types available to Job Parameters without it (`STRING`, `INT`, `FLOAT`, `PATH`), as Job Parameters
  are.
- The extension applies per document, as RFC 0009 established for Environment Templates. A Job
  Template that does not list `ENV_PARAMS` runs with an Environment Template that does: every
  Environment parameter takes its default, which is how a queue administrator keeps existing Job
  Templates working while adding per-Step configuration. An Environment Template that does not list
  `ENV_PARAMS` runs with a Job Template that does: the Job Template's `environment:` entries that name
  it find no parameters to set, which is an error only if they supply any (see
  [`<StepDependency>`](#stepdependency) constraint 6).
- No existing field changes meaning. A Job with no `environment:` entries and no Environment
  `parameterDefinitions` behaves exactly as today, including how Sessions are shared between Steps.

## Specification

### Terminology

| Term | Definition |
|------|------------|
| **Environment parameter** | A value declared in an `<Environment>`'s `parameterDefinitions`, read inside that Environment as `Env.Param.<name>`. Distinct from a Job Parameter, which is declared in a template's top-level `parameterDefinitions` and read as `Param.<name>`. |
| **Supplying entity** | The Step (or, with `SERVICE`, the Service) whose `dependencies` contain an `environment:` entry. Its Sessions enter the named Environment with the values the entry supplies. |
| **Effective values** | For one supplying entity and one Environment its Sessions enter, the value of each of the Environment's parameters: the value the entity supplies if it supplies one, otherwise the parameter's resolved `default`. |
| **External Environment** | An Environment the scheduler attaches to a Job from an Environment Template, as opposed to one of the Job Template's own `jobEnvironments`. |

### Schema modifications

> Changes to [the template schema](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas).

#### Job Template

> A modification to [`1.1. Job Template`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#11-job-template)

The Job Template schema is unchanged. Extension list addition to item 3 (*extensions*): `ENV_PARAMS`
(this RFC).

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
  script: <EnvironmentScript> # @optional
  variables: <EnvironmentVariables> # @optional
```

New property:

* *parameterDefinitions* — The parameters that a Step whose Sessions enter this Environment may
  supply, through an `environment:` entry in its `dependencies` (see
  [`<StepDependency>`](#stepdependency)). Their values are available within this Environment as
  `Env.Param.<name>` (see [Value references](#value-references)). Constraints:
    1. Minimum number of elements: If provided, then this list must contain at least one element.
    2. Maximum number of elements: 50.
    3. No two definitions in this list may have the same value for the `name` property. A name may
       coincide with a Job Parameter name or with a parameter of another Environment; they do not
       conflict.
    4. Permitted on a Job Template's `jobEnvironments` entries and on an Environment Template's
       `environment`. Not permitted on a `stepEnvironments` entry: its Step can write the values it
       wants directly in the Environment's actions.

The format string scopes available within an Environment gain one entry:

> 5. `Env.Param.*` and `Env.RawParam.*` — The effective values of this Environment's parameters for
>    the Step whose Session is entering it. Available with the `ENV_PARAMS` extension.

#### `<EnvironmentParameterDefinition>`

> A new section, inserted after [`4.4. <EnvironmentVariables>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#44-environmentvariables)
> as `4.5`.

Available when using the `ENV_PARAMS` extension.

An `<EnvironmentParameterDefinition>` declares one value that a Step may supply to the Environment
that contains it. It has the shape of a [`<JobParameterDefinition>`](#2-jobparameterdefinition) of the
same `type`, with these differences:

1. *type* — Any type available to a `<JobParameterDefinition>` in the same document, except
   `LIST[PATH]` and `RANGE_EXPR`. As for Job Parameters, the types other than `STRING`, `INT`, `FLOAT`,
   and `PATH` require the `EXPR` extension.
2. *userInterface* — Not permitted. Environment parameters are set by templates, not by a person
   submitting a Job, so no submission UI presents them.
3. *objectType* and *dataFlow* (`PATH`) — Not permitted. Which files a Job reads and writes is declared
   by its Job Parameters; an Environment parameter that carries a path should take its default from
   such a Job Parameter.
4. *default* — Optional, and a Format String (`@fmtstring`) resolved at job creation. It may reference
   `RawParam.<name>` for any Job Parameter, and `Param.<name>` for any Job Parameter not of type `PATH`
   or `LIST[PATH]`, defined in the top-level `parameterDefinitions` of the same document; it may not
   reference any other scope. With the `EXPR` extension the field is an expression field: a
   `default` consisting of a single `{{ }}` expression is evaluated with the parameter's type as its
   target type (RFC 0005, *Expression Evaluation Types*), so `default: "{{Param.Packages}}"` with a
   `LIST[STRING]` Job Parameter yields a `list[string]`. Otherwise the resolved string is converted to
   the parameter's type as a submitted Job Parameter value of that type would be. A literal `default`
   is written as for a `<JobParameterDefinition>` of the same type.
5. *allowedValues*, *minLength*, *maxLength*, *minValue*, *maxValue*, and *item* — As for the
   `<JobParameterDefinition>` of the same type. They constrain every effective value, whether
   supplied by a Step or taken from the resolved `default`.

A parameter without a `default` is **required**: every supplying entity whose Sessions enter the
Environment must supply it (see [`<StepDependency>`](#stepdependency) constraint 8).

A `PATH` value, whether supplied or defaulted, must be an absolute path or, with `EXPR`, a URI, after
it is resolved at job creation; a relative path is an error. Environment parameters have no template
directory to be relative to. A Job Template that ships a file next to itself passes it through a
`PATH` Job Parameter, whose `default` is joined with the Job Template's directory as today, and
references that parameter's `RawParam.<name>`.

The value of an Environment parameter is referenced, within the Environment that defines it, as both:

1. `Env.Param.<name>` — the effective value; for type `PATH`, with path mapping rules applied; and
2. `Env.RawParam.<name>` — the effective value with no path mapping applied.

With the `EXPR` extension, `Env.Param.<name>` has the expression type of the corresponding Job
Parameter type (`string`, `int`, `float`, `bool`, `path`, `list[string]`, and so on), and
`Env.RawParam.<name>` has the same type except that it is `string` for `PATH`.
