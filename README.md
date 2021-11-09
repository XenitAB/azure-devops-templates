# Azure DevOps Templates

Collection of templates to use in Azure DevOps and GitHub

## Usage

Currently neither Azure DevOps nor GitHub support referencing templates from public GitHub repositories without
specifying a valid service connection. This would be too cumbersome to deal with for multiple organizations.
Instead this repo is best used by creating a mirror of it, and using a pipeline to synchronize the content from upstream.

### Importing the repository to Azure DevOps

Begin by [importing the repository](https://docs.microsoft.com/en-us/azure/devops/repos/git/import-git-repository?view=azure-devops) into a global project in your organization.

- Azure DevOps > Project > Repos > Import Repository
  - Repository type: Git
  - Clone URL: https://github.com/XenitAB/azure-devops-templates.git
  - [ ] Requires Authentication
  - Name: azure-devops-templates
- Press "Import"

#### Configuring Build Service permissions

Configure Build Service to have permission to push changes to the template repository.

- Azure DevOps > Project > Project settings > Repositories > azure-devops-templates > Permissions
- [Project Name] Build Service ([org name]):
  - Contribute: `Allow`
  - Create branch: `Allow`
  - Force push (rewrite history, delete branches and tags): `Allow`

#### Adding Azure Pipelines for repository syncronization

Then create a pipeline from the definition located in `./ci/pipeline.yaml`, this pipeline will sync with Github.
Any time you want to get the latest changes you need to run this pipeline.

- Azure DevOps > Project > Repos > Set up build
- Choose Azisting Azure Pipelines YAML file
  - Branch: master
  - Path: /.ci/pipeline.yaml
- Press Continue > Press Run

#### Adding reference to the synchronized repository

You should be able to use the templates when the mirroring is complete by referencing the git repository.

```yaml
resources:
  repositories:
    - repository: templates
      type: git
      name: <project>/azure-devops-templates
      ref: refs/tags/<version>

stages:
  - template: gitops/deploy/pipeline.yaml@templates
```

### Importing the repository to GitHub

Begin by [importing the repository](https://github.com/new/import) into a global project in your organization.

- GitHub > Repositories > New > Import a repository
  - Clone URL: https://github.com/XenitAB/azure-devops-templates.git
  - Owner: <your organization> / azure-devops-templates
  - Privacy: Public
- Press "Begin Import"

The repository contains a GitHub Action that will run and update from upstream at least once per hour, see `./.github/workflows/update-azure-devops-templates-from-upstream.yaml`.

# Versioning

Versions follow the [CalVer](https://calver.org/) standard. This simplifies detecting usage of very old versions.
The versions should use the following pattern `YYYY.0M.MICRO`. The micro value starts at 0 and increments by one
for each release during the same month.

# License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
