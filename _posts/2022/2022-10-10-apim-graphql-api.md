---
title: "Build a GraphQL API on Azure API Management using Bicep"
categories:
  - Azure
tags:
  - API Management
  - Bicep
---

When I build a service in the cloud, I describe the infrastructure as a blob of code.  There are lots of solutions out there for this.  Azure has the Azure Resource Manager (or ARM), which has it's own JSON or YAML format, for example.  Terraform is cross-cloud capable, as is the Serverless Framework.  Since I mostly work in Azure, these days, I'be been working more and more with [Bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview) for my Infrastructure as Code standard.  Bicep uses declarative syntax to deploy Azure resources (much like Terraform), but it's specific to Azure, so it understands things a little better.  There is a [Visual Studio Code extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-bicep) to help you write the required files.  The [Azure CLI](https://learn.microsoft.com/azure/azure-resource-manager/bicep/install#azure-cli) and [Azure PowerShell](https://learn.microsoft.com/azure/azure-resource-manager/bicep/install#azure-powershell) both understand Bicep, so there are no extra tools to install.  I tend to use the Azure CLI for most things since it is cross-platform and I work a lot on Macs as well.

## Creating an Azure API Management instance

I split my work into two pieces.  First, I have the settings for the Azure API Management instance.  I don't create the APIM service repeatedly, so why should I check it all the time?  It's just extra time in the CI/CD process I can spend elsewhere.  As a result, I have a separate file for the APIM service.  Here is what the `apim-service.bicep` file looks like:

``` js
@description('The name of the API Management service instance')
param apiManagementServiceName string = 'apim${uniqueString(resourceGroup().id)}'

@description('The email address of the owner of the service')
@minLength(1)
param publisherEmail string = 'photoadrian@outlook.com'

@description('The name of the owner of the service')
@minLength(1)
param publisherName string = 'Adrian Hall'

@description('The pricing tier of this API Management service')
@allowed([
  'Consumption'
  'Developer'
  'Standard'
  'Premium'
])
param sku string = 'Developer'

@description('The instance size of this API Management service.')
@allowed([ 1, 2 ])
param skuCount int = 1

@description('Location for all resources')
param location string = resourceGroup().location

resource apiManagementService 'Microsoft.ApiManagement/service@2021-08-01' = {
  name: apiManagementServiceName
  location: location
  sku: {
    name: sku
    capacity: skuCount
  }
  properties: {
    publisherEmail: publisherEmail
    publisherName: publisherName
  }
}
```

The top part of the file describes all the parameters I can set on it.  However, they are all defaulted to my standard stuff.  Basically, I always will default things to the "developer" settings.  Then I have a parameters file that sets things up for "production".  That way I'm not affecting production unless I have to.  An example `apim-service-parameters.json` file looks like this:

``` json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "apiManagementServiceName": "api-prod",
    "publisherEmail": "service-team-distribution-list@company.com",
    "publisherName": "Service Team for API Management",
    "sku": "Premium",
    "skuCount": 1
  }
}
```

