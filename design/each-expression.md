# "Each" Template Expression

## Scenarios

### Scenario: Wrap steps

For example, consider the following template:

```yaml
parameters:
  jobs: []

jobs:
- ${{ each job in parameters.jobs }} # Each job
  - job: ${{ job.job }}
    displayName: ${{ job.displayName }}
    dependsOn: ${{ job.dependsOn }}
    condition: ${{ job.condition }}
    strategy: ${{ job.strategy }}
    continueOnError: ${{ job.continueOnError }}
    timeoutInMinutes: ${{ job.timeoutInMinutes }}
    cancelTimeoutInMinutes: ${{ job.cancelTimeoutInMinutes }}
    container: ${{ job.container }}
    variables: ${{ job.variables }}
    pool: ${{ job.pool }}
    workspace: ${{ job.workspace }}
    steps:
      - task: SetupMyBuildTools@1  # Pre steps
      - ${{ job.steps }}           # Users steps
      - task: PublishMyTelemetry@1 # Post steps
        condition: always()
```

### Scenario: Wrap jobs

For example, consider the following template:

```yaml
parameters:
  jobs: []

jobs:
- job: CredScan                      # Cred scan first
  pool: MyCredScanPool
  steps:
  - task: MyCredScanTask@1
- ${{ each job in parameters.jobs }} # Then each job
  - job: ${{ job.job }}
    displayName: ${{ job.displayName }}
    ${{ if job.dependsOn }}:         # Inject dependency
      dependsOn:
      - ${{ CredScan }}
      - ${{ job.dependsOn }}
    ${{ if not(job.dependsOn) }}:
      dependsOn:
      - CredScan
    condition: ${{ job.condition }}
    strategy: ${{ job.strategy }}
    continueOnError: ${{ job.continueOnError }}
    timeoutInMinutes: ${{ job.timeoutInMinutes }}
    cancelTimeoutInMinutes: ${{ job.cancelTimeoutInMinutes }}
    container: ${{ job.container }}
    variables: ${{ job.variables }}
    pool: ${{ job.pool }}
    workspace: ${{ job.workspace }}
    steps: ${{ job.steps }}
```

## Syntax details

### Iterative sequence insertion

Customer scenarios are listed above.

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

Actual customer scenarios?

Technical syntax example:

```yaml
parameters:
  myCollection:
  - key: myKey1
    value: my value 1
  - key: myKey2
    value: my value 3
  - key: myKey2
    value: my value 3
myMapping:
  outer pre: abc
  ${{ each myItem in parameters.myCollection }}:
    pre_${{ myItem.key }}: pre ${{ myItem.value }}
    ${{ myItem.key }}: ${{ myItem.Value }}
    post_${{ myItem.key }}: post ${{ myItem.value }}
  outer post: def
```
