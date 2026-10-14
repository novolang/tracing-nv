# Changelog

All notable changes to tracing-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-11

The package is implemented.  Every body behind the 0.0.4 interface is
written — the span, the context, the W3C headers, the samplers, the
batch, the OTLP/JSON encoding and the exporter over it, the two
effectful identifier calls, and the log join in both directions — and
the published surface is 0.0.4's: the 144 `pub` declarations the
interface release published are the same lines under the same modules,
with one addition, `trexport.clear_slot` (see Added), so a program
written against the interface release compiles against this one with
nothing to change.

- **Stability moves from `draft` to `experimental`.**  Draft is the mark
  of a release whose bodies are `todo()`; this one has none, and what
  stays provisional is the surface rather than the code.
- **Toolchain floor moves to `>= 0.20.2`**, the version the bodies are
  verified against.  The interface release's floor covered signatures
  and says nothing about the code behind them.
- **124 tests, all green in both modes and no leak lines.**  `novo test
  tests` runs all nine suites in one process — 9 of 9 green — and
  `novo test --isolate` runs each suite against fresh module state with
  the same result.  The eight suites the interface release published —
  the oracle this implementation was fitted to — carry 86 of them;
  seven took no edit beyond deleting the "RED ON PURPOSE" release-stamp
  sentence from the header (the `redundant-bool` rewrite under
  `### Changed` is the only other edit any suite took), and
  `tests/trid_tests.nv` also had one header paragraph the bodies had
  made false rewritten (a seed does NOT make `start_span` predictable;
  the gap is a Known bullet below).  `tests/trexport_tests.nv` has one
  added line, the `clear_slot` call under `### Changed`.  The ninth,
  `tests/tracing_nv_edge_tests.nv`, is this package's own: 38 tests
  over the edges the specification suites leave standing, including the
  socket cases against a loopback listener.
- **Every instrumented line under `src/` is covered or excused.**  697
  of the 728 lines ran over the nine suites; the other 31 are excused
  by eleven `// cov: skip` markers, each naming its reason — ten in
  `trotlp` (the reply-side parsing, which needs a collector that
  ANSWERS, and which the loopback listener cannot supply from one
  thread) and one in `trprop` (`hex_value`'s `-1` arm, which its one
  caller cannot reach because it runs only after `is_hex_run` has
  accepted the same bytes).  Two of the `trotlp` ten are on the one
  write-failure path, at the call and on the helper it calls, because a
  marker excuses only lines indented deeper than the line it names, and
  the close and the return that used to sit side by side in that branch
  could not share one.  The measurement is this package's own, run the
  way publish would run it, for the reason the dependency bullet gives.
- **logging-nv is a path line for the duration of the work.**  The
  registry's newest logging-nv is 0.0.4 — an interface release whose
  every call panics — and 0.1.0 is implemented in the directory next to
  this one, so the manifest names the directory.  The consequence is a
  fact about this release and is stated rather than worked around:
  `novo pkg publish --dry-run` refuses at that line (`novo.toml:63`),
  so this package's five shard rows and its coverage line have not been
  measured.  The line becomes a registry range when logging-nv ships.

### Added

- `trotlp`'s encoder, as a pure function of the spans it is handed.  It
  writes the specification's three levels — `resourceSpans`, then
  `scopeSpans`, then `spans` — and every attribute value through the
  `AnyValue` wrapper its type calls for, arrays included: OTLP's
  `arrayValue` holds `values` of `AnyValue`, so a bare JSON scalar in
  that position is a document a collector cannot decode (see Fixed).
  Timestamps are strings, because a nanosecond count exceeds 2^53 for
  every date after 1970, and `SPAN_KIND_INTERNAL` is 1 rather than an
  enum position.
