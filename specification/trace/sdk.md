# Tracing SDK

**Status**: [Stable](../document-status.md)

<details>

<summary>Table of Contents</summary>

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->
<!-- https://github.com/jonschlinkert/markdown-toc -->

<!-- toc -->

- [Tracer Provider](#tracer-provider)
  * [Tracer Creation](#tracer-creation)
  * [Shutdown](#shutdown)
  * [ForceFlush](#forceflush)
- [Additional Span Interfaces](#additional-span-interfaces)
- [Sampling](#sampling)
  * [SDK Span creation](#sdk-span-creation)
  * [Sampler](#sampler)
    + [ShouldSample](#shouldsample)
    + [GetDescription](#getdescription)
  * [Built-in samplers](#built-in-samplers)
    + [AlwaysOn](#alwayson)
    + [AlwaysOff](#alwaysoff)
    + [TraceIdRatioBased](#traceidratiobased)
      - [Requirements for `TraceIdRatioBased` sampler algorithm](#requirements-for-traceidratiobased-sampler-algorithm)
    + [ParentBased](#parentbased)
- [Span Limits](#span-limits)
- [Id Generators](#id-generators)
- [Span processor](#span-processor)
  * [Interface definition](#interface-definition)
    + [OnStart](#onstart)
    + [OnEnd(Span)](#onendspan)
    + [Shutdown()](#shutdown)
    + [ForceFlush()](#forceflush)
  * [Built-in span processors](#built-in-span-processors)
    + [Simple processor](#simple-processor)
    + [Batching processor](#batching-processor)
- [Span Exporter](#span-exporter)
  * [Interface Definition](#interface-definition)
    + [`Export(batch)`](#exportbatch)
    + [`Shutdown()`](#shutdown)
    + [`ForceFlush()`](#forceflush)
  * [Further Language Specialization](#further-language-specialization)
    + [Examples](#examples)
      - [Go SpanExporter Interface](#go-spanexporter-interface)
      - [Java SpanExporter Interface](#java-spanexporter-interface)

<!-- tocstop -->

</details>

## Tracer Provider

### Tracer Creation

New `Tracer` instances are always created through a `TracerProvider` (see
[API](api.md#tracerprovider)). The `name` and `version` arguments
supplied to the `TracerProvider` must be used to create an
[`InstrumentationLibrary`][otep-83] instance which is stored on the created
`Tracer`.

新的 Tracer 实例总是通过 TracerProvider 创建（见 API）。必须使用提供给 TracerProvider 的名称和版本参数来创建存储在创建的 Tracer 上的 InstrumentationLibrary 实例。

Configuration (i.e., [SpanProcessors](#span-processor), [IdGenerator](#id-generators),
[SpanLimits](#span-limits) and [`Sampler`](#sampling)) MUST be managed solely by
the `TracerProvider` and it MUST provide some way to configure all of them that
are implemented in the SDK, at least when creating or initializing it.

配置（即 SpanProcessors、IdGenerator、SpanLimits 和 Sampler）必须由 TracerProvider 单独管理，并且它必须提供某种方式来配置在 SDK 中实现的所有这些，至少在创建或初始化它时。

The TracerProvider MAY provide methods to update the configuration. If
configuration is updated (e.g., adding a `SpanProcessor`),
the updated configuration MUST also apply to all already returned `Tracers`
(i.e. it MUST NOT matter whether a `Tracer` was obtained from the
`TracerProvider` before or after the configuration change).
Note: Implementation-wise, this could mean that `Tracer` instances have a
reference to their `TracerProvider` and access configuration only via this
reference.

TracerProvider 可以提供更新配置的方法。如果配置被更新（例如，添加一个 SpanProcessor），更新的配置也必须适用于所有已经返回的 Tracer（即 Tracer 是在配置更改之前还是之后从 TracerProvider 获得的都无关紧要）。注意：在实现方面，这可能意味着 Tracer 实例具有对其 TracerProvider 的引用，并且只能通过此引用访问配置。

### Shutdown

This method provides a way for provider to do any cleanup required.

此方法为提供程序提供了一种进行所需清理的方法。

`Shutdown` MUST be called only once for each `TracerProvider` instance. After
the call to `Shutdown`, subsequent attempts to get a `Tracer` are not allowed. SDKs
SHOULD return a valid no-op Tracer for these calls, if possible.

Shutdown 必须为每个 TracerProvider 实例调用一次。在调用 Shutdown 之后，不允许后续尝试获取 Tracer。如果可能，SDK 应该为这些调用返回一个有效的 no-op Tracer。

`Shutdown` SHOULD provide a way to let the caller know whether it succeeded,
failed or timed out.

Shutdown应该提供一种方法让调用者知道它是成功、失败还是超时。

`Shutdown` SHOULD complete or abort within some timeout. `Shutdown` can be
implemented as a blocking API or an asynchronous API which notifies the caller
via a callback or an event. OpenTelemetry client authors can decide if they want to
make the shutdown timeout configurable.

Shutdown应该在某个超时时间内完成或中止。Shutdown可以实现为阻塞 API 或异步 API，它通过回调或事件通知调用者。 OpenTelemetry 客户端作者可以决定他们是否要使Shutdown超时可配置。

`Shutdown` MUST be implemented at least by invoking `Shutdown` within all internal processors.

必须至少通过在所有内部处理器中调用 Shutdown 来实现 Shutdown。

### ForceFlush

This method provides a way for provider to immediately export all spans that have not yet been exported for all the internal processors.

此方法为提供程序提供了一种方法，可以立即导出所有内部处理器尚未导出的所有span。


`ForceFlush` SHOULD provide a way to let the caller know whether it succeeded,
failed or timed out.ForceFlush 应该提供一种方法让调用者知道它是否成功、失败或超时。


`ForceFlush` SHOULD complete or abort within some timeout. `ForceFlush` can be
implemented as a blocking API or an asynchronous API which notifies the caller
via a callback or an event. OpenTelemetry client authors can decide if they want to
make the flush timeout configurable.

`ForceFlush` MUST invoke `ForceFlush` on all registered `SpanProcessors`.ForceFlush 必须在所有注册的 SpanProcessor 上调用 ForceFlush。

## Additional Span Interfaces

The [API-level definition for Span's interface](api.md#span-operations)
only defines write-only access to the span.
This is good because instrumentations and applications are not meant to use the data
stored in a span for application logic.
However, the SDK needs to eventually read back the data in some locations.
Thus, the SDK specification defines sets of possible requirements for
`Span`-like parameters:

Span 接口的 API 级别定义仅定义了对 Span 的只写访问。这很好，因为instrumentations和应用程序并不打算将存储在跨度中的span用于应用程序逻辑。但是，SDK 最终需要回读某些位置的数据。因此，SDK 规范为类似 Span 的参数定义了一组可能的要求：

* **Readable span**: A function receiving this as argument MUST be able to
  access all information that was added to the span,
  as listed [in the API spec](api.md#span-data-members).
  In particular, it MUST also be able to access
  the `InstrumentationLibrary` and `Resource` information (implicitly)
  associated with the span.
  It must also be able to reliably determine whether the Span has ended
  (some languages might implement this by having an end timestamp of `null`,
  others might have an explicit `hasEnded` boolean).
  
  可读的Span:接收此作为参数的函数必须能够访问添加到范围的所有信息，如 API 规范中所列。特别是，它还必须能够访问与跨度关联的 InstrumentationLibrary 和资源信息（隐式）。它还必须能够可靠地确定 Span 是否已经结束（某些语言可能通过将结束时间戳设为 null 来实现这一点，而其他语言可能具有明确的 hasEnded 布尔值）。

  Counts for attributes, events and links dropped due to collection limits MUST be
  available for exporters to report as described in the [exporters](./sdk_exporters/non-otlp.md#dropped-attributes-count)
  specification.
  
  由于收集限制而丢弃的属性、事件和链接的计数必须可供exporters按照exporter规范中的描述进行报告。

  A function receiving this as argument might not be able to modify the Span.

   接收 this 作为参数的函数可能无法修改 Span。

  Note: Typically this will be implemented with a new interface or
  (immutable) value type.
  In some languages SpanProcessors may have a different readable span type
  than exporters (e.g. a `SpanData` type might contain an immutable snapshot and
  a `ReadableSpan` interface might read information directly from the same
  underlying data structure that the `Span` interface manipulates).
  
  注意：通常这将使用新的接口或（不可变的）值类型来实现。在某些语言中，SpanProcessor 可能具有与导出器不同的可读跨度类型（例如，SpanData 类型可能包含不可变快照，而 ReadableSpan 接口可能直接从 Span 接口操作的相同底层数据结构中读取信息）。

* **Read/write span**: A function receiving this as argument must have access to
  both the full span API as defined in the
  [API-level definition for span's interface](api.md#span-operations) and
  additionally must be able to retrieve all information that was added to the span
  (as with *readable span*).
  
  读/写跨度：接收此参数的函数必须能够访问跨度接口的 API 级别定义中定义的完整跨度 API，此外还必须能够检索添加到跨度中的所有信息（如可读的跨度）。

  It MUST be possible for functions being called with this
  to somehow obtain the same `Span` instance and type
  that the [span creation API](api.md#span-creation) returned (or will return) to the user
  (for example, the `Span` could be one of the parameters passed to such a function,
  or a getter could be provided).
  
  使用 this 调用的函数必须有可能以某种方式获得 Span 创建 API 返回（或将返回）给用户的相同 Span 实例和类型（例如，Span 可能是传递给此类函数的参数之一） ，或者可以提供一个getter）。

## Sampling  采样

Sampling is a mechanism to control the noise and overhead introduced by
OpenTelemetry by reducing the number of samples of traces collected and sent to
the backend.

采样是一种通过减少收集并发送到后端的跟踪样本数量来控制 OpenTelemetry 引入的噪声和开销的机制。

Sampling may be implemented on different stages of a trace collection. The
earliest sampling could happen before the trace is actually created, and the
latest sampling could happen on the Collector which is out of process.

可以在跟踪收集的不同阶段实施采样。最早的采样可能发生在实际创建跟踪之前，而最新的采样可能发生在进程外的收集器上。

The OpenTelemetry API has two properties responsible for the data collection: OpenTelemetry API 有两个属性负责数据收集：

* `IsRecording` field of a `Span`. If `false` the current `Span` discards all
  tracing data (attributes, events, status, etc.). Users can use this property
  to determine if collecting expensive trace data can be avoided. [Span
  Processor](#span-processor) MUST receive only those spans which have this
  field set to `true`. However, [Span Exporter](#span-exporter) SHOULD NOT
  receive them unless the `Sampled` flag was also set.
  
  Span 的 IsRecording 字段。如果为 false，当前 Span 将丢弃所有跟踪数据（属性、事件、状态等）。用户可以使用此属性来确定是否可以避免收集昂贵的跟踪数据。Span处理器必须只接收那些将此字段设置为 true 的Span。但是，除非同时设置了 Sampled 标志，否则 Span Exporter 不应接收它们。
  
* `Sampled` flag in `TraceFlags` on `SpanContext`. This flag is propagated via
  the `SpanContext` to child Spans. For more details see the [W3C Trace Context
  specification](https://www.w3.org/TR/trace-context/#sampled-flag). This flag indicates that the `Span` has been
  `sampled` and will be exported. [Span Exporters](#span-exporter) MUST
  receive those spans which have `Sampled` flag set to true and they SHOULD NOT receive the ones
  that do not.
  
  SpanContext 上 TraceFlags 中的Sampled标志。此标志通过 SpanContext 传播到子 Span。有关更多详细信息，请参阅 W3C 跟踪上下文规范。此标志表示 Span 已被采样并将被导出。 Span 导出器必须接收那些将 Sampled 标志设置为 true 的 Span，并且他们不应接收那些没有设置的 Span。

The flag combination `SampledFlag == false` and `IsRecording == true`
means that the current `Span` does record information, but most likely the child
`Span` will not.

标志组合 SampledFlag == false 和 IsRecording == true 表示当前 Span 确实记录信息，但很可能子 Span 不会。

The flag combination `SampledFlag == true` and `IsRecording == false`
could cause gaps in the distributed trace, and because of this OpenTelemetry API
MUST NOT allow this combination.

标志组合 SampledFlag == true 和 IsRecording == false 可能会导致分布式跟踪出现间隙，并且由于此 OpenTelemetry API 不得允许这种组合。

<a name="recording-sampled-reaction-table"></a>

The following table summarizes the expected behavior for each combination of
`IsRecording` and `SampledFlag`.

| `IsRecording` | `Sampled` Flag | Span Processor receives Span? | Span Exporter receives Span? |
| ------------- | -------------- | ----------------------------- | ---------------------------- |
| true          | true           | true                          | true                         |
| true          | false          | true                          | false                        |
| false         | true           | Not allowed                   | Not allowed                  |
| false         | false          | false                         | false                        |

The SDK defines the interface [`Sampler`](#sampler) as well as a set of
[built-in samplers](#built-in-samplers) and associates a `Sampler` with each [`TracerProvider`].

SDK 定义了接口 Sampler 以及一组内置的采样器，并为每个 [TracerProvider] 关联了一个 Sampler。

### SDK Span creation

When asked to create a Span, the SDK MUST act as if doing the following in order:当要求创建 Span 时，SDK 必须像按顺序执行以下操作一样：

1. If there is a valid parent trace ID, use it. Otherwise generate a new trace ID
   (note: this must be done before calling `ShouldSample`, because it expects
   a valid trace ID as input).如果存在有效的父跟踪 ID，请使用它。否则生成一个新的跟踪 ID（注意：这必须在调用 ShouldSample 之前完成，因为它需要一个有效的跟踪 ID 作为输入）。
2. Query the `Sampler`'s [`ShouldSample`](#shouldsample) method
   (Note that the [built-in `ParentBasedSampler`](#parentbased) can be used to
   use the sampling decision of the parent,
   translating a set SampledFlag to RECORD and an unset one to DROP).查询 Sampler 的 ShouldSample 方法（请注意，内置的 ParentBasedSampler 可用于使用父级的采样决策，将设置的 SampledFlag 转换为 RECORD，将未设置的转换为 DROP）。
3. Generate a new span ID for the `Span`, independently of the sampling decision.
   This is done so other components (such as logs or exception handling) can rely on
   a unique span ID, even if the `Span` is a non-recording instance.为 Span 生成一个新的 span ID，与采样决策无关。这样做是为了其他组件（例如日志或异常处理）可以依赖唯一的跨度 ID，即使Span是非记录实例。
4. Create a span depending on the decision returned by `ShouldSample`:
   see description of [`ShouldSample`'s](#shouldsample) return value below
   for how to set `IsRecording` and `Sampled` on the Span,
   and the [table above](#recording-sampled-reaction-table) on whether
   to pass the `Span` to `SpanProcessor`s.
   A non-recording span MAY be implemented using the same mechanism as when a
   `Span` is created without an SDK installed or as described in
   [wrapping a SpanContext in a Span](api.md#wrapping-a-spancontext-in-a-span).根据 ShouldSample 返回的决定创建一个 span：有关如何在 Span 上设置 IsRecording 和 Sampled 的信息，请参阅下面对 ShouldSample 返回值的说明，以及有关是否将 Span 传递给 SpanProcessors 的上表。可以使用与在未安装 SDK 的情况下创建 Span 或将 SpanContext 包装在 Span 中所述相同的机制来实现非记录跨度。

### Sampler  取样器


`Sampler` interface allows users to create custom samplers which will return a
sampling `SamplingResult` based on information that is typically available just
before the `Span` was created.

采样器接口允许用户创建自定义采样器，该采样器将根据创建 Span 之前通常可用的信息返回采样 SamplingResult。

#### ShouldSample

Returns the sampling Decision for a `Span` to be created.

**Required arguments:**

* [`Context`](../context/context.md) with parent `Span`.
  The Span's SpanContext may be invalid to indicate a root span.
* `TraceId` of the `Span` to be created.
  If the parent `SpanContext` contains a valid `TraceId`, they MUST always match.
* Name of the `Span` to be created.
* `SpanKind` of the `Span` to be created.
* Initial set of `Attributes` of the `Span` to be created.
* Collection of links that will be associated with the `Span` to be created.
  Typically useful for batch operations, see
  [Links Between Spans](../overview.md#links-between-spans).

Note: Implementations may "bundle" all or several arguments together in a single
object.  注意：实现可以将所有或多个参数“捆绑”在一个对象中。

**Return value:**

It produces an output called `SamplingResult` which contains:

* A sampling `Decision`. One of the following enum values:
  * `DROP` - `IsRecording() == false`, span will not be recorded and all events and attributes
  will be dropped.
  * `RECORD_ONLY` - `IsRecording() == true`, but `Sampled` flag MUST NOT be set.
  * `RECORD_AND_SAMPLE` - `IsRecording() == true` AND `Sampled` flag MUST be set.
* A set of span Attributes that will also be added to the `Span`. The returned
object must be immutable (multiple calls may return different immutable objects).
* A `Tracestate` that will be associated with the `Span` through the new
  `SpanContext`.
  If the sampler returns an empty `Tracestate` here, the `Tracestate` will be cleared,
  so samplers SHOULD normally return the passed-in `Tracestate` if they do not intend
  to change it.

#### GetDescription

Returns the sampler name or short description with the configuration. This may
be displayed on debug pages or in the logs. Example:
`"TraceIdRatioBased{0.000100}"`.

Description MUST NOT change over time and caller can cache the returned value.

### Built-in samplers  内置采样器


OpenTelemetry supports a number of built-in samplers to choose from.
The default sampler is `ParentBased(root=AlwaysOn)`.

OpenTelemetry 支持多种内置采样器可供选择。默认采样器是 ParentBased(root=AlwaysOn)。

#### AlwaysOn

* Returns `RECORD_AND_SAMPLE` always.
* Description MUST be `AlwaysOnSampler`.

#### AlwaysOff

* Returns `DROP` always.
* Description MUST be `AlwaysOffSampler`.

#### TraceIdRatioBased

* The `TraceIdRatioBased` MUST ignore the parent `SampledFlag`. To respect the
parent `SampledFlag`, the `TraceIdRatioBased` should be used as a delegate of
the `ParentBased` sampler specified below.
* Description MUST return a string of the form `"TraceIdRatioBased{RATIO}"`
  with `RATIO` replaced with the Sampler instance's trace sampling ratio
  represented as a decimal number. The precision of the number SHOULD follow
  implementation language standards and SHOULD be high enough to identify when
  Samplers have different ratios. For example, if a TraceIdRatioBased Sampler
  had a sampling ratio of 1 to every 10,000 spans it COULD return
  `"TraceIdRatioBased{0.000100}"` as its description.

TODO: Add details about how the `TraceIdRatioBased` is implemented as a function
of the `TraceID`. [#1413](https://github.com/open-telemetry/opentelemetry-specification/issues/1413)

##### Requirements for `TraceIdRatioBased` sampler algorithm

* The sampling algorithm MUST be deterministic. A trace identified by a given
  `TraceId` is sampled or not independent of language, time, etc. To achieve this,
  implementations MUST use a deterministic hash of the `TraceId` when computing
  the sampling decision. By ensuring this, running the sampler on any child `Span`
  will produce the same decision.
* A `TraceIdRatioBased` sampler with a given sampling rate MUST also sample all
  traces that any `TraceIdRatioBased` sampler with a lower sampling rate would
  sample. This is important when a backend system may want to run with a higher
  sampling rate than the frontend system, this way all frontend traces will
  still be sampled and extra traces will be sampled on the backend only.
* **WARNING:** Since the exact algorithm is not specified yet (see TODO above),
  there will probably be changes to it in any language SDK once it is, which
  would break code that relies on the algorithm results.
  Only the configuration and creation APIs can be considered stable.
  It is recommended to use this sampler algorithm only for root spans
  (in combination with [`ParentBased`](#parentbased)) because different language
  SDKs or even different versions of the same language SDKs may produce inconsistent
  results for the same input.

#### ParentBased

* This is a composite sampler. `ParentBased` helps distinguish between the
following cases:
  * No parent (root span).
  * Remote parent (`SpanContext.IsRemote() == true`) with `SampledFlag` equals `true`
  * Remote parent (`SpanContext.IsRemote() == true`) with `SampledFlag` equals `false`
  * Local parent (`SpanContext.IsRemote() == false`) with `SampledFlag` equals `true`
  * Local parent (`SpanContext.IsRemote() == false`) with `SampledFlag` equals `false`

Required parameters:

* `root(Sampler)` - Sampler called for spans with no parent (root spans)

Optional parameters:

* `remoteParentSampled(Sampler)` (default: AlwaysOn)
* `remoteParentNotSampled(Sampler)` (default: AlwaysOff)
* `localParentSampled(Sampler)` (default: AlwaysOn)
* `localParentNotSampled(Sampler)` (default: AlwaysOff)

|Parent| parent.isRemote() | parent.IsSampled()| Invoke sampler|
|--|--|--|--|
|absent| n/a | n/a |`root()`|
|present|true|true|`remoteParentSampled()`|
|present|true|false|`remoteParentNotSampled()`|
|present|false|true|`localParentSampled()`|
|present|false|false|`localParentNotSampled()`|

## Span Limits

Span attributes MUST adhere to the [common rules of attribute limits](../common/common.md#attribute-limits).

Span属性必须遵守属性限制的通用规则。


SDK Spans MAY also discard links and events that would increase the number of
elements of each collection beyond the configured limit.

SDK Span 还可以丢弃会增加每个集合的元素数量超出配置限制的链接和事件。

If the SDK implements the limits above it MUST provide a way to change these
limits, via a configuration to the TracerProvider, by allowing users to
configure individual limits like in the Java example bellow.

如果 SDK 实现了上面的限制，它必须提供一种方法来更改这些限制，通过配置到 TracerProvider，允许用户像下面的 Java 示例中那样配置单个限制。

The name of the configuration options SHOULD be `EventCountLimit` and `LinkCountLimit`. The options MAY be bundled in a class,
which then SHOULD be called `SpanLimits`. Implementations MAY provide additional
configuration such as `AttributePerEventCountLimit` and `AttributePerLinkCountLimit`.

配置选项的名称应该是 EventCountLimit 和 LinkCountLimit。这些选项可以捆绑在一个类中，然后应该称为 SpanLimits。实现可以提供额外的配置，例如 AttributePerEventCountLimit 和 AttributePerLinkCountLimit。

```java
public final class SpanLimits {
  SpanLimits(int attributeCountLimit, int linkCountLimit, int eventCountLimit);

  public int getAttributeCountLimit();

  public int getAttributeCountPerEventLimit();

  public int getAttributeCountPerLinkLimit();

  public int getEventCountLimit();

  public int getLinkCountLimit();
}
```

**Configurable parameters:**

* [all common options applicable to attributes](../common/common.md#attribute-limits-configuration)
* `EventCountLimit` (Default=128) - Maximum allowed span event count;
* `LinkCountLimit` (Default=128) - Maximum allowed span link count;
* `AttributePerEventCountLimit` (Default=128) - Maximum allowed attribute per span event count;
* `AttributePerLinkCountLimit` (Default=128) - Maximum allowed attribute per span link count;

There SHOULD be a log emitted to indicate to the user that an attribute, event,
or link was discarded due to such a limit. To prevent excessive logging, the log
should not be emitted once per span, or per discarded attribute, event, or links.

## Id Generators

The SDK MUST by default randomly generate both the `TraceId` and the `SpanId`.

默认情况下，SDK 必须随机生成 TraceId 和 SpanId。

The SDK MUST provide a mechanism for customizing the way IDs are generated for
both the `TraceId` and the `SpanId`.

The SDK MAY provide this functionality by allowing custom implementations of
an interface like the java example below (name of the interface MAY be
`IdGenerator`, name of the methods MUST be consistent with
[SpanContext](./api.md#retrieving-the-traceid-and-spanid)), which provides
extension points for two methods, one to generate a `SpanId` and one for `TraceId`.

```java
public interface IdGenerator {
  byte[] generateSpanIdBytes();
  byte[] generateTraceIdBytes();
}
```

Additional `IdGenerator` implementing vendor-specific protocols such as AWS
X-Ray trace id generator MUST NOT be maintained or distributed as part of the
Core OpenTelemetry repositories.

## Span processor

Span processor is an interface which allows hooks for span start and end method
invocations. The span processors are invoked only when
[`IsRecording`](api.md#isrecording) is true.

Span处理器是一个接口，它允许Span开始和结束方法调用的钩子。只有当 IsRecording 为真时才会调用跨度处理器

Built-in span processors are responsible for batching and conversion of spans to
exportable representation and passing batches to exporters.

内置的span处理器负责将span批处理和转换为可导出的表示并将批处理传递给exporters。

Span processors can be registered directly on SDK `TracerProvider` and they are
invoked in the same order as they were registered.

Span 处理器可以直接在 SDK TracerProvider 上注册，它们的调用顺序与注册的顺序相同。

Each processor registered on `TracerProvider` is a start of pipeline that consist
of span processor and optional exporter. SDK MUST allow to end each pipeline with
individual exporter.

TracerProvider 上注册的每个处理器都是管道的开始，由跨度处理器和可选的导出器组成。 SDK 必须允许以单独的导出器结束每个管道。

SDK MUST allow users to implement and configure custom processors and decorate
built-in processors for advanced scenarios such as tagging or filtering.

SDK 必须允许用户实现和配置自定义处理器，并为标记或过滤等高级场景装饰内置处理器。

The following diagram shows `SpanProcessor`'s relationship to other components
in the SDK:下图显示了 SpanProcessor 与 SDK 中其他组件的关系：

```
  +-----+--------------+   +-------------------------+   +-------------------+
  |     |              |   |                         |   |                   |
  |     |              |   | Batching Span Processor |   |    SpanExporter   |
  |     |              +---> Simple Span Processor   +--->  (JaegerExporter) |
  |     |              |   |                         |   |                   |
  | SDK | Span.start() |   +-------------------------+   +-------------------+
  |     | Span.end()   |
  |     |              |
  |     |              |
  |     |              |
  |     |              |
  +-----+--------------+
```

### Interface definition

#### OnStart

`OnStart` is called when a span is started. This method is called synchronously
on the thread that started the span, therefore it should not block or throw
exceptions.

当跨度启动时调用 OnStart。此方法在启动跨度的线程上同步调用，因此它不应阻塞或抛出异常。

**Parameters:**

* `span` - a [read/write span object](#additional-span-interfaces) for the started span.
  It SHOULD be possible to keep a reference to this span object and updates to the span
  SHOULD be reflected in it.
  For example, this is useful for creating a SpanProcessor that periodically
  evaluates/prints information about all active span from a background thread.
  
  启动跨度的读/写跨度对象。应该可以保留对这个 span 对象的引用，并且应该在其中反映对 span 的更新。例如，这对于创建一个 SpanProcessor 很有用，该 SpanProcessor 定期评估/打印有关来自后台线程的所有活动跨度的信息。
  
* `parentContext` - the parent `Context` of the span that the SDK determined
  (the explicitly passed `Context`, the current `Context` or an empty `Context`
  if that was explicitly requested).
  
  parentContext - SDK 确定的跨度的父 Context（显式传递的 Context、当前 Context 或空的 Context，如果明确请求）。

**Returns:** `Void`

#### OnEnd(Span)

`OnEnd` is called after a span is ended (i.e., the end timestamp is already set).
This method MUST be called synchronously within the [`Span.End()` API](api.md#end),
therefore it should not block or throw an exception.

在跨度结束后调用 OnEnd（即，结束时间戳已经设置）。此方法必须在 Span.End() API 中同步调用，因此它不应阻塞或抛出异常。

**Parameters:**

* `Span` - a [readable span object](#additional-span-interfaces) for the ended span.
  Note: Even if the passed Span may be technically writable,
  since it's already ended at this point, modifying it is not allowed.

**Returns:** `Void`

#### Shutdown()

Shuts down the processor. Called when SDK is shut down. This is an opportunity
for processor to do any cleanup required.

关闭处理器。 SDK 关闭时调用。这是处理器进行任何所需清理的机会。

`Shutdown` SHOULD be called only once for each `SpanProcessor` instance. After
the call to `Shutdown`, subsequent calls to `OnStart`, `OnEnd`, or `ForceFlush`
are not allowed. SDKs SHOULD ignore these calls gracefully, if possible.

`Shutdown` SHOULD provide a way to let the caller know whether it succeeded,
failed or timed out.

`Shutdown` MUST include the effects of `ForceFlush`.

`Shutdown` SHOULD complete or abort within some timeout. `Shutdown` can be
implemented as a blocking API or an asynchronous API which notifies the caller
via a callback or an event. OpenTelemetry client authors can decide if they want to
make the shutdown timeout configurable.

#### ForceFlush()

This is a hint to ensure that any tasks associated with `Spans` for which the
`SpanProcessor` had already received events prior to the call to `ForceFlush` SHOULD
be completed as soon as possible, preferably before returning from this method.

这是一个提示，以确保在调用 ForceFlush 之前，SpanProcessor 已经接收到事件的任何与 Span 关联的任务都应该尽快完成，最好在从此方法返回之前完成。

In particular, if any `SpanProcessor` has any associated exporter, it SHOULD
try to call the exporter's `Export` with all spans for which this was not
already done and then invoke `ForceFlush` on it.
The [built-in SpanProcessors](#built-in-span-processors) MUST do so.
If a timeout is specified (see below), the SpanProcessor MUST prioritize honoring the timeout over
finishing all calls. It MAY skip or abort some or all Export or ForceFlush
calls it has made to achieve this goal.

`ForceFlush` SHOULD provide a way to let the caller know whether it succeeded,
failed or timed out.

`ForceFlush` SHOULD only be called in cases where it is absolutely necessary,
such as when using some FaaS providers that may suspend the process after an
invocation, but before the `SpanProcessor` exports the completed spans.

`ForceFlush` SHOULD complete or abort within some timeout. `ForceFlush` can be
implemented as a blocking API or an asynchronous API which notifies the caller
via a callback or an event. OpenTelemetry client authors can decide if they want to
make the flush timeout configurable.

### Built-in span processors

The standard OpenTelemetry SDK MUST implement both simple and batch processors,
as described below. Other common processing scenarios should be first considered
for implementation out-of-process in [OpenTelemetry Collector](../overview.md#collector)

#### Simple processor

This is an implementation of `SpanProcessor` which passes finished spans
and passes the export-friendly span data representation to the configured
`SpanExporter`, as soon as they are finished.

**Configurable parameters:**

* `exporter` - the exporter where the spans are pushed.

#### Batching processor

This is an implementation of the `SpanProcessor` which create batches of finished
spans and passes the export-friendly span data representations to the
configured `SpanExporter`.

**Configurable parameters:**

* `exporter` - the exporter where the spans are pushed.
* `maxQueueSize` - the maximum queue size. After the size is reached spans are
  dropped. The default value is `2048`.
* `scheduledDelayMillis` - the delay interval in milliseconds between two
  consecutive exports. The default value is `5000`.
* `exportTimeoutMillis` - how long the export can run before it is cancelled.
  The default value is `30000`.
* `maxExportBatchSize` - the maximum batch size of every export. It must be
  smaller or equal to `maxQueueSize`. The default value is `512`.

## Span Exporter

`Span Exporter` defines the interface that protocol-specific exporters must
implement so that they can be plugged into OpenTelemetry SDK and support sending
of telemetry data.

Span 导出器定义了特定于协议的导出器必须实现的接口，以便它们可以插入 OpenTelemetry SDK 并支持遥测数据的发送。

The goal of the interface is to minimize burden of implementation for
protocol-dependent telemetry exporters. The protocol exporter is expected to be
primarily a simple telemetry data encoder and transmitter.

该接口的目标是最大程度地减少依赖于协议的遥测导出器的实现负担。协议导出器主要是一个简单的遥测数据编码器和发射器。

### Interface Definition

The exporter must support two functions: **Export** and **Shutdown**. In
strongly typed languages typically there will be 2 separate `Exporter`
interfaces, one that accepts spans (SpanExporter) and one that accepts metrics
(MetricsExporter).

导出器必须支持两种功能：导出和关闭。在强类型语言中，通常会有 2 个单独的 Exporter 接口，一个接受跨度 (SpanExporter)，另一个接受度量 (MetricsExporter)。

#### `Export(batch)`

Exports a batch of [readable spans](#additional-span-interfaces).
Protocol exporters that will implement this
function are typically expected to serialize and transmit the data to the
destination.

导出一批可读的跨度。将实现此功能的协议导出器通常需要序列化数据并将其传输到目的地。

Export() will never be called concurrently for the same exporter instance.
Export() can be called again only after the current call returns.

Export() 永远不会为同一个导出器实例同时调用。 Export() 只有在当前调用返回后才能再次调用

Export() MUST NOT block indefinitely, there MUST be a reasonable upper limit
after which the call must time out with an error result (`Failure`).

Any retry logic that is required by the exporter is the responsibility
of the exporter. The default SDK SHOULD NOT implement retry logic, as
the required logic is likely to depend heavily on the specific protocol
and backend the spans are being sent to.

**Parameters:**

batch - a batch of [readable spans](#additional-span-interfaces). The exact data type of the batch is language
specific, typically it is some kind of list,
e.g. for spans in Java it will be typically `Collection<SpanData>`.

**Returns:** ExportResult:

ExportResult is one of:

* `Success` - The batch has been successfully exported.
  For protocol exporters this typically means that the data is sent over
  the wire and delivered to the destination server.
* `Failure` - exporting failed. The batch must be dropped. For example, this
  can happen when the batch contains bad data and cannot be serialized.

Note: this result may be returned via an async mechanism or a callback, if that
is idiomatic for the language implementation.

#### `Shutdown()`

Shuts down the exporter. Called when SDK is shut down. This is an opportunity
for exporter to do any cleanup required.

`Shutdown` should be called only once for each `Exporter` instance. After the
call to `Shutdown` subsequent calls to `Export` are not allowed and should
return a `Failure` result.

`Shutdown` should not block indefinitely (e.g. if it attempts to flush the data
and the destination is unavailable). OpenTelemetry client authors can decide if they
want to make the shutdown timeout configurable.

#### `ForceFlush()`

This is a hint to ensure that the export of any `Spans` the exporter has received prior to the
call to `ForceFlush` SHOULD be completed as soon as possible, preferably before
returning from this method.

`ForceFlush` SHOULD provide a way to let the caller know whether it succeeded,
failed or timed out.

`ForceFlush` SHOULD only be called in cases where it is absolutely necessary,
such as when using some FaaS providers that may suspend the process after an
invocation, but before the exporter exports the completed spans.

ForceFlush 应该只在绝对必要的情况下被调用，例如当使用一些 FaaS 提供者时，这些提供者可能会在调用后暂停流程，但在导出器导出已完成的跨度之前。

`ForceFlush` SHOULD complete or abort within some timeout. `ForceFlush` can be
implemented as a blocking API or an asynchronous API which notifies the caller
via a callback or an event. OpenTelemetry client authors can decide if they want to
make the flush timeout configurable.

ForceFlush 应该在某个超时时间内完成或中止。 ForceFlush 可以实现为阻塞 API 或异步 API，通过回调或事件通知调用者。 OpenTelemetry 客户端作者可以决定是否要使刷新超时可配置。


### Further Language Specialization  将来的语言规范


Based on the generic interface definition laid out above library authors must
define the exact interface for the particular language.

基于上面列出的通用接口定义，库作者必须为特定语言定义确切的接口。

Authors are encouraged to use efficient data structures on the interface
boundary that are well suited for fast serialization to wire formats by protocol
exporters and minimize the pressure on memory managers. The latter typically
requires understanding of how to optimize the rapidly-generated, short-lived
telemetry data structures to make life easier for the memory manager of the
specific language. General recommendation is to minimize the number of
allocations and use allocation arenas where possible, thus avoiding explosion of
allocation/deallocation/collection operations in the presence of high rate of
telemetry data generation.

鼓励作者在接口边界上使用高效的数据结构，这些结构非常适合协议导出器快速序列化为有线格式，并最大限度地减少内存管理器的压力。后者通常需要了解如何优化快速生成的、短期的遥测数据结构，以使特定语言的内存管理器更轻松。一般建议是在可能的情况下尽量减少分配数量并使用分配区域，从而避免在遥测数据生成率很高的情况下分配/解除分配/收集操作的爆炸性增长。

#### Examples

These are examples on what the `Exporter` interface can look like in specific
languages. Examples are for illustration purposes only. OpenTelemetry client authors
are free to deviate from these provided that their design remain true to the
spirit of `Exporter` concept.

这些示例说明了导出器界面在特定语言中的外观。示例仅用于说明目的。 OpenTelemetry 客户端作者可以自由地偏离这些，前提是他们的设计仍然忠于 Exporter 概念的精神。

##### Go SpanExporter Interface

```go
type SpanExporter interface {
    Export(batch []ExportableSpan) ExportResult
    Shutdown()
}

type ExportResult struct {
    Code         ExportResultCode
    WrappedError error
}

type ExportResultCode int

const (
    Success ExportResultCode = iota
    Failure
)
```

##### Java SpanExporter Interface

```java
public interface SpanExporter {
 public enum ResultCode {
   Success, Failure
 }

 ResultCode export(Collection<ExportableSpan> batch);
 void shutdown();
}
```

[trace-flags]: https://www.w3.org/TR/trace-context/#trace-flags
[otep-83]: https://github.com/open-telemetry/oteps/blob/main/text/0083-component.md
