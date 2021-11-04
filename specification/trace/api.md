# Tracing API

**Status**: [Stable, Feature-freeze](../document-status.md)

<details>
<summary>
Table of Contents
</summary>

* [Data types](#data-types)
  * [Time](#time)
    * [Timestamp](#timestamp)
    * [Duration](#duration)
* [TracerProvider](#tracerprovider)
  * [TracerProvider operations](#tracerprovider-operations)
* [Context Interaction](#context-interaction)
* [Tracer](#tracer)
  * [Tracer operations](#tracer-operations)
* [SpanContext](#spancontext)
  * [Retrieving the TraceId and SpanId](#retrieving-the-traceid-and-spanid)
  * [IsValid](#isvalid)
  * [IsRemote](#isremote)
* [Span](#span)
  * [Span creation](#span-creation)
    * [Determining the Parent Span from a Context](#determining-the-parent-span-from-a-context)
    * [Specifying Links](#specifying-links)
  * [Span operations](#span-operations)
    * [Get Context](#get-context)
    * [IsRecording](#isrecording)
    * [Set Attributes](#set-attributes)
    * [Add Events](#add-events)
    * [Set Status](#set-status)
    * [UpdateName](#updatename)
    * [End](#end)
    * [Record Exception](#record-exception)
  * [Span lifetime](#span-lifetime)
  * [Wrapping a SpanContext in a Span](#wrapping-a-spancontext-in-a-span)
* [SpanKind](#spankind)
* [Concurrency](#concurrency)
* [Included Propagators](#included-propagators)

</details>

The Tracing API consist of these main classes:

- [`TracerProvider`](#tracerprovider) is the entry point of the API.  
  It provides access to `Tracer`s.  TracerProvider 是 API 的入口点。它提供了对 Tracer 的访问。
- [`Tracer`](#tracer) is the class responsible for creating `Span`s.  Tracer 是负责创建 Span 的类。
- [`Span`](#span) is the API to trace an operation. Span 是用于跟踪操作的 API。

## Data types

While languages and platforms have different ways of representing data,
this section defines some generic requirements for this API.

虽然语言和平台具有不同的数据表示方式，但本节定义了此 API 的一些通用要求。

### Time

OpenTelemetry can operate on time values up to nanosecond (ns) precision.
The representation of those values is language specific.

OpenTelemetry 可以对高达纳秒 (ns) 精度的时间值进行操作。这些值的表示是特定于语言的。

#### Timestamp

A timestamp is the time elapsed since the Unix epoch.

使用unix时间戳

* The minimal precision is milliseconds.  最小精度为毫秒。
* The maximal precision is nanoseconds.  最大精度为纳秒。


#### Duration

A duration is the elapsed time between two events.

持续时间是两个事件之间经过的时间。

* The minimal precision is milliseconds.
* The maximal precision is nanoseconds.

## TracerProvider

`Tracer`s can be accessed with a `TracerProvider`.

In implementations of the API, the `TracerProvider` is expected to be the
stateful object that holds any configuration.

在 API 的实现中，TracerProvider 应该是保存任何配置的有状态对象。


Normally, the `TracerProvider` is expected to be accessed from a central place.
Thus, the API SHOULD provide a way to set/register and access
a global default `TracerProvider`.

通常， TracerProvider 应该从一个中心位置访问。因此，API 应该提供一种设置/注册和访问全局默认 TracerProvider 的方法。

Notwithstanding any global `TracerProvider`, some applications may want to or
have to use multiple `TracerProvider` instances,
e.g. to have different configuration (like `SpanProcessor`s) for each
(and consequently for the `Tracer`s obtained from them),
or because its easier with dependency injection frameworks.
Thus, implementations of `TracerProvider` SHOULD allow creating an arbitrary
number of `TracerProvider` instances.

尽管有任何全局 TracerProvider，一些应用程序可能想要或必须使用多个 TracerProvider 实例，例如为每个（以及从它们获得的跟踪器）有不同的配置（如 SpanProcessors），或者因为依赖注入框架更容易。因此， TracerProvider 的实现应该允许创建任意数量的 TracerProvider 实例。


### TracerProvider operations

The `TracerProvider` MUST provide the following functions:

- Get a `Tracer`

TracerProvider 必须提供获取Tracer的方法

#### Get a Tracer

This API MUST accept the following parameters:

此 API 必须接受以下参数：


- `name` (required): This name must identify the [instrumentation library](../overview.md#instrumentation-libraries)
  (e.g. `io.opentelemetry.contrib.mongodb`).
  If an application or library has built-in OpenTelemetry instrumentation, both
  [Instrumented library](../glossary.md#instrumented-library) and
  [Instrumentation library](../glossary.md#instrumentation-library) may refer to the same library.
  In that scenario, the `name` denotes a module name or component name within that library
  or application.
  In case an invalid name (null or empty string) is specified, a working
  Tracer implementation MUST be returned as a fallback rather than returning
  null or throwing an exception, its `name` property SHOULD be set to an **empty** string,
  and a message reporting that the specified value is invalid SHOULD be logged.
  A library, implementing the OpenTelemetry API *may* also ignore this name and
  return a default instance for all calls, if it does not support "named"
  functionality (e.g. an implementation which is not even observability-related).
  A TracerProvider could also return a no-op Tracer here if application owners configure
  the SDK to suppress telemetry produced by this library.
- `version` (optional): Specifies the version of the instrumentation library (e.g. `1.0.0`).
- [since 1.4.0] `schema_url` (optional): Specifies the Schema URL that should be
  recorded in the emitted telemetry.
  
It is unspecified whether or under which conditions the same or different
`Tracer` instances are returned from this functions.

未指定是否或在何种条件下从此函数返回相同或不同的 Tracer 实例。


Implementations MUST NOT require users to repeatedly obtain a `Tracer` again
with the same name+version+schema_url to pick up configuration changes.
This can be achieved either by allowing to work with an outdated configuration or
by ensuring that new configuration applies also to previously returned `Tracer`s.

实现不得要求用户重复获取具有相同名称+版本+schema_url 的 Tracer 以获取配置更改。这可以通过允许使用过时的配置或通过确保新配置也适用于以前返回的 Tracer 来实现。

Note: This could, for example, be implemented by storing any mutable
configuration in the `TracerProvider` and having `Tracer` implementation objects
have a reference to the `TracerProvider` from which they were obtained. If
configuration must be stored per-tracer (such as disabling a certain tracer),
the tracer could, for example, do a look-up with its name+version+schema_url in
a map in the `TracerProvider`, or the `TracerProvider` could maintain a registry
of all returned `Tracer`s and actively update their configuration if it changes.

注意：例如，这可以通过在 TracerProvider 中存储任何可变配置并让 Tracer 实现对象具有对从中获取它们的 TracerProvider 的引用来实现。如果必须按跟踪器存储配置（例如禁用某个跟踪器），例如，跟踪器可以在 TracerProvider 的映射中使用其名称+版本+schema_url 进行查找，或者 TracerProvider 可以维护注册表所有返回的 Tracer 并在其配置发生变化时主动更新其配置。

The effect of associating a Schema URL with a `Tracer` MUST be that the
telemetry emitted using the `Tracer` will be associated with the Schema URL,
provided that the emitted data format is capable of representing such
association.

将 Schema URL 与 Tracer 相关联的效果必须是使用 Tracer 发出的遥测数据将与 Schema URL 相关联，前提是发出的数据格式能够表示这种关联。

## Context Interaction

This section defines all operations within the Tracing API that interact with the
[`Context`](../context/context.md).

The API MUST provide the following functionality to interact with a `Context`
instance:

- Extract the `Span` from a `Context` instance
- Insert the `Span` to a `Context` instance

The functionality listed above is necessary because API users SHOULD NOT have
access to the [Context Key](../context/context.md#create-a-key) used by the Tracing API implementation.

上面列出的功能是必要的，因为 API 用户不应该访问Tracing API 实现使用的 上下文key

If the language has support for implicitly propagated `Context` (see
[here](../context/context.md#optional-global-operations)), the API SHOULD also provide
the following functionality:

如果该语言支持隐式传播的上下文（参见此处），则 API 还应提供以下功能：

- Get the currently active span from the implicit context. This is equivalent to getting the implicit context, then extracting the `Span` from the context.
- 从隐式上下文中获取当前active span。这相当于获取隐式上下文，然后从上下文中提取 Span。
- Set the currently active span to the implicit context. This is equivalent to getting the implicit context, then inserting the `Span` to the context.
- 将当前active span设置为隐式上下文。这相当于获取隐式上下文，然后将 Span 插入上下文。

All the above functionalities operate solely on the context API, and they MAY be
exposed as either static methods on the trace module, or as static methods on a class
inside the trace module. This functionality SHOULD be fully implemented in the API when possible.

所有上述功能都仅在上下文 API 上运行，它们可以作为trace模块上的静态方法或作为trace模块内类上的静态方法公开。如果可能，此功能应在 API 中完全实现。

## Tracer

The tracer is responsible for creating `Span`s.     

tracer 负责创建 Span。

Note that `Tracer`s should usually *not* be responsible for configuration.
This should be the responsibility of the `TracerProvider` instead.

请注意，Tracer 通常不应该负责配置。这应该是 TracerProvider 的责任。

### Tracer operations

The `Tracer` MUST provide functions to:

- [Create a new `Span`](#span-creation) (see the section on `Span`)

Tracer 必须提供Create a new Span 方法

## SpanContext

A `SpanContext` represents the portion of a `Span` which must be serialized and
propagated along side of a distributed context. `SpanContext`s are immutable.

SpanContext 表示 Span 的一部分，必须在分布式上下文的一侧进行序列化和传播。 SpanContext 是不可变的。

The OpenTelemetry `SpanContext` representation conforms to the [W3C TraceContext
specification](https://www.w3.org/TR/trace-context/). It contains two
identifiers - a `TraceId` and a `SpanId` - along with a set of common
`TraceFlags` and system-specific `TraceState` values.

OpenTelemetry SpanContext 表示符合 W3C TraceContext 规范。它包含两个标识符 - TraceId 和 SpanId - 以及一组常见的 TraceFlags 和系统特定的 TraceState 值。

`TraceId` A valid trace identifier is a 16-byte array with at least one
non-zero byte.

 一个 16 字节的数组，其中至少有一个非零字节。

`SpanId` A valid span identifier is an 8-byte array with at least one non-zero
byte.

一个 8 字节数组，其中至少有一个非零字节。

`TraceFlags` contain details about the trace. Unlike TraceState values,
TraceFlags are present in all traces. The current version of the specification
only supports a single flag called [sampled](https://www.w3.org/TR/trace-context/#sampled-flag).

traceFlags 包含有关跟踪的详细信息。与 TraceState 值不同，TraceFlags 存在于所有跟踪中。该规范的当前版本仅支持一个名为 sampled 的标志。

`TraceState` carries vendor-specific trace identification data, represented as a list
of key-value pairs. TraceState allows multiple tracing
systems to participate in the same trace. It is fully described in the [W3C Trace Context
specification](https://www.w3.org/TR/trace-context/#tracestate-header). For
specific OTel values in `TraceState`, see the [TraceState Handling](tracestate-handling.md)
document.

TraceState 携带特定于供应商的跟踪标识数据，表示为键值对列表。 TraceState 允许多个跟踪系统参与同一个跟踪。它在 W3C 跟踪上下文规范中有完整的描述。有关 TraceState 中的特定 OTel 值，请参阅 TraceState 处理文档。

The API MUST implement methods to create a `SpanContext`. These methods SHOULD be the only way to
create a `SpanContext`. This functionality MUST be fully implemented in the API, and SHOULD NOT be
overridable.

API 必须实现创建 SpanContext 的方法。这些方法应该是创建 SpanContext 的唯一方法。此功能必须在 API 中完全实现，并且不应被覆盖。

### Retrieving the TraceId and SpanId   获取 TraceId 和 SpanId


The API MUST allow retrieving the `TraceId` and `SpanId` in the following forms:

API 必须允许以下列形式获取 TraceId 和 SpanId：

* Hex - returns the lowercase [hex encoded](https://tools.ietf.org/html/rfc4648#section-8)
`TraceId` (result MUST be a 32-hex-character lowercase string) or `SpanId`
(result MUST be a 16-hex-character lowercase string).
Hex - 返回小写十六进制编码的 TraceId（结果必须是 32 个十六进制字符的小写字符串）或 SpanId（结果必须是一个 16 个十六进制字符的小写字符串）。
* Binary - returns the binary representation of the `TraceId` (result MUST be a
16-byte array) or `SpanId` (result MUST be an 8-byte array).
二进制 - 返回 TraceId（结果必须是 16 字节数组）或 SpanId（结果必须是 8 字节数组）的二进制表示。

The API SHOULD NOT expose details about how they are internally stored.

API 不应公开有关它们如何在内部存储的详细信息。

### IsValid

An API called `IsValid`, that returns a boolean value, which is `true` if the SpanContext has a
non-zero TraceID and a non-zero SpanID, MUST be provided.

必须提供一个名为 IsValid 的 API，它返回一个布尔值，如果 SpanContext 具有非零 TraceID 和非零 SpanID，则该值为真。

### IsRemote

An API called `IsRemote`, that returns a boolean value, which is `true` if the SpanContext was
propagated from a remote parent, MUST be provided.
When extracting a `SpanContext` through the [Propagators API](../context/api-propagators.md#propagators-api),
`IsRemote` MUST return true, whereas for the SpanContext of any child spans it MUST return false.

必须提供一个名为 IsRemote 的 API，它返回一个布尔值，如果 SpanContext 是从远程父级传播的，则该值为 true。当通过 Propagators API 提取 SpanContext 时，IsRemote 必须返回 true，而对于任何子 span 的 SpanContext，它必须返回 false。

### TraceState

`TraceState` is a part of [`SpanContext`](./api.md#spancontext), represented by an immutable list of string key/value pairs and
formally defined by the [W3C Trace Context specification](https://www.w3.org/TR/trace-context/#tracestate-header).
Tracing API MUST provide at least the following operations on `TraceState`:

TraceState 是 SpanContext 的一部分，由字符串键/值对的不可变列表表示，并由 W3C Trace Context 规范正式定义。 Tracing API 必须至少提供以下对 TraceState 的操作：

* Get value for a given key
* Add a new key/value pair
* Update an existing value for a given key
* Delete a key/value pair

These operations MUST follow the rules described in the [W3C Trace Context specification](https://www.w3.org/TR/trace-context/#mutating-the-tracestate-field).
All mutating operations MUST return a new `TraceState` with the modifications applied.
`TraceState` MUST at all times be valid according to rules specified in [W3C Trace Context specification](https://www.w3.org/TR/trace-context/#tracestate-header-field-values).
Every mutating operations MUST validate input parameters.
If invalid value is passed the operation MUST NOT return `TraceState` containing invalid data
and MUST follow the [general error handling guidelines](../error-handling.md).

这些操作必须遵循 W3C 跟踪上下文规范中描述的规则。所有变异操作都必须返回一个新的 TraceState 并应用了修改。根据 W3C 跟踪上下文规范中指定的规则，TraceState 必须始终有效。每个变异操作都必须验证输入参数。如果传递了无效值，则操作不得返回包含无效数据的 TraceState，并且必须遵循一般错误处理指南。

Please note, since `SpanContext` is immutable, it is not possible to update `SpanContext` with a new `TraceState`.
Such changes then make sense only right before
[`SpanContext` propagation](../context/api-propagators.md)
or [telemetry data exporting](sdk.md#span-exporter).
In both cases, `Propagator`s and `SpanExporter`s may create a modified `TraceState` copy before serializing it to the wire.

请注意，由于 SpanContext 是不可变的，因此无法使用新的 TraceState 更新 SpanContext。此类更改仅在 SpanContext 传播或遥测数据导出之前才有意义。在这两种情况下，Propagators 和 SpanExporters 可能会在将其序列化到线路之前创建修改后的 TraceState 副本。

## Span

A `Span` represents a single operation within a trace. Spans can be nested to
form a trace tree. Each trace contains a root span, which typically describes
the entire operation and, optionally, one or more sub-spans for its sub-operations.

`Span`表示跟踪中的单个操作。`Span`可以嵌套以形成跟踪树。每个trace包含一个根Span，它通常描述整个操作，并且可选地描述其子操作的一个或多个子Span。

<a name="span-data-members"></a>
`Span`s encapsulate:  封装

- The span name
- An immutable [`SpanContext`](#spancontext) that uniquely identifies the
  `Span`
- A parent span in the form of a [`Span`](#span), [`SpanContext`](#spancontext),
  or null
- A [`SpanKind`](#spankind)
- A start timestamp
- An end timestamp
- [`Attributes`](../common/common.md#attributes)
- A list of [`Link`s](#specifying-links) to other `Span`s
- A list of timestamped [`Event`s](#add-events)
- A [`Status`](#set-status).

The _span name_ concisely identifies the work represented by the Span,
for example, an RPC method name, a function name,
or the name of a subtask or stage within a larger computation.
The span name SHOULD be the most general string that identifies a
(statistically) interesting _class of Spans_,
rather than individual Span instances while still being human-readable.
That is, "get_user" is a reasonable name, while "get_user/314159",
where "314159" is a user ID, is not a good name due to its high cardinality.
Generality SHOULD be prioritized over human-readability.

Span 名称简洁地标识了 Span 所代表的工作，例如，RPC 方法名称、函数名称或较大计算中的子任务或阶段的名称。span名称应该是最通用的字符串，用于标识（统计上） 有趣的？ Span类，而不是单个 Span实例，同时仍然是人类可读的。也就是说，“get_user”是一个合理的名字，而“get_user/314159”，其中“314159”是一个用户ID，由于基数高？，不是一个好名字。普遍性应该优先于人类可读性。

For example, here are potential span names for an endpoint that gets a
hypothetical account information:

| Span Name         | Guidance     |
| ----------------- | ------------ |
| `get`             | Too general  |
| `get_account/42`  | Too specific |
| `get_account`     | Good, and account_id=42 would make a nice Span attribute |
| `get_account/{accountId}` | Also good (using the "HTTP route") |

The `Span`'s start and end timestamps reflect the elapsed real time of the
operation.

Span 的开始和结束时间戳反映了操作经过的实时时间。

For example, if a span represents a request-response cycle (e.g. HTTP or an RPC),
the span should have a start time that corresponds to the start time of the
first sub-operation, and an end time of when the final sub-operation is complete.
This includes:

例如，如果一个span代表一个请求-响应周期（例如HTTP或RPC），那么span应该有一个开始时间对应于第一个子操作的开始时间，以及一个结束时间为最后一个子操作的结束时间。操作完成。这包括：

- receiving the data from the request
- parsing of the data (e.g. from a binary or json format)
- any middleware or additional processing logic
- business logic
- construction of the response
- sending of the response

Child spans (or in some cases events) may be created to represent
sub-operations which require more detailed observability. Child spans should
measure the timing of the respective sub-operation, and may add additional
attributes.

可以创建子spans（或在某些情况下事件）来表示需要更详细可观察性的子操作。子spans应该测量相应子操作的时间，并且可以添加附加属性。

A `Span`'s start time SHOULD be set to the current time on [span
creation](#span-creation). After the `Span` is created, it SHOULD be possible to
change its name, set its `Attribute`s, add `Event`s, and set the `Status`. These
MUST NOT be changed after the `Span`'s end time has been set.

Span的开始时间应该设置为Span创建的当前时间。创建 Span 后，应该可以更改其名称、设置其属性、添加事件和设置状态。在设置 Span 的结束时间后，不得更改这些内容。

`Span`s are not meant to be used to propagate information within a process. To
prevent misuse, implementations SHOULD NOT provide access to a `Span`'s
attributes besides its `SpanContext`.

`Span`并不意味着用于在流程中传播信息。为防止误用，实现不应提供对除 SpanContext 之外的 Span 属性的访问。

Vendors may implement the `Span` interface to effect vendor-specific logic.
However, alternative implementations MUST NOT allow callers to create `Span`s
directly. All `Span`s MUST be created via a `Tracer`.

供应商可以实现 Span 接口来影响供应商特定的逻辑。但是，替代实现不得允许调用者直接创建 Span。所有 Span 都必须通过 Tracer 创建。

### Span Creation

There MUST NOT be any API for creating a `Span` other than with a [`Tracer`](#tracer).

除了 Tracer 之外，不得有任何用于创建 Span 的 API。

In languages with implicit `Context` propagation, `Span` creation MUST NOT
set the newly created `Span` as the active `Span` in the
[current `Context`](#context-interaction) by default, but this functionality
MAY be offered additionally as a separate operation.

在具有隐式上下文传播的语言中，默认情况下，Span创建不得将新创建的跨度设置为当前上下文中的活动跨度，但此功能可以作为单独的操作另外提供。

The API MUST accept the following parameters:

- The span name. This is a required parameter.
- The parent `Context` or an indication that the new `Span` should be a root `Span`.
  The API MAY also have an option for implicitly using
  the current Context as parent as a default behavior.
  This API MUST NOT accept a `Span` or `SpanContext` as parent, only a full `Context`.

  The semantic parent of the Span MUST be determined according to the rules
  described in [Determining the Parent Span from a Context](#determining-the-parent-span-from-a-context).
  
  Span 的语义父级必须根据从上下文确定父级 Span 中描述的规则来确定。
  
- [`SpanKind`](#spankind), default to `SpanKind.Internal` if not specified.
- [`Attributes`](../common/common.md#attributes). Additionally,
  these attributes may be used to make a sampling decision as noted in [sampling
  description](sdk.md#sampling). An empty collection will be assumed if
  not specified.

  Whenever possible, users SHOULD set any already known attributes at span creation
  instead of calling `SetAttribute` later.
  
  只要有可能，用户应该在创建跨度时设置任何已知的属性，而不是稍后调用 SetAttribute。

- `Link`s - an ordered sequence of Links, see API definition [here](#specifying-links).
- `Start timestamp`, default to current time. This argument SHOULD only be set
  when span creation time has already passed. If API is called at a moment of
  a Span logical start, API user MUST NOT explicitly set this argument.

Each span has zero or one parent span and zero or more child spans, which
represent causally related operations. A tree of related spans comprises a
trace. A span is said to be a _root span_ if it does not have a parent. Each
trace includes a single root span, which is the shared ancestor of all other
spans in the trace. Implementations MUST provide an option to create a `Span` as
a root span, and MUST generate a new `TraceId` for each root span created.
For a Span with a parent, the `TraceId` MUST be the same as the parent.
Also, the child span MUST inherit all `TraceState` values of its parent by default.

每个span都有零个或一个父span以及零个或多个子span，它们表示因果相关的操作。相关span的树包括跟踪。如果span没有父span，则称其为根span。每个跟踪包含一个根span，它是跟踪中所有其他span的共享祖先。实现必须提供一个选项来创建一个 Span 作为根span，并且必须为每个创建的根span生成一个新的 TraceId。对于具有父级的 Span，TraceId 必须与父级相同。此外，默认情况下，子span必须继承其父span的所有 TraceState 值。

A `Span` is said to have a _remote parent_ if it is the child of a `Span`
created in another process. Each propagators' deserialization must set
`IsRemote` to true on a parent `SpanContext` so `Span` creation knows if the
parent is remote.

如果 Span 是在另一个进程中创建的 Span 的子项，则称该 Span 具有远程父项。每个传播器的反序列化必须在父 SpanContext 上将 IsRemote 设置为 true，以便 Span 创建知道父对象是否是远程的。

Any span that is created MUST also be ended.
This is the responsibility of the user.
API implementations MAY leak memory or other resources
(including, for example, CPU time for periodic work that iterates all spans)
if the user forgot to end the span.

创建的任何span也必须结束。这是用户的责任。如果用户忘记结束跨度，API 实现可能会泄漏内存或其他资源（包括，例如，迭代所有跨度的周期性工作的 CPU 时间）。

#### Determining the Parent Span from a Context

When a new `Span` is created from a `Context`, the `Context` may contain a `Span`
representing the currently active instance, and will be used as parent.
If there is no `Span` in the `Context`, the newly created `Span` will be a root span.

当从上下文创建新的 Span 时，上下文可能包含一个表示当前活动实例的 Span，并将用作父级。如果上下文中没有 Span，则新创建的 Span 将是根 Span。

A `SpanContext` cannot be set as active in a `Context` directly, but by
[wrapping it into a Span](#wrapping-a-spancontext-in-a-span).
For example, a `Propagator` performing context extraction may need this.

SpanContext 不能直接在 Context 中设置为 active，而是通过将其包装到 Span 中。例如，执行上下文提取的传播器可能需要这个。

#### Specifying links

During the `Span` creation user MUST have the ability to record links to other
`Span`s. Linked `Span`s can be from the same or a different trace. See [Links
description](../overview.md#links-between-spans). `Link`s cannot be added after
Span creation.

在创建 Span 期间，用户必须能够记录到其他 Span 的链接。链接的 Span 可以来自相同或不同的跟踪。请参阅链接说明。创建 Span 后无法添加链接。

A `Link` is structurally defined by the following properties: Link 在结构上由以下属性定义：

- `SpanContext` of the `Span` to link to.
- Zero or more [`Attributes`](../common/common.md#attributes) further describing
  the link.

The Span creation API MUST provide:  Span 创建 API 必须提供：

- An API to record a single `Link` where the `Link` properties are passed as
  arguments. This MAY be called `AddLink`. This API takes the `SpanContext` of
  the `Span` to link to and optional `Attributes`, either as individual
  parameters or as an immutable object encapsulating them, whichever is most
  appropriate for the language. Implementations MAY ignore links with an
  [invalid](#isvalid) `SpanContext`.
  
  用于记录单个链接的 API，其中链接属性作为参数传递。这可以称为 AddLink。此 API 将 Span 的 SpanContext 链接到可选属性，作为单独的参数或作为封装它们的不可变对象，以最适合该语言的方式为准。实现可以忽略具有无效 SpanContext 的链接。

Links SHOULD preserve the order in which they're set.  链接应该保留它们设置的顺序。

### Span operations

With the exception of the function to retrieve the `Span`'s `SpanContext` and
recording status, none of the below may be called after the `Span` is finished.

除了检索 Span 的 SpanContext 和录制状态的函数外，在 Span 完成后不能调用以下任何一个。

#### Get Context

The Span interface MUST provide:

- An API that returns the `SpanContext` for the given `Span`. The returned value
  may be used even after the `Span` is finished. The returned value MUST be the
  same for the entire Span lifetime. This MAY be called `GetContext`.
  
  返回给定 Span 的 SpanContext 的 API。即使在 Span 完成后，也可以使用返回的值。整个 Span 生命周期的返回值必须相同。这可以称为 GetContext。

#### IsRecording

Returns true if this `Span` is recording information like events with the
`AddEvent` operation, attributes using `SetAttributes`, status with `SetStatus`,
etc.

如果此 Span 正在使用 AddEvent 操作记录事件、使用 SetAttributes 的属性、使用 SetStatus 的状态等信息，则返回 true。

After a `Span` is ended, it usually becomes non-recording and thus
`IsRecording` SHOULD consequently return false for ended Spans.
Note: Streaming implementations, where it is not known if a span is ended,
are one expected case where `IsRecording` cannot change after ending a Span.

在 Span 结束后，它通常会变为非记录状态，因此 IsRecording 应该因此为结束的 Span 返回 false。注意：流实现，其中不知道跨度是否结束，是一种预期情况，即在结束跨度后 IsRecording 无法更改。

`IsRecording` SHOULD NOT take any parameters.

This flag SHOULD be used to avoid expensive computations of a Span attributes or
events in case when a Span is definitely not recorded. Note that any child
span's recording is determined independently from the value of this flag
(typically based on the `sampled` flag of a `TraceFlags` on
[SpanContext](#spancontext)).

这个标志应该用于在一个 Span 绝对没有被记录的情况下避免对 Span 属性或事件进行昂贵的计算。请注意，任何子跨度的记录都是独立于该标志的值确定的（通常基于 SpanContext 上 TraceFlags 的采样标志）。

This flag may be `true` despite the entire trace being sampled out. This
allows to record and process information about the individual Span without
sending it to the backend. An example of this scenario may be recording and
processing of all incoming requests for the processing and building of
SLA/SLO latency charts while sending only a subset - sampled spans - to the
backend. See also the [sampling section of SDK design](sdk.md#sampling).

尽管采样了整个跟踪，但该标志可能为真。这允许记录和处理有关单个 Span 的信息，而无需将其发送到后端。此场景的一个示例可能是记录和处理所有传入请求以处理和构建 SLA/SLO 延迟图表，同时仅向后端发送一个子集（采样跨度）。另请参阅 SDK 设计的采样部分。

Users of the API should only access the `IsRecording` property when
instrumenting code and never access `SampledFlag` unless used in context
propagators.

API 的用户应该只在检测代码时访问 IsRecording 属性，除非在上下文传播器中使用，否则永远不要访问 SampledFlag。

#### Set Attributes

A `Span` MUST have the ability to set [`Attributes`](../common/common.md#attributes) associated with it.

The Span interface MUST provide:

- An API to set a single `Attribute` where the attribute properties are passed
  as arguments. This MAY be called `SetAttribute`. To avoid extra allocations some
  implementations may offer a separate API for each of the possible value types.

The Span interface MAY provide:

- An API to set multiple `Attributes` at once, where the `Attributes` are passed in a
  single method call.

Setting an attribute with the same key as an existing attribute SHOULD overwrite
the existing attribute's value.

请注意，OpenTelemetry 项目记录了某些具有规定语义含义的“标准属性”

Note that the OpenTelemetry project documents certain ["standard
attributes"](semantic_conventions/README.md) that have prescribed semantic meanings.

Note that [Samplers](sdk.md#sampler) can only consider information already
present during span creation. Any changes done later, including new or changed
attributes, cannot change their decisions.

请注意，采样器只能考虑跨度创建期间已经存在的信息。以后所做的任何更改，包括新的或更改的属性，都不能改变他们的决定。

#### Add Events

A `Span` MUST have the ability to add events. Events have a time associated
with the moment when they are added to the `Span`.

Span 必须能够添加事件。事件有一个与它们被添加到 Span 的时刻相关联的时间。

An `Event` is structurally defined by the following properties:

- Name of the event.
- A timestamp for the event. Either the time at which the event was
  added or a custom timestamp provided by the user.
- Zero or more [`Attributes`](../common/common.md#attributes) further describing
  the event.

The Span interface MUST provide:

- An API to record a single `Event` where the `Event` properties are passed as
  arguments. This MAY be called `AddEvent`.
  This API takes the name of the event, optional `Attributes` and an optional
  `Timestamp` which can be used to specify the time at which the event occurred,
  either as individual parameters or as an immutable object encapsulating them,
  whichever is most appropriate for the language. If no custom timestamp is
  provided by the user, the implementation automatically sets the time at which
  this API is called on the event.

Events SHOULD preserve the order in which they are recorded.
This will typically match the ordering of the events' timestamps,
but events may be recorded out-of-order using custom timestamps.

事件应该保持它们被记录的顺序。这通常与事件时间戳的顺序相匹配，但可以使用自定义时间戳无序记录事件。


Consumers should be aware that an event's timestamp might be before the start or
after the end of the span if custom timestamps were provided by the user for the
event or when starting or ending the span.
The specification does not require any normalization if provided timestamps are
out of range.

Note that the OpenTelemetry project documents certain ["standard event names and
keys"](semantic_conventions/README.md) which have prescribed semantic meanings.

Note that [`RecordException`](#record-exception) is a specialized variant of
`AddEvent` for recording exception events.

#### Set Status

Sets the `Status` of the `Span`. If used, this will override the default `Span`
status, which is `Unset`.

`Status` is structurally defined by the following properties:

- `StatusCode`, one of the values listed below.
- Optional `Description` that provides a descriptive message of the `Status`.
  `Description` MUST only be used with the `Error` `StatusCode` value.
  An empty `Description` is equivalent with a not present one.

`StatusCode` is one of the following values:

- `Unset`
  - The default status.
- `Ok`
  - The operation has been validated by an Application developer or Operator to
    have completed successfully.
- `Error`
  - The operation contains an error.

These values form a total order: `Ok > Error > Unset`.
This means that setting `Status` with `StatusCode=Ok` will override any prior or future attempts to set
span `Status` with `StatusCode=Error` or `StatusCode=Unset`. See below for more specific rules.

The Span interface MUST provide:

- An API to set the `Status`. This SHOULD be called `SetStatus`. This API takes
  the `StatusCode`, and an optional `Description`, either as individual
  parameters or as an immutable object encapsulating them, whichever is most
  appropriate for the language. `Description` MUST be IGNORED for `StatusCode`
  `Ok` & `Unset` values.

The status code SHOULD remain unset, except for the following circumstances:

An attempt to set value `Unset` SHOULD be ignored.

When the status is set to `Error` by Instrumentation Libraries, the status codes
SHOULD be documented and predictable. The status code should only be set to `Error`
according to the rules defined within the semantic conventions. For operations
not covered by the semantic conventions, Instrumentation Libraries SHOULD
publish their own conventions, including status codes.

Generally, Instrumentation Libraries SHOULD NOT set the status code to `Ok`,
unless explicitly configured to do so. Instrumention libraries SHOULD leave the
status code as `Unset` unless there is an error, as described above.

Application developers and Operators may set the status code to `Ok`.

When span status is set to `Ok` it SHOULD be considered final and any further
attempts to change it SHOULD be ignored.

Analysis tools SHOULD respond to an `Ok` status by suppressing any errors they
would otherwise generate. For example, to suppress noisy errors such as 404s.

Only the value of the last call will be recorded, and implementations are free
to ignore previous calls.

#### UpdateName

Updates the `Span` name. Upon this update, any sampling behavior based on `Span`
name will depend on the implementation.

Note that [Samplers](sdk.md#sampler) can only consider information already
present during span creation. Any changes done later, including updated span
name, cannot change their decisions.

Alternatives for the name update may be late `Span` creation, when Span is
started with the explicit timestamp from the past at the moment where the final
`Span` name is known, or reporting a `Span` with the desired name as a child
`Span`.

Required parameters:

- The new **span name**, which supersedes whatever was passed in when the
  `Span` was started

#### End

Signals that the operation described by this span has
now (or at the time optionally specified) ended.

Implementations SHOULD ignore all subsequent calls to `End` and any other Span methods,
i.e. the Span becomes non-recording by being ended
(there might be exceptions when Tracer is streaming events
and has no mutable state associated with the `Span`).

Language SIGs MAY provide methods other than `End` in the API that also end the
span to support language-specific features like `with` statements in Python.
However, all API implementations of such methods MUST internally call the `End`
method and be documented to do so.

`End` MUST NOT have any effects on child spans.
Those may still be running and can be ended later.

`End` MUST NOT inactivate the `Span` in any `Context` it is active in.
It MUST still be possible to use an ended span as parent via a Context it is
contained in. Also, any mechanisms for putting the Span into a Context MUST
still work after the Span was ended.

Parameters:

- (Optional) Timestamp to explicitly set the end timestamp.
  If omitted, this MUST be treated equivalent to passing the current time.

Expect this operation to be called in the "hot path" of production
applications. It needs to be designed to complete fast, if not immediately.
This operation itself MUST NOT perform blocking I/O on the calling thread.
Any locking used needs be minimized and SHOULD be removed entirely if
possible. Some downstream SpanProcessors and subsequent SpanExporters called
from this operation may be used for testing, proof-of-concept ideas, or
debugging and may not be designed for production use themselves. They are not
in the scope of this requirement and recommendation.

#### Record Exception

To facilitate recording an exception languages SHOULD provide a
`RecordException` method if the language uses exceptions.
This is a specialized variant of [`AddEvent`](#add-events),
so for anything not specified here, the same requirements as for `AddEvent` apply.

The signature of the method is to be determined by each language
and can be overloaded as appropriate.
The method MUST record an exception as an `Event` with the conventions outlined in
the [exception semantic conventions](semantic_conventions/exceptions.md) document.
The minimum required argument SHOULD be no more than only an exception object.

If `RecordException` is provided, the method MUST accept an optional parameter
to provide any additional event attributes
(this SHOULD be done in the same way as for the `AddEvent` method).
If attributes with the same name would be generated by the method already,
the additional attributes take precedence.

Note: `RecordException` may be seen as a variant of `AddEvent` with
additional exception-specific parameters and all other parameters being optional
(because they have defaults from the exception semantic convention).

### Span lifetime

Span lifetime represents the process of recording the start and the end
timestamps to the Span object:

- The start time is recorded when the Span is created.
- The end time needs to be recorded when the operation is ended.

Start and end time as well as Event's timestamps MUST be recorded at a time of a
calling of corresponding API.

### Wrapping a SpanContext in a Span

The API MUST provide an operation for wrapping a `SpanContext` with an object
implementing the `Span` interface. This is done in order to expose a `SpanContext`
as a `Span` in operations such as in-process `Span` propagation.

API 必须提供一个操作，用于用实现 Span 接口的对象包装 SpanContext。这样做是为了在诸如进程内 Span 传播之类的操作中将 SpanContext 公开为 Span。

If a new type is required for supporting this operation, it SHOULD NOT be exposed
publicly if possible (e.g. by only exposing a function that returns something
with the Span interface type). If a new type is required to be publicly exposed,
it SHOULD be named `NonRecordingSpan`.

The behavior is defined as follows:

- `GetContext` MUST return the wrapped `SpanContext`.
- `IsRecording` MUST return `false` to signal that events, attributes and other elements
  are not being recorded, i.e. they are being dropped.

The remaining functionality of `Span` MUST be defined as no-op operations.
Note: This includes `End`, so as an exception from the general rule,
it is not required (or even helpful) to end such a Span.

This functionality MUST be fully implemented in the API, and SHOULD NOT be overridable.

## SpanKind

`SpanKind` describes the relationship between the Span, its parents,
and its children in a Trace.  `SpanKind` describes two independent
properties that benefit tracing systems during analysis.

SpanKind 在 Trace 中描述了 Span、它的父母和它的孩子之间的关系。 SpanKind 描述了在分析过程中有益于跟踪系统的两个独立属性。

The first property described by `SpanKind` reflects whether the Span
is a "logical" remote child or parent. By "logical", we mean that
the span is logically a remote child or parent, from the point of view
of the library that is being instrumented. Spans with a remote parent are
interesting because they are sources of external load.  Spans with a
remote child are interesting because they reflect a non-local system
dependency.

SpanKind 描述的第一个属性反映了 Span 是“逻辑”远程子项还是父项。 “逻辑”是指从被检测的库的角度来看，跨度在逻辑上是远程子项或父项。具有远程父级的 Span 很有趣，因为它们是外部负载的来源。带有远程子节点的 Span 很有趣，因为它们反映了非本地系统依赖性。

The second property described by `SpanKind` reflects whether a child
Span represents a synchronous call.  When a child span is synchronous,
the parent is expected to wait for it to complete under ordinary
circumstances.  It can be useful for tracing systems to know this
property, since synchronous Spans may contribute to the overall trace
latency. Asynchronous scenarios can be remote or local.

SpanKind 描述的第二个属性反映了子 Span 是否表示同步调用。当子跨度同步时，父跨在正常情况下应该等待它完成。了解此属性对跟踪系统很有用，因为同步 Span 可能会导致整体跟踪延迟。异步场景可以是远程的，也可以是本地的。

In order for `SpanKind` to be meaningful, callers SHOULD arrange that
a single Span does not serve more than one purpose.  For example, a
server-side span SHOULD NOT be used directly as the parent of another
remote span.  As a simple guideline, instrumentation should create a
new Span prior to extracting and serializing the SpanContext for a
remote call.

Note: there are complex scenarios where a CLIENT span may have a child
that is also logically a CLIENT span, or a PRODUCER span might have a local child
that is a CLIENT span, depending on how the various libraries that are providing
the functionality are built and instrumented. These scenarios, when they occur,
should be detailed in the semantic conventions appropriate to the relevant
libraries.

These are the possible SpanKinds:

* `SERVER` Indicates that the span covers server-side handling of a
  synchronous RPC or other remote request.  This span is often the child
  of a remote `CLIENT` span that was expected to wait for a response.
* `CLIENT` Indicates that the span describes a request to
  some remote service.  This span is usually the parent of a remote `SERVER`
  span and does not end until the response is received.
* `PRODUCER` Indicates that the span describes the initiators of an
  asynchronous request.  This parent span will often end before
  the corresponding child `CONSUMER` span, possibly even before the
  child span starts. In messaging scenarios with batching, tracing
  individual messages requires a new `PRODUCER` span per message to
  be created.
* `CONSUMER` Indicates that the span describes a child of an
  asynchronous `PRODUCER` request.
* `INTERNAL` Default value. Indicates that the span represents an
  internal operation within an application, as opposed to an
  operations with remote parents or children.

To summarize the interpretation of these kinds:

| `SpanKind` | Synchronous | Asynchronous | Remote Incoming | Remote Outgoing |
|---|---|---|---|---|
| `CLIENT` | yes | | | yes |
| `SERVER` | yes | | yes | |
| `PRODUCER` | | yes | | maybe |
| `CONSUMER` | | yes | maybe | |
| `INTERNAL` | | | | |

## Concurrency

For languages which support concurrent execution the Tracing APIs provide
specific guarantees and safeties. Not all of API functions are safe to
be called concurrently.

**TracerProvider** - all methods are safe to be called concurrently.

**Tracer** - all methods are safe to be called concurrently.

**Span** - All methods of Span are safe to be called concurrently.

**Event** - Events are immutable and safe to be used concurrently.

**Link** - Links are immutable and safe to be used concurrently.

## Included Propagators

See [Propagators Distribution](../context/api-propagators.md#propagators-distribution)
for how propagators are to be distributed.

## Behavior of the API in the absence of an installed SDK

In general, in the absence of an installed SDK, the Trace API is a "no-op" API.
This means that operations on a Tracer, or on Spans, should have no side effects and do nothing. However, there
is one important exception to this general rule, and that is related to propagation of a `SpanContext`:
The API MUST create a [non-recording Span](#wrapping-a-spancontext-in-a-span) with the `SpanContext`
that is in the `Span` in the parent `Context` (whether explicitly given or implicit current) or,
if the parent is a non-recording Span (which it usually always is if no SDK is present),
it MAY return the parent Span back from the creation method.
If the parent `Context` contains no `Span`, an empty non-recording Span MUST be returned instead
(i.e., having a `SpanContext` with all-zero Span and Trace IDs, empty Tracestate, and unsampled TraceFlags).
This means that a `SpanContext` that has been provided by a configured `Propagator`
will be propagated through to any child span and ultimately also `Inject`,
but that no new `SpanContext`s will be created.
