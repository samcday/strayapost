# Design

These notes record rationale and a possible implementation approach.
The specs define the accepted contract; landing this document does not implement or verify the software.

## Context

See `proposal.md` for motivation and scope.
The repository currently contains a one-line README and the OpenSpec setup.
It has no Cargo manifest, Rust source, tests, or build workflow.

The earlier experiment retrieved tracking JSON with Rust `reqwest` and no account or solver.
That evidence covers one parcel, request shape, connection, and point in time.
It does not establish permanent upstream access.

The observed response used `consignmentId`, `articles`, and `milestone`.
Journey events were nested under `articles[].details[].events[]`.
The response's `status: "Success"` described the request, not the parcel.
Event timestamps included offset-bearing `localeDateTime` and epoch-millisecond `dateTime`.
A milestone timestamp differed from the latest event time.
There is no retained experiment implementation in this repository.

## Goals / Non-Goals

**Goals:**

- Keep acquisition and normalization usable without an operating system or a specific async runtime.
- Make the native CLI a caller of the same public library API available to other applications.
- Keep upstream wire data separate from stable public models.
- Make all normal tests deterministic and offline.

**Non-Goals:**

- Provide a browser transport or promise that browser cross-origin restrictions permit direct tracking.
- Guarantee that a browser-style HTTP header makes every network or parcel accessible.
- Infer missing delivery facts from plausible-looking event text.
- Introduce an extension framework for hypothetical future CAPTCHA providers.

## Decisions

### 1. One crate with a portable core and optional adapters

Use one Cargo package with `src/lib.rs` and a binary requiring the `cli` feature.
Use `#![cfg_attr(not(feature = "std"), no_std)]` and `extern crate alloc` in the library.

The feature relationships are:

| Feature | Responsibility |
| --- | --- |
| No default features | Models, parsing, discovery, lookup orchestration, transport contract, and clock contract using `core` and `alloc` |
| `std` | Standard-library integration without requiring an HTTP client or CLI |
| `native` | `std`, a `reqwest` HTTP adapter, and an operating-system clock |
| `cli` | `native`, argument parsing, terminal rendering, and a Tokio runtime |
| Default features | `cli`, so a normal native install provides the command |

Use Serde and `serde_json` with default features disabled and allocation support enabled.
Use Chrono with default features disabled and allocation support for offset-aware parsing and date handling.
Do not enable its operating-system clock features in the portable dependency graph.
Native time comes through the clock adapter.

Use optional `reqwest` with Rustls for native HTTPS.
Use optional Clap for argument parsing and a small CLI-only terminal rendering dependency.
Keep all runtime and terminal types out of the portable public API.

Alternative: separate library and CLI packages.
That adds packaging work before there is a second consumer and does not match the requested single-crate shape.
Alternative: make only the data structs portable.
That leaves consumers reimplementing discovery and tracking, so the whole lookup orchestration remains portable.

### 2. Inject HTTP transport and time

The public client accepts a transport and clock through generic parameters.
The transport exchanges small owned request and response types containing status, selected headers, and bounded body bytes.
Requests carry the phase-specific body limit and timeout policy.
The transport must enforce the body limit while reading; the core also validates the returned length.
Return a distinct response-too-large transport outcome, retaining safe HTTP metadata when available.
The core maps that outcome according to the request stage.
Never return a truncated body as a successful response.

Use an associated future type on the transport trait.
Do not impose `Send`, `Sync`, or `'static` on every caller.
The native adapter can return a boxed `Send` future while a future browser adapter can return a local future.
This avoids fixing a runtime or thread model in the core API.

The clock returns an absolute instant.
Capture `retrieved_at` when the successful tracking response has been received.
Do not read the clock separately while rendering relative ages.

A scripted in-memory transport will assert outgoing URLs and headers and return synthetic responses.
A fixed clock will make timestamps and relative output repeatable.
CLI tests will inject this client through the command runner, not add public test-only endpoint flags.

