# Packer

The Packer templates are meant to be used to build VM images in Azure.

## Template Format

Parameter names that end with `Template` will be templated before they are used. They are templted with the [format command](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#format).
The format command is passed the environment name, which can be placed in the string.

## Build

| Name                      | Default                                                              | Description                                                     |
| ------------------------- | -------------------------------------------------------------------- | --------------------------------------------------------------- |
| poolNameTemplate          | `""`                                                                 | Pool name template.                                             |
| poolVmImage               | `"ubuntu-22.04"`                                                     | Pool vm image (only used if `poolNameTemplate` is empty)        |
| azureSubscriptionTemplate | `""`                                                                 | Azure subscription name template.                               |
| resourceGroupTemplate     | `""`                                                                 | Azure resource group name template.                             |
| packerTemplateRepo        | `"https://github.com/XenitAB/cloud-automation.git"`                  | GIT repository to use for packer template.                      |
| packerTemplateRepoBranch  | `"master"`                                                           | GIT branch to use for packer template repository.               |
| packerTemplateFile        | `""`                                                                 | Location (inside of packerTemplateRepo) of the packer template. |
| preBuild                  | `[]`                                                                 | Steps to run before Docker build, takes a list of steps.        |
| postBuild                 | `[]`                                                                 | Steps to run after Docker build, takes a list of steps.         |
| binaries.packer.tag       | `"1.6.5"`                                                            | Packer binary version.                                          |
| binaries.packer.sha       | `"a49f6408a50c220fe3f1a6192ea21134e2e8f31092c507614cd27ad4f913234b"` | Packer binary sha.                                              |

## Examples

**Build**

```yaml
name: $(Build.BuildId)

trigger: none

resources:
  repositories:
    - repository: templates
      type: git
      name: XenitKubernetesService/azure-devops-templates
      ref: refs/tags/2020.11.9

stages:
  - template: packer/main.yaml@templates
    parameters:
      poolNameTemplate: "xks-{0}"
      azureSubscriptionTemplate: "azure-{0}-contributor"
      resourceGroupTemplate: "rg-{0}-we-packer"
      packerTemplateRepo: "https://github.com/XenitAB/cloud-automation.git"
      packerTemplateRepoBranch: "2020.11.2"
      packerTemplateFile: "azure/packer/azure-pipelines-agent/ubuntu1804.json"
```
