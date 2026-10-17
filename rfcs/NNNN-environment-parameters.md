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
that contains it. It has the shape of a [`<JobParameterDefinition>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#2-jobparameterdefinition) of the
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

The `parameterDefinitions` property and the `Env.Param.*` scope were introduced in this RFC.

#### `<StepTemplate>`

> A modification to [`3. <StepTemplate>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#3-steptemplate)

The `<StepTemplate>` object is unchanged. One of its items changes under `ENV_PARAMS`:

* *dependencies* — The Steps this Step depends on and, with `ENV_PARAMS`, the Environments its
  Sessions enter with values it supplies; each entry is a [`<StepDependency>`](#stepdependency). The
  Step's Tasks may be scheduled only when every listed Step has completed successfully. An
  `environment:` entry does not delay scheduling; it states what the Step's Sessions enter.

#### `<StepDependency>`

> A modification to [`3.2. <StepDependency>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#32-stepdependency)

A `<StepDependency>` is one of:

```diff
  # <StepDependency> is one of:
  dependsOn: "<StepName>"
+ # or, with the SERVICE extension (RFC 0009):
+ service: "<ServiceName>" # @extension SERVICE
+ # or, with the ENV_PARAMS extension:
+ environment: "<EnvironmentName>" # @extension ENV_PARAMS
+ required: <bool> # @optional @extension ENV_PARAMS
+ parameters: <EnvironmentParameterValues> # @optional @extension ENV_PARAMS
```

`<EnvironmentParameterValues>` is a map from `<Identifier>` to `<EnvironmentParameterValue>`:

```yaml
<Identifier>: <EnvironmentParameterValue>, # @fmtstring
...
```

Where:

1. *dependsOn* — The name of a Step in the same Job Template, as today.
2. *environment* — The name of an Environment that the Sessions of the listing entity enter, and to
   which it supplies parameter values. The name resolves, in order:
    1. to the entry of the same Job Template's `jobEnvironments` with that `name`, if there is one;
    2. otherwise, to an **external Environment**: the `environment` of an Environment Template that
       the scheduler attaches to the submission, matched by `name` at job creation (see
       [Resolving external Environments](#resolving-external-environments)).

   An entry that names an Environment declares that the listing entity depends on it: its Sessions
   need that Environment, entered with these values.
3. *required* — Whether the Environment must be present. Default `true`. When `false` and the name
   resolves to no Environment at job creation, the entry has no effect. A `jobEnvironments` entry is
   always present, so for one the field has no effect.
4. *parameters* — The values the listing entity supplies, keyed by the names in the Environment's
   `parameterDefinitions`. A parameter the entry omits takes its `default`. If not provided, the entry
   supplies no values; it still declares the dependency.

`<EnvironmentParameterValue>` is one of:

* A literal of the parameter's type, written as a `default` of the corresponding
  `<JobParameterDefinition>` type is written: a string, an integer or `<intstring>`, a float or
  `<floatstring>`, and with `EXPR` a boolean or a list.
* A Format String resolved at job creation. Its available scopes are `RawParam.*` for every Job
  Parameter, `Param.*` for every Job Parameter not of type `PATH` or `LIST[PATH]`, and with `EXPR`,
  `Job.Name`, `Step.Name`, and the names bound by the Step's `let`. `Task.*`, `Session.*`, and
  `Env.*` are not available: a value is fixed for the whole Step. With `EXPR`, a value consisting of a
  single `{{ }}` expression is evaluated with the parameter's type as its target type, as for
  *default* in [`<EnvironmentParameterDefinition>`](#environmentparameterdefinition).

Constraints:

1. Exactly one of *dependsOn*, *service*, or *environment* MUST be present. *required* and
   *parameters* are permitted only with *environment*.
2. An *environment* entry is permitted in the `dependencies` of a `<StepTemplate>` and, with
   `SERVICE`, of a `<Service>` (see [Services](#services)). It is not permitted in the
   `dependencies` of an `<Environment>` (`SERVICE`).
3. *environment* MUST NOT name a `stepEnvironments` entry of the listing Step. A Step Environment
   cannot declare `parameterDefinitions`, and its Step writes the values it wants into it directly.
4. No two *environment* entries of one `dependencies` list may name the same Environment.
5. *parameters*, if provided, has at least one entry and at most 50.
6. Every key of *parameters* MUST name a parameter declared in the named Environment's
   `parameterDefinitions`. An Environment that declares none, including one from a document that does
   not list `ENV_PARAMS`, accepts an entry with no *parameters* and rejects one with any.
7. Every value MUST be valid for the parameter it supplies: of its type, and within its
   `allowedValues`, `minLength`, `maxLength`, `minValue`, `maxValue`, and `item` constraints. A
   literal value is checked at template validation when the Environment is a `jobEnvironments` entry;
   every value is checked once resolved at job creation.
8. For every Environment that a supplying entity's Sessions enter, the entity MUST supply every
   parameter of that Environment that has no `default`. For a `jobEnvironments` entry, which every
   Task Session enters, this means every Step lists it with such a value; a template that fails this
   is rejected at template validation, naming the Step and the parameter.

Constraints 6 and 7 are checked at template validation for an entry that names a `jobEnvironments`
entry, and at job creation for an entry that resolves to an external Environment.

##### Resolving external Environments

When a Job is created from a Job Template and one or more Environment Templates, the scheduler
resolves each *environment* entry that names no `jobEnvironments` entry against the attached
Environment Templates:

1. If exactly one attached Environment Template's `environment` has the entry's name, the entry
   refers to it, and constraints 6 and 7 are checked against its `parameterDefinitions`.
2. If none does, then with `required: true` the submission MUST be rejected, naming the entry and
   the missing Environment; with `required: false` the entry is ignored.
3. If two or more do, the submission MUST be rejected, naming the entry and the ambiguous name.
4. An attached Environment with the same name as a `jobEnvironments` entry is not reachable by name
   from the Job Template; a scheduler SHOULD warn when it sees one.

After resolution, constraint 8 is checked over the combined Job: every Step, and with `SERVICE`
every Service, whose Sessions enter an attached Environment with a parameter that has no `default`
MUST supply it, or the submission is rejected. A queue administrator who wants existing Job
Templates to keep working gives every Environment parameter a `default`.

##### Services

> Applies only when the document also uses the `SERVICE` extension (RFC 0009).

A `<Service>`'s `dependencies` may contain *environment* entries, with the same meaning as on a Step:
the Service Session enters the named Environment with the values the entry supplies. Constraint 8
applies to each Job Environment whose `runScope` includes `SERVICE`, and to each attached Environment
a Service Session enters. A Service Session enters only Job Environments, so an entry that names an
Environment whose `runScope` excludes `SERVICE` is a template validation error.

#### Effective values

For each supplying entity and each Environment its Sessions enter, the effective value of every
parameter of that Environment is determined at job creation:

1. If an *environment* entry of the entity names the Environment and supplies the parameter, the
   resolved value of that entry.
2. Otherwise, the resolved `default` of the parameter.

An Environment whose parameter has no effective value for an entity is the error of constraint 8.
The effective values are part of the Job: a scheduler may display them, and tooling can compute them
from the templates and the Job Parameter values alone. Path mapping is applied to `PATH`-typed
effective values on the Worker Host, when `Env.Param.<name>` is read, exactly as for `PATH` Job
Parameters.

#### Value references

> A modification to [`7.3.1. Value References`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#731-value-references)

```diff
  |`Env.File.<name>`|...|Available within the Environment Script Actions and Embedded Files.|
+ |`Env.Param.<ParamName>`|The effective value of the Environment parameter `<ParamName>` for the Step (or Service) whose Session is entering the Environment. The same as `Env.RawParam.<ParamName>` except for type PATH, where path mapping rules are applied. Requires the `ENV_PARAMS` extension.|Available within the Environment that declares the parameter: its `variables`, and its Script's `let`, Actions, and Embedded Files. Not available in any other Environment, in a Step, or in a Service.|
+ |`Env.RawParam.<ParamName>`|The effective value of the Environment parameter `<ParamName>`, with no path mapping applied. Requires the `ENV_PARAMS` extension.|As for `Env.Param.<ParamName>`.|
```

A reference to `Env.Param.<name>` or `Env.RawParam.<name>` that does not name a parameter of the
enclosing Environment is a template validation error. A wrapping Environment's wrap hooks (RFC 0008)
read the wrapping Environment's own parameters through `Env.Param.*`; an inner Environment's
parameters are not visible to them.

#### Template processing stages

> A modification to [`7.4. Template Processing Stages`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#74-template-processing-stages)

* **Template validation** — `Env.Param.*` and `Env.RawParam.*` are unknown. With `EXPR` they are
  type-checked using the declared types of the Environment's `parameterDefinitions`. Literal
  *parameters* values for `jobEnvironments` entries are checked against their definitions.
* **Job creation** — Environment parameter `default`s and *parameters* values are resolved, external
  Environments are resolved, and the effective values of every supplying entity are fixed and checked.
* **Task execution** — `Env.Param.*` is available in the Environment's host-evaluated format strings,
  with path mapping applied to `PATH` values.

### Modifications to How Jobs Are Run

> Modifications to [How Jobs Are Run](https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run)

```diff
  Once the Environments have been created and entered for a Session, a series of Tasks are
  run within that Session and the Environments. Tasks from any Step in a Job can run within
- the same Session provided that the set of Environments that are required to run those
- Tasks are identical.
+ the same Session provided that the set of Environments that are required to run those
+ Tasks are identical and, with the ENV_PARAMS extension, that the effective values of
+ every Environment's parameters are identical for those Steps.
```

And a new subsection, *Environment Parameters*:

> An Environment may declare parameters that the Steps whose Sessions enter it supply values for
> (ENV_PARAMS). The values are resolved once, when the Job is created: for each Step and each
> Environment its Sessions enter, each parameter takes the value the Step supplies in an
> `environment:` entry of its `dependencies`, or else the parameter's default. A Session's
> Environment parameter values are therefore fixed by the Step it serves, for the whole Session:
> every Task the Session runs, and the Environment's `onEnter` and `onExit`, see the same values.
>
> Because the values are fixed per Step, a scheduler can group Sessions by them. Two Steps whose
> effective values are identical for every Environment may share Sessions, and a scheduler may also
> use the values to prefer a Worker Host that has recently entered an Environment with the same
> values, for example one that has a matching Conda environment or container image cached. Two Steps
> whose values differ for any Environment never share a Session.
>
> An `environment:` entry with `required: false` names an Environment the Job can run without. If
> the scheduler attaches no Environment of that name, the entry has no effect, and the Step's
> Sessions run without it.

### Validation

Every rule this extension adds, by the stage that checks it.

| Rule | Checked at |
|------|-----------|
| `ENV_PARAMS` listed by any document that uses Environment `parameterDefinitions` or `environment:` entries | Template validation |
| Environment `parameterDefinitions`: 1–50 entries, unique names, not on `stepEnvironments`, permitted types and fields | Template validation |
| `default` references only `Param.*` / `RawParam.*` of the same document's Job Parameters | Template validation |
| `Env.Param.*` / `Env.RawParam.*` name a parameter of the enclosing Environment, and appear only inside it | Template validation |
| `<StepDependency>` has exactly one of `dependsOn` / `service` / `environment`; `required` and `parameters` only with `environment` | Template validation |
| `environment:` not on an Environment's `dependencies`; does not name the Step's own `stepEnvironments` entry; no duplicates per list | Template validation |
| Keys and literal values of `parameters` for a `jobEnvironments` entry (constraints 6, 7) | Template validation |
| Every `jobEnvironments` parameter without a `default` supplied by every Step (and Service entering it) | Template validation |
| External name resolution: missing with `required: true`, or ambiguous | Job creation |
| Keys and values for an external Environment; resolved format-string values; resolved defaults; relative `PATH` values | Job creation |
| Required parameters of attached Environments supplied over the combined Job | Job creation |

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
rejects the submission rather than run it on a queue that lacks it.

### `required` defaults to `true`

Listing something in `dependencies` declares a dependency on it, and the safe reading of an
undeclared choice is that the Step needs what it named. A Job Template that targets several kinds of
queue writes `required: false` on each alternative, which states in the template that the Job runs
without them. The opposite default would let a typo in an Environment name silently do nothing.

### Names resolve to the Job Template first

An `environment:` entry is checkable at template validation whenever it names one of the template's
own `jobEnvironments`, which is the case a template author controls completely. Resolving the
template's own names first means an attached Environment can never change what such an entry refers
to. An external name cannot be checked until job creation, since the template cannot know what a queue
attaches; resolving it then, and rejecting ambiguity rather than guessing, matches how RFC 0009 matches
`externalServices`.

### Values fixed per Step

An Environment is entered once per Session and serves every Task of that Session, and a Session runs
Tasks of one Step. Values that varied per Task would force one Session per Task, which defeats the
Environment's purpose. Values fixed per Step are known at job creation, so tooling can display them, a
scheduler can group Sessions by them, and two Steps with equal values can still share Sessions as
today.

### Values resolved at job creation

`parameters` values and defaults see only the Job Parameters (and, with `EXPR`, the Step's `let`
bindings and names), all known at job creation. That makes the effective values a static property of
the Job, which is what lets a scheduler compare them to group Sessions and cache environments. Values
that depended on `Session.*` or host state would be different on every host, and the grouping would
be meaningless. Path mapping is the one host-side step, and it applies to `PATH` values as it does to
`PATH` Job Parameters.

### Environment parameters are not merged with Job Parameters

§1.2.1 merges same-named Job Parameters because a Job has one value per Job Parameter name. An
Environment's parameters are a per-Environment namespace with one value per Step; merging a queue
Conda Environment's `Packages` with a Job Environment's `Packages` would make two independent
Environments share an input they never agreed to share. The names are therefore scoped to the
Environment, and the only link to Job Parameters is the explicit one a `default` writes.

### Not on Step Environments

A Step Environment is entered only by its own Step's Sessions, and its Step writes its actions. The
Step can write any value directly in the Environment, or use a Step `let` binding, so a parameter
block there would add a second way to say the same thing.

### `LIST[PATH]` and `RANGE_EXPR` excluded

`LIST[PATH]` defaults are relative paths joined with the template directory (§2.12), which has no
meaning for an Environment parameter, and a list of paths whose mapping happens per element is a large
surface for a narrow need; a `LIST[STRING]` of `RawParam` values covers it. `RANGE_EXPR` describes a
parameter space, which belongs to a Step. Either can be added later without changing anything here.

### Required parameters are allowed

A parameter without a `default` makes the Environment useless unless each Step configures it, which is
the right contract for a Job Environment like `RenderLicense` whose author wants every Step to say
which product it uses. For an Environment Template the same rule would break every Job Template that
does not know the parameter, so this RFC states the consequence and leaves the choice to the queue
administrator, who normally gives every parameter a default.

### No extension dependency on `EXPR`

The core of the feature, string and numeric values passed to a provisioning script, needs nothing from
the expression language. Requiring `EXPR` would keep this feature from schedulers and templates that
have not adopted it. With `EXPR`, list and boolean parameters and typed expression defaults become
available, as they do for Job Parameters.

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
  `envmodules:`, and the command line's `--use-conda`, `--use-singularity`, or `--use-envmodules`
  selects which is used. Snakemake caches each distinct Conda environment by the hash of its
  definition, which is the reuse a scheduler can achieve by grouping Sessions by effective values.
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

- **Rust (openjd-rs)**: `openjd-model` gains the `parameterDefinitions` field on `Environment`,
  reusing the existing parameter-definition types with a validator that rejects the excluded fields
  and types; the `environment` variant of `StepDependency`; and scope rules for `Env.Param.*` and
  `Env.RawParam.*`. Job creation computes the effective values per Step and Environment and stores
  them on the Job's Steps, alongside the existing resolved Environment list, and the external
  resolution reuses the code that already merges Environment Template parameter definitions.
  `openjd-sessions` takes the effective values as part of entering an Environment, the way it takes
  Job Parameter values today, and adds them to the Environment's symbol table. `openjd run` passes the
  values of the Step being run. The work is moderate and mostly in validation.
- **Python (openjd-model, openjd-sessions)**: The same changes in the pydantic models and the
  session's symbol table. As RFC 0009 notes, the Python packages are intended to become bindings over
  openjd-rs, in which case they inherit the feature.
- **Schedulers**: Store the effective values per Step, pass them to the Worker when a Session enters
  each Environment, include them in the key that decides whether two Steps can share a Session, and
  resolve external names when attaching Environment Templates. A scheduler that already groups
  Sessions by Step needs no grouping change at all, since values are per Step. A scheduler may
  additionally use the values for host affinity, which is an optimization and not required.

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
motivating cases. A future RFC can add a matching key if queues standardize on one.

### Environment parameters on Step Environments

See [Not on Step Environments](#not-on-step-environments). A Step Environment's own Step is the only
possible supplier, so the values would always be written by the same author in the same Step.

## Open Questions

None block acceptance. The following are noted for the review:

1. **Standard names for queue Environments.** The portability of a Job Template that writes
   `environment: Conda` depends on queues naming their Conda Environment `Conda` and its input
   `Packages`. The samples use those names, with `CondaPackages` and `CondaChannels` as the
   template-level Job Parameters as the Deadline Cloud sample queue environments use them; a short
   conventions page in the wiki may be worth adding when this RFC is accepted.
2. **Interaction with `WRAP_ACTIONS` beyond the wrapper's own values.** A wrapping Environment reads
   its own parameters (the image to run) but not those of the Environments it wraps. No use case for
   the latter is known; `WrappedEnv.Param.*` is listed under [Future Work](#future-work).

## Future Work

* **`WrappedEnv.Param.*` for wrap hooks**, so a wrapping Environment can read the effective values of
  the inner Environment whose action it is wrapping, if a use case appears.
* **Matching external Environments by a standard kind**, as in
  [Rejected Ideas](#matching-external-environments-by-something-other-than-name).
* **A Job-level default for a whole `environment:` entry**, so a Job Template could set a value once
  for all its Steps without a Job Parameter. Today a Job Parameter plus the Environment's `default`
  covers this for queue Environments, and a `jobEnvironments` entry's own `default` covers it for Job
  Environments.

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is
more permissive.
