---
title: "Part 2 - Deploy a .NET 10 Service To Azure Container Apps Using Github Actions"
published: 2025-12-07
draft: false
description: 'Follow up to part 1 of the series. This one focuses on infrastructure and CI/CD for deploying .NET 10 services.'
series: '.NET 10 on Azure Container Apps'
tags: ['dotnet', 'azure', 'containers']
---

Welcome to part two of my series on working with .NET 10 and containers. I will be picking up where we left off last time. Last time, we spun up a new .NET 10 web api and got it building as a docker image locally. This time, I'll be focusing on setting up the infrastructure in Azure and wiring up Github Actions to deploy to Azure. 

### Setting up your Container App
Before we can deploy our code to Azure, we need to create the required resources. We will need to create a new Container App (CA) and an accompanying Container Apps Environment (CAE) resource.
The CA represents a single service we might want to deploy while the CAE represents a grouping of Container Apps. 

For example, you might have one service to serve up a UI and another one to perform server side processing of data. These two services would separately run and scale on their own respective CAs while being associated to the same CAE. In an enterprise scenario, you might choose to have your non-prod services associated with one CAE and your production services associated with a different CAE. 

I created my infrastructure using something similar to the terraform code [here](https://github.com/gouthampai/terraform-samples/tree/main/net-10-container-app-sample). You just need to copy what's there and provide the respective values in a tfvars file of your choice. After that, running `terraform apply -var-file="your-vars.tfvars" should create the needed infrastructure in Azure.

:::tip
To create container apps in Azure, you must have the registration enabled for the `Microsoft.App` resource provider in the Azure subscription. You can locate this setting by navigating to your Subscription resource in Azure and clicking on Resource Providers under the Settings drop down tab on the left hand side. Then, search for `Microsoft.App` in the "Filter by name" search box and click the necessary prompts to enable the registration.
:::

### Deploying to Azure using Github Actions
Let's get to the fun part of making all this work! Copy the below YAML code to a new file called `build-and-deploy.yml` under `.github/workflows` in the root of your repository.
```yml
name: Build and Deploy to Azure Container Apps

on:
  push:
    branches: [ main ]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image: ${{ steps.meta.outputs.tags }}
      short_sha: ${{ steps.short-sha.outputs.short_sha }}

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Generate short SHA
      id: short-sha
      run: echo "short_sha=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT

    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=raw,value=${{ github.ref_name }}-${{ steps.short-sha.outputs.short_sha }}
          type=raw,value=latest,enable={{is_default_branch}}

    - name: Build and push Docker image
      id: build
      uses: docker/build-push-action@v6
      with:
        context: .
        platforms: linux/amd64,linux/arm64
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy:
    runs-on: ubuntu-latest
    needs: build-and-push
    environment: production
    
    steps:
    - name: Log in to Azure
      uses: azure/login@v1
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: Deploy to Azure Container Apps
      uses: azure/container-apps-deploy-action@v1
      with:
        containerAppName: ${{ secrets.AZURE_CONTAINER_APP_NAME }}
        containerAppEnvironment: ${{secrets.AZURE_APP_ENV_NAME}}
        resourceGroup: ${{ secrets.AZURE_RESOURCE_GROUP }}
        imageToDeploy: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}-${{ needs.build-and-push.outputs.short_sha }}
        registryUrl: ${{ env.REGISTRY }}
        registryUsername: ${{ github.actor }}
        registryPassword: ${{ secrets.PACKAGE_ACCESS_TOKEN }}
```

Let's break down what each part does.
```yml
name: Build and Deploy to Azure Container Apps

on:
  push:
    branches: [ main ]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
```
The first part before we get to the jobs section specifies the name of the workflow, the conditions under which it runs, the permissions that GITHUB_TOKEN gets in this workflow and some env variables to be used during the course of the workflow run. This workflow is setup to trigger on pushes to main and on manual workflow triggers from Github. Moving onto the permissions section, we are granting write permissions for id-token and read for contents. This allows the workflow to read the contents of the github repository and to exchange the GITHUB_TOKEN value for a OpenID Connect (OIDC) token from your Entra Id App Registration.

#### How it works
:::github{align="right"}
Hi, I want a token to deploy a docker image to container app abc.
:::

:::azure{align="left"}
Who says you can deploy things to container app abc?
:::

:::github{align="right"}
Check the list, it says xyz repo in github can deploy to container app abc.
:::

:::azure{align="left"}
Sure does. Can you prove you're from github repo xyz?
:::

:::github{align="right"}
Here is my Github OIDC token. Take a look!
:::

:::azure{align="left"}
Hmm... okay. Here's your Azure token that will let you deploy. Have a nice day!
:::