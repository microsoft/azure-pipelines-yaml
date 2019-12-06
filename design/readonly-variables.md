# Read-only variables

Pipeline variables are little bits of data which help tasks/scripts in a pipeline decide what work to perform.
For example, we make the triggering reason available as a variable called `Build.Reason`.
Variables are exposed to tasks/scripts by making them available as environment variables.
Scripts and tasks can also create new variables or update the contents of existing ones.

## The problems with variables

Creating new variables is not a problem.
Mutating existing ones, however, can be problematic:
- A step can accidentally or maliciously clobber another step's variable, changing future steps' behavior
- Similarly, a system variable can have its contents changed, altering behavior
- Because of the mechanism for setting variables ([logging commands](https://docs.microsoft.com/azure/devops/pipelines/scripts/logging-commands)), using a feature like `set -x` can cause a variable to get its intended contents plus a spurious `'` at the end
- Steps can accidentally or maliciously overwrite important shell variables like `TEMP`, altering the agent's behavior

We can imagine scenarios where a poorly-secured step in a pipeline could allow for an injection attack against another step / job.
It's important to close this gap as much as possible, without wrecking the ability to pass data between steps.

## Solution: make variables readonly

Many of these problems can be fixed, or at least partially mitigated, by marking some variables as readonly.
In fact, we document that this is the case for system-injected pipeline variables, but don't actually enforce it today.

A readonly variable can be set once, then its contents cannot be changed using `##vso[task.setVariable]`.
Any attempt to do so generates a warning.
(This is to help with troubleshooting, in case there are tasks/scripts relying on the older behavior.)
The warning will be something like:
> `VariableName` is read-only and can't be changed.

Azure Pipelines itself can still change variables.
For instance, if the build name is changed using the appropriate logging command, then `Build.BuildNumber` may be changed to reflect that.

All of the following should be readonly:
- All system variables (whether from the server or agent) - this includes `System.`, `Agent.`, `Build.`, and so on
- Output variables (those set using `isOutput=true`)
- Queue-time variables
- Task- and script-created variables with a new setting, `isReadonly=true`
- YAML-described variables created with a new property, `readonly: true`

## Example

This example YAML shows all the kinds of readonly variables.

```yaml
variables:
- name: first
  value: one
  readonly: true  # new syntax marking this variable readonly

steps:
- script: echo "##vso[task.setVariable variable=second;isReadonly=true]two"
  displayName: Set a readonly variable from a script/task
- script: echo "##vso[task.setVariable variable=third;isOutput=true]three"
  displayName: Output variables are automatically readonly
- script: echo Another readonly variable is $(Build.SourcesDirectory)
  displayName: System variables are readonly
# not pictured: queue-time variables that came from the server
# are also readonly
```

## Enforcement

For variables marked readonly, the agent should refuse to update them.
Additionally, if the server is asked to update these variables by the agent, it should also refuse.

## Not covered

This feature does not address the scenario where a pipeline variable overwrites an important shell environment variable (think `TEMP`/`TMP`, `PATH`, etc.).
This feature only covers which pipeline variables are readonly.
A sufficiently paranoid pipeline author could implement a pattern like the following:

```yaml
steps:
- script: echo "##vso[task.setVariable variable=PATH;isReadonly=true]$PATH"
  displayName: Preserve the initial value of PATH
```

Then all subsequent steps will get the frozen copy of `PATH`.

A future feature may be needed to address not clobbering existing environment variables.

## Safe deployment

There's not a lot of support or telemetry for detecting how widely our customers depend on current behavior.
Therefore, we need to put this behind a feature flag and roll it out safely.
Ideally this is a server-side feature flag, but if we need an agent-side setting as well, that's acceptable.

Plan:
- Sprint _X_: When the flag is on, warn in situations where a variable would be overwritten.
Turning off the feature flag turns off this warning (to fix situations where a customer is overwhelmed with warnings).
- Sprint _X+1_: When the flag is on, variables cannot be overwritten.
Turning off the flag turns off the behavior but does _not_ turn off the warning.
- Sprint _X+2_: Remove feature flag.

## On-prem considerations

An older TFS server may be using a newer agent.
The older TFS server knows nothing about readonly variables.
Those variables may remain mutable since the agent is taking its cues from the server.
