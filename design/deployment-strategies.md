## Deployment strategies

**runOnce**: default strategy, if not defined. Supports `deploy`, along with with `onFailure` and `onSucess` lifecycle hooks. 

```yaml
jobs:
- deployment:
  environment: musicCarnivalProd
  pool:
    name: musicCarnivalProdPool  
  strategy:                 
    runOnce:              
      deploy:    
        steps:             # deploy & test steps ...
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
you do your final stage of testing in the green environment. 
Once the software is working in the green environment, 
you switch the traffic so that all incoming requests go to the green environment - the blue one is now idle.

```yaml
jobs:
- deployment:
  environment: musicCarnivalProd
  pool:
    name: musicCarnivalProdPool
  strategy:                   
    blueGreen:
      blueSelector: string
      greenSelector: string      
      deploy:              
        steps:           # deploy & test steps ...
        - script: echo deploy web app...      
      routeTraffic:
        delay: 60m
        steps:
        - script: echo swap slots...   
      postRouteTrafficChecks:
        timeout: 60m
        samplingInterval: 5m
        steps:          
        - task: appInsightsAlerts        
      onFailure:
        steps:
        - script: echo swap slots back..     
      onSuccess:
        steps:
        - script: echo checks passed...
```

**Rolling**: A rolling deployment replaces instances of the previous version of an application with instances of the new version of the application on a fixed set of machines (rolling set) in each iteration. 
For example, a rolling deployment typically waits for new pods to become ready via a readiness check before scaling down the old components. 
If a significant issue occurs, the rolling deployment can be aborted.

```yaml
jobs:
- deployment:
  environment: musicCarnivalProd
  pool:
    name: musicCarnivalProdPool
  strategy:                 
    rolling:
      maxBatchSize: 5
      selector: string   # comma separated string. For example. tags in case of VM or label-selector in case of AKS etc
      deploy:               
        steps:           # deploy & test, route traffic steps ...
        - script: echo deploy web app...   
      onFailure:
        steps:
        - script: echo swap slots back...
      onSuccess:
        steps:
        - script: echo checks passed...
```


**Canary**: reduce the risk by slowly rolling out the change to a small subset of users. 
As you gain more confidence in the new version, you can start releasing it to more servers in your infrastructure
and routing more users to it. 

//ToDo
