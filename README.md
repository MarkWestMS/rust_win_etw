# Rust support for Event Tracing for Windows (ETW)

Provides the `#[trace_logging_provider]` macro, which allows you to define a
[Trace Logging Provider](https://docs.microsoft.com/en-us/windows/win32/etw/about-event-tracing#providers)
for use with the [Event Tracing for Windows (ETW)](https://docs.microsoft.com/en-us/windows/win32/etw/event-tracing-portal)
framework.

This macro is intended for use only when targeting Windows. When targeting other platforms,
this macro will still work, but will generate code that does nothing.

This framework allows applications to log schematized events, rather than textual strings.
ETW analysis tools can reliably identify fields within your events, and treat them as
strongly-typed data, rather than text strings.

To use [tracing](https://tracing.rs) with ETW, see [tracing-etw](https://github.com/microsoft/tracing-etw).

# How to create and use an event provider

In ETW, an _event provider_ is a software object that generates events. _Event controllers_
set up event logging sessions, and _event consumers_ read and interpret event data. This crate
focuses on enabling applications to create _event providers_.

## Add crate dependencies
Add these dependencies to your `Cargo.toml` file:

```text
[dependencies]
win_etw_macros = "0.1.*"
win_etw_provider = "0.1.*"
```

`win_etw_macros` contains the procedural macro that generates eventing code.
`win_etw_provider` contains library code that is called by the code that is generated by
`win_etw_macros`.

## Create a new GUID for your event provider
You _must_ assign a new, unique GUID to each event provider. ETW uses this GUID to identify
events that are generated by your provider. Windows contains many event providers, so it is
important to be able to select only the events generated by your application. This GUID is also
used internally by ETW to identify event metadata (field types), so it is important that your
GUID be unique. Otherwise, events from conflicting sources that use the same GUID may be
incorrectly interpreted.

There are many tools which can create a GUID, such as:

* In Visual Studio, in the Tools menu, select "Create GUID".
* From a Visual Studio command line, run `uuidgen.exe`.
* From an Ubuntu shell, run `uuidgen`.

## Define the event provider and its events
Add a trait definition to your source code and annotate it with the
`#[trace_logging_provider(guid = "...")]` macro, using the GUID that you just created. The trait
definition is only used as input to the procedural macro; the trait is not emitted into your
crate, and cannot be used as a normal trait.

In the trait definition, add method signatures. Each method signature defines an _event type_.
The parameters of each method define the fields of the event type. Only a limited set of field
types are supported (enumerated below).

The `#[trace_logging_provider]` macro consumes the trait definition and produces a `struct`
definition with the same name and the same method signatures. (The trait is _not_ available for
use as an ordinary trait.)

```rust
# use win_etw_macros::trace_logging_provider;
#[trace_logging_provider(guid = "... your guid here ...")]
pub trait MyAppEvents {
    fn http_request(client_address: &SockAddr, is_https: bool, status_code: u32, status: &str);
    fn database_connection_created(connection_id: u64, server: &str);
    fn database_connection_closed(connection_id: u64);
    // ...
}
```

## Create an instance of the event provider
At initialization time (in your `fn main()`, etc.), create an instance of the event provider:

```rust
let my_app_events = MyAppEvents::new();
```

Your application should only create a single instance of each event provider, per process.
That is, you should create a single instance of your event provider and share it across your
process. Typically, an instance is stored in static variable, using a lazy / atomic assignment.
There are many crates and types which can support this usage pattern.

## Call event methods to report events
To report an event, call one of the methods defined on the event provider. The method will
call into ETW to report the event, but there is no guarantee that the event is stored or
forwarded; events can be dropped if event buffer resources are scarce.

```rust
my_app_events.client_connected(None, &"192.168.0.42:6667".parse(), false, 100, "OK");
```

Note that all generated event methods have an added first parameters,
`options: Option<&EventOptions>`. This parameter allows you to override per-event parameters,
such as the event level and event correlation IDs. In most cases, you should pass `None`.

# Supported field types
Only a limited set of field types are supported.

* Integer primitives up to 64 bits: `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`
* Floating point primitives: `f32`, `f64`
* Architecture-dependent sizes: `usize`, `isize`.
* Boolean: `bool`
* Slices of all of the supported primitives: `&[u8]`, `&[u16]`, etc.
* Windows [`FILETIME`](https://docs.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-filetime).
  The type must be declared _exactly_ as `FILETIME`; type aliases or fully-qualified paths
  (such as `winapi::shared::minwindef::FILETIME`) _will not work_. The parameter type in the
  generated code will be `win_etw_provider::FILETIME`, which is a newtype over `u64`.
* `std::time::SystemTime` is supported, but it must be declared _exactly_ as `SystemTime`;
  type aliases or fully-qualified paths (such as `std::time::SystemTime`) _will not work_.
* `SockAddr`, `SockAddrV4`, and `SockAddrV6` are supported. They must be declared exactly as
  shown, not using fully-qualified names or type aliases.

# How to capture and view events

There are a variety of tools which can be used to capture and view ETW events.
The simplest tool is the `TraceView` tool from the Windows SDK. Typically it is installed
at this path: `C:\Program Files (x86)\Windows Kits\10\bin\10.0.<xxxxx>.0\x64\traceview.exe`,
where `<xxxxx>` is the release number of the Windows SDK.

Run `TraceView`, then select "File", then "Create New Log Session". Select "Manually Entered
GUID or Hashed Name" and enter the GUID that you have assigned to your event provider. Click OK.
The next dialog will prompt you to choose a source of WPP format information; select Auto
and click OK.

At this point, `TraceView` should be capturing events (for your assigned GUID) and displaying
them in real time, regardless of which process reported the events.

These tools can also be used to capture ETW events:
* [Windows Performance Recorder](https://docs.microsoft.com/en-us/windows-hardware/test/wpt/windows-performance-recorder)
  This tool is intended for capturing system-wide event streams. It is not useful for capturing
  events for a specific event provider.
* [logman](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/logman)
  is a command-line tool for managing events.
* [Tracelog](https://docs.microsoft.com/en-us/windows-hardware/drivers/devtest/tracelog)

There are other tools, such as the Windows Performance Recorder, which can capture ETW events.

# Ideas for improvement

* Better handling of per-event overrides, rather than using `Option<&EventOptions>`.

# References

* [Event Tracing for Windows (ETW) Simplified](https://support.microsoft.com/en-us/help/2593157/event-tracing-for-windows-etw-simplified)
* [TraceLogging for Event Tracing for Windows (ETW)](https://docs.microsoft.com/en-us/windows/win32/tracelogging/trace-logging-portal)
* [Record and View TraceLogging Events](https://docs.microsoft.com/en-us/windows/win32/tracelogging/tracelogging-record-and-display-tracelogging-events)

# Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
