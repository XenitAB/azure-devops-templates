# Terraform (Docker)

The Terraform templates (using Docker) are used to run Terraform in a `dev->qa->prod` environment promotion flow.

## Architecture

![terraform-architecture](../assets/terraform-architecture.jpg)

## Template Format

Parameter names that end with `Template` will be templated before they are used. They are templted with the [format command](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#format).
The format command is passed the environment name, which can be placed in the string.

## AWS vs Azure

The template is built to support both Azure and AWS.
Azure have to be used since we store our state in Azure, but we support two service connections in a single pipeline.

### AWS config

As a user of the template you have the power to define any ARN policy that you want to use.
This is allot of power but the main protection is that you can't exceed the access you got in your AWS service connection.
So as long as the you have given the service connection a **reasonable** amount of access you are okay.

The Makefile supports AWS environment variables. If you don't use AWS it will just mount empty volumes.

## Plan

| Name                         | Default                                   | Description                                       |
| ---------------------------- | ----------------------------------------- | ------------------------------------------------- |
| poolNameTemplate             | `""`                                      | Name of pool to use in template format.           |
| sourceBranch                 | `"refs/heads/master"`                     | Source branch to limit image builds to.           |
| environments                 | `[{name: dev}, {name: qa}, {name: prod}]` | Environments that should be deployed to.          |
| azureSubscriptionTemplate    | `""`                                      | Name of Azure subscription in template format.    |
| terraformFolder              | `""`                                      | Path to Terraform directory.                      |
| validateEnabled              | `true`                                    | Should `make validate` run during plan?           |
| awsRegion                    | `""`                                      | The name of the AWS region you want to use        |
| awsServiceConnectionTemplate | `""`                                      | Name of AWS service connection in template format |
| awsArn                       | `""`                                      | The AWS arn to use when applying config to AWS    |

## Apply

| Name                         | Default                                                                                          | Description                                       |
| ---------------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------- |
| poolNameTemplate             | `""`                                                                                             | Name of pool to use in template format.           |
| sourceBranch                 | `"refs/heads/master"`                                                                            | Source branch to limit image builds to.           |
| azureSubscriptionTemplate    | `""`                                                                                             | Name of Azure subscription in template format.    |
| terraformFolder              | `""`                                                                                             | Path to Terraform directory.                      |
| environments                 | `[{name: dev, deployTags: false}, {name: qa, deployTags: true}, {name: prod, deployTags: true}]` | Environments that should be deployed to.          |
| awsRegion                    | `""`                                                                                             | The name of the AWS region you want to use        |
| awsServiceConnectionTemplate | `""`                                                                                             | Name of AWS service connection in template format |
| awsArn                       | `""`                                                                                             | The AWS arn to use when applying config to AWS    |

## Examples

### Pipeline

#### Azure

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

#### AWS

```yaml
name: $(Build.BuildId)

trigger:
  batch: true
  branches:
    include:
      - main
  paths:
    include:
      - TF_FOLDER_NAME

resources:
  repositories:
    - repository: templates
      type: git
      name: PROJ/azure-devops-templates
      ref: resf/tags/2021.07.1

stages:
  - template: terraform-docker/plan/main.yaml@templates
    parameters:
      azureSubscriptionTemplate: "SUB"
      poolNameTemplate: "POOL"
      terraformFolder: "TF_FOLDER_NAME"
      awsServiceConnectionTemplate: terraform_aws_{0}
      awsRegion: eu-west-1
      awsArn: arn:aws:iam::aws:policy/RandomBuiltInPolicy

  - template: terraform-docker/apply/main.yaml@templates
    parameters:
      azureSubscriptionTemplate: "SUB"
      poolNameTemplate: "POOL"
      terraformFolder: "TF_FOLDER_NAME"
      awsServiceConnectionTemplate: terraform_aws_{0}
      awsRegion: eu-west-1
      awsArn: arn:aws:iam::aws:policy/RandomBuiltInPolicy
```

### Makefile

```makefile
SHELL:=/bin/bash

SUFFIX="tfstate<4 random digits>"
IMAGE="ghcr.io/xenitab/github-actions/tools:2021.05.2"
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
	docker run --entrypoint "/opt/terraform.sh" -e AWS_PROFILE=$${AWS_PROFILE} -v $${PWD}/$(DIR)/.terraform/$(ENV).env:/tmp/$(ENV).env -v $(AZURE_CONFIG_DIR):/home/tools/.azure -v $${AWS_CONFIG_FILE}:/home/tools/.aws/config -v $${AWS_SHARED_CREDENTIALS_FILE}:/home/tools/.aws/credentials -v $${PWD}/$(DIR):/tmp/$(DIR) -v $${PWD}/global.tfvars:/tmp/global.tfvars $(IMAGE) plan $(DIR) $(ENV) $(SUFFIX) $(OPA_BLAST_RADIUS)

apply: check
	docker run --entrypoint "/opt/terraform.sh" -e AWS_PROFILE=$${AWS_PROFILE} -v $${PWD}/$(DIR)/.terraform/$(ENV).env:/tmp/$(ENV).env -v $(AZURE_CONFIG_DIR):/home/tools/.azure -v $${AWS_CONFIG_FILE}:/home/tools/.aws/config -v $${AWS_SHARED_CREDENTIALS_FILE}:/home/tools/.aws/credentials -v $${PWD}/$(DIR):/tmp/$(DIR) -v $${PWD}/global.tfvars:/tmp/global.tfvars $(IMAGE) apply $(DIR) $(ENV) $(SUFFIX)

prepare: check
	docker run --entrypoint "/opt/terraform.sh" -e AWS_PROFILE=$${AWS_PROFILE} -v $${PWD}/$(DIR)/.terraform/$(ENV).env:/tmp/$(ENV).env -v $(AZURE_CONFIG_DIR):/home/tools/.azure -v $${AWS_CONFIG_FILE}:/home/tools/.aws/config -v $${AWS_SHARED_CREDENTIALS_FILE}:/home/tools/.aws/credentials -v $${PWD}/$(DIR):/tmp/$(DIR) -v $${PWD}/global.tfvars:/tmp/global.tfvars $(IMAGE) prepare $(DIR) $(ENV) $(SUFFIX)

destroy: check
	docker run -it --entrypoint "/opt/terraform.sh" -e AWS_PROFILE=$${AWS_PROFILE} -v $${PWD}/$(DIR)/.terraform/$(ENV).env:/tmp/$(ENV).env -v $(AZURE_CONFIG_DIR):/home/tools/.azure -v $${AWS_CONFIG_FILE}:/home/tools/.aws/config -v $${AWS_SHARED_CREDENTIALS_FILE}:/home/tools/.aws/credentials -v $${PWD}/$(DIR):/tmp/$(DIR) -v $${PWD}/global.tfvars:/tmp/global.tfvars $(IMAGE) destroy $(DIR) $(ENV) $(SUFFIX)

state-remove: check
	docker run -it --entrypoint "/opt/terraform.sh" -e AWS_PROFILE=$${AWS_PROFILE} -v $${PWD}/$(DIR)/.terraform/$(ENV).env:/tmp/$(ENV).env -v $(AZURE_CONFIG_DIR):/home/tools/.azure -v $${AWS_CONFIG_FILE}:/home/tools/.aws/config -v $${AWS_SHARED_CREDENTIALS_FILE}:/home/tools/.aws/credentials -v $${PWD}/$(DIR):/tmp/$(DIR) -v $${PWD}/global.tfvars:/tmp/global.tfvars $(IMAGE) state-remove $(DIR) $(ENV) $(SUFFIX)

validate: check
	docker run --entrypoint "/opt/terraform.sh" -e AWS_PROFILE=$${AWS_PROFILE} -v $${PWD}/$(DIR)/.terraform/$(ENV).env:/tmp/$(ENV).env -v $(AZURE_CONFIG_DIR):/home/tools/.azure -v $${AWS_CONFIG_FILE}:/home/tools/.aws/config -v $${AWS_SHARED_CREDENTIALS_FILE}:/home/tools/.aws/credentials -v $${PWD}/$(DIR):/tmp/$(DIR) -v $${PWD}/global.tfvars:/tmp/global.tfvars $(IMAGE) validate $(DIR) $(ENV) $(SUFFIX)

fmt: check
	docker run --entrypoint "/opt/terraform.sh" -e AWS_PROFILE=$${AWS_PROFILE} -v $${PWD}/$(DIR)/.terraform/$(ENV).env:/tmp/$(ENV).env -v $(AZURE_CONFIG_DIR):/home/tools/.azure -v $${AWS_CONFIG_FILE}:/home/tools/.aws/config -v $${AWS_SHARED_CREDENTIALS_FILE}:/home/tools/.aws/credentials -v $${PWD}/$(DIR):/tmp/$(DIR) -v $${PWD}/global.tfvars:/tmp/global.tfvars $(IMAGE) "fmt -recursive" $(DIR) $(ENV) $(SUFFIX)
```
