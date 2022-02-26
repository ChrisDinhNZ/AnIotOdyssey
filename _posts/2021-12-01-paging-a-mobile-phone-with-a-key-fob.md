---
layout: post
title: Paging A Mobile Phone With A Key Fob
date: 2021-12-01
background: '/assets/posts/2021-12-01-paging-a-mobile-phone-with-a-key-fob/post-banner-2021-12-01-paging-a-mobile-phone-with-a-key-fob.jpg'
Tag:
    - Internet of Things
    - Raspberry Pi
    - Python
    - DIY IoT
---

Growing up, I remembered a friend of mine who had a pager which he usually have strapped to his belt. I thought they were pretty cool, doesn’t matter if we were down at `Block Buster` or at the local cinema, his beeper would go off and he would find a payphone and respond to the message.

![A pager](/assets/posts/2021-12-01-paging-a-mobile-phone-with-a-key-fob/pager.png)

_A pager_

## The High Level Overview

In this blog post, I am going to be using a 433MHz key fob transmitter to act as a panic button. It has 2 buttons (one to indicate panic button is activated and the other to indicate panic button is deactivated). A 433MHz receiver will be connected to a Raspberry Pi (acting as a gateway to a backend service). The backend service will forwards it to a mobile network which in turn forwards it to a pre-configured mobile number.

![The High Level Overview](/assets/posts/2021-12-01-paging-a-mobile-phone-with-a-key-fob/high_level_sequence_diagram.png)

## Handling the key fob button events

In this section, the focus will be connecting a 433MHz RF receiver to a Raspberry Pi so that it can intercept and process the signal from the 433MHz key fob transmitter.

**A little bit about 433MHz RF**

