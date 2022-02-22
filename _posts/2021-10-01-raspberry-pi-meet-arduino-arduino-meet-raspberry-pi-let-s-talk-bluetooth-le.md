---
layout: post
title: Raspberry Pi, meet Arduino. Arduino, meet Raspberry Pi. Let’s talk Bluetooth (LE).
date: 2021-10-01
background: '/assets/posts/2021-10-01-raspberry-pi-meet-arduino-arduino-meet-raspberry-pi-let-s-talk-bluetooth-le/post-banner-2021-10-01-raspberry-pi-meet-arduino-arduino-meet-raspberry-pi-let-s-talk-bluetooth-le.jpg'
Tag:
    - Raspberry Pi
    - Arduino
    - Bluetooth
    - BLE
    - Python
    - C
---

Some months back I wrote a blog post about `Arduino Nano 33 IoT` and how to interact with it's onboard Bluetooth module. At that time I used `Nordic Semiconductor's` [nRF Connect](https://apps.apple.com/us/app/nrf-connect-for-mobile/id1054362403) tool from the App store to interact with Arduino. In this post I will build upon what I did and use a Raspberry Pi to interact with the Arduino.

There are quite a few good BLE helper libraries out there that can make my job easier. But not all are created equal, some may be perfect for building peripheral BLE applications but may not be the best for central BLE applications. As I am going to make the Raspberry Pi the client, [Bleak](https://github.com/hbldh/bleak) is a great one to use. Bleak is a GATT client software, capable of connecting to BLE devices acting as GATT servers. It is designed to provide a asynchronous, cross-platform Python API to connect and communicate with other BLE devices e.g. sensors.

The scenario is described by the sequence diagram below.

![High level sequence diagram](/assets/posts/2021-10-01-raspberry-pi-meet-arduino-arduino-meet-raspberry-pi-let-s-talk-bluetooth-le/ObservingSwitchToggleStateUsingBLE.png)

The above diagram consists of 3 main elements:

* A Raspberry Pi
* Arduino Nano 33 IoT
* A Switch

**The Raspberry Pi**

The Raspberry Pi is a single board computer (SBC) with an operating system, and because it is managed by an operating system you get the same benefits as you would on a PC such as multi-tasking, develop applications using higher languages such as Python and C#, hosting .NET applications and the list goes on. On the other hand, being managed by an operating system means access to lower level services and hardware resources are not as simple.

The Raspberry Pi in this case is acting as a BLE client, specifically searching for the Arduino device. Once found, it will connect to the Arduino and registered to be notified when the state of the switch changes (defined by a characteristic UUID).

**Arduino Nano 33 IoT**

Arduino Nano 33 IoT board is a microcontroller, has no operating system and host a single program written using a lower level language such as C/C++. This board makes it very easy to work with other electronic devices which is why I see the Raspberry Pi and Arduino board as complementary devices rather than competing devices. Which is, use the Raspberry Pi as a gateway between the local network and the internet, and use the Arduino board to talk to local sensors/actuators.

The Arduino Nano 33 IoT board in this case is acting as a BLE GATT server, consisting of a characteristic describing the current state of the switch. When the state of the switch changes, the Arduino board will notify any registered client of the new state.

**The Switch**

The switch is a physical device which controls a GPIO line on the Arduino. When it is pressed, the line will be set to high. When it is depressed, the line will be set to low.

## Setting up Python environment on the Raspberry Pi

Bleak is a Python library, so let’s set up a Python 3 virtual environment on the raspberry pi and install Bleak.

![Setting up Python virtual environment](/assets/posts/2021-10-01-raspberry-pi-meet-arduino-arduino-meet-raspberry-pi-let-s-talk-bluetooth-le/setup_virtual_env_on_pi_demo.gif)
_Setting up Python virtual environment on Raspberry Pi_

Now install Bleak.

```
    (venv) pi@raspberrypi:~/Workspace/ble_client $ pip install bleak
    Looking in indexes: https://pypi.org/simple, https://www.piwheels.org/simple
    Collecting bleak
    Using cached https://www.piwheels.org/simple/bleak/bleak-0.12.1-py2.py3-none-any.whl (123 kB)
    Collecting dbus-next
    Using cached https://www.piwheels.org/simple/dbus-next/dbus_next-0.2.3-py3-none-any.whl (57 kB)
    Installing collected packages: dbus-next, bleak
    Successfully installed bleak-0.12.1 dbus-next-0.2.3
    (venv) pi@raspberrypi:~/Workspace/ble_client $ pip list
    Package    Version
    ---------- -------
    bleak      0.12.1
    dbus-next  0.2.3
    pip        21.2.4
    setuptools 57.4.0
    wheel      0.37.0
    venv) pi@raspberrypi:~/Workspace/ble_client $
