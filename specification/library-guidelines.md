# OpenTelemetry Client Design Principles  客户端设计原则

This document defines common principles that will help designers create OpenTelemetry clients that are easy to use, are uniform across all supported languages, yet allow enough flexibility for language-specific expressiveness.

本文档定义了通用原则，这些原则将帮助设计人员创建易于使用的 OpenTelemetry 客户端，这些客户端在所有支持的语言中都是统一的，但为特定语言的表达提供了足够的灵活性。

OpenTelemetry clients are expected to provide full features out of the box and allow for innovation and experimentation through extensibility.

OpenTelemetry 客户端有望提供开箱即用的全部功能，并允许通过可扩展性进行创新和实验。

Please read the [overview](overview.md) first, to understand the fundamental architecture of OpenTelemetry.

请先阅读overview，以了解 OpenTelemetry 的基本架构。

This document does not attempt to describe the details or functionality of the OpenTelemetry client API. For API specs see the [API specifications](../README.md).

本文档不试图描述 OpenTelemetry 客户端 API 的细节或功能。有关 API 规范，请参阅 API 规范。

_Note to OpenTelemetry client Authors:_ OpenTelemetry specification, API and SDK implementation guidelines are work in progress. If you notice incomplete or missing information, contradictions, inconsistent styling and other defects please let specification writers know by creating an issue in this repository or posting in [Slack](https://cloud-native.slack.com/archives/C01N7PP1THC). As implementors of the specification you will often have valuable insights into how the specification can be improved. The Specification SIG and members of Technical Committee highly value your opinion and welcome your feedback.

OpenTelemetry 客户端作者须知：OpenTelemetry 规范、API 和 SDK 实施指南正在制定中。如果您发现不完整或缺失的信息、矛盾、不一致的样式和其他缺陷，请通过在此存储库中创建issue或在 Slack 中发布来告知规范作者。作为规范的实现者，您通常会对如何改进规范有宝贵的见解。规范SIG和技术委员会成员高度重视您的意见，欢迎您的反馈。

## Requirements  要求


1. The OpenTelemetry API must be well-defined and clearly decoupled from the implementation. This allows end users to consume API only without also consuming the implementation (see points 2 and 3 for why it is important).

OpenTelemetry API 必须明确定义并与实现明确分离。这允许最终用户仅使用 API，而无需使用实现（请参阅第 2 点和第 3 点了解其重要性）。

2. Third party libraries and frameworks that add instrumentation to their code will have a dependency only on the API of OpenTelemetry client. The developers of third party libraries and frameworks do not care (and cannot know) what specific implementation of OpenTelemetry is used in the final application.

向其代码添加 instrumentation 的第三方库和框架将仅依赖于 OpenTelemetry 客户端的 API。第三方库和框架的开发人员并不关心（也不知道）最终应用程序中使用了 OpenTelemetry 的具体实现。

3. The developers of the final application normally decide how to configure OpenTelemetry SDK and what extensions to use. They should be also free to choose to not use any OpenTelemetry implementation at all, even though the application and/or its libraries are already instrumented.  The rationale is that third-party libraries and frameworks which are instrumented with OpenTelemetry must still be fully usable in the applications which do not want to use OpenTelemetry (so this removes the need for framework developers to have "instrumented" and "non-instrumented" versions of their framework).

最终应用程序的开发人员通常决定如何配置 OpenTelemetry SDK 以及使用哪些扩展。他们也应该可以自由选择根本不使用任何 OpenTelemetry 实现，即使应用程序和/或其库已经被instrumented。基本原理是使用 OpenTelemetry 检测的第三方库和框架仍然必须在不想使用 OpenTelemetry 的应用程序中完全可用（因此这消除了框架开发人员“instrumented”和“non-instrumented”的需要他们框架的版本）

4. The SDK must be clearly separated into wire protocol-independent parts that implement common logic (e.g. batching, tag enrichment by process information, etc.) and protocol-dependent telemetry exporters. Telemetry exporters must contain minimal functionality, thus enabling vendors to easily add support for their specific protocol.

SDK 必须清楚地分为实现通用逻辑（例如批处理、通过过程信息丰富标签等）的与有线协议无关的部分和依赖于协议的telemetry exporters。Telemetry exporters必须包含最少的功能，从而使供应商能够轻松添加对其特定协议的支持。

5. The SDK implementation should include the following exporters: SDK 实现应包括以下导出器：

    - logs, metrics, trace
        - OTLP (OpenTelemetry Protocol).OpenTelemetry协议
        - Standard output (or logging) to use for debugging and testing as well as an input for the various log proxy tools.用于调试和测试的标准输出（或日志记录）以及各种日志代理工具的输入。

        - In-memory (mock) exporter that accumulates telemetry data in the local memory and allows to inspect it (useful for e.g. unit tests).内存中（模拟）导出器在本地内存中累积遥测数据并允许检查它（例如用于单元测试）。
    - metrics
        - Prometheus.
    - trace
        - Jaeger.
        - Zipkin.

    Note: some of these support multiple protocols (e.g. gRPC, Thrift, etc). The exact list of protocols to implement in the exporters is TBD.
    
    注意：其中一些支持多种协议（例如 gRPC、Thrift 等）。要在exporters中实施的协议的确切列表 待确定。

    Other vendor-specific exporters (exporters that implement vendor protocols) should not be included in OpenTelemetry clients and should be placed elsewhere (the exact approach for storing and maintaining vendor-specific exporters will be defined in the future).
    
    其他特定于供应商的导出器（实现供应商协议的导出器）不应包含在 OpenTelemetry 客户端中，而应放在其他地方（存储和维护特定于供应商的导出器的确切方法将在未来定义）。

## OpenTelemetry Client Generic Design  通用设计

Here is a generic design for an OpenTelemetry client (arrows indicate calls):

这是 OpenTelemetry 客户端的通用设计（箭头表示调用）：

![OpenTelemetry client Design Diagram](../internal/img/library-design.png)

### Expected Usage  预期用途

The OpenTelemetry client is composed of 4 types of [packages](glossary.md#packages): API packages, SDK packages, a Semantic Conventions package, and plugin packages.
The API and the SDK are split into multiple packages, based on signal type (e.g. one for api-trace, one for api-metric, one for sdk-trace, one for sdk-metric) is considered an implementation detail as long as the API artifact(s) stay separate from the SDK artifact(s).

OpenTelemetry 客户端由 4 种类型的包组成：API 包、SDK 包、语义约定包和插件包。 API 和 SDK 被分成多个包，基于signal类型（例如一个用于 api-trace，一个用于 api-metric，一个用于 sdk-trace，一个用于 sdk-metric）被认为是一个实现细节，只要API 工件与 SDK 工件保持分离。

Libraries, frameworks, and applications that want to be instrumented with OpenTelemetry take a dependency only on the API packages. The developers of these third-party libraries will make calls to the API to produce telemetry data.

希望使用 OpenTelemetry 进行instrumented的库、框架和应用程序仅依赖于 API 包。这些第三方库的开发人员将调用 API 以生成遥测数据。

Applications that use third-party libraries that are instrumented with OpenTelemetry API control whether or not to install the SDK and generate telemetry data. When the SDK is not installed, the API calls should be no-ops which generate minimal overhead.

使用通过 OpenTelemetry API 检测的第三方库的应用程序控制是否安装 SDK 并生成遥测数据。未安装 SDK 时，API 调用应为无操作，从而产生最少的开销。

In order to enable telemetry the application must take a dependency on the OpenTelemetry SDK. The application must also configure exporters and other plugins so that telemetry can be correctly generated and delivered to their analysis tool(s) of choice. The details of how plugins are enabled and configured are language specific.

为了启用遥测，应用程序必须依赖于 OpenTelemetry SDK。应用程序还必须配置导出器和其他插件，以便可以正确生成遥测数据并将其传送到他们选择的分析工具。如何启用和配置插件的详细信息是特定于语言的。

### API and Minimal Implementation  API 和最小实现

The API package is a self-sufficient dependency, in the sense that if the end-user application or a third-party library depends only on it and does not plug a full SDK implementation then the application will still build and run without failing, although no telemetry data will be actually delivered to a telemetry backend.

API 包是一个自给自足的依赖项，从某种意义上说，如果最终用户应用程序或第三方库仅依赖于它并且没有插入完整的 SDK 实现，那么该应用程序仍将构建和运行而不会失败，尽管遥测数据实际上不会传送到遥测后端。

This self-sufficiency is achieved the following way.

这种自给自足是通过以下方式实现的。

The API dependency contains a minimal implementation of the API. When no other implementation is explicitly included in the application no telemetry data will be collected. Here is what active components look like in this case:

API 依赖项包含 API 的最小实现。当应用程序中没有明确包含其他实现时，将不会收集遥测数据。在这种情况下，活动组件的外观如下所示：

![Minimal Operation Diagram](../internal/img/library-minimal.png)

It is important that values returned from this minimal implementation of API are valid and do not require the caller to perform extra checks (e.g. createSpan() method should not fail and should return a valid non-null Span object). The caller should not need to know and worry about the fact that minimal implementation is in effect. This minimizes the boilerplate and error handling in the instrumented code.

重要的是，从 API 的这个最小实现返回的值是有效的，并且不需要调用者执行额外的检查（例如 createSpan() 方法不应该失败并且应该返回一个有效的非空non-null Span 对象）。调用者不需要知道和担心最小实现有效的事实。这最大限度地减少了检测代码中的样板和错误处理。

It is also important that minimal implementation incurs as little performance penalty as possible, so that third-party frameworks and libraries that are instrumented with OpenTelemetry impose negligible overheads to users of such libraries that do not want to use OpenTelemetry too.

同样重要的是，最少的实现会导致尽可能少的性能损失，因此使用 OpenTelemetry 检测的第三方框架和库对不想使用 OpenTelemetry 的此类库的用户施加的开销可以忽略不计。

### SDK Implementation

SDK implementation is a separate (optional) dependency. When it is plugged in it substitutes the minimal implementation that is included in the API package (exact substitution mechanism is language dependent).

SDK 实现是一个单独的（可选）依赖项。当它被插入时，它会替换 API 包中包含的最小实现（确切的替换机制取决于语言）。

SDK implements core functionality that is required for translating API calls into telemetry data that is ready for exporting. Here is how OpenTelemetry components look like when SDK is enabled:

SDK 实现了将 API 调用转换为可供导出的遥测数据所需的核心功能。以下是启用 SDK 时 OpenTelemetry 组件的外观：

![Full Operation Diagram](../internal/img/library-full.png)

SDK defines an [Exporter interface](trace/sdk.md#span-exporter). Protocol-specific exporters that are responsible for sending telemetry data to backends must implement this interface.

SDK 定义了一个 Exporter 接口。负责将遥测数据发送到后端的特定于协议的导出器必须实现此接口。

SDK also includes optional helper exporters that can be composed for additional functionality if needed.

SDK 还包括可选的辅助导出器，如果需要，可以组合这些辅助导出器以实现附加功能。


Library designers need to define the language-specific `Exporter` interface based on [this generic specification](trace/sdk.md#span-exporter).

库设计者需要根据这个通用规范定义特定于语言的 Exporter 接口。

#### Protocol Exporters  协议Exporters


Telemetry backend vendors are expected to implement [Exporter interface](trace/sdk.md#span-exporter). Data received via Export() function should be serialized and sent to the backend in a vendor-specific way.

遥测后端供应商有望实现 Exporter 接口。通过 Export() 函数接收的数据应该被序列化并以特定于供应商的方式发送到后端。

Vendors are encouraged to keep protocol-specific exporters as simple as possible and achieve desirable additional functionality such as queuing and retrying using helpers provided by SDK.

鼓励供应商使特定于协议的导出器尽可能简单，并使用 SDK 提供的帮助器实现所需的附加功能，例如排队和重试。

End users should be given the flexibility of making many of the decisions regarding the queuing, retrying, tagging, batching functionality that make the most sense for their application. For example, if an application's telemetry data must be delivered to a remote backend that has no guaranteed availability the end user may choose to use a persistent local queue and an `Exporter` to retry sending on failures. As opposed to that for an application that sends telemetry to a locally running Agent daemon, the end user may prefer to have a simpler exporting configuration without retrying or queueing.

最终用户应该能够灵活地做出对他们的应用程序最有意义的排队、重试、标记、批处理功能的许多决定。例如，如果必须将应用程序的遥测数据传送到无法保证可用性的远程后端，则最终用户可以选择使用持久本地队列和导出器在失败时重试发送。与将遥测发送到本地运行的代理守护程序的应用程序相反，最终用户可能更喜欢使用更简单的导出配置，而无需重试或排队。

If additional exporters for the sdk are provided as separate libraries, the
name of the library should be prefixed with the terms "OpenTelemetry" and "Exporter" in accordance with the naming conventions of the respective technology.

如果 sdk 的其他导出器作为单独的库提供，则库的名称应根据相应技术的命名约定以术语“OpenTelemetry”和“Exporter”作为前缀。

For example:

- Python and Java: opentelemetry-exporter-jaeger
- Javascript: @opentelemetry/exporter-jeager

#### Resource Detection  资源检测


Cloud vendors are encouraged to provide packages to detect resource information from the environment. These MUST be implemented outside of the SDK. See [Resource SDK](./resource/sdk.md#detecting-resource-information-from-the-environment) for more details.

鼓励云供应商提供包来检测环境中的资源信息。这些必须在 SDK 之外实现。有关更多详细信息，请参阅资源 SDK。

### Alternative Implementations  替代实现

The end-user application may decide to take a dependency on alternative implementation.

最终用户应用程序可以决定 依赖于 替代实现。

SDK provides flexibility and extensibility that may be used by many implementations. Before developing an alternative implementation, please, review extensibility points provided by OpenTelemetry.

SDK 提供了许多实现可以使用的灵活性和可扩展性。在开发 替代实现 之前，请查看 OpenTelemetry 提供的扩展点。


An example use-case for alternate implementations is automated testing. A mock implementation can be plugged in during automated tests. For example, it can store all generated telemetry data in memory and provide a capability to inspect this stored data. This will allow the tests to verify that the telemetry is generated correctly. OpenTelemetry client authors are encouraged to provide such a mock implementation.

替代实现的一个示例用例是自动化测试。可以在自动化测试期间插入模拟实现。例如，它可以将所有生成的遥测数据存储在内存中，并提供检查这些存储数据的能力。这将允许测试验证遥测数据是否正确生成。鼓励 OpenTelemetry 客户端作者提供这样的模拟实现。

Note that mocking is also possible by using SDK and a Mock `Exporter` without needing to swap out the entire SDK.

请注意，通过使用 SDK 和 Mock Exporter 也可以进行模拟，而无需更换整个 SDK。

The mocking approach chosen will depend on the testing goals and at which point exactly it is desirable to intercept the telemetry data path during the test.

选择的模拟方法将取决于测试目标以及在测试期间截取遥测数据路径的确切时间点。

### Version Labeling

API and SDK packages must use semantic version numbering. API package version number and SDK package version number are decoupled and can be different (and they both can be also different from the Specification version number that they implement). API and SDK packages MUST be labeled with their own version number.

API 和 SDK 包必须使用语义版本编号。 API 包版本号和 SDK 包版本号是解耦的，可以不同（它们也可以与它们实现的规范版本号不同）。 API 和 SDK 包必须标有自己的版本号。

This decoupling of version numbers allows OpenTelemetry client authors to make API and SDK package releases independently without the need to coordinate and match version numbers with the Specification.

版本号的这种解耦允许 OpenTelemetry 客户端作者独立发布 API 和 SDK 包，而无需协调和匹配版本号与规范。

Because API and SDK package version numbers are not coupled, every API and SDK package release MUST clearly mention the Specification version number that they implement. In addition, if a particular version of SDK package is only compatible with a specific version of API package, then this compatibility information must be also published by OpenTelemetry client authors. OpenTelemetry client authors MUST include this information in the release notes. For example, the SDK package release notes may say: "SDK 0.3.4, use with API 0.1.0, implements OpenTelemetry Specification 0.1.0".

由于 API 和 SDK 包版本号不耦合，因此每个 API 和 SDK 包版本都必须明确提及它们实现的规范版本号。此外，如果特定版本的 SDK 包仅兼容特定版本的 API 包，则此兼容性信息也必须由 OpenTelemetry 客户端作者发布。 OpenTelemetry 客户端作者必须在发行说明中包含此信息。例如，SDK 包发行说明可能会说：“SDK 0.3.4，与 API 0.1.0 一起使用，实现 OpenTelemetry 规范 0.1.0”。

_TODO: How should third-party library authors who use OpenTelemetry for instrumentation guide their end users to find the correct SDK package?_

TODO：使用 OpenTelemetry 进行检测的第三方库作者应该如何指导他们的最终用户找到正确的 SDK 包？

### Performance and Blocking

See the [Performance and Blocking](performance.md) specification for
guidelines on the performance expectations that API implementations should meet, strategies for meeting these expectations, and a description of how implementations should document their behavior under load.

有关 API 实现应满足的性能期望、满足这些期望的策略以及实现应如何记录其在负载下的行为的描述，请参阅性能和阻塞规范。

### Concurrency and Thread-Safety  并发和线程安全


Please refer to individual API specification for guidelines on what concurrency
safeties should API implementations provide and how they should be documented:

有关 API 实现应提供哪些并发安全性以及应如何记录这些安全性的指南，请参阅各个 API 规范：


* [Metrics API](./metrics/api.md#concurrency-requirements)
* [Metrics SDK](./metrics/sdk.md#concurrency-requirements)
* [Tracing API](./trace/api.md#concurrency)
