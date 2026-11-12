# Duplicate-record detector

Goal: Build “Duplicate-record detector” for a 100-row fixture with one transformation, one validation rule, and one output.

This document is the complete public requirements contract. A developer can
implement and verify the goal from this file alone. Conformance is judged on
files, not on any private task-generation information.

## Scope

In scope (exactly):

- One 100-row fixture (input).
- Exactly one transformation (email normalization).
- Exactly one validation rule (email shape).
- Exactly one output artifact (duplicate report).

Out of scope:

- No additional transformations, validation rules, or output artifacts.
- No database, network service, UI, or background job.
- No performance, scalability, or concurrency requirements beyond handling
  100 rows deterministically.

## Public contracts

### R1 — Fixture location and size (test: file inspection)

- The fixture is a UTF-8 CSV file at `fixture/records.csv`.
- It has one header row followed by exactly 100 data rows.
- Line endings are LF. Fields are comma-separated using RFC 4180 dialect:
  comma delimiter, double-quote quoting with `""` escape for an embedded
  quote; embedded commas, quotes, or newlines (LF only; no CR byte is
  permitted anywhere, see R10 rule 4) are permitted only inside
  quoted fields. Column counting follows this dialect.
- The header is exactly `id,email,name` in that order, with no extra
  columns and no BOM.

### R2 — Record schema (test: CSV parse)

- Each data row has three fields:
  - `id`: non-empty after trimming ASCII space, tab, CR, and LF; unique
    across the fixture; compared as a string by Unicode scalar value
    (ordinal, locale-independent comparison, equivalent to UTF-8 byte
    order for valid UTF-8).
  - `email`: raw string as stored (may contain leading/trailing whitespace
    or mixed case); this is the only field the detector normalizes and
    validates.
  - `name`: opaque display string; the detector MUST NOT use it for
    matching, normalization, or validation.
- `id` uniqueness and non-emptiness are structural fixture guarantees, not
  the single validation rule. A fixture violating R1–R2 is malformed input
  (see R10).

### R3 — The one transformation: email normalization (test: unit examples + output)

- For each row, compute `normalized_email` as:
  1. Remove leading and trailing ASCII whitespace (space, tab, CR, LF).
  2. Lowercase ASCII `A–Z` to `a–z`.
- The transformation is deterministic and idempotent: normalizing an
  already-normalized value returns it unchanged.
- No other normalization is permitted (no `name` trimming for matching, no
  `id` folding, no domain canonicalization, no plus-address stripping).
- Examples:
  - `"  Alice@Example.COM "` → `"alice@example.com"`
  - `"BOB@example.com"` → `"bob@example.com"`
  - `"   "` → `""` (empty)

### R4 — The one validation rule: email shape (test: unit examples)

- A row is **valid** iff its `normalized_email` matches:
  `^[^@ \t\r\n]+@[^@ \t\r\n]+\.[^@ \t\r\n]+$`
  (non-empty local part, exactly one `@`, non-empty domain containing at
  least one `.`, none of space/tab/CR/LF and no additional `@`).
- For R4, "whitespace" means exactly space, tab, CR, and LF — the same
  four characters trimmed in R3. This definition governs in every language
  and runtime regardless of regex-engine `\s` semantics. Other characters
  sometimes classed as whitespace (e.g. VT, FF, NBSP, or other Unicode
  spaces) are ordinary characters here and are judged literally by the
  pattern above.
- This is the ONLY row-level validation rule. No other value is rejected
  for content reasons.
- Examples:
  - valid: `"alice@example.com"`, `"a.b@sub.example.co"`
  - invalid: `""`, `"alice"`, `"alice@"`, `"@example.com"`,
    `"alice@example"`, `"a@b@c.com"`, `"a @example.com"`

### R5 — Invalid-row handling (test: output inspection)

- Rows failing R4 are **invalid**: they are excluded from duplicate
  matching and their `id` values appear in the output's `invalid_record_ids`
  list (see R7).
- Invalid rows never form or join a duplicate group, even if two invalid
  rows share the same raw or normalized value.

### R6 — Duplicate definition (test: output inspection)

- Two valid rows are duplicates iff their `normalized_email` values are
  byte-equal.
- A duplicate group is a set of 2 or more valid row `id`s sharing one
  `normalized_email`.
- Singleton emails (appearing once among valid rows) produce no group.

### R7 — The one output artifact (test: JSON parse + sort checks)

- The detector produces exactly one output file at
  `output/duplicates.json`, UTF-8 encoded, LF line endings, 2-space indent,
  trailing newline.
- Its JSON value is an object with exactly two keys in this order:
  ```json
  {
    "duplicate_groups": [
      {"normalized_email": "alice@example.com", "record_ids": ["001", "042"]},
      {"normalized_email": "bob@example.com", "record_ids": ["003", "010", "099"]}
    ],
    "invalid_record_ids": ["007"]
  }
  ```