Alternative: call `reqwest` and `SystemTime` directly from the core.
That violates the portability requirement and makes offline testing harder.

### 3. Use the public tracking flow, without authentication

On a cold client, perform at most three GET requests:

1. Fetch `https://auspost.com.au/mypost/track/`.
2. Fetch the single matching module asset referenced by that page.
3. Fetch `https://digitalapi.auspost.com.au/shipments-gateway/v1/watchlist/shipments/{tracking-number}`.

Resolve asset references against the tracking page.
Accept only HTTPS assets on `auspost.com.au` under `/mypost/track/assets/`.
Reject user information, unexpected ports, ambiguous entry assets, and other origins.
Keep the gateway origin and route fixed.
Reject control characters before trimming surrounding whitespace.
Use the trimmed number for requests, identity matching, and output; reject empty values and the dot segments `.` and `..`.
Percent-encode it as one path segment, including any literal percent sign.
Do not decode caller input as an already encoded URL.

Extract only the public `shipmentsGateway` client-key value from the referenced asset.
Use a small, tested parser for the relevant HTML attributes and JavaScript string literal.
Require exactly one matching key-and-string definition; a `.shipmentsGateway` member access is not a definition.
Do not execute JavaScript or crawl imported assets.
An unsupported frontend layout produces a configuration error.

Reuse resolved configuration within a client instance.
Do not persist it or ship a real key in source.
Do not acquire bearer tokens, inspect browser state, or initialize the website's account-login flow.

The tracking request uses `api-key`, `AP_CHANNEL_NAME: WEB_DETAIL`, and `Accept: application/json`.
Start the native adapter with the header pair demonstrated by the experiment:

```text
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/150.0.0.0 Safari/537.36
Accept-Language: en-US,en;q=0.9
```

Keep that compatibility profile centralized and covered by request tests.
It is not an identity guarantee or a challenge bypass contract.
Do not add automatic user-agent, proxy, or address-family rotation.
Do not persist cookies.

Disable automatic redirects.
Inspect redirects for known challenge destinations, but never visit them.
Other redirects produce a phase-appropriate error rather than silently changing the trust boundary.

Use a 10-second connection timeout and a 25-second per-request timeout in the native adapter.
Bound decoded response bodies to 1 MiB for the page, 16 MiB for the asset, and 2 MiB for tracking.
Apply these limits while consuming bodies, not only after allocation.
Perform no automatic retry or forced configuration rediscovery after failure.
Explicitly disable the selected HTTP client's automatic retry policy.
Verify actual request counts in a controlled native-adapter test; a scripted core transport alone cannot establish this.
These values are proposed implementation defaults, not carrier-published limits.

Alternative: require users to copy a client key.
That makes the simplest command depend on manual website inspection.
Alternative: embed the current key or bundle filename.
That couples releases to public frontend changes.

### 4. Classify failures before parsing shipment data

Each request stage classifies recognized challenge redirects, HTML, or JSON descriptors before its expected payload is parsed.
Use specific DataDome markers and known challenge destinations, not any occurrence of the word `captcha`.
A generic HTTP 403 is access denial, not proof of a challenge.

The public error model and JSON codes cover:

| Condition | Code |
| --- | --- |
| Invalid CLI arguments | `invalid_arguments` |
| Invalid tracking input | `invalid_tracking_number` |
| Recognized CAPTCHA or device check | `challenge_required` |
| Denied access without challenge evidence | `access_denied` |
| HTTP 429 | `rate_limited` |
| Tracking gateway HTTP 400 | `not_found` |
| Tracking gateway HTTP 404 | `tracking_unavailable` |
| Missing or unsafe public configuration | `configuration_unavailable` |
| Malformed or inconsistent tracking response | `invalid_response` |
| DNS, connection, TLS, or request timeout failure | `transport_error` |
| Other unsuccessful HTTP response | `upstream_error` |

