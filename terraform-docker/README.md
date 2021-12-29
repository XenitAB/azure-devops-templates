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
      ref: refs/tags/2020.12.1

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
      ref: refs/tags/2021.07.1

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

### Packaged Terraform tooling

In order to simplify the interaction with Terraform, you may want to use [XenitAB tools](https://github.com/XenitAB/github-actions). In particular, this provides an interface that can be used by both humans and CIs.