---
layout: post
title: Controlling IoT Devices With Siri Shortcuts
date: 2022-07-13 20:42 +1200
background: '/assets/posts/2022-07-13-controlling-iot-devices-with-siri-shortcuts/post-banner-2022-07-13-controlling-iot-devices-with-siri-shortcuts.jpg'
Tag:
  - Arduino
  - Raspberry Pi
  - IoT
  - Azure
  - Siri Shortcuts
---

This is part 3 of a three-part blog series where we will be looking at connecting my old [Kenwood KR-V7080 Receiver](https://www.hifiengine.com/manual_library/kenwood/kr-v7080.shtml) to the internet and controlling it using [Siri Shortcuts](https://apps.apple.com/us/app/shortcuts/id915249334). In this post we will be:

* Connecting the KR-V7080 Receiver to an Azure IoT Hub
* Expose the KR-V7080 Receiver to the World Wide Web with an Azure Function
* Setting up Siri shortcuts to control the KR-V7080 Receiver using voice commands

If you feel like you need a bit of context to get up to speed, do check out my previous posts in the series:

* [Part 1: An Introduction To IR Modules Using An Arduino Uno](/2022/04/17/an-introduction-to-ir-modules.html)
* [Part 2: Bluetooth Communication Using A HM-10 Module](/2022/05/09/bluetooth-communication-using-a-hm-10-module.html)

## But Why?

1. Ok, I know some of you are wondering why would anyone want to use this old thing nowadays let alone trying to control it from the phone. Well you are somewhat right, it is an old thing and I wouldn't be getting more than $100 for it. So I thought it would be a more fun challenge to lift the lid and see what it takes to connect it to the internet and control it with our voice.

2. Why voice commands? So I was watching [Ad Astra](https://en.wikipedia.org/wiki/Ad_Astra_(film)) the other day, it's a Sci-Fi movie set in a not so distant future. Well there was scene where Brad Pitt was composing an email/message to his wife via an AI personal assistant and I thought how cool it would be to control things around the house with voice commands. Initially I thought of using [Amazon Alexa](https://en.wikipedia.org/wiki/Amazon_Alexa), but then I stumbled on Siri Shortcuts which seems like an easier path given I have an Apple device already.

## Connecting the KR-V7080 Receiver to an Azure IoT Hub

From a high level point of view, what we got by the end of the previous blog post is shown below.

![Bluetooth Communication Using A HM-10 Module](/assets/posts/2022-07-13-controlling-iot-devices-with-siri-shortcuts/high_level_sequence_diagram_nrf_connect_hm10.png)

In this section, what we want to get to is shown below.

![Communicating With The KR-V7080 From Azure IoT Hub](/assets/posts/2022-07-13-controlling-iot-devices-with-siri-shortcuts/high_level_sequence_diagram_azure_iot_hub_krv7080.png)

##### Integrating the Raspberry Pi into the solution

Establishing a BLE data connection between a Raspberry Pi and an Arduino device is fairly straight forward process given we have already covered this in the post [Raspberry Pi, meet Arduino. Arduino, meet Raspberry Pi. Let’s talk Bluetooth (LE)](/2021/10/01/raspberry-pi-meet-arduino-arduino-meet-raspberry-pi-let-s-talk-bluetooth-le.html). Let's create a small python application so that we can communicate with our Arduino Uno.

`KenwoodRemoteAgent.py`
```
   '''
   Description: A helper class to simulate a Kenwood RC-R0803 remote over BlE.
   '''

   import asyncio
   from bleak import BleakScanner
   from bleak.backends.bluezdbus.client import BleakClientBlueZDBus

   class KenwoodRemoteAgent:
      def __init__(self, device_name, data_channel_uuid, logger):
         self.device_name = device_name
         self.data_channel_uuid = data_channel_uuid
         self.logger = logger
         self.device_found = False
         self.device_connected = False
         self.client = None

      async def run(self):
         while not self.device_found:
               device = await BleakScanner.find_device_by_filter(
                  lambda d, ad: d.name and d.name.lower() == self.device_name.lower()
               )

               if device is None:
                  self.logger.info("{} not found".format(self.device_name))
                  await asyncio.sleep(1)
               else:
                  self.logger.info("{} found".format(device))
                  self.device_found = True

         self.client = BleakClientBlueZDBus(device)

         while not self.device_connected:
               try:
                  if await self.client.connect():
                     self.device_connected = True
                     self.logger.info("Connected to {}".format(self.device_name))
               except:
                  self.logger.info("Connected to {} failed".format(self.device_name))

               if not self.device_connected:
                  await asyncio.sleep(1)
                  self.logger.info("Retrying...")

      async def send_command(self, command: str):
         await self.client.write_gatt_char(self.data_channel_uuid, command.encode('UTF-8'))

      async def stop(self):
         if not self.device_connected:
               return

         try:
               if await self.client.disconnect():
                  self.device_connected = False
                  self.logger.info("Disconnected from {}".format(self.device_name))
         except:
               self.logger.info("Disconnected from {} failed".format(self.device_name))
```

`app.py`
```
   '''
   Description: Simple app to interact with HM-10 BLE module.
   '''
   import asyncio
   import logging
   import signal
   import sys
   from time import gmtime
   from turtle import delay

   from KenwoodRemoteAgent import KenwoodRemoteAgent

   logging.basicConfig(filename='/home/pi/MyHomeAgent/events.log', encoding='utf-8', format='%(asctime)s %(module)-20s %(message)s', level=logging.DEBUG)
   logging.Formatter.converter = gmtime

   device_name = "KR-V7080"
   data_channel_uuid = "0000ffe1-0000-1000-8000-00805f9b34fb"

   async def main():

      kenwood_remote_agent = None
      execution_is_over = asyncio.Future()

      async def abort_handler(signame):
         execution_is_over.set_result("Ctrl+C")
         logging.info("Cleaning up before exiting...")

         if kenwood_remote_agent is not None:
               await kenwood_remote_agent.stop()

      loop = asyncio.get_event_loop()
      for signame in ('SIGINT', 'SIGTERM'):
         loop.add_signal_handler(getattr(signal, signame),
                                 lambda: asyncio.ensure_future(abort_handler(signame)))

      try:
         logging.info("Started...")
         kenwood_remote_agent = KenwoodRemoteAgent(device_name, data_channel_uuid, logging)
         await kenwood_remote_agent.run()

         # Send command to toggle power on/off
         await kenwood_remote_agent.send_command("power#")
         await asyncio.sleep(3)
         # Send an unknown command
         await kenwood_remote_agent.send_command("Blah blah#")

         await execution_is_over
         asyncio.get_event_loop().stop()

      except Exception:
         logging.error("Stopped!")

   if __name__ == "__main__":
      asyncio.get_event_loop().run_until_complete(main())
```

Let's review the code above to get a rough idea what it's doing. The `main()` method in `app.py` basically create an instance of KenwoodRemoteAgent class with the name `KR-V7080` and the `uuid` (universally unique identifier) associated with the custom BLE characteristic provided by the HM-10 module. It's instance method `run()` then gets invoked where it will try to find the KR-V7080 device and connects to it.

Once connected, the main() method will send the command `power#` followed by `Blah blah#` to the HM-10 module. The HM-10 module acts purely as data channel, it simply take what it received and passes it onto the Arduino Uno and vice versa. Note that the sketch loaded onto the Arduino Uno defines a table of valid commands, so only valid commands are forwarded to the IR Tx module and transmitted to the Kenwood KR-V7080 Receiver.

![Raspberry Pi Communicating With HM-10](/assets/posts/2022-07-13-controlling-iot-devices-with-siri-shortcuts/pi_and_hm10_ble_demo.gif)

The demo above shows the terminal on the left where we ran the `app.py` program while the terminal on the right showing the activities on the Arduino Uno.

##### Integrating Azure IoT Hub into the solution

Carrying on from my previous blog posts on Azure IoT Hub we will be using resources created from those blog posts and integrate them into this solution to save me some time. If you want to learn more about Azure IoT Hub, feel free to check them out below:

* [Getting Started With Azure IoT Hub](/2021/04/18/getting-started-with-azure-iot-hub.html)
* [Getting Started With Azure IoT Devices](/2021/04/28/getting-started-with-azure-iot-devices.html)
* [Processing Azure IoT Hub Events Using Azure Function](/2021/05/16/processing-azure-iot-hub-events-using-azure-function.html)
* [Implementing an Azure IoT Device Using Python](/2021/07/28/implementing-an-azure-iot-device-using-python.html)
* [Paging A Mobile Phone With A Key Fob](/2021/12/01/paging-a-mobile-phone-with-a-key-fob.html)

We are going to expand our little application to connect to the Azure IoT Hub as `MyHomeAgent`, an Azure IoT device I created in a previous blog post.

`AzureIotDeviceAgent.py`
```
   '''
   Description: A helper class to interact with Azure IoT Hub.
   '''

   import logging
   from pb_Parcel_pb2 import pb_Parcel
   from azure.iot.device.aio import IoTHubDeviceClient
   from google.protobuf import json_format
   from azure.iot.device import MethodResponse

   class AzureIotDeviceAgent:
      def __init__(self, name: str, connection_string: str, logger: logging):
         self.name= name
         self.device_client = IoTHubDeviceClient.create_from_connection_string(connection_string)
         self.device_client.on_method_request_received = self.method_request_handler
         self.logger = logger
         self.clients = []

      async def connect(self):
         if self.device_client.connected:
               return

         await self.device_client.connect()
         self.logger.info("{} connected".format(self.name))

      async def disconnect(self):
         if not self.device_client.connected:
               return

         await self.device_client.disconnect()
         self.logger.info("{} disconnected".format(self.name))

      # Add handler for incoming parcels
      def add_client(self, client):
         self.clients.append(client)

      async def send_parcel(self, parcel: pb_Parcel):
         if not self.device_client.connected:
               self.logger.error("{} not connected".format(self.name))
               return

         # Populate source domain and domain agent
         parcel.source.domain_agent = self.name
         parcel.source.domain = "Device Domain"
         parcel = parcel.SerializeToString()

         # Note that parcel here is serialised to a byte array, not UTF8 string.
         await self.device_client.send_message(parcel)

      async def method_request_handler(self, method_request):
         status_code = 200
         payload = {"result": True, "data": "parcel handled"}

         if method_request.name != "ProcessMessage":
               status_code = 404
               payload = {"result": False, "data": "unknown method request"}

         parcel = json_format.ParseDict(method_request.payload, pb_Parcel(), True)

         if parcel is None:
               status_code = 400
               payload = {"result": False, "data": "no parcel received"}
         else:
               for client in self.clients:
                  if client.name == parcel.destination.name:
                     await client.process_parcel(parcel)
                     return
               status_code = 503
               payload = {"result": False, "data": "no parcel handler"}

         method_response = MethodResponse.create_from_method_request(method_request, status_code, payload)
         await self.device_client.send_method_response(method_response)
```

`KenwoodRemoteAgent.py`
```
   '''
   Description: A helper class to simulate a Kenwood RC-R0803 remote over BlE.
   '''

   ...

   class KenwoodRemoteAgent:
      def __init__(self, device_name, data_channel_uuid, logger: logging):
         ...

      async def run(self):
         ...

      async def send_command(self, command: str):
         ...

      async def stop(self):
         ...

      async def process_parcel(self, parcel: pb_Parcel):
         if parcel.type == "ASCII":
               await self.send_command(parcel.content)
         else:
               self.logger.warning("Unexpected content type: {}".format(parcel.type))
```

`pb_Endpoint.proto`
```
   syntax = "proto3";

   message pb_Endpoint {
   string name           = 1;
   string local_id       = 2;
   string domain_agent   = 3;
   string domain         = 4;
   }
```

`pb_Parcel.proto`
```
   syntax = "proto3";

   import "pb_Endpoint.proto";

   message pb_Parcel {
   pb_Endpoint source      = 1;  // Sender endpoint
   pb_Endpoint destination = 2;  // Recipient endpoint
   string type             = 3;  // Name of parcel content
   string content          = 4;  // UTF-8 encoded of parcel content
   }
```

`app.py`
```
   '''
   Description: Simple app to interact with HM-10 BLE module.
   '''

   ...

   from AzureIotDeviceAgent import AzureIotDeviceAgent

   my_home_agent_name = "MyHomeAgent"
   my_home_agent_connection_string = "my-home-agent-connection-string"

   async def main():
      ...

      my_home_agent = None

      try:
         ...

         my_home_agent = AzureIotDeviceAgent(my_home_agent_name, my_home_agent_connection_string, logging)
         await my_home_agent.connect()
         my_home_agent.add_client(kenwood_remote_agent)

      except Exception:
         ...

         if my_home_agent is not None:
               my_home_agent.disconnect()

         logging.error("Stopped!")

   if __name__ == "__main__":
      asyncio.get_event_loop().run_until_complete(main())
```

Now let's run through the code.

We added `AzureIotDeviceAgent.py`, a helper class for connecting to the IoT Hub as well as sending and receiving data from it.
When we want to send data to the IoT Hub, we can invoke the `send_parcel()` method.
When we want to receive data, we register a client via the `add_client()` method. If the incoming data is addressed to the registered client, then the data is passed to that client for handling via the client's `process_parcel()` method.

`KenwoodRemoteAgent.py` was modified to include the `process_parcel()` method as this will be called if data was sent over the internet addressed to this device (`KR-V7080`). All that `process_parcel()` does is forward the data to the Arduino Uno over BLE.

The `main()` method in `app.py` was modified to create an instance of AzureIotDeviceAgent, establishes a connection with the IoT Hub and registers an instance of KenwoodRemoteAgent as a client.

The data that are sent to and received from the IoT Hub are encapsulated as pb_Parcel objects (`parcels`) which contains information about where it came from, where it needs to go and what type of data is in the payload. To help us send these parcels between devices/services/apps over the internet, I have implemented a backend solution (`Multilinks`) consisting of a collection of serverless functions to ensure parcels are forwarded to the correct destination. I hope to blog about Multilinks in more details at some point, but for now just assume a back end service exists to hand parcels over the internet.

## Talking to the KR-V7080 Receiver from the Internet using an Azure Function

At this point we have only connected the KR-V7080 Receiver to the IoT Hub. However we don't really have a way to communicate with the IoT Hub and control the KR-V7080 Receiver.

In this section, we will adding an [Azure Function](https://docs.microsoft.com/en-us/azure/azure-functions/) as a [webhook](https://www.twilio.com/docs/glossary/what-is-a-webhook) to communicate with the KR-V7080 Receiver. The solution will look something like the diagram shown below.

![Communicating With The KR-V7080 Via A Webhook](/assets/posts/2022-07-13-controlling-iot-devices-with-siri-shortcuts/high_level_sequence_diagram_webhook_krv7080.png)

As I have mentioned earlier, I have already implemented `Multilinks`, a backend service that is capable of passing data between two endpoint across the internet. The only requirement is that all endpoints need to be registered with Multilinks' Endpoint Registry, this allows Multilinks to handle the delivery of the data without the sender needing to worry about how it's going to get there. The concept is very similar to the real world scenario where we want to send a parcel to someone. First we put the content in a box, then label where it needs to go and who sent it, then drop it off at the post office. We don't really worry if it's going to get there by land, sea or air as long as it gets there.

The webhook that we are going to create is analogous to the post office where we dropped off the parcel. An implementation of this webhook is shown below.

`ExternalServicesInboundAgent.cs`
```
   using System;
   using System.IO;
   using System.Threading.Tasks;
   using Microsoft.AspNetCore.Mvc;
   using Microsoft.Azure.WebJobs;
   using Microsoft.Azure.WebJobs.Extensions.Http;
   using Microsoft.AspNetCore.Http;
   using Microsoft.Extensions.Logging;
   using Newtonsoft.Json;
   using Microsoft.Azure.WebJobs.Extensions.DurableTask;

   namespace Multilinks.InboundAgents
   {
      public class ExternalServicesInboundAgent
      {
         [FunctionName("ExternalServicesInboundAgent")]
         public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] HttpRequest req,
            [DurableClient] IDurableOrchestrationClient starter,
            ILogger log)
         {
            log.LogInformation("[ExternalServicesInboundAgent] Parcel received");

            string requestBody = String.Empty;

            using (StreamReader streamReader = new StreamReader(req.Body))
            {
               requestBody = await streamReader.ReadToEndAsync();
            }

            dynamic data = JsonConvert.DeserializeObject(requestBody);

            string source = data?.source;
            string destination = data?.destination;
            string type = data?.type;
            string content = data?.content;

            if (string.IsNullOrEmpty(source) ||
               string.IsNullOrEmpty(destination) ||
               string.IsNullOrEmpty(type) ||
               string.IsNullOrEmpty(content))
            {
               log.LogError("[ExternalServicesInboundAgent] Invalid parcel");
               return new BadRequestObjectResult("Parcel paramters Source, Destination, Type and Content are all required.");
            }

            pb_Parcel parcel = new pb_Parcel();
            parcel.Source = new pb_Endpoint();
            parcel.Destination = new pb_Endpoint();

            parcel.Source.Name = source;
            parcel.Destination.Name = destination;
            parcel.Type = type;
            parcel.Content = content;

            /*
               Let's start the orchestration.
            */
            var instanceId = await starter.StartNewAsync("MultilinksOrchestrator", parcel);

            log.LogInformation("[ExternalServicesInboundAgent] Parcel sent");
            return new OkObjectResult("Parcel sent");
         }
      }
   }
```

Let’s review the code above to get a rough idea of what is going on.

Starting with the function header declaration, we see that it takes 3 parameters:

* `[HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] HttpRequest req`
* `[DurableClient] IDurableOrchestrationClient starter`
* `ILogger log`

The first is what will trigger this function to be executed, in this case a HTTP POST request. The `AuthorizationLevel.Function` indicates this function is protected by a SAS key (Check out the [Azure Functions HTTP trigger](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-http-webhook-trigger) under the section `Function access keys`).

The second is a `DurableClient` which is used to specify which `Durable Function Orchestrator` to handle the request.

Lastly a logging service for logging purposes.

We expects that the POST request to have a json typed payload containing the following fields:

* source
* destination
* type
* content

If any of these are missing, indicate to the sender with a `bad request` status response. Otherwise, create a parcel using the provided data and pass it onto a function orchestrator to handle i.e. `MultilinksOrchestrator`.

Right, let's try out this new webhook. We are going to use [Postman](https://www.postman.com/product/tools/) to send a POST request. In theory, the request will propagate along as shown in the sequence diagram above and act on the KR-V7080 Receiver as requested.

![Webhook Requests Demo](/assets/posts/2022-07-13-controlling-iot-devices-with-siri-shortcuts/webhook_requests_demo.gif)

In the demo above, we posted 3 HTTP requests.

* In the first, we sent a POST request without any payload and as expected we got a Bad Request response.
* In the second, we sent a POST request with a valid payload. However, the addressed endpoint has not been registered in the `endpoint registry` (plus was not addressed to KR-V7080) so the request was never forwarded to the Arduino Uno and we see no output in the Arduino Uno's serial console.
* Lastly, we sent a POST request with a valid payload and is addressed to KR-V7080. The request was forwarded to the Arduino Uno and we see in the Arduino Uno's serial console that it has received the request and have sent the appropriate IR command to the KR-V7080 receiver.

## Setting up Siri shortcuts to control the KR-V7080 Receiver using voice commands

In this section we are going to expand on the previous section by controlling the KR-V7080 Receiver via voice commands. The solution will look something like the diagram shown below.

![Communicating With The KR-V7080 Via Voice Commands](/assets/posts/2022-07-13-controlling-iot-devices-with-siri-shortcuts/high_level_sequence_diagram_user_krv7080.png)

As mentioned at the start of this blog post, we will be using [Siri Shortcuts](https://support.apple.com/en-nz/guide/shortcuts/welcome/ios) with an Apple Watch to trigger the voice commands. Specifically, we want to use Siri Shortcuts to send a HTTP POST request whenever we say `Stereo On` or `Stereo Off`. I will be summarising what we needed to do for our scenario, but definitely checkout the Siri Shortcuts link above if you want to learn more.

![Siri Shortcuts](/assets/posts/2022-07-13-controlling-iot-devices-with-siri-shortcuts/siri_shortcut_1.png)

In the image above, two shortcuts have been created. One for when Siri is asked to turn the `stereo on` and one for when Siri is asked to turn the `stereo off`.

![Siri Shortcuts Details](/assets/posts/2022-07-13-controlling-iot-devices-with-siri-shortcuts/siri_shortcut_2.png)

In the image above, we see the details of the `stereo on` shortcut:

* The sub heading `Hey Siri, Stereo On` is the voice command that will trigger this shortcut
* The request will be sent to `https://webhookaddress.com?code=access-code-goes-here`
* The request will be a HTTP `POST`
* The payload will be a `JSON` object containing the `parcel` data

Below is a demo of the Shortcuts in action. What we have is an Arduino Uno, an Apple Watch and the Kenwood stereo. The Apple Watch is used to issue voice commands which Siri processes and sends them off onto the Internet, which then gets forward to the Arduino Uno which turns the stereo on and off.

<iframe width="840" height="472" src="https://www.youtube.com/embed/JnNagd_-_aM" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture">&nbsp;</iframe>

## Final thoughts

While we achieved what we set out to do, I wasn't too happy with the latency I was seeing (but I guess is expected given the solution involves so many network-bound tasks). E.g:

* Siri processing the voice command and converting it to a HTTP request
* The serverless function may require a [cold start](https://azure.microsoft.com/en-us/blog/understanding-serverless-cold-start/)
* Processing the request sent from the IoT Hub to the Raspberry Pi.

In the worst case scenario, it would take around 10-15 seconds to complete a request. But generally, it takes around half of that time. While we can't do much about the latency due to Siri I reckon we definitely can improve the latency of the other two, but I think I 'll leave that for another day.
