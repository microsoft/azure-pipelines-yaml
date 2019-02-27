# Process parameters

We need a way to take queue-time inputs, giving pipeline authors the right amount of control / expressivity.
YAML pipelines already accept parameters when used as part of a template.
Process parameters are a natural evolution of that syntax.

Scenarios:
- Pipeline users can supply different values to tools and tasks at queue time
- Pipeline authors can control the types, ranges allowed, and defaults for queue-time parameters
- Pipeline authors can allow flexible, queue-time control over stages/jobs to run, including adding/removing matrix legs
- The pipeline itself can be altered at queue time: pools, agentspecs, service connections, environments, and more

## Syntax

```yaml
parameters:
# this is a sequence, so the block may be repeated as many times as needed
- name: string          # name of the parameter; required
  displayName: string   # UI string
  displayHint: enum     # see below
  type: enum            # data type; see below
  default: any          # default value if nothing specified
  queueTime: boolean    # whether the value can be set at queue time; defaults to false
  values: [ string ]    # allowed list of values (for some data types)
```

See [below](#back-compat-with-current-syntax) for details about how this coexists with existing `parameter` syntax.

## Data types

Available data types:

| Data type | Notes |
|-----------|-------|
| `string` | default data type if none is specified
| `number` | can be restricted to `values:`
| `boolean`
| `object` | YAML serialization expected
| `enum` | can be restricted to `values:`
| `filePath`
| `secureFile`
| `pool`
| `serviceConnection`
| `environment`

It's also useful to tighten up the editor experience and error messages for template parameters.
Customers are already using templates to pass along step and job lists.
These aren't very useful for queue-time parameters: it's unclear how a user would be expected to set them.
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

## Full examples

**TODO**
 
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

Since existing parameter syntax is only for templates, we assume none of the parameters are settable at queue time.
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
