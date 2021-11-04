# Error handling in OpenTelemetry

OpenTelemetry generates telemetry data to help users monitor application code.
In most cases, the work that the library performs is not essential from the perspective of application business logic.
We assume that users would prefer to lose telemetry data rather than have the library significantly change the behavior of the instrumented application.

OpenTelemetry 生成遥测数据以帮助用户监控应用程序代码。在大多数情况下，从应用程序业务逻辑的角度来看，库执行的工作并不是必不可少的。我们假设用户更愿意丢失遥测数据，而不是让库显着改变检测应用程序的行为。

OpenTelemetry may be enabled via platform extensibility mechanisms, or dynamically loaded at runtime.
This makes the use of the library non-obvious for end users, and may even be outside of the application developer's control.
This makes for some unique requirements with respect to error handling.

OpenTelemetry 可以通过平台扩展机制启用，或者在运行时动态加载。这使得库的使用对于最终用户来说并不明显，甚至可能超出应用程序开发人员的控制范围。这里对错误处理提出了一些独特的要求。

## Basic error handling principles  基本错误处理原则


OpenTelemetry implementations MUST NOT throw unhandled exceptions at run time.

OpenTelemetry 实现不得在运行时抛出未处理的异常。

1. API methods MUST NOT throw unhandled exceptions when used incorrectly by end users.
   The API and SDK SHOULD provide safe defaults for missing or invalid arguments.
   For instance, a name like `empty` may be used if the user passes in `null` as the span name argument during `Span` construction.
   
   当最终用户使用不当时，API 方法不得抛出未处理的异常。 API 和 SDK 应该为丢失或无效的参数提供安全的默认值。例如，如果用户在 Span 构造期间传入 null 作为 Span 名称参数，则可以使用像 empty 这样的名称。
   
   
2. The API or SDK may _fail fast_ and cause the application to fail on initialization, e.g. because of a bad user config or environment, but MUST NOT cause the application to fail later at run time, e.g. due to dynamic config settings received from the Collector.

API 或 SDK 可能会快速失败并导致应用程序在初始化时失败，例如由于错误的用户配置或环境，但不得导致应用程序在运行时稍后失败，例如由于从Collector收到的动态配置设置。

3. The SDK MUST NOT throw unhandled exceptions for errors in their own operations.
   For example, an exporter should not throw an exception when it cannot reach the endpoint to which it sends telemetry data.
   
   SDK 不得因自身操作中的错误引发未处理的异常。例如，当导出器无法到达将遥测数据发送到的端点时，它不应抛出异常。

## Guidance 指导

1. API methods that accept external callbacks MUST handle all errors.  接受外部回调的 API 方法必须处理所有错误。
2. Background tasks (e.g. threads, asynchronous tasks, and spawned processes) should run in the context of a global error handler to ensure that exceptions do not affect the end user application. 后台任务（例如线程、异步任务和衍生进程）应在全局错误处理程序的上下文中运行，以确保异常不会影响最终用户应用程序。
3. Long-running background tasks should not fail permanently in response to internal errors.
   In general, internal exceptions should only affect the execution context of the request that caused the exception.长时间运行的后台任务不应因响应内部错误而永久失败。一般来说，内部异常应该只影响导致异常的请求的执行上下文。
4. Internal error handling should follow language-specific conventions.
   In general, developers should minimize the scope of error handlers and add special processing for expected exceptions.内部错误处理应遵循特定于语言的约定。一般来说，开发人员应该尽量减少错误处理程序的范围，并为预期的异常添加特殊处理。
5. Beware external callbacks and overrideable interfaces: Expect them to throw.当心外部回调和可覆盖的接口：期待它们抛出。？
6. Beware to call any methods that wasn't explicitly provided by API and SDK users as a callbacks.
   Method `ToString` that SDK may decide to call on user object may be badly implemented and lead to stack overflow.
   It is common that the application never calls this method and this bad implementation would never be caught by an application owner.
   请注意调用 API 和 SDK 用户未明确提供的任何方法作为回调。 SDK 可能决定在用户​​对象上调用的 ToString 方法可能未正确实现并导致堆栈溢出。应用程序从不调用此方法是很常见的，并且应用程序所有者永远不会发现这种糟糕的实现。
7. Whenever API call returns values that is expected to be non-`null` value - in case of error in processing logic - SDK MUST return a "no-op" or any other "default" object that was (_ideally_) pre-allocated and readily available.
   This way API call sites will not crash on attempts to access methods and properties of a `null` objects.每当 API 调用返回预期为非空值的值时 - 在处理逻辑中出现错误的情况下 - SDK 必须返回（理想情况下）预先分配且随时可用的“无操作”或任何其他“默认”对象.这样 API 调用站点就不会在尝试访问空对象的方法和属性时崩溃。

## Error handling and performance  错误处理和性能


