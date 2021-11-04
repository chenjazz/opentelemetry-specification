# Baggage API

**Status**: [Stable, Feature-freeze](../document-status.md)

<details>
<summary>
Table of Contents
</summary>

- [Overview](#overview)
- [Operations](#operations)
  - [Get Value](#get-value)
  - [Get All Values](#get-all-values)
  - [Set Value](#set-value)
  - [Remove Value](#remove-value)
- [Context Interaction](#context-interaction)
  - [Clear Baggage in the Context](#clear-baggage-in-the-context)
- [Propagation](#propagation)
- [Conflict Resolution](#conflict-resolution)

</details>

## Overview

`Baggage` is used to annotate telemetry, adding context and information to
metrics, traces, and logs. It is a set of name/value pairs describing
user-defined properties. Each name in `Baggage` MUST be associated with
exactly one value.

Baggage 用于注释遥测，将上下文和信息添加到指标、跟踪和日志中。它是一组描述用户定义属性的名称/值对。 Baggage 中的每个名称必须与一个值相关联。

The Baggage API consists of:

- the `Baggage`
- functions to interact with the `Baggage` in a `Context`  Baggage与Context交互的函数

The functions described here are one way to approach interacting with the
`Baggage` via having struct/object that represents the entire Baggage content.
Depending on language idioms, a language API MAY implement these functions by
interacting with the baggage via the `Context` directly.

此处描述的函数是通过具有代表整个Baggage内容的结构/对象来与Baggage交互的一种方式。根据语言习语，语言 API 可以通过直接通过上下文与Baggage交互来实现这些功能。

The Baggage API MUST be fully functional in the absence of an installed SDK.
This is required in order to enable transparent cross-process Baggage
propagation. If a Baggage propagator is installed into the API, it will work
with or without an installed SDK.

在没有安装 SDK 的情况下，Baggage API 必须是完整的。这是启用透明跨进程Baggage传播所必需的。如果将Baggage传播器安装到 API 中，无论是否安装了 SDK，它都可以工作。

The `Baggage` container MUST be immutable, so that the containing `Context`
also remains immutable.

Baggage 容器必须是不可变的，这样包含的 Context 也保持不可变。

## Operations

### Get Value

To access the value for a name/value pair set by a prior event, the Baggage API
MUST provide a function that takes the name as input, and returns a value
associated with the given name, or null if the given name is not present.

要访问由先前事件设置的名称/值对的值，Baggage API 必须提供一个函数，该函数将名称作为输入，并返回与给定名称关联的值，如果给定名称不存在，则返回 null。

REQUIRED parameters:

`Name` the name to return the value for.

### Get All Values

Returns the name/value pairs in the `Baggage`. The order of name/value pairs
MUST NOT be significant. Based on the language specifics, the returned
value can be either an immutable collection or an iterator on the immutable
collection of name/value pairs in the `Baggage`.

返回Baggage中的名称/值对。名称/值对的顺序不得重要。根据语言细节，返回值可以是不可变集合，也可以是 Baggage 中名称/值对不可变集合上的迭代器。

### Set Value

To record the value for a name/value pair, the Baggage API MUST provide a
function which takes a name, and a value as input. Returns a new `Baggage`
that contains the new value. Depending on language idioms, a language API MAY
implement these functions by using a `Builder` pattern and exposing a way to
construct a `Builder` from a `Baggage`.

为了记录名称/值对的值，Baggage API 必须提供一个函数，该函数接受一个名称和一个值作为输入。返回一个包含新值的新Baggage。根据语言习语，语言 API 可以通过使用构建器模式并公开一种从Baggage构造构建器的方法来实现这些功能。

REQUIRED parameters:

`Name` The name for which to set the value, of type string.

`Value` The value to set, of type string.

OPTIONAL parameters:

`Metadata` Optional metadata associated with the name-value pair. This should be
an opaque wrapper for a string with no semantic meaning. Left opaque to allow
for future functionality.

### Remove Value

To delete a name/value pair, the Baggage API MUST provide a function which
takes a name as input. Returns a new `Baggage` which no longer contains the
selected name. Depending on language idioms, a language API MAY
implement these functions by using a `Builder` pattern and exposing a way to
construct a `Builder` from a `Baggage`.

REQUIRED parameters:

`Name` the name to remove.

## Context Interaction  上下文交互


This section defines all operations within the Baggage API that interact with
the [`Context`](../context/context.md).

本节定义了Baggage API 中与上下文交互的所有操作。

If an implementation of this API does not operate directly on the `Context`, it
MUST provide the following functionality to interact with a `Context` instance:

- Extract the `Baggage` from a `Context` instance
- Insert the `Baggage` to a `Context` instance

The functionality listed above is necessary because API users SHOULD NOT have
access to the [Context Key](../context/context.md#create-a-key) used by the
Baggage API implementation.

上面列出的功能是必要的，因为 API 用户不应该访问Baggage API 实现使用的上下文key。

If the language has support for implicitly propagated `Context` (see
[here](../context/context.md#optional-global-operations)), the API SHOULD also
provide the following functionality:

如果该语言支持隐式传播的上下文（参见此处），则 API 还应提供以下功能：


- Get the currently active `Baggage` from the implicit context. This is
equivalent to getting the implicit context, then extracting the `Baggage` from
the context.
从隐式上下文中获取当前活动的 Baggage。这相当于获取隐式上下文，然后从上下文中提取 Baggage。
- Set the currently active `Baggage` to the implicit context. This is equivalent
to getting the implicit context, then inserting the `Baggage` to the context.
将当前活动的 Baggage 设置为隐式上下文。这相当于获取了隐式上下文，然后将 Baggage 插入到上下文中。

All the above functionalities operate solely on the context API, and they MAY be
exposed as static methods on the baggage module, as static methods on a class
inside the baggage module (it MAY be named `BaggageUtilities`), or on the
`Baggage` class. This functionality SHOULD be fully implemented in the API when
possible.

所有上述功能都只在上下文 API 上运行，它们可以作为 Baggage 模块上的静态方法、作为 Baggage 模块内部类（它可以命名为 BaggageUtilities）或 Baggage 类的静态方法公开。如果可能，此功能应在 API 中完全实现。

### Clear Baggage in the Context

To avoid sending any name/value pairs to an untrusted process, the Baggage API
MUST provide a way to remove all baggage entries from a context.

This functionality can be implemented by having the user set an empty `Baggage`
object/struct into the context, or by providing an API that takes a `Context` as
input, and returns a new `Context` with no `Baggage` associated.

## Propagation  传播


`Baggage` MAY be propagated across process boundaries or across any arbitrary
boundaries (process, $OTHER_BOUNDARY1, $OTHER_BOUNDARY2, etc) for various
reasons.  

由于各种原因，Baggage可能会跨进程边界或任意边界（进程、$OTHER_BOUNDARY1、$OTHER_BOUNDARY2 等）传播。

The API layer or an extension package MUST include the following `Propagator`s:

API 层或扩展包必须包含以下传播器：

* A `TextMapPropagator` implementing the [W3C Baggage Specification](https://w3c.github.io/baggage).

See [Propagators Distribution](../context/api-propagators.md#propagators-distribution)
for how propagators are to be distributed.

Note: The W3C baggage specification does not currently assign semantic meaning
to the optional metadata.

On `extract`, the propagator should store all metadata as a single metadata instance per entry.
On `inject`, the propagator should append the metadata per the W3C specification format.
Refer to the API Propagators
[Operation](../context/api-propagators.md#operations) section for the
additional requirements these operations need to follow.

在提取时，传播者应将所有元数据存储为每个条目的单个元数据实例。在注入时，传播者应该按照 W3C 规范格式附加元数据。有关这些操作需要遵循的其他要求，请参阅 API 传播器操作部分。

## Conflict Resolution  解决冲突


If a new name/value pair is added and its name is the same as an existing name,
than the new pair MUST take precedence. The value is replaced with the added
value (regardless if it is locally generated or received from a remote peer).

如果添加了新的名称/值对并且其名称与现有名称相同，则新对必须优先。该值将替换为附加值（无论它是本地生成的还是从远程对等方接收的）
