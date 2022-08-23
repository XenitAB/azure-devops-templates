# Packer (Docker)

The Packer (Docker) templates are meant to be used to build VM images in Azure.

## Template Format

Parameter names that end with `Template` will be templated before they are used. They are templted with the [format command](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#format).
The format command is passed the environment name, which can be placed in the string.

## Build

| Name                      | Default                                             | Description                                                     |
| ------------------------- | --------------------------------------------------- | --------------------------------------------------------------- |
| poolNameTemplate          | `""`                                                | Pool name template.                                             |
| poolVmImage               | `"ubuntu-22.04"`                                    | Pool vm image (only used if `poolNameTemplate` is empty)        |
| azureSubscriptionTemplate | `""`                                                | Azure subscription name template.                               |
| resourceGroupTemplate     | `""`                                                | Azure resource group name template.                             |
| packerTemplateRepo        | `"https://github.com/XenitAB/packer-templates.git"` | GIT repository to use for packer template.                      |
| packerTemplateRepoBranch  | `"master"`                                          | GIT branch to use for packer template repository.               |
| packerTemplateFile        | `""`                                                | Location (inside of packerTemplateRepo) of the packer template. |
| preBuild                  | `[]`                                                | Steps to run before Docker build, takes a list of steps.        |
| postBuild                 | `[]`                                                | Steps to run after Docker build, takes a list of steps.         |

## Examples

### Pipeline

```yaml
name: $(Build.BuildId)

trigger: none

resources:
  repositories:
    - repository: templates
      type: git
      name: XenitKubernetesService/azure-devops-templates
      ref: refs/tags/2020.12.2

stages:
  - template: packer-docker/main.yaml@templates
    parameters:
      poolNameTemplate: "xks-{0}"
      azureSubscriptionTemplate: "azure-{0}-contributor"
      resourceGroupTemplate: "rg-{0}-we-packer"
      packerTemplateRepo: "https://github.com/XenitAB/packer-templates.git"
      packerTemplateRepoBranch: "2020.12.1"
      packerTemplateFile: "templates/azure/azure-pipelines-agent/ubuntu1804.json"
```

### Makefile

```makefile
SHELL:=/bin/bash

IMAGE="ghcr.io/xenitab/github-actions/tools:2020.12.6"
ENV?=""
GIT_REPO?=""
GIT_BRANCH?=""
TEMPLATE_FILE?=""
AZURE_CONFIG_DIR := $(if $(AZURE_CONFIG_DIR), $(AZURE_CONFIG_DIR), "$${HOME}/.azure")
TEMP_DIRECTORY := $(if $(AGENT_TEMPDIRECTORY), $(AGENT_TEMPDIRECTORY), "/tmp")
RESOURCE_GROUP?=""

check:
ifeq ($(GIT_REPO),"")
	echo "Need to set GIT_REPO"
	exit 1
endif
ifeq ($(GIT_TAG),"")
	echo "Need to set GIT_TAG"
	exit 1
endif
ifeq ($(TEMPLATE_FILE),"")
	echo "Need to set TEMPLATE_FILE"
	exit 1
endif
ifeq ($(RESOURCE_GROUP),"")
	echo "Need to set RESOURCE_GROUP"
	exit 1
endif

build: check
	docker run --rm --entrypoint "/opt/packer.sh" -v $(AZURE_CONFIG_DIR):/home/tools/.azure -v $(TEMP_DIRECTORY)/$(ENV).env:/tmp/$(ENV).env $(IMAGE) build $(ENV) $(GIT_REPO) $(GIT_BRANCH) $(TEMPLATE_FILE) $(RESOURCE_GROUP)
```
