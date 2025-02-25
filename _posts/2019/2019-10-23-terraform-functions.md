---
title: "Deploying an Azure Function App with Terraform"
categories:
  - Cloud
tags:
  - azure_functions
  - terraform
---

You may have caught this from my previous blog posts, but I like automated deployments.  I like something where I can run one command and magic happens, resulting in my whole deployment changing to a new state.  I've recently been looking around at options for Azure, checking out [Serverless Framework](https://serverless.com), Azure Resource Manager (ARM), and others.  My favorite thus far has been [Terraform](https://terraform.io).  These are the instructions for deploying a basic Azure Function app with TypeScript code from start to finish.

Let's first of all go through all the steps at a high-level:

1. Get your environment in order.
2. Create an Azure Function app.
3. Create a Terraform module describing your infrastructure.
4. Adjust the Azure Function app to produce a deployment file.
5. Run the deployment.

When re-deploying, you just edit your code and re-run the last step.

## Get your environment in order.

You need some basic software to get started:

* Node and npm (so you can compile TypeScript).
* Terraform (install from [their website](https://www.terraform.io/downloads.html) or via Chocolatey).
* Visual Studio Code with the Azure Functions extension.

In addition, your Azure account must be configured for Terraform.  Fortunately, [Azure has made the instructions easily accessible](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/terraform-install-configure#set-up-terraform-access-to-azure).  Their instructions require the use of the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows?view=azure-cli-latest), installation of which is an additional step.

## Create an Azure Function app.

I use [Visual Studio Code](https://visualstudio.com/code) to [create an Azure Function](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-first-function-vs-code), selecting TypeScript as the language.  However, there was a lot of old stuff in there, so I've updated the `package.json` file, as you will see later.  For right now, what you have is good enough.  This isn't about "how I write my Azure Functions".  That's a topic for another time.

## Create a Terraform module.

This is the first piece I'm going into some detail about.  The Terraform module is written in a `.tf` file.  It's main job is to describe the infrastructure.  All modules are comprised of resources, which can be variables, generated strings, and actual cloud resources.  Let's go through my file bit-by-bit.  The first step is to define some variables:

{% highlight terraform %}
variable "prefix" {
    type = "string"
    default = "twphoto"
}

variable "location" {
    type = "string"
    default = "westus"
}

variable "environment" {
    type = "string"
    default = "dev"
}

variable "functionapp" {
    type = "string"
    default = "./build/functionapp.zip"
}

resource "random_string" "storage_name" {
    length = 24
    upper = false
    lower = true
    number = true
    special = false
}
{% endhighlight %}

The `variable` declarations can be overridden on the command line.  I'm basically defining a prefix for all the resources (except one - see below), the location for all the resources, and the location of my build file - a zipped up file of all the code for the function app.  

The final `resource` is a random string.  Azure Storage requires a name that is 3-24 characters long and consists only of lower-case letters and numbers.  Since I don't have to access this storage account from a client, I can generate a random name for it.  I suppose the name might conflict at some point, but I doubt it.

Next, let's create a resource group and the storage account.  Storage in Azure consists of containers, and containers hold the blobs of data you actually want to store. 

{% highlight terraform %}
resource "azurerm_resource_group" "rg" {
    name = "${var.prefix}-${var.environment}"
    location = "${var.location}"
}

resource "azurerm_storage_account" "storage" {
    name = "${random_string.storage_name.result}"
    resource_group_name = "${azurerm_resource_group.rg.name}"
    location = "${var.location}"
    account_tier = "Standard"
    account_replication_type = "LRS"
}

resource "azurerm_storage_container" "deployments" {
    name = "function-releases"
    storage_account_name = "${azurerm_storage_account.storage.name}"
    container_access_type = "private"
}

resource "azurerm_storage_blob" "appcode" {
    name = "functionapp.zip"
    storage_account_name = "${azurerm_storage_account.storage.name}"
    storage_container_name = "${azurerm_storage_container.deployments.name}"
    type = "block"
    source = "${var.functionapp}"
}
{% endhighlight %}

Here, I am creating a resource group (which, in my default case, will be called `twphoto-dev`), a storage account (which has a random name), and a deployment container.  Then I upload my build ZIP file (more on that later) to the blob.

Next, I want to create a Function app.  This requires an App Service Plan (I'm going to be using the consumption plan), the app itself, and a shared access secret that allows the function app to access the blob I uploaded.

{% highlight terraform %}
data "azurerm_storage_account_sas" "sas" {
    connection_string = "${azurerm_storage_account.storage.primary_connection_string}"
    https_only = true
    start = "2019-01-01"
    expiry = "2021-12-31"
    resource_types {
        object = true
        container = false
        service = false
    }
    services {
        blob = true
        queue = false
        table = false
        file = false
    }
    permissions {
        read = true
        write = false
        delete = false
        list = false
        add = false
        create = false
        update = false
        process = false
    }
}

resource "azurerm_app_service_plan" "asp" {
    name = "${var.prefix}-plan"
    resource_group_name = "${azurerm_resource_group.rg.name}"
    location = "${var.location}"
    kind = "FunctionApp"
    sku {
        tier = "Dynamic"
        size = "Y1"
    }
}

resource "azurerm_function_app" "functions" {
    name = "${var.prefix}-${var.environment}"
    location = "${var.location}"
    resource_group_name = "${azurerm_resource_group.rg.name}"
    app_service_plan_id = "${azurerm_app_service_plan.asp.id}"
    storage_connection_string = "${azurerm_storage_account.storage.primary_connection_string}"
    version = "~2"

    app_settings = {
        https_only = true
        FUNCTIONS_WORKER_RUNTIME = "node"
        WEBSITE_NODE_DEFAULT_VERSION = "~10"
        FUNCTION_APP_EDIT_MODE = "readonly"
        HASH = "${base64encode(filesha256("${var.functionapp}"))}"
        WEBSITE_RUN_FROM_PACKAGE = "https://${azurerm_storage_account.storage.name}.blob.core.windows.net/${azurerm_storage_container.deployments.name}/${azurerm_storage_blob.appcode.name}${data.azurerm_storage_account_sas.sas.sas}"
    }
}
{% endhighlight %}

The Shared Access Secret (or SAS) is a cryptographic set of query params that I can add to the URL for the blob to allow me to do something to it.  In this case, I want to be able to read it for a long time.  Once I have the SAS, I can construct the package that gets injected into the `WEBSITE_RUN_FROM_PACKAGE` app setting for the Function app.  There is a piece of magic code here.  The Function app will only use the code in the blob if the computed hash matches the hash you specify in the app settings.  The computed hash takes the SHA256 hash of the file and then base64 encodes it.  Fortunately, Terraform has functions for this.

Now that you have a `.tf` file, you can do the following to download the resource providers:

{% highlight bash %}
terraform init
{% endhighlight %}

It will tell you that it has downloaded the `azurerm` and `random_string` providers since those are the two we are using.  There are lots of resource providers - for other cloud providers and for special circumstances.  You can now validate that the module is syntactically correct:

{% highlight bash %}
terraform validate
{% endhighlight %}

Get a clean bill of health here, and you are ready to move to the next step.  If not, you will see errors to be corrected on the screen.  They are generally typing errors.

## Adjust your Function environment

When you create a function app, it isn't set up for Functions + Terraform.  It's set up for a Visual Code + Functions deployment.  We need to adjust both the `package.json` so that it will produce the ZIP file for us, and the `.gitignore` so that it ignores the Terraform build files.  I use a bunch of helper NPM packages:

* `azure-functions-core-tools` for the `func` command.
* `@ffflorian/jszip-cli` to ZIP my files up.
* `mkdirp` for creating directories.
* `npm-run-all` and particularly the `run-s` command for executing things in order.
* `rimraf` for deleting things.

My prototype `package.json` looks like this:

{% highlight json %}
{
  "name": "backend",
  "version": "1.0.0",
  "description": "",
  "scripts": {
    "func": "func",
    "clean": "rimraf build",
    "build:compile": "tsc",
    "build:prune": "npm prune --production",
    "prebuild:zip": "mkdirp --mode=0700 build",
    "build:zip": "jszip-cli",
    "build": "run-s clean build:compile build:zip",
    "predeploy": "npm run build",
    "deploy": "terraform apply"
  },
  "dependencies": {
  },
  "devDependencies": {
    "azure-functions-core-tools": "^2.7.1724",
    "@azure/functions": "^1.0.3",
    "@ffflorian/jszip-cli": "^3.0.2",
    "mkdirp": "^0.5.1",
    "npm-run-all": "^4.1.5",
    "rimraf": "^3.0.0",
    "typescript": "^3.3.3"
  }
}
{% endhighlight %}

Note that my scripts have pretty much all changed.  I run a lot on Windows.  Whenever you see `&&` in the scripts section, replace it with an equivalent `run-s` command.  I have two basic commands I run:

* `npm run build` will build the ZIP file.
* `npm run deploy` will build the ZIP file and deploy it to Azure.

I need to explicitly configure `.gitignore` by adding the `build` directory and [the Terraform gitignore.io profile](https://www.toptal.com/developers/gitignore/api/terraform) to the file.  Finally, I need a configuration file for the `jszip-cli` command (called `.jsziprc.json`):

{% highlight json %}
{
    "compressionLevel": 9,
    "dereferenceLinks": true,
    "entries": [
        "."
    ],
    "force": true,
    "ignoreEntries": [
        ".terraform",
        "build",
        "*.zip",
        "*.map",
        "terraform.*",
        "*.tf"
    ],
    "mode": "add",
    "outputEntry": "build/functionapp.zip",
    "quiet": false,
    "verbose": false
}
{% endhighlight %}

This creates a ZIP file from the current directory contents, but excluding all the Terraform stuff plus the build directory and ZIP files.

## Ready to deploy!

Now you can run `npm run deploy`.  Terraform will tell you what it is going to do (hint: the first time through, it's going to create resources).  Then you type `yes` to proceed and watch for a couple of minutes as the magic happens.  The nice thing about this (and the advantage over ARM) is that everything is in one command - the compilation of TypeScript, generation of the storage container, upload of the blob and deployment of all resources is done in a single step.  You get early warning (and the process bails) if your code doesn't compile. 

## Destroying your infrastructure.

You can destroy your infrastructure using:

{% highlight bash %}
terraform destroy
{% endhighlight %}

That's a convenient set-up and tear-down process!  It's also much faster than doing the same thing through the portal.

## Next steps.

You can also extend this to locally run your functions (instead of deploying them) and to run unit tests if you like.  You just need to adjust the `package.json` scripts or the resource file.  My next steps are to work out the best way of integrating App Insights and EasyAuth into this process.
