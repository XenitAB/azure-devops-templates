# Gitops V2

These Azure DevOps tempalates are meant to be used to enable an application promotion flow together with Flux V2 in a multi tenant deployment.

# Prerequisites

The pipelines and tools expect that the tenant repository is setup in a specifc strucuture.

```
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

Look in the [examples directory](./examples/template-repository) for a template directory structure to get started.

# Background

The tempaltes are meant to be used in two different repositories, `./build/main.yaml` should be used in the application
repository. It builds a Docker image and published the image as an artificat on the pipeline. After that is complete
its responsibility is done.

The `./new/main.yaml`, `./status/main.yaml`, and `./promote/main.yaml` are meant to be used to create pipelines in
the "gitops" respository. These three pipelines are responsible for createing the PRs to move an application from
one environment to another. The templates make heavy use of the [gitops-promotion](https://github.com/XenitAB/gitops-promotion)
tool to execute the majority of logic.

# Usage

Create the build pipeline in the application repository. The `serviceName` is the name of the application.

build.yaml
```
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

new.yaml
```
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
      azureSubscriptionTemplate: "msdn"
      acrNameTemplate: "acr"
      imagePathPrefix: "foobar"
      group: "apps"
      environments:
        - name: dev
        - name: qa
        - name: prod
```

status.yaml
```
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
```
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

TBD - Repository configuration

After all of the pipelines are complete the last thing to do is to annotate where the image tag should be updated.
Without an annotation the gitops-promotion tool will not know where to update the image tag. This annotation is very
simple and can either be set on a kustomization image replacement or helm release.

./examples/template-repository/apps/dev/kustomization.yaml
```
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
