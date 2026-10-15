# Implementation Notes

This checklist is uncompleted reference material for a separate implementation effort.
It is not a completion gate for accepting or landing the specifications.
The original items remain unchecked because no implementation has been completed.

## 1. Establish the crate and feature boundaries

- [ ] 1.1 Create the `strayapost` Cargo package, library, and feature-gated binary. Verify that default features build the command and disabling them excludes it.
- [ ] 1.2 Add the `std`, `native`, and `cli` feature relationships and allocation-only parsing dependencies. Verify host, `wasm32-unknown-unknown`, and `thumbv7em-none-eabi` library checks with default features disabled.
- [ ] 1.3 Document the feature matrix, local installation, and unofficial status in the README and crate documentation. Verify documented feature selections with Cargo and ensure no solver or browser support is claimed.

## 2. Define the portable public API

- [ ] 2.1 Add typed shipment, article, detail-group, event, ETA, timestamp-issue, and error models. Verify tests cover absent values, unknown statuses, source timestamp retention, and safe error formatting.
- [ ] 2.2 Add caller-provided asynchronous transport and clock contracts, plus the library client. Verify a public-API example works with a non-`Send` in-memory transport and fixed clock.
- [ ] 2.3 Define a distinct response-too-large transport outcome with safe HTTP metadata. Verify oversized bodies never succeed as truncated responses and map to the correct stage-specific error.
- [ ] 2.4 Add a scripted transport test helper that asserts requests and fails on unexpected extra requests. Verify the helper catches changed URLs, headers, and request counts.
- [ ] 2.5 Document the transport obligations, runtime independence, and retrieval-time semantics. Verify library examples compile without relying on CLI modules.

## 3. Implement direct acquisition and failure handling

- [ ] 3.1 Add canonical input validation and path-segment encoding. Verify spaces are trimmed, controls and dot segments are rejected, and `/`, `?`, `#`, and literal `%2e` cannot alter routing.
- [ ] 3.2 Implement bounded public-page and module-asset discovery, with in-client configuration reuse. Verify changed filenames, permitted URLs, ambiguous assets, missing keys, malformed literals, and rejected origins using synthetic fixtures.
- [ ] 3.3 Implement the anonymous gateway request and centralized compatibility headers. Verify a cold lookup makes at most three expected GETs and a warm client reuses its configuration without account authentication.
- [ ] 3.4 Classify challenges, access denial, rate limits, configuration errors, invalid responses, and upstream failures. Verify gateway 400/404 meanings differ from discovery failures and every failure stops further requests.
- [ ] 3.5 Implement optional native HTTP and clock adapters with redirects and automatic retries disabled, timeouts, and streaming body limits. Verify request counts and bounds using controlled loopback tests, then check `native` without `cli`.
- [ ] 3.6 Document direct acquisition, error categories, request limits, and safe diagnostics. Verify fixture tests cover both forms of `Retry-After` and exclude keys, cookies, and challenge tokens from formatted errors.

## 4. Normalize shipment data without inventing facts

- [ ] 4.1 Map verified fields into every shipment, article, and detail group. Verify consignment, article, and barcode aliases match case-insensitively, unrelated identities fail, and unverified cities and shipment start remain unknown.
- [ ] 4.2 Implement offset-aware event timestamp validation. Verify agreeing, missing, malformed, wrong-type, and conflicting representations retain their source values and expected issues.
- [ ] 4.3 Implement the documented ETA eligibility, precedence, and article scope. Verify dates, ranges, undated time windows, calculation-required responses, invalid selected values, and supplied revision metadata.
- [ ] 4.4 Preserve and stably order every event by instant, with undated events last. Verify mixed offsets, equal instants, repeated delivery scans, and flattened multi-article ordering.
- [ ] 4.5 Derive first-recorded and latest-event times separately from shipment start and status-change time. Verify repeated statuses preserve age and multiple groups, missing or conflicting statuses, or contradictory ordering prevent history-derived shipment age.
- [ ] 4.6 Document source mappings and missing-data rules beside the parser and in public model documentation. Verify examples distinguish request success, shipment status, article status, scan time, and retrieval time.

## 5. Build terminal and JSON output on the library

- [ ] 5.1 Implement the single-number argument interface and injectable command runner. Verify help, version, invalid input, unsupported options, and missing or extra numbers make no lookup requests.
- [ ] 5.2 Add the summary and chronological journey table. Verify snapshots cover complete data, unknown fields, first-recorded fallback labeling, multiple articles, no events, and status ages from a fixed retrieval time.
- [ ] 5.3 Add automatic color selection and safe terminal rendering. Verify interactive output, redirected output, `NO_COLOR`, `TERM=dumb`, long descriptions, and embedded terminal controls.
- [ ] 5.4 Add the version-1 JSON success and error models and exit-code mapping. Verify every output parses as one newline-terminated object, errors use stable codes, null values remain null, and stdout contains no diagnostics.
- [ ] 5.5 Route argument-parser failures through the selected renderer, respecting `--` and help or version requests. Verify `--json` produces structured argument errors with exit 2 and detected challenges produce exit 1.
- [ ] 5.6 Document command examples and the complete JSON shape using synthetic shipment data. Verify examples against renderer tests and clearly state that unknown values are not evidence of a missing shipment.

## 6. Verify the complete release candidate

- [ ] 6.1 Exercise the full command runner from arguments through discovery, tracking, normalization, and output with scripted transport responses. Verify both output modes and all failure stages without live carrier requests.
- [ ] 6.2 Run `cargo fmt --check`, `cargo test --all-features`, `cargo test --lib --no-default-features`, and `cargo clippy --all-targets --all-features -- -D warnings`. Verify all pass and record any environment limitation.
- [ ] 6.3 Add CI coverage for the host, `std`-only, `native`-only, Wasm, and no-standard-library feature checks listed in the design. Verify each exact command locally where targets are available and keep live lookups out of CI.
- [ ] 6.4 Inspect the package contents and dependency feature graph. Verify no credentials, real client keys, live parcel histories, or raw browser captures are packaged, and no native or CLI dependency enters the allocation-only build.
