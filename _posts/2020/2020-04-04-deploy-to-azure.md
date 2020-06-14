---
title: "Deploy your React app to Azure"
categories:
  - JavaScript
  - React
  - Azure
---

I've got to the point with my template where I am thinking about deployment options.  There is already a great option (the `gh-pages` module) for deploying to a github.io site.  However, I am going to be running most of my services on Microsoft Azure.  I want to deploy my service automatically and copy the web application to a web site on Azure.

Just like AWS, you can deploy a website directly to Azure Storage.  This is a great option if you just want cheap web hosting plus the option of consuming extra resources.

If you remember from [my first article]({% post_url 2020/2020-03-29-parcel-typescript-react %}), I put the web application in its own directory within the git repository.  This was to facilitate adding an infrastructure folder for storing the backend deployment scripts.

> I use [Terraform](https://terraform.io) for deployment.  Make sure you download and install both the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) and [Terraform](https://www.terraform.io/downloads.html).  You need to be able to run the `terraform` and `az` commands.  Login to Azure with the Azure CLI before continuing.

## Deploying an Azure Storage site

Although I use Terraform for deployment, I run it via `npm`.  This allows me to integrate into a common task runner.  

* Create an `infrastructure` directory.
* In the `infrastructure` directory, run `npm init -y`.
* Create a `.gitignore` file (I use [gitignore.io](https://gitignore.io/api/terraform))
* Add `configuration.json` to the `.gitignore` file (we'll why in a moment)

Take a look at the script section of my `package.json` file:

```json
{
  "name": "parcel-typescript-template-infrastructure",
  "version": "1.0.0",
  "description": "Infrastructure deployment for parcel-typescript-template",
  "author": "Adrian Hall <photoadrian@outlook.com>",
  "license": "MIT",
  "private": true,
  "scripts": {
    "predeploy": "terraform validate",
    "deploy": "terraform apply -auto-approve",
    "postdeploy": "terraform output -json > configuration.json",
    "destroy": "terraform destroy",
    "postinstall": "terraform init"
  }
}
```

It isn't very big, and comes down to three command:

* `npm install` will install the Terraform modules you need.
* `npm run deploy` will deploy the infrastructure and write out the configuration file.
* `npm run destroy` will destroy the infrastructure.

There is one other file we need - `resources.tf` is the explanation of the resources to deploy.  You can split the resources into many files if you like.  My backends tend to be small enough that I like to keep them in one file.  Here is my basic configuration:

```terraform
######################################################################
###
###       INPUT VARIABLES
###
######################################################################
variable "appname" {
  type = string
  default = "appname"
  description = "The short name for the application"
}

variable "region" {
  type = string
  default = "westus2"
  description = "The Azure region used for resources"
}

variable "environment" {
  type = string
  default = "dev"
  description = "The environment (dev, stage, test, prod, etc.)"
}

######################################################################
###
###       PROVIDER DEFINITIONS
###
######################################################################
provider "azurerm" {
  version = ">= 2.3.0"
  features {}
}

provider "random" {
  version = "~> 2.2.1"
}

######################################################################
###
###       LOCAL VARIABLES
###
######################################################################
resource "random_string" "longid" {
  length = 23
  upper = false
  lower = true
  number = true
  special = false
}

locals {
  shortname   = "s${random_string.shortid.result}"
  longname    = "s${random_string.longid.result}"
  defaultname = "${var.appname}-${var.environment}"

  tags = {
    ApplicationName = var.appname
    Environment = var.environment
  }
}

######################################################################
###
###       RESOURCES
###
######################################################################

resource "azurerm_resource_group" "rg" {
  name                      = local.defaultname
  location                  = var.region
  tags                      = local.tags
}

resource "azurerm_storage_account" "storage" {
  name                      = local.longname
  resource_group_name       = azurerm_resource_group.rg.name
  location                  = azurerm_resource_group.rg.location
  tags                      = local.tags

  account_kind              = "StorageV2"
  account_tier              = "Standard"
  account_replication_type  = "LRS"
  static_website {
    index_document          = "index.html"
  }
}

######################################################################
###
###       OUTPUTS
###
######################################################################
output "AZURE_STORAGE_CONNECTION_STRING" {
  value = azurerm_storage_account.storage.primary_connection_string
}

output "WEBSITE" {
  value = azurerm_storage_account.storage.primary_web_endpoint
}
```

This creates a resource group (a container for multiple related resources) and a storage account.  By configuring the `static_website`, a container called `$web` is created as well.

You can go ahead and run `npm install` followed by `npm run deploy` in the `infrastructure` directory now.  It will deploy the resources for you and create the `configuration.json` file (that file we added to the `.gitignore` file).

> You can put any other resources you commonly use here.  For example, I regularly add Application Insights, so I would add that to the template as well.

## Deploying the web application

Moving over to the web application, it needs to be copied to the storage service I have just created.  There is no cross-platform way of doing this that uses the output file from Terraform, but the script isn't that hard to write.  First, I need some libraries.  Change directory to the `webapp`, then run:

```bash
$> npm i -D readdirp mime-types @azure/storage-blob
```

The `readdirp` is a recursive version of `readdir`, which reads directory contents.  The `mime-types` package converts a file into the appropriate MIME type based on extension, thus allowing me to set the content type within Azure Storage when I upload it.  The `@azure/storage-blob` package is the Azure SDK for Azure Storage, which we will use to upload the files to Azure Storage.

Now for the script.  I place scripts in a handy `scripts` directory, since this is unlikely to be the last script.  Here is `deploy_to_azure.js`:

```javascript
#!/usr/bin/env node
const fs = require('fs');
const path = require('path');
const mime = require('mime-types');
const readdirp = require('readdirp');
const { BlobServiceClient } = require('@azure/storage-blob');
const configuration = require('../../infrastructure/configuration.json');

/**
 * Uploads a file to the designated container.
 *
 * @param {ContainerClient} container the container client to upload to.
 * @param {String} filename filename of the file within the container
 * @param {String} filepath filename for the source file (fully-qualified)
 */
const uploadToContainer = async (container, filename, filepath) => {
  console.log(`--> Uploading ${filename}`);
  const blobClient = container.getBlockBlobClient(filename);
  const options = {
    blobHTTPHeaders: {
      blobContentType: mime.lookup(filepath) || 'application/octet-stream'
    }
  };
  try {
    const response = await blobClient.uploadFile(filepath, options);
  } catch (error) {
    console.log(`--> Upload results: (ERROR) ${error.message}`);
  }
};

// Create a connection client to the Azure Storage service.
const connectionString = configuration.AZURE_STORAGE_CONNECTION_STRING.value;
const serviceClient = BlobServiceClient.fromConnectionString(connectionString);
const containerClient = serviceClient.getContainerClient('$web');
const sourcedir = path.resolve(__dirname, '../dist');

console.log('Upload to Azure Storage:');
console.log(`--> Source: ${sourcedir}`);
console.log(`--> Destination: ${containerClient.url}`);

// Get all the files within the dist directory and transfer them to the
// storage container with the same name.
readdirp(sourcedir)
  .on('data', ({ path: filename, fullPath: filepath }) => {
    uploadToContainer(containerClient, filename, filepath);
  })
  .on('warn', (warning) => console.warn('[WARNING] ', warning))
  .on('error', (error) => console.error('[ERROR]', error))
  .on('end', () => console.info('[DONE]'));
```

The script is relatively easy to understand.  First, I create a client for sending data to Azure Storage using the connection string within the `configuration.json` file that is generated when I run the `terraform output` command.  Then I loop through each file in the `dist` directory (recursively), uploading each one in turn.  The only thing I have to be aware of is to set the MIME type so that it can be downloaded correctly.

Hooking this into the `package.json` is simple enough:

```json
"scripts": {
  // ... rest of scripts
  "predeploy": "run-s build",
  "deploy": "node ./scripts/deploy_to_azure.js",
},
```

Each time I run `npm run deploy` within the `webapp` directory, it will do a build, then copy the build up to Azure.  I've got a pre-deploy step that also runs clean, lint, and test scripts so that the uploaded code is as clean as possible.

## Put it together

I don't really want to have to remember the ordering.  The best deployment scripts "just work".  Let's add some scripts at the top level of the git repository.  Here is the `package.json` file:

```json
{
  "name": "parcel-typescript-template",
  "version": "1.0.0",
  "author": "Adrian Hall <photoadrian@outlook.com>",
  "license": "MIT",
  "description": "Template repository for a deployable React app",
  "keywords": [
    "react",
    "azure",
    "template",
    "parcel",
    "terraform"
  ],
  "private": true,
  "scripts": {
    "deploy": "run-s infrastructure:deploy webapp:deploy",
    "postinstall": "run-s infrastructure:install webapp:install",
    "infrastructure:deploy": "cd infrastructure && npm run deploy",
    "infrastructure:destroy": "cd infrastructure && npm run destroy",
    "infrastructure:install": "cd infrastructure && npm install",
    "webapp:deploy": "cd webapp && npm run deploy",
    "webapp:install": "cd webapp && npm install"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/adrianhall/parcel-typescript-template.git"
  },
  "bugs": {
    "url": "https://github.com/adrianhall/parcel-typescript-template/issues"
  },
  "homepage": "https://github.com/adrianhall/parcel-typescript-template#readme",
  "devDependencies": {
    "npm-run-all": "^4.1.5"
  }
}
```

Take a look at the scripts.  If I run `npm install`, it will do the install locally, then run the install in both the `infrastructure` and `webapp` directories.  The same happens if I run `npm run deploy` - first the `infrastructure` version is run, then the webapp version is run.  If there is an error along the way, the rest of the deployment is stopped.

Finally, I added a `infrastructure:destroy` command to destroy the infrastructure.  I'm not producing a shortened version since I want you to be extra-sure you want to do that.  A few extra characters won't mean much when you want to do it, but it makes the world of difference when your fingers are on auto-pilot.

So, to recap:

* `npm install` installs everything.
* `npm run deploy` deploys everything.
* `npm run infrastructure:destroy` removes everything.

Using `npm` as a task runner is an easy way to get some automation into the process.  Utilizing the Azure SDK for Storage allows me to easily use the output of the Terraform deployment to upload the web application automatically.

I hoped you like this series on setting up a new React template as an alternative to CRA.  You can find this template on [my GitHub repository](https://github.com/adrianhall/parcel-typescript-template/tree/v0.4).
