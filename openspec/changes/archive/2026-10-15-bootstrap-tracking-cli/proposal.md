# Proposal

This change publishes the initial tracking contract, not a Rust implementation.
The behavior below describes what an implementation must provide.

## Why

Australia Post tracking needs a simple command and a reusable library that do not require a personal account.
The earlier direct-HTTP experiment worked, but this repository has no packaged implementation or public API.

## What Changes

- Introduce one Rust crate named `strayapost`, with a library and a CLI built on that library.
- Make the library usable with `no_std + alloc` when default features are disabled.
  Keep native networking, clocks, and terminal handling outside the portable core.
- Discover the public website client configuration and request tracking data directly.
  Return a distinct error when Australia Post presents a challenge; do not integrate a CAPTCHA solver.
- Expose shipment details, article-level journey events, source timestamps, and retrieval time through a typed library API.
- Support `strayapost <tracking-number>` with a colorized terminal summary followed by a journey table.
  Include From/To cities, shipment start, ETA, current status, and time since the last evidenced status change.
- Support `strayapost --json <tracking-number>` with structured success and error output suitable for scripts.
- Show unavailable details explicitly rather than infer addresses, delivery promises, or status transitions from unrelated scans.

This change excludes persistent caching, background watching, bulk tracking, a server, account login, solver integrations, and a browser runtime adapter.
The portable library will compile for `wasm32-unknown-unknown`; working browser networking is later work.

## Capabilities

### New Capabilities

- `shipment-tracking`: Portable library access to anonymous Australia Post tracking, including configuration discovery, shipment data, event normalization, and typed failures.
- `tracking-cli`: Single-shipment terminal and JSON output, with explicit output, color, and exit-status behavior.

### Modified Capabilities

None. The project has no existing capability specs.

## Impact

- Publish the accepted tracking-library and CLI specs, with archived rationale and unfinished implementation guidance.
- A separate implementation effort will add the Cargo package, library modules, native transport, CLI, fixtures, tests, and build checks.
- Add allocation-only parsing dependencies to the portable library.
  Keep HTTP, TLS, async runtime, argument parsing, and terminal dependencies behind native or CLI features.
- Use Australia Post's public website configuration and its unofficial shipments gateway.
  That upstream interface can change or reject a request.
- Keep tracking numbers and shipment data local, apart from the lookup sent to Australia Post.
  Do not commit live client keys, account credentials, raw browser captures, or real parcel histories.
- Document installation, library feature selection, output semantics, and the unofficial nature of this tool.
