---
title: "Purge Azure API Management soft-deleted services with ease."
categories:
  - Azure
tags:
  - PowerShell
---

I work a lot with Azure API Management, which means I turn up and down services quite a few times a day.  Azure API Management has an awesome feature that prevents you from accidentally deleting a service, called soft-delete.  Instead of immediately deleting the service, it marks it as soft deleted and purges it later on.  Unfortunately, that means that you can't immediately reuse that service name.  In production, this is a great thing to have.  In development, it turns into a pain. That's because there is no Azure CLI, PowerShell, or Portal way to purge the soft-deleted service.

There is a REST API for it, though.  That means you can write nice PowerShell functions to do the work for you, wrap them in a PowerShell module, and use them instead.  The main thing you have to get past is "how do I invoke a REST API on Azure?"

## Get an access token

The first task is to get an access token:

```powershell
$azContext = Get-AzContext
$azProfile = [Microsoft.Azure.Commands.Common.Authentication.Abstractions.AzureRmProfileProvider]::Instance.Profile
$profileClient = New-Object -TypeName Microsoft.Azure.Commands.ResourceManager.Common.RMProfileClient -ArgumentList ($azProfile)
$token = $profileClient.AcquireAccessToken($azContext.Subscription.TenantId)
```

This is a common enough method that I want to re-use it.  I've got a file called `AzApiMgmtExtras.psm1` in my `~/Documents/WindowsPowerShell/Scripts` directory.  It contains (thus far) the following:

```powershell
<#
 .Synopsis
 Retrieves an access token for using the Azure REST APIs with the current subscription.
#>
function Get-AzAccessToken
{
  $azContext = Get-AzContext
  $azProfile = [Microsoft.Azure.Commands.Common.Authentication.Abstractions.AzureRmProfileProvider]::Instance.Profile
  $profileClient = New-Object -TypeName Microsoft.Azure.Commands.ResourceManager.Common.RMProfileClient -ArgumentList ($azProfile)
  $token = $profileClient.AcquireAccessToken($azContext.Subscription.TenantId)
  return $token
}

Export-ModuleMember -Function Get-AzAccessToken
```

You can import this module with `Import-Module <path-to-psm1-file>` and the function will now be available:

```powershell
PS1> Get-AzAccessToken

AccessToken        : eyJ0{A really long string which I'm not going to repeat}
UserId             : photoadrian@outlook.com
TenantId           : {removed but expect the GUID of your AAD tenant}
LoginType          : User
HomeAccountId      : {pair-of-guids separated by a period}
ExtendedProperties : {}
ExpiresOn          : {some date/time about an hour from now}
```

## Add a List command

Let's look at the first REST API command I want to produce - listing the deleted services.  According to [the documentation](https://learn.microsoft.com/rest/api/apimanagement/current-preview/deleted-services/list-by-subscription?tabs=HTTP), I need to do a GET of a big long URL, but with authentication.  Something like the following:

```powershell
function List-AzApiManagementDeletedServices {
    $azContext = Get-AzContext
    $token = Get-AzAccessToken
    $authHeader = @{
        'Content-Type'='application/json'
        'Authorization'='Bearer ' + $token.AccessToken
    }
    $baseUri = "https://management.azure.com/subscriptions/$($azContext.Subscription)/providers/Microsoft.ApiManagement"
    $apiVersion = '?api-version=2022-04-01-preview'

    $restUri = "${baseUri}/deletedservices${apiVersion}"
    $result = Invoke-RestMethod -Uri $restUri -Method GET -Header $authHeader
    return $result.value
}

Export-ModuleMember -Function List-AzApiManagementDeletedServices
```

Yes, I know List is not an approved verb - these are my PowerShell scripts, so I can name them whatever I want!  So, what's it doing?  First, it gets the information it needs to build the REST API call.  The `AzContext` is needed because it contains the subscription ID and that is part of the URL to execute.  The access token is required for authorization.

Next, construct the URI.  This comes directly from the documentation.  Just replace the swappable pieces in the URI with the bits from the context or parameters (which we'll see next).  I use a two-phase construction because the base URI and API version strings are the same for all operations, so I can copy-paste the code.

Then I execute the `Invoke-RestMethod` to get the result. If your REST API call needs a body, that can be included as well.  Finally, I return the result.  In this case, the result is always returned in the "value" property, so I just return the value property.

What does the output look like?

```powershell
PS1> List-AzApiManagementDeletedServices

id         : /subscriptions/removed/providers/Microsoft.ApiManagement/locations/southcentralus/deleteds 
             ervices/apim-fvtyoremoved
name       : apim-fvtyoremoved
type       : Microsoft.ApiManagement/deletedservices
location   : South Central US
properties : @{serviceId=/subscriptions/removed/resourceGroups/rg-test1/providers/Microsoft 
             .ApiManagement/service/apim-fvtyoremoved; scheduledPurgeDate=1/22/2023 6:43:33 PM; deletionDate=1/20/2023 6:44:06 PM}  
```

### Add a Purge command

What about the equivalent purge command?  After all, I want this service totally gone!  This is just another PowerShell function, this time following the purge documentation.  For this operation, I need a location and name.  Even though the location in the results is "South Central US", I need the short version "southcentralus", so I have to do that adjustment (left for an exercise for you).  Something like the following:

```powershell
function Purge-AzApiManagementDeletedService {
    param(
        [string] $name,
        [string] $location
    )
    $azContext = Get-AzContext
    $token = Get-AzAccessToken
    $authHeader = @{
        'Content-Type'='application/json'
        'Authorization'='Bearer ' + $token.AccessToken
    }
    $baseUri = "https://management.azure.com/subscriptions/$($azContext.Subscription)/providers/Microsoft.ApiManagement"
    $apiVersion = '?api-version=2022-04-01-preview'

    $restUri = "${baseUri}/locations/${location}/deletedservices/${name}${apiVersion}"
    Invoke-RestMethod -Uri $restUri -Method DELETE -Header $authHeader
}

Export-ModuleMember -Function Get-AzApiManagementDeletedService
```

Everything from the `Get-AzContext` to the `$apiVersion` assignment is the same as the previous example.  Now, I can use the command:

```powershell
PS1> Purge-AzApiManagementDeletedService -Location southcentralus -Name apim-fvtyoremoved
```

The API Management service will be purged immediately, allowing me to progress with re-creating it from scratch. (This is super useful right now as I am developing Azure Developer CLI scripts).

## Wrap up

A final note about development.  If you import the module, then change it, make sure you re-import it with the -Force flag:

```powershell
PS1> Import-Module .\Scripts\AzApiMgmtExtra.psm1 -Force
```

This will overwrite the functions that you have previously defined.

Azure doesn't produce PowerShell cmdlets for preview features generally.  However, that doesn't mean you can't use or configure the features.  The process of producing additional PowerShell functions is easy and allows you to do things that don't have a command line equivalent!
