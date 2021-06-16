# Terraform (Docker)

The Terraform templates (using Docker) are used to run Terraform in a `dev->qa->prod` environment promotion flow.

## Architecture

![terraform-architecture](../assets/terraform-architecture.jpg)

## Template Format

Parameter names that end with `Template` will be templated before they are used. They are templted with the [format command](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#format).
The format command is passed the environment name, which can be placed in the string.

## Plan

| Name                      | Default                                   | Description                                    |
| ------------------------- | ----------------------------------------- | ---------------------------------------------- |
| poolNameTemplate          | `""`                                      | Name of pool to use in template format.        |
| sourceBranch              | `"refs/heads/master"`                     | Source branch to limit image builds to.        |
| environments              | `[{name: dev}, {name: qa}, {name: prod}]` | Environments that should be deployed to.       |
| azureSubscriptionTemplate | `""`                                      | Name of Azure subscription in template format. |
| terraformFolder           | `""`                                      | Path to Terraform directory.                   |
| validateEnabled           | `true`                                    | Should `make validate` run during plan?        |


## Apply

| Name                      | Default                                                                                          | Description                                    |
| ------------------------- | ------------------------------------------------------------------------------------------------ | ---------------------------------------------- |
| poolNameTemplate          | `""`                                                                                             | Name of pool to use in template format.        |
| sourceBranch              | `"refs/heads/master"`                                                                            | Source branch to limit image builds to.        |
| azureSubscriptionTemplate | `""`                                                                                             | Name of Azure subscription in template format. |
| terraformFolder           | `""`                                                                                             | Path to Terraform directory.                   |
| environments              | `[{name: dev, deployTags: false}, {name: qa, deployTags: true}, {name: prod, deployTags: true}]` | Environments that should be deployed to.       |

## Examples

### Pipeline

```yaml
name: $(Build.BuildId)

trigger:
  batch: true
  branches:
    include:
      - master
  paths:
    include:
      - TF_FOLDER_NAME

resources:
  repositories:
    - repository: templates
      type: git
      name: PROJ/azure-devops-templates
      ref: resf/tags/2020.12.1

stages:
  - template: terraform-docker/plan/main.yaml@templates
    parameters:
      azureSubscriptionTemplate: "SUB"
      poolNameTemplate: "POOL"
      terraformFolder: "TF_FOLDER_NAME"
  - template: terraform-docker/apply/main.yaml@templates
    parameters:
      azureSubscriptionTemplate: "SUB"
      poolNameTemplate: "POOL"
      terraformFolder: "TF_FOLDER_NAME"
```

### Makefile

```makefile
SHELL:=/bin/bash

SUFFIX="tfstate<4 random digits>"
IMAGE="ghcr.io/xenitab/github-actions/tools:2021.03.1"
ENV?=""
DIR?=""
OPA_BLAST_RADIUS := $(if $(OPA_BLAST_RADIUS), $(OPA_BLAST_RADIUS), 50)
AZURE_CONFIG_DIR := $(if $(AZURE_CONFIG_DIR), $(AZURE_CONFIG_DIR), "$${HOME}/.azure")

check:
ifeq ($(ENV),"")
	echo "Need to set ENV"
	exit 1
endif
ifeq ($(DIR),"")
	echo "Need to set DIR"
	exit 1
endif

plan: check
	docker run --rm --entrypoint "/opt/terraform.sh" -v $${PWD}/$(DIR)/.terraform/$(ENV).env:/tmp/$(ENV).env -v $(AZURE_CONFIG_DIR):/home/tools/.azure -v $${PWD}/$(DIR):/tmp/$(DIR) -v $${PWD}/global.tfvars:/tmp/global.tfvars $(IMAGE) plan $(DIR) $(ENV) $(SUFFIX) $(OPA_BLAST_RADIUS)

apply: check
	docker run --rm --entrypoint "/opt/terraform.sh" -v $${PWD}/$(DIR)/.terraform/$(ENV).env:/tmp/$(ENV).env -v $(AZURE_CONFIG_DIR):/home/tools/.azure -v $${PWD}/$(DIR):/tmp/$(DIR) -v $${PWD}/global.tfvars:/tmp/global.tfvars $(IMAGE) apply $(DIR) $(ENV) $(SUFFIX)

validate: check
	docker run --rm --entrypoint "/opt/terraform.sh" -v $${PWD}/$(DIR)/.terraform/$(ENV).env:/tmp/$(ENV).env -v $(AZURE_CONFIG_DIR):/home/tools/.azure -v $${PWD}/$(DIR):/tmp/$(DIR) -v $${PWD}/global.tfvars:/tmp/global.tfvars $(IMAGE) validate $(DIR) $(ENV) $(SUFFIX)

prepare: check
	docker run --rm --entrypoint "/opt/terraform.sh" -v $${PWD}/$(DIR)/.terraform/$(ENV).env:/tmp/$(ENV).env -v $(AZURE_CONFIG_DIR):/home/tools/.azure -v $${PWD}/$(DIR):/tmp/$(DIR) -v $${PWD}/global.tfvars:/tmp/global.tfvars $(IMAGE) prepare $(DIR) $(ENV) $(SUFFIX)

destroy: check
	docker run --rm -it --entrypoint "/opt/terraform.sh" -v $${PWD}/$(DIR)/.terraform/$(ENV).env:/tmp/$(ENV).env -v $(AZURE_CONFIG_DIR):/home/tools/.azure -v $${PWD}/$(DIR):/tmp/$(DIR) -v $${PWD}/global.tfvars:/tmp/global.tfvars $(IMAGE) destroy $(DIR) $(ENV) $(SUFFIX)

state-remove: check
	docker run --rm -it --entrypoint "/opt/terraform.sh" -v $${PWD}/$(DIR)/.terraform/$(ENV).env:/tmp/$(ENV).env -v $(AZURE_CONFIG_DIR):/home/tools/.azure -v $${PWD}/$(DIR):/tmp/$(DIR) -v $${PWD}/global.tfvars:/tmp/global.tfvars $(IMAGE) state-remove $(DIR) $(ENV) $(SUFFIX)
```
