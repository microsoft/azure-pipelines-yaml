# Resources in YAML

Any source that is consumed as part of your pipeline is a resource. Resources in YAML represent sources of types pipelines, repositories, containers and packages.

An example of a resource can be resources published by another CI/CD pipeline (say Azure pipelines, Jenkins etc.), code repositories (GitHub), container registry (ACR, Docker hub etc.) and package feeds (Azure artifact feed, Artifactor etc.).  

## Why resources?

Resources enable you to define the sources at one place and consume anywhere in your pipeline. Resources provide you the full traceablity of the sources consumed in your pipeline including branches, versions, tags, associated commits and work-items. 

### Schema

```yaml
resources:
  pipelines: [ pipeline ]  
  repositories: [ repository ]
  containers: [ container ]
  packages: [ package ]
```

---

## Resources: `pipelines`

If you have a pipeline that produces artifacts, you can consume the artifacts using `pipelines` resource. A pipeline can be another Azure DevOps pipeline or any external pipelines like Jenkins etc.

### Schema

```yaml
resources:
  pipelines:
  - pipeline:
    type:
    project:
    source:
    branch:
    version:
    tags:
```

### Examples

```yaml
- downloadArtifact:
  name: webapp
- downloadArtifact: tools-pipeline
```
