# tracing-nv

OpenTelemetry tracing for novo-lang: spans as values, a context the
caller threads, W3C propagation as a pure codec, samplers as named
functions, and a batch exporter contract whose effect parameter charges a
caller what its own exporter costs.

**Status: NOT IMPLEMENTED — interface only.**  Every `pub fn` body is a
`todo()`, so the signatures, the effect rows and the tests are published
and nothing is implemented.  The first implementation is the `0.1.0`
published over this.

## What this is

| module | holds | rows |
| --- | --- | --- |
| `trspan` | the span, its attributes, events, links and status | `[]` |
| `trctx` | the span context, the threaded context, baggage | `[]` |
| `trprop` | W3C `traceparent`, `tracestate` and `baggage` | `[]` |
| `trsample` | samplers as named functions, and the decision | `[]` |
| `trexport` | **`TracingExporter[e]`**, the batch contract | `[e]`, `[mutate]` |
| `trotlp` | the OTLP/JSON encoding, and the one exporter that ships | `[]`, `[io]`, `[net]` |
| `trid` | ids, the clock, and the tracer that starts spans | `[rand]`, `[time]` |
| `trlog` | the logging-nv bridge, both directions | `[]`, `[e]` |

## The load-bearing interface

```novo norun:pseudo
pub trait TracingExporter[e]
    fn export(self, spans: [TrSpan]) -> TrExportOutcome [e]
    fn flush(self) -> TrExportOutcome [e]
    fn shutdown(self) -> TrExportOutcome [e]
```

**The effect parameter is the argument.**  An exporter over HTTP costs
`[net]`, one over a unix socket `[io]`, one that writes a file `[fs]`,
and the in-memory one a test drains costs **nothing** — so
`trexport.flush_into`, the whole export pipeline, is written once and
charged exactly what the caller's own exporter costs.  An enum of
exporters would charge every caller the union, which means a test
asserting on the spans a library produced is charged `[net]` for a
collector it never reaches.

**Export takes a batch, and the signature admits nothing else.**  OTLP is
defined over a batch; a collector receiving one span per request is the
commonest way a traced service becomes slower than the service it traces.

**`TrExportOutcome` is not a `Result`.**  A partial send — a collector
that took 400 of 500 spans and then applied back-pressure — is the
ordinary outcome, and a `Result` has two shapes to put that in, both of
which lose the number a caller needs.

## Three decisions worth reading before the API

**There are no thread-locals.**  Every mainstream tracing library keeps
the current span in ambient storage.  That is a lie about effects — the
storage is `[mutate]`, charged to every call that starts a span — it is
wrong under concurrency, which is why every such library also has an
explicit-context escape hatch, and it makes a trace untestable.  Here the
context is a `TrContext` a caller threads, one parameter per boundary,
and the assertion a test writes is
`child.context.trace_id == parent.context.trace_id`.

**A malformed `traceparent` is not an error.**  W3C Trace Context
§ 3.2.2.3 says a receiver that cannot parse one must restart the trace
and must **not** reject the request.  So `trprop.extract` answers a
`TrPropagation` with a reason, never a `Result` — a `Result` at the edge
of a service invites `?`, and `?` turns a header a middlebox mangled into
a 500 the client cannot fix.  The reason is carried rather than
discarded, because "two hundred unparseable traceparents an hour" is a
fact an operator needs.

**The sampling decision is made once, at the root, and carried in the
flags.**  A sampler that ran per span would keep the child of a dropped
parent, and a backend receiving a span whose parent never arrived shows a
broken tree an operator cannot tell from a crashed service.
`trsample.trace_id_ratio` decides by hashing the trace id rather than
drawing at random, so every service running the same ratio agrees about
the same trace without coordinating — a per-service coin at ratio 0.1
over five services keeps one trace in a hundred thousand.

## The one example that will work

```novo
use trctx
use trexport
use trid
use trotlp
use trprop
use trsample
use trspan

fn handle(t: TrTracer, traceparent: Str, now: Float) -> TrSpan [rand]
    let ctx = trid.continue_trace(trprop.extract(traceparent, ""))
    let started = trid.start_span(t, ctx, "GET /users", TrServer, [], [], now)
    let tagged = trspan.with_attr(started.span,
                                  trspan.attr_int("http.status_code", 200))
    trspan.finish(tagged, now + 0.012)

fn main() [io]
    println("a server span, continuing whatever arrived")
```

