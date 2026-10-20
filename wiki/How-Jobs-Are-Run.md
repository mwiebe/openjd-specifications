# How Jobs Are Run

Before reading this we recommend diving into [How Jobs Are Constructed](How-Jobs-Are-Constructed).

## Sessions

Open Job Description's Tasks are run within the context of a **Session**. A **Session** is an ephemeral runtime
environment on a Worker Host created to run a set of Tasks from the same Job, and is destroyed when the Worker Host is
done running Tasks for that Job. The **Session** provides a way for you to customize the environment that your Tasks
run within, such as defining a shared set of environment variables or starting a container to run Tasks. Allowing
multiple tasks to execute within the same **Session** **Environment** also allows for costly set up and tear down operations
to be used for multiple Tasks, resulting in more efficient job execution. This additionally allows for operations like
installing software or downloading files to the host.

A temporary **Working Directory** is created on the Worker Host for each **Session**, and deleted once all the Tasks
within that session have completed running, any **Environments** exited, and the **Session** terminated. The
**Working Directory** is available as scratch space for the Tasks running within the **Session**, which enables things
like sharing data between Tasks.

A Job and/or Step can define zero or more **Environments** that must be entered in a defined order at the start of
a **Session** prior to running any Tasks, and exited in the reverse order prior to terminating the **Session**. An
**Environment** defines **Actions**, command-lines with arguments, to run when it is entered or exited, and may also
define a set of environment variable values that become available in all subsequent **Actions** run in the **Session**
until the **Environment** is exited.

Once the **Environments** have been created and entered for a **Session**, a series of Tasks are run within that
**Session** and the **Environments**. Tasks from any Step in a Job can run within the same **Session** provided that
the set of **Environments** that are required to run those Tasks are identical. If the parameter space of a Step
includes a task parameter with type `CHUNK[INT]` (part of the TASK_CHUNKING extension), then the **Session**
runs Chunks for that Step instead of individual Tasks. A Chunk is a set of Tasks with all of the Task parameter
values identical except for the chunked Task parameter. It takes values from an integer range expression
like "1-3" or 1-3,5,7" depending on whether the chunks are constrained to be contiguous or not.

If exactly one Environment in the session stack defines the wrap hooks (`onWrapEnvEnter`, `onWrapTaskRun`,
`onWrapEnvExit`, part of the WRAP_ACTIONS extension introduced in
[RFC 0008](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0008-environment-wrap-actions.md)),
the lifecycle actions of inner Environments and tasks
are intercepted: an inner Environment's `onEnter` is replaced by the wrapping Environment's
`onWrapEnvEnter`, a task's `onRun` is replaced by the wrapping Environment's `onWrapTaskRun`, and an inner
Environment's `onExit` is replaced by the wrapping Environment's `onWrapEnvExit`. The wrapping Environment's
own `onEnter` and `onExit` are never wrapped; they always run normally. A wrap hook runs only in place
of an action the inner Environment defines: if an inner Environment defines no `onExit` (or defines no
script at all), there is nothing to replace and the corresponding wrap hook does not run for it. If more
than one Environment
in the session stack defines any wrap hook, the session is invalid and the scheduler must reject it
before entering any Environment. If an Environment defines any wrap hook, it must define all three.

A wrap hook is treated as the action it replaces. A failed `onWrapEnvEnter` is an inner `onEnter`
failure, a failed `onWrapTaskRun` is a task `onRun` failure, and a failed `onWrapEnvExit` is an inner
`onExit` failure; the **Session** handles each failure exactly as it would in the unwrapped case. Wrap
scripts must propagate the wrapped process's exit status as their own.

Wrap hooks extend the **Session**'s cleanup guarantee symmetrically: if `onWrapEnvEnter` starts for an
inner Environment, that inner Environment's `onWrapEnvExit` must run, and the wrapping Environment's own
`onExit` must run on top of that. Resources allocated by the wrapping Environment's `onEnter` (for
example, a container) must remain available until every `onWrapEnvExit` returns.

