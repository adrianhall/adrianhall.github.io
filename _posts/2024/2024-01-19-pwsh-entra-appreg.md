---
title: "Create an App Registration for RBAC with PowerShell and Microsoft Graph."
categories:
  - Cloud
tags:
  - powershell
  - msgraph
  - azure_active_directory
  - microsoft_entra
---

I'm currently working on some automation within Azure to deploy a hub-spoke web application.  This web application authenticates with Entra ID using an App Registration and Role-Based Access Controls (RBAS) using App Roles.  So, I need to create an app registration, enterprise app, and create the app roles.

Seems easy, right?

## Get an access token

Since this is part of a deployment, I'm going to assume the user has already connected to Azure using `Connect-AzAccount` and selected a subscription.  The command `Get-AzContext` can be used to see both the signed-in account and the selected subscription.

However, I want to use the Microsoft Graph cmdlets for this.  I'm assuming that the Microsoft Graph cmdlets are better since they are created from the REST API.  How do I do that?

{% highlight powershell %}
$token = (Get-AzAccessToken -ResourceTypeName MSGraph -ErrorAction Stop).token
if ((Get-Help Connect-MgGraph -Parameter accesstoken).type.name -eq "securestring") {
  $token = ConvertTo-SecureString $token -AsPlainText -Force
}
$null = Connect-MgGraph -AccessToken $token -ErrorAction Stop
{% endhighlight %}

There is a little magic here.  Earlier versions of the `Connect-MgGraph` took a plain text token, but newer versions require this to be a secure string.  We have to convert, but only if necessary.  It's annoying.

## Create the App Roles object

Next, I need to create a list of App Roles.  However, I want to only create the app roles that are new.  If I am running the script for a second time (maybe to add new roles that have not yet been defined), then I need to merge the existing app roles with the new roles.  The following function creates the array of objects that I need:

{% highlight powershell %}
<#
.SYNOPSIS
    Gets the list of application roles that should be created.
.PARAMETER RoleNames
    The list of role names that are needed by the application.
.PARAMETER ExistingRoles
    The list of existing roles that are already defined in the app registration.
#>
function Get-AppRoles {
    param(
        [Parameter(Mandatory=$true)]
        [string[]] $RoleNames,

        [Parameter(Mandatory=$false)]
        [Microsoft.Graph.PowerShell.Models.IMicrosoftGraphAppRole[]] $ExistingRoles = [Microsoft.Graph.PowerShell.Models.IMicrosoftGraphAppRole[]]@()
    )

    Write-Verbose "Getting App Roles for $($RoleNames | ConvertTo-Json -Compress)"
    $result = @()
    foreach ($roleName in $RoleNames) {
        $role = $ExistingRoles | Where-Object { $_.Value -eq $roleName }
        if ($null -eq $role) {
            Write-Verbose "Creating App Role for $roleName"
            $newRole = @{
                'AllowedMemberTypes' = @( 'User' )
                'Description' = "The $roleName role for the Contoso Fiber CAMS application."
                'DisplayName' = $roleName
                'Id' = [guid]::NewGuid()
                'IsEnabled' = $true
                'Value' = $roleName
            }
            $result += $newRole
        } else {
            Write-Verbose "Using Existing App Role for $roleName"
            $result += $role
        }
    }
    Write-Verbose "New App Roles: $($result | ConvertTo-Json -Compress)"
    return $result
}
{% endhighlight %}

I can create the necessary role names using an array - e.g.

{% highlight powershell %}
$MyRoleNames = "Role1", "Role2", "Role3"
{% endhighlight %}

## Create or update the app registration

This is the big long function that I use to create or update the app registration.  It's an all-in-one "create an app registration and the app roles and the associated enterprise application".