The gateway 400/404 distinction follows the public frontend's error handling, not a verified response-body schema.
A 400 message must allow for a number that has not entered the carrier's system yet.
A 404 message says tracking is unavailable, not that the parcel does not exist.
Discovery-resource 400/404 responses are configuration errors.
A body exceeding its phase limit maps to configuration or invalid-response failure, with a safe reason.
Preserve stage, HTTP status, and valid `Retry-After` information when applicable.
Use stage names `discovery_page`, `discovery_asset`, and `tracking`.
Accept both delta-seconds and HTTP-date forms of `Retry-After`.
Do not include raw bodies, client keys, cookie values, or token-bearing challenge URLs in errors or debug formatting.

The detector uses synthetic examples of CAPTCHA and device-check responses.
Challenge recognition does not create a solver interface or suggest that payment is required.

### 5. Normalize into a stable, conservative data model

Keep carrier deserialization types private.
The public result contains shipment identity, optional summary fields, shipment status, and all articles with their detail groups.
Each group retains its own status and complete event list.
Ignore unfamiliar optional wire fields but reject missing required shipment identity.
Match the canonical tracking number against `consignmentId`, any `articles[].articleId`, or the first `|`-delimited segment of `articles[].barcode`.
Use ASCII case-insensitive matching, while preserving the carrier's original identifier values.
Reject a response only when none of those supported identifiers matches.
An article lookup can legitimately return a different consignment identifier.

The source mappings established by the earlier experiment are:

| Source | Meaning |
| --- | --- |
| `consignmentId` | Returned shipment identity |
| `status` | API request outcome, never the displayed parcel state |
| Top-level `milestone.name` | Shipment status text |
| `articles[].articleId` | Article identity |
| `articles[].details[]` | Distinct detail groups |
| `articles[].details[].events[]` | Journey events |
| Event `description`, `location`, `eventCode` | Carrier event text, location, and code |
| Event `localeDateTime`, `dateTime` | Source timestamp representations |

Expose summary cities, shipment start, and ETA only through source mappings supported by upstream evidence.
A missing mapping produces `None`, not a guessed field name or a fabricated fixture contract.
The terminal uses `Unknown`; JSON uses `null`.
The public frontend provides no confirmed anonymous city or shipment-start mapping.
It uses sender and state or country for its From label, and hides the destination address from anonymous viewers.
That does not prove which fields the anonymous response contains.
Keep these unverified summary fields unavailable rather than adding an authenticated address flow or guessing a city.
Do not substitute first or last scan locations for endpoint cities.
Keep `first_recorded_at` separate from a carrier-reported `shipment_start`.

The public frontend supplies additional evidence from its detail view:

| Effective article/detail source | Meaning |
| --- | --- |
| Original `events[0].milestone`, then `milestones[0].name`, then `trackStatusOfArticle` | Detail-status precedence |
| Each event's `milestone` | Explicit status association for that event |
| `timeWindow.start`, `timeWindow.end`, `timeWindow.deliveryDate`, `timeWindow.passedWindow` | Delivery-window information |
| `estimatedDeliveryDateRange.fromDate`, `estimatedDeliveryDateRange.toDate` | Estimated delivery range |
| `deliveredByDateISO` | Deliver-by date |
| `estimatedDeliveryDateEligible` | Whether the UI requires a separate user-requested estimate calculation |
| `dateRevised`, `timeWindow.onTimeStatus` | Carrier estimate metadata |

The frontend merges each article with its first detail group, with detail fields taking precedence.
This library retains every group and applies that precedence independently within each group.
Resolve detail status from the original response order before sorting events for display.
Do not fabricate the frontend's `Pending` fallback when all actual status sources are absent.
Keep a top-level shipment status distinct from detail status, even when they disagree.

