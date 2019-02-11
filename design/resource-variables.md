# Resource variables
For each resource that is defined in a YAML pipeline, following pre-defined resource variables are made available in the jobs automatically for every pipeline run.
Each resource type has a different set of variables available.

## `pipeline` variables
When a `pipeline` resource is consumed in YAML, pipeline specific details are available as pre-defined variables.

| Variable name | Description |
|---------------|-------------|
|`Pipeline.Resources.{Alias}.Version` | The actual pipeline run number consumed as part of this pipeline. <br/><br />|
|`Pipeline.Resources.{Alias}.Source` | The pipeline definition name. <br/><br />|
|`Pipeline.Resources.{Alias}.Project` | The project where Pipeline definition is created. <br/><br />|
|`Pipeline.Resources.{Alias}.Branch` | The branch of the pipeline resource. <br/><br />|
|`Pipeline.Resources.{Alias}.BranchCommit`| The commit on which pipeline resource was executed. <br/><br />|
|`Pipeline.Resources.{Alias}.Tags`| The tags set on the pipeline resource. <br/><br />|
|`Pipeline.Resources.{Alias}.Provider`| The type of the resource. In this case it will be 'Pipelines'. <br/><br />|
|`Pipeline.Resources.{Alias}.SourceID`| The pipeline definition ID. <br/><br |

## `repository` varaibles
When a `repository` resource is consumed in YAML, the resource specific details are available as pre-defined variables.

| Variable name | Description |
|---------------|-------------|
|`Pipeline.Resources.{Alias}.Source`| The source repository name. <br/><br />|
|`Pipeline.Resources.{Alias}.Branch`| The branch of the repository consumed in this pipeline run. <br/><br />|
|`Pipeline.Resources.{Alias}.BranchCommit`| The commit from the repo consumed. <br/><br />|
|`Pipeline.Resources.{Alias}.Tags`| Tags set on the repo (if any). <br/><br />|
|`Pipeline.Resources.{Alias}.Provider`| The type of the resource. In this case it can be 'GitHub', 'Azure Repos', 'BitBucket' etc. <br/><br />|


## `container` variables:
When a `container` resource is consumed in YAML, the image specific details are available as pre-defined variables.

| Variable name | Description |
|---------------|-------------|
|`Pipeline.Resources.{Alias}.Image`| The name of the image consumed in this pipeline. <br/><br />|
|`Pipeline.Resources.{Alias}.Tag`| The image tag picked in this pipeline. <br/><br />|
|`Pipeline.Resources.{Alias}.Commit`| The commit of the image picked in this pipeline. <br/><br />|
|`Pipeline.Resources.{Alias}.Provider`| The type of resource. In this case it can be 'DockerRegistry' or 'ACR'. <br/><br />|
|`Pipeline.Resources.{Alias}.Registry`| The Registry where the image is located. <br/><br />|
|`Pipeline.Resources.{Alias}.Location`| The Azure location image is published to. This is ACR specific variable. <br/><br />|
