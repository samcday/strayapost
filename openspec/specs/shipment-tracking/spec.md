# shipment-tracking Specification

## Purpose

Provide portable, account-free access to Australia Post shipment details and journey events.
Expose source data and lookup failures without confusing missing information with verified facts.

## Requirements

### Requirement: Allocation-only library support

The crate MUST provide a library that uses `core` and `alloc` without requiring `std` when default features are disabled.
Its public tracking API MUST remain usable in that build.
Native networking and CLI dependencies MUST be optional.

#### Scenario: Build the portable library
- **WHEN** the library is checked with default features disabled on the host and `wasm32-unknown-unknown`
- **THEN** both checks succeed without enabling native networking or CLI features

#### Scenario: Check against a target without the standard library
- **WHEN** the library is checked with default features disabled on a supported compile-only target without `std`
- **THEN** the check succeeds without a runtime, operating system, or global allocator implementation
- **AND** this check does not claim a working network adapter for that target

### Requirement: Caller-provided platform services

The portable tracking API MUST accept caller-provided asynchronous HTTP transport and time sources.
It MUST NOT require a particular executor, filesystem, environment variables, terminal, or operating-system clock.
It MUST NOT require transport futures to implement `Send`.

#### Scenario: Use a platform-independent caller
- **WHEN** a caller supplies an in-memory HTTP transport and a fixed clock
- **THEN** the caller can complete configuration discovery and a tracking lookup through the public library API
- **AND** the result uses the supplied clock for retrieval time

### Requirement: Public client configuration discovery

The library MUST obtain the public tracking client configuration without a personal account, bearer token, or caller-supplied key.
Discovery MUST use bounded requests to approved Australia Post resources.
It MUST NOT execute downloaded scripts, read browser profiles, or follow arbitrary origins.

#### Scenario: The frontend asset changes
- **WHEN** the public tracking page references a new application asset containing the public gateway configuration
- **THEN** discovery uses that asset rather than a hardcoded asset filename or client key

#### Scenario: Configuration cannot be resolved safely
- **WHEN** the configuration is absent, ambiguous, oversized, or points outside the approved resources
- **THEN** discovery returns a configuration error without making an unapproved request

### Requirement: Canonical tracking input

The library MUST reject control characters before trimming surrounding whitespace.
It MUST reject a trimmed value that is empty, `.` or `..`.
Otherwise, that trimmed value MUST be the canonical number used for lookup, response matching, and output.
Validation MUST finish before any request.

#### Scenario: The input has surrounding spaces
- **WHEN** the caller supplies ` EXAMPLE123 `
- **THEN** lookup, response matching, and output use `EXAMPLE123`

#### Scenario: A path-normalizing value is supplied
- **WHEN** the trimmed input is `.` or `..`
- **THEN** the library returns an invalid-input error without making requests

#### Scenario: Controls or empty input are supplied
- **WHEN** input contains a control character, including a surrounding newline, or is empty after trimming
- **THEN** the library returns an invalid-input error without making requests

### Requirement: Direct lookup of the supplied tracking number

The library MUST look up the canonical tracking number through the public shipments gateway without account login.
It MUST encode the number as data rather than URL structure.
It MUST NOT enumerate other tracking numbers.

#### Scenario: Anonymous lookup succeeds
- **WHEN** the gateway returns valid tracking data for the supplied number
- **THEN** the library returns a typed shipment result without requesting a login or CAPTCHA service

#### Scenario: The input contains URL syntax
- **WHEN** a valid tracking number contains `/`, `?`, `#`, or literal `%2e`
- **THEN** those characters cannot change the gateway host, path structure, or query
- **AND** percent signs are encoded rather than interpreted as pre-encoded input

### Requirement: Explicit challenge detection

The library MUST return a distinct challenge-required error when a recognized CAPTCHA or device-check response blocks discovery or tracking.
It MUST NOT solve the challenge, open a browser, submit a solver task, or retry around the challenge.
The error MUST identify the blocked lookup stage without exposing cookies or challenge tokens.

#### Scenario: A device check replaces tracking JSON
- **WHEN** a recognized device-check page or challenge descriptor arrives instead of shipment data
- **THEN** the library returns a challenge-required error, including when the HTTP status is 200
- **AND** it makes no further lookup requests

#### Scenario: Configuration discovery is challenged
- **WHEN** a recognized challenge blocks the tracking page or application asset
- **THEN** the library returns a challenge-required error for discovery
- **AND** it does not misreport a missing configuration key

#### Scenario: Access is denied without challenge evidence
- **WHEN** the server returns HTTP 403 without a recognized challenge
- **THEN** the library returns an access-denied error rather than claiming a CAPTCHA was detected