If your service runs inside a VNET, you can set up the entire private networking setup using Bicep as well.  Similarly, this is where you will set up custom domains, notification sender emails, private endpoints, and any other service wide settings.  You can find all the settings that you can set on the service in the [REST API Reference](https://learn.microsoft.com/en-us/rest/api/apimanagement/current-ga/api-management-service/create-or-update).

On to deploying - I do this with an Azure CLI:

``` bash
az group create --name dev-myservice-ahall --location westus
az deployment group create --resource-group dev-myservice-ahall --template-file ./apim-service.bicep
```

I just need to add `--template-parameter-file .\apim-service-parameters.json` to the end of the deployment command to set it up for production.  Again - the idea is make setting up a developer instance easy; make it harder to mess with production.

## Building the GraphQL API

A GraphQL API is comprised of three parts: an API definition, a GraphQL schema, and a policy file.  For this demonstration, I'm going to use three distinct files.  First, the policy file, which I call `policy.xml`:

``` xml
<policies>
    <inbound>
        <base/>
    </inbound>
    <backend>
        <base/>
    </backend>
    <outbound>
        <base/>
    </outbound>
</policies>
```

Yep - it's an empty policy file.  It's really here so I can manage the policies later on.  If you install the [Visual Studio Code extension for Azure API Management](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-apimanagement), you get syntax highlighting as well!  On to the `graphql.schema` file:

``` graphql
type Query {
  info: String!
}
```

Again - nothing to it.  In reality, the `policy.xml` and `graphql.schema` file are going to be the ones you are editing most often.  Once the API is configured, you likely won't edit the bicep file too much, but you will edit the policies and schemas.

Talking of the `api.bicep` file, we need to set up two resources - the first is the API definition itself, and the second is the policy resource.

``` js
@description('The name of the API Management service instance')
param apiManagementServiceName string = 'apim${uniqueString(resourceGroup().id)}'

@description('The name of the GraphQL API')
@minLength(1)
param apiName string = 'graphql'

@description('The path of the API')
@minLength(1)
param apiPath string = 'graphql'

@description('The description of the API')
@minLength(1)
param apiDescription string = 'A GraphQL API via Bicep'

@description('The Display Name of the API')
@minLength(1)
param apiDisplayName string = 'Bicep GraphQL'

resource apimService 'Microsoft.ApiManagement/service@2021-08-01' existing = {
  name: apiManagementServiceName
}

resource graphqlApi 'Microsoft.ApiManagement/service/apis@2021-12-01-preview' = {
  name: apiName
  parent: apimService
  properties: {
    description: apiDescription
    displayName: apiDisplayName
    path: apiPath
    protocols: [ 'https', 'wss' ]
    subscriptionRequired: false
    type: 'graphql'
  }
}

resource graphqlSchema 'Microsoft.ApiManagement/service/apis/schemas@2021-12-01-preview' = {
  name: 'graphql'
  parent: graphqlApi
  properties: {
    contentType: 'application/vnd.ms-azure-apim.graphql.schema'
    document: {
      value: loadTextContent('./graphql.schema')
    }
  }
}

resource graphqlPolicy 'Microsoft.ApiManagement/service/apis/policies@2021-12-01-preview' = {
  name: 'policy'
  parent: graphqlApi
  properties: {
    format: 'rawxml'
    value: loadTextContent('./policy.xml')
  }
}

```

The first note to make here is that we need to specify the existing API Management service.  The `uniqueString()` function generates a repeatable string based on a seed - in this case, the resource group name.  As a result, the same service name will be used in both the original bicep (that I used to create the service) and the API creation bicep file.  

Once I have the service, I can create the GraphQL API.  There is only one real "gotcha" here - you must specify the protocols.  Here, I am allowing https and secure websockets (but not http+ws).

> A good way to figure out what properties to use: create the API in the Azure Portal, then use **Automation > Export template** in the left hand menu to generate the ARM template.  The properties section for each resource are identical in Bicep files.

Now we come to the policy and schema.  These are each a resource that has the GraphQL API as the parent.  However, there are a couple of "magic strings" that you just have to know.  The first is the content type in the schema resource, and the second is the format in the policy resource.  Again, exporting the ARM template helps to understand what these are.

You can find the specifics of each properties section in the [REST API documentation](https://learn.microsoft.com/rest/api/apimanagement/current-ga/apis/create-or-update).

You now have a Bicep file that **just** deploys the API.  Deploying a single API is much quicker than deploying the whole collection.  In fact, this is the premise of the [APIOps](https://github.com/azure/apiops) deployment methodology.  You can store the policies separately or together with your service definition.  

Deployin the API is simple:

``` bash
az deployment group create --resource-group dev-myservice-ahall --template-file ./graphql-api.bicep
```

Building APIs with policies and schemas is easy with Bicep.  I hope you'll give it a try!