The wrap hook's own cancelation method governs how the wrap script is canceled. The wrapped action's
cancelation semantics are surfaced to the wrap script via the `WrappedAction.Cancelation.Mode` and
`WrappedAction.Cancelation.NotifyPeriodInSeconds` template variables, so the wrap script may honor the
inner action's cancelation semantics when it propagates the termination signal — for example, allowing
a graceful notification period equal to `WrappedAction.Cancelation.NotifyPeriodInSeconds` before
terminating when the wrapped action requested `NOTIFY_THEN_TERMINATE`. When the wrap script receives a
termination signal, it must cause the wrapped process to receive a termination signal within the
cancelation grace period; after the grace period the wrap script receives SIGKILL, which cannot be
trapped.

When the EXPR extension is used, format strings evaluated on the Worker Host support the full
[expression language](2026-02-Expression-Language) including arithmetic, conditionals, function calls,
path manipulation, and script embedding functions like `repr_sh()` for safe shell quoting.

All failures and cancellations in a **Session** are terminal for the **Session**, as the system generally does not know
what state a failure or cancellation leaves the **Session** in. Think of a Task failure which inadvertently terminates
the container that was set up in an **Environment** for the Tasks to run
within. A failed Task will result in a failed **Session**, which will subsequently exit all **Environments** that the
**Session** entered or attempted to enter, and then be terminated itself.

The following diagram provides an overview of the steps taken to run a Session.

![Actions Run in a Session](./images/2023-09_how_jobs_are_run.png)

## Session Environment Variables

The following environment variables are automatically set for all **Actions** running within a **Session**:

| **Environment Variable** | **Value** | **Corresponding Template Variable** |
|---|---|---|
| `OPENJD_SESSION_WORKING_DIR` | The absolute path to the **Session**'s **Working Directory**. | `Session.WorkingDirectory` |

These environment variables are provided for nested subprocesses that cannot access Open Job Description template
variables directly. Unlike template variables, environment variables are automatically inherited by child processes.

As a rule of thumb, OpenJD favors template variables over environment variables. When jobs use template substitution, tools can look at the template and accurately determine data flow to understand and modify the template. When jobs use environment variables, there is no reliable way to do this. We use the Tooling parseable design tenet to guide choices like this.

The `OPENJD_SESSION_WORKING_DIR` environment variable is an example of this consideration in practice. Unlike other template variables, the session working directory is already accessible without using a variable; session actions start their CWD set to the session working directory. This is the same value as the {{Session.WorkingDirectory}} template variable, so exposing it as an environment variable does not reduce the tooling parseability of Job Templates. Any future environment variable additions should be intentionally considered with the design tenets in mind rather than assumed to follow this precedent.

## Step States and Step Input Binding

Without the CONTROL_FLOW extension (introduced in [RFC 0011](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0011-control-flow.md), draft), every Step's Tasks are known when the Job is
created, and a Step's Tasks are scheduled once the Steps it depends on have completed successfully.

With CONTROL_FLOW, a Step may compute its parameter space, its host requirements, its `let` bindings, its Step
Environments, or whether it runs at all (`enableIf`) from the output variables of the Steps it depends on. Those
values don't exist until the producing Steps have run, or (for `TaskParam`) their Tasks are known, so such a Step is
**pending inputs** (`PENDING_INPUTS`) from Job creation until it is bound (every group of its fields described below
has been evaluated), skipped, or failed. None of its Tasks can be scheduled while it is pending inputs.

The scheduler performs **step input binding** progressively. The Step's fields that are resolved before its Tasks run
are divided into groups, and each group can be evaluated as soon as the values it reads are available. A scheduler
should evaluate it then, and must have evaluated every group before it schedules any of the Step's Tasks:

1. `enableIf` is always decided first. Nothing else of the Step is evaluated until it is. If it is `false`, the Step
   is **skipped**: it has zero Tasks, and none of its other fields is evaluated. A Step without `enableIf` is enabled
   at once.
