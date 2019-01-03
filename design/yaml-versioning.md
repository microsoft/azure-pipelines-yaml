# YAML Versioning

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

## Industry landscape
*TODO*

## Solution

We introduce a new field, `version`, which lives at the root level of the pipeline.
It's an integer whose valid values are (currently) only `0` and `1`.
Any other number is a syntax error.
If not present, the value is assumed to be `0`.

We'll continue our commitment to backcompat within a single version.
Whatever version is currently in active development will grow new features over time.
Down-level versions will continue to be supported for the foreseeable future, though we reserve the right to remove versions eventually.

### Version 0
Version 0 is the current set of syntax, back-compat warts and all.
No behavior changes are expected, and we'll freeze this version when v1 ships.
No new features will land in v0.

### Version 1
Version 1 becomes our active development version.
This spec doesn't attempt to list all the new things it will gain, since that's not predictable.
Instead, it lists only the breaking changes we'll make now.

#### Additions
* `version: 1` must be specified at the root of the pipeline.

#### Removals
* Remove `phases`/`phase`. This became `jobs`/`job` with minimal other changes.
* Remove `queue`. This includes all the nodes which moved under `strategy`.
* Remove Git-LFS, shallow fetch, and other non-core properties from `repository`. These are now handled by `checkout`.
* ...

#### Behavior changes
* `repository` and `container` gain triggers by default. Users must suppress them using `trigger: none` if they don't want trigger behavior.
* ...
