+++
title = "Metrics"
weight = 440
+++

Eclipse Hono&trade;'s components report several metrics which may be used to gain some insight
into the running system. For instance, the HTTP adapter reports the number of successfully
processed telemetry messages. Some of these metrics are considered part of Hono's external
interface. This section describes the semantics and format of the metrics, how they can be retrieved
and how to interpret actual values.
<!--more-->

## Reported Metrics

Hono uses [Micrometer](https://micrometer.io/) in combination with Spring Boot
to internally collect metrics. Those metrics can be exported to different
back ends. Please refer to [Configuring Metrics]({{< relref "/admin-guide/monitoring-tracing-config.md#configuring-metrics" >}})
for details.

The example deployment by default uses [Prometheus](https://prometheus.io/) as the metrics back end.

When deploying to Kubernetes/OpenShift, the metrics reported by Hono may contain
environment specific tags (like the *pod* name) which are added by the Prometheus
scraper. However, those tags are not part of the Hono metrics definition.

Hono applications may report other metrics in addition to the ones defined here.
In particular, all components report metrics regarding the JVM's internal state, e.g.
memory consumption and garbage collection status. Those metrics are not considered
part of Hono's *official* metrics definition. However, all those metrics
will still contain tags as described below.

### Common Metrics

Tags for common metrics are:

| Tag              | Value                              | Description |
| ---------------- | ---------------------------------- | ----------- |
| *host*           | *string*                           | The name of the host that the component reporting the metric is running on |
| *component-type* | `adapter`, `service`             | The type of component reporting the metric |
| *component-name* | *string*                           | The name of the component reporting the metric. |

The names of Hono's standard components are as follows:

| Component         | *component-name*      |
| ----------------- | --------------------- |
| Auth Server       | `hono-auth`         |
| Device Registry   | `hono-registry`     |
| AMQP adapter      | `hono-amqp`         |
| CoAP adapter      | `hono-coap`         |
| HTTP adapter      | `hono-http`         |
| Kura adapter      | `hono-kura-mqtt`   |
| MQTT adapter      | `hono-mqtt`         |

### Protocol Adapter Metrics

Additional tags for protocol adapters are:

| Name        | Value                                              | Description |
| ----------- | -------------------------------------------------- | ----------- |
| *direction* | `one-way`, `request`, `response`               | The direction in which a Command &amp; Control message is being sent:<br>`one-way` indicates a command sent to a device for which the sending application doesn't expect to receive a response.<br>`request` indicates a command request message sent to a device.<br>`response` indicates a command response received from a device. |
| *qos*       | `0`, `1`, `unknown`                              | The quality of service used for a telemetry or event message.<br>`0` indicates *at most once*,<br>`1` indicates *at least once* and<br> `none` indicates unknown delivery semantics. |
| *status*    | `forwarded`, `unprocessable`, `undeliverable` | The processing status of a message.<br>`forwarded` indicates that the message has been forwarded to a downstream consumer<br>`unprocessable` indicates that the message has not been processed not forwarded, e.g. because the message was malformed<br>`undeliverable` indicates that the message could not be forwarded, e.g. because there is no downstream consumer or due to an infrastructure problem |
| *tenant*    | *string*                                           | The identifier of the tenant that the metric is being reported for |
| *ttd*       | `command`, `expired`, `none`                    | A status indicating the outcome of processing a TTD value contained in a message received from a device.<br>`command` indicates that a command for the device has been included in the response to the device's request for uploading the message.<br>`expired` indicates that a response without a command has been sent to the device.<br>`none` indicates that either no TTD value has been specified by the device or that the protocol adapter does not support it. |
| *type*      | `telemetry`, `event`                             | The type of (downstream) message that the metric is being reported for. |

Metrics provided by the protocol adapters are:

| Metric                             | Type                | Tags                                                                                         | Description |
| ---------------------------------- | ------------------- | -------------------------------------------------------------------------------------------- | ----------- |
| *hono.commands.received*           | Timer               | *host*, *component-type*, *component-name*, *tenant*, *type*, *status*, *direction*          | The time it took to process a message conveying a command or a response to a command. |
| *hono.commands.payload*            | DistributionSummary | *host*, *component-type*, *component-name*, *tenant*, *type*, *status*, *direction*          | The number of bytes conveyed in the payload of a command message. |
| *hono.connections.authenticated*   | Gauge               | *host*, *component-type*, *component-name*, *tenant*                                         | Current number of connected, authenticated devices. <br/> **NB** This metric is only supported by protocol adapters that maintain *connection state* with authenticated devices. In particular, the HTTP adapter does not support this metric. |
| *hono.connections.unauthenticated* | Gauge               | *host*, *component-type*, *component-name*                                                   | Current number of connected, unauthenticated devices. <br/> **NB** This metric is only supported by protocol adapters that maintain *connection state* with authenticated devices. In particular, the HTTP adapter does not support this metric. |
| *hono.messages.received*           | Timer               | *host*, *component-type*, *component-name*, *tenant*, *type*, *status*, *qos*, *ttd*         | The time it took to process a message conveying telemetry data or an event. |
| *hono.messages.payload*            | DistributionSummary | *host*, *component-type*, *component-name*, *tenant*, *type*, *status*                       | The number of bytes conveyed in the payload of a telemetry or event message. |

#### Minimum Message Size

If a minimum message size is configured for a [tenant]({{< relref "/api/tenant#tenant-information-format" >}}), 
then the payload size of the telemetry, event and command messages are calculated in accordance with the configured 
value and then reported to the metrics by the AMQP, HTTP and MQTT protocol adapters. If minimum message size is not 
configured for a tenant then the actual message payload size is reported.

Assume that the minimum message size for a tenant is configured as 4096 bytes (4KB). The payload size of 
an incoming message with size 1KB is calculated as 4KB by the protocol adapters and reported to the metrics system.
For an incoming message of size 10KB, it is reported as 12KB.

### Service Metrics

Hono's service components do not report any metrics at the moment.