- Schema:
  - `duplicate_groups`: array of objects with `normalized_email` (string)
    and `record_ids` (non-empty array of strings, length ≥ 2, sorted
    ascending by Unicode scalar value, no duplicates).
  - Groups sorted ascending by `normalized_email` by Unicode scalar value.
  - All R7 orderings use ordinal string comparison by Unicode scalar value
    (equivalent to UTF-8 byte order for valid UTF-8), never locale
    collation.
  - `normalized_email` values are distinct across groups.
  - `invalid_record_ids`: array of strings sorted ascending by Unicode
    scalar value, no duplicates; empty array when no invalid rows exist.
- Every `id` in a duplicate group or in `invalid_record_ids` appears
  exactly once across both lists. Valid singleton `id`s (R6) appear in
  neither list.
- No other output artifact counts as the result. Diagnostic console output
  or logs are permitted but MUST NOT be required to verify correctness;
  file-to-file conformance (`fixture/records.csv` → `output/duplicates.json`)
  is the acceptance criterion regardless of invocation mechanism.

### R8 — Fixture coverage (test: file inspection + output)

- The shipped 100-row fixture MUST exercise the whole contract:
  - at least one duplicate group of size ≥ 2,
  - at least one duplicate pair differing only by case and/or surrounding
    whitespace (proving R3 ran),
  - at least one invalid row per R4 (proving R5 is observable).
- The remaining rows are unique valid emails unless needed for the above.
- Valid singleton rows (including those remaining unique valid emails)
  appear in neither output list per R7.

### R9 — Determinism (test: run twice, byte-compare)

- The same fixture bytes always produce byte-identical output bytes.
- No timestamps, random values, process IDs, absolute paths, or network
  data appear in the output.
- Sorting rules in R7 are mandatory, not advisory.

### R10 — Malformed-input behavior (test: negative cases)

- The malformed set is closed and exhaustive. Malformed input is exactly:
  1. missing fixture file;
  2. non-UTF-8 bytes;
  3. BOM present (R1 forbids a BOM);
  4. any CR byte anywhere in the file (including inside a quoted field,
     where it is malformed content rather than permitted field content) or
     any record terminator other than LF, including a final data row lacking
     its terminating LF (R1 requires LF);
  5. header not exactly `id,email,name` (including header-whitespace
     variants or extra columns);
  6. any data row with ≠ 3 columns counted under the R1 RFC 4180 dialect
     (embedded commas/quotes/newlines count per that dialect);
  7. data-row count ≠ 100, where data rows exclude the header and blank or
     whitespace-only lines are never skipped: a blank/whitespace-only line
     is itself a data row and therefore fails rule 6 or this rule;
  8. duplicate `id` or `id` empty after trimming ASCII space/tab/CR/LF.
  A single terminating LF after the final data row is required line
  termination, not a blank line; a file whose final data row lacks that
  terminating LF is malformed under rule 4.
- On malformed input the detector MUST fail visibly: exit with a non-zero
  status and emit a human-readable error message on stderr. The
  implementation MUST document this exit/stderr behavior alongside its
  command.
- On malformed input the detector MUST NOT leave a valid-looking
  `output/duplicates.json`: the path MUST either be absent or contain
  bytes that fail JSON parsing or R7 validation. A stale output MUST be
  removed, or overwritten only on success.
- Row-level R4 invalid emails are NOT malformed input; they are handled per
  R5 with a successful output.

### R11 — Boundedness (test: review)

- The implementation MUST NOT add a second transformation, a second
  validation rule, or a second result artifact (no extra reports, metrics
  files, or alternate duplicate lists).
- `name` MUST NOT influence matching. Domain-specific equivalences
  (dot-insensitivity, plus-addressing, internationalized-domain mapping) are
  explicitly excluded.

## Boundary cases (summary, all covered by R1–R11)

- Empty/whitespace-only email → invalid, never grouped.
- Same email with different case or padding → same group.
- Three or more rows sharing one email → one group with all ids.
- Valid singleton email → appears in neither list (R6–R7).
- No duplicates → `"duplicate_groups": []`.
- No invalid rows → `"invalid_record_ids": []`.
- Duplicate `id`s or empty `id` → malformed input (R10), not a group.

## Acceptance checklist

1. `fixture/records.csv` exists, parses as CSV, has header `id,email,name`
   and exactly 100 data rows.
2. Normalization examples in R3 hold for the implementation.
3. Validation examples in R4 hold (valid accepted, invalid listed).
4. `output/duplicates.json` validates against the R7 schema and
   Unicode-scalar sort orders.
5. Every `id` in a group or in the invalid list appears exactly once, and
   every valid singleton appears in neither list.
6. Re-running on the same fixture yields byte-identical output.
7. Each malformed case in the closed R10 list fails with non-zero exit plus
   a stderr message, leaving output absent or failing R7 validation.
8. No second transformation, validation, or output artifact exists.

## Notes for implementers

- Any language or runtime may be used; conformance is file-to-file.
- Document the exact command that maps `fixture/records.csv` to
  `output/duplicates.json` so the acceptance checklist can be repeated.
