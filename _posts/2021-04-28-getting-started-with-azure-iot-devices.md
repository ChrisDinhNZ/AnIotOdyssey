---
layout: post
title: Getting Started With Azure IoT Devices
date: 2021-04-28
background: '/assets/posts/2021-04-28-getting-started-with-azure-iot-devices/post-banner-2021-04-28-getting-started-with-azure-iot-devices.jpg'
Tag:
  - IoT Hub
  - Azure
  - IoT
  - Powershell
  - .NET
---

Unlike desktop computers and mobile phones, generally speaking IoT devices are constraint devices with specific purposes. They either act as sensors, capturing data and events, or they act as actuators performing some action based on some digital inputs.

This blog post will expand on how Azure IoT Hub works by focussing on creating an Azure IoT device and connecting it to an Azure IoT Hub.

## What we will need:

* [An Azure account](https://azure.microsoft.com/en-us/free/)
* [Powershell](https://docs.microsoft.com/en-us/powershell/scripting/overview?view=powershell-7.2)
* [Azure PowerShell](https://docs.microsoft.com/en-us/powershell/azure/what-is-azure-powershell?view=azps-7.1.0)
* [.NET 5 SDK Installed](https://dotnet.microsoft.com/download/dotnet/5.0)
* An active Azure IoT Hub

## Creating an IoT device

In this section we will be using Azure Powershell to register an IoT device with the Azure IoT Hub. You can easily add new devices using the `Add-AzIotHubDevice` commandlet.

```
    PS D:\Workspace\IoTHub> Add-AzIotHubDevice -ResourceGroupName "rg-sea-aniotodyssey" -IotHubName "ih-sea-aniotodyssey" -DeviceId "HomeGateway" -AuthMethod "shared_private_key"

    DeviceId                   : HomeGateway
    GenerationId               : 123456789101112131
    ETag                       : "abcdefghijkl"
    LastActivityTime           : 1/01/0001 12:00:00 am
    ConnectionState            : Disconnected
    ConnectionStateUpdatedTime : 1/01/0001 12:00:00 am
    Status                     : Enabled
    StatusReason               :
    StatusUpdatedTime          : 1/01/0001 12:00:00 am
    CloudToDeviceMessageCount  : 0
    Authentication             : Sas
    EdgeEnabled                : False
    DeviceScope                :
```

Now if we have a look at the IoT Hub in Azure portal, and under Explorers > IoT Devices we should see the new device we have just created.

![IoT Hub devices view](/assets/posts/2021-04-28-getting-started-with-azure-iot-devices/azure_iot_hub_devices_view.png)
_IoT Hub devices view_

## Creating an IoT device agent

In the previous section we registered an IoT device with the IoT Hub and named it “HomeGateway”. Next we will create a .NET application to manage the device activities and communication with the hub. For now we will just be simulating the device using a Windows service, but in a real deployment scenario we would deploy this service on the actual device.

Let’s create a Worker Service using the dotnet cli template and then run it just to make sure everything is as expected.

![Creating a new .NET Worker Service using dotnet cli](/assets/posts/2021-04-28-getting-started-with-azure-iot-devices/new_worker_project_demo.gif)
_Creating a new .NET Worker Service using dotnet cli_

As a Windows service, it will be running in the background. There is no console or GUI so there’s a couple of things we will need to do:

* Log activities to a file
* Register it with the Service registry and set it to run when the machine starts

**Setup logging to local file**

Install the Serilog dependencies by installing any necessary Nuget packages:

* Serilog.AspNetCore
* Serilog.Sink.File

![Install Serilog dependencies](/assets/posts/2021-04-28-getting-started-with-azure-iot-devices/install_serilog_dependencies.gif)
_Install Serilog dependencies_

and then reconfigure the logging in the Main() method.

```
    using System;
    using Microsoft.Extensions.DependencyInjection;
    using Microsoft.Extensions.Hosting;
    using Serilog;
    using Serilog.Events;

    namespace HomeGateway
    {
        public class Program
        {
            public static void Main(string[] args)
            {
                Log.Logger = new LoggerConfiguration()
                    .MinimumLevel.Debug()
                    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
                    .Enrich.FromLogContext()
                    .WriteTo.File(@"C:\ProgramData\IoTApps\HomeGateway\activities_report.txt")
                    .CreateLogger();

                try
                {
                    Log.Information("Starting up the service");
                    CreateHostBuilder(args).Build().Run();
                    return;
                }
                catch (Exception ex)
                {
                    Log.Fatal(ex, "There was a problem starting the service");
                    return;
                }
                finally
                {
                    Log.CloseAndFlush();
                }
            }

            public static IHostBuilder CreateHostBuilder(string[] args) =>
                Host.CreateDefaultBuilder(args)
                    .ConfigureServices((hostContext, services) =>
                    {
                    services.AddHostedService<Worker>();
                    })
                    .UseSerilog();
        }
    }
```

At this point, if we run the service, then instead of logging to the console, all loggings will be written to `C:\ProgramData\IoTApps\HomeGateway\activities_report.txt`.

**Deploy and run as a Windows Service**

Install the Windows Service dependency by installing any necessary Nuget packages:

* Microsoft.Extensions.Hosting.WindowsServices

Update IHostBuilder to use Windows service:

```
    public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .UseWindowsService()
        .ConfigureServices((hostContext, services) =>
            {
                services.AddHostedService<Worker>();
            })
        .UseSerilog();
```

Next we will deploy the service to a specified folder and register it as a Windows service. Open a Powershell terminal as an administrator and execute the following commands:

```
    PS D:\Workspace\IoTHub\HomeGateway> dotnet publish -o "C:\Program Files (x86)\IoTApps\HomeGateway"
    Microsoft (R) Build Engine version 16.9.0+57a23d249 for .NET
    Copyright (C) Microsoft Corporation. All rights reserved.

    Determining projects to restore...
    All projects are up-to-date for restore.
    HomeGateway -> C:\Users\Chris\Source\Repos\IoTApps\HomeGateway\bin\Debug\net5.0\HomeGateway.dll
    HomeGateway -> C:\Program Files (x86)\IoTApps\HomeGateway\
    PS D:\Workspace\IoTHub\HomeGateway> sc.exe create HomeGateway binpath= "C:\Program Files (x86)\IoTApps\HomeGateway\HomeGateway.exe"
    [SC] CreateService SUCCESS
    PS D:\Workspace\IoTHub\HomeGateway> sc.exe start HomeGateway
    SERVICE_NAME: HomeGateway
            TYPE               : 10  WIN32_OWN_PROCESS
            STATE              : 2  START_PENDING
                                    (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
            WIN32_EXIT_CODE    : 0  (0x0)
            SERVICE_EXIT_CODE  : 0  (0x0)
            CHECKPOINT         : 0x0
            WAIT_HINT          : 0x7d0
            PID                : 17864
            FLAGS              :
```

If we open Windows Task Manager, under the Services tab we should see HomeGateway running as a Windows service.

![HomeGateway running as a service](/assets/posts/2021-04-28-getting-started-with-azure-iot-devices/image_2021-04-27_191209.png)
_HomeGateway running as a service_

## Connecting to the IoT Hub

Install the Azure IoT dependencies by installing any necessary Nuget packages:

* Microsoft.Azure.Devices.Client

Now update the Worker class so that it connects to the IoT Hub when the HomeGateway starts and disconnect from the IoT Hub when the HomeGateway stops.

```
    public class Worker : BackgroundService
    {
        ...
        private const string DeviceConnectionString = "our-device-connection-string";
        private DeviceClient _client = null;
        private bool _clientIsConnected;
        ...
        // Placeholder for anything we want to setup
        public override Task StartAsync(CancellationToken cancellationToken)
        {
            try
            {
                _logger.LogInformation("Service is connected at: {time}", DateTimeOffset.Now);
                _client = DeviceClient.CreateFromConnectionString(DeviceConnectionString, Microsoft.Azure.Devices.Client.TransportType.Amqp);
                _client.OpenAsync();
                _clientIsConnected = true;
            }
            catch (Exception ex)
            {
                _logger.LogCritical(ex, "Failed to connect to service at: {time}", DateTimeOffset.Now);
                _clientIsConnected = false;
            }

            return base.StartAsync(cancellationToken);
        }

        // Placeholder for anything we want to teardown
        public override Task StopAsync(CancellationToken cancellationToken)
        {
            if (_client != null)
            {
                _logger.LogInformation("Disconnect service at: {time}", DateTimeOffset.Now);
                _client.CloseAsync();
            }

            _logger.LogInformation("Stop service at: {time}", DateTimeOffset.Now);

            return base.StopAsync(cancellationToken);
        }
        ...
    }
```

Now we should see the device connection state reflected on the Azure IoT Hub when the HomeGateway starts and stops using Get-AzIotHubDevice commandlet.

![Device connecting and disconnecting to IoT Hub](/assets/posts/2021-04-28-getting-started-with-azure-iot-devices/device_connect_to_hub_demo.gif)
_Device connecting and disconnecting to IoT Hub_

Great, the service is now connecting to the IoT Hub as expected. Let’s update the Worker class so we can send events/messages to the hub. Since the service is simulating an IoT device, pretend it is a doorbell sending doorbell pressed events at every 60 seconds interval.

```
    public class Worker : BackgroundService
    {
        ...
        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            while (!stoppingToken.IsCancellationRequested && _clientIsConnected)
            {
                SendDeviceToCloudMessageAsync("Ding Dong").Wait();
                await Task.Delay(60000, stoppingToken);
            }
        }

        private async Task SendDeviceToCloudMessageAsync(String messageToSend)
        {
            Message message = new Message(Encoding.ASCII.GetBytes(messageToSend));
            message.Properties.Add("DoorbellPressed", "true");

            _logger.LogInformation("Sending message at: {time}", DateTimeOffset.Now);

            try
            {
                await _client.SendEventAsync(message);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Fail to send message at: {time}", DateTimeOffset.Now);
            }
        }
    }
```

Now redeploy the service again. If we go to the IoT Hub in Azure Portal, we should be able to see some telemetry showing that service have been sending messages to the hub.

![HomeGateway messages sent telemetry](/assets/posts/2021-04-28-getting-started-with-azure-iot-devices/image_2021-04-28_223933.png)
_HomeGateway messages sent telemetry_
