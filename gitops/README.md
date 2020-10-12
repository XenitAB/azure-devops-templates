# GitOps
The GitOps templates are meant to be used with Flux v1 to build docker images and update a repository with the new image tag.

## Architecture
![gitops-architecture](../assets/gitops-architecture.jpg)

## Template Format
Parameter names that end with  `Template` will be templated before they are used.  They are templted with the [format command](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#format).
The format command is passed the environment name, which can be placed in the string.

## Build
| Name | Default | Description |
| --- | --- | --- |
| poolVmImage | "ubuntu-16.04" | VM Image to set in pool configuration. |
| poolName | "" | Pool name to set in pool configuration. |
| sourceBranch | "refs/heads/master" | Source branch to limit image builds to. |
| dockerfilePath | "./Dockerfile" | Path to Dockerfile used in build. |
| dockerBuildArgs | "" | Additional build args to append when building docker image. |
| serviceName | "" | Name of application or service, will also be the name of the image. |

## Deploy
| Name | Default | Description |
| --- | --- | --- |
| poolNameTemplate | `""` | Name of pool to use in template format. |
| sourceBranch | `"refs/heads/master"` | Source branch to limit image builds to. |
| serviceName | `""` | Name of the service, should match service name in build. |
| azureSubscriptionTemplate | `""` | Name of Azure subscription in template format. |
| acrNameTemplate | `""` | Name of ACr to use in template format. |
| imagePathPrefix | `""` | Prefix to append to image name before pushing to ACR. |
| environments | `[{name: dev, deployTags: false}, {name: qa, deployTags: true}, {name: prod, deployTags: true}]` | Environments that should be deployed to. |

## Examples
**Build**
```yaml
name: $(Build.BuildId)

trigger:
  batch: true
  branches:
    include:
      - master
  tags:
    include:
      - "*"
  paths:
    include:
      - "*"

resources:
  repositories:
    - repository: templates
      type: git
      name: XenitKubernetesService/azure-devops-templates
      ref: refs/tags/2020.10.0

stages:
  - template: gitops/build/main.yaml@templates
    parameters:
      serviceName: "podinfo"
```

**Deploy**
```yaml
name: $(Build.BuildId)

trigger: none

resources:
  repositories:
    - repository: templates
      type: git
      name: XenitKubernetesService/azure-devops-templates
      ref: refs/tags/2020.10.2
  pipelines:
    - pipeline: ciPipeline
      source: ci-podinfo
      trigger: true

stages:
  - template: gitops/deploy/main.yaml@templates
    parameters:
      poolNameTemplate: "xks-{0}"
      serviceName: "podinfo"
      azureSubscriptionTemplate: "azure-{0}-lab-contributor"
      acrNameTemplate: "acr{0}wexks"
      imagePathPrefix: "lab"
      imageTagPath: ".spec.values.application.buildId"
      environments:
        - name: dev
          deployTags: false
```
