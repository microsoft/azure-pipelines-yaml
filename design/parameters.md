# Loading parameters from external files
Currently, parameters passed to templates must be declared inline. Here is an example of how one would implement, for example, an expansion of jobs: 

```yaml
# groups-template.yaml
parameters:
    groups: []
jobs:
  - ${{ each group in parameters.groups }}:
    - job: ${{ group.name }}
      pool:
        vmImage: 'ubuntu-16.04'
      steps: 
      - ${{ each item in group.items }}
        - script: | 
            echo "the item payload is: ${{ item.payload }}"
          displayName: "${{ item.displayName }}"
```

```yaml
# azure-pipelines.yaml

stages: 
- stage: Group-Stage
  jobs:
  - template: groups-template.yaml
    parameters:
      groups:
      - group1:
        name: "first group"
        items:
        - item: 
          name: "item1"
          displayName: "group1-item2"
          payload: "the first item in group1"
        - item: 
          name: "item2"
          displayName: "group1-item2"
          payload: "the second item in group1"
      - group2:
        name: "second group"
        items:
        - item: 
          name: "item1"
          displayName: "group2-item1"
          payload: "the first item in group2"
        - item: 
          name: "item2"
          displayName: "group2-item2"
          payload: "the second item in group2"
```

While this is useful, it doesn't allow for separation of the parameters from the pipeline file itself. Consider the case where we have multiple pipelines in one repo, with different functionalities, but where some of the pieces have the same values. In this case, we would need to specify the same values twice. Or the case where we have have multiple templates used in the same azure-pipeline, but they use the same set of data. In that case we need to replicate the values again. 

We already support the use of variable templates, but it would be useful to load arbitrary values as indicated above in a parameter definition file, such that the above example instead looks like this: 

```yaml
# group-data.yaml
groups:
- group1:
  name: "first group"
  items:
  - item: 
    name: "item1"
    displayName: "group1-item2"
    payload: "the first item in group1"
  - item: 
    name: "item2"
    displayName: "group1-item2"
    payload: "the second item in group1"
- group2:
  name: "second group"
  items:
  - item: 
    name: "item1"
    displayName: "group2-item1"
    payload: "the first item in group2"
  - item: 
    name: "item2"
    displayName: "group2-item2"
    payload: "the second item in group2"

```

```yaml
# azure-pipelines.yaml

stages: 
- stage: Group-Stage
  jobs:
  - template: groups-template.yaml
    parameters: 
    - template: group-data.yaml
  - template: another-groups-template.yaml
    parameters: 
    - template: group-data.yaml
    .
    .
    . 
```

This would allow for encapsulation of data as a separate entities. It could also allow for some other interesting examples to do, such as selection of different set of values at execution at runtime. 

```yaml
# azure-pipelines.yaml

variables: 
    region: 'region1'

jobs:
- template: region-template.yaml
  parameters: 
  - template: region-data-${{ variables.region }}.yaml
    .
    .
    . 
```

in this example, region-template.yaml could contain a generic set of instructions to execute commands in a region agnostic way, but the data could be region specific (allowing overrides to be applied at queing time)


