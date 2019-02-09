# Control the checkout location of code

Customers often wish to control the location of checked-out code.
While we're still designing and working on [multiple checkout](multicheckout.md), we can make progress on controlling the checkout of `self`.

Scenarios:
- Previous versions of Go were quite particular about directory structure.
This would let users more easily meet the default expectations, rather than play games with environment variables.
- Some infrastructure makes assumptions about directory structure.
For example, Facebook's Jest repo used to assume the code was in a directory called `jest`.
- Containers.
libgit2's CI pipeline maps sources into a particular directory in the container.
Tools installed in the container expect to find the sources there.

## Schema

```yaml
steps:
- checkout: self
  path: string # where to put the repo; always rooted at $(Build.SourcesDirectory)
```

`$(Build.SourcesDirectory)` is unchanged from its default.

### Example

```yaml
steps:
- checkout: self
  path: PutMyCodeHere   # will checkout at $(Agent.WorkDir)/s/PutMyCodeHere
- script: ./PutMyCodeHere/build.sh
  # build.sourcesdirectory and default working directory still point to $(Agent.WorkDir)/s,
  # so the extra /PutMyCodeHere/ is needed
```

### Relative vs absolute paths

Relative paths are supported cross-platform.
Consider `path: foo/src`.
That resolves to `$(Build.SourcesDirectory)/foo/src`, which correctly resolves on both Windows and Linux.

Absolute paths are challenging to support cross-platform in a clean way.
They also make it harder to trust running multiple agents on a single host.
Finally, they can lead to hard-to-debug issues with permissions and cleanup.
The agent's work is supposed to be self-contained, so for our initial implementation, we will not support absolute paths.
