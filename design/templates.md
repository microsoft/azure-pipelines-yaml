# Templates

## First class reuse

### Problems solved

- What \"build\" means to me
- Inline re-use (external file not required)

### Inline template syntax

```yaml
define:

# Define "build"
- steps: build
  parameters:
    config: debug
  result:
  - script: restore my-solution.sln
  - script: compile my-solution.sln config=${{ parameters.config }}

# Define "test"
- steps: test
  result:
  - script: test my-solution.sln

jobs:

# Linux (debug)
- job: linux
  pool: my-linux-pool
  steps:
  - build                   # No parameters overridden
  - test

# Windows (debug)
- job: windows_debug
  pool: my-windows-pool
  steps:
  - build
  - test

# Windows (release)
- job: windows_release
  pool: my-windows-pool
  steps:
  - build:
      config: release       # Override 'config'
  - test
```

Downside: sometimes referenced as a string, sometimes a mapping.

Note, we could also support an alternative syntax, like:

```yaml
- build: --config release
```

However, there are two problems with that approach:
- It get's us into arg parsing - yet another layer of escaping problems
- It's limited because objects can't be passed. So an alternative parameter-mapping-style would still need to be supported - i.e. multiple ways of doing the same thing

### Reuse across files

Explicit import:

```yaml
#
# azure-pipelines.yml
#

# Import templates
import:
- my-templates.yml

# Build and test
steps:
- build:
    parameters: my-solution.sln
- test:
    parameters: my-solution.sln



#
# my-templates.yml
#

define:

# Define "build"
- steps: build
  parameters:
    solution: "*.sln"
    config: debug
  result:
  - script: restore ${{ parameters.solution }}
  - script: compile ${{ parameters.solution }} config=${{ parameters.config }}

# Define "test"
- steps: test
  parameters:
    solution: '*.sln'
  result:
  - script: test ${{ parameters.solution }}
```

Additional things to figure out:
- Support multiple levels of import? That is, import a template, which in-turn imports a template?
  - If so, what gets exported? Everything? Only local things? Is there an export keyword? The approach powershell takes is, everything is exported by default unless you explicitly specify an export.

### Other first-class types

`stages` and `jobs` are two other natural fits for first-class reuse. We will support those as well.

## Generic re-use

The above first-class syntax does not fit well for pools, variables, or any random element in the DOM.

Although we want to optimize for the main scenarios (`steps`, `jobs`, `stages`), we should make other scenarios possible.

### Re-usable values

The follow example solves two generic problems:
- Arbitrary object re-use
- Reusable expression values

```yaml
define:

# Custom values
- ${{ if eq(variables['System.TeamProject'], 'public') }}:
  - value: public
    result: true
- ${{ if or(eq(variables['System.TeamProject'], 'public'), in(variables['Build.Reason'], 'PullRequest')) }}:
  - value: publicOrPR
    result: true
- value: win10
  result:
    name: myWindowsPool
    demands: windows10
- value: win8
  result:
    name: myWindowsPool
    demands: windows8

# Init steps
- steps: init
  result:
  - ${{ if not(values.public) }}:     # References a custom value from above, within an if-expression
    - script: init-internal-tools.cmd

# Cleanup steps
- steps: cleanup
  result:
  - ${{ if not(values.publicOrPR) }}: # References a custom value from above, within an if-expression
    - script: publish-telemetry.cmd

jobs:
- job: a
  pool: ${{ values.win10 }}           # Inserts the custom object
  steps:
  - init
  - script: build
  - cleanup

- job: b
  pool: ${{ values.win8 }}            # Inserts the custom object
  steps:
  - init
  - script: build
  - cleanup
```

### Partial templates

Partial templates allow arbitrary object templates, that accept parameters.

We may not need to do this. Steps, jobs, stages are the more common scenarios for parameters.

```yaml
define:

# Custom expression values
# Init steps
- steps: init
  result:
  - script: init-internal-tools.cmd

# Cleanup steps
- steps: cleanup
  result:
  - script: publish-telemetry.cmd

# Pool
- partial: windows
  parameters:
    version: 10
  result:
    name: myWindowsPool
    demands: windows${{ parameters.version }}

jobs:
- job: win10
  pool: ${{ apply-partial 'windows' }}
  steps:
  - init
  - script: build
  - cleanup

- job: win8
  pool:
    ${{ apply-partial 'windows' }}:
      version: 8
  steps:
  - init
  - script: build
  - cleanup
```
