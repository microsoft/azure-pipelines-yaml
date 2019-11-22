# Step target command "restricted mode"

With the introduction of [step targets](step-target.md), the code running inside a given step can be mostly isolated from the agent host.
Agent logging commands remain a vector for untrusted user code to make permanent changes to its environment.
For instance, `task.setVariable` can alter system variables for the duration of the job.
`artifact.upload` can exfiltrate bits off the machine or poison otherwise-trusted artifacts.

This spec proposes an extension to step targets allowing them to be placed in "command restricted mode".
In command restricted mode, only a small, approved subset of logging commands are processed.
These are necessary for communicating the status or outcome of a step.
All other commands are excluded.

## What gets isolated?

By default, all commands are denied.
In order to be allowed, a command should meet all of these criteria:
- Command is necessary to update the agent about the current step's progress or state
- There is no safer way to perform the action
- After reasonable threat modeling, we can't see a way for untrusted code to exploit the command

## Syntax

The `target` property of a step will be expanded to allow either a simple string or a mapping.
If it's a string, the string is the target of the step (either `host` or a container resource name) and safe mode is off (as if `commands: any` were specified).
If it's a mapping, the following keys are supported:

```yaml
- script: ...
  target:
    container: string (container name or the word `host`)
    commands: enum (one of "restricted" or "any", with "any" the default)
```

## Example

```yaml
resources:
  containers:
  - container: somecontainer
    image: azcr.io/secureazurecontainer:latest

- script: echo running this step on host
- script: echo running this step in container
  target:
    container: somecontainer
- script: echo running this step in container, in restricted mode
  target:
    container: somecontainer
    commands: restricted
```

## Which commands are allowed?

As of October 2019, the following commands exist.
Commands allowed in restricted mode are marked with ✅:

| Area | Command | Allowed? | Notes
|------|---------|----------|------
| `artifact` |  `associate`
| `artifact` |  `upload`
| `build` |  `uploadlog`
| `build` |  `uploadsummary`
| `build` |  `updatebuildnumber`
| `build` |  `addbuildtag`
| `codecoverage` |  `publish`
| `codecoverage` |  `enable`
| `plugininternal` |  `updaterepositorypath`
| `release` |  `updatereleasename` | | not needed since `release` plugin isn't loaded for YAML
| `task` |  `addattachment`
| `task` |  `complete` | ✅
| `task` |  `debug` | ✅
| `task` |  `logdetail` | ✅
| `task` |  `logissue` | ✅
| `task` |  `prependpath` | ✅
| `task` |  `setprogress` | ✅
| `task` |  `setsecret` | ✅
| `task` |  `setvariable` | ✅ | requires future "readonly variables" feature to be secure
| `task` |  `settaskvariable` | ✅ | investigating whether this is truly needed
| `task` |  `setendpoint`
| `task` |  `uploadfile`
| `task` |  `uploadsummary`
| `telemetry` |  `publish` | ✅
| `results` |  `publish`

Many of the "publish" commands are probably allowable, but they apply to the job rather than the particular step.
Therefore, given the principles above, they're disallowed in restricted mode.
A follow-up script running outside of safe mode can do any sanitization required and then call the relevant command.
