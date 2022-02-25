---
layout: post
title: Implementing an Azure IoT Device Using Python
date: 2021-07-28
background: '/assets/posts/2021-07-28-implementing-an-azure-iot-device-using-python/post-banner-2021-07-28-implementing-an-azure-iot-device-using-python.jpg'
Tag:
    - Protocol Buffers
    - Serialization
    - Python
    - IoT
    - Azure IoT Hub
    - Azure Powershell
---

The nice thing about Azure IoT is that there are SDKs already available in popular programming languages including C, C#, Java, Node.js, and Python and therefore should cater for our devices and preferred languages. You can find more details on the specific SDKs can be found with the following links:

* [Constrained Device SDKs](https://docs.microsoft.com/en-us/azure/iot-develop/about-iot-sdks#constrained-device-sdks)
* [Unconstrained Device SDKs](https://docs.microsoft.com/en-us/azure/iot-develop/about-iot-sdks#unconstrained-device-sdks)

In this blog post the focus will be on creating a couple of Azure IoT devices using the Python Device SDK, one for sending events to the IoT hub, while the other for receiving events from the hub.

The scenario is that we have an alarm which is managed by an alarm agent. When the status of the alarm changes, the alarm agent will send an alarm status to the IoT hub. The IoT hub then passes the alarm status to an alarm monitor agent.

![High level sequence diagram](/assets/posts/2021-07-28-implementing-an-azure-iot-device-using-python/AlarmMonitoringSequence.png)
_High level sequence diagram_

## What we will need:

* An Azure account
* Azure PowerShell
* An active Azure IoT Hub
* A Python 3 Environment

## Creating required IoT devices and check their connections

Let’s create the Azure IoT Devices for the `Alarm Agent` and `Alarm Monitor Agent`.

Open a Powershell console and make sure we are signed in with our subscription using Azure Powershell. Once that is done then create the devices as shown below.

```
    PS D:\Workspace\IoTHub> $AlarmAgent= Add-AzIotHubDevice -ResourceGroupName "rg-sea-multilinks" -IotHubName "ih-sea-multilinks" -DeviceId "AlarmAgent" -AuthMethod "shared_private_key"
    PS D:\Workspace\IoTHub>
    PS D:\Workspace\IoTHub> $AlarmMonitorAgent = Add-AzIotHubDevice -ResourceGroupName "rg-sea-multilinks" -IotHubName "ih-sea-multilinks" -DeviceId "AlarmMonitorAgent" -AuthMethod "shared_private_key"
    PS D:\Workspace\IoTHub>
```

Next, retrieved the connection string and assign it to environment variables which can then be use later.

```
    PS D:\Workspace\IoTHub> $TempObject = Get-AzIotHubDeviceConnectionString -ResourceGroupName "rg-sea-multilinks" -IotHubName "ih-sea-multilinks" -DeviceId "AlarmAgent" -KeyType secondary
    PS D:\Workspace\IoTHub> $Env:AlarmAgentConnectionString = $TempObject.ConnectionString
    PS D:\Workspace\IoTHub> $TempObject = Get-AzIotHubDeviceConnectionString -ResourceGroupName "rg-sea-multilinks" -IotHubName "ih-sea-multilinks" -DeviceId "AlarmMonitorAgent" -KeyType secondary
    PS D:\Workspace\IoTHub> $Env:AlarmMonitorAgentConnectionString = $TempObject.ConnectionString
    PS D:\Workspace\IoTHub> $Env:AlarmAgentName = "AlarmAgent"
    PS D:\Workspace\IoTHub> $Env:AlarmMonitorAgentName = "AlarmMonitorAgent"
    PS D:\Workspace\IoTHub>
```

Create a base class for interacting with IoT hub. At this point, it just need to be able to connect and disconnect.

```
    # DeviceAgent.py

    from azure.iot.device.aio import IoTHubDeviceClient

    class DeviceAgent:
        def __init__(self, conn_str, agent_name, logging):
            self.agent_name = agent_name
            self.logging = logging

            self.device_client = IoTHubDeviceClient.create_from_connection_string(conn_str)

        async def connect(self):
            await self.device_client.connect()

        async def disconnect(self):
            await self.device_client.disconnect()

        def is_connected(self):
            return self.device_client.connected
```

Now create `AlarmAgent` which inherits from `DeviceAgent`

```
    # AlarmAgent.py

    from DeviceAgent import DeviceAgent

    class AlarmAgent(DeviceAgent):
        def __init__(self, conn_str, agent_name, destination_name, logging):
            super(AlarmAgent, self).__init__(conn_str, agent_name, logging)
            self.destination_name = destination_name
```

and `AlarmMonitorAgent`.

```
    # AlarmMonitorAgent.py

    from DeviceAgent import DeviceAgent

    class AlarmMonitorAgent(DeviceAgent):
        def __init__(self, conn_str, agent_name, logging):
            super(AlarmMonitorAgent, self).__init__(conn_str, agent_name, logging)
```

Now create the main application codes to interact with the IoT hub.

```
    # App.py

    import asyncio
    import os
    import logging
    import sys
    from time import gmtime

    from AlarmAgent import AlarmAgent
    from AlarmMonitorAgent import AlarmMonitorAgent

    REQUIRED_DEVICES = 2

    logging.basicConfig(filename='events.log', encoding='utf-8', format='%(asctime)s %(message)s', level=logging.INFO)
    logging.Formatter.converter = gmtime

    def all_devices_are_connected(devices):
        return len(devices) == REQUIRED_DEVICES

    async def disconnect_devices(devices):
        coroutines = []
        for device in devices:
            coroutines.append(device.disconnect())
        await asyncio.gather(*coroutines)

    async def main():

        alarm_agent = None
        alarm_monitor_agent = None

        connected_devices = []

        try:
            logging.info("App Started...")

            # Get neccessary application configs
            alarm_agent_conn_str = os.getenv("AlarmAgentConnectionString")
            alarm_monitor_agent_conn_str = os.getenv("AlarmMonitorAgentConnectionString")
            alarm_agent_name = os.getenv("AlarmAgentName")
            alarm_monitor_agent_name = os.getenv("AlarmMonitorAgentName")

            # Connect our devices to IoT hub
            alarm_agent = AlarmAgent(alarm_agent_conn_str, alarm_agent_name, alarm_monitor_agent_name, logging)
            alarm_monitor_agent = AlarmMonitorAgent(alarm_monitor_agent_conn_str, alarm_monitor_agent_name, logging)
            await asyncio.gather(alarm_agent.connect(), alarm_monitor_agent.connect())

            if alarm_agent.is_connected():
                connected_devices.append(alarm_agent)

            if alarm_monitor_agent.is_connected():
                connected_devices.append(alarm_monitor_agent)

            if all_devices_are_connected(connected_devices) is False:
                await disconnect_devices(connected_devices)

                # Can't do much if AlarmAgent and AlarmMonitorAgent are not both connect
                logging.error("Failed to connect necessary devices.")
                sys.exit()

            logging.info("All devices are connected.")
            await asyncio.sleep(5)

            # Nothing else to do, clean up as required.
            await disconnect_devices(connected_devices)

            logging.info("App Stopped!")
        except Exception:
            logging.error(Exception.with_traceback())

    if __name__ == "__main__":
        asyncio.get_event_loop().run_until_complete(main())
```

Ok let's quickly summarise the code above to get an idea of what it's doing:

* Read the neccessary app settings which was stored as environment variables earlier, i.e the devices name and their connection string.
* Instantiate the devices by passing in the required parameters to the respective classes.
* Connect the devices, the `asyncio.gather()` allows we to start the connections at the same time (well at least calls the `connect()` method asynchronously anyway).
* Once the `asyncio.gather()` completes, check that both devices are connected:
    + If both devices are not connected, then we can't do much so disconnect the ones that is connected and exit.
    + Otherwise, sleep for 5 seconds then disconnect and exit.

That is it, very simple conectivity test at this point. If everything went smoothly, we should see in the logs that both devices connected to the IoT hub successfully.

![Connecting to IoT hub](/assets/posts/2021-07-28-implementing-an-azure-iot-device-using-python/devices_connection_test_demo.gif)
_Connecting to IoT hub_

## Reporting a new alarm status

Here the focus is on preparing the data to be sent to the IoT hub.

Let's update the `AlarmAgent` so that it randomly simulate an alarm status and if the new status is different to the current status, report it to the `AlarmMonitorAgent`.

Changes to `DeviceAgent`

```
    # DeviceAgent.py

    ...

    from enum import Enum

    class DeviceAgentState(Enum):
        UNKNOWN = 1
        IDLE = 2
        NEW_EVENT_TO_PROCESS = 3
        PROCESSING_DEVICE_TO_CLOUD_STARTED = 4
        PROCESSING_DEVICE_TO_CLOUD_COMPLETED = 5

    class DeviceAgent:
        def __init__(self, conn_str, agent_name, logging):
            ...
            self.state = DeviceAgentState.IDLE

        ...

        async def send_parcel(self, parcel):
            await self.device_client.send_message(parcel)
```

Changes to `AlarmAgent`

```
    # AlarmAgent.py

    ...

    import random
    from datetime import datetime, timezone

    from DeviceAgent import DeviceAgentState
    from pb_AlarmStatus_pb2 import pb_AlarmStatus
    from pb_Parcel_pb2 import pb_Parcel

    class AlarmAgent(DeviceAgent):
        def __init__(self, conn_str, agent_name, destination_name, logging):
            ...
            self.current_alarm_status = False
            self.new_alarm_status = False
            self.parcel_to_send = None

        ...

        def simulate_event(self):
            return bool(random.getrandbits(1))

        def pack_alarm_event(self):
            self.logging.info("Packing event: %r" % self.current_alarm_status)
            date_time_utc = datetime.now(timezone.utc)
            alarm_status = pb_AlarmStatus()
            alarm_status.alarm_active = self.current_alarm_status
            alarm_status.time_utc = "{}".format(date_time_utc.time())
            alarm_status.date_utc = "{}".format(date_time_utc.date())

            self.logging.info("*************************************************************************")
            self.logging.info("Packing event: Active: %r, time: %r, date: %r" % (alarm_status.alarm_active, alarm_status.time_utc, alarm_status.date_utc))
            self.logging.info("*************************************************************************")

            parcel = pb_Parcel()
            parcel.source.name = "Door Alarm"
            parcel.source.local_id = "1"
            parcel.source.domain_agent = self.agent_name
            parcel.source.domain = "Device Domain"
            parcel.destination.name = "Alarm Monitor"
            parcel.type = "pb_AlarmStatus"
            parcel.content = str(alarm_status.SerializeToString(), 'utf-8')
            self.parcel_to_send = str(parcel.SerializeToString(), 'utf-8')

        async def do_work(self):
            if self.state is DeviceAgentState.IDLE or self.state is DeviceAgentState.PROCESSING_DEVICE_TO_CLOUD_COMPLETED:
                self.new_alarm_status = self.simulate_event()

                if self.new_alarm_status == self.current_alarm_status:
                    self.state = DeviceAgentState.IDLE
                else:
                    self.current_alarm_status = self.new_alarm_status
                    self.state = DeviceAgentState.NEW_EVENT_TO_PROCESS
                return

            if self.state is DeviceAgentState.NEW_EVENT_TO_PROCESS:
                self.pack_alarm_event()
                self.state = DeviceAgentState.PROCESSING_DEVICE_TO_CLOUD_STARTED
                return

            if self.state is DeviceAgentState.PROCESSING_DEVICE_TO_CLOUD_STARTED:
                await self.send_parcel(self.parcel_to_send)
                await asyncio.sleep(0.5) # Delayed for test
                self.parcel_to_send = None
                self.state = DeviceAgentState.PROCESSING_DEVICE_TO_CLOUD_COMPLETED
                return
```

Changes to `App`

```
    # App.py

    ...

    import time

    ...

    async def main():

        ...

        try:
            ...

            start_time = time.time()

            while True:
                await alarm_agent.do_work()

                # Let alarm agent runs for 5 sec
                if (time.time() - start_time) > 5:
                    break

            ...

            # Give alarm monitor some time before cleaning up connections.
            await asyncio.sleep(5)

    if __name__ == "__main__":
        asyncio.get_event_loop().run_until_complete(main())
```

The main changes in the codes above is the `AlarmAgent.do_work()` method. The `main()` method will continuously tell the alarm agent to do work for 5 seconds, and depending on it's state it performs a certain task. E.g:

* If in `IDLE`, `PROCESSING_DEVICE_TO_CLOUD_COMPLETED` state
    + No pending tasks to be done so simulate a new alarm status
    + If alarm status have changed, set state to `NEW_EVENT_TO_PROCESS`
* If in `NEW_EVENT_TO_PROCESS` state
    + Create a new `parcel` object to send.
    + The `parcel` object will have the source details (the device or service that initiated this parcel transaction).
    + The `parcel` object will have the destination details needed for the backend service (a collection of serverless functions I created already) to forward it to wherever it needs to go.
    + The `parcel` object will have details of the data content so that the recipient can process it.
    + Set state to `PROCESSING_DEVICE_TO_CLOUD_STARTED`
* If in `PROCESSING_DEVICE_TO_CLOUD_STARTED` state
    + Send parcel to IoT hub for processing
    + Set state to `PROCESSING_DEVICE_TO_CLOUD_COMPLETED`

## Processing an incoming alarm status report

Here the focus is on handling data from the IoT hub.

Let’s update the `AlarmMonitorAgent` so it can process incoming alarm status updates.

```
    # AlarmMonitorAgent.py

    ...
    from pb_AlarmStatus_pb2 import pb_AlarmStatus
    from pb_Parcel_pb2 import pb_Parcel
    from azure.iot.device import MethodResponse
    from google.protobuf import json_format

    class AlarmMonitorAgent(DeviceAgent):
        def __init__(self, conn_str, agent_name, logging):
            ...
            self.device_client.on_method_request_received = self.method_request_handler

        async def method_request_handler(self, method_request):
            status_code = 200
            payload = {"result": True, "data": "parcel handled"}

            if method_request.name != "ProcessMessage":
                status_code = 404
                payload = {"result": False, "data": "unknown method request"}

            parcel = json_format.ParseDict(method_request.payload, pb_Parcel(), True)

            if parcel is None or parcel.type != "pb_AlarmStatus":
                status_code = 400
                payload = {"result": False, "data": "bad parcel received"}
            else:
                alarm_status = pb_AlarmStatus()
                alarm_status.ParseFromString(bytes(parcel.content, 'utf-8'))
                self.logging.info("****************************************************************************")
                self.logging.info("Alarm status received from: %r" % (parcel.source.name))
                self.logging.info("alarm_active: %r, time_utc: %r, date_utc: %r" % (alarm_status.alarm_active, alarm_status.time_utc, alarm_status.date_utc))
                self.logging.info("****************************************************************************")

            method_response = MethodResponse.create_from_method_request(method_request, status_code, payload)
            await self.device_client.send_method_response(method_response)
```

When `AlarmMonitorAgent` is initialised, a direct method handler `method_request_handler` was registered. So when the IoT hub receives an alarm status updates, it will invoke this method.

One thing worth noting is that the payload is in JSON format, this is a [current limitation of Azure IoT Hub direct method](https://github.com/Azure/iotedge/issues/610), so we will need to convert this back to a protobuf message.

## A quick test run

Ensure our python environment is activate and run `python .\App.py`

![Run app demo](/assets/posts/2021-07-28-implementing-an-azure-iot-device-using-python/run_app_demo.gif)
_Run app demo_

and if we watch the file `events.log`, we can see what is sent to and from the IoT hub.

![Events.log demo](/assets/posts/2021-07-28-implementing-an-azure-iot-device-using-python/events_log_demo.gif)
_Events.log demo_

For examples,

```
    2022-02-14 12:44:14,182 *************************************************************************
    2022-02-14 12:44:14,182 Packing event: Active: True, time: '12:44:14.182005', date: '2022-02-14'
    2022-02-14 12:44:14,182 *************************************************************************
```

is sent from `AlarmAgent`, and

```
    2022-02-14 12:44:14,908 ****************************************************************************
    2022-02-14 12:44:14,908 Alarm status received from: 'Door Alarm'
    2022-02-14 12:44:14,908 alarm_active: True, time_utc: '12:44:14.182005', date_utc: '2022-02-14'
    2022-02-14 12:44:14,909 ****************************************************************************
```

is received by `AlarmMonitorAgent`.

If we have any questions or comment we can reach me on [Twitter](https://twitter.com/ChrisDinhNZ).