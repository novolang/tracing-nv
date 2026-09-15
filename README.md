# tracing-nv

A **trace** records one request as it moves through a system: what was called,
in what order, and how long each step took. The vocabulary and the wire format
are OpenTelemetry's
[trace specification](https://opentelemetry.io/docs/specs/otel/trace/api/) and
[OTLP](https://opentelemetry.io/docs/specs/otlp/), and the headers that carry a
trace between services are
[W3C Trace Context](https://www.w3.org/TR/trace-context/). This package brings
all three to novo-lang. It is built on
[logging-nv](https://novo-lang.org/packages/logging-nv), which supplies the log
record that a span's identifiers are written into.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What it is

A **span** is one unit of work with a name, a start instant and an end instant.
A trace is a tree of spans. Here a span is a value: `start` answers one that is
not yet ended, each `with_*` call answers a new one, and `finish` answers the
one an exporter takes. `is_finished` is a predicate, so a span nobody ended is
something a test can catch.

A **span kind** says what the span measured. There are five. `TrInternal` is
work inside one process. `TrClient` and `TrServer` are the two ends of a
request. `TrProducer` and `TrConsumer` are the two ends of a queued message.

An **attribute** is a key and a typed value on a span: text, a whole number, a
real number, true or false, or an array of one of those. Attributes are flat,
because a backend indexes on them.

An **event** is something that happened at an instant inside a span. A **link**
points at a span in another trace.

A **span context** is the identity a span propagates: a 32-character **trace
id**, a 16-character **span id**, eight **trace flags** whose lowest bit is the
sampled bit, and the `tracestate` text. A **context** is a span context plus
**baggage**, which is key-and-value text that travels with a request. The
caller threads a context through its own calls. There is no ambient storage in
this package.

The **`traceparent`** header carries a span context between services, as
`version-traceid-spanid-flags`. The **`tracestate`** header carries
vendor-specific text beside it, most recent writer first.

A **sampler** decides whether a trace is kept. The decision is made once, when a
span has no parent, and every child inherits it through the sampled bit.

An **exporter** sends finished spans somewhere. `TracingExporter[e]` is a trait
with one effect parameter, so an exporter over the network costs `[net]`, one
that writes a file costs `[fs]`, and the in-memory one a test drains costs
`[mutate]`. A pipeline written against the trait is charged exactly what the
caller's own exporter costs.

A **batch** is the queue in front of an exporter. It is a value the caller pushes
into and drains, with three bounds: the largest batch, the longest delay and the
largest queue.

**OTLP** is the protocol a collector speaks. This package writes the JSON form
over HTTP.

## Install

```
novo pkg add tracing-nv
```

## Example

```novo
use trctx
use trid
use trotlp
use trprop
use trsample
use trspan

fn main() [io, rand, time]
    // A tracer: one sampler, keeping a tenth of the traces it roots.
    let t = trid.tracer(trsample.parent_based_with_root(trsample.trace_id_ratio(0.1)))

    // Read the inbound header. A malformed one restarts the trace here.
    let arrived = trprop.extract("00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01", "")
    let ctx = trid.continue_trace(arrived)

    // The clock, read once, by the caller. Nothing stamps itself.
    let began = trid.now_unix()

    // Start a server span under that context.
    let started = trid.start_span(t, ctx, "GET /users", TrServer, [], [], began)

    // Attach an attribute and close the span. Each call answers a new value.
    let tagged = trspan.with_attr(started.span, trspan.attr_int("http.status_code", 200))
    let done = trspan.finish(tagged, began + 0.012)

    // The exact document a collector would receive. Sending it is a separate call.
    println(trotlp.encode(trotlp.resource("users", []), trotlp.this_scope(), [done]))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails on
purpose: every test reaches a `not implemented` panic.

## What the package contains

| Module | Contents |
| --- | --- |
| `trspan` | The span: its kind, its typed attributes, its events, its links, its status, and the calls that add to one and close it. |
| `trctx` | The span context and the context a caller threads, with baggage, the sampled bit and the two invalid identifiers. |
| `trprop` | The `traceparent`, `tracestate` and `baggage` headers, read and written, with a named reason for every way a header goes wrong. |
| `trsample` | The samplers: always, never, a ratio of the trace id, the parent's decision, a composition of the two, and one that drops by span name. |
| `trexport` | The `TracingExporter[e]` trait, the outcome type, the batch queue with its three bounds, and the in-memory exporter a test drains. |
| `trotlp` | The OTLP/JSON document, encoded from spans, and the exporter that sends it over HTTP. |
| `trid` | The two things a value cannot supply: a random identifier and the current time. Also the tracer, and a seeded tracer for tests. |
| `trlog` | The join between a log line and a span, in both directions. |

## How to choose an entry point

**A service starts at `trid`.** Build a tracer once with the sampler it wants,
then call `start_span` per request. `TrStarted` hands back three things: the
span to finish, the context to thread into everything the handler calls, and
whether the span is worth attaching attributes to.

**A service at the edge calls `trprop.extract` first.** It reads the inbound
headers and answers a context that is always usable. Pass it to
`trid.continue_trace`. On the way out, `trprop.inject_traceparent` and
`inject_tracestate` write the headers for a downstream call.

**A test starts at `trid.seeded_tracer`.** Its identifiers are predictable, so a
suite can assert which trace id a child carried, not merely that it carried
one.

**Exporting has two halves, and they are separate calls.**
`trotlp.encode` answers the exact JSON document a collector would receive and
performs nothing, so a test can assert on it. `trotlp.send` and the exporter
that wraps it reach the network. A caller with its own transport uses the
encoding and never takes `[net]`.

**A service that exports in batches holds a `TrBatch`.** Push finished spans
into it, ask `should_flush` with a clock reading the caller already has, and
call `flush_into` with its exporter.

**A test collects spans with `trexport.memory_exporter`.** It writes into a slot
the caller owns, so the pipeline costs `[mutate]` and reaches no collector.

**A service that also logs calls `trlog.with_trace`.** It puts the trace id, the
span id and the trace flags onto a log record, which is the one thing that makes
a log line and a span findable from each other.

## The rules a user needs

1. **A span is finished by value, not by a method on a handle.** `trspan.start`
   answers a span whose end instant is `0.0`. `trspan.finish` answers the one an
   exporter takes. `trspan.is_finished` is the check.
2. **Time is a parameter everywhere except `trid.now_unix`.** Read the clock
   once, per request, and pass the value. OpenTelemetry trace specification,
   "Span Creation".
3. **Identifiers are random and not counters.** A trace id is 128 bits and a
   span id is 64 bits. A ratio sampler hashes the trace id, so a non-uniform
   identifier makes it keep the wrong fraction. OpenTelemetry trace
   specification, "SpanContext".
4. **A span's kind is not cosmetic.** A backend builds its service graph from
   `TrClient` and `TrServer` pairs. A span that crossed a process boundary and
   called itself `TrInternal` leaves a hole no attribute fills. OpenTelemetry
   trace specification, "SpanKind".
5. **Attributes are a list, not a map, and duplicates are reported rather than
   refused.** The specification defines no uniqueness rule a library may enforce
   for a caller. `trspan.duplicate_keys` is the report.
6. **A malformed `traceparent` is not an error.** `trprop.extract` never answers
   a `Result`. A receiver that cannot parse the header must restart the trace
   and must not reject the request. The reason is carried on the answer, because
   two hundred unparseable headers an hour is a fact an operator needs. W3C
   Trace Context section 3.2.2.3.
7. **A version higher than `00` is read, not refused.** The first four fields
   are parsed and the rest ignored, and `TrPropFutureVersion` says so. The
   version `ff` is reserved and is invalid. W3C Trace Context section 3.2.2.3.
8. **`tracestate` is text and stays text.** Its ordering is its meaning: the
   vendor that wrote most recently is first. `trprop.state_set` performs the one
   mutation the specification defines and carries everything else through
   unchanged. W3C Trace Context section 3.3.
9. **The sampled bit is one bit of eight.** Use `trctx.set_sampled` rather than
   assigning the whole flags byte, so a reserved bit a middlebox set still
   travels. W3C Trace Context section 3.3.
10. **The sampling decision is made at the root and inherited.** A span with a
    parent does not ask the sampler. A sampler that ran per span would keep the
    child of a dropped parent, and a backend cannot tell that tree from a
    crashed service. OpenTelemetry trace SDK specification, "Sampling".
11. **`trace_id_ratio` decides from the trace id, not from a random draw.** Five
    services running the same ratio therefore agree about the same trace. Five
    services each tossing their own coin at ratio 0.1 keep one trace in a
    hundred thousand complete.
12. **`parent_based` honours whatever the upstream decided.** At a trust
    boundary that means an upstream which samples everything makes this service
    export everything. `parent_based_with_root` keeps the tree intact and still
    bounds a root.
13. **`TrRecordOnly` is recorded and not exported.** It is a third state, not a
    synonym for dropped. `trsample.is_recorded` and `trsample.is_exported` are
    the two questions, and they are not the same question.
14. **A dropped span still propagates.** `start_span` answers a span and a
    context even on a `TrDrop`, because a downstream service that samples on its
    own needs this span's id as its parent. `trid.is_recording` is what tells a
    caller not to spend anything on attributes.
15. **Export takes a batch.** A collector receiving one span per request is the
    commonest way a traced service becomes slower than the service it traces.
    An exporter that wants one span still receives a list of one.
16. **`TrExportOutcome` is not a `Result`.** A collector that accepted 400 of
    500 spans is the ordinary outcome, and it carries both numbers.
17. **Export never fails the caller's work.** The specification requires
    telemetry not to affect the application. The one error path is about the
    exporter itself.
18. **An OTLP timestamp is a string, not a JSON number.** It is a nanosecond
    count that exceeds 2^53 for every date after 1970, and a JSON number loses
    precision in any consumer using IEEE doubles. `trotlp.nanos_of` answers the
    string. OTLP specification, "OTLP/JSON".
19. **`SPAN_KIND_INTERNAL` is 1, not 0.** An encoder that used an enum's
    position would write every internal span as unspecified.
    `trotlp.kind_code` is the mapping.
20. **The OTLP document nests three levels:** `resourceSpans`, then
    `scopeSpans`, then `spans`. A document with two levels validates and
    contains nothing.
21. **An all-zero trace id means there is no trace.** `trlog.with_trace` adds
    nothing for an invalid context. A log line carrying all zeroes looks
    correlated and joins to nothing.
22. **The correlation keys are `trace_id`, `span_id` and `trace_flags`.** They
    are OpenTelemetry's logs data model spellings, so a collector that already
    knows how to correlate needs no configuration. `trlog.trace_id_key` and its
    two siblings are the strings.
23. **A span event is dropped when its span is.** `trlog.as_event` puts a log
    line on the span, which needs no correlation. An unsampled request's events
    are gone, while its log lines are not.

## What is not included

- **Ambient context.** There is no thread-local current span. An instrumented
  function takes a `TrContext` and returns one, which is one parameter per
  boundary. A task moved between threads keeps its context, and a test asserts
  `child.context.trace_id == parent.context.trace_id` with no setup.
- **Metrics.** A metric is a different signal with a different document.
  `std.otlp`'s `record_metric` keeps them, so a program has one endpoint to
  configure rather than two.
- **Protobuf.** OTLP defines both encodings over HTTP and every collector
  accepts JSON. Protobuf would need a generated schema this package cannot check
  against the specification. A JSON span is roughly three times the size of its
  protobuf.
- **A background exporter thread.** The batch is a value and the flush is the
  caller's. A service that already has a loop puts the drain in it.
- **A wrapping `with_span` call.** A span here is a value, so starting and
  finishing are two ordinary calls and nothing is wrapped in a thunk.
- **Running on a microcontroller.** A trace id is 32 characters of text and a
  span carries three lists. A device that wants to be traced sends its frames
  through the deferred-logging path and is correlated on the host.

## Related packages

- [logging-nv](https://novo-lang.org/packages/logging-nv) supplies the log
  record `trlog` writes into and the sink it emits through. Taking this package
  therefore also resolves logging-nv's own closure, which is seven further
  packages. Six of the seven are there for its device bridge.
- [logging-core-nv](https://novo-lang.org/packages/logging-core-nv) publishes
  the same record and sink declarations for a library that performs nothing. A
  program cannot hold it and logging-nv at once, so it cannot hold it and this
  package at once either.
- `std.otlp` in the standard library keeps metrics, and keeps the configuration
  convention this package reads: `OTLP_ENDPOINT`, `OTLP_SERVICE`, and `off` to
  disable. `trotlp.endpoint_from_env` answers from the same variables rather
  than a second spelling. Its `with_span` is deferred and needs closure support.
- `std.log` in the standard library is the logging this package does not use.
  Its records carry no place for a trace id.

## Tests

```bash
novo test tests                            # every suite
novo test tests/trspan_tests.nv            # a span as a value, and its states
novo test tests/trctx_tests.nv             # the context, and the flags byte
novo test tests/trprop_tests.nv            # the W3C headers, and the restart rule
novo test tests/trsample_tests.nv          # who decides, and what a decision means
novo test tests/trexport_tests.nv          # the batch, and the exporter trait
novo test tests/trotlp_tests.nv            # the document a collector receives
novo test tests/trid_tests.nv              # the effects, and the seeded tracer
novo test tests/trlog_tests.nv             # the join key, in both directions
```

`novo test` fails on purpose today. Every assertion reaches a `not implemented:
tracing-nv.<module>.<fn>` panic, because every body is a `todo()`. The tests are
the specification the implementation will have to satisfy.

The reference data is the specifications' own: W3C Trace Context's example
headers and its list of invalid ones, OpenTelemetry's span kind and status
enumerations with their protocol numbers, and the OTLP/JSON document shape.

The malformed headers in `tests/trprop_tests.nv` have all arrived at a real
endpoint: a truncating proxy, a client that built the header by hand, a
middlebox that raised the version, and a service that sent all zeroes. Every one
of them restarts the trace and none of them refuses the request.

`tests/trotlp_tests.nv` covers three faults that each produce a 200 from the
collector and no data in the backend: a timestamp written as a JSON number, an
internal span written with kind 0, and the three-level nesting written with two.
`tests/trctx_tests.nv` asserts that setting the sampled bit leaves the other
seven alone. `tests/trexport_tests.nv` carries an assertion the compiler makes
rather than `test.assert`: a pipeline declared `[mutate]` runs over the
in-memory exporter, whose implementation supplies exactly `[mutate]`.

## Implementation status

| Item | Implemented |
| --- | --- |
| `trspan.TrSpanKind`, `.TrAttrValue`, `.TrAttr`, `.TrEvent`, `.TrLink`, `.TrStatusCode`, `.TrStatus`, `.TrSpan` | declared |
| `trspan`'s twenty functions, from `start` to `kind_name` | no |
| `trctx.TrSpanContext`, `.TrContext`, `.TrBaggage` | declared |
| `trctx`'s twelve functions, from `empty` to `invalid_span_id` | no |
| `trprop.TrPropagation`, `.TrPropReason` | declared |
| `trprop`'s fourteen functions, from `extract` to `reason_message` | no |
| `trsample.TrDecision`, `.TrSamplingResult`, `.TrSamplingParams`, `.TrSampler` | declared |
| `trsample`'s thirteen functions, from `always_on` to `is_recorded` | no |
| `trexport.TrExportFault` and `impl Error`, `.TrExportOutcome`, `.TracingExporter[e]`, `.TrMemoryExporter`, `.TrBatchLimits`, `.TrBatch` | declared |
| `impl TracingExporter[mutate] for TrMemoryExporter`: `export`, `flush`, `shutdown` | no |
| `trexport`'s thirteen functions, from `default_limits` to `is_retryable` | no |
| `trotlp.TrResource`, `.TrScope`, `.TrOtlpExporter` | declared |
| `impl TracingExporter[net] for TrOtlpExporter`: `export`, `flush`, `shutdown` | no |
| `trotlp`'s fifteen functions, from `encode` to `traces_path` | no |
| `trid.TrTracer`, `.TrStarted` | declared |
| `trid`'s eleven functions, from `tracer` to `seeded_ids` | no |
| `trlog`'s thirteen functions, from `trace_id_key` to `record_of_event` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
