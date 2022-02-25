---
layout: post
title: Deploying An Azure Function using VS Code
date: 2021-05-28
background: '/assets/posts/2021-05-28-deploying-an-azure-function-using-vs-code/post-banner-2021-05-28-deploying-an-azure-function-using-vs-code.jpg'
Tag:
    - Azure Functions
    - Azure
    - IoT
    - Powershell
    - .NET
---

In the previous post the focus was about binding an Azure Function to an Azure Iot Hub so that it can process the events coming into the hub. However, the function was running within a local development environment. In this post the focus is to deploy it to a live environment, using `VS Code's Azure Functions extension`.

## What we will need:

* [An Azure account](https://azure.microsoft.com/en-us/free/)
* [Powershell](https://docs.microsoft.com/en-us/powershell/scripting/overview?view=powershell-7.2)
* [Azure PowerShell](https://docs.microsoft.com/en-us/powershell/azure/what-is-azure-powershell?view=azps-7.1.0)
* [Visual Studio Code](https://code.visualstudio.com/)

## Creating an Azure Function App resource

Let’s go into the Powershell console and create an Azure Function App resource, using the `New-AzFunctionApp` commandlet. For that we will need the following parameters:

* Function App Location
* Function App Name
* Resource Group Name
* Storage Account Name

For the Function App location and resource group name we can use the same as we did for the IoT Hub. However the storage account have not been created yet, so let’s go ahead and do that first. Once the storage account has been created, we then create the Azure Function App resource.

```
    PS D:WorkspaceIoTHub> $sn = "storageaniotodyssey"
    PS D:WorkspaceIoTHub> $rg = "rg-sea-aniotodyssey"
    PS D:WorkspaceIoTHub> $loc = "southeastasia"
    PS D:WorkspaceIoTHub> $sku = "Standard_LRS"
    PS D:WorkspaceIoTHub> New-AzStorageAccount -ResourceGroupName $rg -AccountName $sn -Location $loc -SkuName $sku
    StorageAccountName  ResourceGroupName   PrimaryLocation SkuName      Kind      AccessTier CreationTime         ProvisioningState EnableHttpsTrafficOnly LargeFileShares
    ------------------  -----------------   --------------- -------      ----      ---------- ------------         ----------------- ---------------------- ---------------
    storageaniotodyssey rg-sea-aniotodyssey southeastasia   Standard_LRS StorageV2 Hot        5/05/2021 9:47:58 am Succeeded         True
    PS D:WorkspaceIoTHub> Import-Module Az.Functions
    PS D:WorkspaceIoTHub> $fan = "fa-sea-aniotodyssey"
    PS D:WorkspaceIoTHub> New-AzFunctionApp -Name $fan -ResourceGroupName $rg -Location $loc -StorageAccount $sn -Runtime "Dotnet" -FunctionsVersion 3
    VERBOSE: 'DotNet' runtime version is specified by FunctionsVersion. The value of the -RuntimeVersion will be set to '3'.
    VERBOSE: OSType for Dotnet is 'Windows'.
    WARNING: Unable to create the Application Insights for the function app. Creation of Application Insights will help we monitor and diagnose our function apps in the Azure Portal.
    Use the 'New-AzApplicationInsights' cmdlet or the Azure Portal to create a new Application Insights project. After that, use the 'Update-AzFunctionApp' cmdlet to update Application Insights for our function app.
    Name                Status  OSType Runtime Location      AppServicePlan ResourceGroupName   SubscriptionId
    ----                ------  ------ ------- --------      -------------- -----------------   --------------
    fa-sea-aniotodyssey Running                southeastasia                rg-sea-aniotodyssey xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    PS D:WorkspaceIoTHub>
```

Now if we check the resources in our Azure Portal, we should be able to see the function app `fa-sea-aniotodyssey` that was created.

![Example Azure Portal resource list](/assets/posts/2021-05-28-deploying-an-azure-function-using-vs-code/screenshot_current_azure_resources.png)
_Example Azure Portal resource list_

## Deploying the Function App to Azure

Now that the Function App resource have been created in Azure, we can deploy the local Function App onto it. The process will be a manual but simple one, using Visual Studio Code.

I am already signed in to my Azure account where the Function App resource was created. The process is as easy as using the Visual Studio Code command palette, select the `Azure Functions: Deploy to Function App…` and select the resource to deploy to.

![Deploying a Function App using VS Code](/assets/posts/2021-05-28-deploying-an-azure-function-using-vs-code/azure_function_app_manual_deploy.gif)
_Deploying a Function App using VS Code_

That’s it, we now have a live Function App which is handling events from Azure IoT devices.