2. Then, independently and in whichever order their values arrive:
   * its parameter space is evaluated and checked against the scheduler's limit on the number of Tasks, and the
     Step's **Tasks are known**: they can be listed and counted;
   * its host requirements are evaluated, and its **host requirements are known**;
   * with a future task-task dependencies extension (`TASK_DEPENDENCIES`), the dependencies between its Tasks are
     evaluated and checked, once its Tasks, and the Tasks of the Steps those dependencies name, are known and the
     values they read are available;
   * the rest of its fields, such as its Step Environments and action timeouts, are evaluated.

   A `let` binding of the Step is evaluated with the first group that uses it, and every one is evaluated before
   the Step is bound.

A Step whose Tasks are known is still pending inputs until every group has been evaluated, but a scheduler or tool
can already show its Tasks to the people monitoring the Job and plan capacity for them, and a Step whose host
requirements are known tells the scheduler which Worker Hosts it may need. A scheduler or tool that shows a pending
Step should report, for whether it runs, its Tasks, and its host requirements, whether each is known, and for each
that is not, what it is waiting for: the values of other Steps that are not available yet, or the Step's undecided
`enableIf`. In a Step whose `enableIf` is decided `true` (or absent) at Job creation, the groups whose values are
available then are evaluated then, and an error in one makes Job creation fail.

The Step then becomes one of:

* **Ready** — Every group has been evaluated, and the Step is bound. Its Tasks are scheduled as they are today, once
  its dependencies are satisfied. A Step whose computed parameter space is empty has zero Tasks and completes
  successfully once its dependencies are satisfied.
* **Skipped** — Its `enableIf` was `false`. It runs nothing, and its output variables are `null` to the Steps that
  read them. A Step whose `enableIf` reads only values available at Job creation, such as Job Parameters, is
  decided at Job creation instead.
* **Failed** — An expression in any group failed, its Task count exceeded the scheduler's limit, or (with a future
  `TASK_DEPENDENCIES` extension) the dependencies between its Tasks have a cycle. It fails as soon as the group with
  the error is evaluated, without waiting for the values its other groups read, and none of its Tasks runs.

No Task of a Step is scheduled until every group has been evaluated, so evaluating groups early changes when a Step's
Tasks and host requirements can be seen, not when its Tasks run.

A skipped Step, and a Step that completed with zero Tasks, satisfies the dependencies of the Steps that depend on it
once its own dependencies are satisfied, so skipping a Step never lets later work start before the work ordered ahead
of it. A failed Step does not satisfy dependencies, and a pending Step whose producer fails never runs, as with any
dependency on a failed Step today.

A scheduler may create a pending Step's Tasks only when it is bound, when its Tasks are known, or create the Step at
submission and fill in its Tasks later. The three are equivalent to a Job's author.