### Requirement: Bounded requests and distinct failures

The lookup MUST bound request count and response size.
The native transport MUST bound connection and request duration.
Failures MUST distinguish transport errors, rate limits, access denial, not-found responses, unavailable tracking, configuration errors, and invalid responses.
The first release MUST NOT retry failures automatically.

#### Scenario: The carrier limits requests
- **WHEN** the carrier returns HTTP 429 without challenge evidence
- **THEN** the library returns a rate-limit error and any valid retry guidance
- **AND** it does not sleep and retry

#### Scenario: A request fails or exceeds a bound
- **WHEN** a request times out, exceeds its response-size bound, or produces malformed data
- **THEN** the lookup ends with an error identifying the failure category
- **AND** it does not return a successful empty shipment

#### Scenario: The gateway cannot find the tracking number
- **WHEN** the tracking gateway returns HTTP 400 without challenge evidence
- **THEN** the library returns a not-found error
- **AND** the error allows for a number that is not yet in the carrier's system
- **AND** it does not classify an arbitrary HTTP or parsing failure as not found

#### Scenario: The gateway reports unavailable tracking
- **WHEN** the tracking gateway returns HTTP 404 without challenge evidence
- **THEN** the library returns an unavailable-tracking error rather than asserting that the parcel does not exist

#### Scenario: Discovery returns a missing page
- **WHEN** a discovery resource returns HTTP 400 or 404 without challenge evidence
- **THEN** the library returns a configuration error, not a tracking-number error

#### Scenario: The native HTTP client encounters a retryable protocol failure
- **WHEN** a controlled native-adapter test produces a failure that the HTTP client normally retries
- **THEN** the request is not automatically reissued

### Requirement: Shipment and article identity

The result MUST retain the canonical tracking number, returned consignment identifier, and all returned article identifiers.
Events MUST remain associated with their source article and detail group.
A missing required shipment identity MUST produce an invalid-response error.

#### Scenario: Several articles share a consignment
- **WHEN** the response contains several articles or detail groups
- **THEN** the result retains each group and its events rather than selecting the first one
- **AND** article status does not silently replace shipment status

### Requirement: Match supported shipment identifiers

The response MUST match the canonical tracking number against its consignment identifier, any article identifier, or the first `|`-delimited segment of an article barcode.
Matching MUST ignore ASCII letter case without changing returned identifiers.
A response matching none of these identifiers MUST produce an invalid-response error.

#### Scenario: The caller uses an article number
- **WHEN** the number matches an article identifier but differs from the returned consignment identifier
- **THEN** the library accepts the response and retains both identities

#### Scenario: The caller uses a barcode alias
- **WHEN** the number matches the first `|`-delimited barcode segment, ignoring ASCII case
- **THEN** the library accepts the response without requiring equality with the consignment identifier

#### Scenario: The response identifies another shipment
- **WHEN** none of the supported response identifiers matches the canonical tracking number
- **THEN** the library returns an invalid-response error instead of presenting another shipment's data

### Requirement: Optional shipment details

The result MUST expose optional From/To cities, carrier-reported shipment start, and ETA information.
It MUST preserve an ETA's available precision, including a date, date range, or time window without a known date or zone.
Missing details MUST remain unavailable.
Scan locations MUST NOT be substituted for From/To cities.
The earliest valid event time MUST be available separately as the first recorded event.

#### Scenario: The response omits summary details
- **WHEN** the response has events but no origin city, destination city, start date, or ETA
- **THEN** those details remain unavailable
- **AND** the earliest event remains available as the first recorded event, not an asserted pickup time

#### Scenario: The ETA has no time of day
- **WHEN** the carrier supplies a date-only delivery estimate
- **THEN** the result retains a date-only estimate without inventing midnight, a time zone, or an exact delivery time

#### Scenario: The ETA window has no date or time zone
- **WHEN** the carrier supplies start and end clock times without a date or zone
- **THEN** the result retains those clock times with an unavailable date and zone
- **AND** it does not assume the window means today at the caller's location

### Requirement: ETA availability and scope

The library MUST respect the carrier's automatic-ETA eligibility and status rules without making a separate postcode-based estimate request.
It MUST retain supplied revision and passed-window metadata without calculating lateness itself.
It MUST NOT present one article's estimate as a shipment-wide estimate when its scope is ambiguous.

#### Scenario: An estimate requires a separate calculation
- **WHEN** the carrier marks an article as requiring the user-requested estimate flow
- **THEN** its automatic ETA is unavailable with that reason
- **AND** the library does not call another estimate service

