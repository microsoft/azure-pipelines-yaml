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

A readonly variable can be set once, then its contents cannot be changed by user code.
Any attempt to do so generates a warning.
(This is to help with troubleshooting, in case there are tasks/scripts relying on the older behavior.)
The warning will be something like:
> `VariableName` is read-only and can't be changed.

Azure Pipelines itself can still change variables.
For instance, if the build name is changed using the appropriate logging command, then `Build.BuilderNumber` may be changed to reflect that.

All of the following should be readonly:
- All system variables (whether from the server or agent)
- Output variables (those set using `isOutput=true`)
- Queue-time variables
- Task- and script-created variables with a new setting, `isReadonly=true`

Additionally, agents should no longer overwrite the following important environment variables, even if there's a pipeline variable with the corresponding name:
- `TODO`

## Enforcement

For variables marked readonly, the agent should refuse to update them.
Additionally, if the server is asked to update these variables by the agent, it should also refuse.

## Safe deployment

There's not a lot of support or telemetry for detecting how widely our customers depend on current behavior.
Therefore, we need to put this behind a feature flag and roll it out safely.
Ideally this is a server-side feature flag, but if we need an agent-side setting as well, that's acceptable.

## On-prem considerations

An older TFS server may be using a newer agent.
The older TFS server knows nothing about readonly variables.
Those variables may remain mutable since the agent is taking its cues from the server.