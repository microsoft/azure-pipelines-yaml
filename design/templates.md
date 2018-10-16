# Templates

## First class reuse

### Problems

- What does 'build' mean to me?
- Inline re-use (external file not required)

### Inline template syntax

Downsides to this approach
- Sometimes a string, sometimes a mapping
- Does not leave anywhere to solve grouping problems
  - Dependencies grouping
  - A grouping of steps that run in a container?

```yaml
templates:

- steps: build
  parameters:
    config: debug
  result:
  - script: restore my-solution.sln
  - script: compile my-solution.sln config=${{ parameters.config }}

- steps: test
  parameters:
    solution: '*.sln'
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



## Generic reuse



## Appendix

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
```