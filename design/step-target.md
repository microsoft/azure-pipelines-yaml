# Step Target

Today, steps run either on the agent host or, if the step is part of a [container job](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/container-phases?view=azure-devops&tabs=yaml), inside of the container specified on the job.
All steps in a job run in one context; separate steps cannot target different contexts throughout the job.
Additionally, steps cannot target [services containers](./sidecar-containers.md) at all.

The goal of a step target is to give the pipeline author flexibility in where a given step in the job runs.
This provides additional flexibility for pipeline authors and template authors.

## Scenarios

1. Some of my steps need to run on the host, while others need to run in a job container for isolation purposes.
(For more on isolation, see the [step target "safe mode"](step-target-safe-mode.md) spec.)
2. I need to run specific steps inside specific service containers for configuration purposes.

## YAML syntax

**Example:** a container job using [services](./sidecar-containers.md) with one step targeting a service container.

In the example below we see a single job that defined two service containers as well as a job container.  

* The first step in the job targets the `postgres` service container in order to run some sort of configuration script.  
* The second step runs in the `python-builder` container that is specified as part of the `container:` property for the job.  
* The third step targets the host machine where the agent is running.

```yaml
jobs:
- job: MyJob
  container: 
    image: python:latest
    name: python-builder

## Multi-container support here
  services:
    redis:
      image: redis:alpine
      ports:
      - "6379"
    postgres:
      image: postgres:9.4
      volumes:
      - db-data:/var/lib/postgresql/data
      env:
        FOO: bar
## End multi-container

  steps:
  - script: ...
    displayName: Configure postrgres
    target: postgres  # targets a service container
  - script: ...
    displayName: Run tests
    # no target, so targets the default (in this case, job container)
  - script: ...
    target: host  # targets the host - another reserved word like "self"
  
```

## A more complicated example

In this example, we run some steps on the host and others in a container.
The user's template is considered "untrusted" code, so all passed-in steps are rewritten to target the container.
Also, we pre-configure the container so that it has no network access.

* The `azure-pipelines.yml` pipeline file specifies one job that is derived from a job template.  Here we see the consumer of that template passing in an enture job definition including a contaienr and two steps.
* The `jobs-template.yml` loops over all of the parameters in the template and all of the jobs and applies the `job-template.yml` to it.
* In the `job-template.yml` we take all of the parameters and steps from the job and remap them into a job that has a step that targets the host that first removes the network from the job container in order to make sure that when the steps run they do not have access to any external network resources.  Then we loop over each step and create new steps from he properties explicilty omitting the `target:` property so they are limited to only targeting the job container.

```yaml
## job-template.yml
parameters:
# Job schema parameters - https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=vsts&tabs=schema#job
  cancelTimeoutInMinutes: ''
  condition: ''
  continueOnError: false
  container: ''
  dependsOn: ''
  displayName: ''
  steps: []
  pool: ''
  strategy: ''
  timeoutInMinutes: ''
  variables: []
  workspace: ''
  
- jobs
  job: ${{ parameters.name }}
  # common jobs parameters
  steps:
  # remove the docker network from the job container need an easy way to determine the job network and job container
  - script: docker network disconnect <job network> <job container>
    target: host # reserved name for the host the worker is running on
  
  # copy each proerty from each step skipping the target property
  - ${{ each step in parameters.steps }}:
      ${{ each property in step }}:
        ${{ if ne(property.key, 'target') }}:
          ${{ property.key }}: ${{ property.value }}
          
## jobs-template.yml
parameters:
  jobs: []

jobs:
- ${{ each job in parameters.jobs }}:
  - template: ./job-template.yml
    parameters: 
      # pass along parameters
      ${{ each parameter in parameters }}:
        ${{ if ne(parameter.key, 'jobs') }}:
          ${{ parameter.key }}: ${{ parameter.value }}

      # pass along job properties
      ${{ each property in job }}:
        ${{ if ne(property.key, 'job') }}:
          ${{ property.key }}: ${{ property.value }}

      name: ${{ job.job }}

## azure-pipelines.yml

resources:
  containers:
  - container: mycontianer
    image: org/mycontainer:stable

jobs:
- template: /templates/jobs-template.yml
  parameters:
  jobs:
  - job: MyBuild
  	container: mycontainer
  	steps:
  	- script: ...
  	- script: ...
```



## Future work

- Need a standard way to get the name of the job network and the job container (probably pipeline variables)
- Service containers need a way to map the `workspace` like the job container does
