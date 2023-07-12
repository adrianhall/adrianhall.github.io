---
title: "Deleting Azure resources the right way."
categories:
  - Azure
tags:
  - PowerShell
---

If you are an Azure developer, you likely spin up an application, do some work, then shut it down again.  However, shutting down resources and deleting them has an order to them.  If your service is network isolated (in a virtual network), then you can't delete the application until the private endpoints are shut down.  Budgets don't get deleted with the resource group.  Diagnostic settings can't be deleted if the resource no longer exists.  There is an order you should do things:

1. Private endpoints
2. Diagnostic settings
3. Budgets
4. Resources that need to be purged (e.g. Key Vault and API Management)
5. Resource groups

The resource groups are last and will delete the rest of the resources for you. I use the [Azure Developer CLI] for deployments, but cleaning up can be hard since the azd tool does not understand this ordering problem.

So, how do I do it?  With a little bit of love from PowerShell.  I have a script that I can run (imaginatively called `cleanup.ps1`).  To start, I provide a list of resource groups that I want to clean up:

```powershell
<#
.SYNOPSIS
  Cleans up the resources in a set of resource groups.
#>

Param(
  [Parameter(Mandatory)][string]$Prefix
  [switch]$AsJob = $false
)

$resourceGroups = [System.Collection.ArrayList]@()
Get-AzResourceGroup | Where-Object { $_.ResourceGroupName -like $Prefix } | Foreach-Object {
  $resourceGroups.Add($_.ResourceGroupName) | Out-Null
}
``````

## Deleting private endpoints, diagnostic settings, and budgets

Now that I have the list of resource groups, I can go through my ordering and start deleting things.  Here is the code for private endpoints:

```powershell
function Remove-ConsumptionBudgetForResourceGroup($resourceGroupName) {
    Get-AzConsumptionBudget -ResourceGroupName $resourceGroupName
    | Foreach-Object {
        "`tRemoving $resourceGroupName::$($_.Name)" | Write-Output
        Remove-AzConsumptionBudget -Name $_.Name -ResourceGroupName $_.ResourceGroupName
    }
}

function Remove-DiagnosticSettingsForResourceGroup($resourceGroupName) {
    Get-AzResource -ResourceGroupName $resourceGroupName
    | Foreach-Object {
        $resourceName = $_.Name
        $resourceId = $_.ResourceId
        Get-AzDiagnosticSetting -ResourceId $resourceId -ErrorAction SilentlyContinue | Foreach-Object {
            "`tRemoving $resourceGroupName::$resourceName::$($_.Name)" | Write-Output
            Remove-AzDiagnosticSetting -ResourceId $resourceId -Name $_.Name 
        }
    }
}

function Remove-PrivateEndpointsForResourceGroup($resourceGroupName) {
    Get-AzPrivateEndpoint -ResourceGroupName $resourceGroupName
    | Foreach-Object {
        "`tRemoving $resourceGroupName::$($_.Name)" | Write-Output
        Remove-AzPrivateEndpoint -Name $_.Name -ResourceGroupName $_.ResourceGroupName -Force
    }
}

"> Private Endpoints:" | Write-Output
foreach ($resourceGroupName in $resourceGroups) {
    Remove-PrivateEndpointsForResourceGroup -ResourceGroupName $resourceGroupName
}

"> Budgets:" | Write-Output
foreach ($resourceGroupName in $resourceGroups) {
    Remove-ConsumptionBudgetForResourceGroup -ResourceGroupName $resourceGroupName
}

"> Diagnostic Settings:" | Write-Output
foreach ($resourceGroupName in $resourceGroups) {
    Remove-DiagnosticSettingsForResourceGroup -ResourceGroupName $resourceGroupName
}
```

This pattern is repeated quite often.  I get a list of the private endpoints within a resource group, then remove each one.  You can do some more work to send the output to the verbose channel (together with all the other work for making this a verbose module), but I find I always want the output, so I just write it out.

## Removing the resource groups

Once all the dependent services are done, I can remove the resource groups:

```powershell
function Remove-ResourceGroupFromAzure($resourceGroupName, $asJob) {
    if (Test-ResourceGroupExists -ResourceGroupName $resourceGroupName) {
        "`tRemoving $resourceGroupName" | Write-Output
        if ($asJob) {
            Remove-AzResourceGroup -Name $resourceGroupName -Force -AsJob
        } else {
            Remove-AzResourceGroup -Name $resourceGroupName -Force
        }
    }
}

"`nRemoving resource groups..." | Write-Output
foreach ($resourceGroupName in $resourceGroups) {
    Remove-ResourceGroupFromAzure -ResourceGroupName $resourceGroupName -AsJob:$AsJob
}
```

Be careful to understand your dependencies though.  For example, let's say you have an App Service that is VNET integrated, and the App Service is in a different resource group to the virtual network.  You cannot delete the virtual network until the App Service is deleted.  Since they are in different resource groups, you will likely bump into a timing issue that is not easily resolved.  Sometimes, it will work, and other times it won't work. Fix this by ensuring you delete the resource group with the App Service first.

## Finally

With the [Azure Developer CLI], I can easily deploy a complete solution with diagnostics, a hub-spoke virtual network setup, and complete network isolation.  Now, I can also tear down the solution easily without worrying about the lingering resources issues.

<!-- Links -->
[Azure Developer CLI]: https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/overview