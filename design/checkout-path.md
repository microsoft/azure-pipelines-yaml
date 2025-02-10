# Control the checkout location of code

Customers often wish to control the location of checked-out code.
While we're still designing and working on [multiple checkout](multi-checkout.md), we can make progress on controlling the checkout of `self`.

Scenarios:
- Previous versions of Go were quite particular about directory structure.
This would let users more easily meet the default expectations, rather than play games with environment variables.
- Some infrastructure makes assumptions about directory structure.
For example, Facebook's Jest repo used to assume the code was in a directory called `jest`.
- Containers.
libgit2's CI pipeline maps sources into a particular directory in the container.
Tools installed in the container expect to find the sources there.

Dependency:
- ~~Introduction of a new `$(Pipeline.Workspace)` variable which always resolves to the workspace for a particular pipeline.~~
_Postponed for now -- Build.SourcesDirectory gives the right loation._ See [the pipeline artifacts spec](pipeline-artifacts.md) for the feature which introduces this variable.

## Schema

```yaml
steps:
- checkout: self
  path: string # where to put the repo; always rooted at $(Pipeline.Workspace)
```

`$(Build.SourcesDirectory)` should also point to the actual `self` checkout location. Over time, we'll deprecate the `$(Build.*)` series of variables.

### Example

```yaml
steps:
- checkout: self
  path: PutMyCodeHere   # will checkout at $(Pipeline.Workspace)/PutMyCodeHere
- script: ../PutMyCodeHere/build.sh
  # default working directory still point to $(Agent.BuildDirectory)/s,
  # so the extra ../PutMyCodeHere/ is needed
```

### Relative vs absolute paths

Relative paths are supported cross-platform.
Consider `path: foo/src`.
That resolves to `$(Pipeline.Workspace)/foo/src`, which correctly resolves on both Windows and Linux.

Absolute paths are challenging to support cross-platform in a clean way.
They also make it harder to trust running multiple agents on a single host.
Finally, they can lead to hard-to-debug issues with permissions and cleanup.
The agent's work is supposed to be self-contained, so for our initial implementation, we will not support absolute paths.
