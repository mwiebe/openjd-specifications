* Feature Name: step_loops
* Author(s): Mark Wiebe <[mwiebe](https://github.com/mwiebe)>
* RFC Tracking Issue: (pending). The numbers `0011`, `0012`, and `0013` are placeholders assigned before tracking
  issues exist, and may change when they are created.
* Start Date: 2026-10-15
* Specification Version: 2023-09 extension CONTROL_FLOW
* Accepted On: (pending)
* Part of: the `CONTROL_FLOW` extension, which this RFC defines together with
  [RFC 0011, Control Flow](0011-control-flow.md) and [RFC 0012, Step Output Variables](0012-step-output-variables.md).
  A template that lists `CONTROL_FLOW` gets all three.
* Depends On: RFC 0011, Control Flow, and RFC 0012, Step Output Variables

> **Stub.** This RFC holds the loop sketch that was RFC 0011's Future Work, moved here unchanged. It specifies
> nothing yet.

## Summary

A placeholder for loops in `CONTROL_FLOW` (use case U17 of
[RFC 0011 §Use cases](0011-control-flow.md#use-cases)).

## Future Work

### Loop Steps

A loop Step runs its Tasks one after another, and each Task's parameters are computed from the Task that ran before
it, until an expression says to stop. It would be a separate extension, provisionally `STEP_LOOPS`, that depends on
`CONTROL_FLOW`, for the same kind of reasons that task-task dependencies are separate (see
[RFC 0011 Design Choice 8](0011-control-flow.md#8-task-task-dependencies-are-a-separate-extension)). Everything else in this RFC assumes that a
Step's Tasks are all known once its Task enumeration group has evaluated; a loop Step's are not, and a scheduler needs
a new capability to create them one at a time and run them strictly in sequence.

A sketch, to test that this RFC doesn't block it:

```yaml
- name: Refine
  dependencies:
    - dependsOn: Setup
  parameterSpace:
    taskParameterDefinitions:
      - name: Iter
        type: INT
        initial: "{{Steps.Setup.Output.StartIter}}"
        next: "{{Task.Param.Iter + 1 if Task.Output.Residual > Param.Tolerance else null}}"
    maxTaskCount: 50
  outputVariables:
    - name: Residual
      type: FLOAT
  script:
    # ...
```

* **`initial` and `next` instead of `range`.** `initial` is the first Task's value, a materialization expression in
  the Task enumeration group that may read `Steps.*`. `next` is evaluated after each Task succeeds; its value is the
  next Task's value, and `null` ends the loop. The loop parameter may be any Task parameter type, so a `STRING`
  continuation token works as well as an index.
* **The scope of `next`.** The Task that just ran: its `Task.Param.*` and its committed outputs, as a new
  `Task.Output.<Var>` symbol that exists only in `next`. That is a new evaluation point, after a Task and before the
  next one, which this RFC's stage table would gain.
* **A required bound.** Termination can't be checked statically, so `maxTaskCount` is required, in keeping with
  [RFC 0011 §Technical Requirements](0011-control-flow.md#technical-requirements) item 7. Reaching it fails the Step; a loop that should stop
  quietly says so in `next`.
* **Parameters.** In the simplest form, a loop Step's parameter space has only loop parameters. A product of a loop
  with static parameters, one independent loop per combination ("refine each configuration until it converges"),
  overlaps with `TASK_DEPENDENCIES` and is a later step.
* **Consumers** read `Steps.<Loop>.Output.*` as lists in the order the Tasks ran, and the groups that read them are
  evaluated once the loop ends, under this RFC's completeness rule ([RFC 0011 §Reductions](0011-control-flow.md#reductions) item 3).
  `collect: LAST` and `APPEND` already cover a single Task's final value and history.
* **Re-runs.** Each Task's outputs are consumed by the next Task's `next`, so under
  [RFC 0012 §Retries and re-runs](0012-step-output-variables.md#retries-and-re-runs) a re-run of a Task other than the last that reports different values
  fails. The rule needs no change.
* **Sessions.** Because a loop runs in sequence, a scheduler would do well to keep its Tasks in one Session, reusing
  its Environments.
* **Not covered.** A loop over one Step can't repeat a group of Steps, such as render, check, and re-render until
  clean. Looping over part of the step graph is a much larger change, close to the dynamic work of U15.

Prior art: CWL's loop feature, whose loop condition and output method (the last iteration's value, or all of them)
correspond to `next` and to reading the last element or the whole list; Argo Workflows' recursive templates with a `when`
condition; and AWS Step Functions' Choice state that transitions back to an earlier state. Apache Airflow, GitHub
Actions, and Tekton have no loops, because their graphs are acyclic.

> **TODO (deferred): loop Steps.** A loop extension would add Steps whose Tasks are created one at a time, so it
> would amend this contract for those Steps: evaluating a loop Step's Task enumeration group makes only its first
> Task known, each later Task becomes known when it is created, its Task count is reported as not yet known until
> the loop ends, and its `TaskParam` symbols are concrete only once the loop ends. See [Loop Steps](#loop-steps).

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is more permissive.
