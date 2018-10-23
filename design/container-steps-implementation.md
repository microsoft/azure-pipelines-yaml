# Container Steps Implementation

## Overview and Requirements

- Container steps should be an abstraction of what the actual technology is used
  - We're going to start with Docker containers
  - But, leave the design generic to allow implementing other technologies in the future (Kubernetes, Kata containers, etc.)
  - One command, and one container per step
- Don't re-use containers between step
  - Container creation is fast, so use that fact in our favor
  - Simpler mental model; user avoids needing to worry about cleaning up resources between steps
- Sane defaults, but let the user override those defaults to enable more advanced scenarios

## Agent plugin, or TS task?

- We can implement the container steps feature in two ways:
  - As an agent plugin, in C#
  - As a task, in TypeScript

Both have their advantages and disadvantages. The current prototype is currently being written as an agent plugin, but can later be written as a task if we think it would be better over the long term.

## Environment variables

`Docker run` supports passing in environment variables in two ways:
- Using a set of -e flags (e.g. -e foo=bar -e foo2=bar2)
- Saving a list of env vars in a file, and reference it with the --env-file flag (--env-file env.list)

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

Docker run: https://docs.docker.com/engine/reference/commandline/run
