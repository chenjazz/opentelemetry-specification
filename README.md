# OpenTelemetry Specification 规范

[![Checks](https://github.com/open-telemetry/opentelemetry-specification/workflows/Checks/badge.svg?branch=main)](https://github.com/open-telemetry/opentelemetry-specification/actions?query=workflow%3A%22Checks%22+branch%3Amain)
![GitHub tag (latest SemVer)](https://img.shields.io/github/tag/open-telemetry/specification.svg)

![OpenTelemetry Logo](https://opentelemetry.io/img/logos/opentelemetry-horizontal-color.png)

_Curious about what OpenTelemetry is? Check out our [website](https://opentelemetry.io) for an explanation!_

_好奇什么是OpenTelemetry？查看我们的网站获取解释！_

The OpenTelemetry specification describes the cross-language requirements and expectations for all OpenTelemetry implementations. Substantive changes to the specification must be proposed using the [OpenTelemetry Enhancement Proposal](https://github.com/open-telemetry/oteps) process. Small changes, such as clarifications, wording changes, spelling/grammar corrections, etc. can be made directly via pull requests.

OpenTelemetry 规范描述了所有 OpenTelemetry 实现的跨语言要求和期望。必须使用 OpenTelemetry Enhancement Proposal 流程提出对规范的实质性更改。小的更改，例如澄清、措辞更改、拼写/语法更正等，可以直接通过 pull requests 进行。

Questions that needs additional attention can be brought to the regular
specifications meeting. EU and US timezone friendly meeting is held every
Tuesday at 8 AM pacific time. Meeting notes are held in the [google
doc](https://docs.google.com/document/d/1-bCYkN-DWJq4jw1ybaDZYYmx-WAe6HnwfWbkm8d57v8/edit?usp=sharing).
APAC timezone friendly meeting is held Tuesdays, 4PM pacific time. See
[OpenTelemetry calendar](https://github.com/open-telemetry/community#calendar).

需要额外注意的问题可以提交给定期的 规格会议。欧盟和美国时区友好会议于太平洋时间每周二上午 8 点举行。会议记录保存在 google 文档中。亚太时区友好会议于太平洋时间每周二下午 4 点举行。请参阅 OpenTelemetry 日历。。

Escalations to technical committee may be made over the
[e-mail](https://github.com/open-telemetry/community#tc-technical-committee).
Technical committee holds regular meetings, notes are held
[here](https://docs.google.com/document/d/17v2RMZlJZkgoPYHZhIFTVdDqQMIAH8kzo8Sl2kP3cbY/edit?usp=sharing).

可以通过电子邮件向技术委员会上报。技术委员会定期召开会议，记录在这里。

## Table of Contents

- [Overview](specification/overview.md) 总览
- [Glossary](specification/glossary.md)  词汇表
- [Versioning and stability for OpenTelemetry clients](specification/versioning-and-stability.md) OpenTelemetry 客户端的版本控制和稳定性
- [Library Guidelines](specification/library-guidelines.md) 库准则
  - [Package/Library Layout](specification/library-layout.md)
  - [General error handling guidelines](specification/error-handling.md)
- API Specification API规范
  - [Context](specification/context/context.md)
    - [Propagators](specification/context/api-propagators.md)
  - [Baggage](specification/baggage/api.md)
  - [Tracing](specification/trace/api.md)
  - [Metrics](specification/metrics/api.md)
- SDK Specification  SDK规范
  - [Tracing](specification/trace/sdk.md)
  - [Resource](specification/resource/sdk.md)
  - [Configuration](specification/sdk-configuration.md)
- Data Specification   数据规范
  - [Semantic Conventions](specification/overview.md#semantic-conventions)
  - [Protocol](specification/protocol/README.md)
    - [Metrics](specification/metrics/datamodel.md)
    - [Logs](specification/logs/data-model.md)
- About the Project   关于工程
  - [Timeline](#project-timeline)
  - [Notation Conventions and Compliance](#notation-conventions-and-compliance)
  - [Versioning the Specification](#versioning-the-specification)
  - [Acronym](#acronym)
  - [Contributions](#contributions)
  - [License](#license)

## Project Timeline

The current project status as well as information on notable past releases is found at
[the OpenTelemetry project page](https://opentelemetry.io/status/).

Information about current work and future development plans is found at the
[specification development milestones](https://github.com/open-telemetry/opentelemetry-specification/milestones).

## Notation Conventions and Compliance   符号约定和合规性


The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in the [specification](./specification/overview.md) are to be interpreted as described in [BCP 14](https://tools.ietf.org/html/bcp14) [[RFC2119](https://tools.ietf.org/html/rfc2119)] [[RFC8174](https://tools.ietf.org/html/rfc8174)] when, and only when, they appear in all capitals, as shown here.

An implementation of the [specification](./specification/overview.md) is not compliant if it fails to satisfy one or more of the "MUST", "MUST NOT", "REQUIRED", "SHALL", or "SHALL NOT" requirements defined in the [specification](./specification/overview.md).
Conversely, an implementation of the [specification](./specification/overview.md) is compliant if it satisfies all the "MUST", "MUST NOT", "REQUIRED", "SHALL", and "SHALL NOT" requirements defined in the [specification](./specification/overview.md).

## Versioning the Specification   版本规范


Changes to the [specification](./specification/overview.md) are versioned according to [Semantic Versioning 2.0](https://semver.org/spec/v2.0.0.html) and described in [CHANGELOG.md](CHANGELOG.md). Layout changes are not versioned. Specific implementations of the specification should specify which version they implement.

Changes to the change process itself are not currently versioned but may be independently versioned in the future.

## Acronym  缩写

The official acronym used by the OpenTelemetry project is "OTel".

Please refrain from using "OT" in order to avoid confusion with the now deprecated "OpenTracing" project.

## Contributions   贡献

See [CONTRIBUTING.md](CONTRIBUTING.md) for details on contribution process.

## License

By contributing to OpenTelemetry Specification repository, you agree that your contributions will be licensed under its [Apache 2.0 License](https://github.com/open-telemetry/specification/blob/main/LICENSE).
