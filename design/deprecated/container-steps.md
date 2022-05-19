# Container steps

Status: **ON HOLD, NOT IMPLEMENTING RIGHT NOW**

An emerging pattern is running discrete steps of a job in different containers.
Each one carries along a persistent, read/write volume as its workspace.

An example straight from a customer:
-	microsoft/dotnet => build C# code
-	yarn => build and bundle JS
-	aws/sdk => push static files to S3
-	gcloud/sdk => download database files from private buckets, update some GCP stuff
-	docker => build and push image for this app
-	kubectl => deploy this new image on Kubernetes
-	our custom container => do checks and send email/Slack

Containers steps can be seen as an alternative to tasks.
Instead of writing to our custom task.json system and packaging dependencies into a VSIX, you write to Docker's system and package dependencies in an image.

## Solving the file permissions problem
When containers write files to volumes, they're written with the UID of the container user.
This means you either have to be root or running as that UID on the host to access the files.
This will break things like uploading artifacts unless we run that in a container as root.
We'll need agent work to move those operations out to a plugin, which we can then run in a container as root.

## Other considerations
- We don't (and can't easily) support Alpine Linux containers at the job level.
There are too many runtime dependencies that it doesn't support.
But, `docker run`ning into an arbitrary container doesn't require any of that infrastructure.
So we'd have a great story for using teeny containers and not feel bad about having some extra requirements on full-job containers.
- Need to support custom workspace mapping, environment variables, and custom UIDs.
- Need to support mapping in the Docker daemon so that `docker` CLI will work as expected.
- Need to support the job running as a container when a container step is used.

## Later
- Arbitrary volume mappings
- Boolean to de-privilege the container so it can't call out to the host Docker daemon
- Build and run from a Dockerfile in the repo.
The .NET CLI team would use this; talk to @livarcocc as needed.
Consider these scenarios:
  - I have have a tool that I want to define as part of a container but don't want to push it to a registry.
  - My deployment tooling is packaged as a container and whenever I run that tooling I want the version of that tooling from the branch.

## Challenges
- This will be tricky to support on-premises.
Containers in general rely on access to a Docker registry, and those tend to be cloud-hosted.
Many Azure DevOps Server instances won't have internet connectivity.

## YAML syntax

This is loosely based on `docker` command line syntax.

```yaml
resources:
  containers:
  - image: microsoft/dotnet:latest
    name: dotnet
  - image: facebook/yarn:latest
    name: yarn

steps:
- run: dotnet
  env:
    DOTNET_CLI_TELEMETRY_OPTOUT: true  # add an environment variable
  cmd: build  # override the default CMD/ENTRYPOINT in the container
- run: yarn
  workspace: /my/custom/workspace # override the default workspace mapping
  uid: 1001 # overrides the default user ID that we create
```

If your container image is from DockerHub, you can skip the forward-declaration and use the image name directly.

```yaml
- run: microsoft/dotnet:latest
  env:
    DOTNET_CLI_TELEMETRY_OPTOUT: true  # add an environment variable
- run: facebook/yarn:latest
```

If you need a custom Docker registry, you have to go with forward-declarations in the `resources` section.