Each Task of a Step that declares `outputVariables` reports its values with `openjd_output_var:` messages (see
[Stdout/Stderr Messages](#stdoutstderr-messages)). The runtime validates each one as it is printed, the values are
committed when the Task succeeds, and the scheduler keeps them for the life of the Job and may display them, for
example as metrics beside the Task's parameters. See the
[Template Schemas](2023-09-Template-Schemas#371-reporting-output-values) for the rules.

A Job Template may also declare Job outputs (`jobOutputDefinitions`), the counterpart of Job Parameters: values
computed from the Job Parameters and the Steps' output variables that the scheduler reports to the Job's submitter
and other callers. The scheduler evaluates each one as soon as every Step value it reads is known, or at Job creation
if it reads none, so that each can be shown as soon as it is known. A Job does not succeed until its Job outputs
whose inputs are known have been evaluated, and a Job output that fails to evaluate fails the Job. See the [Template Schemas](2023-09-Template-Schemas#217-joboutputdefinition-extension-control_flow).


## Host Requirements

Open Job Description provides a mechanism for a Step to impose requirements that must be satisfied for Tasks from the
Step to be scheduled. Each requirement corresponds to an attribute of a Worker Host or render manager that must be
satisfied to allow the Step to be scheduled to the Worker Host. Some examples of concrete attributes include
processor architecture (x86_64, arm64, etc.), the number of CPU cores, the amount of system memory, or available
floating licenses for an application. We also allow for user-defined requirements whose meaning is specified by the
user; a `“SoftwareConfig”` requirement whose values could be `“Option1”` or `“Option2”`, for example.

There are two types of requirements: amount and attribute.

Amount requirements are the mechanism for defining a quantity of something that the Worker Host or render manager needs
for a Step to run. They represent quantifiable things that must be reserved — vCPUs, memory, licenses, etc. They are
always a non-negative floating point value, and a Step can require a certain amount of that capability, “at least 4 CPU
cores” for example. Further, a quantity of each amount required are allocated to a **Session** while that **Session**
is running on a Worker Host. A Step requiring “at least 4 CPU cores” might result in a **Session** with 4 CPU cores
allocated to it being created on a Worker Host. Those cores are reserved for that **Session** while it is running on
the Worker Host. For scheduling purposes, this effectively reduces the number of available cores on the Worker Host by
4 for the duration of the **Session**. Logically fitting multiple **Sessions** into a single Worker Host is the key
mechanism by which system resources can be optimized by a render management system.

Attribute requirements are the mechanism for specifying an attribute a Worker Host must have for a Step to be scheduled
to it. They are always defined on a Worker Host as a set of strings. A Step can assert that it requires a Worker Host to
have specific value(s) of the attribute for it to be scheduled to the Worker Host. For example, a Step may require its
Tasks run on a host that is in the `"linux"` or `"macos"` family of operating systems, or that a host has the shared
filesystems `"movie-A"`, `"shared-assets"`, and `"software"` all mounted.

## Stdout/Stderr Messages

Open Job Description defines some special messages that can be emitted on stdout or stderr as a single line by an
**Action** when it is running within a **Session**. The application that is running the **Action** may intercept these
messages to convey information about the **Action** to the render management system. These special lines contain one of:

* `openjd_progress: <number>` where `<number>` is a percentile between 0 and 100 that indicates what the progress of
  the **Action** is.
* `openjd_status: <message>` where `<message>` is any string. This provides a human-readable status message indicating
  what the **Action** is doing. For example, a status message of `“loading assets”` or `"rendering"` or
  `"dohickying the whatsits."`
* `openjd_fail: <message>` where `<message>` is any string. This provides a human-readable message that indicates why
  an **Action** has failed.
* `openjd_env: <var>=<value>` where `<var>` is the string name of an environment variable, and `<value>` is a string. This
  can only be emitted by the **Action** for entering an **Environment**. It defines the value of an environment variable for
  all subsequent **Action**s in the **Session** until the defining **Environment** is exited.
* `openjd_redacted_env: <var>=<value>` has identical behavior to openjd_env except `<value>` shall be redacted by the
  application running the action in the emitted stdout line and in any future lines. Requires the REDACTED_ENV_VARS
  extension, however supporting applications must honor the redaction even when the extension is not specified.
* `openjd_unset_env: <var>` where `<var>` is the string name of an environment variable. This can only be emitted by the
  **Action** for entering an **Environment**. This unsets the given environment variable for all subsequent **Action**s
  in the **Session** until the **Environment** that emitted it is exited.
* `openjd_output_var: <name>=<json>` or `openjd_output_var: <name>=@<path>` where `<name>` is the name of an output
  variable that the Step declares, `<json>` is a JSON value on one line, and `<path>` names a file inside the
  **Session**'s **Working Directory** that holds one JSON value. This reports a value of an output variable of the
  **Task** whose `onRun` **Action** printed it, and can only be emitted by that **Action**. In a Step with a
  `CHUNK[INT]` Task parameter, the name is followed by `[<value>]`, the chunk parameter's value for the **Task** of
  the chunk that the value is for, as in `openjd_output_var: <name>[<value>]=<json>`. The runtime reads a `@` file
  as soon as it reads the line. An invalid message, or one emitted by any other **Action** of the Job Template, cancels
  the **Action** at once and fails it, and stays in the log with the error. Under WRAP_ACTIONS, a line from a wrap hook
  counts as output of the **Action** it runs in place of. Requires the CONTROL_FLOW extension; without it, and for
  **Actions** of an Environment Template, the line is ordinary output. See [Reporting output
  values](2023-09-Template-Schemas#371-reporting-output-values).

When the WRAP_ACTIONS extension is in use, schedulers must scan the stdout of the wrap script (`onWrapEnvEnter`,
`onWrapTaskRun`, or `onWrapEnvExit`) for these macros, not the stdout of the wrapped process. Wrap scripts must
forward the wrapped process's stdout and stderr verbatim — without buffering, filtering, or transformation —
which causes macros emitted by the wrapped process to be recognized identically to macros emitted by the wrap
script itself. A wrap script may also emit these macros directly. The `WrappedAction.Environment` template
variable includes every `openjd_env`-defined variable emitted by any earlier **Action** in the same
**Session**, regardless of whether that **Action** ran normally or via a wrap hook.

## Path Mapping

An artists' workstation's filesystem may not match that of the Worker Host that will be running a Task. Shared
filesystems may be mounted in different locations, or the workstation and Worker Host may be different operating systems.
Open Job Description provides mechanisms for running Tasks to discover and correct for these differences in environment.

An application running an Open Job Description **Session** may communicate **Path Mapping Rules** to running subprocesses
via a JSON file in the **Session**'s **Working Directory**. This allows the render management system, which knows how and
from what environment a Job was submitted, as well as the configuration of the Worker Host, to accurately control path
mapping within the **Session**, by providing a set of programmatically generated **Path Mapping Rules**.

The format for the path mapping rules is:

```yaml
{
   "version": "pathmapping-1.0",
   # The path_mapping_rules list will be empty if there are no path mapping rules defined
   "path_mapping_rules": [
      {
         # The path format that the submitting operating system uses
         "source_path_format": "POSIX" | "WINDOWS",
         # The filepath prefix to map (e.g. "/home/user" or "c:\Users\user")
         "source_path": <string>,
         # The location on the worker host where the path can be found
         # (e.g. "/mnt/shared/user" or "z:\")
         "destination_path": <string>
      },
      ...
   ]
}
```

The location of this file, and whether there are path mapping rules defined, are available as value references for
use in [Format Strings](How-Jobs-Are-Constructed#format-strings) within a Job Template. The value references available are:

**`@extension EXPR` — URI source paths**: When the EXPR extension is enabled,
`source_path_format` also accepts `"URI"` for mapping URI prefixes (e.g. `s3://bucket/assets`)
to local filesystem paths. URI matching uses string prefix comparison on path boundaries,
preserving the exact URI content without normalization.

1. `Session.HasPathMappingRules` -- Has the string value `true` or `false`.
    * `true` — means that the path mapping JSON contains path mapping rules.
    * `false` — means that the path mapping JSON contains no path mapping rules.
2. `Session.PathMappingRulesFile` -- Has a string value providing the absolute file path to the path mapping rules JSON file.
   The JSON object in this file is allowed to be the empty object if there are no path mapping rules defined.

### Applying Path Mapping Rules within a Job Template

The **PATH** type Job and Task parameters provide an integration point in the Job Template for the path mapping system.
If a collection of path mapping rules are provided to a **Session**, then those rules are applied to all **PATH** type
parameters when they are resolved in Format Strings. The path mapping rules that are supplied are applied in order of
decreasing length of the source path, and processing stops for particular value once a rule matches.

With the CONTROL_FLOW extension, a **PATH** value that a Task reports as an output variable is mapped in the other
direction before it is stored, giving the value in the submission path context, the form in which **PATH** Job
Parameters are given; the stored value must map forward with the same rules back to the reported path. It is then
mapped forward, as a **PATH** parameter is, in each **Session** that reads it. A value for which no such form can be
found is invalid when the **Session** has rules, and is kept as it is when the **Session** has none. A URI is stored
unchanged, and a relative path is invalid. See the
[Template Schemas](2023-09-Template-Schemas#371-reporting-output-values).