Represent ETA as a date, date range, local time window with optional date and zone, or unparsed carrier text.
Preserve source metadata and invalid optional values without claiming greater precision.
The frontend reads six-digit `HHmmss` window times and ISO date fields; parse those formats defensively.
An offset-less date-time is not an absolute instant.
Do not label an undated window as today using the caller's clock.

Apply the frontend's automatic-estimate eligibility first.
When `estimatedDeliveryDateEligible` is true, return no automatic ETA with a calculation-required reason.
Only expose active estimates for `Label created by sender`, `We've got it`, `It's on its way`, or `It's coming today`.
For an eligible active detail, use this priority:

1. A passed window with a supplied delivery date.
2. A complete start/end time window, retaining a supplied date but never inventing one.
3. A complete estimated date range.
4. A supplied deliver-by date.

This avoids the frontend's viewer-local today comparison when the carrier's zone is unknown.
If a selected estimate is malformed, report it as unavailable rather than fall back to a potentially older estimate.
Preserve revision and passed-window metadata; do not calculate lateness independently.
Do not call the separate postcode/suburb estimate service.
A single-article, single-detail estimate can populate the shipment summary.
With multiple articles or groups, keep estimates scoped to those groups and leave the shipment ETA unknown.

Preserve unfamiliar carrier status labels rather than make every new label a deserialization error.
Never promote one delivered article to a delivered multi-article shipment.

For each event, retain the raw local-time value and raw epoch value, as well as a normalized instant and timestamp issue.
Normalize a single valid representation when the other is absent.
When both are present, require them to identify the same instant.
Malformed or conflicting supplied representations leave the normalized instant unavailable.
Preserve original source values even when their wire type is unexpected.

Sort dated events by instant, retaining original order for equal instants.
Append undated events in source order.
Do not deduplicate repeated descriptions or event codes.
For a flattened CLI table, use article order, detail-group order, and event order as the stable tie-breaker.
Only validated instants contribute to `first_recorded_at` and `last_event_at`.

