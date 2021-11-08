# Terraform docker GitHub Actions

To make the files available for other github actions we have to store them in `.github/workflows/` folder,
[terraform-docker.yaml](https://github.com/XenitAB/azure-devops-templates/blob/main/.github/workflows/terraform-docker.yaml).

In [terraform.yaml](https://github.com/XenitAB/azure-devops-templates/blob/main/terraform-docker-github/terraform.yaml) you can find a example how to use the action.

The action supplies the **plan validate and apply** stages.

## Secret structure

This workflow supports 3 environments, dev/qa/prod and to run this workflow you have to create 3 secrets in your repository.
One per environment that you want to deploy to.

The secret should contain SP credentials and it have to look exactly like
bellow as defined in [azure-login github action](https://github.com/marketplace/actions/azure-login).

```.json
{"clientId": "<GUID>",
  "clientSecret": "<CLIENT_SECRET_VALUE>",
  "subscriptionId": "<GUID>",
  "tenantId": "<GUID>"}
```

Example:

```.json
{"clientId": "00000000-0000-0000-0000-000000000000",
  "clientSecret": "foooooooooooooooooooooobaaaaaaaaaaaar",
  "subscriptionId": "00000000-0000-0000-0000-000000000000",
  "tenantId": "00000000-0000-0000-0000-000000000000"}
```

## Input

### Variables

| Name             | Default | Required | Description                                    |
| ---------------- | ------- | -------- | ---------------------------------------------- |
| runs-on          | `""`    | `false`  | Tag of the host that the job should be run on. |
| opa_blast_radius | `"50"`  | `false`  | The allowed blast radius of terraform.         |
| ENVIRONMENTS     | `""`    | `true`   | Environments that should be deployed to.       |
| DIR              | `""`    | `true`   | Path to Terraform directory.                   |
| VALIDATE_ENABLED | `true`  | `false`  | Should `make validate` run during plan?        |

### Secrets

| Name                   | Default | Required | Description                       |
| ---------------------- | ------- | -------- | --------------------------------- |
| AZURE_CREDENTIALS_DEV  | `""`    | `false`  | Azure login credentials for dev.  |
| AZURE_CREDENTIALS_QA   | `""`    | `false`  | Azure login credentials for qa.   |
| AZURE_CREDENTIALS_PROD | `""`    | `false`  | Azure login credentials for prod. |
