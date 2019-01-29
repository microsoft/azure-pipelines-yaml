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
  path: string # where to put the repo; if a relative path, rooted at the default $(Build.SourcesDirectory)
```

If the `self` repo is redirected to a non-default path, `$(Build.SourcesDirectory)` is set to the actual path.
`$(System.DefaultWorkingDirectory)` is also set to match the location of the `self` repo.

### Example

```yaml
# Example 1 - relative path
steps:
- checkout: self
  path: PutMyCodeHere   # will checkout at $(Agent.WorkDir)/s/PutMyCodeHere
- script: ./build.sh
  # build.sourcesdirectory and working directory are set to $(Agent.WorkDir)/s/PutMyCodeHere

# Example 2 - absolute path
steps:
- checkout: self
  path: /src   # absolute path, useful for example in a container
- script: ./build.sh
  # build.sourcesdirectory and working directory are set to /src
```
