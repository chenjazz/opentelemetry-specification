# Versioning and stability for OpenTelemetry clients

**Status**: [Stable](document-status.md)

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [Design goals](#design-goals)
- [Signal lifecycle](#signal-lifecycle)
  * [Experimental](#experimental)
  * [Stable](#stable)
    + [API Stability](#api-stability)
      - [Extending Existing API Calls](#extending-existing-api-calls)
    + [SDK Stability](#sdk-stability)
    + [Contrib Stability](#contrib-stability)
    + [NOT DEFINED: Telemetry Stability](#not-defined-telemetry-stability)
    + [NOT DEFINED: Semantic Conventions Stability](#not-defined-semantic-conventions-stability)
  * [Deprecated](#deprecated)
  * [Removed](#removed)
  * [A note on replacing signals](#a-note-on-replacing-signals)
- [Version numbers](#version-numbers)
  * [Major version bump](#major-version-bump)
  * [Minor version bump](#minor-version-bump)
  * [Patch version bump](#patch-version-bump)
- [Long Term Support](#long-term-support)
  * [API support](#api-support)
  * [SDK Support](#sdk-support)
  * [Contrib Support](#contrib-support)
- [OpenTelemetry GA](#opentelemetry-ga)

<!-- tocstop -->

This document defines the stability guarantees offered by the OpenTelemetry clients, along with the rules and procedures for meeting those guarantees.

本文档定义了 OpenTelemetry 客户端提供的稳定性保证，以及满足这些保证的规则和程序。

In this document, the terms "OpenTelemetry" and "language implementations" both specifically refer to the OpenTelemetry clients.
These terms do not refer to the specification or the Collector in this document.

在本文档中，术语“OpenTelemetry”和“语言实现”都特指 OpenTelemetry 客户端。这些术语不涉及本文档中的规范或Collector。

Each language implementation MUST take these versioning and stability requirements, and produce a language-specific document which details how these requirements will be met.
This document SHALL be placed in the root of each repo and named `VERSIONING`.

每种语言实现都必须采用这些版本控制和稳定性要求，并生成特定于语言的文档，详细说明如何满足这些要求。该文档应放置在每个 repo 的根目录中，并命名为 VERSIONING。

## Design goals  设计目标


Versioning and stability procedures are designed to meet the following goals.

版本控制和稳定性程序旨在满足以下目标。

**Ensure that application owners stay up to date with the latest release of the SDK.**
We want all users to stay up to date with the latest version of the OpenTelemetry SDK.
We do not want to create hard breaks in support, of any kind, which leave users stranded on older versions.
It MUST always be possible to upgrade to the latest minor version of the OpenTelemetry SDK, without creating compilation or runtime errors.

**确保应用程序所有者与 SDK 的最新版本保持同步。** 我们希望所有用户都了解最新版本的 OpenTelemetry SDK。我们不希望在支持方面造成任何形式的硬中断，从而使用户陷入旧版本的困境。必须始终可以升级到 OpenTelemetry SDK 的最新次要版本，而不会产生编译或运行时错误。

**Never create a dependency conflict between packages which rely on different versions of OpenTelemetry. Avoid breaking all stable public APIs.**
Backwards compatibility is a strict requirement.
Instrumentation APIs cannot create a version conflict, ever. Otherwise, the OpenTelemetry API cannot be embedded in widely shared libraries, such as web frameworks.
Code written against older versions of the API MUST work with all newer versions of the API.
Transitive dependencies of the API cannot create a version conflict. The OpenTelemetry API cannot depend on a particular package if there is any chance that any library or application may require a different, incompatible version of that package.
A library that imports the OpenTelemetry API should never become incompatible with other libraries due to a version conflict in one of OpenTelemetry's dependencies.
Theoretically, APIs can be deprecated and eventually removed, but this is a process measured in years and we have no plans to do so.

永远不要在依赖于不同版本的 OpenTelemetry 的包之间创建依赖冲突。  避免破坏所有稳定的公共 API。向后兼容性是一项严格的要求。Instrumentation API 永远不会产生版本冲突。否则，OpenTelemetry API 无法嵌入到广泛共享的库中，例如 Web 框架。针对旧版本 API 编写的代码必须适用于所有较新版本的 API。 API 的传递依赖项不会造成版本冲突。如果任何库或应用程序可能需要该包的不同的、不兼容的版本，则 OpenTelemetry API 不能依赖于特定的包。由于 OpenTelemetry 依赖项之一的版本冲突，导入 OpenTelemetry API 的库永远不应该与其他库不兼容。理论上，API 可以被弃用并最终被删除，但这是一个以年为单位的过程，我们没有这样做的计划。

**Allow for multiple levels of package stability within the same release of an OpenTelemetry component.**
Provide maintainers a clear process for developing new, experimental [signals](glossary.md#signals) alongside stable signals.
Different packages within the same release may have different levels of stability.
This means that an implementation wishing to release stable tracing today MUST ensure that experimental metrics are factored out in such a way that breaking changes to metrics API do not destabilize the trace API packages.

允许在 OpenTelemetry 组件的同一版本中实现多个级别的包稳定性。为维护者提供一个清晰的过程，以便在稳定signal的同时开发新的实验性signal。同一发行版中的不同软件包可能具有不同的稳定性级别。这意味着希望今天发布稳定跟踪的实现必须确保以这样一种方式排除实验性指标，即metrics API 的重大更改不会破坏 trace API 包的稳定性。

## Signal lifecycle 生命周期

The development of each signal follows a lifecycle: experimental, stable, deprecated, removed.

每个signal的发展遵循一个生命周期  experimental, stable, deprecated, removed.


The infographic below shows an example of the lifecycle of an API component.

下面的信息图显示了 API 组件的生命周期示例。

![API Lifecycle](../internal/img/api-lifecycle.png)

### Experimental

Signals start as **experimental**, which covers alpha, beta, and release candidate versions of the signal.
While signals are experimental, breaking changes and performance issues MAY occur.
Components SHOULD NOT be expected to be feature-complete.
In some cases, the experiment MAY be discarded and removed entirely.
Long-term dependencies SHOULD NOT be taken against experimental signals.

OpenTelemetry clients MUST be designed in a manner that allows experimental signals to be created without breaking the stability guarantees of existing signals.

OpenTelemetry clients MUST NOT be designed in a manner that breaks existing users when a signal transitions from experimental to stable. This would punish users of the release candidate, and hinder adoption.

Terms which denote stability, such as "experimental," MUST NOT be used as part of a directory or import name.
Package **version numbers** MAY include a suffix, such as -alpha, -beta, -rc, or -experimental, to differentiate stable and experimental packages.

### Stable

Once an experimental signal has gone through rigorous beta testing, it MAY transition to **stable**.
Long-term dependencies MAY now be taken against this signal.

All signal components MAY become stable together, or MAY transition to stability component-by-component. The API MUST become stable before the other components.

Once a signal component is marked as stable, the following rules MUST apply until the end of that signal’s existence.

#### API Stability

Backward-incompatible changes to API packages MUST NOT be made unless the major version number is incremented.
All existing API calls MUST continue to compile and function against all future minor versions of the same major version.

Languages which ship binary artifacts SHOULD offer [ABI compatibility](glossary.md#abi-compatibility) for API packages.

##### Extending Existing API Calls

An existing API call MAY be extended without incrementing the major version
number if the particular language allows to do it in a backward-compatible
manner.

To add a new parameter to an existing API call depending on the language several
approaches are possible:

- Add a new optional parameter to existing methods. This may not be the right
  approach for languages where ABI stability is part of our guarantees since it
  likely breaks the ABI.

- Add a method overload that allows passing a different set of parameters, that
  include the new parameter. This is likely the preferred approach for languages
  where method overloads are possible.

There may be other ways to extend existing APIs in non-breaking manner. Language
maintainers SHOULD choose the idiomatic way for their language.

#### SDK Stability

Public portions of SDK packages MUST remain backwards compatible.
There are two categories of public features: **plugin interfaces** and **constructors**.
Examples of plugins include the SpanProcessor, Exporter, and Sampler interfaces.
Examples of constructors include configuration objects, environment variables, and SDK builders.

Languages which ship binary artifacts SHOULD offer [ABI compatibility](glossary.md#abi-compatibility) for SDK packages.

#### Contrib Stability

**NOTE: Until telemetry stability is defined, Contrib instrumentation MUST NOT be marked as stable. See below.**

Plugins, instrumentation, and other contrib packages SHOULD be kept up to date and compatible with the latest versions of the API, SDK, and Semantic Conventions.
If a release of the API, SDK, or Semantic Conventions contains changes which are relevant to a contrib package, that package SHOULD be updated and released in a timely fashion.
The goal is to ensure users can update to the latest version of OpenTelemetry, and not be held back by the plugins that they depend on.

Public portions of contrib packages (constructors, configuration, interfaces) SHOULD remain backwards compatible.

Languages which ship binary artifacts SHOULD offer [ABI compatibility](glossary.md#abi-compatibility) for contrib packages.

**Exception:** Contrib packages MAY break stability when a required downstream dependency breaks stability.
For example, a database integration may break stability if the required database client breaks stability.
However, it is strongly RECOMMENDED that older contrib packages remain stable.
A new, incompatible version of an integration SHOULD be released as a separate contrib package, rather than break the existing contrib package.

#### NOT DEFINED: Telemetry Stability

**Telemetry stability guarantees are TBD.**

Changes to telemetry produced by OpenTelemetry instrumentation SHOULD avoid breaking analysis tools, such as dashboards and alerts.
However, it is not clear at this time what type of instrumentation changes (for example, adding additional spans and attributes) would actually cause a breaking change.

#### NOT DEFINED: Semantic Conventions Stability

Telemetry stability, including semantic conventions, is not currently defined. The following practices are recommended.

Semantic Conventions SHOULD NOT be removed once they are added.
New conventions MAY be added to replace usage of older conventions, but the older conventions SHOULD NOT be removed.
Older conventions SHOULD be marked as deprecated when they are replaced by newer conventions.

### Deprecated

Signals MAY eventually be replaced. When this happens, they are marked as deprecated.

Signals SHALL only be marked as deprecated when the replacement becomes stable.
Deprecated code MUST abide by the same support guarantees as stable code.

### Removed

Support is ended by the removal of a signal from the release.
The release MUST make a major version bump when this happens.

### A note on replacing signals

Note that we currently have no plans for creating a major version of OpenTelemetry past v1.0.

For clarity, it is still possible to create new, backwards incompatible versions of existing signals without actually moving to v2.0 and breaking support.

For example, imagine we develop a new, better tracing API - let's call it AwesomeTrace.
We will never mutate the current tracing API into AwesomeTrace.
Instead, AwesomeTrace would be added as an entirely new signal which coexists and interoperates with the current tracing signal.
This would make adding AwesomeTrace a minor version bump, *not* v2.0.
v2.0 would mark the end of support for current tracing, not the addition of AwesomeTrace.
And we don't want to ever end that support, if we can help it.

This is not actually a theoretical example.
OpenTelemetry already supports two tracing APIs: OpenTelemetry and OpenTracing.
We invented a new tracing API, but continue to support the old one.

## Version numbers

OpenTelemetry clients follow [Semantic Versioning 2.0.0](https://semver.org/spec/v2.0.0.html), with the following clarifications.

OpenTelemetry clients have four components: API, SDK, Semantic Conventions, and Contrib.

For the purposes of versioning, all code within a component MUST treated as if it were part of a single package, and versioned with the same version number,
except for Contrib, which may be a collection of packages versioned separately.

* All stable API packages MUST version together, across all signals.
Stable signals MUST NOT have separate version numbers.
There is one version number that applies to all signals that are included in the API release that is labeled with that particular version number.
* SDK packages for all signals MUST version together, across all signals.
Signals MUST NOT have separate version numbers.
There is one version number that applies to all signals that are included in the SDK release that is labeled with that particular version number.
* Semantic Conventions are a single package with a single version number.
* Each contrib package MAY have it's own version number.
* The API, SDK, Semantic Conventions, and contrib components have independent version numbers.
For example, the latest version of `opentelemetry-python-api` MAY be at v1.2.3 while the latest version of `opentelemetry-python-sdk` is at v2.3.1.
* Different language implementations have independent version numbers.
For example, it is fine to have `opentelemetry-python-api` at v1.2.8 when `opentelemetry-java-api` is at v1.3.2.
* Language implementations have version numbers which are independent of the specification they implement.
For example, it is fine for v1.8.2 of `opentelemetry-python-api` to implement v1.1.1 of the specification.

**Exception:** in some languages, package managers may react poorly to experimental packages having a version higher than 0.X.
In these cases, experimental signals MAY version independently from stable signals, in order to retain a 0.X version number.
When a signal becomes stable, the version MUST be bumped to match the other stable signals in the release.

### Major version bump

Major version bumps MUST occur when there is a breaking change to a stable interface, the removal of a deprecated signal, or a drop in support for a language or runtime version.
Major version bumps SHOULD NOT occur for changes which do not result in a drop in support of some form.

### Minor version bump

Most changes to OpenTelemetry clients result in a minor version bump.

* New backward-compatible functionality added to any component.
* Breaking changes to internal SDK components.
* Breaking changes to experimental signals.
* New experimental signals are added.
* Experimental signals become stable.
* Stable signals are deprecated.

### Patch version bump

Patch versions make no changes which would require recompilation or potentially break application code.
The following are examples of patch fixes.

* Bug fixes which don't require minor version bump per rules above.
* Security fixes.
* Documentation.

Currently, the OpenTelemetry project does NOT have plans to backport bug and security fixes to prior minor versions of the SDK.
Security and bug fixes MAY only be applied to the latest minor version.
We are committed to making it feasible for end users to stay up to date with the latest version of the OpenTelemetry SDK.

## Long Term Support

![long term support](../internal/img/long-term-support.png)

### API support

Major versions of the API MUST be supported for a minimum of **three years** after the release of the next major API version.
API support is defined as follows.

* API stability, as defined above, MUST be maintained.

* A version of the SDK which supports the latest minor version of the last major version of the API will continue to be maintained during LTS.
Bug and security fixes MUST be backported. Additional feature development is NOT RECOMMENDED.

* Contrib packages available when the API is versioned MUST continue to be maintained for the duration of LTS.
Bug and security fixes will be backported.
Additional feature development is NOT RECOMMENDED.

### SDK Support

SDK stability, as defined above, will be maintained for a minimum of **one year** after the release of the next major SDK version.

### Contrib Support

Contrib stability, as defined above, will be maintained for a minimum of **one year** after the release of the next major version of a contrib package.

## OpenTelemetry GA

The term “OpenTelemetry GA” refers to the point at which OpenTracing and OpenCensus will be fully deprecated.
The **minimum requirements** for declaring GA are as followed.

* A stable version of both tracing and metrics MUST be released in at least four languages.
* CI/CD, performance, and integration tests MUST be implemented for these languages.
