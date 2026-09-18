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

## Services

When the SERVICE extension is in use (introduced in
[RFC 0009](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0009-service.md)), a Job or
Step can additionally declare **Services**. A **Service** is a long-lived process that the scheduler starts *before*
scheduling any Task in its scope, keeps running for the lifetime of its scope, and stops once the scope no longer
needs it. The scope of a Job Service (`jobServices`) is the whole Job; the scope of a Step Service (`stepServices`) is
the declaring Step. A **Service** publishes one or more named TCP ports; the scheduler allocates a concrete address and
port on the **Service**'s host and makes them available to every entity in the scope through the `Service.*`
format-string scope, so that a Task on any Worker Host knows where to connect.

Unlike an **Environment**, a **Service** is not tied to the host that runs Tasks. It runs in a **Session** of its own,
the **Service Session**, on a **Service host** chosen by the scheduler subject to the **Service**'s host requirements,
and it outlives every **Session** that runs Tasks against it. A **Service Session** enters the same **Environments**
that a **Session** for a Task in the **Service**'s scope would (the Job's, and for a Step Service also the Step's), in the
same order, around the **Service**'s own actions, so a **Service** is provisioned by the same Conda, Rez, or container
**Environments** as the Tasks. Which kinds of **Session** an **Environment** is entered in is declared by its `runScope`:
by default every kind, so existing **Environments** apply to **Services** unchanged; an **Environment** that configures
Tasks to use a **Service** (and therefore references `Service.*`) declares `runScope: [TASK]` and is skipped in
**Service Sessions**. Under the WRAP_ACTIONS extension, a wrapping **Environment** whose `runScope` includes `SERVICE`
wraps the **Service**'s `onEnter`, `onRun`, `onReadinessCheck`, and `onExit` with its `onWrapServiceEnter`,
`onWrapServiceRun`, `onWrapServiceReadinessCheck`, and `onWrapServiceExit` hooks, exactly as `onWrapTaskRun` wraps a
Task's `onRun`. Placement is up to the scheduler: a distributed render
manager might dedicate a host to a **Service**, while a single-host runner starts it alongside the Tasks on loopback.
Whatever the placement, the scheduler is responsible for making `connectAddress` and `port`, as seen from any host in the
scope, reach the service process.

### Service lifecycle

A **Service** is always in one of three states: **UNREADY** (no instance is ready), **READY** (the current instance has
passed its readiness check and has not exited), or **FAILED** (the **Service** will not be relaunched and its scope has
failed). A Job Service is **UNREADY** from Job creation; a Step Service is **UNREADY** from the time its Step's
dependencies are satisfied. Starting a **Service** means opening a **Service Session** on a host that satisfies the
**Service**'s host requirements and, within it, entering the **Environments** of the **Service**'s scope whose
`runScope` includes `SERVICE`, running `onEnter` if defined, launching `onRun`, and applying the readiness check. When
the check passes, the **Service** is **READY**.

Rather than prescribe the scheduler's sequencing in detail, the specification states constraints that a scheduler must
satisfy; how it satisfies them is its own concern.

1. All of a **Service**'s ports are allocated, and its `bindAddress` and `connectAddress` values determined, before any
   **Action** of its **Service Session** runs.
2. No **Action** of a **Service Session** (including the `onEnter` of an **Environment** it enters) begins until every
   **Service** earlier in the same list is **READY**, and, for a Step Service, until every Job Service is **READY**. A
   referenced **Service** is therefore **READY** before the referencing **Service**'s **Session** starts.
3. No Task of a Step is scheduled until every Job Service and every Step Service of that Step is **READY**. Once they
   are, Tasks are scheduled exactly as described above, with no change to how **Sessions** are formed; Step Services do
   not affect which Tasks may share a **Session**.