```

## Setting up the Arduino Nano 33 IoT board

Load the following Arduino sketch onto the Nano 33 IoT board. It is practically the same code as that in the [Beyond The Built-In LED](/2021/04/17/beyond-the-built-in-led.html) post I talked about.

```
    #include <ArduinoBLE.h>

    #define SWITCH_PIN 2u

    static bool isPressed;

    BLEService switchService("8158b2fd-94e4-4ff5-a99d-9a7980e998d7");
    BLEByteCharacteristic switchServiceCharacteristic("8158b2fe-94e4-4ff5-a99d-9a7980e998d7", BLERead | BLENotify);

    void setup()
    {
        Serial.begin(19200);
        pinMode(LED_BUILTIN, OUTPUT);
        pinMode(SWITCH_PIN, INPUT);
        isPressed = false;

        if (!BLE.begin())
        {
            /* Just keep looping until BLE module is up and running. */
            while (1);
        }

        /* BLE module is up and running, now add our service and characteristic to it. */
        BLE.setLocalName("My Arduino");

        switchServiceCharacteristic.writeValue(isPressed);
        switchService.addCharacteristic(switchServiceCharacteristic);
        BLE.addService(switchService);

        BLE.setAdvertisedService(switchService);
        BLE.advertise();
        Serial.println("My Arduino started");
    }

    void loop()
    {
        BLE.poll();

        if (digitalRead(SWITCH_PIN) == HIGH)
        {
            if (!isPressed)
            {
                digitalWrite(LED_BUILTIN, HIGH);
                isPressed = true;
                switchServiceCharacteristic.writeValue(isPressed);
                Serial.println("Switch is pressed");
            }
        }
        else
        {
            if (isPressed)
            {
                digitalWrite(LED_BUILTIN, LOW);
                isPressed = false;
                switchServiceCharacteristic.writeValue(isPressed);
                Serial.println("Switch is depressed");
            }
        }
    }
```

Now if you connect to the Arduino serial port using a terminal such as `Putty` for example, you should be able to see some console logging when the switch is pressed and depressed.

![Console log switch activity](/assets/posts/2021-10-01-raspberry-pi-meet-arduino-arduino-meet-raspberry-pi-let-s-talk-bluetooth-le/switch_pressed_depressed_demo.gif)
_Console log switch activity_

## Let’s talk BLE

In the sketch above, I gave the Arduino board a Bluetooth device name `My Arduino`. I then added a service with one characteristic and also state that this characteristic can be `read` or `be notified` when the value changes.

Just as a side note, because the raspberry pi initiated the Bluetooth connection, it is considered as a BLE central device (a client). The Arduino board on the other hand accepts connection requests and is providing a service, it is considered as a BLE peripheral device (a server).

Let’s add some Python code to the raspberry pi to start receiving BLE updates from out Arduino board.

```
    import asyncio
    import sys

    from bleak import BleakScanner
    from bleak.backends.bluezdbus.client import BleakClientBlueZDBus

    device_name = "My Arduino"
    switch_status_char_uuid = "8158b2fe-94e4-4ff5-a99d-9a7980e998d7"

    def notification_handler(sender, data):
        print("Switch is active: {}".format(bool(data[0])))

    async def run():
        client = None
        device = await BleakScanner.find_device_by_filter(
            lambda d, ad: d.name and d.name.lower() == device_name.lower()
        )

        if device is None:
            print("{} not found".format(device_name))
            sys.exit()
        else:
            print("{} found".format(device))

        client = BleakClientBlueZDBus(device)

        while True:
            if not client.is_connected:
                try:
                    if await client.connect():
                        print("Connected to {}".format(device_name))
                        await client.start_notify(switch_status_char_uuid, notification_handler)
                except:
                    print("Connected to {} failed or lost".format(device_name))
                    await asyncio.sleep(1)
                    client = BleakClientBlueZDBus(device)
                    print("Retrying...")
            else:
                await asyncio.sleep(1)

    loop = asyncio.get_event_loop()
    loop.run_until_complete(run())
```

Basically what the above python code does is scan for a specific BLE device with the name of `My Arduino`. If one was in range it will connect to it. The program then start observing the switch status characteristic and when an update is received, it will be handled by notification_handler().

## Let’s see them in action

Below is a short clip showing:

* Arduino Nano 33 IoT powered by a portable USB charger
* A switch connected to the Arduino
* Phone connected to Raspberry Pi via SSH (Using `Termius`)

<iframe width="560" height="315" src="https://www.youtube.com/embed/WLf3vILkqXM" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture">&nbsp;</iframe>

As you can see, when the switch is pressed or depressed, we can see that the phone received a BLE notification.

That is basically it, it doesn't have to be a switch. You can connect whatever sensor you like (analog or digital) and add the necessary BLE services to our Arduino device as required.
