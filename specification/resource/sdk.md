# Resource SDK

**Status**: [Stable](../document-status.md)

A [Resource](../overview.md#resources) is an immutable representation of the entity producing
telemetry as [Attributes](../common/common.md#attributes).
For example, a process producing telemetry that is running in a
container on Kubernetes has a Pod name, it is in a namespace and possibly is
part of a Deployment which also has a name. All three of these attributes can be
included in the `Resource`. Note that there are certain
["standard attributes"](semantic_conventions/README.md) that have prescribed meanings.

资源是将遥测作为属性生成的实体的不可变表示。例如，在 Kubernetes 上的容器中运行的产生遥测的进程有一个 Pod 名称，它位于命名空间中，并且可能是也有名称的 Deployment 的一部分。所有这三个属性都可以包含在资源中。请注意，某些“标准属性”具有规定的含义。

The primary purpose of resources as a first-class concept in the SDK is
decoupling of discovery of resource information from exporters. This allows for
independent development and easy customization for users that need to integrate
with closed source environments. The SDK MUST allow for creation of `Resources` and
for associating them with telemetry.

资源作为 SDK 中的 一流概念 的主要目的是将资源信息的发现与导出器解耦。这允许需要与闭源环境集成的用户进行独立开发和轻松定制。 SDK 必须允许创建 `资源` 并将它们与遥测相关联。

When used with distributed tracing, a resource can be associated with the
[TracerProvider](../trace/api.md#tracerprovider) when the TracerProvider is created.
That association cannot be changed later.
When associated with a `TracerProvider`,
all `Span`s produced by any `Tracer` from the provider MUST be associated with this `Resource`.

当与分布式tracing一起使用时，可以在创建 TracerProvider 时将资源与 TracerProvider 相关联。以后无法更改该关联。当与 TracerProvider 相关联时，来自提供者的任何 Tracer 产生的所有 Span 必须与此资源相关联。

Analogous to distributed tracing, when used with metrics,
a resource can be associated with a `MeterProvider`.
When associated with a [`MeterProvider`](../metrics/api.md#meterprovider),
all metrics produced by any `Meter` from the provider will be
associated with this `Resource`.

类似于分布式tracing，当与指标metrics一起使用时，资源可以与 MeterProvider 相关联。当与 MeterProvider 相关联时，提供者的任何 Meter 生成的所有指标都将与此资源相关联。

## SDK-provided resource attributes

The SDK MUST provide access to a Resource with at least the attributes listed at
[Semantic Attributes with SDK-provided Default Value](semantic_conventions/README.md#semantic-attributes-with-sdk-provided-default-value).
This resource MUST be associated with a `TracerProvider` or `MeterProvider`
if another resource was not explicitly specified.

SDK 必须至少提供对具有 SDK 提供的默认值的语义属性中列出的属性的资源的访问。如果未明确指定另一个资源，则该资源必须与 TracerProvider 或 MeterProvider 相关联。

Note: This means that it is possible to create and associate a resource that
does not have all or any of the SDK-provided attributes present. However, that
does not happen by default. If a user wants to combine custom attributes with
the default resource, they can use [`Merge`](#merge) with their custom resource
or specify their attributes by implementing
[Custom resource detectors](#detecting-resource-information-from-the-environment)
instead of explicitly associating a resource.

注意：这意味着可以创建和关联不存在所有或任何 SDK 提供的属性的资源。但是，默认情况下不会发生这种情况。如果用户想要将自定义属性与默认资源组合，他们可以将 Merge 与他们的自定义资源一起使用，或者通过实现自定义资源检测器而不是显式关联资源来指定他们的属性。

## Resource creation

The SDK must support two ways to instantiate new resources. Those are:

SDK 必须支持两种方式来实例化新资源。那些是：

### Create

The interface MUST provide a way to create a new resource, from [`Attributes`](../common/common.md#attributes).
Examples include a factory method or a constructor for a resource
object. A factory method is recommended to enable support for cached objects.

接口必须提供一种从Attributes创建新资源的方法。示例包括资源对象的工厂方法或构造函数。建议使用工厂方法来启用对缓存对象的支持。

Required parameters:

- [`Attributes`](../common/common.md#attributes)
- [since 1.4.0] `schema_url` (optional): Specifies the Schema URL that should be
  recorded in the emitted resource. If the `schema_url` parameter is unspecified
  then the created resource will have an empty Schema URL.  指定应记录在发出的资源中的Schema URL。如果未指定 schema_url 参数，则创建的资源将具有空Schema URL。

### Merge

The interface MUST provide a way for an old resource and an
updating resource to be merged into a new resource.

该接口必须提供一种将旧资源和更新资源合并到新资源中的方法。

Note: This is intended to be utilized for merging of resources whose attributes
come from different sources,
such as environment variables, or metadata extracted from the host or container.

注意：这旨在用于合并其属性来自不同来源的资源，例如环境变量，或从主机或容器中提取的元数据。

The resulting resource MUST have all attributes that are on any of the two input resources.
If a key exists on both the old and updating resource, the value of the updating
resource MUST be picked (even if the updated value is empty).

结果资源必须具有两个输入资源中任何一个的所有属性。如果旧资源和更新资源上都存在一个键，则必须选择更新资源的值（即使更新后的值为空）。

The resulting resource will have the Schema URL calculated as follows:

生成的资源将具有如下计算的Schema URL：

- If the old resource's Schema URL is empty then the resulting resource's Schema
  URL will be set to the Schema URL of the updating resource,
- Else if the updating resource's Schema URL is empty then the resulting
  resource's Schema URL will be set to the Schema URL of the old resource,
- Else if the Schema URLs of the old and updating resources are the same then
  that will be the Schema URL of the resulting resource,
- Else this is a merging error (this is the case when the Schema URL of the old
  and updating resources are not empty and are different). The resulting resource is
  undefined, and its contents are implementation-specific.

Required parameters:

- the old resource
- the updating resource whose attributes take precedence

### The empty resource

It is recommended, but not required, to provide a way to quickly create an empty
resource.

### Detecting resource information from the environment

Custom resource detectors related to generic platforms (e.g. Docker, Kubernetes)
or vendor specific environments (e.g. EKS, AKS, GKE) MUST be implemented as
packages separate from the SDK.

Resource detector packages MUST provide a method that returns a resource. This
can then be associated with `TracerProvider` or `MeterProvider` instances as
described above.

资源检测器包必须提供返回资源的方法。然后可以将其与 TracerProvider 或 MeterProvider 实例相关联，如上所述。

Resource detector packages MAY detect resource information from multiple
possible sources and merge the result using the `Merge` operation described
above.

资源检测器包可以检测来自多个可能来源的资源信息，并使用上述合并操作合并结果。

Resource detection logic is expected to complete quickly since this code will be
run during application initialization. Errors should be handled as specified in
the [Error Handling
principles](../error-handling.md#basic-error-handling-principles). Note the
failure to detect any resource information MUST NOT be considered an error,
whereas an error that occurs during an attempt to detect resource information
SHOULD be considered an error.

资源检测逻辑有望快速完成，因为此代码将在应用程序初始化期间运行。应按照错误处理原则中的规定处理错误。请注意，检测到任何资源信息的失败绝不能被视为错误，而在尝试检测资源信息期间发生的错误应该被视为错误。

Resource detectors that populate resource attributes according to OpenTelemetry
semantic conventions MUST ensure that the resource has a Schema URL set to a
value that matches the semantic conventions. Empty Schema URL SHOULD be used if
the detector does not populate the resource with any known attributes that have
a semantic convention or if the detector does not know what attributes it will
populate (e.g. the detector that reads the attributes from environment values
will not know what Schema URL to use). If multiple detectors are combined and
the detectors use different non-empty Schema URL it MUST be an error since it is
impossible to merge such resources. The resulting resource is undefined, and its
contents are implementation specific.

根据 OpenTelemetry 语义约定填充资源属性的资源检测器必须确保资源的 Schema URL 设置为与语义约定匹配的值。如果检测器没有使用任何具有语义约定的已知属性填充资源，或者如果检测器不知道它将填充哪些属性（例如，从环境值读取属性的检测器将不知道什么），则应使用空模式 URL要使用的架构 URL）。如果多个检测器组合在一起，并且检测器使用不同的非空模式 URL，那么它一定是一个错误，因为不可能合并这些资源。结果资源是未定义的，其内容是特定于实现的。

### Specifying resource information via an environment variable

The SDK MUST extract information from the `OTEL_RESOURCE_ATTRIBUTES` environment
variable and [merge](#merge) this, as the secondary resource, with any resource
information provided by the user, i.e. the user provided resource information
has higher priority.

SDK 必须从 OTEL_RESOURCE_ATTRIBUTES 环境变量中提取信息，并将其作为次要资源与用户提供的任何资源信息合并，即用户提供的资源信息具有更高的优先级。

The `OTEL_RESOURCE_ATTRIBUTES` environment variable will contain of a list of
key value pairs, and these are expected to be represented in a format matching
to the [W3C Baggage](https://github.com/w3c/baggage/blob/fdc7a5c4f4a31ba2a36717541055e551c2b032e4/baggage/HTTP_HEADER_FORMAT.md#header-content),
except that additional semi-colon delimited metadata is not supported, i.e.:
`key1=value1,key2=value2`. All attribute values MUST be considered strings.

OTEL_RESOURCE_ATTRIBUTES 环境变量将包含键值对列表，这些键值对应该以与 W3C Baggage 匹配的格式表示，除了不支持额外的分号分隔的元数据，即：key1=value1,key2=值2。所有属性值必须被视为字符串。

## Resource operations

Resources are immutable. Thus, in addition to resource creation,
only the following operations should be provided:

资源是不可变的。因此，除了资源创建之外，只应提供以下操作：

### Retrieve attributes

The SDK should provide a way to retrieve a read only collection of attributes
associated with a resource. SDK 应该提供一种方法来检索与资源关联的只读属性集合。

There is no need to guarantee the order of the attributes. 不需要保证属性的顺序。

The most common operation when retrieving attributes is to enumerate over them. As
such, it is recommended to optimize the resulting collection for fast
enumeration over other considerations such as a way to quickly retrieve a value
for a attribute with a specific key.

检索属性时最常见的操作是枚举它们。因此，建议优化生成的集合以进行快速枚举，而不是其他考虑因素，例如使用特定键快速检索属性值的方法。

