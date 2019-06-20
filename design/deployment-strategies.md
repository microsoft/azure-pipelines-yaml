 ## Deployment strategies

One of the key advantages of continuous delivery of application updates is the ability to quickly push updates into production for specific microservices, giving developers the ability to respond rapidly to changing business requirements. 

With Azure DevOps, environments are first-class constructs, handling the underlying orchestration such as **deployment strategies**, verifying health checks, disabling old version and rolling out new version, traceability down artifact versions are key to continuous value delivery. 

This proposal contains a number of interconnected elements. For example, an [environment](environment.md), a [deployment](deployment.md) job, primitive building blocks, and simplified YAML syntax to enable **safe deployments** in the majority of cases. Our approach to building a deployment orchestration language for **safe deployments** will be to get the fundamental building blocks right. 

**runOnce**: default strategy, if not defined. Supports `deploy`, along with with `onFailure` and `onSuccess` lifecycle hooks. 

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
      onFailure:
        steps:
        - script: echo restore from backup...
      onSuccess:
        steps:
        - script: echo passed...
 ```

**Blue-Green**: Reduce deployment downtime by having identical standby environment. 
At any time one of them, let's say `blue` for the example, is live. As you prepare a new release of your software,
you do your final stage of testing in the green environment. Once the software is working in the green environment, 
you switch the traffic so that all incoming requests go to the green environment - the blue one is now idle.

```yaml
jobs:
- deployment:
  environment: musicCarnivalProd
  pool:
    name: musicCarnivalProdPool
  strategy:                   
    blueGreen:    
      init:
        steps:
        - script: echo initialize, cleanup, install certs...
      deploy:              
        steps:                                   
        - script: echo deploy web app... 
      route:
        delay: 60m
        steps:
        - script: echo swap slots...   
      check:
        timeout: 60m
        samplingInterval: 5m
        steps:          
        - task: appInsightsAlerts        
    on:
      failure:
        steps:
        - script: echo swap slots back..     
      success:
        steps:
        - script: echo checks passed...
```

For example, deploy a web app using blue-green strategy


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
      max-parallel: 5
      init:
        steps:
        - script: echo initialize, cleanup, install certs...
      deploy:              
        steps:                                     
        - script: echo deploy web app...      
      route:
        delay: 60m
        steps:
        - script: echo swap slots...   
      check:
        timeout: 60m
        samplingInterval: 5m
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

**Canary**: reduce the risk by slowly rolling out the change to a small subset of users. 
As you gain more confidence in the new version, you can start releasing it to more servers in your infrastructure
and routing more users to it. 

```yaml
jobs:
- deployment:
  environment: musicCarnivalProd
  pool:
    name: musicCarnivalProdPool 
  strategy:                 
    canary:     
      increments: [10,20] 
      init:                                    
        steps:          
        - script: initialize, cleanup....  
      deploy            
        steps:
        - script: echo deploy web app...
      route:
        delay: 60m 
        steps:
        - script: echo swap slots...
      check:
        timeout: 60m
        samplingInterval: 5m
        steps:          
        - task: appInsightsAlerts  
    on:
      failure:
        steps:
        - script: echo swap slots back..     
      success:
        steps:
        - script: echo checks passed...
 ```
