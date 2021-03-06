---
layout: post
title: Setting Up Remote Access For Raspberry Pi Development
date: 2021-09-09
background: '/assets/posts/2021-09-09-setting-up-remote-access-for-raspberry-pi-development/post-banner-2021-09-09-setting-up-remote-access-for-raspberry-pi-development.jpg'
Tag:
    - Raspberry Pi
    - Tooling
    - SSH
    - VS Code
---

The Raspberry Pi is a single board computer (SBC) and is capable of running a desktop capable operating system so we can use it as we would on other desktop machines. But for me, a Raspberry Pi will likely to be used as a gateway or home automation agent. So it will probably be tucked away somewhere and accessed through using SSH.

## SSH configuration

The Raspberry Pi by default will not have the SSH server enabled so the first thing to do is enable that. It is as easy as ticking a checkbox.

![Enabling SSH via desktop UI](/assets/posts/2021-09-09-setting-up-remote-access-for-raspberry-pi-development/gui-version-of-enabling-ssh.jpg)
_Enabling SSH via desktop UI_

By default the hostname is raspberrypi and the user is pi

Now let’s head over another machine and check that we can connect to it. Usually the first thing I do is quickly ping it just to make sure it’s there by executing the command “ping raspberrypi.local” where raspberrypi is the host name and local is the top level domain, in this case is the local network.

![Ping demo](/assets/posts/2021-09-09-setting-up-remote-access-for-raspberry-pi-development/pi_ping_test.gif)
_Ping demo_

Next let’s check that we can `ssh` into the Raspberry Pi.

![SSH demo](/assets/posts/2021-09-09-setting-up-remote-access-for-raspberry-pi-development/pi_ssh_test.gif)
_SSH demo_

In the example above, we need to enter our password everytime we log into the Raspberry Pi. To log without needing to enter our password we can copy the `public ssh key` of the machine we will be connecting from to the Raspberry Pi (assuming we have generated a private/public ssh key pair). This basically tell the Raspberry Pi to use this public key (lock) for authentication and the machine we are connecting from will have the private key (key) to complete the authentication process.

![Copy public ssh key to Raspberry Pi](/assets/posts/2021-09-09-setting-up-remote-access-for-raspberry-pi-development/copy_ssh_key_to_pi.gif)
_Copy public ssh key to Raspberry Pi_

On the machine where we will be connecting from, add the following text to a file call “config” and save it under the “.ssh” folder. All we are saying here is when ssh into this host, use this key.

```
   Host raspberrypi.local
      User pi
      IdentityFile ~/.ssh/id_rsa
```

That’s it. Now if we try to ssh into the Raspberry Pi again, we will see that we won’t need to provide a password.

![Authenticating via ssh key](/assets/posts/2021-09-09-setting-up-remote-access-for-raspberry-pi-development/ssh_into_pi_without_password.gif)
_Authenticating via ssh key_

## Remote development with VS Code

Install `Remote - SSH` VS Code extension

![Installing Remote ssh extension](/assets/posts/2021-09-09-setting-up-remote-access-for-raspberry-pi-development/install_vscode_ssh_extension.gif)
_Installing Remote ssh extension_

Making a ssh connection with a remote machine is quite straight forward using the above extension. If we need more info, check out it’s documentation at [VS Code marketplace](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh).

Once we are connected, we can start coding just like we would on a local machine.
