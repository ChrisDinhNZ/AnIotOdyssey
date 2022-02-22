---
layout: post
title: Processing Azure IoT Hub Events Using Azure Function
date: 2021-05-16
background: '/assets/posts/2021-05-16-processing-azure-iot-hub-events-using-azure-function/post-banner-2021-05-16-processing-azure-iot-hub-events-using-azure-function.jpg'
Tag:
    - IoT Hub
    - Azure Functions
    - Azure
    - IoT
    - Powershell
    - .NET
---

When an IoT Hub receives an event, the event will be stored in an Azure Event Hub. In this article, I will go through the process of creating an Azure Function and use it to pass the events to another Azure IoT device.

## What you will need:

* [An Azure account](https://azure.microsoft.com/en-us/free/)
* [Powershell](https://docs.microsoft.com/en-us/powershell/scripting/overview?view=powershell-7.2)
* [Azure PowerShell](https://docs.microsoft.com/en-us/powershell/azure/what-is-azure-powershell?view=azps-7.1.0)
* [Visual Studio Code](https://code.visualstudio.com/)
* An active Azure IoT Hub

## Creating an Azure Function with VS Code

To develop Azure Function Apps in VS Code, you will first need to install the Azure Functions extension.

![Azure Functions extension for VS Code](/assets/posts/2021-05-16-processing-azure-iot-hub-events-using-azure-function/screenshot_vscode_azure_function_extension.png)

_Azure Functions extension for VS Code_

Press `F1` to open the command palette, select `Azure Functions: Create new project` and follow the instructions. What you want to do is create a function that will get executed when an event occurs, in this case you want to create one with an `EventHubTrigger`.

![Example of creating an Azure Function App using VS Code](/assets/posts/2021-05-16-processing-azure-iot-hub-events-using-azure-function/vscode_new_azure_function_project-1.gif)
_Example of creating an Azure Function App using VS Code_

Once VS Code generated the code for your function, you will want to update the input parameters to the function, more specifically the name and connection string to your hub:

* MyHub
    + An app setting variable containing the event hub name
    + When developing locally, this is stored “local.settings.json”
    + When deployed in production, this is stored in application settings
* MyHubConnection
    + An app setting variable containing the connection string to the event hub
    + When developing locally, this is stored “local.settings.json”
    + When deployed in production, this is stored in application settings

![VS Code generated function](/assets/posts/2021-05-16-processing-azure-iot-hub-events-using-azure-function/screenshot_vscode_azure_function_new.png)
_VS Code generated function_

![Local settings example](/assets/posts/2021-05-16-processing-azure-iot-hub-events-using-azure-function/screenshot_vscode_azure_function_local_settings.png)
_Local settings example_

The event hub name and connection string can be obtained from the IoT Hub built-in endpoints settings in Azure Portal.

At this point, you can run the Azure App in debug mode and watch the events being handled and printed in the console output.

![IoT device agent sending events to IoT Hub and being processed by an Azure function](/assets/posts/2021-05-16-processing-azure-iot-hub-events-using-azure-function/azure_function_handling_events_demo.gif)
_IoT device agent sending events to IoT Hub and being processed by an Azure function_

## Putting the pieces together

Now let’s abstract the underlying infrastructure components away and focus on the things that matters. In this particular scenario, there are 3 things:

* The Home Gateway
* The Home Gateway Handler
* The Home Gateway Client

**The Home Gateway**

The Home Gateway in this case is an Azure IoT device agent that monitors and control some connected devices in the home. In a real world environment you would be deploying this service on any Azure IoT Hub capable devices ([supported platforms](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-device-sdk-platform-support#microsoft-sdks-and-device-platform-support)) but for the sake of demonstration, the Home Gateway will be deployed as a Windows background service.

Basically every 3s the Home Gateway will simulate a doorbell event, it will send a `MultiLinksMessage` event to the IoT hub. The event contains the following data:

* Source (the event originator)
* Destination (the event recipient)
* MessageType (the type of event)
* MessageData (the specific data associated with the event)

```
    namespace HomeGateway
    {
        public class Worker : BackgroundService
        {
            ...

            private async Task SendDoorbellTriggerMessageAsync()
            {
                _logger.LogInformation("Sending doorbell event");

                var doorbellTrigger = new DoorbellTrigger
                {
                    TriggerTime = DateTimeOffset.Now
                };

                var messageData = JsonSerializer.Serialize(doorbellTrigger);

                var multilinksMessage = new MultiLinksMessage
                {
                    Source = "HomeGateway",
                    Destination = "HomeGatewayClient",
                    MessageType = "DoorbellTrigger",
                    MessageData = messageData
                };

                var jsonData = JsonSerializer.Serialize(multilinksMessage);

                var message = new Message(Encoding.ASCII.GetBytes(jsonData));
                message.ContentType = "MultiLinksMessage";

                try
                {
                    await _client.SendEventAsync(message);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Fail to send message");
                }
            }
        }
    }
```

In the code above, every 3s the SendDoorbellTriggerMessageAsync() method will get called. It will generate the data for the doorbell event and send it to the IoT hub.

**The Home Gateway Handler**

The Home Gateway Handler is the Azure Function created earlier. It will process events the IoT hub receives.

```
    namespace com.aniotodyssey
    {
        public static class HomeGatewayHandler
        {
            private const string HubConnectionString = "iot-hub-connection-string";

            [FunctionName("HomeGatewayHandler")]
            public static async Task Run([EventHubTrigger("MyHub", Connection = "MyHubConnection")] EventData[] events, ILogger log)
            {
                var exceptions = new List<Exception>();
                ServiceClient client = ServiceClient.CreateFromConnectionString(HubConnectionString, Microsoft.Azure.Devices.TransportType.Amqp, null);

                foreach (EventData eventData in events)
                {
                    try
                    {
                        await CallDirectMethod(client, eventData);
                    }
                    ...
                }

                ...
            }

            private static async Task CallDirectMethod(ServiceClient client, EventData eventData)
            {
                var method = new CloudToDeviceMethod("ProcessMessage");

                var payload = Encoding.ASCII.GetString(eventData.Body.Array,
                                                    eventData.Body.Offset,
                                                    eventData.Body.Count);

                var multilinksMessage = JsonSerializer.Deserialize<MultiLinksMessage>(payload);
                method.SetPayloadJson(payload);

                var response = await client.InvokeDeviceMethodAsync(multilinksMessage.Destination, method);
            }
        }
    }
```

In the code above, events are extracted from the event hub and passed into `CallDirectMethod()`. CallDirectMethod() will reconstruct the message to determine where it needs to go and then invoke a direct method on the destination device (in this case the Home Gateway Client) and passing in the payload.

**The Home Gateway Client**

Similar to the `Home Gateway`, the `Home Gateway Client` is an Azure IoT device agent that can send commands to the Home Gateway or get notified of Home Gateway events. In a real world environment you would be deploying this service on any Azure IoT Hub capable device ([supported platforms](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-device-sdk-platform-support#microsoft-sdks-and-device-platform-support)) but for the sake of demonstration, the Home Gateway Client will be deployed as a Windows background service.

```
    namespace HomeGatewayClient
    {
        public class Worker : BackgroundService
        {
            ...

            private Task<MethodResponse> ProcessMessage(MethodRequest request, object context)
            {
                var multilinksMessage = JsonSerializer.Deserialize<MultiLinksMessage>(request.DataAsJson);

                if (multilinksMessage.MessageType == "DoorbellTrigger")
                {
                    var doorbellTrigger = JsonSerializer.Deserialize<DoorbellTrigger>(multilinksMessage.MessageData);
                    _logger.LogInformation($"{multilinksMessage.Source} Doorbell triggered at: {doorbellTrigger.TriggerTime}");
                }

                var responseData = Encoding.ASCII.GetBytes("\"response\": \"Message processed\"");
                return Task.FromResult(new MethodResponse(responseData, 200));
            }
        }
    }
```

When the `Home Gateway Client` receives an event, it will invoke `ProcessMessage()` method above. This method will unpack the payload provided by the `Home Gateway Handler` and checks the MessageType to make sure that it can handle the message, in this case if the type is “DoorbellTrigger” it will log the doorbell trigger time to a local file. After that it will respond to the Home Gateway Handler that everything is good.

To verify that things are working end to end you can inspect the logs in live view using the following Powershell commandlet:

```
    Get-Content -Path C:\ProgramData\IoTApps\HomeGateway\activities_report.txt -Tail 10 -Wait
```

```
    Get-Content -Path C:\ProgramData\IoTApps\HomeGatewayClient\activities_report.txt -Tail 10 -Wait
```

![Live Home Gateway activity logs](/assets/posts/2021-05-16-processing-azure-iot-hub-events-using-azure-function/handling_home_gateway_events_live_log_view.gif)
_Live Home Gateway activity logs_

