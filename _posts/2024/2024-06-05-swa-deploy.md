---
title:  "Build a Blog: Deploy Azure Infrastructure three ways"
date:   2024-06-05
categories:
  - Azure
tags:
  - Azure Static Web Apps
  - Bicep
  - Azure Developer CLI
---

For most developers, dealing with the infrastructure part of the job is hard.  I like to say "give me a database and a web site" and prefer not to get into the other requirements.  My web sites and other cloud projects (including this one) are pretty open. So, what's the minimum I need to know to deploy stuff on Azure?

<!-- more -->

This post is part of a sequence showing how to deploy a blog on Azure Static Web Apps.  The topics include:

1. [Deploying Azure Static Web Apps]({% post_url 2024/2024-06-05-swa-deploy %})
2. Configuring Azure DNS
3. Configuring Static Web Apps Custom Domains
4. Taking Static Web Apps to Production

Today, I'm going to look at three ways to deploy the same thing.  That "thing" is an Azure Static Web App. I'll look at the techniques and why it is better (or worse) than the others.

Three ways?  Surely, there are more than that!  Why, yes.  There are.  You can use any number of infrastructure as code (IaC) tools, and there are lots of opinions on how to lay out an infrastructure for enterprise use. I prefer to "keep it simple"

When one is developing cloud code, THREE things need to happen to get the code on the cloud:

1. Provision the infrastructure.
2. Configure the infrastructure to accept the code.
3. Deploy the code to the infrastructure.

For the purposes of the walk-through today, I'm using the Jekyll site that I used to build this blog.  But you could use anything that generates static web content.

## The infrastructure