4. **Services** are stopped in the reverse of the order in which they were started.
5. A **Service** has at most one live **Service Session** at a time, and at most one launched `onRun` within it.
6. A **Service Session** ends when the **Service**'s scope completes (the Job or Step has no Task that could still run,
   whether because every Task completed or because the scope failed or was canceled), when the **Service** is relocated,
   or when the **Session** fails to start. A **Service** whose scope completes must have its **Session** ended whatever
   its state, including **UNREADY** and **FAILED**, so that partial state is cleaned up.
7. Before a **Service Session** ends: any running **Action** is canceled with its own cancelation method; `onExit` runs
   if it is defined and any **Action** of the **Service** has run; and every **Environment** entered is exited in reverse
   order, as at the end of any **Session**. Then the working directory is deleted and the host's allocated amounts and
   ports are released.
8. Constraint 7 does not apply to a host the scheduler has lost. Nothing is run or awaited there.
9. A **Service** that is started again after its **Session** has ended begins a new **Service Session**: new host
   selection, new ports, new working directory, **Environments** re-entered, `onEnter` re-run.
10. A scheduler may start a **Service** at any time consistent with these constraints, including lazily when it is first
    prepared to schedule a Task in the **Service**'s scope, and may decline to start a **Service** whose scope will
    schedule no Task. Once started, a **Service** is kept **READY** until its scope completes or it fails.

### Service failure and restart

Two kinds of failure lead to the restart decision. An **instance failure** occurs when `onRun` exits, with any exit
status, before the scheduler cancels it, when the readiness check times out, or when the scheduler loses the **Service
host** (it determines, by its own means, that the host is gone or unreachable). A **start failure** occurs when a
**Service Session** fails before `onRun` is launched: an **Environment**'s `onEnter` fails, or the **Service**'s `onEnter`
exits non-zero or times out.

On either failure the **Service** becomes **UNREADY**, and the scheduler cancels every Task in the scope that is currently
running and returns it to the queue; this is not counted as a Task failure. If the **Service**'s restart policy has
attempts remaining, the scheduler relaunches the **Service**. After an instance failure on a host that is still
available it may relaunch `onRun` within the existing **Service Session** (same working directory, same ports, `onEnter`
not re-run, so the state `onEnter` established is preserved), or it may relocate; after a start failure or host loss it
must begin a new **Service Session**. If the policy's `completedTasks` is `RERUN`, every Task in the scope that had
completed successfully is also returned to the queue, because the **Service** held state that made those results depend
on the lost instance; with `KEEP`, completed Tasks keep their results. If no attempts remain, the **Service** becomes
**FAILED** and its scope fails: a failed Job Service fails the Job, and a failed Step Service fails the Step.

Relaunching in a new **Service Session** on a different host is relocation. It counts as one relaunch, is governed by
the same restart policy, and changes every `Service.<name>.*` value. Because entities in the scope resolve `Service.*`
at task execution time on the Worker Host, a Task scheduled after relocation sees the new endpoint automatically. When
relocating away from a host that is still available, constraint 7 above applies to the old **Session** in full. Nothing
can run on a lost host, so the scheduler does not wait for `onExit`, **Environment** exits, or working-directory cleanup
there; if the host later reappears the scheduler should terminate any surviving service process and remove the working
directory, but the **Service**'s state is unaffected.

`RERUN` on a Step Service requeues only that Step's Tasks: a Step Service cannot fail after its Step completes, so no
other Step is affected. `RERUN` on a Job Service returns every completed Task in the Job to the queue; every Step
returns to a pending state, Step dependencies are resolved again from scratch, and a Step Service that had been stopped
because its Step completed is started again, in a new **Service Session**, when its Step next becomes schedulable.

A Task failure never fails a **Service**. A Task that fails because it could not reach a **Service** that the scheduler
still considers **READY** is an ordinary Task failure.

### Services from Environment Templates

