# Templates

## First class reuse

### Problems solved

- What does \"build\" mean to me?
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

Downside: sometimes referenced as a string, sometimes a mapping

Note, we could also support an alternative syntax, like:

```yaml
- build: --config release
```

However, that get's us into arg parsing (yet another layer of escaping problems). Also it's limited because objects can't be passed. So an alternative parameter-mapping-style would still need to be supported.

### Other first-class types

Stages and jobs are two other natural fits for first-class reuse.

What about pools? variables? The same syntax does not apply nicely to pools and variables. Which is probably OK, since steps and jobs are the main reuse scenario (optimized for).

### Reuse across files?

Something like this:

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

## Generic reuse

The above first-class reuse is great for the main scenarios. But what about everything else? For example, pools.

Let's optimize for the main scenarios (steps, jobs, stages). And let's make the other things possible.

The follow example solves two generic problems:
- Arbitrary object templates
- Reusable expression values

```yaml
define:

# Custom expression values
- ${{ if eq(variables['System.TeamProject'], 'public') }}:
  - expressionValue: public
    result: true
- ${{ if or(eq(variables['System.TeamProject'], 'public'), in(variables['Build.Reason'], 'PullRequest')) }}:
  - expressionValue: publicOrPR
    result: true

# Init steps
- steps: init
  result:
  - ${{ if not(values.public) }}:
    - script: init-internal-tools.cmd

# Cleanup steps
- steps: cleanup
  result:
  - ${{ if not(values.publicOrPR) }}:
    - script: publish-telemetry.cmd

# Pool
- templateValue: windows
  parameters:
    version: 10
  result:
    name: myWindowsPool
    demands: windows${{ parameters.version }}

jobs:
- job: win10
  pool:
    ${{ apply }}:
      name: windows
  steps:
  - init
  - script: build
  - cleanup

- job: win8
  pool:
    ${{ apply }}:
      name: windows
      version: 8
  steps:
  - init
  - script: build
  - cleanup
```