For my purposes, I only need one resource - an [Azure Static Web App](https://learn.microsoft.com/azure/static-web-apps/overview).  A static web app is a resource that holds and serves the HTML, CSS, JavaScript, and other assets for a web site.  It can be any web site - React, Vue, Angular, Svelte, Solid, Qwik, Next, Nuxt, Nest, or any other HTML/JS web framework.  You don't even need a web framework.  Use a static site generator like Jekyll (used to generate this site), Hugo, Docusaurus, Hexo, Vuepress, and many more can be used.

You only need the ability to generate the HTML, CSS, JavaScript, and assets for the web site.

Once you've deployed the code, the site is globally available and cached at the edge.  You can use GitHub Actions to deploy your code automatically - it's a zero downtime deployment (i.e. the new site doesn't take over until all the files are in place for the new site).

Once you are ready to go to production, you can upgrade the SKU and add a custom domain without any down time.

There are other good things about Azure Static Web Apps (and we'll discover them as we go along).  For now, let's look at the three ways to get your code running in the cloud.

## Method 1: The CLIs

Some people like to have control and this method gives me the ability to control exactly what is going on every step of the way. Let's take a look at what I need to do to provision, configure, and deploy this site.

### Prerequisites

I need a few tools.  If you took a look at [my last post]({% post_url 2024/2024-06-04-devcontainers %}), you'll find that I can configure a container that has all the tools I need in my dev container.  The links are to the installation page for the tool and don't include any tooling you need to build your application.  If you are deploying a Jekyll site, for example, you need to install Ruby and Jekyll before starting as well.  Since you are already developing code, you'll have those tools installed.

* [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)
* [NodeJS](https://nodejs.org)

You will also want to [sign in to Azure](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively) and [select a subscription](https://learn.microsoft.com/cli/azure/manage-azure-subscriptions-azure-cli) if you have access to more than one subscription.

### Create the resources

All resources are created inside a [resource group](https://learn.microsoft.com/azure/azure-resource-manager/management/overview) - a container for related resources, so that's the first step.

{% highlight bash %}
export RG="my-resourcegroup"
export REGION="centralus"
az group create -n $RG -l $REGION
{% endhighlight %}

Using bash variables helps you to avoid spelling mistakes, especially where the same name is re-used. The version for PowerShell is slightly different as a result.  Next, create the Static Web App:

{% highlight bash %}
export SWA="my-blog"
az staticwebapp create -n $SWA -g $RG -l $REGION
{% endhighlight %}

> You can also use [Azure PowerShell](https://learn.microsoft.com/powershell/azure/install-azure-powershell) and the [Azure portal](https://portal.azure.com/#view/Microsoft_Azure_Marketplace/GalleryItemDetailsBladeNopdl/id/Microsoft.StaticApp) for provisioning the resources.

### Configure the resources

The main thing to do here is to configure a `staticwebapp.config.json` file.  This is sent to Azure Static Web Apps to configure the service features you need to run your site.  I don't need anything beyond the basic hosting, so my file is about as simple as it can go:

{% highlight json %}
{
  "navigationFallback": {
    "rewrite": "/index.html"
  }
}
{% endhighlight %}

This tells Static Web Apps where the index page that I want to use is located.  Check this file into source code control.

### Deploy the code with the SWA CLI

I'm going to use the [Static Web Apps CLI](https://azure.github.io/static-web-apps-cli) to deploy the code (the HTML, CSS, and JavaScript) to the service.  This is one of those "npm" tools, so it's easy to install in the project:

{% highlight bash %}
npm add -D @azure/static-web-apps-cli
npx swa init --yes
{% endhighlight %}

> Like all "npm" tools, it downloads a **LOT** of packages from the package manager. Don't forget to add the `package.json` file to source code control and ensure that `node_modules` is added to the `.gitignore` file (or equivalent).

The initialization tries to detect what the application is based on and thus where it can expect the output or distributable files.  My build process is `bundle exec jekyll build` and the files are placed in `_site` after the build.  Check out the output of the `swa init` command:

![Screenshot of the swa init output](/assets/images/2024/2024-06-05-swa-init-output.png)

Now I can deploy the code:

{% highlight bash %}
npx swa login -R $RG -n $SWA
{% endhighlight %}

The first run-through of this command prompts me for subscription and tenant information.  You can avoid these prompts by setting environment variables before you run the command:

{% highlight bash %}
export AZURE_SUBSCRIPTION_ID=`az account show --query "id" --output tsv`
export AZURE_TENANT_ID=`az account show --query "tenantId" --output tsv`
{% endhighlight %}

However, I find this is too much typing for an infrequent operation.  I tend to switch to automated techniques after the first deployment, so I don't find the extra typing worth the effort.

Finally, deploy the code:

{% highlight bash %}
npx swa build
npx swa deploy -n $SWA
{% endhighlight %}

This will build my site and deploy the code to the static web app (downloading the tool that actually deploys the code if necessary).  Once the deployment is complete, the URL of the site is displayed. I can click on it to view the site.

## Method 2: Azure Developer CLI

Now that I know what I need to do with individual steps, I can start to automate the process.  The best tool for this is the [Azure Developer CLI](https://learn.microsoft.com/azure/developer/azure-developer-cli/) - an open-source tool maintained by Microsoft that bundles both an infrastructure deployment and a code deployment in one tool.  This tool makes it easy for me to get up and running fast, even if I need to start from scratch.

To use this tool, you need to:

* Write some [bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview) code to describe your infrastructure.
* Write an `azure.yaml` file to describe the deployment.
* Do any code changes necessary to support the infrastructure.
* Run the `azd up` command to deploy your code.

The nice thing about this method is that you can just run `azd up` again whenever you make any changes to either the infrastructure or the code.  I'll assume you have [installed azd](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) already, but that's the only pre-requisite here.

### Write the infrastructure code

All the infrastructure code is created inside an `infra` folder at the top level of your project.  It consists of a `main.bicep` file to describe the infrastructure, and a `main.parameters.json` file to adjust the deployment parameters.  Let's take a look at my `main.parameters.json` file first:

> Every single cloud provider has their own 'infrastructure as code' language.  For AWS, it's CloudFormation.  For Azure, it's called "ARM" (or Azure Resource Manager).  However, the good people of Microsoft decided ARM was too low-level and complex for the average user (they weren't wrong!)  As a result, bicep was born.  If you are a developer that targets Azure, bicep is well worth learning.

{% highlight json %}
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environmentName": {
        "value": "${AZURE_ENV_NAME}"
    },
    "location": {
        "value": "${AZURE_LOCATION}"
    }
  }
}
{% endhighlight %}

When you run the "azd" tool, it will prompt you for an environment name, location, and subscription to target for your application.  It's completely normal to pass these things into the bicep tooling so that your resource naming can be adjusted according to these values.  "azd" provides these values as environment variables, and this JSON file converts those environment variables into bicep parameters.

Static Web Apps supports just five regions (eastus, westus2, centralus, westeurope, and eastasia), so you'll want to select one of those when prompted.

Now, let's take a look at the bicep code.  I've removed all the comments from this code so that it reads well inside the blog, but you should definitely comment your code.

{% highlight bicep linenos %}
targetScope = 'subscription'

@description('The environment name - a unique string that is used to identify THIS deployment.')
param environmentName string

@description('The name of the Azure region that will be used for the deployment.')
param location string

var resourceToken = uniqueString(subscription().subscriptionId, environmentName, location)
var tags = { 'azd-env-name': environmentName }

resource rg 'Microsoft.Resources/resourceGroups@2024-03-01' = {
  name: 'rg-${environmentName}'
  location: location
  tags: tags
}

module swa 'br/public:avm/res/web/static-site:0.3.0' = {
  name: 'swa-${resourceToken}'
  scope: rg
  params: {
    name: 'swa-${environmentName}'
    location: location
    sku: 'Free'
    tags: union(tags, { 'azd-service-name': 'web' })
  }
}

output AZURE_LOCATION string = location
output SERVICE_URL string = 'https://${swa.outputs.defaultHostname}'
{% endhighlight %}

Let's walk through this:

* **Line 1** - bicep can operate assuming a subscription level deployment or a resource group deployment.  I need a subscription level deployment so that I can create a resource group.
* **Lines 3-7** - read in the values from the `main.parameters.json` - azd will set these for me.
* **Line 9** - create a unique string based on the subscription ID, environment name and location.  I'll use this if needed to generate a unique name for a resource or deployment.
* **Line 10** - create an object to use for tagging the resources I create.
* **Lines 12-16** - create a resource group to hold the resources being created.
* **Lines 18-27** - create a static web app inside the resource group with the Free SKU, tagged with a specific service name (for deployment).
* **Lines 29-30** - output the things I need to know about the deployment.

I'm using a module from the [Azure Verified Modules](https://aka.ms/AVM) collection here.  Azure Verified Modules (or AVM) is a collection of bicep modules that you can use to help configure your resources.  You can consider them "batteries included" modules.  They generally cover how to deal with diagnostics, resource locks, role assignments, and private networking in a standardized way across all resource types.  You should definitely consider using them in your infrastructure projects if you use bicep as they will significantly reduce the size of your bicep code.

## Write an azure.yaml file

The `azure.yaml` file is used by the Azure Developer CLI to link the resources deployed in the infrastructure step to the code that needs to be deployed on them.  You need to give each resources that has code deployed to it a unique name and set the `azd-service-name` tag to that name.  In my case, I've tagged my static web app with the name `web`.  Let's look at the `azure.yaml` file:

{% highlight yaml linenos %}
# yaml-language-server: $schema=https://raw.githubusercontent.com/Azure/azure-dev/main/schemas/v1.0/azure.yaml.json

name: apps-on-azure-blog
services:
  web:
    project: .
    dist: _site
    language: js
    host: staticwebapp
{% endhighlight %}

The `web` service (at line 5) corresponds to the service name that I specified in the bicep file (explicitly, it's the value of the 'azd-service-name' in the tags of the resource we created).

## Code changes

As in the first method, you will need to create a `staticwebapp.config.json` file.  I've copied the same file here in case you are following the instructions:

{% highlight json %}
{
  "navigationFallback": {
    "rewrite": "/index.html"
  }
}
{% endhighlight %}

## Deploy the code with the Azure Developer CLI

Finally, I can deploy my code.  First, sign in with the Azure Developer CLI:

{% highlight bash %}
azd auth login
{% endhighlight %}

And then run the end-to-end deployment:

{% highlight bash %}
azd up
{% endhighlight %}

You are prompted for additional information when running this for the first time (the environment name, subscription, and location).  You won't be prompted for these again.  The information is stored in a `.azure` directory within your project.

## Method 3: GitHub Actions

I now have a single and easy command to deploy my code.  But it's still a manual process - I have to do something.  It would be nice if the deployment fit with my workflow.  I use GitHub, so it would be nice if the site was deployed whenever I pushed my code to GitHub.  That's where GitHub Actions comes in.  I can write a workflow that automatically builds and deploys my site whenever I check in code.  We are delving into the area of "CI/CD" (which standard for **Continuous Integration / Continuous Deployment**)

There is a large selection of GitHub Actions to do just about anything you can think of. Of those, there are two actions (both written by Microsoft) that will support what I am trying to accomplish:

* [Azure Static Web Apps Deploy](https://github.com/marketplace/actions/azure-static-web-apps-deploy)
* [Azure Developer CLI](https://github.com/marketplace/actions/setup-azd)

Of these two, I prefer the Azure Developer CLI method.  The Azure Developer CLI method allows me to set up the deployment first (see Method 2 above) and then add the GitHub Action later on.  It also handles my infrastructure deployment and can deploy additional resources if necessary. If my code runs into a problem, I can diagnose it outside of the GitHub Actions mechanism, which means I don't need to use a runner.

The Static Web Apps Deploy method has the advantage that you don't need to set up a service principal in Entra ID so that your GitHub Action can communicate with Azure.  Instead, you need a deployment token that you can retrieve from the Azure Portal.  You can also set up the deployment from the Azure portal by linking the GitHub repository to your Static Web Apps resource. In addition, you can automatically deploy preview sites based on pull requests, giving you an automatic mechanism for previewing code before it is merged into production.

So, let's walk through setting up a GitHub Action using the Azure Developer CLI method.

First, create the `.github/workflows` directory. This is where the GitHub Action definitions live in your project, and you check the workflow definition into your git repository.

Next, create a workflow in that directory.  I called mine `deploy-blog.yml`:

{% highlight yaml linenos %}
{% raw %}
on:
  workflow_dispatch:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      AZURE_CLIENT_ID: ${{ vars.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ vars.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install azd
        uses: Azure/setup-azd@v1.0.0

      - name: Install nodejs
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install Ruby
        uses: ruby/setup-ruby@v1
        with:
            ruby-version: '3.1.3'
            bundler-cache: true

      - name: Build Site
        run: bundle exec jekyll build
        env:
          JEKYLL_ENV: production

      - name: Log in with Azure (Federated Credentials)
        if: ${{ env.AZURE_CLIENT_ID != '' }}
        run: |
          azd auth login `
            --client-id "$Env:AZURE_CLIENT_ID" `
            --federated-credential-provider "github" `
            --tenant-id "$Env:AZURE_TENANT_ID"
        shell: pwsh

      - name: Log in with Azure (Client Credentials)
        if: ${{ env.AZURE_CREDENTIALS != '' }}
        run: |
          $info = $Env:AZURE_CREDENTIALS | ConvertFrom-Json -AsHashtable;
          Write-Host "::add-mask::$($info.clientSecret)"

          azd auth login `
            --client-id "$($info.clientId)" `
            --client-secret "$($info.clientSecret)" `
            --tenant-id "$($info.tenantId)"
        shell: pwsh
        env:
          AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}
      
      - name: Provision Infrastructure
        run: azd provision --no-prompt
        env:
          AZURE_ENV_NAME: ${{ vars.AZURE_ENV_NAME }}
          AZURE_LOCATION: ${{ vars.AZURE_LOCATION }}
          AZURE_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Application
        run: azd deploy --no-prompt
        env:
          AZURE_ENV_NAME: ${{ vars.AZURE_ENV_NAME }}
          AZURE_LOCATION: ${{ vars.AZURE_LOCATION }}
          AZURE_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}
{% endraw %}
{% endhighlight %}

Let's go through this file line by line:

* **Lines 1-5** dictate when this action will be executed - in this case, either manually or when I push a change to the main branch.
* **Lines 7-9** indicate what permissions are required.
* **Lines 11-77** are the actual steps needed to deploy the code.
  * **Lines 13-18** set up the environment for the deployment.
  * **Lines 20-21** checks out the code.
  * **Lines 23-35** install tools necessary for the deployment.
  * **Lines 37-40** build the code for later deployment.
  * **Lines 42-63** sign in to Azure using the credentials I've provided in the environment.  There are two versions - one for a service principal and one for federated credentials.
  * **Lines 65-77** provision the resources, then deploy the code.

Before I can use this action, I need to establish some credentials for Azure in my GitHub repository.  These are stored as secrets. Use the following command to establish those credentials:

{% highlight bash %}
azd pipeline config
{% endhighlight %}

This command will copy your current environment to the GitHub Actions, log into Azure, and store those federated credentials as secrets with the GitHub Action as well.  I hope you named your environment the way you wanted to before you ran the command!

The command will ask if you want to push your local changes (say yes) and then it will display the link to the actions.  You can (and should) click on the actions link and see if your action was successful.

## Clean up resources

If you've been following along, you probably don't want the resources you created to stick around.  Even though the Azure Static Web Apps SKU is free, you only have 10 of them and you may want to use it for something else.  To clean up the resources, just delete the resource group:

{% highlight bash %}
az group delete -n $RG
azd down
{% endhighlight %}

Each command will take a couple of minutes.  When you run `azd down`, make sure you also disable the GitHub Action in your repository and [remove the secrets and variables that were stored](https://docs.github.com/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions) alongside it.

## Final thoughts

So, which method should you use?

* For **one-off deployments** and **tweaks to existing resources**, I reach for the Azure CLI and associated tools.  You wouldn't stress about using the Azure portal either - it's a one off - the method is really not important.
* For **repeatable deployments** and **continuous deployments** via GitHub Actions (or any other pipeline runner, including Azure Pipelines), I use the Azure Developer CLI.  It's a single command to spin up a development environment that is separate from my production environment, and it's easy to integrate into GitHub Actions.

## Further reading

* [Azure Developer CLI](https://learn.microsoft.com/azure/developer/azure-developer-cli/overview)
* [Azure Static Web Apps](https://learn.microsoft.com/azure/static-web-apps/overview)
* [Azure Verified Modules](https://aka.ms/AVM)
* [GitHub Actions](https://docs.github.com/actions)