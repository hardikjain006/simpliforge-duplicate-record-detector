# Duplicate-record detector

Detects duplicate records in a fixed 100-row fixture.

Goal: build a duplicate-record detector for a 100-row fixture with one
transformation, one validation rule, and one output. This document is the
complete public requirements contract. No other behavior is required.

## Scope

In scope:

- One input: a 100-row CSV fixture (`data/records.csv`).
- One transformation (T1): email normalization.
- One validation rule (V1): normalized-email shape check.
- One output (O1): a duplicate report (`output/duplicates.json`).

Out of scope: additional transformations, additional validation rules,
additional outputs or report formats, databases, network services, and
streaming or incremental detection.

## Input contract

- Path: `data/records.csv`, UTF-8, LF line endings, comma-separated with a
  header row.
- Header must be exactly: `id,name,email`.
- Exactly 100 data rows. No blank lines (a blank line is any empty line;
  the single trailing LF that terminates the final row is not a blank line).
- The file ends with exactly one trailing LF.
- Conforming fields contain no commas, double quotes, or line breaks, and
  every row has exactly three columns. Parsing is a plain comma split per
  line; no quoting or escaping is defined.
- Only conforming fixtures are in scope. Behavior on any input-constraint
  violation (wrong header, row count other than 100, blank lines, extra
  columns, or fields containing commas, quotes, or line breaks) is
  unconstrained: no exit status, error signal, or output-file guarantee is
  specified.
- `id`: non-empty string, unique within the fixture.
- `name`: arbitrary string (may be empty; never affects detection).
- `email`: raw string (may contain leading/trailing whitespace or mixed case;
  may be empty).

## Transformation T1 (the only transformation)

Normalize `email` per row:

1. Remove leading and trailing spaces and tabs.
2. Convert ASCII `A-Z` to lowercase. All other characters are unchanged.

The normalized value is used for validation and duplicate grouping. The raw
value is never used after this step.

## Validation rule V1 (the only validation rule)

A row is **valid** if and only if its normalized email contains exactly one
`@` character with at least one character before it and at least one
character after it. Every other row is **invalid**.

Invalid rows are counted and excluded from duplicate grouping: they never
form groups and never join a group.

## Duplicate definition

Two valid rows are duplicates of each other if and only if their normalized
emails are equal strings. A duplicate group is a maximal set of valid rows
sharing one normalized email, reported only when the set has size 2 or more.

## Output contract O1 (the only output)

Write one JSON report to `output/duplicates.json` (UTF-8, LF):

```json
{
  "total_rows": 100,
  "valid_rows": 97,
  "invalid_rows": 3,
  "duplicate_groups": [
    {"key": "a@example.com", "ids": ["r002", "r017"]},
    {"key": "b@example.com", "ids": ["r005", "r009", "r041"]}
  ]
}
```

Rules:

- `total_rows` is always 100; `valid_rows + invalid_rows` equals 100.
- `duplicate_groups` contains one entry per duplicate group and no entries
  for unique or invalid rows.
- `key` is the shared normalized email.
- `ids` lists the `id` values of the group's rows, sorted ascending by
  Unicode codepoint (e.g. `r10` sorts before `r2`).
- Groups are sorted ascending by `key` under the same Unicode codepoint
  ordering.
- The report contains no other fields and no other files are written.
- Byte encoding is canonical: UTF-8 with LF line endings and exactly one
  trailing LF; 2-space indentation with no tabs and no trailing whitespace,
  exactly as shown; object keys appear in the fixed order shown
  (`total_rows`, `valid_rows`, `invalid_rows`, `duplicate_groups`, and
  within each group `key` then `ids`); string escaping follows JSON
  (RFC 8259) with non-ASCII characters written literally as UTF-8.

## Determinism

The same fixture bytes always produce byte-identical report bytes under the
canonical encoding above. Ordering follows the sorting rules above,
independent of input row order.

## Boundary cases

- Case/whitespace variants (`"A@X.com"`, `" a@x.com "`) normalize equal and
  group together.
- Empty normalized email is invalid, never grouped.
- `a@b@c`, `@b.com`, `a@` are invalid (zero or two `@` signs, or an empty
  side).
- All-unique fixture yields `"duplicate_groups": []` with correct counts.
- All rows sharing one normalized email yield one group with all valid ids.
- `name` differences never split or merge groups.

## Acceptance criteria

1. Given the 100-row fixture, the tool writes only
   `output/duplicates.json` with `total_rows` 100 and consistent counts.
2. Rows differing only by email case or surrounding whitespace are grouped.
3. Rows failing V1 are counted as invalid and appear in no group.
4. Group and id ordering is sorted as specified on every run.
5. Two consecutive runs on the same fixture produce identical bytes.

## Unresolved requirements

None. This contract is implementable as written.
