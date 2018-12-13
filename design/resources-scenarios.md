# Resources: scenarios

*Resources* represent the products and payloads of pipelines.
A resource is the interesting unit that a customer wants to produce, consume, and trace.
If tasks and jobs are the "verbs", resources are the "nouns".

Resources in Azure Pipelines provide three things:
1. versioning: locking to a specific, immutable version
2. triggering: the ability to perform work in response to a new version of a resource appearing
3. traceability: seeing where a version of a resource came from and went to

These three properties support all of the following customer scenarios.

## Scenarios

### Build source into a product: classic continuous integration
- My source code lives in a repository on GitHub.
- Whenever I push changes to a branch, I want the source built and tested.
- Test status is reported back to GitHub on that particular commit.

The only resource is a repository.
A new version of the resource triggers a pipeline.
That particular version (a commit) is built and tested.
That version of the source can be marked with a status update in GitHub.
Azure Pipelines provides a comprehensive, tracked view of that resource, from its start as a push to Git through a successful test pass.

### Deploy a container to Kubernetes
- My web app is containerized and deployed to Kubernetes. My containers are based on Alpine Linux, and I use the official Alpine container as my base.
- Whether or not my source code has changed, changes to Alpine can affect stability and correctness of my service.
- When a new image of `alpine:edge` (bleeding edge of Alpine development) is released, I want to build my container and deploy it to a local testing cluster.
- From there, I'll run integration tests and perhaps do a bit of manual testing to make sure nothing's broken.

Here, two resources are used.
The first is a repository containing both my source and my pipeline definition.
The other is a container on Docker Hub.
In this case, the trigger isn't source, it's the container.
When I look at my testing Kubernetes cluster, I need to trace back both the commit version and the container image SHA to understand exactly what's been tested together.

### Complex, multi-stage deployment
- For safety, I run canary deployments before pushing changes out to 100% of my customers.
- When I'm ready to release code (which could be every time changes hit `master`), I need to build it, test it, and then build my container.
- Then, my container is deployed to a canary environment, where it must pass health checks and a test pass before continuing to the rest of production.
- At any given moment, other engineers in my organization need to understand what bits are deployed to which environments.

There are the two resources described in previous scenarios.
This pipeline adds a container doesn't exist at the beginning of the pipeline and is created during the pipeline.
The pipeline is triggered by one resource, depends on another resource, and produces a third resource.
All three are tracked across multiple stages and environments.
When I want to know what's deployed where, where it came from, and if I've ever seen/tested this combination of resources before, Azure Pipelines can answer those questions.

### Work items and commits tracked through to the environment
- Beginning from a work item, I can trace its path through source to CI to deployment into an environment.
- This gives me end-to-end traceability for a feature from inception to consumption.
- The same tracing can be done in reverse; starting from an environment, I can track back to what work items got deployed for the first time.

### Retry
- In the middle of a deployment stage, the network fails. This causes the deployment to fail.
- After fixing the infrastructure, I want to re-try the deployment.
- It must snap to exactly the same versions of all resources as the previous attempt.

### Rollback (redeploy)
- A recent code change broke something for a small number of customers.
- I need to roll back to the last "good" code before the break.
- I want to re-deploy the exact versions of each resource (code, containers, etc.) which were working previously.

### Long-term preservation of metadata
- On January 1, I deploy a version of my code.
- On February 1, because it's been at least 30 days, my pipelines age out of retention. This includes all the resources like containers, packages, etc.
- On July 1, a massive exploit is revealed that has affected hosted services for at least 6 months.
- In order to determine if my customers' data was ever at risk, I want to audit what versions of each resource were in production on January 1. Although the resources have aged out, the information still has forensic value. For example, I can look back at commits (code is unlikely to have been cleaned up).

