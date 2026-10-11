---
title: Machine-readable contracts for published structured data, piloted on Cua Driver trajectories
authors:
  - methanoya
created: 2026-10-10
last_updated: 2026-10-11
status: review
discussion: https://github.com/trycua/cua/issues/4956
rfc_pr:
implementation:
supersedes:
superseded_by:
---

# RFC: Machine-readable contracts for published structured data, piloted on Cua Driver trajectories

## Summary

Cua will adopt one rule for structured data it publishes to other components,
other languages, or user tooling: the document has a machine-readable schema,
a hermetic test validates the real writer's output against it, and separate
value tests cover meaning that a schema cannot express.

This RFC asks for two decisions:

1. accept the rule for new published documents and for existing ones whose
   shape changes; and
2. apply it first to one pilot, the Cua Driver trajectory files
   (`action.json`, `session.json`, `evidence.json`).

The pilot needs three design choices to be implementable. The proposal makes
each one and marks it as a default that review can overturn: the published
schemas are open and the producer test is strict, versions stay where they are
today, and the first schemas are hand-written.

Three other formats examined in the same audit (sandbox state, Spaces stores,
Lume HTTP and CLI JSON) are listed as candidates. They are not part of this
decision. Each would be proposed separately, using what the pilot shows.

## Motivation

trycua/cua#4710 reported that every recorded turn's `action.json` carried
`timestamp` as fractional Unix seconds (`"1791255986.559"`), while the
`start_recording` description, the recording reference page, and the bundled
`RECORDING.md` skill all promised ISO-8601. The format of `action.json` is
defined only by that prose. No schema exists, and no test compared the written
value with the documented format. trycua/cua#4949 fixes the value and adds a
focused test for that one field.

Where the repository already has an executable definition and a drift gate,
the published shape and its description have stayed aligned:

- Cua Driver tool inputs and outputs are checked by protocol and schema tests.
- Cua Driver history events have a published JSON Schema
  (`libs/cua-driver/docs/computer-history-event-v0.schema.json`).
- SDK telemetry declares every event and property in one table, validates each
  payload against it, and a test checks the public documentation table against
  the same data.
- The Spaces MCP manifest is generated and has a `--check` drift gate.
- RPC messages are governed by the `.proto` files and `buf` checks.

### What the audit found, and what this rule would and would not catch

The audit that followed #4710 looked for time values whose name or
documentation promises one thing while the code writes another. It produced
eight findings. Six are time values; two (the documented `pid` and the fields
with no value test) surfaced while reading the recorder. With #4710 that makes
nine items. They are grouped here by which part of the rule addresses them,
because the groups are not equal.

**Caught by a schema validated against real output, with format checking**

| Finding                                                             | Where           | Status                               |
| ------------------------------------------------------------------- | --------------- | ------------------------------------ |
| `action.json` `timestamp` was epoch seconds, documented as ISO-8601 | Driver recorder | Reported as #4710; fix open as #4949 |

**Caught by checking the field reference against the schema**

| Finding                                                                                                                                                | Where                               | Status                                                      |
| ------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------- | ----------------------------------------------------------- |
| The one-line field list in three documents names a top-level `pid` in `action.json`; none is written. The same list omits four fields that are written | Driver recorder and three documents | Confirmed by reading; not filed. Corrected inside the pilot |

**Caught only by the value tests the rule requires beside the schema**

| Finding                                                                                                                | Where                      | Status                                                                                                                                                             |
| ---------------------------------------------------------------------------------------------------------------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `session.json` `started_at_monotonic_ms` is wall-clock time and is overwritten with the stop time                      | Driver recorder            | Reproduced twice: moved 1,525 ms after a 1.5 s recording; in a real six-turn recording it held a time about 37 s after the session start. Filed as trycua/cua#4953 |
| `result_summary`, `arguments`, and the two relative timing fields of `action.json` have no direct test of their values | Driver recorder tests      | Confirmed by search; the timing fields gained an alignment check in #4949                                                                                          |
| Python `LumeRuntime.list_checkpoints` sets every checkpoint's `created_at` to the time of the call                     | `cua-sandbox` Lume runtime | Reproduced with a stubbed VM list: moved 1.201 s between two calls. Filed as trycua/cua#4954. Outside the pilot                                                    |

**Reduced by a shared schema across writers, outside the pilot**

| Finding                                                                                              | Where         | Status                                        |
| ---------------------------------------------------------------------------------------------------- | ------------- | --------------------------------------------- |
| TypeScript group chats write millisecond ISO dates; Rust and TypeScript routines write whole seconds | Spaces stores | Confirmed by reading; no misordering observed |

**Not addressed by this rule**

