# Triggers in pipelines
Any DevOps lifecycle comprises of bunch of process that run at different stages of the lifecycle consuming and exposing data through various channels. Triggers enable customer to orchestrate the DevOps process in an efficient manner by automating the CI/CD process.

Triggers are events on which you can start your pipeline run automatically. You can enable triggers on your pipeline by subscribing to both internal and external events. An event can be completion of a process, availability of a resource, status update from a service or a timed event. 

At high level there are 3 different types of pipeline triggers.
1. Resource triggers
2. Webhook triggers
3. Schedule triggers

## Resource triggers
You can enable triggers on the resources defined in your pipeline. Resources can be of types pipelines, repositories, containers and packages.
Triggers are enabled by default on all the resources. However, you can choose to override/disable triggers for each resource.



### Pipelines
A new pipeline is triggered automatically whenever a new run of the `pipeline` resource is succesfully completed. See [pipeline resources](pipeline-resources.md#resources-pipelines) for more details.

#### Scenarios
- I would like to trigger my pipeline when an artifact is published by ‘Helm-CI’ pipeline that ran on `releases/*` branch.
- I would like to trigger my pipeline when an artifact is published and tested as part of Helm-CI pipeline and tagged as 'Production'.
- I would like to trigger my pipeline when ‘TFS-Update’ pipeline has completed ‘Ring2’ stage so that I can run some diagnostics.

Usually, artifacts produced by a CI pipeline are consumed in another CD pipeline. Triggers help you achieve CICD scenarios.
So we enable triggers on `pipeline` resource by default unless expliciltly opted out.

#### Schema
```yaml
resources:       
  pipelines:
  - pipeline: string 
    source: string  
    trigger:     # Optional; Triggers are enabled by default.
      branches:  # branch conditions to filter the events, optional; Defaults to all branches.
        include: [ string ]  # branches to consider the trigger events, optional; Defaults to all branches.
        exclude: [ string ]  # branches to discard the trigger events, optional; Defaults to none.
      stages: [ string ]  # trigger after completion of given stage, optional; Defaults to all stage completion. stages are OR'd
      tags: [ string ]  # tags on which the trigger events are considered, optional; Defaults to any tag or no tag. tags are OR'd
```

#### Examples
 You can disable the triggers on the `pipeline` resource.
```yaml
resources:
  pipelines:
  - pipeline: SmartHotel
    source: SmartHotel-CI 
    trigger: none   
```


You can control which branches get the triggers with a simple syntax.
```yaml
resources:
  pipelines:
  - pipeline: SmartHotel
    source: SmartHotel-CI 
    trigger: 
      branches:
      - releases/*
      - master
 ```
 You can specify the full name of the branch (for example, master) or a prefix-matching wildcard (for example, releases/*). You cannot put a wildcard in the middle of a value. For example, releases/*2018 is invalid.
 


You can specify the branches to include and exclude.
```yaml
resources:
  pipelines:
  - pipeline: SmartHotel
    source: SmartHotel-CI 
    trigger: 
      branches:
        include: 
        - releases/*
        exclude:
        - master
 ```
 
 
 You can specify which tags to control the triggers.
 ```yaml
resources:
  pipelines:
  - pipeline: SmartHotel
    source: SmartHotel-CI 
    trigger: 
      branches:
        include: 
        - releases/*
      tags: 
      - Production
      - Sign 
```

If you don't want to wait until all the stages of the run are completed for the `pipeline` resource. You can provide the stage to be completed to trigger you pipeline. 

```yaml
resources:
  pipelines:
  - pipeline: SmartHotel
    source: SmartHotel-CI 
    trigger: 
      branches:
        include: master
      stages: 
      - QA
 ```
 
 
### Repositories
Triggers can be set on `repository` resources defined the pipeline. See [repository resource](pipeline-resources.md#resources-repositories) for more details.

For repositories, you can set two types of triggers.
1. Repo triggers
2. PR triggers


#### Repo triggers
Whenever a commit goes to your `repository`, a new pipeline run gets triggered. 


#### Scenarios: 
- I would like to trigger my pipeline only when a commit happens on ‘releases/*’ branch of the repository.
- I would like to trigger my pipeline when a new commit happens, however, I would like to enable batching so that only one pipeline runs at a time. 
- I would like to trigger my pipeline only when a new commit goes into the file path “Repository/Web/*”.

`repository` resource is used when you have to build the code residing in multiple repositories or you need set of deployable files from another repo. These scenarios would require triggers to be enabled by default and any new change to your repo will trigger a new pipeline run automatically. 

However, triggers are not enabled on `repository` resource today. So, we will keep the current behavior and in the next version of YAML we will enable the triggers by default.

#### Schema
```yaml
resources:       
  repositories:
  - repository: string    
    type: enum 
    connection: string 
    source: string  
    trigger:  # Optional; Triggers are enabled by default
      batch: boolean 
      branches:  # branch conditions to filter the events, optional; Defaults to all branches.
        include: [ string ]  # branches to consider the trigger events, optional; Defaults to all branches.
        exclude: [ string ]  # branches to discard the trigger events, optional; Defaults to none.  
      paths:
        include: [ string ]  # file paths to consider the trigger events, optional; Defaults to all paths.
        exclude: [ string ]  # file paths to discard the trigger events, optional; Defaults to none.  
      tags: # filter on the events when tags are added to the commit
        include: [string] # tags on commits to consider the trigger events, optional. 
        exclude: [string] # tags on commit to discard the trigger events, optional.
```

#### Examples : Repo triggers

You can control which branches to get triggers with simple syntax.
```yaml
resources:         
  repositories:
  - repository: myPHPApp      
    type: GitHub
    connection: myGitHubConnection
    source: ashokirla/phpApp
    trigger:
      branches:
      - master
      - releases/*
```

You can specify branches and paths to include and exclude.
```yaml
resources:         
  repositories:
  - repository: myPHPApp      
    type: GitHub
    connection: myGitHubConnection
    source: ashokirla/phpApp
    trigger:
      branches:
        include:
        - features/*
        exclude:
        - features/experimental/*
      paths:
        exclude:
        - README.md
```


If you have a lot of team members uploading changes often, then you might want to reduce the number of builds you're running. If you set batch to true, when a build is running, the system waits until the build is completed, then queues another build of all changes that have not yet been built.

```yaml
resources:         
  repositories:
  - repository: myPHPApp      
    type: GitHub
    connection: myGitHubConnection
    source: ashokirla/phpApp
    trigger:
      batch: true
      branches:
      - master
```


YAML pipelines can be triggered when tags are added to a commit. This is valuable for teams whose workflows include tags. For instance, you can kick off a process when a commit is tagged as the "last known good".
You can specify which tags to include and exclude. 

```yaml
resources:         
  repositories:
  - repository: myPHPApp      
    type: GitHub
    connection: myGitHubConnection
    source: ashokirla/phpApp
    trigger:
      tags:
        include:
        - releases/*
        exclude:
        - releases/old*
```


#### PR triggers
Whenever a PR is raised on the repository, you can choose to trigger your pipeline using PR triggers. PR triggers are not enabled by default. You can enable PR triggers on the repository by defining `pr` trigger on the `repository` resource.


#### Scenarios
- I would like to trigger my pipeline only when a PR is targeted to `releases/*` branch of the repository.
- I would like to trigger my pipeline only when a PR is targeted to the file path `Web/*` of the repository.


#### Schema
```yaml
resources:         
  repositories:
  - repository: string    
    source: string  
    pr:        # Optional; pr triggers are disabled by default
      autoCancel: boolean  # cancel the pr triggered pipelines when the pr is updated.
      branches:  # branch conditions to filter the events, optional; Defaults to all branches.
        include: [ string ]  # branches to consider the trigger events, optional; Defaults to all branches.
        exclude: [ string ]  # branches to discard the trigger events, optional; Defaults to none.  
      paths:
        include: [ string ]  # file paths to consider the trigger events, optional; Defaults to all branches.
        exclude: [ string ]  # file paths to discard the trigger events, optional; Defaults to none.
```

#### Examples: PR triggers
Unless you specify, `pr` triggers are disabled for your repository. You can enable pull request based pipeline runs.
You can control the target branches for your pull request based pipeline runs by simple syntax.

```yaml
resources:         
  repositories:
  - repository: myPHPApp      
    type: GitHub
    connection: myGitHubConnection
    source: ashokirla/phpApp
    pr: 
      branches:
      - master
      - releases/*
```

You can specify the branches and file paths to include and exclude.
```yaml
resources:         
  repositories:
  - repository: myPHPApp      
    type: GitHub
    connection: myGitHubConnection
    source: ashokirla/phpApp
    pr:
      branches:
        include:
        - releases/*
        exclude:
        - releases/experimental/*
      paths:
        include:
        - web/*         
        exclude:
        - web/README.md
```


You can auto cancel an existing pipeline when a pull request is updated. By default, pipelines triggered by pull requests (PRs) will be canceled if a new commit is pushed to the same PR. This is desirable in most cases since usually you don't want to continue running a pipeline on out-of-date code. If you don't want this behavior, you can add autoCancel: false to your PR trigger.

```yaml
resources:         
  repositories:
  - repository: myPHPApp      
    type: GitHub
    connection: myGitHubConnection
    source: ashokirla/phpApp
    pr: 
      autoCancel: false
      branches:
      - master
 ```    

### Containers
Whenever a new image got published to the container registry, your pipeline run will be triggered automatically. See [container resource](pipeline-resources.md#resources-containers) for more details.


#### Scenarios
- I would like to trigger my pipeline whenever a new version of my application image got published so that I can deploy the image as part of my pipeline.
- I would like to trigger my pipeline whenever a new image got published to ‘East-US’ location (ACR specific filter).

`container` resource is used in a pipeline when you need an image from a registry to be deployed as part of your pipeline. The scenarios above would require triggers to be enabled by default.   

However, triggers are not enabled on `container` resource today. So, we will keep the current behavior. In the next version of YAML we will enable the triggers by default.

#### Schema - generic container resource

```yaml
resources:          
  containers:
  - container: string       
    connection: string 
    image: string # container image name, Tag/Digest is optional; defaults to latest image
    trigger: # Optional; Triggers are enabled by default
      tags:
        include: [ string ]  # image tags to consider the trigger events, optional; defaults to any new tag
        exclude: [ string ]  # image tags on discard the trigger events, option; defaults to none
```

#### Examples

You can specify the image tag pattern to get the trigger.
```yaml
resources:         
  containers:
  - container: smartHotel 
    connection: myDockerRegistry
    image: smartHotelApp 
    trigger:
      tags:
      - version-*
```
When a new image gets published which matches the pattern (say version-02), the pipeline gets triggered. 


You can specify the image tags to include and exclude.
```yaml
resources:         
  containers:
  - container: smartHotel 
    connection: myDockerRegistry
    image: smartHotelApp 
    trigger:
      tags:
        include:
        - version-*
        exclude:
        - version-2017*
```

If you are using ACR container resource, you can configure triggers based on the geo-location the image got published.
#### Schema - ACR container resource

```yaml
resources:          
  containers:
  - container: string       
    type: ACR  
    connection: string 
    image: string # container image name, Tag/Digest is optional; defaults to latest image
    trigger: # Optional; Triggers are enabled by default
      tags:
        include: [ string ]  # image tags to consider the trigger events, optional; defaults to any new tag
        exclude: [ string ]  # image tags on discard the trigger events, option; defaults to none
      location:
        include: [ string ]  # the image publish location to consider the trigger event, optional; defaults to any
        exclude: [ string ]  # the image publish location to discard the trigger event, optional; defaults to none
```

#### example 

```yaml
repositories:
  containers:
  - container: MyACR  
    type: ACR
    subscription: RMPM
    registry: contosodemo
    image: Microsoft/alphaworz
    trigger: 
      tags:
        include: 
        - production*
      location: 
      - east-US
      - west-EU
```
When a new 'production' image gets published to east-US or west-US geo locations, a new pipeline gets triggered.

### Rules for evaluation of resource triggers.
Based on the trigger defined on each resource, a new pipeline run gets triggered whenever an event is received. The branch of the self repo from which the YAML definition will be picked is based on the following rules:
- If the `pipeline` resource is from the same repo as the current pipeline, we will follow the same branch on which the `pipeline` resource event is raised.

For example, lets say there is an Azure pipeline 'SmartHotel.CI' from 'SmartHotelsRepo'. And 'SmartHotel.CI' is added as a `pipeline` resource for another Azure pipeline 'SmartHotel.CD' which is also from the same repo. Lets say a new pipeline run is completed for 'SmartHotel.CI' on 'releases/M145' branch. Now, a new pipeline run gets triggered for 'SmartHotel.CD' by picking the YAML from 'releases/M145' branch.

- For all the other resources i.e. `repository` or `container` or if the `pipeline` resource is from a repo different from current YAML pipeline, then the pipeline run is triggered from default branch of the pipeline which is same as the default branch of the repo.

For example, lets say there is a 'HelmRepo' added as a `repository` resource to the current pipeline 'SmartHotel.CD' which runs on 'SmartHotelsRepo'. Lets say a new commit goes into the 'releases/M145' branch of 'HelmRepo'. Now, a new pipeline run gets triggered for 'SmartHotel.CD' by picking the YAML from default branch (say master) set on the pipeline.

## Webhook triggers
Webhook based triggers allow users to subscribe to external events and enable pipeline triggers as part of their pipeline yaml definition. 

Webhooks are simple HTTP callback requests and you can define a webhook event based on any http event and define the target to receive the event using the payload url. You can define your webhook based on a repo commit, pr comment, registry update or simple http post request. 

Webhook triggers are slightly different from other resource based triggers. There is no downloadable artifact component or version associated for each event or there is no traceability. However, webhook events contain JSON payload data that can be used for basic analysis of the event.

With webhook triggers feature, we are providing a way to subscribe to such events(webhooks) and enable pipeline triggers and cosume the payload data.

#### Scenarios
- I would like to configure my pipeline to trigger based on an external event.
- I would like to apply some additional filters on the payload I get from external event and trigger my pipeline.
-	As part of the triggered pipeline, I would like to consume the JSON payload available as part of the event in my jobs.

#### Proposal
We will introduce a new service connection type called `Incoming Webhook`. Following are steps to create an `Incoming Webhook` service connection.
1. Go to the external service, create the webhook and give a name.
2. Provide a secret for the webhook (We recommend using the secret every time you use webhooks).
3. Provide your ADO url as the payload url for the webhook.
4. Now go to ADO service connections page and create an `Incoming Webhook` service connection. 
5. Provide the name of the webhook created in the external service.
6. Provide the secret used. (The secret will be used to validate the checksum and avoid DOS attacks.)

Once the service connection is created, you can use it to subscribe to the webhook event in your YAML pipeline. 
As part of the pipeline, you can choose to further the filter the JSON payload you get as part of the webhook and define conditions on the JSON path to trigger your pipeline. If you would like to consume the payload data as part of your jobs, you can define a variable and assign the JSON path. We extract the value for the JSON path provided and assign the value to the variable defined and make it available in the jobs.

#### Schema
```yaml
resources:       
  webhooks:
  - webhook: string # identifier for the webhook
    connection: string # incoming webhook service connection 
    filters:  # JSON paths to filter the payload and define the target value.
    variables: # Define the variable and assign the JSON path so that the payload data can be passed to the jobs.
```
This is a generic webhook trigger where user has to take care of manually creating the webhook in the external service and subscribe to it in ADO. `webhoooks` is an extensible category. Anyone can build a custom extension what automatically configures triggers and define it as a new type in `webhooks`.

#### Example
```yaml
resources:       
  webhooks:
  - webhook: MyWebHook
    connection: BookStoreIncomingWebHook
    filters:  
      "$.store.book[0].title": "TrainYourPets"
    variables:
      price: "$.store.book[0].Price"
```
In this case the pipeline will be triggered when a book is published to the BookStore and if the payload contains the book tile as 'TrainYourPets'. And the a variable $(Resources.WebHooks.MyWebHook.price), gives price of the book and is made available to the jobs.

Note: Incase you are using Quotes ('', "") in JSON path, you need to escape them. 

## Schedule triggers
This is out of scope for this iteration.

