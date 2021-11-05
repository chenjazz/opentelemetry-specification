# Overview

<details>
<summary>
Table of Contents
</summary>
<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [OpenTelemetry Client Architecture](#opentelemetry-client-architecture)   客户端架构
  * [API](#api)
  * [SDK](#sdk)
  * [Semantic Conventions](#semantic-conventions)
  * [Contrib Packages](#contrib-packages)
  * [Versioning and Stability](#versioning-and-stability)
- [Tracing Signal](#tracing-signal) 跟踪
  * [Traces](#traces)
  * [Spans](#spans)
  * [SpanContext](#spancontext)
  * [Links between spans](#links-between-spans)
- [Metric Signal](#metric-signal)
  * [Recording raw measurements](#recording-raw-measurements)
    + [Measure](#measure)
    + [Measurement](#measurement)
  * [Recording metrics with predefined aggregation](#recording-metrics-with-predefined-aggregation)
  * [Metrics data model and SDK](#metrics-data-model-and-sdk)
- [Log Signal](#log-signal)
  * [Data model](#data-model)
- [Baggage Signal](#baggage-signal)
- [Resources](#resources)
- [Context Propagation](#context-propagation)
- [Propagators](#propagators)
- [Collector](#collector)
- [Instrumentation Libraries](#instrumentation-libraries)

<!-- tocstop -->

</details>

This document provides an overview of the OpenTelemetry project and defines important fundamental terms.

本文档概述了 OpenTelemetry 项目并定义了重要的基本术语。

Additional term definitions can be found in the [glossary](glossary.md).

其他术语定义可以在 (glossary.md) 中找到。

## OpenTelemetry Client Architecture 客户端架构

![Cross cutting concerns](../internal/img/architecture.png)

At the highest architectural level, OpenTelemetry clients are organized into [**signals**](glossary.md#signals).
Each signal provides a specialized form of observability. For example, tracing, metrics, and baggage are three separate signals.
Signals share a common subsystem – **context propagation** – but they function independently from each other.

在最高的架构中，OpenTelemetry 是由 signals组织的，每个signals都提供了一种特殊形式的可观察性。比如tracing, metrics, and baggage是不同的signal。signal共享一个子系统-- **context propagation**，但他们彼此独立。

Each signal provides a mechanism for software to describe itself. A codebase, such as web framework or a database client, takes a dependency on various signals in order to describe itself. OpenTelemetry instrumentation code can then be mixed into the other code within that codebase.
This makes OpenTelemetry a [**cross-cutting concern**](https://en.wikipedia.org/wiki/Cross-cutting_concern) - a piece of software which is mixed into many other pieces of software in order to provide value. Cross-cutting concerns, by their very nature, violate a core design principle – separation of concerns. As a result, OpenTelemetry client design requires extra care and attention to avoid creating issues for the codebases which depend upon these cross-cutting APIs.

每个signal为软件提供了一种描述自身的机制。代码库，例如 Web 框架或数据库客户端，依赖于各种signal来描述自身。然后可以将 OpenTelemetry instrumentation 代码可以被混合到该代码库中的其他代码中。
这使得 OpenTelemetry 成为一个横切关注点——一种混合到许多其他软件中以提供价值的软件。横切关注点，就其本质而言，违反了核心设计原则——关注点分离。所以，OpenTelemetry 客户端设计需要格外小心和注意，以避免为依赖于这些横切 API 的代码库产生问题。

>这里的描述自己指的是 自己的运行时间，运行状态，消耗内存等

>“cross-cutting concerns” 横切关注点：指的是两个非常不一样的组件存在一些类似的功能

OpenTelemetry clients are designed to separate the portion of each signal which must be imported as cross-cutting concerns from the portions which can be managed independently. OpenTelemetry clients are also designed to be an extensible framework.
To accomplish these goals, each signal consists of four types of packages: API, SDK, Semantic Conventions, and Contrib.

OpenTelemetry 客户端旨在将每个signal中必须作为横切关注点导入的部分与可以独立管理的部分分开。OpenTelemetry 客户端也被设计成一个可扩展的框架.为了实现这些目标，每个signal由四种类型的包组成：API、SDK、Semantic Convention（语义约定）和 Contrib。

### API

API packages consist of the cross-cutting public interfaces used for instrumentation. Any portion of an OpenTelemetry client which is imported into third-party libraries and application code is considered part of the API.

API 包 包括 ： 用于 instrumentation 的横切公共接口 。OpenTelemetry 客户端 （导入的第三方库 和 应用程序代码 ） 的任何部分都被视为 API 的一部分。


### SDK

The SDK is the implementation of the API provided by the OpenTelemetry project. Within an application, the SDK is installed and managed by the [application owner](glossary.md#application-owner).
Note that the SDK includes additional public interfaces which are not considered part of the API package, as they are not cross-cutting concerns. These public interfaces are defined as [constructors](glossary.md#constructors) and [plugin interfaces](glossary.md#sdk-plugins).
Application owners use the SDK constructors; [plugin authors](glossary.md#plugin-author) use the SDK plugin interfaces.
[Instrumentation authors](glossary.md#instrumentation-author) MUST NOT directly reference any SDK package of any kind, only the API.

SDK是OpenTelemetry项目提供的API的实现，SDK由应用程序所有者安装和管理。请注意，SDK 包含其他公共接口，这些接口不被视为 API 包的一部分，因为它们不是横切关注点。这些公共接口 指的是 构造器 和插件接口。应用程序所有者使用 SDK 构造器；插件作者使用 SDK 插件接口。Instrumentation作者不得直接引用任何类型的任何 SDK 包，只能引用 API

### Semantic Conventions   语义约定


The **Semantic Conventions** define the keys and values which describe commonly observed concepts, protocols, and operations used by applications.
语义约定定义了描述应用程序使用的常见概念、协议和操作的键和值。

* [Resource Conventions](resource/semantic_conventions/README.md)  //TODO
* [Span Conventions](trace/semantic_conventions/README.md)    //TODO
* [Metrics Conventions](metrics/semantic_conventions/README.md)   //TODO

Both the collector and the client libraries SHOULD autogenerate semantic
convention keys and enum values into constants (or language idomatic
equivalent). Generated values shouldn't be distributed in stable packages
until semantic conventions are stable.
The [YAML](../semantic_conventions/README.md) files MUST be used as the
source of truth for generation. Each language implementation SHOULD
provide language-specific support to the
[code generator](https://github.com/open-telemetry/build-tools/tree/main/semantic-conventions#code-generator).

collector和客户端库都应该自动生成语义约定键和枚举值到常量（或语言惯用语等价方式）。在语义约定稳定之前，不应将生成的值分发到稳定的包中。YAML 文件必须用作生成的真实来源？。每个语言实现都应该为代码生成器提供特定于语言的支持？。

 

### Contrib Packages   Contrib包

The OpenTelemetry project maintains integrations with popular OSS projects which have been identified as important for observing modern web services.
Example API integrations include instrumentation for web frameworks, database clients, and message queues.
Example SDK integrations include plugins for exporting telemetry to popular analysis tools and telemetry storage systems.

OpenTelemetry 项目保持与流行的 OSS 项目的集成，这些项目已被确定为对观察现代 Web 服务很重要。
示例 API 集成包括 Web 框架、数据库客户端和消息队列的检测。
示例 SDK 集成包括用于将遥测数据导出到流行分析工具和遥测存储系统（telemetry storage systems）的插件。

Some plugins, such as OTLP Exporters and TraceContext Propagators, are required by the OpenTelemetry specification. These required plugins are included as part of the SDK.

OpenTelemetry 规范需要一些插件，例如 OTLP 导出器（Exporters）和 TraceContext 传播器（Propagators）。

Plugins and instrumentation packages which are optional and separate from the SDK are referred to as **Contrib** packages.
**API Contrib** refers to packages which depend solely upon the API; **SDK Contrib** refers to packages which also depend upon the SDK.

可选且与 SDK 分离的插件和检测包称为 Contrib 包。API Contrib 是指完全依赖 API 的包； SDK Contrib 是指也依赖于 SDK 的包。

The term Contrib specifically refers to the collection of plugins and instrumentation maintained by the OpenTelemetry project; it does not refer to third-party plugins hosted elsewhere.

### Versioning and Stability 版本控制和稳定性

OpenTelemetry values stability and backwards compatibility. Please see the [versioning and stability guide](./versioning-and-stability.md) for details.

OpenTelemetry 重视稳定性和向后兼容性。有关详细信息，请参阅版本控制和稳定性指南



## Tracing Signal

A distributed trace is a set of events, triggered as a result of a single
logical operation, consolidated across various components of an application. A
distributed trace contains events that cross process, network and security
boundaries. A distributed trace may be initiated when someone presses a button
to start an action on a website - in this example, the trace will represent
calls made between the downstream services that handled the chain of requests
initiated by this button being pressed.

distributed trace（分布式跟踪）是一组事件，由单个逻辑操作触发，跨应用程序的各个组件进行整合。分布式跟踪包含跨越进程、网络和安全边界的事件。当有人按下按钮在网站上开始操作时，可能会启动分布式跟踪 - 在此示例中，跟踪将表示下游服务之间进行的调用，这些服务处理由按下此按钮发起的请求链。

### Traces

**Traces** in OpenTelemetry are defined implicitly by their **Spans**. In
particular, a **Trace** can be thought of as a directed acyclic graph (DAG) of
**Spans**, where the edges between **Spans** are defined as parent/child
relationship.

OpenTelemetry 中的**跟踪（Traces）** 是由它们的 **Spans** 隐式定义的。特别是，可以将 Trace 视为 Span 的有向无环图 (DAG)，其中 Span 之间的边被定义为父/子关系。

For example, the following is an example **Trace** made up of 6 **Spans**:
例如，以下是由 6 个 Span 组成的示例 Trace：

```
Causal relationships between Spans in a single Trace
单个 Trace 中 Span 之间的因果关系

        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C is a `child` of Span A)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F]
```

Sometimes it's easier to visualize **Traces** with a time axis as in the diagram
below:
有时使用时间轴可视化跟踪更容易，如下图所示：

```
Temporal relationships between Spans in a single Trace

––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··············································]
      [Span D··········································]
    [Span C········································]
         [Span E·······]        [Span F··]
```
一个trace相当于完整的调用链路，span相当于一部分

### Spans

A span represents an operation within a transaction. Each **Span** encapsulates
the following state:
span表示事务中的操作。每个 Span 封装了以下状态：

- An operation name  操作名
- A start and finish timestamp  开始结束时间戳
- [**Attributes**](./common/common.md#attributes): A list of key-value pairs.   键值对列表
- A set of zero or more **Events**, each of which is itself a tuple (timestamp, name, [**Attributes**](./common/common.md#attributes)). The name must be strings. 一组零个或多个Events，每个本身都是一个元组（时间戳、名称、Attributes）。名称必须是字符串。
- Parent's **Span** identifier.  父级的 Span 标识符
- [**Links**](#links-between-spans) to zero or more causally-related **Spans**  
  (via the **SpanContext** of those related **Spans**).   Links到零个或多个因果相关的 Span（通过这些相关 Span 的 SpanContext）。
- **SpanContext** information required to reference a Span. See below.  引用 Span 所需的 SpanContext 信息。见下文。

### SpanContext

Represents all the information that identifies **Span** in the **Trace** and
MUST be propagated to child Spans and across process boundaries. A
**SpanContext** contains the tracing identifiers and the options that are
propagated from parent to child **Spans**.

表示在 Trace 中标识 Span 的所有信息，并且必须传播到子 Span 并跨越进程边界。 SpanContext 包含跟踪标识符和从父级传播到子级 Span 的选项。

- **TraceId** is the identifier for a trace. It is worldwide unique with
  practically sufficient probability by being made as 16 randomly generated
  bytes. TraceId is used to group all spans for a specific trace together across
  all processes.
  TraceId 是跟踪的标识符。它被制作为 16 个随机生成的字节，在世界范围内是独一无二的，具有足够的可能性。 TraceId 用于将所有进程的特定跟踪的所有跨度组合在一起。
- **SpanId** is the identifier for a span. It is globally unique with
  practically sufficient probability by being made as 8 randomly generated
  bytes. When passed to a child Span this identifier becomes the parent span id
  for the child **Span**.
  SpanId 是span的标识符。通过将其生成为 8 个随机生成的字节，它是全局唯一的，具有实际足够的可能性。当传递给子span时，此标识符成为子span的父span ID。
- **TraceFlags** represents the options for a trace. It is represented as 1
  byte (bitmap).TraceFlags 表示trace的选项。它表示为 1 个字节（位图）
  - Sampling bit -  Bit to represent whether trace is sampled or not (mask
    `0x1`).
     采样位 - 表示是否对trace进行采样的位（掩码 0x1）。
- **Tracestate** carries tracing-system specific context in a list of key value
  pairs. **Tracestate** allows different vendors propagate additional
  information and inter-operate with their legacy Id formats. For more details
  see [this](https://w3c.github.io/trace-context/#tracestate-field).
  Tracestate 在键值对列表中携带tracing-system（跟踪系统）特定的上下文。 Tracestate 允许不同的供应商传播额外的信息并与其旧的 Id 格式进行互操作。有关更多详细信息，请参阅此。

### Links between spans

A **Span** may be linked to zero or more other **Spans** (defined by
**SpanContext**) that are causally related. **Links** can point to
**Spans** inside a single **Trace** or across different **Traces**.
**Links** can be used to represent batched operations where a **Span** was
initiated by multiple initiating **Spans**, each representing a single incoming
item being processed in the batch.

一个 Span 可以链接到零个或多个其他有因果关系的 Span（由 SpanContext 定义）。Links 可以指向单个Trace内 或 跨不同Trace 的Spans 。Links 可用于表示批处理操作，其中一个 Span 由多个启动 Span 启动，每个 Span 表示正在批处理中处理的单个传入项目。

Another example of using a **Link** is to declare the relationship between
the originating and following trace. This can be used when a **Trace** enters trusted
boundaries of a service and service policy requires the generation of a new
Trace rather than trusting the incoming Trace context. The new linked Trace may
also represent a long running asynchronous data processing operation that was
initiated by one of many fast incoming requests.

使用Link的另一个示例是声明原始trace和后续trace之间的关系。当 Trace 进入服务的可信边界并且服务策略需要生成新 Trace 而不是信任传入的 Trace 上下文时，可以使用此方法。新链接的 Trace 也可能表示由许多快速传入请求之一启动的长时间运行的异步数据处理操作。

When using the scatter/gather (also called fork/join) pattern, the root
operation starts multiple downstream processing operations and all of them are
aggregated back in a single **Span**. This last **Span** is linked to many
operations it aggregates. All of them are the **Spans** from the same Trace. And
similar to the Parent field of a **Span**. It is recommended, however, to not
set parent of the **Span** in this scenario as semantically the parent field
represents a single parent scenario, in many cases the parent **Span** fully
encloses the child **Span**. This is not the case in scatter/gather and batch
scenarios.

使用 scatter/gather（也称为 fork/join）模式时，根操作会启动多个下游处理操作，并且所有这些操作都会聚合回单个 Span。最后一个 Span 与其聚合的许多操作相关联。它们都是来自同一 Trace 的 Span。类似于 Span 的 Parent 字段。但是，建议不要在此场景中设置 Span 的父级，因为在语义上父字段表示单个父级场景，在许多情况下，父级 Span 完全包含子级 Span。在分散/聚集和批处理场景中，情况并非如此。

## Metric Signal

OpenTelemetry allows to record raw measurements or metrics with predefined
aggregation and a [set of attributes](./common/common.md#attributes).

OpenTelemetry 允许使用预定义的聚合和一组属性记录原始测量值或指标。

Recording raw measurements using OpenTelemetry API allows to defer to end-user
the decision on what aggregation algorithm should be applied for this metric as
well as defining attributes (dimensions). It will be used in client libraries like
gRPC to record raw measurements "server_latency" or "received_bytes". So end
user will decide what type of aggregated values should be collected out of these
raw measurements. It may be simple average or elaborate histogram calculation.

使用 OpenTelemetry API 记录原始测量结果允许将最终用户关于应为该指标应用哪种聚合算法以及定义属性（维度）的决定推迟。它将在 gRPC 等客户端库中用于记录原始测量结果“server_latency”或“received_bytes”。因此，最终用户将决定应从这些原始测量中收集何种类型的聚合值。它可能是简单的平均或复杂的直方图计算。

Recording of metrics with the pre-defined aggregation using OpenTelemetry API is
not less important. It allows to collect values like cpu and memory usage, or
simple metrics like "queue length".

使用 OpenTelemetry API 通过预定义聚合记录指标同样重要。它允许收集诸如 cpu 和内存使用率之类的值，或诸如“队列长度”之类的简单指标。

### Recording raw measurements   记录原始测量值

The main classes used to record raw measurements are `Measure` and
`Measurement`. List of `Measurement`s alongside the additional context can be
recorded using OpenTelemetry API. So user may define to aggregate those
`Measurement`s and use the context passed alongside to define additional
dimensions of the resulting metric.

用于记录原始测量值的主要类是 Measure 和 Measurement。可以使用 OpenTelemetry API 记录与附加上下文一起的Measurement列表。因此，用户可以定义聚合这些度量，并使用随同传递的上下文来定义结果度量的其他维度。

#### Measure

`Measure` describes the type of the individual values recorded by a library. It
defines a contract between the library exposing the measurements and an
application that will aggregate those individual measurements into a `Metric`.
`Measure` is identified by name, description and a unit of values.

度量（Measure）  描述了库记录的单个值的类型。它定义了公开测量值的库和将这些单个测量值聚合为 指标（ `Metric` ）的应用程序之间的契约。度量 `Measure`  由名称、描述和值单位标识。

#### Measurement

`Measurement` describes a single value to be collected for a `Measure`.
`Measurement` is an empty interface in API surface. This interface is defined in
SDK.

`Measurement` 描述了要为`Measure`收集的单个值。`Measurement`是 API 表面中的一个空接口。该接口在 SDK 中定义。

### Recording metrics with predefined aggregation  使用预定义聚合记录指标

The base class for all types of pre-aggregated metrics is called `Metric`. It
defines basic metric properties like a name and attributes. Classes inheriting from
the `Metric` define their aggregation type as well as a structure of individual
measurements or Points. API defines the following types of pre-aggregated
metrics:

所有类型的预聚合指标的基类称为 Metric。它定义了基本的度量属性，如名称和属性。从 Metric 继承的类定义了它们的聚合类型以及单个测量或点的结构。 API 定义了以下类型的预聚合指标：

- Counter metric to report instantaneous measurement. Counter values can go
  up or stay the same, but can never go down. Counter values cannot be
  negative. There are two types of counter metric values - `double` and `long`.
  
   （Counter metric ）用于报告瞬时测量值的。Counter值可以上升或保持不变，但永远不会下降。Counter值不能为负。有两种类型的计数器度量值 - double 和 long。
   
- Gauge metric to report instantaneous measurement of a numeric value. Gauges can
  go both up and down. The gauges values can be negative. There are two types of
  gauge metric values - `double` and `long`.
  
  （Gauge metric ）用于报告数值的瞬时测量值 。Gauges可以上升也可以下降。Gauges值可以为负。有两种类型的Gauges度量值 - double 和 long。

API allows to construct the `Metric` of a chosen type. SDK defines the way to
query the current value of a `Metric` to be exported.

API 允许构建所选类型的  `Metric`（指标）。 SDK 定义了查询要导出的 Metric 的当前值的方式。

Every type of a `Metric` has it's API to record values to be aggregated. API
supports both - push and pull model of setting the `Metric` value.

每种类型的`Metric`（指标）都有其 API 来记录要聚合的值。 API 支持设置 Metric 值的推和拉模型。

### Metrics data model and SDK

Metrics data model is [specified here](metrics/datamodel.md) and is based on
[metrics.proto](https://github.com/open-telemetry/opentelemetry-proto/blob/master/opentelemetry/proto/metrics/v1/metrics.proto).
This data model defines three semantics: An Event model used by the API, an
in-flight data model used by the SDK and OTLP, and a TimeSeries model which
denotes how exporters should interpret the in-flight model.

此处指定了指标数据模型，该模型基于 metrics.proto。该数据模型定义了三种语义：API 使用的事件模型、SDK 和 OTLP 使用的动态数据模型以及表示导出器应如何解释动态模型的 TimeSeries 模型。

Different exporters have different capabilities (e.g. which data types are
supported) and different constraints (e.g. which characters are allowed in attribute
keys). Metrics is intended to be a superset of what's possible, not a lowest
common denominator that's supported everywhere. All exporters consume data from
Metrics Data Model via a Metric Producer interface defined in OpenTelemetry SDK.

不同的导出器（exporters）具有不同的功能（例如支持哪些数据类型）和不同的约束（例如属性键中允许哪些字符）。Metrics 旨在成为可能的超集，而不是任何地方都支持的最低。所有导出器（exporters）都通过 OpenTelemetry SDK 中定义的 Metric Producer 接口使用来自 Metrics Data Model 的数据。

Because of this, Metrics puts minimal constraints on the data (e.g. which
characters are allowed in keys), and code dealing with Metrics should avoid
validation and sanitization of the Metrics data. Instead, pass the data to the
backend, rely on the backend to perform validation, and pass back any errors
from the backend.

因此，Metrics 对数据的限制最小（例如，键中允许使用哪些字符），并且处理 Metrics 的代码应避免对 Metrics 数据进行验证和清理。相反，将数据传递给后端，依靠后端执行验证，并将任何错误从后端传回。

See [Metrics Data Model Specification](metrics/datamodel.md) for more
information.

## Log Signal

### Data model

[Log Data Model](logs/data-model.md) defines how logs and events are understood by
OpenTelemetry.

日志数据模型定义了 OpenTelemetry 如何理解日志和事件。


## Baggage Signal

In addition to trace propagation, OpenTelemetry provides a simple mechanism for propagating
name/value pairs, called `Baggage`. `Baggage` is intended for
indexing observability events in one service with attributes provided by a prior service in
the same transaction. This helps to establish a causal relationship between these events.

除了trace propagation之外，OpenTelemetry 还提供了一种用于传播名称/值对的简单机制，称为 Baggage。 Baggage 用于索引一项服务中的可观察性事件，其属性由同一事务中的先前服务提供。这有助于在这些事件之间建立因果关系。

While `Baggage` can be used to prototype other cross-cutting concerns, this mechanism is primarily intended
to convey values for the OpenTelemetry observability systems.

虽然 Baggage 可用于构建其他横切关注点的原型，但该机制主要用于传达 OpenTelemetry 可观察性系统的值。

These values can be consumed from `Baggage` and used as additional dimensions for metrics,
or additional context for logs and traces. Some examples:
这些值可以从 Baggage 中使用，并用作指标的附加维度，或日志和跟踪的附加上下文。一些例子：

- a web service can benefit from including context around what service has sent the request
-  Web 服务可以从包含发送请求的服务的上下文中受益
- a SaaS provider can include context about the API user or token that is responsible for that request
- SaaS 提供商可以包含有关负责该请求的 API 用户或令牌的上下文
- determining that a particular browser version is associated with a failure in an image processing service
- 确定特定浏览器版本与图像处理服务中的故障相关联

For backward compatibility with OpenTracing, Baggage is propagated as `Baggage` when
using the OpenTracing bridge. New concerns with different criteria should consider creating a new
cross-cutting concern to cover their use-case; they may benefit from the W3C encoding format but
use a new HTTP header to convey data throughout a distributed trace.

为了与 OpenTracing 向后兼容，当使用 OpenTracing bridge时，Baggage 作为 Baggage 传播。具有不同标准的 cross-cutting   应考虑创建一个新的跨领域关注点以涵盖其用例；它们可能受益于 W3C 编码格式，但使用新的 HTTP 标头在整个分布式跟踪中传送数据。

## Resources

`Resource` captures information about the entity for which telemetry is
recorded. For example, metrics exposed by a Kubernetes container can be linked
to a resource that specifies the cluster, namespace, pod, and container name.

`Resource` 捕获有关记录遥测的实体的信息。例如，Kubernetes 容器公开的指标可以链接到指定集群、命名空间、pod 和容器名称的资源。

`Resource` may capture an entire hierarchy of entity identification. It may
describe the host in the cloud and specific container or an application running
in the process.

`Resource`可以捕获实体标识的整个层次结构。它可以描述云中的主机和特定的容器或进程中运行的应用程序。

Note, that some of the process identification information can be associated with
telemetry automatically by OpenTelemetry SDK or specific exporter. See
OpenTelemetry
[proto](https://github.com/open-telemetry/opentelemetry-proto/blob/a46c815aa5e85a52deb6cb35b8bc182fb3ca86a0/src/opentelemetry/proto/agent/common/v1/common.proto#L28-L96)
for an example.

请注意，某些进程标识信息可以通过 OpenTelemetry SDK 或特定导出器自动与遥测相关联。有关示例，请参阅 OpenTelemetry proto。

## Context Propagation  上下文传播


All of OpenTelemetry cross-cutting concerns, such as traces and metrics,
share an underlying `Context` mechanism for storing state and
accessing data across the lifespan of a distributed transaction.

所有 OpenTelemetry 横切关注点（例如 跟踪和度量 traces and metrics）共享底层 Context 机制，用于在分布式事务的整个生命周期中存储状态和访问数据。

See the [Context](context/context.md)

## Propagators 传播者

OpenTelemetry uses `Propagators` to serialize and deserialize cross-cutting concern values
such as `Span`s (usually only the `SpanContext` portion) and `Baggage`. Different `Propagator` types define the restrictions
imposed by a specific transport and bound to a data type.

OpenTelemetry 使用 Propagators 序列化和反序列化横切关注值，例如 Span（通常只有 SpanContext 部分）和 Baggage。不同的Propagator类型定义了由特定传输施加并绑定到数据类型的限制。

The Propagators API currently defines one `Propagator` type: 

Propagators API 当前定义了一种 Propagator 类型：

- `TextMapPropagator` injects values into and extracts values from carriers as text.

-   TextMapPropagator 将值注入到载体中并从载体中提取值作为文本。

## Collector 


The OpenTelemetry collector is a set of components that can collect traces,
metrics and eventually other telemetry data (e.g. logs) from processes
instrumented by OpenTelemetry or other monitoring/tracing libraries (Jaeger,
Prometheus, etc.), do aggregation and smart sampling, and export traces and
metrics to one or more monitoring/tracing backends. The collector will allow to
enrich and transform collected telemetry (e.g. add additional attributes or
scrub personal information).

OpenTelemetry collector是一组组件，可以从 OpenTelemetry 或其他监控/跟踪库（Jaeger、Prometheus 等）检测的进程中收集traces,metrics和最终其他遥测（telemetry）数据（例如日志），进行聚合和智能采样，以及将traces and metrics导出到一个或多个监控/跟踪后端。收集器将允许丰富和转换收集到的遥测数据（例如添加附加属性或清理个人信息）。

The OpenTelemetry collector has two primary modes of operation: Agent (a daemon
running locally with the application) and Collector (a standalone running
service).

OpenTelemetry collector有两种主要的操作模式：代理（与应用程序一起在本地运行的守护进程）和收集器（独立运行的服务）。

Read more at OpenTelemetry Service [Long-term
Vision](https://github.com/open-telemetry/opentelemetry-collector/blob/master/docs/vision.md).

## Instrumentation Libraries

See [Instrumentation Library](glossary.md#instrumentation-library)

The inspiration of the project is to make every library and application
observable out of the box by having them call OpenTelemetry API directly. However,
many libraries will not have such integration, and as such there is a need for
a separate library which would inject such calls, using mechanisms such as
wrapping interfaces, subscribing to library-specific callbacks, or translating
existing telemetry into the OpenTelemetry model.

该项目的灵感是通过让它们直接调用 OpenTelemetry API 来使每个库和应用程序开箱即用。但是，许多库不会有这样的集成，因此需要一个单独的库来注入此类调用，使用诸如包装接口、订阅特定于库的回调或将现有遥测数据转换为 OpenTelemetry 模型等机制。

A library that enables OpenTelemetry observability for another library is called
an [Instrumentation Library](glossary.md#instrumentation-library).

为另一个库启用 OpenTelemetry 可观察性的库称为Instrumentation库。
 
An instrumentation library should be named to follow any naming conventions of
the instrumented library (e.g. 'middleware' for a web framework).

应命名 instrumentation库 以遵循 instrumented 库的任何命名约定（例如，Web 框架的“中间件”）。

If there is no established name, the recommendation is to prefix packages
with "opentelemetry-instrumentation", followed by the instrumented library
name itself. Examples include:

如果没有确定的名称，建议使用“opentelemetry-instrumentation”作为包的前缀，然后是instrumented库名称本身。例子包括：

* opentelemetry-instrumentation-flask (Python)
* @opentelemetry/instrumentation-grpc (Javascript)