| Finding                                                          | Where                       | Why not                                                           |
| ---------------------------------------------------------------- | --------------------------- | ----------------------------------------------------------------- |
| Stale `LastAccessTime` wording in a macOS field comment          | `platform-macos`            | A code comment, not published data                                |
| Three separate ISO-8601 formatters in the driver                 | Driver crates               | Code duplication; each produces valid output                      |
| Queued telemetry events record a capture time that is never read | TypeScript telemetry client | An unused internal field; the sent payload is unaffected in shape |

Three conclusions follow, and the proposal is built on them:

- A schema with format checking, validated against real output, would have
  caught #4710.
- It would not have caught the `pid` mismatch. A schema that describes what the
  recorder writes has no `pid`, and the recorder's output still validates. The
  mismatch is between the prose and the file, so the check that catches it is
  the comparison of the field reference with the schema.
- Neither check would have caught the `session.json` or checkpoint bugs: both
  write a well-formed value that means the wrong thing. Those need value tests.

The audit looked mainly at time values. It is evidence that unprotected formats
drift, not a measurement of how often.

## Goals

- Give `action.json`, `session.json`, and `evidence.json` a published schema
  and a test that validates real recorder output against it.
- Record, in one field-contract table, the value test that owns each field's
  meaning, or that only the field's shape is contractual.
- Settle the rule's wording, the validator configuration, and the cost of
  adoption on one component before asking any other component to follow it.
- Detect undeclared, missing, or retyped fields and invalid date-time values in
  the driver's ordinary pull-request CI.

## Non-goals

- Migrate the sandbox state file, the Spaces stores, or the Lume API. They are
  candidates only.
- Build a repository-wide inventory or a CI gate that discovers unregistered
  formats.
