# Default SDK Configuration

<details>
<summary>Table of Contents</summary>

* [Abstract](#abstract)
* [Configuration Interface](#configuration-interface)

</details>

## Abstract

The default Open Telemetry SDK (hereafter referred to as "The SDK")
is highly configurable. This specification outlines the mechanisms by
which the SDK can be configured. It does
not attempt to specify the details of what can be configured.

默认的 Open Telemetry SDK（以下简称“SDK”）是高度可配置的。本规范概述了可以配置 SDK 的机制。它不会尝试指定可以配置的详细信息。

## Configuration Interface 配置接口

### Programmatic  程序化

The SDK MUST provide a programmatic interface for all configuration.
This interface SHOULD be written in the language of the SDK itself.
All other configuration mechanisms SHOULD be built on top of this interface.

SDK 必须为所有配置提供一个编程接口。这个接口应该用 SDK 本身的语言编写。所有其他配置机制都应该建立在这个接口之上。

An example of this programmatic interface is accepting a well-defined
struct on an SDK builder class. From that, one could build a CLI that accepts a
file (YAML, JSON, TOML, ...) and then transforms into that well-defined struct
consumable by the programatic interface.

此编程接口的一个示例是在 SDK 构建器类上接受定义良好的结构。由此，我们可以构建一个接受文件（YAML、JSON、TOML 等）的 CLI，然后通过编程接口将其转换为定义良好的结构体。

### Other Mechanisms

Additional configuration mechanisms SHOULD be provided in whatever
language/format/style is idiomatic for the language of the SDK. The
SDK can include as many configuration mechanisms as appropriate.

应该以 SDK 语言惯用的任何语言/格式/样式提供额外的配置机制。 SDK 可以包含尽可能多的配置机制。
