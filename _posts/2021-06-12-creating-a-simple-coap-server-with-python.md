---
layout: post
title: Creating a Simple CoAP Server with Python
date: 2021-06-12
background: '/assets/posts/2021-06-12-creating-a-simple-coap-server-with-python/post-banner-2021-06-12-creating-a-simple-coap-server-with-python.jpg'
Tag:
    - IoT
    - Python
    - CoAP
    - Aiocoap
    - Asyncio
---

# Creating a Simple CoAP Server with Python

## What is CoAP?

CoAP stands for Constrained Application Protocol, is a communication protocol intended for constrained devices or constrained communication channels. It has a client/server based model much like HTTP but with the bonus of the client able to register as a resource observer, unlike HTTP where the client needs to poll the server for updates. While CoAP is built on top of UDP, it does provide mechanism for reliable communications.

More information about CoAP can be found in the following resources:

* [Wikipedia for CoAP](https://en.wikipedia.org/wiki/Constrained_Application_Protocol)
* [CoAP Technology](https://coap.technology/)
* [CoAP RFC7252](https://datatracker.ietf.org/doc/html/rfc7252)

## What is a CoAP server?

A CoAP server is generally a service running on a constrained device hosting/managing one or more resources (e.g. light switches, alarms, sensors). Each resource will be provided with a unique CoAP address endpoint and a defined API so that CoAP clients could interact with that resource.

## Creating a simple CoAP server

We will create simple CoAP server with a single resource. The resource will hold the state of an alarm. There will be two access methods:

* PUT
    + This method allows a CoAP client connected to the alarm to update it’s state on the server.
* OBSERVABLE GET
    + This method allows other CoAP clients to register with the resource so they can be notified when the state of the alarm changes.

To create a simple CoAP server you can use Python 3 and the [Aiocoap](https://github.com/chrysn/aiocoap) library.

![Create a Python3 environment for the simple CoAP server](/assets/posts/2021-06-12-creating-a-simple-coap-server-with-python/coap_server_create_venv_demo.gif)
_Create a Python3 environment for the simple CoAP server_

![Install Aiocoap Python library](/assets/posts/2021-06-12-creating-a-simple-coap-server-with-python/coap_server_install_aiocoap_demo.gif)
_Install Aiocoap Python library_

In VS Code, create a file named server.py and add a class named AlarmResource. This class will manage CoAP requests for the alarm resource.

```
    # server.py

    import aiocoap.resource as resource
    import aiocoap

    class AlarmResource(resource.Resource):
        """This resource supports the PUT method.
        PUT: Update state of alarm."""

        def __init__(self):
            super().__init__()
            self.state = "OFF"

        async def render_put(self, request):
            self.state = request.payload
            print('Update alarm state: %s' % self.state)

            return aiocoap.Message(code=aiocoap.CHANGED, payload=self.state)
```

Now create a main() method to initialise the server and add the alarm resources to it.

```
    import asyncio

    def main():
        # Resource tree creation
        root = resource.Site()
        root.add_resource(['alarm'], AlarmResource())

        asyncio.Task(aiocoap.Context.create_server_context(root, bind=('localhost', 5683)))

        asyncio.get_event_loop().run_forever()

    if __name__ == "__main__":
        main()
```

To test the server you can create a simple client that randomly update the state of the alarm every time it is run (by sending a PUT request with either an “ON” or “OFF” payload).

```
    # client_put.py
    import asyncio
    import random

    from aiocoap import *

    async def main():
        context = await Context.create_client_context()
        alarm_state = random.choice([True, False])
        payload = b"OFF"

        if alarm_state:
            payload = b"ON"

        request = Message(code=PUT, payload=payload, uri="coap://localhost/alarm")

        response = await context.request(request).response
        print('Result: %s\n%r'%(response.code, response.payload))

    if __name__ == "__main__":
        asyncio.get_event_loop().run_until_complete(main())
```

Below is a demo showing the client (left) updating the server (right).

![PUT request test](/assets/posts/2021-06-12-creating-a-simple-coap-server-with-python/coap_server_put_request_demo.gif)
_PUT request test_

**Implementing CoAP Observe option**

The Observe option is an extension of the CoAP GET method. More precisely, it is an optional field in the GET request header. When a client queries the server with an Observe option, it basically asking for the current status of the alarm and also to be notified if it changes in the future.

Note that the server does not have to honour the observe request. For example, if a CoAP resource doesn’t support Observers or it has reached the maximum registered observers. In this case, the Observe option will just be ignored and the request will default to a plain GET request.

The server may also periodically send the current state of the resource to all registered observers. If it doesn’t hear anything back from any observer then that observer will be removed from the resource’s registered observers list. This is one mechanism the server uses to clean up the observers list in the event any client silently disappears.

Let proceed with the implementation by updating the `AlarmResource` class.

Instead of inheriting from Aiocoap's `Resource` class, AlarmResource will now inherit from Aiocoap's `ObservableResource`. This will provide the functionality to manage the observers, you just need to handle what to send and when to send it.

Let also update the handling of the PUT request so that when the status of the alarm is updated, it will set a flag to indicate the server to notify observers.

Add a `notify_observers_check()` method which continuously loop and check to see if the `notify_observers` flag is set or not. If it is set then the server will send an update to each observer by calling `render_get()`.

```
    # server.py

    import aiocoap.resource as resource
    import aiocoap
    import threading
    import logging
    import asyncio

    class AlarmResource(resource.ObservableResource):
        """This resource supports the GET and PUT methods and is observable.
        GET: Return current state of alarm
        PUT: Update state of alarm and notify registered observers
        """

        def __init__(self):
            super().__init__()

            self.status = "OFF"
            self.has_observers = False
            self.notify_observers = False

        # Ensure observers are notify if required
        def notify_observers_check(self):
            while True:
                if self.has_observers and self.notify_observers:
                    print('Notifying observers')
                    self.updated_state()
                    self.notify_observers = False

        # Observers change event callback
        def update_observation_count(self, count):
            if count:
                self.has_observers = True
            else:
                self.has_observers = False

        # Handles GET request or observer notify
        async def render_get(self, request):
            print('Return alarm state: %s' % self.status)
            payload = b'%s' % self.status.encode('ascii')

            return aiocoap.Message(payload=payload)

        # Handles PUT request
        async def render_put(self, request):
            self.status = request.payload.decode('ascii')
            print('Update alarm state: %s' % self.status)
            self.notify_observers = True

            return aiocoap.Message(code=aiocoap.CHANGED, payload=b'%s' % self.status.encode('ascii'))
```

Update the `main()` method so that it will spawn a separate daemon thread with the sole purpose of calling `notify_observers_check()` to handle the notify_observers flag event.

```
    logging.basicConfig(level=logging.INFO)
    logging.getLogger("coap-server").setLevel(logging.DEBUG)

    def main():
        # Resource tree creation
        root = resource.Site()
        alarmResource = AlarmResource()
        root.add_resource(['alarm'], alarmResource)
        asyncio.Task(aiocoap.Context.create_server_context(root, bind=('localhost', 5683)))

        # Spawn a daemon to notify observers when alarm status changes
        observers_notifier = threading.Thread(target=alarmResource.notify_observers_check)
        observers_notifier.daemon = True
        observers_notifier.start()

        asyncio.get_event_loop().run_forever()

    if __name__ == "__main__":
        main()
```

To test the Observe option you can create another client that will start observing the alarm status. This observe client will also use the Aiocoap python library. Whenever a notification is received, observe_callback() is called.

```
    # client_observe.py

    import logging
    import asyncio

    from aiocoap import *

    logging.basicConfig(level=logging.INFO)

    def observe_callback(response):
        if response.code.is_successful():
            print("Alarm status: %s" % (response.payload.decode('ascii')))
        else:
            print('Error code %s' % response.code)

    async def main():
        context = await Context.create_client_context()

        request = Message(code=GET)
        request.set_request_uri('coap://localhost/alarm')
        request.opt.observe = 0
        observation_is_over = asyncio.Future()

        try:
            context_request = context.request(request)
            context_request.observation.register_callback(observe_callback)
            response = await context_request.response
            exit_reason = await observation_is_over
            print('Observation is over: %r' % exit_reason)
        finally:
            if not context_request.response.done():
                context_request.response.cancel()
            if not context_request.observation.cancelled:
                context_request.observation.cancel()

    if __name__ == "__main__":
        asyncio.get_event_loop().run_until_complete(main())
```

Below is a demo showing a client (left) which randomly set the alarm to “ON” or “OFF” each time it is called. The server in the middle and another client (right) which has been registered with the server to get notified.

![Observe request test](/assets/posts/2021-06-12-creating-a-simple-coap-server-with-python/coap_observe_demo.gif)
_Observe request test_

