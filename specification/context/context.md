# Context

**Status**: [Stable, Feature-freeze](../document-status.md).

<details>
<summary>
Table of Contents
</summary>

- [Overview](#overview)
- [Create a key](#create-a-key)
- [Get value](#get-value)
- [Set value](#set-value)
- [Optional global operations](#optional-global-operations)
  - [Get current Context](#get-current-context)
  - [Attach Context](#attach-context)
  - [Detach Context](#detach-context)

</details>

## Overview

A `Context` is a propagation mechanism which carries execution-scoped values
across API boundaries and between logically associated execution units.
Cross-cutting concerns access their data in-process using the same shared
`Context` object.

`上下文`是一种传播机制，它在 API 边界和逻辑关联的执行单元之间承载执行范围的值。横切关注点使用相同的共享 上下文对象`Context` 在进程中访问它们的数据。


A `Context` MUST be immutable, and its write operations MUST
result in the creation of a new `Context` containing the original
values and the specified values updated.

上下文必须是不可变的，并且它的写操作必须导致创建一个包含原始值和更新的指定值的新上下文。


Languages are expected to use the single, widely used `Context` implementation
if one exists for them. In the cases where an extremely clear, pre-existing
option is not available, OpenTelemetry MUST provide its own `Context`
implementation. Depending on the language, its usage may be either explicit
or implicit.

语言应该使用单一的、广泛使用的 Context 实现（如果存在的话）。在非常明确的、预先存在的选项不可用的情况下，OpenTelemetry 必须提供自己的上下文实现。根据语言的不同，它的用法可能是显式的，也可能是隐式的。

Users writing instrumentation in languages that use `Context` implicitly are
discouraged from using the `Context` API directly. In those cases, users will
manipulate `Context` through cross-cutting concerns APIs instead, in order to
perform operations such as setting trace or baggage entries for a specified
`Context`.

不鼓励使用隐式使用 Context 的语言编写instrumentation的用户直接使用 Context API。在这些情况下，用户将通过横切关注点 API 来操作 Context，以便为指定的 Context 执行诸如设置trace or baggage entrie之类的操作。

A `Context` is expected to have the following operations, with their
respective language differences:

虽然它们各自的语言差异，一个 Context 应该有以下操作，：

## Create a key

Keys are used to allow cross-cutting concerns to control access to their local state.
They are unique such that other libraries which may use the same context
cannot accidentally use the same key. It is recommended that concerns mediate
data access via an API, rather than provide direct public access to their keys.

key用于允许横切关注点控制对其本地状态的访问。它们是独一无二的，因此可能使用相同上下文的其他库不会意外使用相同的key。建议关注通过 API 调解数据访问，而不是提供对其key的直接公共访问。

The API MUST accept the following parameter:
API 必须接受以下参数：


- The key name. The key name exists for debugging purposes and does not uniquely identify the key. Multiple calls to `CreateKey` with the same name SHOULD NOT return the same value unless language constraints dictate otherwise. Different languages may impose different restrictions on the expected types, so this parameter remains an implementation detail.

key名。key名称用于调试目的，并不唯一标识密钥。除非语言限制另有规定，否则多次调用具有相同名称的 CreateKey 不应返回相同的值。不同的语言可能对预期的类型施加不同的限制，因此该参数仍然是一个实现细节。

The API MUST return an opaque object representing the newly created key.

API 必须返回一个表示新创建的key的不透明对象。


## Get value

Concerns can access their local state in the current execution state
represented by a `Context`.

关注点可以在由上下文表示的当前执行状态中访问它们的本地状态。

The API MUST accept the following parameters:

API 必须接受以下参数：


- The `Context`.
- The key.

The API MUST return the value in the `Context` for the specified key.

## Set value

Concerns can record their local state in the current execution state
represented by a `Context`.

The API MUST accept the following parameters:

- The `Context`.
- The key.
- The value to be set.

The API MUST return a new `Context` containing the new value.

API 必须返回一个包含新值的新上下文。


## Optional Global operations  可选的全局操作


These operations are expected to only be implemented by languages
using `Context` implicitly, and thus are optional. These operations
SHOULD only be used to implement automatic scope switching and define
higher level APIs by SDK components and OpenTelemetry instrumentation libraries.

这些操作预计只能由隐式使用 Context 的语言实现，因此是可选的。这些操作应该仅用于实现自动范围切换并通过 SDK 组件和 OpenTelemetry 工具库定义更高级别的 API。

### Get current Context

The API MUST return the `Context` associated with the caller's current execution unit.

API 必须返回与调用者当前执行单元关联的上下文。

### Attach Context

Associates a `Context` with the caller's current execution unit.

将 Context 与调用者的当前执行单元相关联。


The API MUST accept the following parameters:API 必须接受以下参数：


- The `Context`.

The API MUST return a value that can be used as a `Token` to restore the previous
`Context`.

API 必须返回一个值，该值可用作令牌以恢复先前的上下文。

Note that every call to this operation should result in a corresponding call to
[Detach Context](#detach-context).

请注意，对该操作的每次调用都应导致对 Detach Context 的相应调用。

### Detach Context  分离上下文


Resets the `Context` associated with the caller's current execution unit
to the value it had before attaching a specified `Context`.

将与调用者当前执行单元关联的 Context 重置为它在附加指定 Context 之前的值。


This operation is intended to help making sure the correct `Context`
is associated with the caller's current execution unit. Users can
rely on it to identify a wrong call order, i.e. trying to detach
a `Context` that is not the current instance. In this case the operation
can emit a signal to warn users of the wrong call order, such as logging
an error or returning an error value.

此操作旨在帮助确保正确的 Context 与调用者的当前执行单元相关联。用户可以依靠它来识别错误的调用顺序，即尝试分离不是当前实例的上下文。在这种情况下，操作可以发出信号警告用户错误的调用顺序，例如记录错误或返回错误值。

The API MUST accept the following parameters:

- A `Token` that was returned by a previous call to attach a `Context`.先前调用附加上下文时返回的令牌。

The API MAY return a value used to check whether the operation
was successful or not.

API 可以返回一个用于检查操作是否成功的值。

