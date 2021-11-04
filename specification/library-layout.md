# OpenTelemetry Project Package Layout

This documentation serves to document the "look and feel" of a basic layout for OpenTelemetry
projects. This package layout is intentionally generic and it doesn't try to impose a language
specific package structure.

本文档用于记录 OpenTelemetry 项目基本布局的“外观和感觉”。这个包布局是故意设计为通用的，它不会试图强加特定于语言的包结构。

## API Package

Here is a proposed generic package structure for OpenTelemetry API package.

这是 OpenTelemetry API 包的建议通用包结构。

A typical top-level directory layout:

一个典型的顶级目录布局：


```
api
   ├── context
   │   └── propagation
   ├── metrics
   ├── trace
   │   └── propagation
   ├── baggage
   │   └── propagation
   ├── internal
   └── logs
```

> Use of lowercase, CamelCase or Snake Case (stylized as snake_case) names depends on the language.

### `/context`

This directory describes the API that provides in-process context propagation.

该目录描述了提供进程内上下文传播的 API。

### [/metrics](./metrics/api.md)

This directory describes the Metrics API that can be used to record application metrics.

此目录描述可用于记录应用程序指标的 Metrics API。

### [/baggage](baggage/api.md)

This directory describes the Baggage API that can be used to manage context propagation
and metric event attributes.

此目录描述了可用于管理上下文传播和指标事件属性的 Baggage API。

### [/trace](trace/api.md)

This API consist of a few main classes:  该 API 由几个主要类组成：

- `Tracer` is used for all operations. See [Tracer](trace/api.md#tracer) section.Tracer 用于所有操作。见示踪部分。
- `Span` is a mutable object storing information about the current operation
   execution. See [Span](trace/api.md#span) section. Span 是一个可变对象，用于存储有关当前操作执行的信息。请参span部分。

### `/internal` (_Optional_)

Library components and implementations that shouldn't be exposed to the users.
If a language has an idiomatic layout for internal components, please follow
the language idiomatic style.

不应向用户公开的库组件和实现。如果语言有内部组件的惯用布局，请遵循语言惯用风格。


### `/logs` (_In the future_)

> TODO: logs operations

## SDK Package

Here is a proposed generic package structure for OpenTelemetry SDK package.

这是 OpenTelemetry SDK 包的建议通用包结构。

A typical top-level directory layout:

```
sdk
   ├── context
   ├── metrics
   ├── resource
   ├── trace
   ├── baggage
   ├── internal
   └── logs
```

> Use of lowercase, CamelCase or Snake Case (stylized as snake_case) names depends on the language.

### `/sdk/context`

This directory describes the SDK implementation for api/context.

该目录描述了 api/context 的 SDK 实现。

### `/sdk/metrics`

This directory describes the SDK implementation for api/metrics.

该目录描述了 api/metrics 的 SDK 实现。

### [/sdk/resource](resource/sdk.md)

The resource directory primarily defines a type [Resource](overview.md#resources) that captures
information about the entity for which stats or traces are recorded. For example, metrics exposed
by a Kubernetes container can be linked to a resource that specifies the cluster, namespace, pod,
and container name.

### `/sdk/baggage`

### [/sdk/trace](trace/sdk.md)

This directory describes the SDK implementation for api/trace.

该目录描述了 api/trace 的 SDK 实现。

### `/sdk/internal` (_Optional_)

Library components and implementations that shouldn't be exposed to the users.
If a language has an idiomatic layout for internal components, please follow
the language idiomatic style.

不应向用户公开的库组件和实现。如果语言有内部组件的惯用布局，请遵循语言惯用风格。


### `/sdk/logs` (_In the future_)

> TODO: logs operations
