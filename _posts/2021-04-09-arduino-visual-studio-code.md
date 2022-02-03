---
layout: post
title: Arduino + Visual Studio Code
background: '/assets/posts/2021-04-09-arduino-visual-studio-code/post-banner-2021-04-09-arduino-visual-studio-code.jpg'
Published: 2021-04-09
Tag:
  - Arduino
  - VS Code
  - Tooling
  - C
---

# Arduino + Visual Studio Code

I use VS Code for a lot of my development work so it would be nice to be able to work in the same dev environment and work flow. With VS Code you have access to a rich set of built-in features and well as community-built extensions.

## What you will need:

* Arduino IDE installed
* Visual Studio Code installed
* Arduino Uno board
* Micro USB cable

To work with Arduino you will need to install [Visual Studio Code extension for Arduino](https://github.com/microsoft/vscode-arduino). This extension is a wrapper for the Arduino IDE. Note that this also has a dependency on the [C/C++ extension](https://github.com/Microsoft/vscode-cpptools) so that will also be installed.

Open VS Code and press Ctrl + Shift + X to open the Extensions panel and type vscode-arduino. Select the Arduino extension and click install.

![Installing Arduino extension](/assets/posts/2021-04-09-arduino-visual-studio-code/install_arduino_extension.png)
_Installing Arduino extension_

This extension provides several commands in the Command Palette which you can activate using `F1` or `Ctrl + Shift + P`) for working with `*.ino` files. You can filter the command palette just specific to this extension by typing arduino.

Under your current working directory, execute the `Arduino: Initialize` command from the command palette. This will allow you to select a name for your new sketch and the Arduino board you want to work with. VS Code will then setup the Arduino environment for you. Two configuration files will be generated which describes your Arduino environment: `arduino.json`, `c_cpp_properties.json`.

![Installing Arduino extension](/assets/posts/2021-04-09-arduino-visual-studio-code/new-arduino-environment-demo.gif)
_Arduino initialise_

Now let’s populate the sketch, it’s a basic countdown timer counting from 10 to 0 and prints it to the serial terminal. If the timer has stopped (either it has reached 0 or interrupted by the user) a ‘.’ will be printed. While the timer has stopped, it will restart again if interrupted by the user.

```
    #define COUNTDOWN_START_VALUE 10U

    static uint8_t currentCount = COUNTDOWN_START_VALUE;
    static uint16_t secondsSinceBoot = 0;
    static uint16_t nextPrintTime = 0;
 
    void setup()
    {
        Serial.begin(9600);
        Serial.println("Countdown Demo");
        Serial.println("--------------");
        Serial.println("");
    }
 
    void loop()
    {
        secondsSinceBoot = millis() / 1000;

        /* -1 if no incoming serial data. */
        if (Serial.read() != -1)
        {
            if (currentCount == 0)
            {
                currentCount = COUNTDOWN_START_VALUE;
                nextPrintTime = secondsSinceBoot + 1;
            }
            else
            {
                currentCount = 0;
                nextPrintTime = secondsSinceBoot;
            }
        }

        if (secondsSinceBoot == nextPrintTime)
        {
            if (currentCount == COUNTDOWN_START_VALUE)
            {
                Serial.println("Countdown Started");
            }
            else if (currentCount == 0)
            {
                Serial.println(".");
                nextPrintTime++;
                return;
            }

            Serial.println(currentCount, DEC);

            /* Countdown to 0 only. */
            if (currentCount != 0)
            {
                currentCount--;
            }

            nextPrintTime++;
        }
    }
```

Once the sketch is ready, select the COM port (if not done so already) and upload the sketch to the Arduino Uno board.

![Upload sketch to Arduino Uno board](/assets/posts/2021-04-09-arduino-visual-studio-code/upload-arduino-sketch-demo.gif)
_Upload sketch to Arduino Uno board_

Finally, let’s see it in action. Below I am using [Putty](https://www.putty.org/) to interact with Arduino board.

![Simple Arduino timer in action](/assets/posts/2021-04-09-arduino-visual-studio-code/countdown-in-putty-demo.gif)
_Simple Arduino timer in action_