Error handling and extensive input validation may cause performance degradation, especially on dynamic languages where the input object types are not guaranteed in compile time.
Runtime type checks will impact performance and are error prone, exceptions may occur despite the best effort.

错误处理和广泛的输入验证可能会导致性能下降，尤其是在编译时无法保证输入对象类型的动态语言上。运行时类型检查会影响性能并且容易出错，尽管尽了最大努力也可能发生异常。

It is recommended to have a global exception handling logic that will guarantee that exceptions are not leaking to the user code.
And make a reasonable trade off of the SDK performance and fullness of type checks that will provide a better on-error behavior and SDK errors troubleshooting.

建议有一个全局异常处理逻辑，以保证异常不会泄露给用户代码。并在 SDK 性能和类型检查的完整性之间做出合理的权衡，这将提供更好的错误行为和 SDK 错误故障排除。

## Self-diagnostics  自我诊断


All OpenTelemetry libraries -- the API, SDK, exporters, instrumentations, etc. -- are encouraged to expose self-troubleshooting metrics, spans, and other telemetry that can be easily enabled and filtered out by default.

鼓励所有 OpenTelemetry 库——API、SDK、导出器、instrumentations等——公开自我故障排除指标、跨度和其他默认情况下可以轻松启用和过滤掉的遥测。

One good example of such telemetry is a `Span` exporter that indicates how much time exporters spend uploading telemetry.
Another example may be a metric exposed by a `SpanProcessor` that describes the current queue size of telemetry data to be uploaded.

此类遥测的一个很好的例子是 Span 导出器，它指示导出器在上传遥测上花费了多少时间。另一个例子可能是一个由 SpanProcessor 公开的度量，它描述了要上传的遥测数据的当前队列大小。

Whenever the library suppresses an error that would otherwise have been exposed to the user, the library SHOULD log the error using language-specific conventions.
SDKs MAY expose callbacks to allow end users to handle self-diagnostics separately from application code.

每当库 隐藏  会暴露给用户的错误时，库应该使用特定于语言的约定记录错误。 SDK 可以公开回调以允许最终用户独立于应用程序代码处理自我诊断。

## Configuring Error Handlers

SDK implementations MUST allow end users to change the library's default error handling behavior for relevant errors.
Application developers may want to run with strict error handling in a staging environment to catch invalid uses of the API, or malformed config.
Note that configuring a custom error handler in this way is the only exception to the basic error handling principles outlined above.
The mechanism by which end users set or register a custom error handler should follow language-specific conventions.

SDK 实现必须允许最终用户针对相关错误更改库的默认错误处理行为。应用程序开发人员可能希望在暂存环境中以严格的错误处理方式运行，以捕获 API 的无效使用或格式错误的配置。请注意，以这种方式配置自定义错误处理程序是上述基本错误处理原则的唯一例外。最终用户设置或注册自定义错误处理程序的机制应遵循特定于语言的约定。

### Examples

These are examples of how end users might register custom error handlers.
Examples are for illustration purposes only. OpenTelemetry client authors
are free to deviate from these provided that their design matches the requirements outlined above.

这些是最终用户如何注册自定义错误处理程序的示例。示例仅用于说明目的。 OpenTelemetry 客户端作者可以自由地偏离这些，前提是他们的设计符合上述要求。

#### Go

```go
// The basic Error Handler interface
type ErrorHandler interface {
  Handle(err error)
}

func Handler() ErrorHandler
func SetHandler(handler ErrorHandler)
```

```go
// Registering a custom Error Handler
type IgnoreExporterErrorsHandler struct{}

func (IgnoreExporterErrorsHandler) Handle(err error) {
    switch err.(type) {
    case *SpanExporterError:
    default:
        fmt.Println(err)
    }
}

func main() {
    // Other setup ...
    opentelemetrysdk.SetHandler(IgnoreExporterErrorsHandler{})
}

```

##### Java

OpenTelemetry Java uses [java.util.logging](https://docs.oracle.com/javase/7/docs/api/java/util/logging/package-summary.html)
to output and handle all logs, including errors. Custom handlers and filters can be registered both in code and using the Java logging configuration file.  

OpenTelemetry Java 使用 java.util.logging 来输出和处理所有日志，包括错误。自定义处理程序和过滤器既可以在代码中注册，也可以使用 Java 日志配置文件进行注册。

```properties
## Turn off all error logging
io.opentelemetry.level = OFF
```

```java
// Creating a custom filter which does not log errors that come from the exporter
public class IgnoreExportErrorsFilter implements Filter {

 public boolean isLoggable(LogRecord record) {
    return !record.getMessage().contains("Exception thrown by the export");
 }
}
```

```properties
## Registering the custom filter on the BatchSpanProcessor
io.opentelemetry.sdk.trace.export.BatchSpanProcessor = io.opentelemetry.extensions.logging.IgnoreExportErrorsFilter
```
