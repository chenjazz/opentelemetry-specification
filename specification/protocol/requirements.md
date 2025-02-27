# OpenTelemetry Protocol Requirements

This document will drive OpenTelemetry Protocol design and RFC.

## Goals

See the goals of OpenTelemetry Protocol design [here](design-goals.md).

## Vocabulary

There are 2 parties involved in telemetry data exchange. In this document the party that is the source of telemetry data is called the Client, the party that is the destination of telemetry data is called the Server.

遥测数据交换涉及两方。在本文档中，遥测数据的来源方称为客户端，遥测数据的目的地方称为服务器。

Examples of a Client are instrumented applications or sending side of telemetry collectors, examples of Servers are telemetry backends or receiving side of telemetry collectors (so a Collector is typically both a Client and a Server depending on which side you look from).

客户端的示例是instrumented应用程序或遥测收集器的发送端，服务器的示例是遥测后端或遥测收集器的接收端（因此，收集器通常既是客户端又是服务器，具体取决于您从哪一边看）。

## Known Issues with Existing Protocols

Our experience with OpenCensus and other protocols has been that many of them have one or more of the following drawbacks:

我们使用 OpenCensus 和其他协议的经验表明，其中许多协议具有以下一个或多个缺点：

- High CPU consumption for serialization and especially deserialization of received telemetry data.序列化特别是接收遥测数据的反序列化的 CPU 消耗很高。
- High and frequent CPU consumption by Garbage Collector.垃圾收集器高且频繁的 CPU 消耗。
- Lack of delivery guarantees for certain protocols (e.g. stream-based gRPC OpenCensus protocol) which makes troubleshooting of telemetry pipelines difficult.某些协议（例如基于流的 gRPC OpenCensus 协议）缺乏交付保证，这使得遥测管道的故障排除变得困难。
- Not aware / not cooperating with load balancers resulting in potentially large imbalances in horizontally scaled backends.不知道/不与负载平衡器合作导致水平扩展后端中潜在的巨大不平衡。
- Support either traces or metrics but not both.

Our goal is to avoid or mitigate these known issues in the new protocol.我们的目标是在新协议中避免或减轻这些已知问题。

## Requirements

The following are OpenTelemetry protocol requirements.

### Supported Node Types

The protocol must be suitable for use between all of the following node types: instrumented applications, telemetry backends, telemetry agents running as local daemons, stand-alone collector/forwarder services.

### Supported Data Types

The protocol must support traces and metrics as data types.

### Reliability of Delivery  交付的可靠性


The protocol must ensure reliable data delivery and clear visibility when the data cannot be delivered. This should be achieved by sending data acknowledgements from the Server to the Client.

Note that acknowledgements alone are not sufficient to guarantee that: a) no data will be lost and b) no data will be duplicated. Acknowledgements can help to guarantee a) but not b). Guaranteeing both at the same is difficult. Because it is usually preferable for telemetry data to be duplicated than to lose it, we choose to guarantee that there are no data losses while potentially allowing duplicate data.

Duplicates can typically happen in edge cases (e.g. on reconnections, network interruptions, etc) when the client has no way of knowing if last sent data was delivered. In these cases the client will usually choose to re-send the data to guarantee the delivery which in turn may result in duplicate data on the server side.

_To avoid having duplicates the client and the server could track sent and delivered items using uniquely identifying ids. The exact mechanism for tracking the ids and performing data de-duplication may be defined at the layer above the protocol layer and is outside the scope of this document._

For this reason we have slightly relaxed requirements and consider duplicate data acceptable in rare cases.

Note: this protocol is concerned with reliability of delivery between one pair of client/server nodes and aims to ensure that no data is lost in-transit between the client and the server. Many telemetry collection systems have multiple nodes that the data must travel across until reaching the final destination (e.g. application -> agent -> collector -> backend). End-to-end delivery guarantees in such systems is outside of the scope for this document. The acknowledgements described in this protocol happen between a single client/server pair and do not span multiple nodes in multi-hop delivery paths.

### Throughput  吞吐量

The protocol must ensure high throughput in high latency networks when the client and the server are not in the same data center.

This requirement may rule out half-duplex protocols. The throughput of half-duplex protocols is highly dependent on network roundtrip time and request size. To achieve good throughput request sizes may be too large to be practical.

### Compression 压缩

The protocol must achieve high compression ratios for telemetry data. The protocol design must consider batching of telemetry data and grouping of similar data (both can help to achieve better compression using common compression algorithms).

### Encryption  加密

Industry standard encryption (e.g. TLS/HTTPS) must be supported.

### Backpressure Signalling and Throttling  背压信号和节流


The protocol must allow backpressure signalling.

If the server is unable to keep up with the pace of data it receives from the client then it must be able to signal that fact to the client. The client may then throttle itself to avoid overwhelming the server.

If the underlying transport is a stream that has its own flow control mechanism then the backpressure could be applied by delaying the reading of data from the server’s endpoint which could then be signalled to the client via underlying flow-control. However this approach makes it difficult for the client to distinguish server overloading from network delays (due to e.g. network losses). Such distinction is important for [observability reasons](https://github.com/open-telemetry/opentelemetry-service/pull/188). Because of this it is required for the protocol to allow to explicitly and clearly signal backpressure from the server to the client without relying on implicit signalling using underlying flow-control mechanisms.

The backpressure signal should include a hint to the client about desirable reduced rate of data.

### Serialization Performance  序列化性能


The protocol must have fast data serialization and deserialization characteristics.

Ideally it must also support very fast pass-through mode (when no modifications to the data are needed), fast “augmenting” or “tagging” of data and partial inspection of data (e.g. check for presence of specific tag). These requirements help to create fast Agents and Collectors.

### Memory Usage Profile  内存使用配置文件


The protocol must impose minimal pressure on memory manager, including pass-through scenarios, when deserialized data is short-lived and must be serialized as-is shortly after and when such short-lived data is created and discarded at high frequency (think telemetry data forwarders).

The implementation of telemetry protocol must aim to minimize the number of memory allocations and dealocations performed during serialization and deserialization and aim to minimize the pressure on Garbage Collection (for GC languages).

### Level 7 Load Balancer Friendly  7 级负载均衡器友好


The protocol must allow Level 7 load balancers such as Envoy to re-balance the traffic for each batch of telemetry data. The traffic should not get pinned by a load balancer to one server for the entire duration of telemetry data sending, thus potentially leading to imbalanced load of servers located behind the load balancer.

该协议必须允许 7 级负载平衡器（例如 Envoy）重新平衡每批遥测数据的流量。在遥测数据发送的整个持续时间内，流量不应被负载平衡器固定到一台服务器，从而可能导致位于负载平衡器后面的服务器负载不平衡。

### Backwards Compatibility  向后兼容


The protocol should be possible to evolve over time. It should be possible for nodes that implement different versions of OpenTelemetry protocol to interoperate (while possibly regressing to the lowest common denominator from functional perspective).

### General Requirements  一般要求

The protocol must use well-known, mature encoding and transport mechanisms with ubiquitous availability of implementations in wide selection of languages that are supported by OpenTelemetry.
