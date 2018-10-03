# Reusable "values" expression context

## Scenario: Common, simple calculations

Consider the below reusable values file. The file defines commonly calculated, simple values.

```yaml
# my-values.yml

values:
  ${{ if eq(variables['System.TeamProject'], 'public') }}:
    public: true
  ${{ if or(eq(variables['System.TeamProject'], 'public'), in(variables['Build.Reason'], 'PullRequest')) }}:
    publicOrPR: true
```

Now consider the below pipeline definition. The common values are imported and used within the definition.

```yaml
# pipeline.yml

import:
- my-values.yml # import commonly calculated values
steps:
- ${{ if not(values.public) }}:
  - setup internal tools
- script: build
- script: test
- ${{ if not(values.publicOrPR) }}:
  - script: publish telemetry
```

## Scenario: Common objects

Consider the reusable pool definitions:

```yaml
# my-values.yml

values:
  # Pools
  windowsCI:
    name: myPool
    demands:
    - agent.os -equals Windows_NT
    - CI -equals true
  windowsPR:
    name: myPool
    demands:
    - agent.os -equals Windows_NT
    - PR -equals true
```

Now consider the pipeline definition:

```yaml
# pipeline.yml

import:
- my-values.yml
pool: ${{ values.windowsCI }}
steps:
- script: build.cmd
- script: test.cmd
```

## Types: String, array, dictionary

All values are string, array, or dictionary.

This is important when reusing values within an if-expression, because values are truthy.

For example, consider the follow values:

```yaml
# my-values.yml

values:
  ${{ if eq(variables['System.TeamProject'], 'public') }}:
    public: true
```

When used within an if-expression:

```yaml
# pipeline.yml

import:
- my-values.yml
steps:
- ${{ if eq('true', values.public) }} # when "public" is defined, it is the string "true"
  - script: echo hi public

- ${{ if values.public }} # truthy
  - script: echo hi public
``` 