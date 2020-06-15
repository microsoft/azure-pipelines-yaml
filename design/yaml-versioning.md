# YAML Versioning

**STATUS**: Cut, we will not be proceeding with YAMLv2.
Merging this PR for historical interest.

So far, our YAML schema has been "append-only".
We're careful not to make breaking changes when we add new concepts.
Also, we still accept older syntax even when we replace it with new, "better" syntax.

Several concepts now have multiple syntaxes.
Also, in several cases, we're constrained by ideas that were good at the time but haven't scaled well.
With unified pipelines, there are defaults and behaviors we wish to change, but doing so would break existing customers.

The solution is to introduce a `version` keyword on the pipeline YAML.
This will let us make breaking changes, giving customers an ability to opt-in as they're ready.
It'll also give us a clean way to make breaking changes in the future, although we hope not to proliferate versions.

## Scenarios

* `pool`s used to be called `queue`s.
We've deprecated the use of the word `queue` in customer-facing scenarios, so we would like to remove it from the YAML vocabulary.

* `checkout` used to cooperate with a `repository` resource to implement things like whether to fetch Git-LFS objects.
We cleaned up the usage pattern so that resources only describe the "nouns" in the system, moving the actions to the `checkout` step.
There are properties on the `repository` that should be removed.

* The `container` resource started as just a job runner in CI.
Under unified pipelines, it has grown to become a consumable/deployable resource.
The more common scenario will be triggering a pipeline when a new container image appears, so triggers should be enabled by default.
This would be a breaking change, so existing customers will need to signal their readiness.

* `trigger` covered only CI triggers.
When we introduced PR triggers, we made a parallel `pr` structure.
We're about to introduce `schedule` as another parallel structure.
This has confused and frustrated customers, so we want to clean up the syntax.

## Industry landscape

### Unversioned YAML
- [Travis CI](https://docs.travis-ci.com/user/reference/overview/).
- [AppVeyor](https://www.appveyor.com/docs/appveyor-yml/).
- [Google Cloud Build](https://cloud.google.com/cloud-build/docs/build-config).

### Versioned YAML
- CircleCI has [3 supported versions](https://circleci.com/docs/2.0/sample-config/#section=configuration): 1.0, 2.0, and 2.1. Mostly it looks like Circle added additional syntax in each version; I didn't find any breaking changes.
- ...

## Solution

We introduce a new field, `version`, which lives at the root level of the pipeline.
It's an integer whose valid values are (currently) only `1` and `2`.
Any other number is a syntax error.
If not present, the value is assumed to be `1` (unless included in a [template; see below](#templates)).

We'll continue our commitment to backcompat within a single version.
Whatever version is currently in active development will grow new features over time.
Down-level versions will continue to be supported for the foreseeable future, though we reserve the right to remove versions eventually.

### Version 1
Version 1 is the current set of syntax, back-compat warts and all.
No behavior changes are expected, and we'll freeze this version when v2 ships.
No new features will land in v1.

### Version 2
Version 2 becomes our active development version.
This spec doesn't attempt to list all the new things it will gain, since that's not predictable.
Instead, it lists only the breaking changes we'll make now.

#### Additions
* `version: 2` must be specified at the root of the pipeline.

#### Removals
* Remove `phases`/`phase`. This became `jobs`/`job` with minimal other changes.
* Remove `queue`. This includes all the nodes which moved under `strategy`.
* Remove Git-LFS, shallow fetch, and other non-core properties from `repository`. These are now handled by `checkout`.
* Remove sequence form of `resources`. This is completely handled by the current mapping form.
* Remove `pr` and `schedule` nodes. (See [trigger changes](#trigger-changes) for their replacements.)
* (Tentative) Remove `$[ ]` syntax, as we're planning to make `${{ }}` execute just-in-time.

#### Trigger changes
`trigger` becomes a parent of all `self`-repo triggers.
CI keywords, which used to be directly under `trigger`, now live under a sub-entry called `push`.
`pr` and `schedule`, which used to be pipeline-level, also live under `trigger`.
Example:
```yaml
# version 1 - old way
trigger:
  branches:
  - master
  - features/*
pr:
  branches:
    include:
    - master
    - releases/*

# version 2 - new way
trigger:
  push:
    branches:
    - master
    - features/*
  pr:
    branches:
      include:
      - master
      - releases/*
```

`trigger: none` disables all triggers.
Each trigger category (PR, CI, and Schedule) can be disabled independently:
```yaml
trigger:
  pr: none
  push: ...
```

These changes also apply to `resource` triggers.

#### Behavior changes
* `repository` and `container` gain triggers by default. Users must suppress them using `trigger: none` if they don't want trigger behavior.
* ...

#### Syntax strictness
* `repository` and `container` allow arbitrary properties today. In v2, we will fix the set of properties that are allowed and error if unknown properties are passed.
* `task` inputs are arbitrary. In v2, only valid inputs (and aliases of inputs) will be accepted.
* ...

#### Still under discussion
* Match YAML representation model semantics. [Mapping keys are unordered](https://yaml.org/spec/1.1/#id863110). Steps are the key offender, requiring a key like `task` or `script` to appear first. v1 relaxes this restriction, allowing for constructions like:
```yaml
steps:
- displayName: Display name first
  task: Foo@1
- name: bar
  task: Bar@1
- condition: failed()
  script: echo Executes only if failed
```
* Our list of `jobs` implies an ordering, but it's actually a graph. Today, you write:
```yaml
jobs:
- job: foo
  displayName: Foo!
- job: bar
  displayName: Bar?
```
but a more correct representation would be:
```yaml
jobs:
  foo:
    displayName: Foo!
  bar:
    displayName: Bar?
```
We would _not_ do the same for stages.
Stages are run consecutively by default, which the sequence order makes clear.
We would have to resolve how `deployment` jobs look in this new syntax.
* Multiline strings: occassionally there places where we expect multiline string inputs (for example, path specifiers on artifact tasks). Today you must use:
```yaml
foo: |
  multi
  line
  string
```
We could support an alternate syntax:
```yaml
foo:
- multi
- line
- string
```
which some people feel is "more YAMLy".
* Identify YAML features which we don't support, but that the spec allows, and directly reject their syntax.
For example, this isn't valid YAML:
```yaml
branches:
- *
```
But it _is_ allowed by our v1 parser (it's identical to `branches: ['*']`).

### Templates

Pipelines composed of templates must have matching versions.
"Child" (included) templates must either explicitly declare the same version as the parent, or not specify a version at all.
If they don't specify a version at all, they're parsed as if they specified the version of the parent.

Examples:
```yaml
##### Legal - both have explicit versions #####
# pipeline.yml
version: 2
steps:
- template: steps.yml

# steps.yml
version: 2
steps:
- script: echo foo


##### Legal - child template doesn't specify version #####
# pipeline.yml
version: 1
steps:
- template: steps.yml

# steps.yml
steps:
- script: echo foo


##### Illegal - mismatched versions #####
# pipeline.yml
version: 2
steps:
- template: steps.yml

# steps.yml
version: 1
steps:
- script: echo foo


##### Illegal - parent has implied version 1 while child specifies version 2 #####
# pipeline.yml
steps:
- template: steps.yml

# steps.yml
version: 2
steps:
- script: echo foo
```