The source inspection used the public [tracking page](https://auspost.com.au/mypost/track/) and its referenced [application asset](https://auspost.com.au/mypost/track/assets/index-b1aea089.js).
The asset name is evidence for this plan, not a filename to hardcode.
No parcel was queried during that inspection.
The source confirms frontend behavior, not every field's presence or type in anonymous responses.

### 6. Derive status age only from evidence

Keep current status, current-status start, latest event, and retrieval time as separate facts.
Do not treat the observed milestone timestamp as a verified transition time without evidence of its meaning.

Use an explicitly verified transition timestamp when available.
Otherwise derive a recorded status transition only from events with explicit status associations and valid instants.
Find entry into the final uninterrupted run of the current status, preceded by a known different status.
Unknown associations, conflicting times, or ambiguous ordering prevent that derivation.
The source order must also agree with newest-first instant ordering; sorting must not conceal a contradictory history.
A later event repeating the same status does not restart the run.

Only a single-article, single-detail shipment can use that history for shipment status age.
Shipment and detail current statuses must both be known and equal; any supplied article status must also agree.
Do not select one article or combine separate detail histories to establish a shipment-level transition.
Without sufficient evidence, return no status-change time.
The CLI still shows the current status and an unknown age.

This conservative rule avoids presenting the newest scan as the time the status first changed.
It also avoids confusing shipping information received with physical pickup.

### 7. Keep rendering and machine output separate

The CLI uses one normalized result for both renderers.
Its runner accepts parsed options, an injected library client, output writers, and terminal capabilities.
The binary provides native implementations of those dependencies.

Human output uses labeled summary fields followed by an oldest-first event table.
A multi-article table includes article and detail-group identity, with their statuses shown separately.
Show timestamps with numeric UTC offsets, and label retrieval time explicitly.
Wrap long carrier descriptions instead of dropping events.
Escape untrusted terminal controls before applying the renderer's own color.

Enable color only for an interactive destination when `TERM` is not `dumb` and `NO_COLOR` is absent or empty.
Keep color decisions separate for stdout and stderr.
Do not print progress or banners to stdout.

JSON uses a versioned output model rather than raw carrier JSON.
Success has `schema_version`, `ok`, `tracking_number`, `retrieved_at`, and `shipment`.
Failure has `schema_version`, `ok`, `tracking_number`, and `error`.
The error object contains `code`, `message`, `stage`, `http_status`, and `retry_after`, using `null` for unavailable values.
The shipment model includes summary fields, status-change time, first and last event times, and nested articles and events.
Use RFC 3339 strings for normalized instants, ISO dates for date-only values, and explicit variants for ETA precision.
Retain source timestamp values and issues beside normalized event times.

Document and test the complete version-1 JSON shape before release.
Use no display-only relative phrases in JSON.
Finish every JSON result with one newline.

Inspect the raw argument list for `--json` before normal parsing, respecting the `--` delimiter.
Use fallible argument parsing so parser failures can use the JSON error renderer.
Explicit help and version requests retain normal text output.
Library invalid-input failures map to exit 2; lookup failures map to exit 1.

Alternative: serialize the raw upstream response.
That leaks unstable fields and does not provide the same interpretation as terminal output.
Alternative: let argument parsing print errors directly.
That breaks the machine-readable contract for malformed JSON-mode invocations.

### 8. Validate portability and behavior without live parcels

Use synthetic fixtures for discovery assets, gateway responses, and challenge pages.
Include mixed offsets, duplicate descriptions, unknown statuses, multiple articles, missing details, timestamp conflicts, and malformed required data.
Request scripts must also assert that failure paths stop instead of making additional requests.

Check native tests and CLI snapshots with fixed clocks and terminal capabilities.
Exercise argument errors, stdout/stderr separation, exit codes, color suppression, and untrusted terminal text.
Include a non-`Send` transport caller to prove that the public portable API is actually usable.

Run these checks during implementation:

```text
cargo fmt --check
cargo test --all-features
cargo test --lib --no-default-features
cargo clippy --all-targets --all-features -- -D warnings
cargo check --lib --no-default-features
cargo check --lib --no-default-features --features std
cargo check --lib --no-default-features --features native
cargo check --lib --no-default-features --target wasm32-unknown-unknown
cargo check --lib --no-default-features --target thumbv7em-none-eabi
```

The embedded target is a compile-only guard against accidentally enabling `std` in a dependency.
It does not add embedded network support to the product.
Install target standard-library artifacts only in the implementation environment that needs those checks.

Live verification is separate from the normal suite.
If the user authorizes a smoke test, use only their supplied tracking number and keep its response out of committed fixtures.
Record whether the real command succeeds or returns a classified failure.
Do not turn an upstream challenge into permission to introduce a solver.

## Risks / Trade-offs

- Unofficial frontend changes -> Keep discovery isolated, reject unsupported layouts, and use bounded requests.
- Requests work on one connection but fail elsewhere -> Report challenge and denial honestly; do not promise uninterrupted access.
- Missing origin, destination, start, ETA, or transition evidence -> Show unknown values rather than invent facts.
- A library dependency enables `std` indirectly -> Check the feature graph and compile against a target without `std`.
- Browser builds compile but cannot access the endpoint -> Defer browser networking and document cross-origin restrictions.
- Default features include the CLI -> Document `default-features = false` prominently for portable library consumers.
- Strict timestamp checks hide a usable-looking date -> Retain source values and expose the issue instead of silently choosing one.
- Three bounded cold-start requests add latency -> Reuse public configuration within one client; do not introduce persistent state yet.

## Migration Plan

There is no deployed implementation or data format to migrate.
This landing promotes the delta specs and archives these notes.
Runtime implementation and its validation remain separate work.
Before a software release, review the public API and JSON examples and run the implementation checks described above.
A later contract correction needs another reviewed spec change; there is no persistent tracking database to convert.