### Retention tracking and resource cleanup
- When I deploy a resource to my environment, this implicitly adds a lock. Those resources will not age out while they're deployed.
- When the resource is replaced by a later version, the lock is removed. Resources no longer in an environment are eligible for cleanup.
- *Stretch goal:* When my containers age out, Azure Container Registry is notified and can begin its retention / cleanup for old, unused container versions. (This isn't limited to containers, either -- any resource could participate.)

### Integration with other systems
- I have a deep investment in my Jenkins infrastructure and don't want to upend all that (yet). But, I want to power my deployments with Azure Pipelines.
- I can set up a Jenkins resource just as easily as I set up other resources.
- Azure Pipelines can trigger and track artifacts produced by Jenkins.
- Additionally, Jenkins offers up the JIRA work items which were included in the build of this artifact.

Here, "Jenkins" stands in for any other CI provider.
"Work items" and "changes" are first-class entities for any upstream CI provider.
In fact, the model can be generalized to any artifact provider: Artifactory or MyGet could also be a valid source.
Resources are third-party extensible -- first by way of tasks, but perhaps also into YAML syntax.

## Resource behaviors

Resources influence the operation of a pipeline.
For each type of resource, we need to define default and available behaviors.
For instance, a pipeline artifact should be automatically available to all jobs downstream of where it's produced (or all jobs, period, if it comes from another pipeline).
Customers must have the ability to override these defaults.
A full listing of resource behaviors is beyond the scope of this spec, however, here's a starting point:

Resource | Default behavior | Other behaviors
---------|------------------|----------------
repository | synced and checked out in a known location | <ul><li>not synced <li>synced with additional provider-specific options <li>checked out in a non-default location</ul>
container | <ul><li>`docker pull`ed for a `job` <li>not pulled for a `deployment`</ul> | <ul><li>not pulled <li>pulled from a non-Docker Hub location</ul>
pipeline | artifacts automatically downloaded | <ul><li>artifacts selectively downloaded <li>artifacts uploaded</ul>

## User interface

A user's first contact with resources is likely the YAML file.
Two resource types exist today: `repositories` and `containers`.
We'll bias towards compat with those resources, but also consider breaking with tradition (or even changing those resources) if it makes the overall model simpler.

### Existing YAML schema
```yaml
# Resources
resources:
  containers: [ container ]
  repositories: [ repository ]

# A container
resources:
  containers:
  - container: string  # identifier (A-Z, a-z, 0-9, and underscore)
    image: string  # container image name
    options: string  # arguments to pass to container at startup
    endpoint: string  # endpoint for a private container registry
    env: { string: string }  # list of environment variables to add

# A repository
resources:
  repositories:
  - repository: string  # identifier (A-Z, a-z, 0-9, and underscore)
    type: enum  # git, github
    name: string  # repository name (format depends on `type`)
    ref: string  # ref name to use, defaults to 'refs/heads/master'
    endpoint: string  # name of the service connection to use (for non-Azure Repos types)
```

In the current system, only the `self` repository (which is implied) supports triggers.
Those triggers live at the pipeline level, not on a resource declaration.
```yaml
# CI triggers
trigger:
  batch: boolean # batch changes if true, start a new build for every push if false
  branches:
    include: [ string ] # branch names which will trigger a pipeline
    exclude: [ string ] # branch names which will not
  paths:
    include: [ string ] # file paths which must match to trigger a pipeline
    exclude: [ string ] # file paths which will not trigger a pipeline

# PR triggers
pr:
  branches:
    include: [ string ] # branch names which will trigger a pipeline
    exclude: [ string ] # branch names which will not
  paths:
    include: [ string ] # file paths which must match to trigger a pipeline
    exclude: [ string ] # file paths which will not trigger a pipeline
```

### Additions to YAML schema

The most obvious change is to support triggering on containers and other repositories.
```yaml
resources:
  repositories:
  - repository: string  # identifier (A-Z, a-z, 0-9, and underscore)
    name: string  # repository name (format depends on `type`)
    ... (properties removed for brevity)
    trigger:
      branches:
        include: [ string ] # branch names which will trigger a pipeline
        exclude: [ string ] # branch names which will not
      paths:
        include: [ string ] # file paths which must match to trigger a pipeline
        exclude: [ string ] # file paths which will not trigger a pipeline
  containers:
  - container: string  # identifier (A-Z, a-z, 0-9, and underscore)
    image: string  # container image name
    ... (properties removed for brevity)
    trigger:
      tags:
        include: [ string ] # tags which will trigger a pipeline
        exclude: [ string ] # tags which will not trigger
```

TODO: add pipelines schema

### Working with resource metadata

TODO - talk about offering up the metadata in expression context

### Other TODOs:
- illustrate version vs versionSpec
- retry vs redeploy: is this about "re-run this stage as a new attempt" vs "start a whole new run, snapping to the same things" (getting re-approvals may be relevant here)
- pipeline triggering another pipeline
- locking/retention - controlled by environment? need some kind of overall policy, too
- show source provider offering up work items
- show how defaults are represented in YAML
- add back `packages` as a resource type (triggers)
- third-party extensibility for steps
- can we offer the auth, proxy, caching stuff as a function of using resources and not even require a task/`use`?
