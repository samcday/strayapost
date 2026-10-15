# tracking-cli Specification

## Purpose

Let people inspect an Australia Post shipment with one command.
Provide readable terminal output and a predictable JSON interface for scripts using the same library result.

## Requirements

### Requirement: Single-number invocation

The CLI MUST support `strayapost <tracking-number>` and `strayapost --json <tracking-number>`.
It MUST accept exactly one tracking number per lookup.
It MUST use the crate's library for discovery, lookup, normalization, and failure classification.
Help, version, and invalid invocations MUST NOT make network requests.

#### Scenario: Look up a shipment
- **WHEN** the user invokes `strayapost EXAMPLE123` with a successful carrier response
- **THEN** the CLI performs one library lookup and prints the shipment result

#### Scenario: Omit or add tracking numbers
- **WHEN** a lookup invocation has no tracking number or more than one
- **THEN** the CLI reports an argument error and exits with status 2
- **AND** no lookup starts

#### Scenario: Request help or version
- **WHEN** the user passes `--help` or `--version`
- **THEN** the CLI prints the requested text and exits with status 0 without a lookup
- **AND** help or version takes precedence over JSON tracking output

### Requirement: Shipment summary precedes the journey

Default output MUST start with the tracking number and shipment summary, followed by the journey table.
The summary MUST include From/To cities, shipment start, ETA, current status, and time since the last evidenced status change.
It MUST identify retrieval time separately from the latest event time.
Information MUST remain understandable without color.

#### Scenario: All summary details are available
- **WHEN** the library returns a shipment with complete summary details
- **THEN** the CLI prints those details before the journey table
- **AND** shipment status is not replaced by the API request outcome

### Requirement: Honest display of unavailable details

The CLI MUST label unavailable summary fields `Unknown`.
It MUST NOT infer cities from scan locations or show the first recorded event as a proven shipment start.
When shipment start is unavailable but the earliest event is known, it MUST show that time separately as `First recorded`.
An ETA MUST retain the precision supplied by the carrier.

#### Scenario: Only journey events are available
- **WHEN** the library returns events but no From/To cities, shipment start, or ETA
- **THEN** those summary fields show `Unknown`
- **AND** the earliest valid event appears as `First recorded`

#### Scenario: An estimated date range is supplied
- **WHEN** the carrier supplies an ETA range without times of day
- **THEN** the CLI labels it as an estimate and displays the range without invented times

### Requirement: Status age uses transition evidence

The CLI MUST calculate elapsed status age from the library's status-change time and retrieval time.
It MUST also show the absolute status-change time when available.
Absent or future transition times MUST produce an unknown age, not a fabricated or negative duration.
Repeated events with the same status MUST NOT reset the displayed age.

#### Scenario: A later scan repeats delivery
- **WHEN** delivery was first recorded at 12:00, repeated at 15:00, and fetched at 16:00 on the same date and offset
- **AND** the library establishes 12:00 as entry into the delivered status
- **THEN** the summary reports four hours in that status, not one hour

#### Scenario: The change time is unavailable
- **WHEN** the current status is known but its change time is unavailable or later than retrieval time
- **THEN** the status remains visible and its elapsed age shows `Unknown`

### Requirement: Complete journey table

The journey table MUST show every library event in chronological order, oldest first, followed by undated events.
Each row MUST include event time, location, and carrier description.
Times MUST show their UTC offset when known.
Multi-article results MUST identify each event's article and detail group and retain their statuses.

#### Scenario: Several articles have events
- **WHEN** a shipment contains events from several articles or detail groups
- **THEN** the output identifies the source of each event without mixing their statuses
- **AND** repeated carrier descriptions do not cause rows to disappear

#### Scenario: There are no events
- **WHEN** a valid shipment has no journey events
- **THEN** the CLI prints the summary and `No journey events reported`
- **AND** it does not treat an empty journey alone as a failed lookup

#### Scenario: An event timestamp is invalid
- **WHEN** an event has no normalized time because its source timestamps conflict or are malformed
- **THEN** the row remains visible with an unknown time and a timestamp warning

### Requirement: Automatic terminal color

Human output MUST use color by default when its output stream is an interactive terminal.
It MUST omit color when redirected, when `TERM=dumb`, or when `NO_COLOR` has a nonempty value.
Color MUST supplement text rather than carry status information alone.
JSON output MUST never contain terminal formatting codes.

#### Scenario: Run in a capable terminal
- **WHEN** human output goes to an interactive terminal with color enabled
- **THEN** the summary and table use readable color accents
- **AND** all labels and status information remain present as text

#### Scenario: Redirect or disable color
- **WHEN** output is redirected, `TERM=dumb`, or `NO_COLOR=1`
- **THEN** the output contains no ANSI color sequences

### Requirement: Versioned JSON results

JSON tracking mode MUST write exactly one JSON object followed by a newline to stdout.
The object MUST include `schema_version: 1`, `ok`, and the canonical tracking number when available.
A success MUST include retrieval time and the structured shipment result, including all events and source timestamps.
Unavailable values MUST be JSON `null`, not display strings.
Diagnostics MUST NOT contaminate stdout.

#### Scenario: Serialize a successful lookup
- **WHEN** `strayapost --json EXAMPLE123` succeeds
- **THEN** stdout parses as one object with `schema_version: 1`, `ok: true`, `tracking_number`, `retrieved_at`, and `shipment`
- **AND** it contains no banner, progress indicator, ANSI formatting, or human table
- **AND** missing shipment details are represented as `null`

#### Scenario: Preserve event data for scripts
- **WHEN** the result contains multiple articles, repeated events, or invalid source timestamps
- **THEN** JSON retains their identities, source values, and timestamp issues
- **AND** it does not replace exact values with relative phrases such as `two hours ago`

### Requirement: Structured errors and stable exits

The CLI MUST exit with 0 for a successful lookup, 1 for a lookup failure, and 2 for invalid arguments or tracking input.
Human-mode failures MUST go to stderr with no success output.
JSON-mode failures MUST produce the versioned error object on stdout, with `ok: false` and a stable error code.
Error text MUST distinguish a detected challenge from ordinary denial or network failure.

#### Scenario: A challenge blocks the lookup
- **WHEN** the library returns a challenge-required error
- **THEN** the CLI exits with status 1 and explains that the lookup stopped at a challenge
- **AND** JSON mode uses error code `challenge_required` and identifies the blocked stage
- **AND** no browser or solver is started

#### Scenario: JSON mode has invalid arguments
- **WHEN** `--json` is present in a lookup invocation with a missing tracking number or unsupported option
- **THEN** stdout contains one error object with code `invalid_arguments`
- **AND** the CLI exits with status 2 without making requests

#### Scenario: Rate limiting differs from access denial
- **WHEN** the library reports a rate limit or access denial without challenge evidence
- **THEN** JSON uses `rate_limited` or `access_denied`, respectively
- **AND** valid retry guidance is included for a rate limit without triggering an automatic retry

### Requirement: Safe terminal rendering

The CLI MUST treat carrier text and user input as untrusted data.
Human rendering MUST escape terminal control characters rather than interpret them.
JSON MUST retain data through normal JSON escaping.
Diagnostics MUST NOT print client keys, cookies, challenge tokens, or raw response pages.

#### Scenario: A description includes terminal controls
- **WHEN** a carrier description contains an escape sequence or embedded control character
- **THEN** human output displays a harmless representation without executing terminal controls
- **AND** JSON represents the value as a valid escaped string
