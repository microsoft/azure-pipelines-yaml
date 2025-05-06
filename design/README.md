# Azure Pipelines YAML - Design Docs

The design docs within this repo are created at different times during the development of Azure Pipelines, to support collaborative contributions to the design process. Designs documents are for,

* features considered for implementation but never implemented
* already implemented features
* future ideas for features

The design docs in this repo may not represent the current state of an Azure Pipelines feature. 

For the most upto date information about a feature refer to the [Azure Pipeline Docs](https://docs.microsoft.com/en-us/azure/devops/pipelines/?view=azure-devops) and specifically the [Azure Pipeline Yaml Reference](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema)

## Improving the .github/CODEOWNERS file

This section provides ideas for improving, extending, completing, or simplifying the `.github/CODEOWNERS` file.

### Summary of Ideas

1. **Add more specific rules**: Instead of having a global rule for all files, you can add more specific rules for different directories or file types. For example, you can add a rule for `design/` directory to assign specific code owners for design documents. This can help in better management and review of code changes.
2. **Use wildcard patterns**: You can use wildcard patterns to match specific file types or directories. For example, you can use `*.md` to match all markdown files and assign specific code owners for documentation. This can help in organizing code ownership based on file types.
3. **Add fallback owners**: You can add fallback owners for specific directories or file types. For example, you can add a fallback owner for all files in the `templates/` directory to ensure that there is always someone responsible for reviewing changes in that directory.
4. **Simplify the file**: If the file becomes too complex with many specific rules, you can simplify it by grouping similar rules together or using wildcard patterns. This can help in maintaining the file and making it easier to understand.

For more details, refer to the relevant section in the `.github/CODEOWNERS` file.
