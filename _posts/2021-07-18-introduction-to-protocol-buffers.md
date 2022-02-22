---
layout: post
title: Introduction to Protocol Buffers
date: 2021-07-18
background: '/assets/posts/2021-07-18-introduction-to-protocol-buffers/post-banner-2021-07-18-introduction-to-protocol-buffers.jpg'
Tag:
    - Protocol Buffers
    - Protobuf
    - Serialization
    - Python
    - C#
    - IoT
    - Powershell
---

Protocol buffers (or protobuf) is a method of serializing structured data, and is ideal for communication across constraint systems. It is language-neutral and platform-neutral, developed and used by Google since 2001. It was made available to the public in 2008 (free and open-sourced). Official documentation and source code are available at the following links:

* [Official documentation](https://developers.google.com/protocol-buffers)
* [Protocol buffers source code](https://github.com/protocolbuffers/protobuf)

The way protocol buffers works is you define a schema that encapsulates all the information you wish to communicate and you store it in a proto file. Let’s say you want to send an alarm status event which also include time and date. Then you might create a schema called AlarmStatus.proto which looks something like below:

```
    // AlarmStatus.proto
    syntax = "proto3";

    message AlarmStatus {
        bool alarm_active = 1;
        string time_utc   = 2;
        string date_utc   = 3;
    }
```

From that, you use the protobuf compiler to create a language specific class that implements the encoding/decoding of the protocol buffer data.

## Installing the Protobuf Compiler

The protocol compiler (protoc) is the common tool chain for converting our protobuf schemas to binary classes and is required regardless of what language you want to compile to.

* From the download page, download the latest protoc pre-built binaries specific to your development environment: protoc-$VERSION-$PLATFORM.zip.
* Unpack the zip file to a preferred folder and add an environment variable pointing the the folder containing the protoc executable e.g.

```
    C:\protoc-3.17.3-win64\bin
```

If everything is setup correctly, you should be able to execute protoc.exe from any working directory.

![Proto.exe demo](/assets/posts/2021-07-18-introduction-to-protocol-buffers/protoc_exe_demo.gif)

## Protocol Buffers Using Python

**Generating Python compiled protocol buffers from message definition**

From the [download page](https://github.com/protocolbuffers/protobuf/releases/latest), download the latest protocol buffers package for Python and follow README.md for instructions.

At this point if you run pip list,  you should see that protobuf is installed.

```
    PS D:\Downloads\protobuf-3.17.3\python> pip list
    Package    Version
    ---------- -------
    pip        21.1.2
    protobuf   3.17.3
    setuptools 56.0.0
    six        1.16.0
    wheel      0.36.2
```
Given that you have created your protobuf schema(s), you can easily generate the associated python class files with the following command:

```
    PS D:\Workspace\ProtobufDemo> protoc --python_out=. *.proto
```

![Generate Python class from schema](/assets/posts/2021-07-18-introduction-to-protocol-buffers/generate_python_class_demo.gif)
_Generate Python class from schema_

**Protobuf serialisation with Python**

To serialise a protobuf message you simply create a message, populate it then call `SerializeToString()`.

```
    # SerialisationDemo.py

    import random
    from datetime import datetime, timezone
    from AlarmStatus_pb2 import AlarmStatus

    def get_new_alarm_status():
        date_time_utc = datetime.now(timezone.utc)
        alarm_status = AlarmStatus()
        alarm_status.alarm_active = bool(random.getrandbits(1))
        alarm_status.time_utc = "{}".format(date_time_utc.time())
        alarm_status.date_utc = "{}".format(date_time_utc.date())

        return alarm_status

    def serialise_alarm_status(alarm_status):
        return alarm_status.SerializeToString()

    def main():
        alarm_status = get_new_alarm_status()
        print("Alarm status: {}, time: {}, date: {}".format(alarm_status.alarm_active, alarm_status.time_utc, alarm_status.date_utc))

        serialised_alarm_status = serialise_alarm_status(alarm_status)
        print("Serialised data: {}".format(serialised_alarm_status))

    if __name__ == "__main__":
        main()
```

![Protobuf serialisation with Python](/assets/posts/2021-07-18-introduction-to-protocol-buffers/serialising_python_class_demo.gif)

**Protobuf deserialisation with Python**

To deserialise a protobuf message you simply create a message, then call ParseFromString() passing in the serialised data.

```
    # SerialisationDemo.py

    import random
    from datetime import datetime, timezone
    from AlarmStatus_pb2 import AlarmStatus

    def get_new_alarm_status():
        date_time_utc = datetime.now(timezone.utc)
        alarm_status = AlarmStatus()
        alarm_status.alarm_active = bool(random.getrandbits(1))
        alarm_status.time_utc = "{}".format(date_time_utc.time())
        alarm_status.date_utc = "{}".format(date_time_utc.date())

        return alarm_status

    def serialise_alarm_status(alarm_status):
        return alarm_status.SerializeToString()

    def deserialise_alarm_status(serialised_alarm_status):
        alarm_status = AlarmStatus()
        alarm_status.ParseFromString(serialised_alarm_status)

        return alarm_status

    def main():
        alarm_status = get_new_alarm_status()
        print("Alarm status: {}, time: {}, date: {}".format(alarm_status.alarm_active, alarm_status.time_utc, alarm_status.date_utc))

        serialised_alarm_status = serialise_alarm_status(alarm_status)
        print("Serialised data: {}".format(serialised_alarm_status))

        alarm_status2 = deserialise_alarm_status(serialised_alarm_status)
        print("Deserialised alarm status: {}, time: {}, date: {}".format(alarm_status2.alarm_active, alarm_status2.time_utc, alarm_status2.date_utc))

    if __name__ == "__main__":
        main()
```

![Protobuf deserialisation with Python](/assets/posts/2021-07-18-introduction-to-protocol-buffers/deserialising_python_class_demo.gif)

## Protocol Buffers Using C#

**Generating C# compiled protocol buffers from message definition**

Given that you have created your protobuf schema(s), you can easily generate the associated C# class files with the following command `protoc –csharp_out=. *.proto`

```
    PS D:\Workspace\ProtobufDemo> ls

        Directory: D:\Workspace\ProtobufDemo

    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    -a---          21/07/2021  4:00 pm            128 AlarmStatus.proto

    PS D:\Workspace\ProtobufDemo> protoc --csharp_out=. *.proto
    PS D:\Workspace\ProtobufDemo> ls

        Directory: D:\Workspace\ProtobufDemo

    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    -a---          21/07/2021  9:44 pm           9841 AlarmStatus.cs
    -a---          21/07/2021  4:00 pm            128 AlarmStatus.proto

    PS D:\Workspace\ProtobufDemo>
```

**Serialising protocol buffers in C#**

Create a new console application and add the Google.Protobuf NuGet package.

```
    PS D:\Workspace\ProtobufDemo> dotnet new console
    ...
    PS D:\Workspace\ProtobufDemo> ls

        Directory: D:\Workspace\ProtobufDemo

    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    d----          21/07/2021  9:44 pm                obj
    -a---          21/07/2021  9:44 pm           9841 AlarmStatus.cs
    -a---          21/07/2021  4:00 pm            128 AlarmStatus.proto
    -a---          21/07/2021  9:44 pm            194 Program.cs
    -a---          21/07/2021  9:44 pm            171 ProtobufDemo.csproj

    PS D:\Workspace\ProtobufDemo> dotnet add package Google.Protobuf
    ...
    PS D:\Workspace\ProtobufDemo>
```

To serialise a protobuf message you simply create a message, populate it then call ToByteString().

```
    using System;
    using Google.Protobuf;

    namespace ProtobufDemo
    {
        class Program
        {
            static void Main(string[] args)
            {
                var alarmStatus = GetNewAlarmStatus();
                var serialisedAlarmStatus = SerialiseAlarmStatus(alarmStatus);
                Console.WriteLine($"Alarm status: {alarmStatus.AlarmActive}, time: {alarmStatus.TimeUtc}, date: {alarmStatus.DateUtc}");
                Console.WriteLine($"Serialised data: {serialisedAlarmStatus}");
            }

            private static AlarmStatus GetNewAlarmStatus()
            {
                var rand = new Random();
                var status = rand.Next(2) == 1;
                var dateTime = DateTimeOffset.UtcNow;

                return new AlarmStatus
                {
                    AlarmActive = status,
                    TimeUtc = dateTime.ToString("T"),
                    DateUtc = dateTime.ToString("d")
                };
            }

            private static ByteString SerialiseAlarmStatus(AlarmStatus alarmStatus)
            {
                return alarmStatus.ToByteString();
            }
        }
    }
```

![Protobuf serialisation with C#](/assets/posts/2021-07-18-introduction-to-protocol-buffers/serialising_csharp_class_demo.gif)
_Protobuf serialisation with C#_

**Deserialising protocol buffers in C#**

To deserialise a protobuf message we simply call MergeFrom() passing in the serialised message.

```
    ...
    static void Main(string[] args)
    {
        ...

        var deserialisedAlarmStatus = DeserialiseAlarmStatus(serialisedAlarmStatus);
        Console.WriteLine($"Deserialised alarm status: {deserialisedAlarmStatus.AlarmActive}, time: {deserialisedAlarmStatus.TimeUtc}, date: {deserialisedAlarmStatus.DateUtc}");
    }

    ...

    private static AlarmStatus DeserialiseAlarmStatus(ByteString serialisedAlarmStatus)
    {
        var deserialisedAlarmStatus = new AlarmStatus();
        deserialisedAlarmStatus.MergeFrom(serialisedAlarmStatus);

        return deserialisedAlarmStatus;
    }
    ...
```

![Protobuf deserialisation with C#](/assets/posts/2021-07-18-introduction-to-protocol-buffers/deserialising_csharp_class_demo.gif)
_Protobuf deserialisation with C#_

And that's all there is to it.