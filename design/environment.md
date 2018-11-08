## Environments in Azure DevOps

Environment represents the resources targeted by pipelines, for example, Kubernetes clusters, app services, virtual machines, service fabric clusters etc.  Typical examples of environments are **Development**, **Test**, **QA**, and **Production**

### Why build it?

- To provide traceability from code to the physical deployment targets
- Improved visibility of resource health/availability
- Support zero downtime deployments using deployment strategies – upgrade confidently!

## Defining environments

Environment at its simplest form is just a string. The pipeline can discover and register the environment. By defining the environment in Azure DevOps, you describe where the code gets deployed. Deployments are created when the pipeline job deploys a new version of the code to the environment.

For example,

```yaml
pool:
  vmImage: 'Ubuntu 16.04'
environment: smarthotel-prod    # creates an environment 'smarhotel-prod' and records deployments against it.
```
With the above, deployment history from multiple pipelines are enabled on the environment 


### Deployment job & Environment

We will be introducing a new a `job` type called `deployment`, that can understand environments, apply deployment strategies and record deployments against the `environment`. In other words, a `deployment` job is a collection of steps to be run against the `environment`. 

```yaml
- deployment: deployWeb
  displayName: Deploy web pkg
  pool:
    vmImage: 'Ubuntu 16.04'
  environment: production    # create environment and/or record deployments
  steps:
  - script: echo deploy web pkg
```

**Note**:
- Existing `job` supports `matrix` and `parallel` strategies, adding `environment` support to it would add additional **mutually exclusive strategies** viz. `canary`, `blueGreen`, and `rolling` to the mix. Including it in existing `job` type would complicate the user experience. 
- Deployment job targeting the environment can run on agent or on server.


## Add resources to an environment

Typically the environment is composed of **resources**. For example, a **production** environment composed with a farm of **web** servers, and **database**. For v1, you can add the resources from UI, and refer the environment with the resources in YAML. 

For the **smarthotel-prod** environment example below, we have an environment that maps to a **Kubernetes namespace** in a cluster with one or more containerized apps (workloads) 

![environment](images/environment.png)


Example YAML

```yaml
jobs:
- deployment:
  environment: smarthotel-prod
  pool:
    name: sh-prod-pool
  steps:
  - script: kubectl apply ...                        
```

Now that the smarthotel-prod has the **kubernetes namespace** linked, you can trace the deployments upto the namespace. 

### Environment with multiple resources

You can target and record deployments against each **group of resource** in an environment using path notation. 

For the **smarthotel-prod** example, when we add a PaaS Database in addition to the existing Kubernetes front-end, the YAML would be,

```yaml
jobs:
- deployment:
  environment: smarthotel-prod/smarthotel-web      # smarthotel-web is the kubernetes namespace that is linked
  pool:
    name: sh-prod-pool
  steps:
  - script: kubectl apply ... 
- deployment: deployDB
  environment: smarthotel-prod/smarthotel-db       # smarthotel-db is the Azure SQL DB that is linked
  pool:
    name: sh-prod-pool
  steps:
  - script: deploy Azure sql script...
```

## Future (discussion only)
`canary`, `blue-green`, and `rolling` strategies to be supported by `deployment` job. 

**Blue-Green**: Reduce deployment downtime by having identical standby environment. At any time one of them, let's say `blue` for the example, is live. As you prepare a new release of your software you do your final stage of testing in the green environment. Once the software is working in the green environment, you switch the traffic so that all incoming requests go to the green environment - the blue one is now idle.

Blue-Green example, 

```yaml
jobs:
- job: Build
  strategy:
    matrix:
      Python35:
        PYTHON_VERSION: '3.5'
      Python36:
        PYTHON_VERSION: '3.6'
  steps:
  - script: echo build/package app 
- deployment: deployWebPkg
  pool:
    image: 'Ubuntu 16.04'
  environment:
    name:  smarthotel-prod/smarthotel-web
  steps:
   - task: AzureWebApp                       
      appName: 'smarthotel'
      slot: staging
  strategy:                          # blue green/rolling/canary, with lifecycle hooks viz, pre/post healthcheck, swap etc
    blueGreen:
 
      preHealthCheck:                
        checks:                     
        - task: appInsightsAlerts
        timeout: 60m   
 
      swap:
        steps:
        - task: swapSlots
            inputs:
              appName: smarthotel
              slot: production

      postHealthCheck:
        checks:
        - task: appInsightsAlerts
        timeout: 60m
 
      onChecksFailure:
        steps:
        - task: swapSlots
            inputs:
              appName: smarthotel
        - task: notify
 
      onChecksPass:
      - script: echo 'checks passed...'

```

**Strategy (alternate option: wip)**: 

Have a typed blueGreen step. For example, typed K8S-blue-green deploy step that works with the manifest yaml file. 

```yaml
- deployment: deployWebPkg
  pool:
    image: 'Ubuntu 16.04'
  environment:
    name:  musicCarnivalQA/smarthotel-web
  steps:
   - task: K8SManifestDeploy                       
      serviceName: 'hotels'
      imageName: $(build.repository.name):$(build.buildid)
      manifest: /manifest/*.yaml 
  strategy:                                                           # blue green/rolling/canary
    blueGreenDeploy:
    - healthTimeOut: 60m

```

**Note**
- **resource discovery**: Current YAML scope is limited to creating an environment and tracking the deployments. Future, we can annotate system tasks to publish the resources (provisioned/targeted) to the environment.
