# Azure DevOps Templates
Collection of templates to use in Azure DevOps.

## Usage
Currently Azure DevOps does not support referencing templates from public GitHub repositories without
specifiying a valid service connection. This would be too cumborsome to deal with for multiple organizations.
Instead this repo is best used by creating a mirror of it, and using a pipeline to synchronize the content from upstream.

Begin by [importing the repository](https://docs.microsoft.com/en-us/azure/devops/repos/git/import-git-repository?view=azure-devops) into a global project in your organization.
Then create a pipeline from the definition located in `./ci/pipeline.yaml`, this pipeline will sync with Github.
Any time you want to get the latest changes you need to run this pipeline.

You should be able to use the templates when the mirroring is complete by referencing the git repository.
```yaml
resources:
  repositories:
    - repository: templates
      type: git
      name: <project>/azure-devops-templates

stages:
  - template: gitops/deploy/pipeline.yaml@templates
```

# License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