- The effectful half, kept to the calls that have no value form.
  `trid.now_unix` is the clock (`[time]`) and the identifier calls —
  `new_trace_id`, `new_span_id`, `root_context`, `child_context` and
  `start_span` — are `[rand]`; `trotlp.send` and the socket calls under
  it are `[net]`, the two environment reads are `[io]`, and the
  in-memory exporter is `[mutate]`, which is the effect the trait's
  parameter exists for.  The batch, the sampler, the codec and the
  encoder are `[]`, which is what makes `tests/trotlp_tests.nv` able to
  assert on the exact bytes a collector would receive with no collector
  in the room.
- The parent-based samplers consult the PARENT's sampled bit and do not
  re-roll: a span with a parent inherits the decision instead of asking,
  so a dropped root's children stay dropped.  `trid.start_span` answers
  a span and a context even on a `TrDrop`, because a downstream service
  sampling on its own still needs this span's id as its parent.
- `trexport.memory_exporter`, writing into a module-level table of slots
  the caller names.  A `TrMemoryExporter` is a value and the trait takes
  `self` by value, so a struct holding its spans would hand them back as
  a new value on every call and a test could never read what a pipeline
  it did not write the return value of had sent.  The reading call
  carries `[mutate]`, which is what the trait's effect parameter is for.
- `trexport.clear_slot`, which empties one of those slots.  Added at
  landing, and the one addition to the published surface the interface
  release left: a slot table a caller can write and read but never
  empty is a gap for a consumer — a process starting a new export run
  over storage an earlier one used — and it is a gap for the suite too,
  because `tests/trexport_tests.nv` asserts on what a FRESH exporter
  received while an earlier test in the same process exported through
  the same slot.  A negative slot is ignored, as it is in
  `memory_exporter`, and no existing declaration, field or effect row
  moved.
- `tests/tracing_nv_edge_tests.nv`, 38 tests: a negative slot, the
  clear of a slot, a ratio of `2.0` and of `-1.0`, an integer under
  `trace_id`, a sink that refuses, a 513-span batch that is refused as
  too large, an endpoint with a trailing slash, a collector that never
  answers, every `TrAttrValue` wrapper and every `LgValue` arm, the
  JSON escaping, and all seven `TrPropReason` messages.

### Changed

- **Forty-four `== false` comparisons are spelled `not`.**  The 0.20.2
  linter raises `redundant-bool` on each and suggests exactly that
  rewrite, which is the same predicate.  Twenty-eight are pre-existing —
  `trctx_tests` 7, `trexport_tests` 7, `trid_tests` 4, `trprop_tests` 3,
  `trsample_tests` 3, `trspan_tests` 2, `trlog_tests` 1, `trotlp_tests`
  1, the same counts at the same lines at `HEAD` — and the other
  sixteen are in the new edge suite.  No assertion's meaning changed and
  every `.nv` file in this package now lints with no warnings.
- **One line added to `tests/trexport_tests.nv`.**  Its "what a fresh
  exporter received" case clears slot 0 before constructing the exporter
  it is about, because the file's own pipeline test exports through
  slot 0 in the same process.  The call is `trexport.clear_slot(0)`, it
  sits under a comment giving that reason, and every assertion in the
  suite is the release's own, unchanged.  Recorded here rather than
  passed off as one of the oracle's two allowed edits, which are the
  release stamp and a sentence the bodies made false.

### Fixed

- **An array attribute is encoded as an array of `AnyValue`, not of
  bare JSON scalars.**  The first version wrote `"values":[2,3]` where
  OTLP/JSON requires `"values":[{"intValue":"2"}]`; the document is
  valid JSON, a collector answers 200, and the attribute is empty in the
  backend.  The edge suite's
  `test_every_attribute_value_type_has_its_otlp_wrapper` caught it, and
  each element now goes through the same `encode_value` the scalar case
  uses, so the two shapes cannot drift apart.
- **`trlog` reads the correlation fields by matching their type.**
  `trace_id_of` answers `None` for a field that is present under
  `trace_id` with another type, rather than crossing it through a
  string-answering helper: the helper's empty string for an integer is
  indistinguishable from a field that IS text and empty, so a record
  carrying `{"trace_id": 7}` would have read as correlated and joined to
  nothing — the failure the module's own header says costs an operator
  more time than a line that plainly has no trace on it.