## Adding it, and checking it

```console
$ novo pkg add tracing-nv
$ novo pkg build
$ novo test tests/trexport_tests.nv
```

The suites are **red on purpose**: every body is a `todo()`, so every
assertion reaches `not implemented: tracing-nv.<module>.<fn>`.

`novo pkg add` says `NOT IMPLEMENTED — interface only` on the way in,
because an interface resolves, downloads and builds exactly like an
implemented package and the difference only shows the first time
something calls it.

## The layer, and why

`host`, and five of the eight modules declare **nothing**.  The span, the
context, the W3C codec, the samplers and the OTLP encoding are all `[]` —
a trace is arithmetic over values the caller already holds, and that is
what lets a test assert on the exact document an exporter would send with
no collector in the room.

What performs is concentrated deliberately: `trid` reads the clock
(`[time]`) and the entropy source (`[rand]`), `trotlp.send` and the OTLP
exporter reach a collector (`[net]`), `trotlp.endpoint_from_env` reads
the environment (`[io]`), the in-memory exporter writes a caller's slot
(`[mutate]`), and everything generic over `TracingExporter[e]` costs what
the caller's exporter costs.

## What `std.otlp` keeps

- **`otlp.record_metric` keeps metrics entirely.**  This package is about
  traces; a metric is a different signal with a different document, and
  duplicating it would give a program two endpoints to configure.
- **`otlp.enabled` and `otlp.endpoint` keep the configuration
  convention** — `OTLP_ENDPOINT`, `OTLP_SERVICE`, and `off` to disable,
  read at call time.  `trotlp.endpoint_from_env` answers from the same
  variables rather than inventing a second spelling, so a program using
  both has one endpoint to configure.
- **`otlp.with_span` is deferred in the standard library and needs
  closure support — and that deferral is the gap this package closes.**
  A span here is a value, so `start` and `finish` are two ordinary calls
  and nothing is wrapped in a thunk.  A library built on `with_span`
  would inherit the deferral.

## JSON rather than protobuf

OTLP defines both over HTTP and every collector accepts JSON.  Protobuf
would want a `protobuf-nv` dependency and a generated schema this package
has no way to check against the specification.  The cost is size — a JSON
span is roughly three times its protobuf — and it is named here rather
than discovered from an egress bill.

Two encoding rules are functions for a reason: `trotlp.nanos_of` answers
a **string**, because an OTLP timestamp is a `fixed64` nanosecond count
that exceeds 2^53 for every date after 1970 and a JSON number loses
precision in any consumer with IEEE doubles; and `trotlp.kind_code`
exists because `SPAN_KIND_INTERNAL` is 1 rather than 0, so an encoder
using the enum's position writes every internal span as unspecified.

## What widened, and what did not

- **`trlog` costs a consumer logging-nv's whole closure.**  A service
  that wants spans and no device still assembles `deflog-decoder` and
  the four packages behind it, because that is logging-nv's dependency
  and not this one's.  It is the second argument for splitting
  logging-nv's `core` half out, and it is recorded in both READMEs.
- **The sibling dependency is a path here and a range at publish.**
  `novo pkg publish` refuses a path dependency, `--dry-run` included, so
  the release carries `logging-nv = "^0.0.1"`.
- **No device claim.**  `Str`, `Result` and the span's lists do not link
  at `@tier(embedded)`, and a trace id is 32 characters of text.  A
  device that wants to be traced sends its frames through the
  deferred-logging path and is correlated on the host.

## Reference

The reference implementations are the
[OpenTelemetry trace specification](https://opentelemetry.io/docs/specs/otel/trace/api/),
its [OTLP/HTTP protocol](https://opentelemetry.io/docs/specs/otlp/), and
[W3C Trace Context](https://www.w3.org/TR/trace-context/).  Rust's
`tracing` is the API this one deliberately diverges from, on the
thread-local question.

## Licence

Apache-2.0.
