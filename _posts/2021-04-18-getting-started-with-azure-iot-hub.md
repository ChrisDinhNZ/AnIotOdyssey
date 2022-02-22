---
layout: post
title: Getting Started With Azure IoT Hub
date: 2021-04-18
background: '/assets/posts/2021-04-18-getting-started-with-azure-iot-hub/post-banner-2021-04-18-getting-started-with-azure-iot-hub.jpg'
Tag:
  - IoT Hub
  - Azure
  - IoT
  - Powershell
---

[Azure IoT Hub](https://azure.microsoft.com/en-us/services/iot-hub/) is a managed cloud service which provides bi-directional communication between the cloud and IoT devices. It is a platform as a service for building IoT solutions. Being an azure offering, it has security and scalability built-in as well as making it easy to integrate with other Azure services.

This blog post is an introduction to Azure IoT Hub and how to get one up and running.

## What you will need:

* [An Azure account](https://azure.microsoft.com/en-us/free/)
* [Powershell](https://docs.microsoft.com/en-us/powershell/scripting/overview?view=powershell-7.2)
* [Azure PowerShell](https://docs.microsoft.com/en-us/powershell/azure/what-is-azure-powershell?view=azps-7.1.0)

## Creating an IoT Hub

You will be using Azure Powershell to create an Azure IoT Hub. Below are some Powershell commandlets that you can use to check that you are signed in and using the correct Azure subscription.

```
    PS D:\Workspace\IoTHub> Get-AzContext
    PS D:\Workspace\IoTHub> Connect-AzAccount
    WARNING: To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code ABCDEFGHIJ to authenticate.
    PS D:\Workspace\IoTHub> Get-AzContext
    Name
    ----
    Free Trial (abc12345-1234-abcd-1234-abc…
    PS D:\Workspace\IoTHub>
```

Note that `Get-AzContext` returns nothing if you are not signed in (i.e. no context). But after signing in, `Get-AzContext` should return your subscription details.

Once you are logged in, you will need to create a resource group (basically a logical container). Every resource we create in Azure needs to be associated with a resource group.

```
    PS D:\Workspace\IoTHub> New-AzResourceGroup -Name 'rg-sea-aniotodyssey' -Location 'southeastasia' -Verbose -Force -ErrorAction Stop
    VERBOSE: Performing the operation "Replacing resource group ..." on target "rg-sea-aniotodyssey".
    VERBOSE: 8:50:49 pm - Created resource group 'rg-sea-aniotodyssey' in location 'southeastasia'
    ResourceGroupName : rg-sea-aniotodyssey
    Location          : southeastasia
    ProvisioningState : Succeeded
    Tags              :
    ResourceId        : /subscriptions/abc12345-1234-abcd-1234-abcdef123456/resourceGroups/rg-sea-aniotodyssey
    PS D:\Workspace\IoTHub>
```

You can now proceed to creating an Iot Hub. One thing I want to point out is the `SkuName` parameter. Depending on your needs, I am inclined to use the `F1` (which is the free tier) for prototyping purposes and basic usage. This should help avoiding unexpected bills.

```
    PS D:\Workspace\IoTHub> New-AzIotHub -ResourceGroupName 'rg-sea-aniotodyssey' -Name 'ih-sea-aniotodyssey' -SkuName 'F1' -Units 1 -Location 'southeastasia'
    Id             : /subscriptions/abc12345-1234-abcd-1234-abcdef123456/resourceGroups/rg-sea-aniotodyssey/providers/Microsoft.Devices/IotHubs/ih-sea-aniotodyssey
    Name           : ih-sea-aniotodyssey
    Type           : Microsoft.Devices/IotHubs
    Location       : southeastasia
    Tags           : {}
    Subscriptionid : abc12345-1234-abcd-1234-abcdef123456
    Resourcegroup  : rg-sea-aniotodyssey
    Properties     : Microsoft.Azure.Commands.Management.IotHub.Models.PSIotHubProperties
    Sku            : Microsoft.Azure.Commands.Management.IotHub.Models.PSIotHubSkuInfo
    PS D:\Workspace\IoTHub>
```

There you have it, you have just created an Azure IoT Hub hosted in a data center in South East Asia.

![New IoT Hub created](/assets/posts/2021-04-18-getting-started-with-azure-iot-hub/azure-iot-hub-demo.png)
_New IoT Hub created_
