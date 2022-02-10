# Gitops V2

These Azure DevOps tempalates are meant to be used to enable an application promotion flow together with Flux V2 in a multi tenant deployment.

## Prerequisites

The pipelines and tools expect that the tenant repository is setup in a specific structure.

```txt
.
├── apps/
│   ├── dev/
│   │   └── kustomization.yaml
│   ├── qa/
│   │   └── kustomization.yaml
│   └── prod/
│       └── kustomization.yaml
└── tenant/
    ├── dev/
    │   ├── app.yaml
    │   ├── kustomization.yaml
    │   └── notification.yaml
    ├── qa/
    │   ├── app.yaml
    │   ├── kustomization.yaml
    │   └── notification.yaml
    └── prod/
        ├── app.yaml
        ├── kustomization.yaml
        └── notification.yaml
```

The tenant directory contains the entrypoint for the tenant into each cluster. From there notifications are configured and
individual `Kustomization` resources are created to deploy applications. The directory `apps` represents a group of applications,
a tenant repository can contain multiple groups which will run their promotion in separate cadences.

Look in the [examples directory](https://github.com/XenitAB/azure-devops-templates/blob/main/gitops-v2/examples/template-repository) for a template directory structure to get started.

## Background

The tempaltes are meant to be used in two different repositories, `./build/main.yaml` should be used in the application
repository. It builds a Docker image and published the image as an artificat on the pipeline. After that is complete
its responsibility is done.

The `./new/main.yaml`, `./status/main.yaml`, and `./promote/main.yaml` are meant to be used to create pipelines in
the "gitops" repository. These three pipelines are responsible for creating the PRs to move an application from
one environment to another. The templates make heavy use of the [gitops-promotion](https://github.com/XenitAB/gitops-promotion)
tool to execute the majority of logic.

In addition to the gitops promotion pipelines mentioned above, the `./validate/main.yaml` can be used in the "gitops" repository
to create a pipeline that validates the syntactical correctness of the manifests.

## Usage

Create the build pipeline in the application repository. The `serviceName` is the name of the application.

build.yaml

```yaml
name: $(Build.BuildId)

trigger:
  branches:
    include:
      - main

resources:
  repositories:
    - repository: templates
      type: git
      name: XKS/azure-devops-templates
      ref: refs/tags/2021.04.2

stages:
  - template: gitops-v2/build/main.yaml@templates
    parameters:
      serviceName: "test-application"
```

Create the three pipelines in the "gitops" repository, note that the pipeline alias should match the `serviceName` it references.
It is totally fine to reference multiple pipelines as triggers, this will be handled by the pipeline to determine which pipeline
triggered the run.

There are support for both Azure and AWS container registry but not at the same time in one pipeline.
To use AWS you have to define the variable `cloud`, it defaults to `azure`.
> The `cloud` variable supports **lowercase** only, accepted variables is `aws` or `azure`.

new-azure.yaml

```yaml
trigger: none

resources:
  repositories:
    - repository: templates
      type: git
      name: XKS/azure-devops-templates
      ref: refs/tags/2021.04.2
  pipelines:
    - pipeline: test-application
      source: test-application
      trigger:
        stages:
          - cd_trigger

stages:
  - template: ./gitops-v2/new/main.yaml@templates
    parameters:
      azureSubscriptionTemplate: "azure-{0}-foobar-contributor"
      acrNameTemplate: "acr{0}weaks"
      imagePathPrefix: foobar
      cloud: azure
      group: apps
      environments:
        - name: dev
        - name: qa
        - name: prod
```

new-aws.yaml

```yaml
trigger: none

resources:
  repositories:
    - repository: templates
      type: git
      name: XKS/azure-devops-templates
      ref: refs/tags/2021.07.1
  pipelines:
    - pipeline: test-application
      source: test-application
      trigger:
        stages:
          - cd_trigger

stages:
  - template: ./gitops-v2/new/main.yaml@templates
    parameters:
      awsServiceConnectionTemplate: "ecr-{0}-foobar-contributor"
      awsRegion: eu-west-1
      cloud: aws
      group: apps
      environments:
        - name: dev
        - name: qa
        - name: prod
```

status.yaml

```yaml
trigger: none

resources:
  repositories:
    - repository: templates
      type: git
      name: XKS/azure-devops-templates
      ref: refs/tags/2021.04.2

stages:
  - template: ./gitops-v2/status/main.yaml@templates
```

promote.yaml

```yaml
trigger:
  - main

resources:
  repositories:
    - repository: templates
      type: git
      name: XKS/azure-devops-templates
      ref: refs/tags/2021.04.2

stages:
  - template: ./gitops-v2/promote/main.yaml@templates
```

validate.yaml

```yaml
trigger: none

resources:
  repositories:
    - repository: templates
      type: git
      name: XKS/azure-devops-templates
      ref: refs/tags/2021.04.2

stages:
  - template: ./gitops-v2/validate/main.yaml@templates
```

TBD - Repository configuration

After all of the pipelines are complete the last thing to do is to annotate where the image tag should be updated.
Without an annotation the gitops-promotion tool will not know where to update the image tag. This annotation is very
simple and can either be set on a kustomization image replacement or helm release.

./examples/template-repository/apps/dev/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: dev-a
resources:
  - certificate.yaml
  - ./podinfo
images:
  - name: acr.azurecr.io/tenant/test-application
    newTag: "0ddf7cc" # {"$imagepolicy": "apps:test-application:tag"}
```

The tag contains the group name in this case "apps" and the service name which in this case is "test-application".
Trigger a build of the application and see how the image tag will be updated.