### Known

- **The in-memory exporter's slot table is module-level, and a reader in
  one process sees what another wrote.**  A `TrMemoryExporter` is a
  value and the trait takes `self` by value, so a struct holding its
  spans would hand them back as a new value on every call and a test
  could never read what a pipeline it did not write the return value of
  had sent.  The consequence is in the API rather than hidden:
  `trexport.clear_slot` empties a slot, which is what a reader that
  wants fresh storage calls first, so no assertion here is written
  against state the test did not make.  Both modes pass — `novo test
  tests` in one process and `--isolate` per test — and coverage is still
  measured the non-isolated way, because `--cov` cannot be combined with
  `--isolate` (the isolating runner branches per test, and the sidecars
  assume a runner without branch sites, which is the refusal `novo test`
  prints).
- **The slot table's name is `export_slots`, and the prefix is a
  workaround.**  A module-level `var` is keyed by its bare name across
  every module of the assembly, so `var slots` here would BE
  logging-nv's `lgring.slots` — a `[LgRing]` — and every read would be
  typed against the dependency
  (`language/two-modules-module-level-vars-of-one-name-are-one-variable`,
  open, 0.20.2).  The prefix comes off when that filing closes; the
  reason and the filing's path are at the declaration.
- **`trid.seeded_tracer` accepts a seed and drops it.**  A `TrTracer`
  carries a sampler and a scope and no id source, and `start_span` is
  `[rand]` and cannot keep a counter of its own — nor could one reachable
  from that call give two children of one parent different ids, since
  they present it with the same tracer, the same context and no other
  state.  A trace whose siblings share a span id is worse than a trace
  with unpredictable ones.  `trid.seeded_ids` is the half of the promise
  that can be kept and is published for it; a test asserting on the
  spans themselves asserts structurally — a child's trace id is its
  parent's, a child's span id is not.
- **`trotlp.this_scope()` writes the version twice.**  The scope's
  version is this package's release number as a literal, so a bump edits
  `novo.toml` and `trotlp.nv`; the comment at the declaration says so.
  A scope whose version drifts from the released one is wrong in a
  collector's UI rather than in a test, which is why it is written down
  rather than left to be remembered.
- **`trsample.sampler` takes a plain `fn`, so a caller's own sampler
  cannot close over configuration**, while three of the six shipped
  samplers are lambdas that do exactly that.  A policy like
  `dropping_names` — which needs the list it drops — is therefore
  published rather than left to the caller: the caller-facing form can
  express a decision that is a function of its parameters and nothing
  else.  Lifting it means a function-typed field that carries state,
  which `@tier(embedded)` refuses, and this package claims no device.
- **No device claim.**  A trace id is 32 characters of text and a span
  carries three lists; neither links at `@tier(embedded)`, and the
  three configured samplers are lambdas.  A device that wants to be
  traced sends its frames through the deferred-logging path and is
  correlated on the host.
- **`trlog` costs a consumer logging-nv's whole closure**, which today
  is ansi-nv plus the five packages the device bridge brings.  That is
  a real price on a service that wanted spans and no device, and it is
  the second argument for splitting logging-nv's `core` half out, which
  logging-core-nv 0.0.1 now is.

## 0.0.4 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- logging-nv: `^0.0.1` to `^0.0.4`.

## 0.0.3 — 2026-09-25

The package builds with novo 0.11.  Every body is still `todo()`.

- The lock file moves leb128-nv 0.1.5 to 0.1.7 and zigzag-nv 0.1.4 to
  0.1.5.  leb128-nv 0.1.5 writes into lists through names that are not
  declared `var`, which novo 0.11 refuses (E2038), so this package did
  not build with novo 0.11 against it.  No requirement in the manifest
  changed.

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