Radio communication over this UHF band have been around for a long time and is very common in the consumer electronic space. The 433MHz transmit and receiver modules are fairly easy to obtain and are very cheap. I won’t be going into details on how they works as there are plenty of great online resources on the topic. One tutorial I thought was really good at laying out the concepts was [How 433MHz RF Tx-Rx Modules Work & Interface with Arduino](https://lastminuteengineers.com/433mhz-rf-wireless-arduino-tutorial/).

**The neccessary hardwares**

* 433MHz transmitter (key fob)
* 433MHz receiver (KR2402)
* Raspberry Pi with Raspberry OS installed

I have chosen not to use a bare transmitter/receiver module as I don’t need to send custom data in this case. All I need is a device to indicate something has happened, so a one or two buttoned device is all I needed (a key fob is perfect and it's something I can just buy off the shelf).

**Getting the transmitter and receiver talking**

I got my 2-channel RF relay receiver/key fob transmitter from `AliExpress`. Depending on where we are, we might be able to get them cheaper on eBay, Amazon, etc.

![Relay receiver and key fob](/assets/posts/2021-12-01-paging-a-mobile-phone-with-a-key-fob/keyfob_receiver.jpg)

_Relay receiver and key fob_

![Relay receiver board layout](/assets/posts/2021-12-01-paging-a-mobile-phone-with-a-key-fob/kr2402_large-1.jpg)

_Relay receiver board layout_

First thing we want to do is ensure the receiver can receive the signals from the key fob.

Looking at the board layout above, we see that the connector along the bottom takes anything between 5 – 30VDC. This is the input voltage that will drive the receiver (therefore we can connect a 9V battery to power it up, with the positive lead connected to V and the negative lead connected to G).

Once the receiver is powered up, we can clear the receiver of any previously stored transmitter sequence. We can do this by pressing the Learning Button 8 times. After that, the receiver should ignore any transmission from the key fob (no LED should be flashing when either button A or B is pressed).

Next we will train the receiver to accept the transmission from the key fob where each button will activate a relay on the receiver. We are going to train the receiver to treat the buttons as momentary presses (i.e. activated when the button is pressed, deactivated when the button is released). This is done by pressing the Learning Button once (more options can be found at [Qiachip.com](https://qiachip.com/blogs/usermenu/dc6-30v-2channel-receiver-instruction-kr2402a)). After that, we should see the LED on the receiver changes state when a button is pressed or released.

<iframe width="560" height="315" src="https://www.youtube.com/embed/CBE7qlTLCxU" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture">&nbsp;</iframe>

**Connecting relay receiver to the Raspberry Pi**

Instead of powering the relay receiver with a 9V battery, we are going to power it using the 5V line on the Raspberry Pi. All we need to do is connect the 5V line on the Raspberry Pi to the input voltage terminal (V) on the relay receiver, then connect a ground pin on the Raspberry Pi to the ground terminal (G) on the relay receiver. Also, when wiring things up, make sure the Raspberry Pi is powered off so we don’t accidentally short something and damage the devices.

The remaining thing to do is connect each relay to a GPIO pin on the Raspberry Pi. We are also going to connect them up as normally opened (NO) mode. What this means is normally the relay will act as an opened circuit. Only when a button is pressed on the key fob, the receiver magnetised the relay, resulting in a closed circuit (GPIO pin active). When the button is released on the key fob, the receiver de-magnetised the relay, resulting in an opened circuit (GPIO pin inactive). In the diagram below, the top relay drives GPIO2 while the bottom relay drives GPIO3.

![Raspberry Pi to relay receiver wiring diagram](/assets/posts/2021-12-01-paging-a-mobile-phone-with-a-key-fob/rpi_relay_connection.jpg)
_Raspberry Pi to relay receiver wiring diagram_

![Raspberry Pi to relay receiver actual wiring](/assets/posts/2021-12-01-paging-a-mobile-phone-with-a-key-fob/rpi_relay_actual_connection.jpg)
_Raspberry Pi to relay receiver actual wiring_

<iframe width="560" height="315" src="https://www.youtube.com/embed/pLEoZh5Z8M4" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture">&nbsp;</iframe>

At this point we can place the Raspberry Pi and relay receiver anywhere around the house as we no longer need to physically access them.

## Processing the GPIO inputs on the Raspberry Pi

Now that the Raspberry Pi have been tucked away in a cabinet somewhere, we can actually setup a SSH session with the Raspberry Pi and code on it. In the following example I am using VS Code to SSH into the Raspberry Pi.

![Raspberry Pi to relay receiver actual wiring](/assets/posts/2021-12-01-paging-a-mobile-phone-with-a-key-fob/vsc_ssh_into_pi_demo.gif)

In the demo above, what we did was SSH into the Raspberry Pi then select the project directory (or create a new one if you like). But more importantly, we want to create the following files:

* App.py
* ButtonMonitor.py
* ButtonObserver.py

**App.py**

This is the main application entry point. All it does is create a ButtonObserver instance and two ButtonMonitor instance. The code for App.py is as follow:

```
    '''
    App.py
    '''
    import asyncio
    import sys
    import signal

    from ButtonMonitor import ButtonMonitor
    from ButtonObserver import ButtonObserver

    async def main():

        execution_is_over = asyncio.Future()

        def abort_handler(*args):
            execution_is_over.set_result("Ctrl+C")
            print("")
            print("Cleaning up before exiting...")
            sys.exit(0)

        signal.signal(signal.SIGINT, abort_handler)

        try:
            observer = ButtonObserver()
            ButtonMonitor(2, "A", observer)  # Monitor button connected to input 2
            ButtonMonitor(3, "B", observer)  # Monitor button connected to input 3

            exit_reason = await execution_is_over
            print('Exit reason detected: %r' % exit_reason)

        finally:
            pass

    if __name__ == "__main__":
        asyncio.get_event_loop().run_until_complete(main())
```

**ButtonMonitor.py**

This file defines the ButtonMonitor class. This is a simple class to monitor a button via a GPIO input pin and notifies the observer when the button is pressed (GPIO Zero library is used to monitor the GPIO pins). The code for ButtonMonitor.py is as follow:

```
    '''
    ButtonMonitor.py
    '''

    from gpiozero import Button

    class ButtonMonitor:
        def __init__(self, input_pin, name, observer):
            self.button = Button(input_pin)
            self.button_name = name
            self.observer = observer

            self.button.when_pressed = self.button_pressed

        def button_pressed(self):
            if self.observer is None:
                pass  # No observer so do nothing
            else:
                self.observer.handle_button_pressed(self.button_name)
```

**ButtonObserver.py**

This file defines the ButtonObserver class. This is a simple class which handles the button presses (which for now just print out which button was pressed). The code for ButtonObserver.py is as follow:

```
    '''
    ButtonObserver.py
    '''

    class ButtonObserver:
        def handle_button_pressed(self, source):
            print("Button %r pressed" % source)
```

In the following demo, I am using [Termius](https://termius.com/) on a mobile phone to SSH into the Raspberry Pi. We can see in the console log which button is being pressed.

<iframe width="560" height="315" src="https://www.youtube.com/embed/WV2bvAm777I" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture">&nbsp;</iframe>
_Handling button presses on the Raspberry Pi demo_

## Forwarding Events To The Cloud

In this section we will look at connecting the Raspberry Pi to an Azure IoT hub. We have already covered Azure IoT hub and how it works in some of the previous blog posts, so here we will focus specifically on the key fob button events.

**Provisioning a new Azure IoT Device for the Raspberry Pi**

Assuming an Azure IoT hub is up and running, we can easily register a new Azure IoT device using Powershell.

```
    PS D:\Workspace\IoTHub> $MyHomeAgent= Add-AzIotHubDevice -ResourceGroupName "your-resource-group-name" -IotHubName "your-azure-iot-hub-name" -DeviceId "MyHomeAgent" -AuthMethod "shared_private_key"
    PS D:\Workspace\IoTHub>
```

**Connecting the Raspberry Pi to Azure IoT Hub**

To connect to an Azure IoT hub, we need the device connection string. The commands below allows us to retrieve the connection string for the device we created.

```
    PS D:\Workspace\IoTHub> $TempObject = Get-AzIotHubDeviceConnectionString -ResourceGroupName "your-resource-group-name" -IotHubName "your-azure-iot-hub-name" -DeviceId "MyHomeAgent" -KeyType secondary
    PS D:\Workspace\IoTHub> $TempObject.ConnectionString
    HostName=your-azure-iot-hub-name.azure-devices.net;DeviceId=MyHomeAgent;SharedAccessKey=my-home-agent-shared-access-key
    PS D:\Workspace\IoTHub>
```

We will need to install Azure IoT Device SDK

```
    PS D:\Workspace\IoTHub> pip install azure-iot-device
```

Now create a class to interface with the Azure IoT hub.

```
    # AzureIotDeviceAgent.py

    import asyncio

    from azure.iot.device.aio import IoTHubDeviceClient

    class AzureIotDeviceAgent:
        def __init__(self, name, connection_string, client = None):
            self.name= name
            self.device_client = IoTHubDeviceClient.create_from_connection_string(connection_string)
            self.client = client

        async def connect(self):
            await self.device_client.connect()

        async def disconnect(self):
            await self.device_client.disconnect()
```

We will also need to update main() to initiate a connection with the Azure IoT hub.

```
    # App.py

    ...
    from AzureIotDeviceAgent import AzureIotDeviceAgent

    async def main():

        ...
        my_home_agent = None
        my_home_agent_name = "MyHomeAgent"
        my_home_agent_connection_string = "my-home-agent-connection-string"
        ...

        try:
            ...
            my_home_agent = AzureIotDeviceAgent(my_home_agent_name, my_home_agent_connection_string)
            await my_home_agent.connect()

        finally:
            pass
```

For now that is all the changes we need to connect to the Azure IoT hub.

If we run the following Powershell command while App.py is running we should see that it is connected. Likewise, when App.py is not running we should see that it is not connected.

```
    PS D:\Workspace\IoTHub> Get-AzIotHubDevice -ResourceGroupName "your-resource-group-name" -IotHubName "your-azure-iot-hub-name" -DeviceId "MyHomeAgent" | Select-Object -Property Id, ConnectionState
    Id          ConnectionState
    --          ---------------
    MyHomeAgent    Connected
    PS D:\Workspace\IoTHub>
```

**Sending data to Azure IoT Hub**

So far we have only implemented ButtonObserver.py to print out if button A or B is pressed. What we want to do now is indicate to the Azure Iot hub that Assistance requested when button A is pressed and Assistance request cancelled when button B is pressed.

```
    # App.py

    ...

    async def main():

        ...

        try:
            ...
            observer.set_agent(my_home_agent)
            my_home_agent.set_client(observer)

        finally:
            pass
```

```
    # ButtonMonitor.py

    ...
    import asyncio

    class ButtonMonitor:
        ...

        async def button_pressed(self):
            if self.observer is None:
                pass  # No observer so do nothing
            else:
                # Create new asyncio loop
                loop = asyncio.new_event_loop()
                asyncio.set_event_loop(loop)
                future = asyncio.ensure_future(self.__execute_when_pressed__(self.button_name)) # Execute async method
                loop.run_until_complete(future)
                loop.close()

        async def __execute_when_pressed__(self, source):
            await self.observer.handle_button_pressed(source)
```

```
    # ButtonObserver.py

    ...
    from pb_Parcel_pb2 import pb_Parcel

    class ButtonObserver:
        ...

        def __pack_button_event__(self, assistance_requested):
            parcel = pb_Parcel()

            # We don't need to populate domain and domain agent, that will
            # be done by the domain agent.
            parcel.source.name = "My Pager"
            parcel.source.local_id = parcel.source.name

            # We only need to indicate destination endpoint name here.
            # Other info will be derived by backend services.
            parcel.destination.name = "My Mobile"

            parcel.type = "ASCII"

            # In this case, the content is just a string. However if content
            # were a protobuffer object, then we will need convert it to a string
            # as SerializeToString() returns a byte array.
            # e.g. parcel.content = str(content.SerializeToString(), 'utf-8')
            if assistance_requested:
                parcel.content = "Assistance requested"
            else:
                parcel.content = "Assistance request cancelled"

            return parcel

        async def handle_button_pressed(self, source):
            if self.agent is None:
                return

            assistance_requested = False

            if source == "A":
                assistance_requested = True

            parcel = self.__pack_button_event__(assistance_requested)
            await self.agent.send_parcel(parcel)

        ...
```

We extended the existing code (as shown above) to do two main things:

* Encapsulate the button event inside a pb_Parcel `protobuf` object and serialised it.
* Send the serialised pb_Parcel to the Azure IoT hub.

## Forwarding Events To A Mobile Phone

In this section, we will be using Azure Functions and Twilio’s messaging service to process and forward those events to a specified mobile number.

When the Azure IoT Hub receives an event, they will be stored in an Event Hub for processing. We can implement an Azure Function App to process these events. For more info on how to create an Azure Function App to handle Event Hub messages, check out [Processing Azure IoT Hub Events Using Azure Function](/2021/05/16/processing-azure-iot-hub-events-using-azure-function.html).

For this case, we need to deserialise the hub event, determine the destination mobile number based on the destination name in the event's payload (in my case I have an Azure Table which I can look up, but you can also provide the number prior to serialising the event).

```
    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Threading.Tasks;
    using Google.Protobuf;
    using Microsoft.Azure.EventHubs;
    using Microsoft.Azure.WebJobs;
    using Microsoft.Extensions.Logging;
    using Twilio;
    using Twilio.Rest.Api.V2010.Account;

    namespace PagingDemo
    {
        public class PagingEventHandler
        {
            [FunctionName("PagingEventHandler")]
            public async Task Run(
                [EventHubTrigger("HubName", Connection = "HubConnection")] EventData[] events,
                ILogger log)
            {
                var exceptions = new List<Exception>();

                foreach (EventData eventData in events)
                {
                    try
                    {
                        string messageBody = Encoding.UTF8.GetString(eventData.Body.Array, eventData.Body.Offset, eventData.Body.Count);

                        pb_Parcel parcel = new pb_Parcel();
                        parcel.MergeFrom(Encoding.UTF8.GetBytes(messageBody));

                        await ForwardPagingEvent(parcel);
                    }
                    catch (Exception e)
                    {
                        exceptions.Add(e);
                    }
                }

                if (exceptions.Count > 1)
                    throw new AggregateException(exceptions);

                if (exceptions.Count == 1)
                    throw exceptions.Single();
            }

            private static async Task<string> ForwardPagingEvent(pb_Parcel parcel)
            {
                var fromNumber = {The Twilio number to use to send the message};
                var toNumber = {Message recipient's mobile number};
                var body = {message reflecting the event}

                /*
                    Twilio credentials stored in application settings.
                */
                TwilioClient.Init(
                    Environment.GetEnvironmentVariable("TwilioAccountSid"),
                    Environment.GetEnvironmentVariable("TwilioAuthToken"));

                var message = MessageResource.Create(
                    body: body,
                    from: new Twilio.Types.PhoneNumber(fromNumber),
                    to: new Twilio.Types.PhoneNumber(toNumber)
                );

                return await Task.FromResult("Handled");
            }
        }
    }
```

To forward the parcel as an SMS, we need to convert the parcel to an SMS format message:

* To: the mobile number to send to.
* From: A registered Twilio mobile number used to send the SMS.
* Body: A text message to reflect the event

To send the SMS using Twilio, we initialised a TwilioClient by passing in our Twilio account cedentials. Then send the message by invoking MessageResource.Create() method.

Now deploy the Function App(s) and we should be all set.

<iframe width="560" height="315" src="https://www.youtube.com/embed/WjwMBzaS5io" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture">&nbsp;</iframe>
_Key fob paging a mobile phone demo_