- Change the shape or meaning of any trajectory field. The `session.json`
  start time (trycua/cua#4953) is fixed independently and does not block this
  RFC. Whether `action.json` should gain a top-level `pid` is a separate
  feature decision; the pilot only corrects the documentation that claims one.
- Cover the other files a recording writes: `cursor.jsonl`,
  `before_state.json`, `after_state.json`, `app_state.json`, the images, and
  the video.
- Standardize time representations across components.
- Replace behavior tests with schema validation.
- Generate tool descriptions or skill text from a schema.
- Touch the read-only `libs/fleet` mirror.

## Terminology

**Published structured data**
: Structured data whose shape Cua owns, written to disk, sent across a process
or network boundary, or printed as machine-readable output, and read by another
component, another language implementation, or external tooling.

**Producer conformance test**
: A hermetic test that calls the real writer through a public interface or
production seam and validates the bytes it wrote against the schema.

**Value test**
: A test of meaning that structure cannot establish: that a start time is
captured once, that a field named for a clock uses that clock, or that two
fields denote the same instant.

## Current state

A recording writes one `session.json` per run and one `turn-NNNNN/` folder per
action, holding `action.json`, `evidence.json`, state snapshots, and images.

- No JSON Schema exists for `action.json`, `session.json`, or `evidence.json`.
- The three files version themselves in three ways. `session.json` carries
  `schema_version: 1`. `evidence.json` carries a version token,
  `"schema": "cua-turn-evidence/v1"`. `action.json` carries nothing.
- Each file has conditional shapes. `session.json` `video` has four: absent,
  absent with an `error`, in flight (`present`, `path`), and finalized (adding
  `absolute_path`, `duration_ms`, `finalized`). `action.json` writes
  `click_point`, `click_point_image`, and `action_truth` only on some turns.
  `evidence.json` adds `click_source` only when a supplemental click image was
  captured or failed.
- A one-line field list for `action.json` appears in three places. Two are
  written by hand: the `start_recording` tool description and `RECORDING.md`.
  The third, `reference/mcp-tools/recording.mdx`, is generated from the tool
  description. `RECORDING.md` also describes some fields, such as
  `click_point_image`, in later paragraphs.
- The recorder takes its video backend through the `VideoBackend` trait, and
  existing tests already supply their own, so the video shapes can be produced
  hermetically.
- Tests assert selected fields. `recording_semantic_evidence.rs` and
  `recording_turn_latency.rs` cover `evidence.json`, `result_error`, click
  points, and `action_truth`. `recording_action_timestamp.rs`, added by #4949,
  covers `timestamp` and its alignment with the relative timing fields.
- The driver workspace already depends on the `jsonschema` crate (0.46), and
  `cua-driver-contract` already uses `schemars`, so the pilot adds no new
  dependency. Existing call sites build validators with
  `jsonschema::validator_for`, which does not assert `format` for 2020-12
  schemas.

## Proposal

### 1. The rule

A document is in scope when Cua owns its shape, it crosses a disk, process,
network, or command-line boundary, and something other than its writer relies
on its fields. Machine-readable command output, such as a `--json` flag, is in
scope on the same terms as a file. For such a document:

1. A machine-readable schema defines required fields, types, nullability,
   enumerations, and formats. JSON uses JSON Schema 2020-12. RPC messages stay
   with protobuf. The published schema is open: readers ignore fields they do
   not know.
2. A producer conformance test validates real writer output against the schema
   with format checking enabled, and separately fails when the writer emits a
   field the schema does not declare. It does not build a second object that
   resembles the output.
3. The schema suite has a negative control showing the validator is configured
   as intended.
4. Each meaning a schema cannot express has one value test at the strongest
   boundary that can observe it, following the existing one-owner-test rule.
5. A change to the published shape updates the schema and its tests in the same
   pull request, and states whether supported readers tolerate it.

The rule applies to new published documents and to existing ones when their
shape changes. Existing untouched formats are not required to migrate. That
clause reaches the recording files the pilot leaves out: the next change to
the shape of `cursor.jsonl` or of the state snapshots would bring a schema with
it.

### 2. The pilot: Cua Driver trajectory files

1. **Schemas.** Publish schemas for `action.json`, `session.json`, and
   `evidence.json` beside the existing history-event schema in
   `libs/cua-driver/docs/`. They are hand-written, open to unknown fields, and
   each has a versioned `$id`. They describe what the recorder writes today and
   change no file: existing version markers stay as they are, and
   `action.json` gains none.
2. **Conformance test.** One test in `cua-driver-core` records through
   `RecordingSession` into a temporary directory and validates every written
   file. The validator is built with `should_validate_formats(true)`, not with
   `validator_for` defaults. The test also fails on any written field the
   schema does not declare, which is the strict producer check; the published
   schema stays open.
3. **Required variants.** The test must produce each conditional shape:
   - `action.json`: a dispatched action; an action refused before dispatch; a
     click with `click_point`; a click with `click_point_image`; turns with and
     without `action_truth`.
   - `evidence.json`: state captured; state capture disabled; a refused action
     (`not_applicable`); `click_source` captured and unavailable.
   - `session.json`: as written at start and as rewritten at stop; all four
     `video` shapes, using a test `VideoBackend`.
4. **Negative controls.** The pre-#4710 value `"1791663498.695"` fails as a
   `date-time`, and an undeclared field added to real output fails the strict
   producer check.
5. **Field-contract table.** One table in `libs/cua-driver/docs/`, beside the
   schemas, lists every field of the three files with a one-line description
   and either the test that owns its meaning or the words "shape only". It is
   the single field reference. The pilot adds the value tests the audit found
   missing for `arguments` and `result_summary`. The `action.json` timing
   fields are already owned by the test in #4949. The `session.json` start time
   is listed as owned by trycua/cua#4953, and its value test lands with that
   fix: it would fail on today's recorder, so the pilot locks only that field's
   type.
6. **One field reference.** The `start_recording` description and
   `RECORDING.md` stop enumerating `action.json` fields and point to the
   field-contract table; the generated `recording.mdx` follows from the tool
   description. This removes the claimed top-level `pid` and the omission of
   four written fields. It is a documentation fix and changes no file. A check
   then compares the table with the schema in both directions, so a field
   present on only one side fails.

### 3. Enforcement, and its limit

Within the pilot, enforcement is mechanical: the conformance test fails when
the recorder writes a field the schema does not declare, omits a required one,
or changes a type.

Outside the pilot, enforcement is by review. `CONTRIBUTING.md` and `AGENTS.md`
state the rule in a few lines and link to the pilot as the worked example.
Nothing in CI can detect that a pull request introduced a new published JSON
file with no schema. This RFC does not claim otherwise. A machine-checked
inventory could narrow that gap later; whether it is worth its upkeep is a
question for after the pilot.

### 4. Candidates after the pilot

Not part of this decision. Listed so reviewers can see where the rule would go
next and judge whether it generalizes.

| Candidate                               | Writers and readers                | What exists today                                                                                                                                                        |
| --------------------------------------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Sandbox state under `~/.cua/sandboxes/` | Rust and Python write; both read   | No shared schema. Rust imitates Python's timestamp spelling through `python_utc_now()`. No drift was observed; the two writers are aligned by hand                       |
| Spaces routine and group-chat stores    | Rust, TypeScript, Swift            | No shared schema. One precision inconsistency, tolerated by all readers                                                                                                  |
| Lume HTTP and CLI JSON                  | Swift writes; Python and Rust read | No OpenAPI document. The checkpoint bug above is in a client, and a response schema alone would not have caught it; it would have made the missing creation time visible |

## Alternatives considered

### Keep per-field tests only

This is what #4949 does for one field, and it is the right size for a bug fix.
It protects only the fields someone remembers to assert, and gives user tooling
nothing to validate against.

### Adopt the rule repository-wide with an inventory and four migrations

An earlier draft of this RFC proposed that. It asks every component to take on
new machinery on the strength of one shipped bug, before the cost is known. The
pilot produces that evidence first.

### Closed published schemas

Rejecting unknown fields in the published schema would make the strict check
available to external tooling too. It would also make every additive field
invalidate trajectories for anyone validating with an older schema. Keeping the
strictness in the driver's own test gives the drift check without that cost.

### Generate the schema from Rust types

Preferable in the long run. The recorder currently builds its payloads with
`json!`, so a generated schema would first require typed payload structs. The
pilot can start with hand-written schemas validated against real output and
move to generation if maintainers want the refactor.

### Golden files

Golden files detect byte changes but cannot tell contractual structure from
incidental formatting, and timestamps make them unstable. They remain useful as
compatibility fixtures beside a schema.

## Compatibility and migration

Adopting the rule and landing the pilot changes no runtime output and needs no
release of its own. The documentation correction in the pilot changes what the
docs claim, not what the recorder writes.

The published schemas are open, so a later additive field does not invalidate a
trajectory for tooling that validates with an older schema. The strict check
lives only in the driver's own test, where it makes every additive change a
deliberate schema change.

Two known mismatches are handled outside the schemas:

- `session.json` `started_at_monotonic_ms` (trycua/cua#4953): the name and the
  value disagree, and external tools may read either. The schema records the
  current type and links the issue. The fix is independent of this RFC and
  carries its own compatibility note.
- Top-level `pid` in `action.json`: the pilot corrects the documentation.
  Adding the field later would be an additive change with its own issue.

Rollback has two parts. The enforcement, meaning the conformance test and the
documentation check, can be disabled or removed at any time. A published
schema is itself a contract once tooling validates against it, so the versioned
schema files stay in place; a wrong schema is corrected by publishing a new
version, not by deleting the old one.

## Security, privacy, and telemetry

Schemas describe field names, types, and structure. They grant no permission
and authorize no new data collection.

`action.json` records tool arguments, which can include typed text. Fixtures
and test failures use synthetic data only, and CI diagnostics name the schema
path and the validation error without printing a payload. Publishing the schema
makes it easier to see which fields can carry sensitive content, including the
host path in `session.json` `video.absolute_path`; it does not change what the
recorder writes or who can read it.

No telemetry changes.

## Implementation plan

1. Add schemas for the three trajectory files.
2. Add the conformance test with its required variants, the strict producer
   check, and the negative controls in `cua-driver-core`.
3. Add the field-contract table and the value tests for `arguments` and
   `result_summary`.
4. Point the tool description and the skill at the field-contract table,
   regenerate `recording.mdx`, and add the two-way check between the table and
   the schema.
5. Add the rule to `CONTRIBUTING.md` and `AGENTS.md`, linking the pilot.
6. Report back on trycua/cua#4956: what the pilot cost, what it caught, and
   whether a candidate from the table should follow.

Steps 1 to 4 are one or two pull requests in one component. Step 5 is
documentation only.

trycua/cua#4953 is not a step. It proceeds on its own schedule. Whichever
lands second updates the other: the fix adds its value test to the
field-contract table, or the table is created with that row already pointing
at the fix.

## Test and acceptance plan

The RFC is complete when:

- The conformance test validates every trajectory file written by a real
  `RecordingSession` for each required variant, with format checking enabled.
- The negative control with the pre-#4710 timestamp fails as a `date-time`.
  Removing `should_validate_formats(true)` makes that control fail, which
  proves the configuration is what catches it.
- Mutation checks: adding, removing, or retyping a written field fails the
  conformance test until the schema is updated.
- The field-contract table has a row for every schema field, and each named
  owner test exists.
- The documentation check fails when a field is present on only one side of
  the schema and the field-contract table, and the generated-docs drift check
  for the driver passes.
- `cargo test -p cua-driver-core --locked` stays hermetic, and the existing
  protocol and schema tests continue to pass.

Completion does not depend on trycua/cua#4953.

No desktop E2E run is required: the change is serialization and tests, proved
at a hermetic boundary on every platform by the shared core crate.

## Unresolved questions

The proposal sets a default for each of the first three. Reviewers are asked to
confirm or overturn it.

- Open published schemas with a strict producer test (the default), or closed
  schemas that also reject unknown fields for external readers?
- Leave the three version markers as they are (the default), or use the pilot
  to give `action.json` a version and align `session.json` `schema_version`
  with the `evidence.json` `schema` token? Either alignment changes a written
  file and would need its own compatibility note.
- Hand-written schemas (the default), or typed payload structs and generated
  schemas as part of the pilot?
- Is a few-line rule in `CONTRIBUTING.md` enough, or do maintainers want the
  inventory before extending the rule past the driver?
- Which candidate, if any, should follow the pilot?

## Decision record

Pending review.
