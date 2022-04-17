---
layout: post
title: An Introduction To IR Modules Using An Arduino Uno
date: 2022-04-17 23:38 +1300
background: '/assets/posts/2022-04-17-an-introduction-to-ir-modules/post-banner-2022-04-17-an-introduction-to-ir-modules.jpg'
Tag:
  - Arduino
  - Arduino Uno
  - IR Receiver
  - IR Transmitter
  - C
---

Introducing my old Kenwood KR-V7080 Receiver rack system, well minus the double tape deck player and the turntable which didn't quite survived the test of time. This stereo system has been with me for almost 25 years I think. But over the years as I settled down, it more or less took a backseat. And as the family grew, I eventually moved it into the garage where it sat for the last few years collecting dust.

![My old Kenwood Rack System](/assets/posts/2022-04-17-an-introduction-to-ir-modules/kenwood_stereo_collecting_dust.jpg)

However, we are going to breath new life into this rack system again. This blog post is going to be the first in a series of posts where we will try to connect it to the internet and maybe control it via some app. Ok I know, many of you will ask why? Well because we can (just kidding 😉, but seriously though, we are doing it because I think we will learn so much for doing it).

As some of you might have guessed from this post's title already, we are going to tinker a little bit with `IR communication`. We are going to simulate the Kenwood receiver's remote, so that whatever we can do with the remote, we should be able to do with an Arduino.

## Infrared

Below is a diagram of the electromagnetic spectrum.

![Electromagnetic Spectrum Diagram](/assets/posts/2022-04-17-an-introduction-to-ir-modules/electromagnetic_spectrum.png)

