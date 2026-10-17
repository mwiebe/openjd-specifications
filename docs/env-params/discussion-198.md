**Title:** Idea for RFC to let Steps supply parameter values to the Environments that wrap them (https://github.com/OpenJobDescription/openjd-specifications/discussions/198)

### Background

A multi-application pipeline is a natural fit for one Job: decode a video into frames with ffmpeg, solve structure-from-motion with COLMAP, place 3D objects into the solved scene with Maya, and encode the result with ffmpeg. Each Step needs different software. Today an external Environment Template like a Deadline Cloud Conda or Docker queue environment takes a single CondaPackages or DockerImage Job Parameter, so the only option is to install every application in every Session or use a single unified container image, which is slow and sometimes impossible when packages conflict. A Step Environment can't help, because it has no way to ask an externally attached Environment to do anything differently.

### The RFC idea

A Session runs Tasks from a single Step, so an Environment's onEnter always runs for a known Step. The idea has two halves.

First, give <Environment> its own small parameter definition block declaring the values a Step (or Service from the SERVICE RFC) may supply, referenced as {{Env.Param.*}} alongside the existing {{Env.File.*}}. A default can point at a template-level Job Parameter, so the job-wide value applies to any Step that says nothing. The block works identically in an Environment Template and in a Job Template's jobEnvironments:

# Environment Template parameterDefinitions: - name: CondaPackages type: STRING default: "" environment: name: Conda parameterDefinitions: # new - name: Packages type: STRING default: "{{Param.CondaPackages}}" script: actions: onEnter: ... conda create ... {{Env.Param.Packages}} ...

Second, let a Step supply values through a new kind of entry in its dependencies, following the pattern the SERVICE RFC uses for service:. Naming an Environment there means "I depend on an Environment called Conda wrapping me, with these parameter values." required: false lets a Job Template run on a queue that attaches no such Environment, since the Job Template can't know what a queue attaches:

# Job Template steps: - name: SolveSfM dependencies: - dependsOn: DecodeFrames - environment: Conda # new required: false parameters: Packages: "colmap=3.11" - environment: Docker required: false parameters: Image: "docker.io/colmap/colmap:latest"

A Session for SolveSfM enters Conda with {{Env.Param.Packages}} equal to colmap=3.11. The scheduler can group Sessions by their effective Environment parameter values instead of installing everything everywhere. Because both entries are optional, this one Job Template runs on a queue that provides software through conda, one that provides it through Docker, or one where COLMAP is simply preinstalled on the hosts.

Some semantics that fall out of this shape. An environment: entry can name a jobEnvironments entry of the same Job Template as well as an external one, so a single parameterized Job Environment can serve several Steps with different values. required defaults to true, since listing something in dependencies is declaring a dependency on it. A Step's own script cannot read {{Env.Param.*}}; two wrapping Environments could declare the same parameter name, so the reference is only unambiguous inside the Environment that declares it.
