# IAF Actions

This repository contains reusable GitHub Actions and Workflows. Inspired by (and copied from) [farfetched-Actions](https://github.com/equinor/farfetched-actions). As a part of moving all pipelines for IAF applications from Azure Devops to Github Actions, we also create and add reusable actions where its possible.

## Workflows

* [Fusion Deploy](#fusion-deploy)
* [Pr Name Validation](#pr-name-validation)
* [Tag Container](#tag-container)
* [Docker build](#docker-build)

### Fusion Deploy

This workflow deploys the created artifact(s) in a workflow to the given Fusion Portal environment. It deploys "on behalf of" a Azure ServicePrincipal with Federated Credentials.
The workflow will login into Azure using the given ServicePrincipal, obtain a Token to use against the given resource and then upload the artifact(s) to the Portal using the token.

_Parameters:_

* **fusion-portal-url** (mandatory)
URL to the Portal where the artifact should be deployed.
* **app-key** (mandatory)
The AppKey (name of the package), as used within the Fusion Portal.
* **artifact-path** (mandatory)
The "local" path (with filename) to the artifact to be uploaded to Fusion.
* **publish** (optional)
Set if the artifact should immediately be published in the Portal or not. Default true.
* **azure-tenant-id**
The Azure Tenant ID of the ServicePrincipal (Application registration) which is setup with the proper Federated Credentials.
* **azure-client-id**
The Client ID of the ServicePrincipal.
* **azure-resource-id**
The Resource ID where the artifact should be deployed to. Note that this doesn't have anything to do with resource groups, but the app registration that hosts the desired environment (CI/QA or PROD). For CI/QA it is most likely `5a842df8-3238-415d-b168-9f16a6a6031b` and for PROD it is most likely `97978493-9777-4d48-b38a-67b0b9cd88d2`.

Example (`deploy` here is a _job_ in a GitHub workflow)

```text
    deploy:
        name: "🚀 Deploy to Fusion Portal"
        runs-on: ubuntu-latest
        needs: build
        steps:
            - name: "Download artifact"
              uses: actions/download-artifact@v2
              with:
                  name: theNameOfTheArtifactThatWasUploaded.zip

            - name: Deploy to Fusion Portal
              uses: equinor/iaf-actions/fusion-deploy@v1.0
              with:
                  fusion-portal-url: ${{ secrets.FUSION_PORTAL_URL }}
                  app-key: ${{ env.FUSION_APP_KEY }}
                  artifact-path: theNameOfTheArtifactThatWasUploaded.zip
                  azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
                  azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
                  azure-resource-id: ${{ secrets.AZURE_RESOURCE_ID }}
```

### PR name validation

This action validates the PR name in accordance to the Farfetched "rules", which again are based on [Conventional Commit](https://www.conventionalcommits.org/en/v1.0.0/).
If the name/title of the PR is not in accordance to the Conventional Commit standard (with possible additional restrictions added by the
Farfetched team), the action will "fail".
In case of failure the PR will be labeled with `invalid_PR_name` (default, can be overridden).

_Parameters:_

* **github-token** (mandatory)
The GitHub token, needed for internal `gh` operations (e.g. labeling the PR). Typically obtained through `${{ secrets.GITHUB_TOKEN }}`.
* **failed-pr-label** (optional)
The label to "stamp" on the PR if validation fails. Default: `invalid_PR_name`.

Example

```text
name: 🔍️ PR Validation
on:
    pull_request:
        branches:
            - main
        types:
            - opened
            - edited
            - reopened
            - synchronize
    workflow_dispatch:

jobs:
    lint-pr-name:
        name: Lint pull request title
        runs-on: ubuntu-latest
        steps:
            - name: Validate name
              uses: equinor/iaf-actions/pr-name-validator@v1.0
              with:
                  github-token:  ${{ secrets.GITHUB_TOKEN }}

```

### Tag Container 

This action adds an environment specific tag  to existing container image created by the same build. The Action uses provided credentials to login into specified Azure Container Registry (ACR), download image with tag corresponding to build-id, add environment specific tag (IE, ci-release, qa-release or prod-release) and push image back to ACR. 

_Parameters:_
* **build-id:** The unique id of the current build used to retrieve the container
* **environment-shortname:** The shortname of the current environment, IE ci, dev or qa
* **registry**: The URI of the container repository in Azure
* **registry-repository:** The repository in azure container registry where the container is placed
* **tag:** The tag to add to the container
* **client-id:** The username or client id to connect to ACR
* **client-secret:** The password or client secret to use

Example

```text
- name: "Tag Image with Env"
    uses: equinor/iaf-actions/add-container-tags@v1.0
    with:
        build-id: ${{ github.sha }}
        environment-shortname: 'ci'
        registry: ${{ secrets.acr_registry }}
        registry-repository: ${{ secrets.acr_repository }}
        tag: ${{ secrets.shortenvname }}
        client-id: ${{ secrets.clientid }}
        client-secret: ${{ secrets.clientsecret }}
```

### Docker Build

This actions builds a docker container based on Dockerfile, adding a github build identificator as tag, and publishes the container to Azure Container Registry. The action also injects Dynatrace OneAgent at buildtime, which is configurable in each environment. 

_Parameters:_
* **name:** The app name
* **build-id:** The unique id of the current build used to retrieve the container
* **registry**: The URI of the container repository in Azure
* **registry-repository:** The repository in azure container registry where the container is placed
* **client-id:** The username or client id to connect to ACR
* **client-secret:** The password or client secret to use
* **dynatrace-tenant:** Dynatrace Tenant to use
* **dynatrace-token:** Dynatrace token to use
* **dynatrace-url:** Dynatrace url to authenticate against
* **azure-appconfig:** Connectionstring to Azure App Configuration 

Example

```text
- name: "Build and publish docker container"
    uses: equinor/iaf-actions/build-docker-container@v1.0
    with:
        name: applicationName
        build-id: ${{ github.sha }}
        registry: ${{ secrets.acr_registry }}
        registry-repository: ${{ secrets.acr_repository }}
        client-id: ${{ secrets.clientid }}
        client-secret: ${{ secrets.clientsecret }}
        dynatrace-tenant: ${{ vars.dynatrace-tenant }}
        dynatrace-token: ${{ secrets.dynatrace-token }}
        dynatrace-url: ${{ vars.dynatrace-url }}
        azure-appconfig: ${{ secrets.appconfig }}
```
