# Variables context and condition simplification

## Processing within a file

Changes:
- Add variables to the context, as they are read (completed)
- Add `else` and `elif` (not completed)

Example jobs template:

```yaml
parameters:
  jobName: ''
jobs:
- job: ${{ parameters.jobName }}
  variables:
    public: ${{ eq(variables['System.TeamProject'], 'public') }}
    publicOrPR: ${{ or(eq(variables.public, 'true'), eq(variables['Build.Reason'], 'PullRequest')) }}
  steps:
  - ${{ if eq(variables.public, 'true') }}:
    - script: ./setup-internal-tools.sh
  - ${{ else }}:
    - script: ./setup-tools.sh
  - script: build
  - script: test
  - ${{ if eq(variables.publicOrPR, 'true') }}:
    - script: publish-telemetry
```

Note, attempts to override system variables will fail.

Note, although variables are added to the context as the are read, they do not flow downstream across files during template compilation.

## Reuse variable definitions

Allow variables to be imported from a file.

Variables can be imported wherever they can normally be defined. That is, at the root of the pipeline, on a stage, or on a job.

For example:

```yaml
# my-variables.yml

parameters:
  config: debug
variables:
  arch: x64
  config: ${{ parameters.config }}
  sign: ${{ eq(parameters.config, 'debug') }}
  publish: ${{ or(eq(parameters.config, 'release'), startsWith(variables['build.sourceBranch'], 'refs/heads/dogfood/')) }}
```

```yaml
# my-jobs.yml

parameters:
  jobName: ''
jobs:
- job: ${{ parameters.jobName }}
  variables:
  - template: my-variables.yml # reference variables
    parameters:
      config: release
  steps:
  - script: build --config ${{ variables.config }} --arch ${{ variables.arch }}
  - ${{ if eq(variables.sign, 'true') }}:
    - script: sign
  - ${{ if eq(variables.publish, 'true') }}:
    - script: publish
```
