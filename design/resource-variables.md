# Resource variables
For each resource that is defined in a YAML pipeline, following pre-defined resource variables are made available in the jobs automatically for every pipeline run.
Each resource type has a different set of variables available.

## `pipeline` variables
When a `pipeline` resource is consumed in YAML, pipeline specific details are available as pre-defined variables.

`Pipeline.Resources.{Alias}.Version` : The actual pipeline run number consumed as part of this pipeline. 
`Pipeline.Resources.{Alias}.Source` : The pipeline definition name
`Pipeline.Resources.{Alias}.Project` : The project where Pipeline definition is created.
`Pipeline.Resources.{Alias}.Branch` : The branch of the pipeline resource.
`Pipeline.Resources.{Alias}.BranchCommit`: The commit on which pipeline resource was executed.
`Pipeline.Resources.{Alias}.Tags`: The tags set on the pipeline resource.
`Pipeline.Resources.{Alias}.Provider`: The type of the resource. In this case it will be 'Pipelines'
`Pipeline.Resources.{Alias}.SourceID`: The pipeline definition ID.

## `repository` varaibles
When a `repository` resource is consumed in YAML, the resource specific details are available as pre-defined variables.
`Pipeline.Resources.{Alias}.Source`: The source repository name.
`Pipeline.Resources.{Alias}.Branch`: The branch of the repository consumed in this pipeline run.
`Pipeline.Resources.{Alias}.BranchCommit`: The commit from the repo consumed.
`Pipeline.Resources.{Alias}.Tags`: Tags set on the repo (if any)
`Pipeline.Resources.{Alias}.Provider`: The type of the resource. In this case it can be 'GitHub', 'Azure Repos', 'BitBucket' etc.


## `container` variables:
When a `container` resource is consumed in YAML, the image specific details are available as pre-defined variables.
`Pipeline.Resources.{Alias}.Image`: The name of the image consumed in this pipeline
`Pipeline.Resources.{Alias}.Tag`: The image tag picked in this pipeline.
`Pipeline.Resources.{Alias}.Commit`: The commit of the image picked in this pipeline.
`Pipeline.Resources.{Alias}.Provider`: The type of resource. In this case it can be 'DockerRegistry' or 'ACR'.
`Pipeline.Resources.{Alias}.Registry`: The Registry where the image is located.
`Pipeline.Resources.{Alias}.Location`: The Azure location image is published to. This is ACR specific variable.
