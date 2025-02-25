---
title: "Choosing a hosting option for your web app in Azure"
categories:
  - Cloud
tags:
  - comparison
  - azure_swa
  - azure_appservice
  - azure_storage
---

Azure, like all the other clouds, has a plethora of mechanisms for getting your web site published.  You have [Static Web Apps](https://azure.microsoft.com/services/app-service/static/), [Azure App Service](https://azure.microsoft.com/services/app-service/), [Azure Blob Storage](https://docs.microsoft.com/azure/storage/blobs/storage-blob-static-website), [Web Apps for Containers](https://azure.microsoft.com/services/app-service/containers/), and then there are the compute ones like virtual machines, and Azure Functions.  It can be a little overwhelming.  For this analysis, I'm going to assume you want to deploy a stand alone web site - either straight HTML, CSS, and JavaScript or a React, Angular, Vue, or similar app.  I'm going to break it down based on how you deploy your site, how you scale your site, the basic cost structure (including the minimum cost, but excluding networking costs, which are additional but minimal in each case), and any restrictions you may have in running your site.

The big question is "what would I use today?"  For a production web site, I would use Azure App Service (also known as Azure Web Apps) - there is a free tier, it's easy to deploy, integrates with CI/CD solutions, and it has all the controls I generally need.  

I am, however, keeping a keen eye on Static Web Apps, as that promises to remove the need for me to think about scaling.  However, it's currently in preview, and requires you to use GitHub.  I would not use Static Web Apps today for production apps as a result.

Bringing all the options into a nice table:

|Service |Status  |Deployment  |Scaling  |Restrictions  |
|:-------|:------:|:----------:|:-------:|--------------|
|Static Web Apps|Preview|GitHub Actions|Serverless|Deployment, data residency|
|App Service|GA|Lots|Per-Server|IPv6, HTTP/2|
|Azure Storage|GA|CLI Copy|Serverless|CORS, Custom Domains|

## Static Web Apps

Static Web Apps is the new kid on the block.  It's designed as a serverless option for your web site and recommends that you pair it with a dynamic serverless backend based in Azure Functions when you need compute or data storage in the cloud.

**Deployment options**

You check your site into GitHub and then deploy the site using a GitHub Action.  It gets deployed whenever you push to a specific branch of your GitHub repository.  There is a Visual Studio Code extension to make this easier for you.

**Scaling**

It's serverless, so you don't have to worry about scaling.  It will scale out automatically.

**Cost Structure**

**Update 5/12/2021** Static Web Sites has gone GA, and has a free tier.  Check [the pricing page](https://azure.microsoft.com/pricing/details/app-service/static/) for more information.

**Awesome features**

You can generate "[staging slots](https://docs.microsoft.com/azure/static-web-apps/review-publish-pull-requests)", which allows you to preview the new version of the app before it goes live, and redirect a portion of the traffic to the new site to ensure there are no problems.

It supports [custom domains](https://docs.microsoft.com/azure/static-web-apps/custom-domain), which is important because you want your site to be "www.example.com" instead of "www-example-com.azurewebsites.net" (or whatever your resource is called).

It has in-built support for [authentication and authorization](https://docs.microsoft.com/azure/static-web-apps/authentication-authorization) (using AAD, Facebook, Google, MSA, or another OIDC provider).  Something I do - use AAD authentication on a staging slot so that only internal people can review the new version of the site.

**Restrictions**

Right now, it's in preview, so we don't know what restrictions will be in force when it goes GA.  The most serious one (at least for me) is that it only supports GitHub and publishing via GitHub Actions.  I store a lot of my code in Azure DevOps, which isn't supported right now.  It's based on Azure App Service and Azure Functions, so it inherits any platform restrictions that those services have.

Currently, you also don't get to decide on regionality.  Some site owners may want to restrict their web site so that it is only stored in a specific region (like US only) or only accessible to a specific region.  This is not possible with Static Web Apps today.

## Azure App Service

Azure App Service (also known as Azure Web Apps) is the grand-daddy of hosting options available today.  Not only can you [host a static web site](https://docs.microsoft.com/azure/app-service/quickstart-html), but you can also host dynamic sites written in .NET, Node, PHP, Java, Python, or Ruby  You can host multiple sites through the same "hosting plan" (which you can equate to a virtual machine).

**Deployment options**

Take your pick of a lot of options - FTP, cloud sync, git, GitHub, BitBucket, OneDrive, Dropbox, or through CI/CD via GitHub Actions, Azure DevOps, or Jenkins.  There are also plugins for many editors like Visual Studio and Visual Studio Code.  You can quite literally take your pick on how you want deployment to work.

**Scaling**

Azure App Service is server-based.  You can set up rules to automatically scale up or down based on CPU or memory load.  You can also set the size of the server that is providing the scaling unit.  If you think you need to scale up or down, you will have to select an appropriate hosting plan that supports scaling.

**Cost Structure**

You pay on a per-running-server instance basis.  At the low-end, these are free (select the F1-Free hosting option).  The hosting plans have various sizes (CPU, memory, disk) and various features, so selecting the right hosting plan is critical to success.

**Awesome features**

There are so many, including service integration (like traffic manager support).  My favorites (aside from the ease by which you can deploy your site) are:

* [Staging slots](https://docs.microsoft.com/azure/app-service/deploy-staging-slots), which allows you to preview the site and redirect a portion of the traffic to the new site for validation purposes.
* [Private endpoints](https://docs.microsoft.com/azure/app-service/networking/private-endpoint) and [Virtual networks](https://docs.microsoft.com/azure/app-service/web-sites-integrate-with-vnet), which allows you to ensure that the site is only available internal to an enterprise rather than the public Internet.
* [Service isolation](https://docs.microsoft.com/azure/app-service/environment/intro), which allows you to run your site on a completely separate infrastructure.

**Restrictions**

Honestly, not that many.  My main complaint about Azure App Service is that it can be overwhelming for a novice user as it has so many things you can do.  The main restriction is that it does not support HTTP/2 and IPv6 right now, which are not major restrictions for a pure web site.

## Azure Blob Storage

Yes, you can [host a static web site](https://docs.microsoft.com/azure/storage/blobs/storage-blob-static-website-host) in Azure Storage.  The content appears at a subdomain of `web.core.windows.net`. 

**Deployment Options**

There are a number of tools (including the Azure CLI, PowerShell, and [AzCopy](https://docs.microsoft.com/azure/storage/common/storage-use-azcopy-v10)) to copy your site to Azure Storage.  You can also integrate the tools into a CI/CD pipeline, like Azure DevOps Pipelines.  There is also a storage explorer in Visual Studio and Visual Studio Code that allows you to deploy your app.

**Scaling**

It's serverless, so you don't have to worry about scaling.

**Cost Structure**

You pay for blob storage, which is billed based on the amount of data you store.

**Awesome features**

Storage accounts are regional, so you get the data residency by default.  You can set up [redundancy in a secondary region](https://docs.microsoft.com/azure/storage/common/storage-redundancy#redundancy-in-a-secondary-region) (then use traffic manager to route traffic), which provides a level of resiliency to your wen site.  You are no longer affected by the Azure region going offline.  

There is also built-in [snapshots](https://docs.microsoft.com/azure/storage/blobs/snapshots-overview), allowing you to quickly revert to an earlier version of the web site should the need arise.

**Restrictions**

Probably the biggest (for me) is that [CORS (cross-origin resource sharing)](https://docs.microsoft.com/rest/api/storageservices/cross-origin-resource-sharing--cors--support-for-the-azure-storage-services) is not supported in Azure Storage accounts.  CORS is an HTTP feature that enables a web application running under one domain to access resources in another domain.  Web browsers implement a security restriction known as "same origin policy" that prevents the loading of resources from another domain unless it is explicitly allowed.  While this is not a problem in many web sites (since the site is the source of the page), it can be a problem when you are trying to load data from another site.

The other big pain is that there is no support for custom domains.  You want your site to be called `www.example.com`, not `mybigstorageaccount.z22.web.core.windows.net`.  

Fortunately, there are solutions for both of these restrictions.  The one I mostly recommend is the use of [Azure Front Door](https://docs.microsoft.com/azure/frontdoor/) as a gateway to your web app.  However, this has [its own associated costs](https://azure.microsoft.com/pricing/details/frontdoor/), which increases the cost of hosting a web site on Azure Storage.