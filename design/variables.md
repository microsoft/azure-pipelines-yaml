# Variables context and condition simplification

## Processing within a file

Changes:
- Add variables to the context, as they are read
- Add `parseBool()` for clarity
- Add `else`

Example jobs template:

```yaml
parameters:
  jobName: ''
jobs:
- job: ${{ parameters.jobName }}
  variables:
    public: ${{ eq(variables['System.TeamProject'], 'public') }}
    publicOrPR: ${{ or(parseBool(public), eq(variables['Build.Reason'], 'PullRequest')) }}
  steps:
  - ${{ if parseBool(variables.public) }}:
    - script: ./setup-internal-tools.sh
  - ${{ else }}:
    - script: ./setup-tools.sh
  - script: build
  - script: test
  - ${{ if parseBool(variables.publicOrPR) }}:
    - script: publish-telemetry
```

## Reuse variable definitions

Allow variables to be imported from a file.

The limitation with this approach is that variables can only be imported where they can normally be defined. That is, at the root of the pipeline, on a stage, or on a job.

For example:

```yaml
parameters:
  jobName: ''
jobs:
- job: ${{ parameters.jobName }}
  variables:
  - template: my-variables.yml
  steps:
  - ${{ if parseBool(variables.public) }}:
    - script: ./setup-internal-tools.sh
  - ${{ else }}:
    - script: ./setup-tools.sh
  - script: build
  - script: test
  - ${{ if parseBool(variables.publicOrPR) }}:
    - script: publish-telemetry
```

## Variables flow downstream across files

If a variable is defined in one file, it flows downstream across templates. Which is consistent with how variables work.

For example:

```yaml
# entry-file.yml

variables:
  public: ${{ eq(variables['System.TeamProject'], 'public') }}
  publicOrPR: ${{ or(parseBool(public), eq(variables['Build.Reason'], 'PullRequest')) }}
jobs:
- template: my-job.yml
  parameters:
    jobName: win
    pool: hosted vs2017
- template: my-job.yml
  parameters:
    jobName: mac
    pool: hosted macos
- template: my-job.yml
  parameters:
    jobName: linux
    pool: hosted ubuntu 1604
```

```yaml
# my-job.yml

parameters:
  jobName: ''
  pool: ''
jobs:
- job: ${{ parameters.jobName }}
  pool: ${{ parameters.pool }}
  steps:
  - ${{ if parseBool(variables.public) }}:
    - script: ./setup-internal-tools.sh
  - ${{ else }}:
    - script: ./setup-tools.sh
  - script: build
  - script: test
  - ${{ if parseBool(variables.publicOrPR) }}:
    - script: publish-telemetry
```

## Other details

### Cannot override system variables

Fail if the user tries to override a system variable within their YAML file.

### `parseBool` behavior

True when the value (cast to a string), equals 'true' (case insensitive). Otherwise false.
