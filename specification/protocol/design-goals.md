# Design Goals for OpenTelemetry Wire Protocol -- OpenTelemetry 有线协议的设计目标

We want to design a telemetry data exchange protocol that has the following characteristics:
我们要设计一个具有以下特点的遥测数据交换协议：

- Be suitable for use between all of the following node types: instrumented applications, telemetry backends, local agents, stand-alone collectors/forwarders.
- 适合在以下所有节点类型之间使用：instrumented应用程序、telemetry后端、本地代理、独立收集器/转发器。

- Have high reliability of data delivery and clear visibility when the data cannot be delivered.
- 数据交付可靠性高，数据无法交付时清晰可见。

- Have low CPU usage for serialization and deserialization.
- 序列化和反序列化的 CPU 使用率低。

- Impose minimal pressure on memory manager, including pass-through scenarios, where deserialized data is short-lived and must be serialized as-is shortly after and where such short-lived data is created and discarded at high frequency (think telemetry data forwarders).
- 对内存管理器施加最小的压力，包括传递场景，其中反序列化的数据是短暂的，必须在不久之后按原样序列化，以及在高频创建和丢弃此类短期数据的情况下（想想遥测数据转发器）。

- Support ability to efficiently modify deserialized data and serialize again to pass further. This is related but slightly different from the previous requirement.
- 支持高效修改反序列化数据并再次序列化以进一步传递的能力。这是相关的，但与之前的要求略有不同。

- Ensure high throughput (within the available bandwidth) in high latency networks (e.g. scenarios where telemetry source and the backend are separated by high latency network).
- 确保高延迟网络中的高吞吐量（在可用带宽内）（例如遥测源和后端被高延迟网络分隔的场景）。

- Allow backpressure signalling.
- 允许背压信号。


- Be load-balancer friendly (do not hinder re-balancing).
- 对负载均衡器友好（不要妨碍重新平衡）。
