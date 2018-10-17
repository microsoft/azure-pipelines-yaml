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

```yaml
define:

- templateValue: windows
  parameters:
    version: 10
  result:
    name: myWindowsPool
    demands: windows${{ parameters.version }}

- ${{ if eq(variables['System.TeamProject'], 'public') }}:
  - expressionValue: isPublic
    result: true
- ${{ if or(eq(variables['System.TeamProject'], 'public'), in(variables['Build.Reason'], 'PullRequest')) }}:
  - expressionValue: publicOrPR
    result: true

jobs:
- job: win10
  pool:
    ${{ apply }}:
      name: windows
  steps:
  - script: build

- job: win8
  pool:
    ${{ apply }}:
      name: windows
      version: 8
  steps:
  - script: build
```



<!-- ## Appendix

### Inline step templates (ordinal parameters)

The problem with this approach, is that a totally different syntax is required when you need to pass complex objects.

Multiple ways of doing the same thing is not worth this optimization.

```yaml
templates:

- steps: build
  parameters:
  - solution: '*.sln'
  - config: debug
  result:
  - script: restore ${{ parameters.solution }}
  - script: compile ${{ parameters.solution }} config=${{ parameters.config }}

- steps: test
  parameters:
  - solution: '*.sln'
  result:
  - script: test ${{ parameters.solution }}

jobs:
- job: linux
  pool: my-linux-pool
  steps:
  - build: my-solution.sln
  - test: my-solution.sln

- job: windows_debug
  pool: my-windows-pool
  steps:
  - build: my-solution.sln
  - test: my-solution.sln

- job: windows_release
  pool: my-windows-pool
  steps:
  - build: my-solution.sln release
  - test: my-solution.sln
  - publish: 
```


### Thought playground:

```yaml
# namespace?


import:
- template: foo
  file: my-other-template.yml




templates:
- steps: build
  export: false
  parameters:
    name: john doe
  result:
  - script: echo hello ${{ parameters.name }}
  - ${{ if eq(parameters.name, 'jane doe' }}:
    - script: echo good luck!
- jobs:
- variables:
- stages:

- pool: windows
  parameters:
  - version: 10
  - someComplex:
      foo: abc
      bar: def
  result:
    name: helix
    os: windows
    version: ${{ parameters.version }}









jobs:
- job: a
  pool:
    ${{ apply }}:
      template: windows
      parameters:
        version: 8
        someComplex:
          #foo: def
          bar: abc
  steps:
  - script: echo hi




jobs:
- job: a
  pool: windows
  steps:
  - script: echo hi



jobs:
- job: a
  pool: ${{ templates.windows(8) }}
  steps:
  - script: echo hi







jobs:
- job: a
  steps:
  - build: 'john doe'



jobs:
- job: myJob
  pool:
    ${{ if eq(item, 'foo') }}:
      template: myPool
      parameters:
        foo:
        - my item 1
        - my item 2


jobs:
- job: myJob
  pool: ${{ templates.myPool('foo') }}


${{ templates.myPool(parameters.foo) }}



${{ import }}: my-templates.yml
${{ define }}:



# jobs:
# - job: a
#   variables: &myVars
#     foo: asdf
#     bar: 1234
# - job: b
#   variables:
#     <<: *myVars
#     baz: 5678
``` -->