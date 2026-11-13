# Dashboard-as-code validator — Requirements

This document is the complete public requirements contract for the project
goal:

> Build "Dashboard-as-code validator" from generated telemetry and produce
> one actionable result in a CLI or static page.

An implementation satisfies the goal when it meets every numbered requirement
below. Nothing outside this document is required to implement the goal.

## 1. Scope

1.1. The validator checks one dashboard definition stored as code against one
generated telemetry data set.

1.2. The validator produces exactly one actionable result per run: a single
verdict with a single primary action for the dashboard owner.

1.3. The result must be delivered through at least one of these surfaces:
a command-line interface (CLI) or a static page. Both surfaces, when
provided, must present the same result content defined in section 4.

1.4. Non-goals: live telemetry collection, dashboard rendering or hosting,
alert dispatch, authentication, and network access are not required.

## 2. Public inputs

2.1. The validator accepts exactly two input files, each a UTF-8 JSON
document:

- (a) a dashboard definition file,
- (b) a telemetry data file holding generated samples.

2.2. Dashboard definition file. A JSON object with:

- `title`: non-empty string, the dashboard name.
- `panels`: non-empty array of panel objects. Each panel has:
  - `name`: non-empty string, unique within the file.
  - `metric`: non-empty string, the telemetry metric the panel observes.
  - `operator`: one of `">"`, `"<"`, `">="`, `"<="`.
  - `threshold`: a finite number.

2.3. Telemetry data file. A JSON object with:

- `samples`: an array (possibly empty) of sample objects. Each sample has:
  - `metric`: non-empty string.
  - `timestamp`: a string.
  - `value`: a finite number.

2.4. Samples whose `metric` is not referenced by any panel are ignored.

2.5. A panel whose `metric` has no samples in the telemetry file is
evaluated as a violation (there is no data to confirm the panel's
condition).

## 3. Evaluation behavior

3.1. For each panel, the validator takes the maximum `value` across all
samples with that panel's `metric` and applies the panel's `operator` and
`threshold` to that maximum. When no sample carries that panel's `metric`,
no maximum exists; the panel is a violation per 2.5 and its `observed_max`
is `null` (see 4.1).

3.2. A panel passes when the comparison in 3.1 holds; otherwise the panel
is a violation.

3.3. When no panel is a violation, the verdict is `PASS`. When at least one
panel is a violation, the verdict is `FAIL`.

3.4. Exactly one violation is reported as the primary finding: the first
violating panel in the dashboard file's `panels` array order. Remaining
violations are counted but not individually listed.

3.5. The primary finding carries one action: a human-readable remediation
sentence that must contain the primary panel's `name` substring, the
panel's `operator` and `threshold` substrings (for example `">"` and
`"500"`), and either the decimal form of `observed_max` when it is a
number or the literal substring `no data` when `observed_max` is `null`.
Surrounding prose is unrestricted.

## 4. Result content

4.1. Every run that successfully reads both input files emits one result
record with exactly these fields:

- `verdict`: `"PASS"` or `"FAIL"`.
- `dashboard`: the dashboard `title`.
- `panels_checked`: number of panels evaluated.
- `violations`: number of violating panels.
- `primary_finding`: `null` when the verdict is `PASS`; otherwise an
  object with exactly these four fields: `panel` is the violating panel's
  `name`, `metric` is that panel's `metric`, `observed_max` is the
  section-3.1 maximum for that panel (a finite number) or `null` when the
  panel had no matching samples per 2.5, and `action` is the section-3.5
  remediation sentence.
- `telemetry_samples`: total length of the input `samples` array,
  regardless of section-2.4 ignoring of unreferenced metrics.

`panels_checked`, `violations`, and `telemetry_samples` are non-negative
integers.

4.2. Runs that cannot read or parse the inputs, or whose inputs violate
the shapes in 2.2–2.3, emit no result record. They report a usage error
instead (see 6.1) and produce no verdict.

## 5. Delivery surfaces

5.1. CLI surface (when provided):

- The CLI takes exactly two positional file paths in this fixed order:
  the first path is the dashboard definition file and the second path is
  the telemetry data file.
- It writes the section-4 result record to stdout as JSON and nothing
  else to stdout.
- Exit status is `0` when the verdict is `PASS`, `1` when the verdict is
  `FAIL`, and `2` for usage errors under 4.2.

5.2. Static-page surface (when provided):

- The implementation writes a single self-contained page that displays
  every field of the section-4 result record with no required user
  interaction.
- The page must be viewable by opening the file directly, with no server
  or network access.

5.3. At least one of the two surfaces must be provided. When both are
provided, both must report the same verdict, counts, and primary finding
for the same inputs.

## 6. Boundary cases

6.1. Missing file, unreadable file, invalid JSON, or a JSON document that
violates the shapes in 2.2–2.3 is a usage error: on the CLI it exits
with status `2` and a one-line error on stderr; the static-page surface
(if it is the only surface) must display the one-line error text with no
result fields and no required interaction instead of a result. No partial
verdict is emitted.

6.2. An empty `samples` array is valid input: every panel is then a
violation per 2.5, so the verdict is `FAIL` (for a non-empty dashboard).

6.3. Extra JSON fields not named in this document are ignored by the
validator and must not change the verdict.

6.4. Rerunning the validator on the same two input files produces an
identical parsed result: the same verdict, counts, and primary finding
fields with a byte-identical `action` string. Result content must not
embed timestamps, random values, or absolute paths. JSON whitespace or
key-order differences in the CLI output are not failures; only
parsed-content identity plus deterministic `action` text is required.

## 7. Acceptance checklist

An implementation is accepted when a developer can demonstrate, using only
this document:

- [ ] A dashboard file and a telemetry file in the section-2 shapes are
  accepted and evaluated per section 3.
- [ ] A passing data set yields verdict `PASS` with `primary_finding`
  `null`; a violating data set yields verdict `FAIL` with exactly one
  primary finding chosen per 3.4.
- [ ] The result record contains exactly the section-4 fields.
- [ ] At least one delivery surface behaves per section 5, including the
  CLI exit statuses `0` / `1` / `2` when the CLI is provided.
- [ ] Every boundary case in section 6 behaves as stated, including
  deterministic reruns per 6.4.

## 8. Deferred decisions (not blocking)

- Choice of implementation language, runtime, and dependency set is left
  to the implementation phase. The contract above constrains only
  observable inputs, outputs, and behavior.
- The CLI program name and the static-page file name are left to the
  implementation phase.
