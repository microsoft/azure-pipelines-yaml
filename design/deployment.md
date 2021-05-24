# A deployment job

A "deployment" job is a collection of steps to be run sequentially against the environment. 
We recommend specifying deployment steps in a deployment job.  

## Scenarios

- Run a set of steps against an environment. Record deployment history and status of the deployments.
- Perform safe deployments. In other words, define how your application is rolled-out.

  
### Schema

```yaml
jobs:
- deployment: string  # name of the deployment job, A-Z, a-z, 0-9, and underscore
  displayName: string  # friendly name to display in the UI
  dependsOn: string | [ string ]
  environment: string # target environment name and optionally a resource-name to record the deployment history; format: <environment-name>.<resource-name>
  condition: string
  continueOnError: boolean  # 'true' if future jobs should run even if this job fails; defaults to 'false'
  pool: pool # see pool schema
  timeoutInMinutes: number # how long to run the job before automatically cancelling
  cancelTimeoutInMinutes: number # how much time to give 'run always even if cancelled tasks' before killing them
  variables: { string: string } | [ variable | variableReference ] 
  strategy: 
    runOnce: # default runOnce strategy, useful for running the steps sequentially once against the environment.
      deploy:
        displayName: string # friendly name to display in the UI
        steps: [ script | bash | pwsh | powershell | checkout | task | templateReference ]
```

Following properties are on hold
```yaml
  container: jobContainer
  services: jobServices 
  workspace: jobWorkSpace. 
```

### Example
 
```YAML
jobs:
  # track deployments on the environment
- deployment: DeployWeb
  displayName: deploy Web App
  pool:
    vmImage: 'Ubuntu-16.04'
  # creates an environment if it doesn't exist
  environment: 'smarthotel-dev'
  strategy:
    # default deployment strategy, more coming...
    runOnce:
      deploy:
        steps:
        - script: echo my first deployment
```

In the above example, with each run of this job, deployment history is recorded against the "smarthotel-dev" environment.

  > Note: 
  >  - Currently only Kubernetes resources are supported within an environment, with support for VMs and other resources on the roadmap.
  >  - It is also possible to create an environment with empty resources and use that as namespace to record deployment history as shown in the example above.

The following example snippet demonstrates how a pipeline can refer an environment and a resource within the same to be used as the target for a deployment job,

```YAML
jobs:
- deployment: DeployWeb
  displayName: deploy Web App
  pool:
    vmImage: 'Ubuntu-16.04'
  # records deployment against bookings resource - Kubernetes namespace
  environment: 'smarthotel-dev.bookings'
  strategy: 
    runOnce:
      deploy:
        steps:
          # No need to explicitly pass the connection details
        - task: KubernetesManifest@0
          displayName: Deploy to Kubernetes cluster
          inputs:
            action: deploy
            namespace: $(k8sNamespace)
            manifests: |
              $(System.ArtifactsDirectory)/manifests/*
            imagePullSecrets: |
              $(imagePullSecret)
            containers: |
              $(containerRegistry)/$(imageRepository):$(tag)
```

The above approach has the following benefits - 
- Records deployment history on a specific resource within the environment as opposed to recording the history on all resources within the environment.
- Steps in the deployment job **automatically inherit** the connection details of the resource (in this case, a kubernetes namespace: *smarthotel-dev.bookings*) as the deployment job is linked to the environment. 
This is particularly useful in the cases where the same connection detail is to be set for multiple steps of the job.
