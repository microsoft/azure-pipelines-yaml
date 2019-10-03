# Step Target

Today, steps run either on the agent host or, if the step is part of a [container job](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/container-phases?view=azure-devops&tabs=yaml), inside of the container specified on the job.
All steps in a job run in one context; separate steps cannot target different contexts throughout the job.
Additionally, steps cannot target [services containers](./sidecar-containers.md) at all.

The goal of a step target is to give the pipeline author flexibility in where a given step in the job runs.
This provides additional flexibility for pipeline authors and template authors.

## Scenarios

1. I have a service defined on my job and I want to run a script that targets the container that service is running in.
2. I have authored a template and I want all steps defined by the consumer of my template to run inside a container after I have run a step on the host to disable the network for that container in order to make sure their build does not pull down any extra dependencies.

## YAML Syntax

**Example:** a container job using [services](./sidecar-containers.md) with one step targeting a service container.

In the example below we see a single job that defined two service containers as well as a job container.  

* The first step in the job targets the `postgres` service container in order to run some sort of configuration script.  
* The second step runs in the `python-builder` container that is specified as part of the `container:`property for the job.  
* The third step targets the host machine where the agent is running.

```yaml
jobs:
- job: MyJob
  pool:
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
  	target: postgres
  - script: ...
    displayName: Run tests
  - script: ...
  	target: host # reserved name could also be agent
  
```

**Example:** A template that runs steps from a consumer in a contianer with no network

This example is significantly more complex and shows us how we might use the step target feature along with the templates feature to create a secure and compliant pipeline by forcing all user defined steps to be run inside a container and not enabling them to target the host.  We also see removing the template author adding a step to all jobs that removes the job container network so the user steps are not able to download any additional dependencies that are not tracked in the source repo.

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



## Notes

Need to figure out the reserved target for the agent or host

Need a standard set of pipeline variables that can be used to get the name of the job network and the job container 

Service containers probably need to map the `workspace` in the same way the job container does
