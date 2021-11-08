# gitops-v2 for github

This README contains information how to use our github actions for building container images
and pushing them to Azure Container Registry (ACR), and how to use our gitops promotion workflow.

## Push image

To make the files available for other github actions we have to store them in `.github/workflows/` folder,
[push-image-acr.yaml](https://github.com/XenitAB/azure-devops-templates/blob/main/.github/workflows/push-image-acr.yaml).

In [push-image.yaml](https://github.com/XenitAB/azure-devops-templates/blob/main/terraform-docker-github/terraform.yaml) you can find a example how to use the action.

### Input

#### Variables

| Name         | Default | Required | Description                                    |
| ------------ | ------- | -------- | ---------------------------------------------- |
| runs-on      | `""`    | `false`  | Tag of the host that the job should be run on. |
| ENVIRONMENTS | `""`    | `true`   | Environments that should be deployed to.       |
| image-name   | `""`    | `true`   | What the image should be called.               |
| organization | `""`    | `true`   | Organization/username in github.               |
| gitops-repo  | `""`    | `true`   | Name of your gitops repository.                |

#### Secrets

| Name                       | Default | Required | Description                                                           |
| -------------------------- | ------- | -------- | --------------------------------------------------------------------- |
| REGISTRY_LOGIN_SERVER_DEV  | `""`    | `false`  | The ACR URL for dev.                                                  |
| REGISTRY_USERNAME_DEV      | `""`    | `false`  | The SP clientId used to login to ACR for dev.                         |
| REGISTRY_PASSWORD_DEV      | `""`    | `false`  | The SP clientSecret used to login to ACR for dev.                     |
| REGISTRY_LOGIN_SERVER_QA   | `""`    | `false`  | The ACR URL for qa.                                                   |
| REGISTRY_USERNAME_QA       | `""`    | `false`  | The SP clientId used to login to ACR for qa.                          |
| REGISTRY_PASSWORD_QA       | `""`    | `false`  | The SP clientSecret used to login to ACR for qa.                      |
| REGISTRY_LOGIN_SERVER_PROD | `""`    | `false`  | The ACR URL for prod.                                                 |
| REGISTRY_USERNAME_PROD     | `""`    | `false`  | The SP clientId used to login to ACR for prod.                        |
| REGISTRY_PASSWORD_PROD     | `""`    | `false`  | The SP clientSecret used to login to ACR for prod.                    |
| XKS_APP_ID                 | `""`    | `true`   | The GitHub application ID use to communicate with other repositories. |
| XKS_PRIVATE_KEY            | `""`    | `true`   | The GitHub application pem file.                                      |
