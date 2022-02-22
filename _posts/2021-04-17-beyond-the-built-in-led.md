---
layout: post
title: Beyond The Built-In LED
date: 2021-04-17
background: '/assets/posts/2021-04-17-beyond-the-built-in-led/post-banner-2021-04-17-beyond-the-built-in-led.jpg'
Tag:
  - Arduino
  - Arduino Nano
  - BLE
  - C
---

The built-in LED example that comes with the Arduino IDE is a great way to familiarise with the Arduino platform. But the Arduino can do so much more than that, it's a doorway to the world of IoT. This blog post will describe the process of connecting a switch to an Arduino Nano 33 IoT board (e.g a BLE doorbell solution).

![High level overview](/assets/posts/2021-04-17-beyond-the-built-in-led/doorbell-high-level-view.png)
_High level overview_

Above is a high level diagram of the components that will be interacting with one another. The doorbell switch is connected to a digital input of the Arduino Nano 33 IoT board so that BLE clients (e.g. nRF Connect) can get notified of the doorbell events.

## What you will need:

* VS Code with Arduino extension installed
* Arduino Nano 33 IoT
* Micro USB cable
* Breadboard
* Jumper wires male to male x 4
* 10K resistor
* Push switch

![Arduino Nano 33 IoT pin layout](/assets/posts/2021-04-17-beyond-the-built-in-led/nano_33_iot_pinout.png)
_Arduino Nano 33 IoT pin layout_

![Circuit’s physical connection and wiring](/assets/posts/2021-04-17-beyond-the-built-in-led/doorbell-wiring-view.png)
_Circuit’s physical connection and wiring_

The above diagram shows 3 main components:

* Arduino Nano 33 IoT board
  + Provides the 3.3V to drive the digital input high when the doorbell is pressed
  + Provides a digital input pin to detect when the doorbell is pressed
* Doorbell (push button switch)
  + Creates an open circuit when not pressed (pin state = LOW)
  + Creates a close circuit when pressed (pin state = HIGH)
* 10k resister
  + Acts as a pull-down resister to avoid false triggers.
  + Ensures digital pin 2 is logically 0 when the doorbell is not pressed
  + Ensures digital pin 2 is logically 1 when the doorbell is pressed

Given the above setup what you want want to do at this stage is to make sure the circuit works:

* When the doorbell is pressed, the built-in LED will be on
* When the doorbell is released, the built-in LED will be off

So the sketch needed to load onto the Arduino is as follow:

```
    #define DOORBELL_PIN 2u

    static bool isPressed;

    void setup()
    {
        pinMode(LED_BUILTIN, OUTPUT);
        pinMode(DOORBELL_PIN, INPUT);
        isPressed = false;
    }

    void loop()
    {
        if (digitalRead(DOORBELL_PIN) == HIGH)
        {
            if (!isPressed)
            {
                digitalWrite(LED_BUILTIN, HIGH);
                isPressed = true;
            }
        }
        else
        {
            if (isPressed)
            {
                digitalWrite(LED_BUILTIN, LOW);
                isPressed = false;
            }
        }
    }
```

![LED toggles reflecting the state of switch](/assets/posts/2021-04-17-beyond-the-built-in-led/button-switch-demo.gif)

_LED toggles reflecting the state of switch_

## Setting up the Bluetooth service

To interact with the BLE module on the Nano, the ArduinoBLE library is a great option and is easy to use.

![Installing ArduinoBLE library via VS Code](/assets/posts/2021-04-17-beyond-the-built-in-led/install-arduinoble-demo.gif)
_Installing ArduinoBLE library via VS Code_

Now update the Arduino sketch to include the doorbell BLE service.

```
    #include <ArduinoBLE.h>

    #define DOORBELL_PIN 2u

    static bool isPressed;

    BLEService doorBellService("8158b2fd-94e4-4ff5-a99d-9a7980e998d7");
    BLEByteCharacteristic doorBellCharacteristic("8158b2fe-94e4-4ff5-a99d-9a7980e998d7", BLERead | BLENotify);

    void setup()
    {
        pinMode(LED_BUILTIN, OUTPUT);
        pinMode(DOORBELL_PIN, INPUT);
        isPressed = false;

        if (!BLE.begin())
        {
            /* Just keep looping until BLE module is up and running. */
            while (1);
        }

        /* BLE module is up and running, now add our service and characteristic to it. */
        BLE.setLocalName("Awesome Doorbell");

        doorBellCharacteristic.writeValue(isPressed);
        doorBellService.addCharacteristic(doorBellCharacteristic);
        BLE.addService(doorBellService);

        BLE.setAdvertisedService(doorBellService);
        BLE.advertise();
    }

    void loop()
    {
        BLE.poll();

        if (digitalRead(DOORBELL_PIN) == HIGH)
        {
            if (!isPressed)
            {
                digitalWrite(LED_BUILTIN, HIGH);
                isPressed = true;
                doorBellCharacteristic.writeValue(isPressed);
            }
        }
        else
        {
            if (isPressed)
            {
                digitalWrite(LED_BUILTIN, LOW);
                isPressed = false;
                doorBellCharacteristic.writeValue(isPressed);
            }
        }
    }
```

Download the sketch to the arduino board and it should be ready to go.

At this point you should be able to use a BLE Client (e.g. nRF Connect) to connect to the `Awesome Doorbell`.

![Awesome Doorbell as shown on nRF Connect](/assets/posts/2021-04-17-beyond-the-built-in-led/nrf-doorbell-connect.png)
_Awesome Doorbell as shown on nRF Connect_

After connecting to the doorbell, you can dig into the characteristic details and see that when:

* Doorbell is not pressed, value is 0x00
* Doorbell is pressed, value is 0x01

![Awesome Doorbell status update](/assets/posts/2021-04-17-beyond-the-built-in-led/nrf-doorbell-state.png)
_Awesome Doorbell status update_

There you have it, the ArduinoBLE library makes it really easy to get Bluetooth up and running on Arduino devices.
