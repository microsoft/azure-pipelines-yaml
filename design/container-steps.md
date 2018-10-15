# Container steps

Status: proposal

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

## Considerations
- We don't (and can't easily) support Alpine Linux containers at the job level.
There are too many runtime dependencies that it doesn't support.
But, `docker run`ning into an arbitrary container doesn't require any of that infrastructure.
So we'd have a great story for using teeny containers and not feel bad about having some extra requirements on full-job containers.
- Need to support custom volume mappings, environment variables, and custom UIDs.

## Challenges
- This will be tricky to support on-premises.
Containers in general rely on access to a Docker registry, and those tend to be cloud-hosted.
Many Azure DevOps Server instances won't have internet connectivity.
- When containers write files to volumes, they're written with the UID of the container user.
This means you either have to be root or running as that UID on the host to access the files.
This will break things like uploading artifacts unless we run that in a container as root.

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
    DOTNET_TELEMETRY_OPT_OUT: true  # add an environment variable
  cmd: build  # override the default CMD/ENTRYPOINT in the container
- run: yarn
  workspace: /my/custom/workspace # override the default workspace mapping
  uid: 1001 # overrides the default user ID that we create
```

We're adding a super abbreviated syntax, too -- if all you need is a container image name, you can say:

```yaml
- run: microsoft/dotnet:latest
- run: facebook/yarn:latest
```