#### Scenario: Several articles have estimates
- **WHEN** a shipment contains several articles or detail groups with separate estimates
- **THEN** their estimates stay associated with those groups
- **AND** the shipment-level ETA remains unavailable unless the carrier explicitly supplies one

### Requirement: Carrier status is separate from request outcome

The result MUST distinguish a successful API request from a shipment's delivery status.
It MUST preserve carrier status text, including unfamiliar statuses.
It MUST keep shipment status separate from article and detail-group statuses.
Absent shipment status MUST remain unknown rather than be inferred from one article.

#### Scenario: The API request succeeds
- **WHEN** the response has request status `Success` and shipment milestone `Delivered`
- **THEN** the shipment status is `Delivered`, not `Success`

#### Scenario: Only one article is delivered
- **WHEN** one article is delivered but no shipment-level status is supplied
- **THEN** the shipment status remains unknown
- **AND** each supplied article status remains visible

### Requirement: Preserve journey events

The result MUST retain all returned events, including repeated descriptions, with their article association, carrier text, location, code, and source timestamps.
Events with valid instants MUST be ordered chronologically, oldest first, using stable source order for ties.
Events without a usable instant MUST follow dated events in source order.

#### Scenario: Events arrive out of order across time zones
- **WHEN** the response contains events with different UTC offsets in reverse order
- **THEN** the result orders them by their actual instants rather than their displayed clock times

#### Scenario: Repeated and undated events exist
- **WHEN** two distinct delivered scans share a description and another event has no valid time
- **THEN** both delivered scans remain present
- **AND** the undated event remains present after the dated events

### Requirement: Keep event time and retrieval time distinct

The result MUST record retrieval time separately from carrier event timestamps.
It MUST preserve supplied local offsets and epoch values.
It MUST NOT substitute milestone or retrieval time for event time.
Conflicting or malformed event timestamps MUST retain their source values and expose an unavailable normalized instant.

#### Scenario: Two timestamp representations agree
- **WHEN** an event's offset-bearing local timestamp and epoch milliseconds identify the same instant
- **THEN** the event exposes that instant and retains both source representations

#### Scenario: Event timestamps disagree
- **WHEN** the local timestamp and epoch milliseconds identify different instants, or a supplied timestamp is malformed
- **THEN** the event has no normalized instant
- **AND** its source values and timestamp issue remain available without failing the whole shipment

#### Scenario: Milestone and event times differ
- **WHEN** the shipment milestone has a time different from the latest recorded event
- **THEN** neither time overwrites the event timestamp or retrieval time

### Requirement: Evidence-based status age

The result MUST expose status-change time only from a verified carrier transition or unambiguous event history.
History-derived shipment age MUST require one article and one detail group, with known and matching shipment and detail statuses.
Any supplied article status MUST agree.
Repeated events MUST NOT reset that time.
Missing, contradictory, or ambiguous transition evidence MUST leave the time unavailable.

#### Scenario: Two scans repeat the current status
- **WHEN** a single-article, single-detail shipment has matching known shipment and detail statuses and no conflicting article status
- **AND** a known prior status is followed by two dated events explicitly associated with the current status
- **THEN** the first event in that uninterrupted current-status sequence establishes its recorded start
- **AND** the second event does not reset the status-change time

#### Scenario: Several articles or detail groups contain status events
- **WHEN** more than one article or detail group exists and no verified shipment transition timestamp is supplied
- **THEN** the shipment status-change time remains unavailable
- **AND** the library does not choose a convenient article or combine detail histories

#### Scenario: Current status identities are missing or disagree
- **WHEN** shipment or detail status is missing, those statuses disagree, or a supplied article status conflicts
- **THEN** event history does not establish shipment status-change time

#### Scenario: Event order contradicts transition history
- **WHEN** source event order and valid instants disagree, or a transition sequence includes missing status or ambiguous times
- **THEN** event history does not establish shipment status-change time

#### Scenario: Only a current milestone label is known
- **WHEN** the response identifies the current status but supplies no verified transition time or usable transition history
- **THEN** the status is available but its change time remains unavailable

### Requirement: Private and offline-testable operation

The library MUST contact only approved carrier resources for a lookup.
It MUST NOT send parcel data to solver, analytics, or other third-party services.
Default tests MUST use synthetic fixtures and must not require live parcels, credentials, or Internet access.

#### Scenario: Run the normal test suite
- **WHEN** the test suite runs without Internet access or credentials
- **THEN** discovery, success, failure, and challenge behavior can be tested with synthetic responses
- **AND** no live carrier request is attempted
- **AND** native-adapter network tests use controlled loopback servers only
