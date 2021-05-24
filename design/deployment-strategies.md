 ## Deployment strategies

One of the key advantages of continuous delivery of application updates is the ability to quickly push updates into production for specific microservices, giving developers the ability to respond rapidly to changing business requirements. 

With Azure DevOps, environments are first-class constructs, handling the underlying orchestration such as **deployment strategies**, verifying health checks, disabling old version and rolling out new version, traceability down artifact versions are key to continuous value delivery. 

This proposal contains a number of interconnected elements. For example, an [environment](environment.md), a [deployment](deployment.md) job, primitive building blocks, and simplified YAML syntax to enable **safe deployments** in the majority of cases. Our approach to building a deployment orchestration language for **safe deployments** will be to get the fundamental building blocks right. 

**runOnce**: default strategy, if not defined. 

```yaml
jobs:
- deployment:
  environment: musicCarnivalProd
  pool:
    name: musicCarnivalProdPool  
  strategy:                 
    runOnce:              
      deploy:    
        steps:             
        - script: echo deploy web app...   
 ```

When deploying application updates it is important that the technique used to deliver update enables initialization, deploying the update, testing the updated version after routing traffic and in case of failure, run steps to restore to last known good version. We achieve this using fundamental building blocks viz., hooks where you can run your steps during deployment lifecycle. Each of the lifecycle hooks will be resolved into an agent, or a [server](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/phases?view=azure-devops&tabs=yaml#server-jobs), (*or a container or validation job in future*). The lifecycle job type would be determined by the `pool` attribute. By default the lifecycle jobs will inherit the pool type specified by the `deployment`. 

Following are descriptions of the lifecycle events where you can run a hook during a deployment lifecycle

**preDeploy** – Use to run tasks before the deploy step is executed.

**Deploy** – Use to run the deploy tasks.  The results of a lifecycle hook event can trigger a rollback.

**routeTraffic** – Use to run tasks that serves the traffic to the updated version. The results of a lifecycle hook event can trigger a rollback.

**postRouteTraffic** - Use to run the tasks after the traffic is routed. Typically these tasks monitor the health of the updated version for defined interval. The results of a lifecycle hook event can trigger a rollback.

**on: Failure or Success** - Use to run the task to peform rollback actions or clean-up. The results of a lifecycle hook event doesn't trigger a rollback.

Using the lifecycle hooks, we can achieve complex deployment events such as - 

- Canary
- Rolling

**Canary**: reduce the risk by slowly rolling out the change to a small subset of users. 
As you gain more confidence in the new version, you can start releasing it to more servers in your infrastructure
and routing more users to it. 

Consider the following Azure Pipelines pipeline defined in YAML

```yaml
jobs:
- deployment:
  environment: musicCarnivalProd
  pool:
    name: musicCarnivalProdPool 
  strategy:                 
    canary:     
      increments: [10,20] 
      preDeploy:                                    
        steps:          
        - script: initialize, cleanup....  
      deploy:            
        steps:
        - script: echo deploy updates...
      routeTraffic:
        steps:
        - script: echo routing traffic...
      postRouteTaffic:
        pool: server
        steps:          
        - script: echo monitor application health...  
      on:
        failure:
          steps:
          - script: echo perform rollback actions.     
        success:
          steps:
          - script: echo checks passed, notify...
 ```


**Rolling**: A rolling deployment replaces instances of the previous version of an application with instances of the new version of the application on a fixed set of machines (rolling set) in each iteration. 

For example, a rolling deployment typically waits for new pods to become ready via a readiness check before scaling down the old components. If a significant issue occurs, the rolling deployment can be aborted.

```yaml
jobs:
- deployment:
  environment: musicCarnivalProd
  pool:
    name: musicCarnivalProdPool
  strategy:                 
    rolling:
      maxParallel: 5
      preDeploy:
        steps:
        - script: echo initialize, cleanup, install certs...
      deploy:              
        steps:                                     
        - script: echo deploy web app...      
      routeTraffic:
        steps:
        - script: echo swap slots...   
      postRouteTaffic:
        pool: server
        steps:          
        - task: appInsightsAlerts .   
      on:
        failure:
          steps:
          - script: echo swap slots back..     
        success:
          steps:
          - script: echo checks passed...
```

#### Variables
During execution, the jobs would have access to pre-defined system variables. For example, $(strategy), $(environment.staging), $(environment.prod), $(action), $(increments) etc. 

### Virtual Machine deployment (for discussion)

Azure Pipelimes supports **push** using remote script such as SSH and **pull** deployments using local agents to Virtual Machines. The proposal involves supporting **push** based deployments as first class option using remote PowerShell or SSH to Virtual Machines in the environment. 