The diagram shows the types of electromagnetic waves and it's real-world application along the x-axis with respect to the waves energy (frequency) from left to right. If we look at the rainbow in the middle of the diagram, that's the `visible spectrum` and is what our eyes can see. And if we think back to our high school physics, `red` sits at the bottom of the visible spectrum (based on the wave's energy) while `blue` sits at the top of the visible spectrum. To the left of the visible spectrum we have the `infrared` spectrum (which has lower energy than red, hence the name `infra` `red`). If you want to learn more about waves and how they relates to colours and light, definitely check out this article [Waves and Energy](https://www.sciencelearn.org.nz/resources/2681-waves-and-energy-energy-transfer) and also the concept of [Black Body Radiation](https://en.wikipedia.org/wiki/Black-body_radiation#Black_body).

## Infrared Communication

Infrared (IR) communication is a form of wireless communication between two devices using infrared light. It is widely used in consumer electronics where the equipment is controlled via a remote control such as:

* TVs
* Stereos
* Heatpumps
* Medical devices

These are often referred to as `consumer infrared` ([CIR](https://en.wikipedia.org/wiki/Consumer_IR)).

Infrared communication is a very common and inexpensive communication technology. The communication consists of a transmitting device which sends data by emitting bursts of infrared light via an infrared LED. As infrared is outside the visible spectrum, these bursts of light will not be detectable by the human eye. On the receiving device, a photodiode is used to detect these bursts of light and convert them into an electrical current.

## Receiving Data Using An IR Receiver Module

In this section we are going to look at using an IR receiver module in combination with an Arduino Uno to capture the signals emitting from the Kenwood KR-V7080 remote. The goal is to map all of the buttons on the remote to a list of key-value pairs.

The IR receiver module we will be using comes with a breakout board as shown below:

![IR Receiver Module](/assets/posts/2022-04-17-an-introduction-to-ir-modules/ir_receiver_module.jpg)

Next we will hook up it up to an Arduino Uno. How they are wired together is as follow:

![Hooking IR Receiver Module To Arduino](/assets/posts/2022-04-17-an-introduction-to-ir-modules/ir_receiver_module_arduino_uno_wiring.jpg)

Actual wiring:

![Actual IR Receiver Module To Arduino Wiring](/assets/posts/2022-04-17-an-introduction-to-ir-modules/ir_receiver_module_arduino_uno_actual_wiring.jpg)

Now let's head over to `VS Code` and initialise a new `Arduino sketch` to interact with the IR receiver module, we will need to give the sketch a name so let's go with IrReceiverModuleDemo.ino. If you are not familiar with working with Arduino in VS Code, have a look at my blog post [Arduino + Visual Studio Code](/2021/04/09/arduino-visual-studio-code.html) to get you started.

To help us decode the IR signals, we will be using an open source library called [IRremote](https://github.com/Arduino-IRremote/Arduino-IRremote). As of writing, the latest version and the one we are using is version 3.6.2. The sketch is implemented as follow:

```
   /*
    * IrReceiverModuleDemo: Simple sketch to interact with the IR receiver module to
    * intercept IR signals.
    *
    * Copyright Chris Dinh 2022
    */

   #include <IRremote.h>

   #define RECV_PIN 11 // IR receiver signal pin

   void setup()
   {
      Serial.begin(115200);
      IrReceiver.begin(RECV_PIN, ENABLE_LED_FEEDBACK);
   }

   void loop()
   {
      if (IrReceiver.decode())
      {
         Serial.println(IrReceiver.decodedIRData.decodedRawData, HEX);
         IrReceiver.resume();
      }
   }
```

This is the remote control that comes with Kenwood KR-V7080 receiver (as I have mentioned, it's been sitting in the garage for the last few years collecting dust):

![Kenwood Remote](/assets/posts/2022-04-17-an-introduction-to-ir-modules/kenwood_stereo_remote.jpg)

We are going to point this remote at the IR receiver module and go through and presses each button on the remote. The idea is to decode the Hexadecimal value being transmitted for each button so that later on we can simulate these button presses by sending these Hexadecimal values using an IR transmitter module. Below is a demo of what was being decoded as each numeric button was pressed (1 - 9, and lastly 0).

![IR Decode Demo](/assets/posts/2022-04-17-an-introduction-to-ir-modules/ir_decode_demo.gif)

**Remote Control Unit (RC-R0803) Code Summary**

So I have been tinkering with the `RC-R0803 Remote` and the `KR-V7080 Receiver` and there were a couple of things I have learnt:

* The RC-R0803 Remote is a universal remote controller, what this mean is that depending what device we are trying to control at the time the button that is pressed may send a different command (e.g. the demo above shows the codes being transmitted when `Video 1` was selected on the remote).
* Some buttons depend on the context the remote control is in, if the remote is not in the appropriate context, pressing a button may not send any command at all.
* The RC-R0803 Remote is using the `NEC` IR protocol to transmit the infrared commands. If you want to dig a bit deeper into the NEC IR protocol, check out [NEC IR Remote Control Interface with 8051](https://exploreembedded.com/wiki/NEC_IR_Remote_Control_Interface_with_8051).

Given we are only interested in controlling the KR-V7080 Receiver, the following table describes how the RC-R0803 Remote interacts with the KR-V7080 Receiver.

```
   Button         IR Code        Notes
   ------         -------        -----
   Power          629D47B8       Toggles power on/off.

   0              7F8047B8       Numbers can be used to select radio presets.
   1              7E8147B8
   2              7D8247B8
   3              7C8347B8
   4              7B8447B8
   5              7A8547B8
   6              798647B8
   7              788747B8
   8              778847B8
   9              768947B8
   +10            F20D47B8       Select radio presets 10 or greater.

   Volume Up      649B47B8
   Volume Down    659A47B8
   Mute           639C47B8       Toggle mute.
   Input          33CC01B8       Iterate through input types (Radio, Phono, CD, Video2, Video1, Tape1).
   Stereo         28D747B8       Stereo mode.
   Surround Mode  20DF47B8       Iterate through surround mode (Dolby Pro Logic, Dolby 3 Stereo, DSP Logic, DSP Arena).

   Audio          639C02B8       Iterate through audio settings (Bass, Trebble, Sub Woofer, Balance).
   Surround       629D02B8       Iterate through surround settings (Rear, Delay, Center).
   Up             AA5501B8       Used to control audio/surround settings.
   Down           AB5401B8       Used to control audio/surround settings.
```

## Sending Data Using An IR Transmitter Module

In this section we are going to look at using an IR transmitter module in combination with an Arduino Uno to send IR coded signals to the KR-V7080 Receiver. The goal is to simulate the RC-R0803 Remote using the table above.

The IR transmitter module we will be using comes with a breakout board as shown below:

![IR Transmitter Module](/assets/posts/2022-04-17-an-introduction-to-ir-modules/ir_transmitter_module.jpg)

Next we will hook up it up to an Arduino Uno. How they are wired together is as follow:

![Hooking IR Transmitter Module To Arduino](/assets/posts/2022-04-17-an-introduction-to-ir-modules/ir_transmitter_module_arduino_uno_wiring.jpg)

Actual wiring:

![Actual IR Transmitter Module To Arduino Wiring](/assets/posts/2022-04-17-an-introduction-to-ir-modules/ir_transmitter_module_arduino_uno_actual_wiring.jpg)

Again let's head over to `VS Code` and initialise a new `Arduino sketch` to interact with the IR transmitter module, we will need to give the sketch a name so let's go with IrTransmitterModuleDemo.ino.

So instead of using the remote, the plan is to have the Arduino Uno listens for IR commands over the serial connection and forward it to the stereo receiver using the IR transmitter module.

```
   /*
    * IrTransmitterModuleDemo_v2: Simple sketch to interact with the IR transmitter module to
    * send IR signals.
    *
    * Copyright Chris Dinh 2022
    */

   /* INCLUDES */
   #include "Arduino.h"
   #include <IRremote.h>
   #include <string.h>

   /* DEFINITIONS */
   #define COMMAND_STRING_MAX_LENGTH 15U

   #define COMMAND_VALUE_MASK 0xFF0000
   #define COMMAND_VALUE_OFFSET 16U
   #define ADDRESS_VALUE_MASK 0xFFFF

   #define LINE_FEED_VALUE 10U
   #define CARRIAGE_RETURN_VALUE 13U

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
      IrSender.begin(IR_SEND_PIN, true, LED_BUILTIN);
      Serial.begin(115200); // opens serial port
      Serial.println("Begin IrTransmitterModuleDemo_v2");
   }

   void loop()
   {
      /* If there are no data to read, do nothing */
      if (!Serial.available())
      {
         return;
      }

      char c = Serial.read();

      if (c == LINE_FEED_VALUE || c == CARRIAGE_RETURN_VALUE)
      {
         /* When receiving data over the serial connection, we will treat LF and CR as command termination. */
         processCommandString(commandString);
         ResetCommandString();
      }
      else if (commandStringLength > COMMAND_STRING_MAX_LENGTH)
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

Ok let's summarise the sketch above.

In the `setup()` we initialises a couple of things:

* Clear the command buffer
* Set which GPIO pin the IR transmit module's signal line is connected to
* Use the built-in LED to provide some visual feedback when transmitting
* Set serial baud rate to listen to

In the `loop()` we continuously listen to the serial port for incoming data:

* Whatever comes through will be added to a buffer
* A line feed and carriage return is use to indicate full command has been received and command can be processed
* Commands assumed to be no more than 15 characters long and it is exceeded, the buffer will be cleared and we start over
* The commands are mapped to IR code which will then be transmitted

Below is a demo of the above sketch in action. In the foreground we have the Arduino Uno connected to a laptop via a USB serial cable, while in the background we have the Kenwood stereo receiver. As you can see, when we issue some commands on the laptop, these get sent to the Arduino Uno which are then forwarded to the receiver via the IR transmit module.

<iframe width="560" height="315" src="https://www.youtube.com/embed/wjqbKmODQoo" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture">&nbsp;</iframe>
