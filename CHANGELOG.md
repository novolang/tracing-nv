# Changelog

All notable changes to tracing-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `trspan` — a span as a finished value rather than an open handle, with
  typed attributes, events, links, the three-state status, and
  `record_exception` writing the four keys the specification fixes.
- `trctx` — `TrSpanContext` (the four fields `traceparent` carries) and
  `TrContext` (the value a caller threads), with baggage and the
  sampled-bit accessors.
- `trprop` — `traceparent`, `tracestate` and `baggage`, with a
  `TrPropReason` per way a header goes wrong and a `restart` flag rather
  than a `Result`.
- `trsample` — `always_on`, `always_off`, `trace_id_ratio`,
  `parent_based`, `parent_based_with_root`, `dropping_names`, and the
  escape hatch that takes a caller's own named function.
- `trexport` — `TracingExporter[e]`, `TrExportOutcome`, the batch value
  with the specification's three bounds, and the in-memory exporter a
  test drains.
- `trotlp` — the OTLP/JSON encoding as a pure function, the `[net]`
  exporter over it, and the environment convention `std.otlp` already
  set.
- `trid` — the two effectful rows in the package, the tracer, and the
  seeded tracer whose ids a test can predict.
- `trlog` — the correlation fields on a log record, and a span event as
  a log record going back the other way.

### Known

- **`TracingExporter[e]` is the load-bearing interface**: one pipeline,
  charged what the caller's own exporter costs, where an enum would
  charge every caller the union.
- **No thread-locals**, on purpose, and the README says what that costs
  and why every library with ambient context also ships the explicit
  form.
- **A malformed `traceparent` restarts the trace** rather than failing
  the request, which is what W3C Trace Context § 3.2.2.3 requires.
- **The sampling decision is made once, at the root, and inherited**,
  because a per-span sampler produces trees a backend cannot
  distinguish from a crashed service.
- **`trlog` costs a consumer logging-nv's whole closure**, which is the
  second argument for splitting logging-nv's `core` half out.
- **No device claim.** A trace id is 32 characters of text and the span
  carries three lists; neither links at `@tier(embedded)`.
- **One dependency**, logging-nv, by path here and by range at publish.

### Design notes

- `TrSampler` is a struct with a function-typed field rather than a
  trait or a lambda.  A trait's bounds carry an effect argument and
  never a type one, and a lambda that becomes a value is refused at
  `@tier(embedded)`.  The field is filled with a named function, which
  is the arrangement proptest-core-nv and matchers-nv arrived at for the
  same two reasons.
- JSON rather than protobuf.  OTLP defines both over HTTP and every
  collector accepts JSON.  Protobuf would want a `protobuf-nv`
  dependency and a generated schema this package has no way to check
  against the specification.  The cost is size: a JSON span is roughly
  three times its protobuf, and the README names that rather than
  leaving it to be discovered from an egress bill.
- `trlog` costs a consumer logging-nv's whole closure.  A service that
  wants spans and no device still assembles deflog-decoder and the four
  packages behind it.  That is the second argument for splitting
  logging-nv's `core` half out, which logging-core-nv 0.0.1 now is.
