---
description: Initiates a long-live stream for transmitting data associated to a resource.
---

# Start Stream

A `Start Stream` method is a way of opening a communication channel between the server and the client. It can be opened by both sides, and each `Stream` will have its own Stream Identifier.

## Stream Identifiers

In all `IOTMP` protocol messages, there is a `Stream Id` field that allows identifying a request and its response over the wire. Once the request is completed \(with an [`Ok` ](../ok.md)or [`Error`](../error.md)\), the `Stream Id`can be discarded by both sides. 

With the `Start Stream` there is also a `Stream Id`, but it is kept until the Stream channel is done, i.e., one side request a [`Stop Stream`](stop-stream.md). So, all the messages sent over an established stream, either with [`Stream Sample`]() or [`Stream Event`](stream-event.md), will include the `Stream Id` established in the `Start Stream` request. This allows to save bandwidth, as it is not required to send the resource identifier over and over again.

## Request

### Header

| Field | Value | Description |
| :--- | :--- | :--- |
| **Message Type** | 0x08 | Start Stream |
| **Message Size** | [varint](../../definitions.md#varint) | Remaining Message Length |

### Body

| Name | Field | Type | Mandatory | Description |
| :--- | :--- | :--- | :--- | :--- |
| **Stream Id** | 0x01 | [varint](../../definitions.md#varint) | Yes | Stream identifier to be used within the stream life-time. |
| **Parameters** | 0x02 | [any](../../definitions.md#any) | No | Start stream parameters. |
| **Resource**  | 0x03 | \*\*\*\*[**any**](../../definitions.md#any)\*\*\*\* | Yes | A string or array of strings with the resource identifier, i.e., "temperature". |
| **Scope** | 0x04 | [any](../../definitions.md#any) | No | Any identifier that specifies the `Start Stream` scope. |

## Response

| Message | Description |
| :--- | :--- |
| \*\*\*\*[**Ok**](../ok.md)\*\*\*\* | Should answer with an OK if it is able to stream the requested resource with the specified parameters. It must contain the Stream Id specified in this message. |
| \*\*\*\*[**Error**](../error.md)\*\*\*\* | Should answer with an Error if the target resource cannot be streamed both if the resource does not exists or the provided parameters are unacceptable. It must contain the Stream Id specified in this message. |

## Example

TBD

## Thinger.io

### Server Scopes

Thinger.io server defines some scopes when clients open streams over the server. In the Thinger.io implementation, clients can start streams to publish/subscribe to MQTT topics, subscribe to server events, or other device resources, etc. 

The following table specifies the available scopes in the Thinger.io server \(sent by Thinger.io clients\):

| Scope | Value | Description |
| :--- | :--- | :--- |
| MQTT Topic Subscribe | 0x01 | The client is opening a stream to publish  in the MQTT topic specified in the resource field. |
| MQTT Topic Publish | 0x02 | The client is opening a stream to subscribe to the MQTT topic specified in the resource field. |
| Server Event Subscribe | 0x03 | The client is opening a stream to receive server events specified in resource field. |
| Device Property Update | 0x04 | The client is opening a stream to receive property updates from the one specified in the resource field. |

### Client Scopes

Thinger.io clients defines resources that can be streamed in two different ways. One way allows starting a stream, so the resource is periodically sampled at a specified interval. The other way allow opening a stream to a resource, that is streamed just when it is necessary \(defined by the client implementation\). 

The following table specifies the available scopes in a Thinger.io client \(sent by Thinger.io server\):

| Scope | Value | Description |
| :--- | :--- | :--- |
| Resource Stream | 0x00 | The server is opening a stream to a client resource |
| Resource Sampling | 0x01 | The server is opening a stream to a device resource. |

### Resource definition

A resource to be used in a stream should be defined in the same way to a normal resource that can be executed over the [Run ](../run.md)message, i.e., a resource defined with the [Thinger.io Arduino Library](https://github.com/thinger-io/Arduino-Library) is defined as the following:

```cpp
void setup(){
  thing["heading"] >> [](pson& out){     
    out = getHeading();
  };
}
```

Internally, the client should keep information about each resource state, i.e., the associated stream identifier \(if any\), and the latest stream timestamp.

{% hint style="info" %}
From the perspective of a developer creating a firmware, streams should be completely transparent to the resource definition.
{% endhint %}

### Streaming with a Sampling interval

When a `Start Stream` message is received with a sampling interval, the client should look for the requested resource, and then adjust the `Stream Identifier` and the expected timestamp. In each loop, it should check all the enabled streams, and then transmit those which are required according to its timestamp and sampling interval. If a following `Start Stream` is received for the same resource, it must overwrite it, that is, update its interval, or change the Stream Identifier if necessary \(depending if the message comes only with Stream Id or Stream Id and Resource\).

### Streaming Events

To send events over a stream, it is usually required to detect the event or the change. It is completely dependent on the use case, so, the firmware must be prepared to detect and transmit the information when it is required. In the following example, the same `heading` resource defined before is transmitted if it is detected an angle difference greater than one degree. To to this, the [Thinger.io Arduino Library](https://github.com/thinger-io/Arduino-Library) defines a method to stream a given resource. This method should take into account if this resource Stream is enabled, that is, it contains a `Stream Identifier`, otherwise the message must not be sent \(as there is no anyone listening for it\).

```cpp
void setup(){
  thing["heading"] >> [](pson& out){     
    out = getHeading();
  };
}

float previousHeading = 0;
void loop() {
  thing.handle();
  float currentHeading = getHeading();
  if(abs(currentHeading-previousHeading)>=1.0f){
    thing.stream("heading");
    previousHeading=currentHeading;
  }
}
```

![Example ESP8266 running IOTMP for sending heading when there is a subtle difference.](../../.gitbook/assets/image%20%283%29.png)

## Server Implementation notes

### HTTP/REST API Integration

As described before, a typical scenario where a `Start Stream` is required is when a new `Websocket`or `Server Sent Event` is opened over the server to start listening over a resource, for example, to feed a real-time dashboard. In [Thinger.io](https://Thinger.io) implementation, a device resource endpoint is defined as follow:

```text
https://.../v3/users/alvarolb/devices/smart_irrigation/resources/temp
```

In such implementation it is possible to open a `Websocket` over the resource, so the endpoint is now:

```text
wss://.../v3/users/alvarolb/devices/smart_irrigation/resources/temp
```

Opening such `Websocket` will listen for events on the `temp` resource, that is, changes that are detected by device and transmitted to the `Websocket`. But it is possible to open also the resource as following:

```text
wss://.../v3/users/alvarolb/devices/smart_irrigation/resources/temp?interval=5
```

In such case, the `Websocket` will start receiving the information every 5 seconds. Moreover, it is possible to open an endpoint without defining a resource:

```text
wss://.../v3/users/alvarolb/devices/smart_irrigation/resources
```

And then transmit over the `Websocket` different frames with expected resources and their intervals:

```javascript
{"resource": "temp", "interval": 5, "enabled" : true}
{"resource": "humidity", "interval": 5, "enabled" : true}
```

### MQTT Integration

~~TBD~~

~~devices/smart\_irrigation/resources/temp?interval=5~~

### Different streaming intervals

There can be conflicts if different sources requires different sampling intervals for the same resource. For example, a data storage configured to write a resource every minute, and a dashboard requiring real-time data from from the same resource every 10 seconds. There are different workarounds here that can be implemented in the server, as the client should remain as simple as possible:

* Forbid multiple`Start Stream` over a resource with different sampling intervals, and force the client to define different resources for each data-sink/interval, i.e., one for the data storage, and another for the dashboard, so there are not conflicts as the resource are different.
* Allow multiple `Start Stream` over a resource with different sampling interval, but ignore newer sampling intervals.
* Allow multiple `Start Stream` and dynamically compute the Greatest Common Divisor \(GCD\) between stream requests, reconfigure the device stream interval, and provide data to each source at the specified interval. This is the most complex approach, and is the one implemented in [Thinger.io](https://thinger.io). In this case, if the dashboard A requires data every 6 seconds, and dashboard B every 9 seconds, the server computes the GCD of both sources \(3\) and configures the device to transmit every 3 seconds.  The dashboard A will get data every 6 seconds, and B every 9. If one source is added or removed, the process is repeated to accommodate to the new situation.

### Sampling intervals and events subscriptions 

There is another use case where a data sink may require to receive a resource data periodically, i.e., to store it, and other data sink require only events, i.e., raised events by the device. ~~TBD~~

