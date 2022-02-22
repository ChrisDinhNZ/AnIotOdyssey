---
layout: post
title: Getting To Know Bluetooth Low Energy
date: 2021-04-12
background: '/assets/posts/2021-04-12-getting-to-know-bluetooth-low-energy/post-banner-2021-04-12-getting-to-know-bluetooth-low-energy.jpg'
Tag:
  - Bluetooth
  - Bluetooth Low Energy
  - BLE
---

Bluetooth Low Energy (BLE) which may also have been referred to as Bluetooth Smart, is a low power wireless communication technology that can be used over a short distance to enable smart devices to communicate (e.g. Personal Area Network, Home Area Network). BLE is not (nor is it compatible with) Bluetooth Classic. Unlike Bluetooth Classic, when powered by a battery BLE can last up months or even years. This makes it an ideal technology for the internet of things (IoT).

Bluetooth Low Energy was formerly known as [Wibree](https://thefutureofthings.com/3041-nokias-wibree-and-the-wireless-zoo/), a technology developed by Nokia and it’s partners. In 2010, Bluetooth SIG merged with Wibree and renamed it Bluetooth LE. In the same year Bluetooth 4.0 was released and include both Bluetooth Classic and Bluetooth LE. Note that while Bluetooth 4.0 (and later versions) supports both BLE and Bluetooth Classic, they are not compatible. This means two devices communicating via Bluetooth must use the same Bluetooth technology.

There are two mandatory services for Bluetooth LE, `Generic Access Profile (GAP)` and `Generic Attribute Profile (GATT)`. GAP deals with connection behaviours when as GATT deals with application behaviours.

## Generic Access Profile (GAP)

![BLE GAP intro](/assets/posts/2021-04-12-getting-to-know-bluetooth-low-energy/ble-gap-intro.jpg)
_Home Area Network (HAN)_

The GAP service determines the device’s visibility to the outside world as well as the role of the device in a Bluetooth network.

In the context of the GAP service, the device will either be a peripheral device or a central device. However, Bluetooth 4.1 spec indicates that a devices can be both at the same time (e.g. A mobile phone can act a central device when connected to a smart bulb while at the same time act as a peripheral device when connected to a desktop computer).

The GAP service defined 2 modes that a device can be in:

* Advertising/discovery mode
* Connected mode

The GAP service also defined 4 roles that a device can act as:

* Broadcaster
* Observer
* Peripheral
* Central

How a device behave depends on the mode it is in and the role it is acting as. Take the smart bulb and mobile phone for example. Given the scenario where the bulb is not connected to the phone, both the bulb and the phone will be in advertising/discovery mode. The bulb will act as the Peripheral and the phone will act as the Central. The bulb will be broadcasting advertising packets with a connectable flag so that on receipt of the advertisement packets, the phone can initiate a persistent connection with the bulb.

At this point, the connection is only in the created state. After a defined period of time (called the connection interval) the mobile phone sends a data packet and the bulb will respond with a data packet. At this point, the connection is now considered established.

Note that in the case if the bulb is broadcasting non-connectable advertising packets, the phone will not be able to connect to the bulb. In this scenario, the bulb is acting as a Broadcaster role and the phone is acting as an Observer role.

## Generic Attribute Profile (GATT)

Once the connection is established, the peripheral device is referred to as the GATT Server and the central device as the GATT Client.

![BLE client server interaction](/assets/posts/2021-04-12-getting-to-know-bluetooth-low-energy/ble-gatt-client-server.jpg)
_BLE client server interaction_

In order to understand how GATT and therefore Bluetooth LE works, it would be useful to explore a little bit about the GATT Client and the GATT Server.

Basically, a GATT Server contains a table of attributes which it exposes to the GATT Client to access (read/update).

The communication between a GATT Client and GATT Server can be summarised as follow:

* Query request from client to server.
* Update request from client to server, response from server required.
* Update request from client to server, response from server not required.
* Update notification from server to client, acknowledgement from client required.
* Update notification from server to client, acknowledgement from client not required.

Each record in the table represent an attribute (a property associated with the GATT Server) and has the following structure:

![BLE attribute example](/assets/posts/2021-04-12-getting-to-know-bluetooth-low-energy/ble-attributes-examples.jpg)
_BLE attribute example_

* The Handle is a unique identifier used by the client to access the attribute.
* The Type indicates what type of attribute it is (because everything on the GATT server is exposed as an attribute).
* The Permissions indicates the accessibility of the attribute.
* The Value is the variable length value of the attribute.
* The Value Length indicates the number of bytes for the attribute value.

GATT allows the table of attributes to be organised in a well defined hierarchical data structure.

At the top of the hierarchical data structure, there is a profile. Every attributes must be encapsulated within a profile.

Data attributes are encapsulated within characteristic containers. A characteristic container includes metadata about the data attributes as well as data itself.

Characteristics can also be logically grouped together as a service. A profile can contain one or more GATT services.

![BLE GATT profile](/assets/posts/2021-04-12-getting-to-know-bluetooth-low-energy/ble-gatt-profile.jpg)
_BLE GATT profile_