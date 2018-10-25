# "Each" Template Expression

## Scenarios

### Scenario: Wrap steps

For example, consider the following template:

```yaml
parameters:
  jobs: []

jobs:
- ${{ each job in parameters.jobs }}: # Each job
  - job: ${{ job.job }}               # Copy the job properties
    displayName: ${{ job.displayName }}
    ${{ if job.dependsOn }}:
      dependsOn: ${{ job.dependsOn }}
    ${{ if job.condition }}:
      condition: ${{ job.condition }}
    ${{ if job.strategy }}:
      strategy: ${{ job.strategy }}
    ${{ if job.continueOnError }}:
      continueOnError: ${{ job.continueOnError }}
    ${{ if job.timeoutInMinutes }}:
      timeoutInMinutes: ${{ job.timeoutInMinutes }}
    ${{ if job.cancelTimeoutInMinutes }}:
      cancelTimeoutInMinutes: ${{ job.cancelTimeoutInMinutes }}
    ${{ if job.container }}:
      container: ${{ job.container }}
    ${{ if job.variables }}:
      variables: ${{ job.variables }}
    ${{ if job.pool }}:
      pool: ${{ job.pool }}
    ${{ if job.workspace }}:
      workspace: ${{ job.workspace }}
    steps:                          # Inject steps
      - task: SetupMyBuildTools@1   # Pre steps
      - ${{ job.steps }}            # Users steps
      - task: PublishMyTelemetry@1  # Post steps
        condition: always()
```

We can shorten the example above, by leveraging an `each` expression to iterate over the job mapping itself:

```yaml
parameters:
  jobs: []

jobs:
- ${{ each job in parameters.jobs }}: # Each job
  - ${{ each pair in job }}:          # Insert all properties other than "steps"
      ${{ if ne(pair.key, 'steps') }}:
        ${{ pair.key }}: ${{ pair.value }}
    steps:                            # Wrap the steps
    - task: SetupMyBuildTools@1       # Pre steps
    - ${{ job.steps }}                # Users steps
    - task: PublishMyTelemetry@1      # Post steps
      condition: always()
```

### Scenario: Wrap jobs

For example, consider the following template:

```yaml
parameters:
  jobs: []

jobs:
- job: CredScan                       # Cred scan first
  pool: MyCredScanPool
  steps:
  - task: MyCredScanTask@1
- ${{ each job in parameters.jobs }}: # Then each job
  - job: ${{ job.job }}
    dependsOn:                        # Inject dependency
      - CredScan
      - ${{ if job.dependsOn }}:
        - ${{ job.dependsOn }}
    ${{ if job.displayName }}:        # Copy all other job properties
      displayName: ${{ job.displayName }}
    ${{ if job.condition }}:
      condition: ${{ job.condition }}
    ${{ if job.strategy }}:
      strategy: ${{ job.strategy }}
    ${{ if job.continueOnError }}:
      continueOnError: ${{ job.continueOnError }}
    ${{ if job.timeoutInMinutes }}:
      timeoutInMinutes: ${{ job.timeoutInMinutes }}
    ${{ if job.cancelTimeoutInMinutes }}:
      cancelTimeoutInMinutes: ${{ job.cancelTimeoutInMinutes }}
    ${{ if job.container }}:
      container: ${{ job.container }}
    ${{ if job.variables }}:
      variables: ${{ job.variables }}
    ${{ if job.pool }}:
      pool: ${{ job.pool }}
    ${{ if job.workspace }}:
      workspace: ${{ job.workspace }}
    ${{ if job.steps }}:
      steps: ${{ job.steps }}
```

We can shorten the example above, by leveraging an `each` expression to iterate over the job mapping itself:

```yaml
parameters:
  jobs: []

jobs:
- job: CredScan                       # Cred scan first
  pool: MyCredScanPool
  steps:
  - task: MyCredScanTask@1
- ${{ each job in parameters.jobs }}: # Then each job
  - ${{ each pair in job }}:          # Insert all properties other than "dependsOn"
      ${{ if ne(pair.key, 'dependsOn') }}:
        ${{ pair.key }}: ${{ pair.value }}
    dependsOn:                        # Inject dependency
    - CredScan
    - ${{if job.dependsOn}}:            
      - ${{ job.dependsOn }}
```

## Syntax details

### Iterative sequence insertion

Scenarios are listed above.

Technical syntax example:

```yaml
mySequence:
- outer pre
- ${{ each myItem in parameters.myCollection }}:
  - nested pre
  - ${{ myItem }}
  - nested post
- outer post
```

## Iterative mapping insertion

A real scenario is detailed above. See the shortened example, for the first scenario.

Technical syntax example:

```yaml
parameters:
  myCollection:
  - key: myKey1
    value: my value 1
  - key: myKey2
    value: my value 2
myMapping:
  outer pre: abc
  ${{ each myItem in parameters.myCollection }}:    # Each key-value pair in the mapping
    pre_${{ myItem.key }}: pre ${{ myItem.value }}
    ${{ myItem.key }}: ${{ myItem.Value }}
  outer post: def
```
