# Runtime parameters

We need a way to take run-time inputs, giving pipeline authors the right amount of control / expressivity.
YAML pipelines already accept parameters when used as part of a template.
Runtime parameters are a natural evolution of that syntax.
Along the way, we can also augment the capabilities of template parameters in a natural way.

Scenarios:
- Pipeline users can supply different values to tools and tasks at run time
- Pipeline authors can control the types, ranges allowed, and defaults for run-time parameters
- Pipeline authors can allow flexible, run-time control over stages/jobs to run, including adding/removing matrix legs
- The pipeline itself can be altered at run time: pools, agentspecs, service connections, environments, and more
- The runtime values can be used in template expressions (`${{ if }}`, for example), for instance so that jobs/steps can be dynamically selected

## Syntax

```yaml
parameters:
# this is a sequence, so the block may be repeated as many times as needed
- name: string          # name of the parameter; required
  displayName: string   # UI string
  displayHint: enum     # see below
  type: enum            # data type; see below
  default: any          # default value; if no default, then the parameter MUST be given by the user at runtime
  values: [ string ]    # allowed list of values (for some data types)
```

See [below](#back-compat-with-current-syntax) for details about how this coexists with existing `parameters` syntax.

## User interactions

### Run panel ("queue dialog")

_TODO_

### Comment triggers

_TODO_

### Editor experiences

We can teach the editors (web, VS Code) how templates fit together.
When a user's pipeline consumes a template, we can offer IntelliSense for the names and allowed values of parameters.
*This is out of scope for v1.*

## Data types

### Data types for runtime parameters

| Data type | Notes |
|-----------|-------|
| `string` | default data type if none is specified. if `values:` is specified, they become suggestions but **not** required (like a combobox)
| `number` | may be restricted to `values:`, otherwise any number-like string is accepted
| `boolean`
| `object` | YAML serialization expected
| `enum` | must be restricted to `values:` (like a selectbox)
| `filePath`
| `secureFile`
| `pool`
| `serviceConnection`
| `environment`

### Data types for template parameters

It's also useful to tighten up the editor experience and error messages for template parameters.
Customers are already using templates to pass along step and job lists.
These aren't particularly useful for run-time parameters.
They're rendered like a plain `object` in the UI.

| YAML string type | Notes |
|------------------|-------|
| `steps` | sequence of steps
| `jobs` | sequence of jobs
| `stages` | sequence of stages

## Display hints

Optionally, the pipeline author can hint to the UI what widget should be displayed.
Not all widgets are compatible with all data types, and UI may evolve over time, so these are only hints.

| Display hint | Compatible data types | Default for |
|--------------|-----------------------|-------------|
| `oneline` | string, number | string, number
| `multiline` | string, object | object
| `pickList` | number, boolean, enum | enum
| `checkbox` | boolean | boolean
| `radio` | number, boolean, enum | -

For data types not listed in the above table, there is a "special" UI (such as a file picker, pool picker, etc.) that's automatically used.

*This part can be postponed. It's nice-to-have, especially for cases like single line vs multiline input.*

## Full examples

### Dial in the amount of parallelism

```yaml
parameters:
- name: parallelism
  displayName: How much parallelism
  type: number
  default: 5
  values:
  - 1
  - 5
  - 10

job: Test
strategy:
  parallel: ${{ parameters.parallelism }}
steps:
- script: echo Running one leg of tests...
```
 
### Select configs to run

```yaml
parameters:
- name: configs
  type: string
  default: x86,x64

jobs:
- ${{ if contains(parameters.configs, 'x86') }}:
  - job: x86
    steps:
    - script: echo Building x86...
- ${{ if contains(parameters.configs, 'x64') }}:
  - job: x64
    steps:
    - script: echo Building x64...
- ${{ if contains(parameters.configs, 'arm') }}:
  - job: arm
    steps:
    - script: echo Building arm...
```

*There's a possible future enhancement hiding here: a data type of "flags" which would be like an enum but allow multiple specification.
This version is more verbose but also clear and straightforward.*

### Select your pool

```yaml
parameters:
- name: pool
  type: pool
  default: ubuntu-16.04

job: Build
pool:
  vmImage: ${{ parameters.pool }}
steps:
- script: echo Building on the pool of your choice
```

### Connecting template parameters with strong typing

This is common for sophisticated customers such as .NET.
_Note: this isn't about runtime and UI-specified parameters as such, but it's an important capability of parameters._

```yaml
# template.yml - comes from the centralized infrastructure team

parameters:
- name: customerSteps
  type: steps

steps:
- script: echo Pre-step injected on all jobs
- ${{ each customerStep in parameters.customerSteps }}:
  - ${{ customerStep }}
- script: echo Post-step injected on all jobs
```

When a client repo such as Roslyn goes to consume this template:

```yaml
# azure-pipelines.yml - in the Roslyn repo

resources:
  repositories:
  - repository: arcade
    name: dotnet/arcade

steps:
- template: template.yml@arcade
  parameters:
    customerSteps:
    - task: Foo@1
    - script: echo bar
```

At runtime, we know that `customerSteps` must be a list of steps.
If it's not, we can give a clear error message.
In the future, our editing experiences can also make this easy to reason about.

## Back-compat with current syntax

The existing `parameters` syntax expects a map of parameters with their defaults.
Since the value of a parameter can be anything including a map, we wouldn't be able to tell whether the customer meant to use the old syntax or the new, richer syntax.
That's why the new syntax is a sequence of defined objects, allowing us to preserve back-compat.

Example of current syntax:
```yaml
parameters:
  image: 'ubuntu-16.04'   # scalar parameter
  preSteps:               # sequence parameter
  - script: echo foo
  vars:                   # map parameter
    NODE_VERSION: '8.x'
    PYTHON_VERSION: '3.7'
```

Since existing parameter syntax is only for templates, none of the parameters are settable at run time.
The above example is equivalent to:
```yaml
parameters:
- name: 'image'     # scalar parameter
  type: string
  default: 'ubuntu-16.04'
- name: 'preSteps'  # sequence parameter
  type: object
  default:
  - script: echo foo
- name: 'vars'      # map parameter
  type: object
  default:
    NODE_VERSION: '8.x'
    PYTHON_VERSION: '3.7'
```
