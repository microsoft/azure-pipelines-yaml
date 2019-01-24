# Freshening the YAML templates

When we launched Azure Pipelines, we opted to make the templates as brief as possible.
This made them feel more approachable, and made the system feel less overwhelming.
A lot has changed since then.

We have traction with lots of real, public customers, so there are lots of examples in the wild.
We've improved our editing experiences significantly in both the web and VS Code.
Finally, we're adding new syntax which further reduces word counts.

For 2019, we'll evolve the templates to cover more of the Pipelines system.
We'll simplify using new syntax wherever possible.

## Ecosystems: `use`

Before, we had to say:
```yaml
- task: NodeTool@0
  inputs:
    versionSpec: '8.x'
  displayName: Install Node.JS
```

The new `use` model lets us shorten this to:
```yaml
- use: node
  version: '8.x'
```

That drops us from 82 to 28 characters, nice!

## Caching

When we finish caching, we'll use new syntax to add it.
In many cases, that will be no additional words:
the `use` tasks will grow to encompass smart defaults for that ecosystem.

## Artifact uploads

In some ecosystems, it almost always makes sense to publish pipeline artifacts.
When the new `upload` syntax ships, it will be as simple as adding:
```yaml
- upload: **/out/**/*.js
```

to automatically publish those artifacts.
We previously opted to leave that capability out of the templates because the syntax was gross.
Now that it's simple, it's worth showing off the feature.
