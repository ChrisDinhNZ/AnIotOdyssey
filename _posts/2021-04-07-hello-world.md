---
layout: post
title: Hello World
background: '/assets/posts/2021-04-07-hello-world/post-banner-2021-04-07-hello-world.jpg'
Published: 2021-04-07
Tag:
  - Arduino
  - Arduino Uno
  - Blink LED
---

# Hello World!

Let’s begin the journey with a simple blinking LED program, it’s a simple Arduino [sketch](https://www.arduino.cc/en/pmwiki.php?n=Tutorial/Sketch) which could be considered as the “Hello World!” of Internet of Things. While simple the “Hello World!” program is very important in that it’s a way of verifying the developing environment is set up correctly.

## What you will need:

* Arduino Uno board
* Arduino IDE installed
* Micro USB cable

![Arduino Uno Board](/assets/posts/2021-04-07-hello-world/arduino-uno-board.jpg)
_Arduino Uno Board_

The Arduino IDE comes pre-loaded with a vast selection of sketches to get things moving, and so there is already a sketch to do exactly what you want and it is called Blink. It can be found under the Basics Built-In Examples.

![Loading prloaded sketch](/assets/posts/2021-04-07-hello-world/using-pre-loaded-sketch-demo.gif)
_Loading a preloaded blink sketch_

I have also updated the sketch to use millis() instead of delay() since delay() actually pauses the program which is not what I wanted to do. The final Blink sketch will look as follow:

```
static void SetLedState(bool state)
{
    if (state)
    {
        digitalWrite(LED_BUILTIN, HIGH);
    }
    else
    {
        digitalWrite(LED_BUILTIN, LOW);
    }
}
 
void setup()
{
    pinMode(LED_BUILTIN, OUTPUT);
    digitalWrite(LED_BUILTIN, LOW);
}
 
void loop()
{
    unsigned int secondsSinceBoot = millis() / 1000;
    
    if (secondsSinceBoot % 2 != 0)
    {
        SetLedState(true);
    }
    else
    {
        SetLedState(false);
    }
}
```

After that, all that’s needed to be done is connect the USB cable, set the correct COM port and load it onto the Arduino Uno.

![Blink sketch demo](/assets/posts/2021-04-07-hello-world/built-in-led-actual-demo.gif)

_Blink sketch demo_
