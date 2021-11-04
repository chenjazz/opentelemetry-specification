# Glossary

This document defines some terms that are used across this specification.

Some other fundamental terms are documented in the [overview document](overview.md).

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [User Roles](#user-roles)
  * [Application Owner](#application-owner)
  * [Library Author](#library-author)
  * [Instrumentation Author](#instrumentation-author)
  * [Plugin Author](#plugin-author)
- [Common](#common)
  * [Signals](#signals)
  * [Packages](#packages)
  * [ABI Compatibility](#abi-compatibility)
  * [In-band and Out-of-band Data](#in-band-and-out-of-band-data)
  * [Manual Instrumentation](#manual-instrumentation)
  * [Automatic Instrumentation](#automatic-instrumentation)
  * [Telemetry SDK](#telemetry-sdk)
  * [Constructors](#constructors)
  * [SDK Plugins](#sdk-plugins)
  * [Exporter Library](#exporter-library)
  * [Instrumented Library](#instrumented-library)
  * [Instrumentation Library](#instrumentation-library)
  * [Tracer Name / Meter Name](#tracer-name--meter-name)
- [Logs](#logs)
  * [Log Record](#log-record)
  * [Log](#log)
  * [Embedded Log](#embedded-log)
  * [Standalone Log](#standalone-log)
  * [Log Attributes](#log-attributes)
  * [Structured Logs](#structured-logs)
  * [Flat File Logs](#flat-file-logs)

<!-- tocstop -->

## User Roles

### Application Owner  应用程序所有者


The maintainer of an application or service, responsible for configuring and managing the lifecycle of the OpenTelemetry SDK.

应用程序或服务的维护者，负责配置和管理 OpenTelemetry SDK 的生命周期。

### Library Author  库作者

The maintainer of a shared library which is depended upon by many applications, and targeted by OpenTelemetry instrumentation.

许多应用程序所依赖的共享库的维护者，并以 OpenTelemetry instrumentation为目标。


### Instrumentation Author

The maintainer of OpenTelemetry instrumentation written against the OpenTelemetry API.
This may be instrumentation written within application code, within a shared library, or within an instrumentation library.

针对 OpenTelemetry API 编写的 OpenTelemetry 工具的维护者。这可能是在应用程序代码、共享库或检测库中编写的检测。

### Plugin Author 插件作者

The maintainer of an OpenTelemetry SDK Plugin, written against OpenTelemetry SDK plugin interfaces.

OpenTelemetry SDK 插件的维护者，针对 OpenTelemetry SDK 插件接口编写。

## Common

### Signals

OpenTelemetry is structured around signals, or categories of telemetry.
Metrics, logs, traces, and baggage are examples of signals.
Each signal represents a coherent, stand-alone set of functionality.
Each signal follows a separate lifecycle, defining its current stability level.

OpenTelemetry 是围绕信号或遥测类别构建的。Metrics, logs, traces, and baggage 都是Signal的例子。每个Signal代表一组连贯的、独立的功能。每个Signal遵循单独的生命周期，定义其当前的稳定性水平。

### Packages

In this specification, the term **package** describes a set of code which represents a single dependency, which may be imported into a program independently from other packages.
This concept may map to a different term in some languages, such as "module."
Note that in some languages, the term "package" refers to a different concept.

在本规范中，术语包描述了一组表示单个依赖项的代码，这些代码可以独立于其他包导入到程序中。这个概念在某些语言中可能映射到不同的术语，例如“模块”。请注意，在某些语言中，术语“包”指的是不同的概念。

### ABI Compatibility  ABI 兼容性

An ABI (application binary interface) is an interface which defines interactions between software components at the machine code level, for example between an application executable and a compiled binary of a shared object library. ABI compatibility means that a new compiled version of a library may be correctly linked to a target executable without the need for that executable to be recompiled.2

ABI（应用程序二进制接口）是在机器代码级别定义软件组件之间交互的接口，例如应用程序可执行文件和共享对象库的编译二进制文件之间的交互。 ABI 兼容性意味着库的新编译版本可以正确链接到目标可执行文件，而无需重新编译该可执行文件。

ABI compatibility is important for some languages, especially those which provide a form of machine code. For other languages, ABI compatibility may not be a relevant requirement.

ABI 兼容性对于某些语言很重要，尤其是那些提供机器代码形式的语言。对于其他语言，ABI 兼容性可能不是相关要求。

<a name="in-band"></a>
<a name="out-of-band"></a>

### In-band and Out-of-band Data  带内和带外数据？


> In telecommunications, **in-band signaling** is the sending of control information within the same band or channel used for data such as voice or video. This is in contrast to **out-of-band signaling** which is sent over a different channel, or even over a separate network ([Wikipedia](https://en.wikipedia.org/wiki/In-band_signaling)).

> 在电信中，带内信令是在用于语音或视频等数据的同一频带或信道内发送控制信息。这与通过不同信道或 通过单独网络发送的带外信令形成对比

In OpenTelemetry we refer to **in-band data** as data that is passed
between components of a distributed system as part of business messages,
for example, when trace or baggages are included
in the HTTP requests in the form of HTTP headers.
Such data usually does not contain the telemetry,
but is used to correlate and join the telemetry produced by various components.
The telemetry itself is referred to as **out-of-band data**:
it is transmitted from applications via dedicated messages,
usually asynchronously by background routines
rather than from the critical path of the business logic.
Metrics, logs, and traces exported to telemetry backends are examples of out-of-band data.

在 OpenTelemetry 中，我们将带内数据称为作为业务消息的一部分在分布式系统的组件之间传递的数据，例如，当trace or baggages以 HTTP 标头的形式包含在 HTTP 请求中时。此类数据通常不包含遥测数据，但用于关联和连接由各种组件产生的遥测数据。遥测本身被称为带外数据：它通过专用消息从应用程序传输，通常由后台例程异步传输，而不是从业务逻辑的关键路径传输。导出到遥测后端的指标、日志和跟踪是带外数据的示例。

### Manual Instrumentation

Coding against the OpenTelemetry API such as the [Tracing API](trace/api.md), [Metrics API](metrics/api.md), or others to collect telemetry from end-user code or shared frameworks (e.g. MongoDB, Redis, etc.).

针对 OpenTelemetry API（例如 Tracing API、Metrics API 或其他）进行编码，以从最终用户代码或共享框架（例如 MongoDB、Redis 等）收集遥测数据。

### Automatic Instrumentation

Refers to telemetry collection methods that do not require the end-user to write or access application code to use the OpenTelemetry APIs. Methods vary by programming language, and examples include bytecode injection or monkey patching.

指的是不需要最终用户编写或访问应用程序代码即可使用 OpenTelemetry API 的遥测收集方法。方法因编程语言而异，示例包括字节码注入或monkey patching。

Synonym: *Auto-instrumentation*.

### Telemetry SDK

Denotes the library that implements the *OpenTelemetry API*.  

表示实现 OpenTelemetry API 的库。

See [Library Guidelines](library-guidelines.md#sdk-implementation) and
[Library resource semantic conventions](resource/semantic_conventions/README.md#telemetry-sdk).

### Constructors

Constructors are public code used by Application Owners to initialize and configure the OpenTelemetry SDK and contrib packages. Examples of constructors include configuration objects, environment variables, and builders.

构造函数是应用程序所有者用来初始化和配置 OpenTelemetry SDK 和 contrib 包的公共代码。构造器的示例包括配置对象、环境变量和构建器。

### SDK Plugins

Plugins are libraries which extend the OpenTelemetry SDK. Examples of plugin interfaces are the `SpanProcessor`, `Exporter`, and `Sampler` interfaces.

插件是扩展 OpenTelemetry SDK 的库。插件接口的示例是 SpanProcessor、Exporter 和 Sampler 接口。

### Exporter Library

Exporters are SDK Plugins which implement the `Exporter` interface, and emit telemetry to consumers.

导出器是实现导出器接口并向消费者发出遥测数据的 SDK 插件。

### Instrumented Library

Denotes the library for which the telemetry signals (traces, metrics, logs) are gathered.

表示收集遥测signals（跟踪、指标、日志）的库。

The calls to the OpenTelemetry API can be done either by the Instrumented Library itself,
or by another [Instrumentation Library](#instrumentation-library).

对 OpenTelemetry API 的调用可以由 Instrumented Library 本身完成，也可以由另一个 Instrumentation Library 完成。

Example: `org.mongodb.client`.

### Instrumentation Library

Denotes the library that provides the instrumentation for a given [Instrumented Library](#instrumented-library).
*Instrumented Library* and *Instrumentation Library* may be the same library
if it has built-in OpenTelemetry instrumentation.

表示为给定的检测库提供检测的库。 Instrumented Library 和 Instrumentation Library 可能是同一个库，如果它有内置的 OpenTelemetry 工具的话。

See [Overview](overview.md#instrumentation-libraries) for a more detailed definition and naming guidelines.

Example: `io.opentelemetry.contrib.mongodb`.

Synonyms: *Instrumenting Library*.

### Tracer Name / Meter Name

This refers to the `name` and (optional) `version` arguments specified when
creating a new `Tracer` or `Meter` (see [Obtaining a Tracer](trace/api.md#tracerprovider)/[Obtaining a Meter](metrics/api.md#meterprovider)).
The name/version pair identifies the [Instrumentation Library](#instrumentation-library).

这是指在创建新 Tracer 或 Meter 时指定的名称和（可选）版本参数（请参阅获取跟踪器/获取仪表）。名称/版本对标识Instrumentation Library。

## Logs

### Log Record

A recording of an event. Typically the record includes a timestamp indicating
when the event happened as well as other data that describes what happened,
where it happened, etc.

一个事件的记录。通常，记录包括指示事件发生时间的时间戳以及描述发生的事情、发生地点等的其他数据。

Synonyms: *Log Entry*.

### Log

Sometimes used to refer to a collection of Log Records. May be ambiguous, since
people also sometimes use `Log` to refer to a single `Log Record`, thus this
term should be used carefully and in the context where ambiguity is possible
additional qualifiers should be used (e.g. `Log Record`).

有时用于指代日志记录的集合。可能是模棱两可的，因为人们有时也使用 Log 来指代单个日志记录，因此应谨慎使用该术语，并且在可能存在歧义的上下文中应使用其他限定符（例如日志记录）

### Embedded Log

`Log Records` embedded inside a [Span](trace/api.md#span)
object, in the [Events](trace/api.md#add-events) list.

在事件列表中嵌入 Span 对象中的日志记

### Standalone Log

`Log Records` that are not embedded inside a `Span` and are recorded elsewhere.

未嵌入 Span 并记录在其他地方的日志记录。

### Log Attributes

Key/value pairs contained in a `Log Record`.

日志记录中包含的键/值对。

### Structured Logs

Logs that are recorded in a format which has a well-defined structure that allows
to differentiate between different elements of a Log Record (e.g. the Timestamp,
the Attributes, etc). The _Syslog protocol_ ([RFC 5424](https://tools.ietf.org/html/rfc5424)),
for example, defines a `structured-data` format.

以具有明确定义结构的格式记录的日志，允许区分日志记录的不同元素（例如时间戳、属性等）。例如，Syslog 协议 (RFC 5424) 定义了结构化数据格式。

### Flat File Logs  纯文本日志

Logs recorded in text files, often one line per log record (although multiline
records are possible too). There is no common industry agreement whether
logs written to text files in more structured formats (e.g. JSON files)
are considered Flat File Logs or not. Where such distinction is important it is
recommended to call it out specifically.

记录在文本文件中的日志，通常每个日志记录一行（尽管多行记录也是可能的）。以更结构化的格式（例如 JSON 文件）写入文本文件的日志是否被视为Flat File日志，目前还没有达成共识。在这种区别很重要的地方，建议特别指出。
