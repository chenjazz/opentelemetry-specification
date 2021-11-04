# Propagators API

**Status**: [Stable, Feature-Freeze](../document-status.md)

<details>
<summary>
Table of Contents
</summary>

- [Overview](#overview)
- [Propagator Types](#propagator-types)
  * [Carrier](#carrier)
  * [Operations](#operations)
    + [Inject](#inject)
    + [Extract](#extract)
- [TextMap Propagator](#textmap-propagator)
  * [Fields](#fields)
  * [TextMap Inject](#textmap-inject)
    + [Setter argument](#setter-argument)
      - [Set](#set)
  * [TextMap Extract](#textmap-extract)
    + [Getter argument](#getter-argument)
      - [Keys](#keys)
      - [Get](#get)
- [Injectors and Extractors as Separate Interfaces](#injectors-and-extractors-as-separate-interfaces)
- [Composite Propagator](#composite-propagator)
  * [Create a Composite Propagator](#create-a-composite-propagator)
  * [Composite Extract](#composite-extract)
  * [Composite Inject](#composite-inject)
- [Global Propagators](#global-propagators)
  * [Get Global Propagator](#get-global-propagator)
  * [Set Global Propagator](#set-global-propagator)
- [Propagators Distribution](#propagators-distribution)
  * [B3 Requirements](#b3-requirements)
    + [B3 Extract](#b3-extract)
    + [B3 Inject](#b3-inject)

</details>

## Overview

Cross-cutting concerns send their state to the next process using
`Propagator`s, which are defined as objects used to read and write
context data to and from messages exchanged by the applications.
Each concern creates a set of `Propagator`s for every supported
`Propagator` type.

横切关注点使用传播器将它们的状态发送到下一个进程，传播器被定义为用于从应用程序交换的消息读取和写入上下文数据的对象。每个关注点为每个支持的传播器类型创建一组传播器。

`Propagator`s leverage the `Context` to inject and extract data for each
cross-cutting concern, such as traces and `Baggage`.

传播器利用上下文为每个横切关注点注入和提取数据，例如traces 和 `Baggage`。

Propagation is usually implemented via a cooperation of library-specific request
interceptors and `Propagators`, where the interceptors detect incoming and outgoing requests and use the `Propagator`'s extract and inject operations respectively.

传播通常通过库特定的请求拦截器和传播器的合作来实现，其中拦截器检测传入和传出的请求，并分别使用传播器的提取和注入操作。

The Propagators API is expected to be leveraged by users writing
instrumentation libraries.

Propagators API 预计将被编写instrumentation库的用户所利用。

## Propagator Types  传播器类型


A `Propagator` type defines the restrictions imposed by a specific transport
and is bound to a data type, in order to propagate in-band context data across process boundaries.

传播器类型定义了特定传输所施加的限制并绑定到数据类型，以便跨进程边界传播 带内 上下文数据。


The Propagators API currently defines one `Propagator` type:Propagators API 当前定义了一种 Propagator 类型：


- `TextMapPropagator` is a type that inject values into and extracts values
  from carriers as string key/value pairs.  TextMapPropagator 是一种将值作为字符串键/值对从载体中注入和提取的类型。

A binary `Propagator` type will be added in the future (see [#437](https://github.com/open-telemetry/opentelemetry-specification/issues/437)).将来会添加二进制传播器类型（请参阅#437）。

### Carrier  载体


A carrier is the medium used by `Propagator`s to read values from and write values to.
Each specific `Propagator` type defines its expected carrier type, such as a string map
or a byte array.

载体是传播器用来读取和写入值的介质。每个特定的传播器类型定义其预期的载体类型，例如字符串映射或字节数组。

Carriers used at [Inject](#inject) are expected to be mutable.Inject 中使用的载体预计是可变的。

### Operations 操作


`Propagator`s MUST define `Inject` and `Extract` operations, in order to write
values to and read values from carriers respectively. Each `Propagator` type MUST define the specific carrier type
and MAY define additional parameters.

传播者必须定义注入和提取操作，以便分别向载体写入值和从载体读取值。每个传播器类型必须定义特定的载波类型并且可以定义附加参数。

#### Inject 注入

Injects the value into a carrier. For example, into the headers of an HTTP request. 

将值注入载体。例如，放入 HTTP 请求的标头。

Required arguments:

- A `Context`. The Propagator MUST retrieve the appropriate value from the `Context` first, such as
`SpanContext`, `Baggage` or another cross-cutting concern context.传播器必须首先从上下文中检索适当的值，例如 SpanContext、Baggage 或其他横切关注上下文。
- The carrier that holds the propagation fields. For example, an outgoing message or HTTP request.承载传播场的载体。例如，传出消息或 HTTP 请求。

#### Extract 提取

Extracts the value from an incoming request. For example, from the headers of an HTTP request.

从传入请求中提取值。例如，来自 HTTP 请求的标头。

If a value can not be parsed from the carrier, for a cross-cutting concern,
the implementation MUST NOT throw an exception and MUST NOT store a new value in the `Context`,
in order to preserve any previously existing valid value.

如果无法从载体解析值，对于横切关注点，实现不得抛出异常并且不得在上下文中存储新值，以保留任何先前存在的有效值。


Required arguments:

- A `Context`.
- The carrier that holds the propagation fields. For example, an incoming message or HTTP request.

Returns a new `Context` derived from the `Context` passed as argument,
containing the extracted value, which can be a `SpanContext`,
`Baggage` or another cross-cutting concern context.

返回从作为参数传递的 Context 派生的新 Context，其中包含提取的值，可以是 SpanContext、Baggage 或其他横切关注上下文。

## TextMap Propagator

`TextMapPropagator` performs the injection and extraction of a cross-cutting concern
value as string key/values pairs into carriers that travel in-band across process boundaries.

TextMapPropagator 执行将横切关注值作为字符串键/值对注入和提取到跨进程边界带内传输的载体中。

The carrier of propagated data on both the client (injector) and server (extractor) side is
usually an HTTP request.

客户端（注入器）和服务器（提取器）端传播数据的载体通常是 HTTP 请求。

In order to increase compatibility, the key/value pairs MUST only consist of US-ASCII characters
that make up valid HTTP header fields as per [RFC 7230](https://tools.ietf.org/html/rfc7230#section-3.2).

为了提高兼容性，键/值对必须仅包含构成符合 RFC 7230 的有效 HTTP 标头字段的 US-ASCII 字符。

`Getter` and `Setter` are optional helper components used for extraction and injection respectively,
and are defined as separate objects from the carrier to avoid runtime allocations,
by removing the need for additional interface-implementing-objects wrapping the carrier in order
to access its contents.

Getter 和 Setter 是分别用于提取和注入的可选辅助组件，它们被定义为与载体分离的对象，以避免运行时分配，通过消除包装载体的额外接口实现对象以访问其内容的需要。

`Getter` and `Setter` MUST be stateless and allowed to be saved as constants, in order to effectively
avoid runtime allocations.

Getter 和 Setter 必须是无状态的并允许保存为常量，以有效避免运行时分配。

### Fields 字段

The predefined propagation fields. If your carrier is reused, you should delete the fields here
before calling [inject](#inject).

预定义的传播字段。如果你的carrier被重复使用，你应该在调用注入之前删除这里的字段。

Fields are defined as string keys identifying format-specific components in a carrier.

字段被定义为标识载体中特定格式组件的字符串键。

For example, if the carrier is a single-use or immutable request object, you don't need to
clear fields as they couldn't have been set before. If it is a mutable, retryable object,
successive calls should clear these fields first.

例如，如果carrier是一次性或不可变的请求对象，则您不需要清除字段，因为它们之前无法设置。如果它是一个可变的、可重试的对象，连续调用应该首先清除这些字段。

The use cases of this are:它的用例是：


- allow pre-allocation of fields, especially in systems like gRPC Metadata  允许预先分配字段，尤其是在 gRPC 元数据等系统中
- allow a single-pass over an iterator  允许单次遍历迭代器


Returns list of fields that will be used by the `TextMapPropagator`.  返回将由 TextMapPropagator 使用的字段列表

Observe that some `Propagator`s may define, besides the returned values, additional fields with
variable names. To get a full list of fields for a specific carrier object, use the
[Keys](#keys) operation.

返回将由 TextMapPropagator 使用的字段列表

### TextMap Inject    TextMap注入

Injects the value into a carrier. The required arguments are the same as defined by
the base [Inject](#inject) operation.  将值注入载体。所需的参数与基本注入操作定义的相同。

Optional arguments: 可选参数：


- A `Setter` to set a propagation key/value pair. Propagators MAY invoke it multiple times in order to set multiple pairs.
  This is an additional argument that languages are free to define to help inject data into the carrier.
  
  设置传播键/值对的 Setter。传播者可以多次调用它以设置多个对。这是语言可以自由定义的附加参数，以帮助将数据注入载体。

#### Setter argument

Setter is an argument in `Inject` that sets values into given fields.  Setter 是 Inject 中的一个参数，用于将值设置到给定的字段中。

`Setter` allows a `TextMapPropagator` to set propagated fields into a carrier.  Setter 允许 TextMapPropagator 将传播的字段设置到载体中。

One of the ways to implement it is `Setter` class with `Set` method as described below.  实现它的方法之一是带有 Set 方法的 Setter 类，如下所述。

##### Set

Replaces a propagated field with the given value.  用给定的值替换传播的字段。

Required arguments:

- the carrier holding the propagation fields. For example, an outgoing message or an HTTP request.  承载传播场的载体。例如，传出消息或 HTTP 请求。
- the key of the field.
- the value of the field.

The implementation SHOULD preserve casing (e.g. it should not transform `Content-Type` to `content-type`) if the used protocol is case insensitive, otherwise it MUST preserve casing.

如果使用的协议不区分大小写，则实现应该保留大小写（例如，它不应该将 Content-Type 转换为content-type ），否则它必须保留大小写。

### TextMap Extract  提取

Extracts the value from an incoming request. The required arguments are the same as defined by
the base [Extract](#extract) operation.

从传入请求中提取值。所需的参数与基本提取操作定义的相同。

Optional arguments:

- A `Getter` invoked for each propagation key to get. This is an additional
  argument that languages are free to define to help extract data from the carrier.
  
  为每个要获取的传播键调用一个 Getter。这是一个额外的参数，语言可以自由定义以帮助从载体中提取数据

Returns a new `Context` derived from the `Context` passed as argument.返回从作为参数传递的上下文派生的新上下文。

#### Getter argument

Getter is an argument in `Extract` that get value from given field  

Getter 是 Extract 中的一个参数，用于从给定字段中获取值

`Getter` allows a `TextMapPropagator` to read propagated fields from a carrier.

Getter 允许 TextMapPropagator 从载体读取传播的字段。

One of the ways to implement it is `Getter` class with `Get` and `Keys` methods
as described below. Languages may decide on alternative implementations and
expose corresponding methods as delegates or other ways.

实现它的方法之一是具有 Get 和 Keys 方法的 Getter 类，如下所述。语言可以决定替代实现并将相应的方法公开为委托或其他方式。

##### Keys

The `Keys` function MUST return the list of all the keys in the carrier.  Keys 函数必须返回载体中所有键的列表。

Required arguments:

- The carrier of the propagation fields, such as an HTTP request.

The `Keys` function can be called by `Propagator`s which are using variable key names in order to
iterate over all the keys in the specified carrier.

For example, it can be used to detect all keys following the `uberctx-{user-defined-key}` pattern, as defined by the
[Jaeger Propagation Format](https://www.jaegertracing.io/docs/1.18/client-libraries/#baggage).

##### Get

The Get function MUST return the first value of the given propagation key or return null if the key doesn't exist.

Get 函数必须返回给定传播键的第一个值，如果键不存在则返回 null。

Required arguments:

- the carrier of propagation fields, such as an HTTP request.
- the key of the field.

The Get function is responsible for handling case sensitivity. If the getter is intended to work with a HTTP request object, the getter MUST be case insensitive.

## Injectors and Extractors as Separate Interfaces  作为独立接口的注射器和提取器

Languages can choose to implement a `Propagator` type as a single object
exposing `Inject` and `Extract` methods, or they can opt to divide the
responsibilities further into individual `Injector`s and `Extractor`s. A
`Propagator` can be implemented by composing individual `Injector`s and
`Extractors`.

语言可以选择将 Propagator 类型实现为暴露 Inject 和 Extract 方法的单个对象，或者他们可以选择将职责进一步划分为单独的 Injector 和 Extractor。传播器可以通过组合单独的注入器和提取器来实现。

## Composite Propagator

Implementations MUST offer a facility to group multiple `Propagator`s
from different cross-cutting concerns in order to leverage them as a
single entity.

A composite propagator can be built from a list of propagators, or a list of
injectors and extractors. The resulting composite `Propagator` will invoke the `Propagator`s, `Injector`s, or `Extractor`s, in the order they were specified.

Each composite `Propagator` will implement a specific `Propagator` type, such
as `TextMapPropagator`, as different `Propagator` types will likely operate on different
data types.

There MUST be functions to accomplish the following operations.

- Create a composite propagator
- Extract from a composite propagator
- Inject into a composite propagator

### Create a Composite Propagator

Required arguments:

- A list of `Propagator`s or a list of `Injector`s and `Extractor`s.

Returns a new composite `Propagator` with the specified `Propagator`s.

### Composite Extract

Required arguments:

- A `Context`.
- The carrier that holds propagation fields.

If the `TextMapPropagator`'s `Extract` implementation accepts the optional `Getter` argument, the following arguments are REQUIRED, otherwise they are OPTIONAL:

- The instance of `Getter` invoked for each propagation key to get.

### Composite Inject

Required arguments:

- A `Context`.
- The carrier that holds propagation fields.

If the `TextMapPropagator`'s `Inject` implementation accepts the optional `Setter` argument, the following arguments are REQUIRED, otherwise they are OPTIONAL:

- The `Setter` to set a propagation key/value pair. Propagators MAY invoke it multiple times in order to set multiple pairs.

## Global Propagators

The OpenTelemetry API MUST provide a way to obtain a propagator for each
supported `Propagator` type. Instrumentation libraries SHOULD call propagators
to extract and inject the context on all remote calls. Propagators, depending on
the language, MAY be set up using various dependency injection techniques or
available as global accessors.

**Note:** It is a discouraged practice, but certain instrumentation libraries
might use proprietary context propagation protocols or be hardcoded to use a
specific one. In such cases, instrumentation libraries MAY choose not to use the
API-provided propagators and instead hardcode the context extraction and injection
logic.

The OpenTelemetry API MUST use no-op propagators unless explicitly configured
otherwise. Context propagation may be used for various telemetry signals -
traces, metrics, logging and more. Therefore, context propagation MAY be enabled
for any of them independently. For instance, a span exporter may be left
unconfigured, although the trace context propagation was configured to enrich logs or metrics.

Platforms such as ASP.NET may pre-configure out-of-the-box
propagators. If pre-configured, `Propagator`s SHOULD default to a composite
`Propagator` containing the W3C Trace Context Propagator and the Baggage
`Propagator` specified in the [Baggage API](../baggage/api.md#propagation).
These platforms MUST also allow pre-configured propagators to be disabled or overridden.

### Get Global Propagator

This method MUST exist for each supported `Propagator` type.

Returns a global `Propagator`. This usually will be composite instance.

### Set Global Propagator

This method MUST exist for each supported `Propagator` type.

Sets the global `Propagator` instance.

Required parameters:

- A `Propagator`. This usually will be a composite instance.

## Propagators Distribution

The official list of propagators that MUST be maintained by the OpenTelemetry
organization and MUST be distributed as OpenTelemetry extension packages:

* [W3C TraceContext](https://www.w3.org/TR/trace-context/). MAY alternatively
  be distributed as part of the OpenTelemetry API.
* [W3C Baggage](https://w3c.github.io/baggage). MAY alternatively
  be distributed as part of the OpenTelemetry API.
* [B3](https://github.com/openzipkin/b3-propagation).
* [Jaeger](https://www.jaegertracing.io/docs/latest/client-libraries/#propagation-format).

This is a list of additional propagators that MAY be maintained and distributed
as OpenTelemetry extension packages:

* [OT Trace](https://github.com/opentracing?q=basic&type=&language=). Propagation format
  used by the OpenTracing Basic Tracers. It MUST NOT use `OpenTracing` in the resulting
  propagator name as it is not widely adopted format in the OpenTracing ecosystem.

Additional `Propagator`s implementing vendor-specific protocols such as AWS
X-Ray trace header protocol MUST NOT be maintained or distributed as part of
the Core OpenTelemetry repositories.

### B3 Requirements

B3 has both single and multi-header encodings. It also has semantics that do not
map directly to OpenTelemetry such as a debug trace flag, and allowing spans
from both sides of request to share the same id. To maximize compatibility
between OpenTelemetry and Zipkin implementations, the following guidelines have
been established for B3 context propagation.

#### B3 Extract

When extracting B3, propagators:

* MUST attempt to extract B3 encoded using single and multi-header
  formats. The single-header variant takes precedence over
  the multi-header version.
* MUST preserve a debug trace flag, if received, and propagate
  it with subsequent requests. Additionally, an OpenTelemetry implementation
  MUST set the sampled trace flag when the debug flag is set.
* MUST NOT reuse `X-B3-SpanId` as the id for the server-side span.

#### B3 Inject

When injecting B3, propagators:

* MUST default to injecting B3 using the single-header format
* MUST provide configuration to change the default injection format to B3
  multi-header
* MUST NOT propagate `X-B3-ParentSpanId` as OpenTelemetry does not support
  reusing the same id for both sides of a request.

#### Fields

Fields MUST return the header names that correspond to the configured format,
i.e., the headers used for the inject operation.

#### Configuration

| Option    | Extract Order | Inject Format | Specification     |
|-----------|---------------|---------------| ------------------|
| B3 Single | Single, Multi | Single        | [Link][b3-single] |
| B3 Multi  | Single, Multi | Multi         | [Link][b3-multi]  |

[b3-single]: https://github.com/openzipkin/b3-propagation#single-header
[b3-multi]: https://github.com/openzipkin/b3-propagation#multiple-headers
