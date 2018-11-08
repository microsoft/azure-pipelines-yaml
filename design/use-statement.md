# The `use` statement

There are currently 9 different [tool or installer](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/go-tool?view=vsts) tasks.
Conceptually they do roughly the same thing (in the user's mind):
- Get a known version of some tool, maybe from the internet
- Set up the CI environment so that when I invoke `tool`, I get that version.

## Problems

### Semantics of the task
Some tools are hard to install on the fly.
So, these are really "use a version from some limited subset of possible versions".
Others can be used to fetch arbitrary versions from the internet.
We introduced a split between "use version" and "install tool", but this has turned out to be more confusing than helpful in practice.

### Naming
The tasks are not named consistently, and they take inconsistent inputs.
We have 2 `*Tool` tasks, 3 `*Installer` tasks, 2 `*ToolInstaller` tasks, and 2 `Use*Version` tasks.
Some take a `version`, others a `versionSpec`, and one a `versionSelector`.
See the appendix for a list of current tasks.

### YAML syntax
Some of the tasks take complex, interdependent inputs (Visual Studio Test Platform Installer).
This is really hard to work with in YAML.
Also, our `taskName@taskVersion` syntax adds a layer of confusion.
```yaml
- task: UsePythonVersion@0
  inputs:
    versionSpec: 3.x
```
The tasks are all at version 0 right now, so real-world confusion is *probably* limited.

### Other things we want to include
- Proxy setup.
The agent knows about its proxy; the tasks don't always pass that along to client tooling.
- Auth setup.
If I'm going to publish to a private npm registry, I should be able to give the creds here.
- Problem matchers.
If the Python compiler emits errors in a known format, we should be able to read that via regex and surface it in our UI.
VS Code has a defined format for problem matchers.

## Solution

We'll add new YAML syntax which puts tools on the `PATH`, adds proxy information, adds problem matchers, and optionally configures auth.
This syntax will be powered by tasks, giving a familiar extensibility point for third parties to work with.

### New syntax

YAML will get a new syntax for `use`-ing a tool or ecosystem.
```yaml
- use: {toolName}
```

`{toolName}` may contain characters in [A-Za-z0-9], plus `-` and `_`.

This can be implemented as sugar over the existing syntax, much like the `powershell` and `bash` keywords today.
You'll be bound to the latest major version of the task.

To select a particular version, pass a `{versionSpec}` to `version`.
```yaml
- use: {toolName}
  version: {versionSpec}
```

`{versionSpec}` is a SemVer or SemVer-like string; see below.

By default, `use` will set up the target ecosystem to use the agent's proxy.
If this behavior isn't desired, it can be overridden with:
```yaml
- use: someTool
  proxy: false
```

If a tool needs credentials, those can be added like this:
```yaml
resources:
  feeds:
  - feed: myFeed
    auth: {auth_information}

steps:
- use: someTool someVersion
  authTo: myFeed
```

(`feed` is not yet a resource type, but it's coming.)

Tasks which implement this contract can expect a service connection at runtime.
It's up to the resource provider to convert other forms of auth (such as a raw username + password) into a service connection.

Some ecosystems will require optional, additional inputs.
Just like on `task`, you can pass a map of `inputs`.
```yaml
- use: python
  inputs:
    architecture: x64
```

We need to allow tasks the ability to remove fields in major versions.
Therefore, if a task doesn't accept one of the inputs, we won't fail the build but we will inject a warning about the mismatch.
Depending on the scenario, the build could still fail at runtime.
This makes the issue debuggable even if not ideal.

## Schema

```yaml
- use: string            # required tool name
  version: string        # optional version
  proxy: boolean         # whether to install proxy information; defaults to true
  authTo: string         # optional name of a resource to derive auth information from
  inputs: {string: any}  # optional additional arguments to pass as task inputs
```

## Example

```yaml
strategy:
  matrix:
    PY35-NODE8:
      py_version: '3.5'
      node_version: '8'
    PY35-NODE10:
      py_version: '3.5'
      node_version: '10'
    PY27-NODE8:
      py_version: '2.7'
      node_version: '8'
    PY27-NODE10:
      py_version: '2.7'
      node_version: '10'

steps:
- use: python
  version: $(py_version)
- use: node
  version: $(node_version)
- script: npm install ...
- script: python ...
```

*Note, as there's been confusion:* nothing changes in the `matrix` section.
It's purely about the `steps`.

### Task changes

The in-box tool installers and Use*Tools will evolve.

Task requirements:
- task.json includes a field indicating what ecosystem the task provides.
Something like `{ 'provides': string }`.
- Accept `version` as an input.
By convention, this should be a SemVer version spec.
It's up to the task to process these (with help from the task lib).
If an ecosystem has additional, non-SemVer "versions", the task may accept those as well.
To support container jobs, `version` is optional.
It's strongly recommended for VM jobs.
- Document the range of acceptable version numbers.
Where it's "install from the internet", say so.
Where it's "from a list in the hosted tools cache", that should be clear in the docs.
Also, we'll make sure all the tooling includes this information - for instance, Intellisense.
  - Future work: some way to acquire the list of available versions at runtime.
- Some tools have multiple architectures available.
By convention, an `architecture` field should use the agent architecture types: `x64`, `x86`, and `arm`.
Others can be added, and of course only the relevant ones should be accepted.

Additional requirements for in-box tasks:
- By convention, in-box tasks will use a consistent name scheme: `Use<Name>`.
This won't be required for third-party tasks.
- Another common parameter is `architecture`.
- Do a simplification pass on each task to make sure each input is necessary, clear, and orthogonal.

## Appendix: List of existing tool and installer tasks
- [GoTool](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/go-tool?view=vsts)
- [NodeTool](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/node-js?view=vsts)
- [HelmInstaller](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/helm-installer?view=vsts)
- [DotNetCoreInstaller](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/dotnet-core-tool-installer?view=vsts)
- [VisualStudioTestPlatformInstaller](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/vstest-platform-tool-installer?view=vsts)
- [JavaToolInstaller](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/java-tool-installer?view=vsts)
- [NuGetToolInstaller](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/nuget?view=vsts)
- [UsePythonVersion](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/use-python-version?view=vsts)
- [UseRubyVersion](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/use-ruby-version?view=vsts)
