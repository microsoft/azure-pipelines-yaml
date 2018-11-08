# Container Steps Implementation

**Important note**
This design doc exists for historical interest and possible future implementation.
Azure Pipelines currently has no plans to build this feature.
For a workload like this, we currently recommend customers use `- script: docker run <container>` steps.

## Overview and Requirements

- Container steps should be an abstraction of what the actual technology is used
  - We're going to start with Docker containers running on the host
  - When we enable a Kubernetes pool provider, this would have to work
- One command, and one container per step
- Don't re-use containers between steps
  - Container creation is fast, so use that fact in our favor
  - Simpler mental model; user avoids needing to worry about cleaning up resources between steps
- Begin support on Linux. As we continue our internal testing, we'll investigate and test support for Windows Containers
- Sane defaults, but let the user override those defaults to enable more advanced scenarios

## Inputs

|Name|Type|Required|Description|
|---|---|---|---|
|image|string|true|The name of the image to run the command (e.g. ubuntu or microsoft/dotnet-samples:latest)|
|cmd|string[]|false|The command that is passed to the entry point (e.g. build or echo Hello World)|
|entrypoint|string|false|The entry point|
|env|string[]|false|A list of key-value pairs in this format: key=value. Each env var goes in its own line.|
|uid|int|false|The UID used to run the command as (unset: default UID)|
|workspace|string|false|The agent workspace path to map (unset: map to a known, consistent location)|

## Agent plugin, or TS task?

- We can implement the container steps feature in two ways:
  - As an agent plugin, in C#
  - As a task, in TypeScript

Both have their advantages and disadvantages. The current prototype is currently being written as an agent plugin, but can later be written as a task if we think it would be better over the long term.

## Agent plugin architecture

- Module: Agent.Plugins
- Namespace: Agent.Plugins.RunInContainer

|Class Name|Description|
|---|---|
|`RunInContainerPlugin`|Agent plugin which calls the appropriate CLI manager to run a command in a container (for now, Docker)|
|`DockerCliManager`|Helper class, which encapsulates the logic to build the command line to run `docker run` with the right image, environment variables, etc.|

## Environment variables

`Docker run` supports passing in environment variables in two ways:
- Using a set of -e flags (e.g. `-e foo=bar -e foo2=bar2`)
- Saving a list of env vars in a file, and reference it with the --env-file flag (e.g. `--env-file env.list`)

Using the --env-file would be the preferred option as it avoids two problems:
- Having the risk of exposing secrets in log files
- Running into a command line length limit

## Workspace mappings and file permissions

The user that runs inside the agent may not be the same as the user that runs inside the container.
Given this, the agent infrastructure should take special care with file permissions, to ensure that files created by the agent are accessible from the container, and vice versa.

One option is to ensure that the root workspace folder is accessible by both the agent user and container user. This requires knowing the UID of both the user running in the agent, and the container in advance.

The other option is to force the UIDs for both the agent and container to be the same. This would imply running a step as a custom UID is not supported.

## \#\#[vso… support

\#\#[vso… variables are a mechanism for steps to output information back to the agent. Because these variables may contain paths, we want to consider the case where the path representing the resource inside the container is different than the path outside the container, in the agent at the host level.

- **Don't support \#\#[vso… commands inside the container steps at all**
  - Easiest to implement, but most restrictive to user scenarios
- **Support \#\#[vso… commands, but do not attempt path translation**
  - Medium complexity, user would need to keep in mind that paths are not being translated if workspace was overridden
- **Support \#\#[vso… commands, and perform path translation**
  - Most complicated to implement, and also we can fall into the risk of mis-translating vars that look like paths but they're not in reality
		
Tentative suggestion: We can start out by not supporting \#\#[vso… variables, and later on see if adding support is beneficial for our customer workflows.

## Resources

`Docker run`: https://docs.docker.com/engine/reference/commandline/run