{% highlight powershell %}
function New-EntraAppRegistration {
    param(
        [Parameter(Mandatory=$true)] [string] $ApplicationName,
        [Parameter(Mandatory=$true)] [string[]] $AppRoles,
        [Parameter(Mandatory=$true)] [string] $BaseUrl,
        [Parameter(Mandatory=$false)] [string] $SignInCallbackPath = "/signin-oidc",
        [Parameter(Mandatory=$false)] [string] $SignOutCallbackPath = "/signout-callback-oidc"
    )

    # Get the app registration if it exists.
    $app = Get-MgApplication -Filter "displayName eq '$($ApplicationName)'" -Top 1 -ErrorAction SilentlyContinue
    if ($null -eq $app) {
        $web = @{
            'HomePageUrl' = $BaseUrl
            'LogoutUrl' = "$($BaseUrl)$($SignOutCallbackPath)"
            'RedirectUris' = @( $BaseUrl, "$($BaseUrl)$($SignInCallbackPath)" )
            'ImplicitGrantSettings' = @{
                'EnableAccessTokenIssuance' = $false
                'EnableIdTokenIssuance' = $true
            }
        }
        $newAppRoleDefinitions = Get-AppRoles -RoleNames $AppRoles

        $newAppRegistration = New-MgApplication -DisplayName $ApplicationName `
            -SignInAudience 'AzureADMyOrg' `
            -Web $web `
            -AppRoles $newAppRoleDefinitions `
            -ErrorAction Stop
        New-MgServicePrincipal -AppId $newAppRegistration.AppId -ErrorAction Stop
    } else {
        $updatedAppRoleDefinitions = if ($app.AppRoles.Length -gt 0) {
            Get-AppRoles -RoleNames $AppRoles -ExistingRoles $app.AppRoles
        } else {
            Get-AppRoles -RoleNames $AppRoles
        }
        Update-MgApplication -ApplicationId $app.Id -AppRoles $updatedAppRoleDefinitions -ErrorAction Stop
    }

    $updatedApp = Get-MgApplication -Filter "displayName eq '$($ApplicationName)'" -Top 1
    return $updatedApp
}
{% endhighlight %}

The first thing I do in this function is to try and get the existing app registration.  I do this by matching the display name.  Since display names can be duplicated (something you should not do), you may want to use some other filter here.

If the app does not exist, I create the app registration with app roles (using the `Get-AppRoles` function I developed earlier) to create the app registration, and then I create the enterprise application.  An enterprise application is just a service principal for the app registration, so I use `New-MgServicePrincipal` for this.

If the app does exist, then I may need to update the app roles.  I'll use the `Get-AppRoles` function again, then call `Update-MgApplication` using the updated list.  This will contain any extras needed.  It's not a big deal if this is called multiple times since the interface is idempotent.

Finally, to ensure I have the latest information no matter which route I took, I go and grab the app registration information again and return it.

## Hooking it all together

My aim here is to configure an ASP.NET Core web application.  That requires a block to be placed in the `appsettings.json` file in a specific form.  So, how do I do that?

{% highlight powershell %}
# Create or update the app registration.
$appRegistration = New-EntraAppRegistration -ApplicationName $ApplicationName `
  -BaseUrl $BaseUrl `
  -AppRoles $AppRoles `
  -SignInCallbackPath $SignInCallbackPath `
  -SignOutCallbackPath $SignOutCallbackPath

$settings = @{
    'AzureAd' = @{
        'Instance' = $context.Environment.ActiveDirectoryAuthority
        'Domain' = $tenant.DefaultDomain
        'TenantId' = $tenant.TenantId
        'ClientId' = $appRegistration.AppId
        'CallbackPath' = $SignInCallbackPath
        'SignedOutCallbackPath' = $SignOutCallbackPath
    }
}

Write-Host "`nPlace the following in your appsettings.json or secrets.json file:`n"
$settings | ConvertTo-Json | Write-Host
{% endhighlight %}

Obviously, I need to provide the information the `New-EntraAppRegistration` function requires.  I do this with script parameters.  However, the result is that I can cut and paste the displayed settings into my `appsettings.json` (or, more normally, my `secrets.json` file).

Once complete, I can also go onto the Azure Portal and add users to my app roles:

* Go to the **Entra ID** section of the Azure Portal.
* Select **App registrations** from the side-bar.
* Select the app registration you just created.
* In the Essentials section, click on the **Managed application in local directory**.
* Finally, click on **Assign users and groups**.

From here, you can press **Add user/group** to initiate the flow to add a user to a role.

## Wrap up

I hope this helps anyone who is trying to automate Microsoft Graph operations from the Azure side of things.  It's a little fiddly to find out what is needed, but the information can be put together.

Until next time, happy hacking!
