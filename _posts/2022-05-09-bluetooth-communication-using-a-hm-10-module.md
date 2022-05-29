---
layout: post
title: Bluetooth Communication Using A HM-10 Module
date: 2022-05-09 21:32 +1200
background: '/assets/posts/2022-05-09-bluetooth-communication-using-a-hm-10-module/post-banner-2022-05-09-bluetooth-communication-using-a-hm-10-module.jpg'
Tag:
  - Arduino
  - Arduino Uno
  - Bluetooth
  - BLE
  - HM-10
---

Previously I hooked up an IR Tx module to an Arduino Uno in an attempt to control my old stereo (yes, the IoT odyssey continues). Unfortunately, the Arduino Uno doesn't have any onboard wireless interface so if I want to send commands to my stereo, I would have to physically hook up my laptop to my Arduino Uno.

Fortunately, I have a spare HM-10 module sitting around doing nothing. So in this blog post, I am going to be hooking that up to the Arduino Uno so that we have some wireless connection capability.

## The Hardware Side Of Things

The HM-10 is a Bluetooth 4.0 BLE module. `Martyn Currey` wrote a great post which really goes into the nitty gritty details of this module. I definitely learnt a lot from that post so I felt he deserves a shout out. If you are interested, check it out at [HM-10 Bluetooth 4 BLE Modules](http://www.martyncurrey.com/hm-10-bluetooth-4ble-modules/).

The HM-10 are quite readily available, the one I got was manufactured by DSD Tech and it's datasheet can be found on their page [DSD TECH Bluetooth 4.0 ble module HM-10 datasheet](http://www.dsdtech-global.com/2017/08/hm-10.html).

My HM-10 module comes with a breakout board as shown below:

![HM-10 Module](/assets/posts/2022-05-09-bluetooth-communication-using-a-hm-10-module/hm-10_module.jpg)

The wiring details between the HM-10 module and the Arduino Uno are as follow:

![Arduino Uno and HM-10 Module Wiring Diagram](/assets/posts/2022-05-09-bluetooth-communication-using-a-hm-10-module/uno-and-hm-10-connection-and-wiring.png)

Note that the Arduino Uno's digital pin output is 5V when high while the HM-10's RxD pin takes 3.3V. So we used a voltage divider to bring the 5V down to 3.3V before feeding it to the RxD pin.

Actual wiring (including IR Tx module):

![Arduino Uno and HM-10 Module Actual](/assets/posts/2022-05-09-bluetooth-communication-using-a-hm-10-module/uno-and-hm-10-actual-wiring.jpg)

## The Software Side Of Things

Now let’s head over to VS Code and initialise a new Arduino sketch to interact with the HM-10 module. The sketch will rely on a serial port emulator library called `AltSoftSerial` which can be installed via the `Arduino Library Manager`.

![Installing AltSoftSerial](/assets/posts/2022-05-09-bluetooth-communication-using-a-hm-10-module/alt_soft_serial_installed.png)

Before trying to integrate the HM-10 with the IR Tx module, we are just going to load a simple sketch so that we know sending and receiving serial data between the HM-10 and the Arduino Uno works.

```
   /*
   * HM10Demo: Simple sketch to interact with the HM-10 module.
   *
   * Copyright Chris Dinh 2022
   */

   /* INCLUDES */
   #include <AltSoftSerial.h>

   static AltSoftSerial btSerial;

   /* There is data waiting to be read from the HM-10 device. */
   static void HandleRxDataIndication(void)
   {
      char c = btSerial.read();

      /* Just echo the character for now. */
      Serial.write(c);
   }

   /* There is data waiting to be sent to the HM-10 device. */
   static void HandleTxDataIndication(void)
   {
      char c = Serial.read();

      /* Echo the character just been sent. */
      Serial.write(c);

      /* We don't send carriage return or line feed. */
      <!-- if (c == 0x0A || c == 0x0D)
      {
         return;
      } -->

      btSerial.write(c);
   }

   void setup()
   {
      Serial.begin(9600);
      btSerial.begin(9600);
      Serial.println("BTserial started at 9600");
   }

   void loop()
   {
      if (btSerial.available())
      {
         HandleRxDataIndication();
      }

      if (Serial.available())
      {
         HandleTxDataIndication();
      }
   }
```

While the HM-10 module is in discovery mode (not connected to another device), we can configure it by sending AT commands over the serial connection (e.g. using the serial monitor). The example below shows the HM-10 module recieved a command to return it's current device name (i.e. `My HM-10`) before being set to `RC-R0803`.

![AT Command Demo](/assets/posts/2022-05-09-bluetooth-communication-using-a-hm-10-module/hm10_at_command_demo.gif)
_In the demo above I later realised the command to set name had a typo. The set name command should not include the '?' in it._

When the HM-10 is in connected mode, it can act as a data communication channel between the Arduino Uno and the connected Bluetooth device. This is done via a custom BLE characteristic provided by the HM-10 module. For example, using a BLE tool such as [nRF Connect for iOS](https://apps.apple.com/us/app/nrf-connect-for-mobile/id1054362403) we can easily connect to the HM-10 module from an iPhone and send data to it.

![HM-10 Module In Discovery Mode](/assets/posts/2022-05-09-bluetooth-communication-using-a-hm-10-module/rc-r0803-discovery-mode.jpg)

Using nRF Connect to scan for BLE devicces should show the HM-10 module to show up as `RC-R0803` just as we named it previously.

After connecting to it, nRF Connect will provide a couple of menu options to either get more details about the HM-10 module or to interact with it. One of these is the `client` menu option. This menu option provides access to the custom characteristic (`UUID: FFE1`) where we can:

* Query it's current value
* Give it a new value
* Request to be notified if this value changes

![HM-10 Module In Connected Mode](/assets/posts/2022-05-09-bluetooth-communication-using-a-hm-10-module/rc-r0803-connected-mode.jpg)

We can update the custom characteristic value by selecting the `up` arrow and give it a new value (up to 20 characters long).

At this point I think we can update the sketch so that we can integrate the HM-10 with the IR Tx module.

```
   /*
   * KenwoodRC_R0803_Remote: A Kenwood RC-R0803 Remote simulator.
   *
   * Copyright Chris Dinh 2022
   */

   /* INCLUDES */
   #include <IRremote.h>
   #include <string.h>
   #include <AltSoftSerial.h>

   /* DEFINITIONS */

   #define COMMAND_STRING_MAX_LENGTH 19U

   #define COMMAND_VALUE_MASK 0xFF0000
   #define COMMAND_VALUE_OFFSET 16U
   #define ADDRESS_VALUE_MASK 0xFFFF

   #define COMMAND_END_MARKER_VALUE 35U // Character '#'

   #define IR_SEND_PIN 13U

   /* Remote Control R0803 Button Code */
   #define BUTTON_CODE_POWER_TOGGLE    0x629D47B8  // Toggles power on/off
   #define BUTTON_CODE_0               0x7F8047B8  // Numbers can be used to select radio presets
   #define BUTTON_CODE_1               0x7E8147B8
   #define BUTTON_CODE_2               0x7D8247B8
   #define BUTTON_CODE_3               0x7C8347B8
   #define BUTTON_CODE_4               0x7B8447B8
   #define BUTTON_CODE_5               0x7A8547B8
   #define BUTTON_CODE_6               0x798647B8
   #define BUTTON_CODE_7               0x788747B8
   #define BUTTON_CODE_8               0x778847B8
   #define BUTTON_CODE_9               0x768947B8
   #define BUTTON_CODE_10X             0xF20D47B8  // Selects radio presets 10 or greater
   #define BUTTON_CODE_VOLUME_UP       0x649B47B8
   #define BUTTON_CODE_VOLUME_DOWN     0x659A47B8
   #define BUTTON_CODE_MUTE            0x639C47B8  // Toggle mute
   #define BUTTON_CODE_INPUT           0x33CC01B8  // Iterate through input types (Radio, Phono, CD, Video2, Video1, Tape1)
   #define BUTTON_CODE_STEREO          0x28D747B8  // Stereo mode
   #define BUTTON_CODE_SURROUND_MODE   0x20DF47B8  // Iterate through surround mode (Dolby Pro Logic, Dolby 3 Stereo, DSP Logic, DSP Arena)
   #define BUTTON_CODE_AUDIO           0x639C02B8  // Iterate through audio settings (Bass, Trebble, Sub Woofer, Balance)
   #define BUTTON_CODE_SURROUND        0x629D02B8  // Iterate through surround settings (Rear, Delay, Center)
   #define BUTTON_CODE_UP              0xAA5501B8  // Used to control audio/surround settings
   #define BUTTON_CODE_DOWN            0xAB5401B8  // Used to control audio/surround settings

   /* VARIABLES */
   static AltSoftSerial btSerial;
   static char commandString[COMMAND_STRING_MAX_LENGTH];
   static uint8_t commandStringLength;

   /* PRIVATE FUNCTIONS */
   static void ResetCommandString(void)
   {
      memset(commandString, 0, sizeof(commandString[0]) * COMMAND_STRING_MAX_LENGTH);
      commandStringLength = 0;
   }

   static void sendNecCommand(uint32_t buttonCode, uint8_t repeats)
   {
      uint16_t address = (buttonCode & ADDRESS_VALUE_MASK);
      uint8_t command = (buttonCode & COMMAND_VALUE_MASK) >> COMMAND_VALUE_OFFSET;

      IrSender.sendNEC(address, command, repeats);
      Serial.println("Command sent");
   }

   static void processCommandString(const char *command)
   {
      if (strcmp(command, "power") == 0)
      {
         sendNecCommand(BUTTON_CODE_POWER_TOGGLE, 0);
         return;
      }

      if (strcmp(command, "vol+") == 0)
      {
         /* Must be in volume context already to change volume. Otherwise the first
            "vol+" will only put receiver in volume context. */
         sendNecCommand(BUTTON_CODE_VOLUME_UP, 0);
         return;
      }

      if (strcmp(command, "vol-") == 0)
      {
         /* Must be in volume context already to change volume. Otherwise the first
            "vol-" will only put receiver in volume context. */
         sendNecCommand(BUTTON_CODE_VOLUME_DOWN, 0);
         return;
      }

      if (strcmp(command, "mute") == 0)
      {
         sendNecCommand(BUTTON_CODE_MUTE, 0);
         return;
      }
   }

   /* PUBLIC FUNCTIONS */
   void setup()
   {
      ResetCommandString();
      Serial.begin(9600);
      IrSender.begin(IR_SEND_PIN, true, LED_BUILTIN);
      btSerial.begin(9600);
      Serial.println("Begin KenwoodRC_R0803_Remote");
   }

   void loop()
   {
      /* If there are no data to read, do nothing */
      if (!btSerial.available())
      {
         return;
      }

      char c = btSerial.read();
      Serial.write(c);

      if (c == COMMAND_END_MARKER_VALUE)
      {
         /* When receiving data over the Bluetooth connection, we will treat '#' as command termination. */
         processCommandString(commandString);
         ResetCommandString();
      }
      else if (commandStringLength == COMMAND_STRING_MAX_LENGTH)
      {
         /* Command length exceeded, start again. */
         ResetCommandString();
      }
      else
      {
         /* Carry on retrieving current command. */
         commandString[commandStringLength] = c;
         commandStringLength++;
      }
   }
```

The above Arduino sketch basically amalgamate what was done in the previous blog post with what is done in this blog post so that we can communicate with the stereo using a Bluetooth BLE app. Let's run through how the code works.

Like any Arduino sketch, the above sketch has two parts, a `setup` part and a `do work` part.

* void setup()
   + This is the `setup` part which is executed on startup.
   + Reset the command string. The command string is the input text representing the user's command to send to the stereo.
   + Initialise the built-in Serial instance (this is for debugging and configuration purposes only and not used in normal operating mode).
   + Initialise the IrSender instance for communicating with the IR Tx module.
   + Initialise the AltSoftSerial instance for communicating with the HM-10 module.
* void loop()
   + This is the `do work` part and is an infinite loop that executes the code within it.
   + The Arduino continuously check if the HM-10 module received any data and if so, will add to the command string.
   + The command string has two attributes, it cannot be longer than what is defined by COMMAND_STRING_MAX_LENGTH and must be terminated by a `#` character.
   + If the command string length exceeds COMMAND_STRING_MAX_LENGTH and the string terminator have not been received, the command string will be reset.
   + If the string terminator is received before the command string exceeds COMMAND_STRING_MAX_LENGTH, the command string will be processed.
   + If the command string can be mapped to a defined NEC command, it will be sent over the IR Tx module.

Below is a demo of the above sketch in action. What we have is an Arduino Uno, a mobile phone and the Kenwood stereo. The mobile phone communicates with the Arduino Uno via Bluetooth, while the Arduino Uno communicates with the Kenwood stereo via infrared. Using a Bluetooth utility tool such as nRF Connect we can toggle the the stereo on and off.

<iframe width="840" height="472" src="https://www.youtube.com/embed/BQDJ6dgicdw" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture">&nbsp;</iframe>