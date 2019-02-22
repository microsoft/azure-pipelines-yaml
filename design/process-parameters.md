# Process parameters

We need a way to take queue-time inputs, giving pipeline authors the right amount of control / expressivity.

Scenarios:
- Pipeline users can supply different values to tools and tasks at queue time
- Pipeline authors can control the types, ranges allowed, and defaults for queue-time parameters
- Pipeline authors can allow flexible, queue-time control over stages/jobs to run, including adding/removing matrix legs
- The pipeline itself can be altered at queue time: pools, agentspecs, service connections, environments, and more

## Schema

_Note: we need to rationalize this with the existing `parameters` syntax._

```yaml
parameters:
# this is a sequence, so the block may be repeated as many times as needed
- name: string          # name of the parameter; required
  displayName: string   # UI string
  displayHint: enum     # see below
  type: enum            # data type; see below
  default: any          # default value if nothing specified
  range: any            # allowed values; varies by data type so see below
```

## Data types

Allowed data types and their range expressions:

| Data type | Range expression | Notes |
|-----------|------------------|-------|
| `string` | positive integer: maximum length | default data type if none is specified
| `number` | `range({min}, {max})` | example: `range(1, 10)`
| `boolean` | n/a
| `filePath` | n/a
| `secureFile` | n/a
| `enum` | comma-separated list of allowed strings | example: `Red,Green,Blue`
| `pool` | comma-separated list of allowed pools | example: `default,Azure Pipelines,PrivatePool`
| `serviceConnection` | _TODO_ | _TODO: can we wire this up to a query somehow rather than forcing the author to know in advance? or maybe only limit by SC types?_
| `environment` | _TODO_ | _TODO: same question as above_

- _TODO: is there a use case for regex as a range for strings?_
- _TODO: is there a use case for floats/decimals?_

## Display hints

Optionally, the pipeline author can hint to the UI what widget should be displayed.
Not all widgets are compatible with all data types, and UI may evolve over time, so these are only hints.

| Display hint | Compatible data types | Default for |
|--------------|-----------------------|-------------|
| `oneline` | string, number, filePath | string, number
| `multiline` | string | -
| `pickList` | number, boolean, enum, pool, serviceConnection, environment | enum
| `checkbox` | boolean | boolean
| `radio` | number, boolean, enum | -
| `filePicker` | filePath | filePath
| `specialPicker` | pool, serviceConnection, environment, secureFile | pool, serviceConnection, environment, secureFile

`specialPicker` varies per resource type, allowing a very targeted experience for well-known resources.

## Full examples

**TODO**
 
