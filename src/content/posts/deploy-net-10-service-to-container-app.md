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
        containerAppName: ${{ vars.AZURE_CONTAINER_APP_NAME }}
        containerAppEnvironment: ${{vars.AZURE_APP_ENV_NAME}}
        resourceGroup: ${{ secrets.AZURE_RESOURCE_GROUP }}
        imageToDeploy: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}-${{ needs.build-and-push.outputs.short_sha }}
        registryUrl: ${{ env.REGISTRY }}
        registryUsername: ${{ github.actor }}
        registryPassword: ${{ secrets.PACKAGE_ACCESS_TOKEN }}
```

 For the most part, this is a pretty standard workflow file so I won't go over every details, just the important ones that should be called out. In the permissions section, we are granting write permissions for id-token and read for contents. This allows the workflow to read the contents of the github repository and to exchange the GITHUB_TOKEN value for a OpenID Connect (OIDC) token from your Entra Id App Registration. If this sounds like black magic, please take a look at the "Github - Azure OIDC Token Exchange" section for a simplified explanation of how it works. 

 Other than that, the workflow the split into two jobs. One to build and push the docker image to the Github Container Registry and another to perform the deployment of the newly built image to the azure container app. Don't worry about what the values for all the environment variables and secrets need to be just yet. I'll be covering that in just a sec.

#### Github - Azure OIDC Token Exchange
You can picture this interaction as similar to passport control at the airport. The officer will check your documents and stamp your passport if they deem you should have access to their country. At a later point during your trip, you can show the stamp in your passport as proof of legal presence.

**Initial Token Exchange**
:::github{align="right"}
Hi, I want to get access to deploy to container app abc.
:::

:::azure{align="left"}
Who are you?
:::

:::github{align="right"}
Here is my ID token from github that says I am from repository xyz.
:::

:::azure{align="left"}
Hmm... okay. Here's your Azure access token that says you can deploy for the next 10 minutes.
:::

**A couple seconds later**

:::github{align="right"}
Hi, I want to deploy a new image to container app abc.
:::

:::azure{align="left"}
Do you have access?
:::

:::github{align="right"}
Here is my access token from azure that says I can.
:::

:::azure{align="left"}
Okay, I will allow this deployment to container app abc.
:::

#### How to set it up
##### Azure
In the azure side you will need to do the following:

1. Navigate to the Microsoft Entra Id home page at [https://entra.microsoft.com](https://entra.microsoft.com)
2. Click on App Registrations on the left hand side menu
3. Click on "+ New Registration"
4. Name your new registration something readable like "Container App .NET 10 Service Deployment User" and click Register
5. On the next page, note the client id and tenant id. You will need these later in Github.
6. Click on Certificates and Secrets on the left hand side menu
7. Click on the Federated Credentials tab and click the button to add a new credential
8. In the scenario dropdown, choose "Github Actions deploying Azure resources". You will now see some fields you have to fill out.
9. If you're using github for personal development, the organization name will be the name of your github account. (for me, this was gouthampai)
10. Fill in the name of your repository and the name of the environment that will be correspond to the azure deployment. If you will be using this to deploy to more than one environment in azure, you might prefer to link it deployments with a specific branch or tag. I opted to use the environment set as "production" in github.
11. Give the credential a name and finish things up.
12. The last thing you'll need to do is to grant the service principal access to deploy to your new container app. In the azure portal, navigate to the container app you created and click on Access Control (IAM).
13. You'll want to grant the role of Container Apps Contributor to the service principal you just created.

##### Github
In Github, we are first going to go to Profile > Settings > Developer Settings > Personal Access Tokens > Tokens (classic).
Here we are going to create a github PAT so that our container app can pull down the image we deploy to it when it needs to scale up the number of replicas. 
1. Add a new PAT, give it a readable name like "container apps image pull access token"
2. Under Scopes, all it needs to have access to is read:packages.
3. Generate the token value and record it someplace safe. We will need it very soon.
4. Put a reminder in your calendar to rotate the token secret a few days before expiry 🙂

Next, navigate to your repository and go to Settings
1. Click on Environments in the side bar and add a new environment called "production".
2. Check the box under Required Reviewers and add yourself. This will make it so you have to approve each deployment.
3. Next we will add some environment variables
    - AZURE_APP_ENV_NAME - the name of the container app environment you created earlier
    - AZURE_CONTAINER_APP_NAME - the name of the container app you created earlier
    - AZURE_RESOURCE_GROUP - the name of the resource group your container app belongs to
    
4. Right now, we will also add a few secrets
    - AZURE_CLIENT_ID - the client id you copied down before
    - AZURE_SUBSCRIPTION_ID - the subscription id of your azure subscription
    - AZURE_TENANT_ID - the tenant id you copied down before while creating the app registration
    - PACKAGE_ACCESS_TOKEN - the PAT you created before with packages:read access

### Time to deploy!
Assuming you have set everything up correctly, you should be able to kick off a workflow in your github repository and see it successfully deploy to your new container app. Let's test if your newly deployed container app web api works! In Postman, make a http request to your container app. If you need to, grab the url for your container app from Azure. It should be something ending in `azurecontainerapps.io`. Make a request to `<your container app url>/todos/1` and you should get back some JSON as you did before when running the service locally. Congrats! You have successfully deployed a .NET 10 Web API to an Azure Container App!

#### Sources
[Deploy to Azure Container Apps with Github Actions](https://learn.microsoft.com/en-us/azure/container-apps/github-actions)
[ACA Core Components Overview](https://azure.github.io/aca-dotnet-workshop/aca/00-workshop-intro/1-aca-core-components/)
[Configure a GitHub Action to create a container instance](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-github-action#configure-github-workflow)