A scheduler may supply **Services** to every Job submitted through it by attaching Environment Templates that define
a `services:` list, in the same way it supplies **Environments** today. Each Job gets its own instance of every such
**external Service**, placed before the Job's own `jobServices` in the scheduler's attachment order, and otherwise
indistinguishable from a Job Service: the Job's Tasks are gated on its readiness, and its restart policy governs the
Job's Tasks. An Environment Template may define both `services:` and an `environment:`; the **Environment** may then
reference the **Services**' endpoints, for example to set environment variables that point Tasks at them. A Job Template
does not reference external **Services** directly; it uses them through the **Environment**'s effects, as it uses any
queue **Environment** today, and needs no changes to do so.

## Session Environment Variables

The following environment variables are automatically set for all **Actions** running within a **Session**:

| **Environment Variable** | **Value** | **Corresponding Template Variable** |
|---|---|---|
| `OPENJD_SESSION_WORKING_DIR` | The absolute path to the **Session**'s **Working Directory**. | `Session.WorkingDirectory` |

These environment variables are provided for nested subprocesses that cannot access Open Job Description template
variables directly. Unlike template variables, environment variables are automatically inherited by child processes.

As a rule of thumb, OpenJD favors template variables over environment variables. When jobs use template substitution, tools can look at the template and accurately determine data flow to understand and modify the template. When jobs use environment variables, there is no reliable way to do this. We use the Tooling parseable design tenet to guide choices like this.

The `OPENJD_SESSION_WORKING_DIR` environment variable is an example of this consideration in practice. Unlike other template variables, the session working directory is already accessible without using a variable; session actions start their CWD set to the session working directory. This is the same value as the {{Session.WorkingDirectory}} template variable, so exposing it as an environment variable does not reduce the tooling parseability of Job Templates. Any future environment variable additions should be intentionally considered with the design tenets in mind rather than assumed to follow this precedent.


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
  all subsequent **Action**s in the **Session** until the defining **Environment** is exited. With the SERVICE extension,
  it may also be emitted by a **Service**'s `onEnter` **Action**, in which case it defines the variable for that
  **Service**'s own subsequent **Action**s only.
* `openjd_redacted_env: <var>=<value>` has identical behavior to openjd_env except `<value>` shall be redacted by the
  application running the action in the emitted stdout line and in any future lines. Requires the REDACTED_ENV_VARS
  extension, however supporting applications must honor the redaction even when the extension is not specified.
* `openjd_unset_env: <var>` where `<var>` is the string name of an environment variable. This can only be emitted by the
  **Action** for entering an **Environment** (or, with the SERVICE extension, a **Service**'s `onEnter`). This unsets the
  given environment variable for all subsequent **Action**s in the **Session** until the **Environment** that emitted it
  is exited, or for the **Service**'s own subsequent **Action**s.
* `openjd_service_ready` or `openjd_service_ready: <message>` where `<message>` is any string. Requires the SERVICE
  extension. This can only be emitted by the `onRun` **Action** of a **Service** whose readiness check type is `STDOUT`, and
  indicates that the service is accepting connections on all of its declared ports. Emitting it more than once has no
  additional effect; emitting it from any other **Action** is ignored. When `onRun` is wrapped by `onWrapServiceRun`,
  the line is recognized on the wrap script's stdout, as for every `openjd_*` message under WRAP_ACTIONS.

In a **Service Session**, `openjd_status`, `openjd_progress`, and `openjd_fail` are honored from the **Service**'s
`onEnter`, `onRun`, and `onExit`, and not from `onReadinessCheck`. `openjd_fail` supplies the reason reported when the
**Action** fails; the exit status decides whether it failed. A failure of `onEnter` is a start failure, of `onRun` an
instance failure, and of `onExit` an `onExit` failure (see [Service lifecycle](#service-lifecycle)). The environment
variables defined for every **Session** in [Session Environment Variables](#session-environment-variables), currently
`OPENJD_SESSION_WORKING_DIR`, are set in every **Action** of a **Service Session** with their usual meanings.

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
