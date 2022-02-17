---
layout: post
title: Setting Up .NET Core on a Raspberry Pi
date: 2021-09-15
background: '/assets/posts/2021-09-15-setting-up-net-core-on-a-raspberry-pi/post-banner-2021-09-15-setting-up-net-core-on-a-raspberry-pi.jpg'
Tag:
    - Raspberry Pi
    - Tooling
    - .NET
---

# Setting Up .NET Core on a Raspberry Pi

This is a quick excercise to get .NET Core SDK up and running on a Raspberry Pi.

![No .NET SDK installed demo](/assets/posts/2021-09-15-setting-up-net-core-on-a-raspberry-pi/dotnet_not_installed.gif)
_No .NET SDK installed demo_

As you can see, executing the `dotnet` command does not work.

At the time of writing, I have downloaded the ARM32 .NET SDK version 5.0.401, however you may be able to download a later/different version by visiting the [.NET Download page](https://dotnet.microsoft.com/download/dotnet). You can get more details about the installed Raspberry Pi OS so that you can install the correct SDK by using the `cat /etc/os-release` and `uname -m` command.

Download and install the .NET SDK onto the raspberry pi by issuing the following commands:

```
   sudo apt-get install curl libunwind8 gettext
   wget https://download.visualstudio.microsoft.com/download/pr/ce3cef63-ade6-4209-80f0-ac2815c5b282/e4a8b52aacf74d2a7d6d1cf5b9dca438/dotnet-sdk-5.0.401-linux-arm.tar.gz
   mkdir -p $HOME/dotnet
   tar zxf dotnet-sdk-5.0.401-linux-arm.tar.gz -C $HOME/dotnet
   export DOTNET_ROOT=$HOME/dotnet
   export PATH=$PATH:$HOME/dotnet
```

Note that the two “export” command only applies to the current shell session. To make them permanent, we will need to add them to “.bashrc”.

![Installing .NET SDK demo](/assets/posts/2021-09-15-setting-up-net-core-on-a-raspberry-pi/install_dotnet_sdk_on_pi_demo.gif)
_Installing .NET SDK demo_

Now let’s quickly create a Hello World console app just to make sure the tooling and environment are correctly setup.

![Creating a new .NET Core console app](/assets/posts/2021-09-15-setting-up-net-core-on-a-raspberry-pi/new_dotnet_console_demo.gif)
_Creating a new .NET Core console app_

and that's it, you are done. The .NET 5 SDK (and therefore also the .NET 5 runtime) are installed. So whether we want to develop on target or cross compiled, the options are there.
