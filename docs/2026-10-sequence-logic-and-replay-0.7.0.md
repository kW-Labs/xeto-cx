# 0.7.0: Sequence logic and replay, design rationale

0.7.0 changes how `cx.sequences` treats a sequence of operation. Through
0.6.0 the family modeled a sequence in text: citable sources and sections,
atomic statements with `trigger` and `response` prose, and a source-level
parameter registry applied to equipment through `SequenceInstance` and
`SequenceParamValue`. From 0.7.0 logic is no longer modeled as text.
Executable control logic has its own standard, ANSI/ASHRAE 231 (the
control description language, CDL, and its JSON-LD exchange form, CXF),
and this family points at it.

What stays is citation: `SequenceSource`, `SequenceSection`,
`SequenceSectionRole`, and `SequenceStatement` trimmed to a citation
anchor. What replaces the rest is a reference to the executable logic
(`SequenceLogic`) and evidence for its parameters (`SequenceLogicParam`).
Two records prove the running building against the logic on the
performance plane: `ReplayCriterion` in `cx.requirements` and
`SequenceReplayTest` in `cx.verification`. Their outcome is defined step by
step below, on LBNL's funnel algorithm.

The change is breaking. The family moves from 0.6.0 to 0.7.0, and the
migration note is below. This document supersedes the `cx.sequences`
section of `2026-07-standards-alignment-0.3.0.md`, which stays as written
as the record of what 0.3.0 decided.

## Why the text model of the logic goes

1. **231 already defines the executable language.** Its elementary blocks
   are defined by Clause 7 and used as defined (5.6). A Xeto copy of
   Clause 7 inside this family would fork a published, licensed standard.
2. **Text logic and executable logic cannot both be the truth.** `trigger`
   and `response` prose beside a CDL class is a second statement of the
   same behavior, and nothing keeps the two in step. A sequence born in
   CDL needs no translation from English, and the open schema should not
   invite one back.
3. **The family's own rule.** Published guideline content is referenced by
   citation, never embedded. Citations survive the change intact.
4. **Upstream ambition.** A block library belongs in its own library,
   maintained by whoever maintains CDL, not in a commissioning ontology
   meant for `ph.cx`.

What this family keeps owning is the join: which requirement a cited
guideline section serves, which logic implements it on which equipment,
where each parameter value came from, and how the running building was
proven against it.

`SequenceSectionRole` keeps all seven members. The roles describe the
anatomy of the guideline document. Commissioning cites and verifies alarm
and fault-detection sections too, but 231 Appendix A1 excludes alarms and
fault detection from CDL's scope unless they are part of the control
logic. So a CDL `SequenceLogic` normally does not list such sections, and
requirements on them are verified by functional test steps or
`PerformanceCriterion` records.

## The chain

```
Requirement ◀─ requirementRef ─ SequenceStatement ─ sectionRef ─▶ SequenceSection ─ sourceRef ─▶ SequenceSource
     ▲                                                                   ▲                              ▲
     │ requirementRef                                                    │ sectionRefs                  │ sourceRef
ReplayCriterion ─ logicRef ─▶ SequenceLogic ─────────────────────────────┴──────────────────────────────┘
                                 ▲     ▲   └─ implRef ─▶ a tool's own record of the logic
                       logicRef  │     │ logicRef (pinned revision)
                                 │     │
                    SequenceLogicParam   SequenceReplayTest ◀─ activityRef ─ VerificationResult
```

The spine edge keeps its direction, child to parent:
`SequenceStatement.requirementRef`. A statement anchors a requirement to a
cited guideline clause, and nothing more. A logic names the guideline it
implements (`sourceRef`) and the sections it covers (`sectionRefs`), so
statement to logic is a join on section. `ReplayCriterion.logicRef` closes
the chain from the requirement side. `implRef` points from the open record
to a tool's record; no pointer runs the other way, so a tool's logic
library needs no reference into `cx`.

Two invariants are stated in the doc comments. A replay test's `equipRef`
is in its logic's `equipRefs`. Its `logicRef` is the criterion's
`logicRef` or a revision that supersedes it, walking `supersedesRef`.
`fits()` cannot check a cross-record invariant, so a consumer checks both
on read.

## SequenceLogic: a neutral record, not a bare Ref or URI

| Option | For | Against |
|-|-|-|
| A bare `Ref` slot to a tool's record | Simple; store-native | Names one format in a neutral lib. Means nothing outside the store that minted it. Carries no version. |
| A bare `Uri` slot | Portable; 231 6.5 defines an instance URI form | Opaque. Cannot be joined or validated. Carries no hash and no library version. |
| **A neutral record holding both** | Carries the identity a result must pin. Holds an optional local Ref for joins in one store. One record per revision. Names no format in its type. | One more type |

`SequenceLogic` also takes over `SequenceInstance`'s job: it says what
equipment the logic is applied to (`equipRefs`, `systemRef`), for which
project, and which guideline sections it implements. The library version
matters. A block library's implementation of a guideline and the
guideline's own text can diverge, so a pass against one library version is
not a pass against a later errata reading of the guideline.

One CDL class applied to ten AHUs is one record with one hash; a parameter
value that differs per unit carries its `equipRef` on the parameter
record. A design whose per-unit values are written into the class itself
has one class per unit, so one record per unit, and the hashes differ.

## SequenceLogicParam: a new spec, not SequenceParamValue reshaped

1. **It is a merge, not a rename.** The record holds what `SequenceParam`
   held (who supplies it, the placeholder flag, the unit, the measurement
   procedure) and what `SequenceParamValue` held (the value and when it
   was resolved). Keeping either old name would describe half of it.
2. **Its identity changed.** `SequenceParamValue` was keyed by
   `(instanceRef, paramRef)`. The new record is keyed by
   `(logicRef, logicParam, equipRef?)`, with `logicParam` spelled as the
   logic spells it. No existing instance fits the new shape.
3. **It represents the gap.** An unresolved placeholder is a
   `SequenceLogicParam` with `placeholder` and no `val`. A record named
   "value" that exists precisely when there is no value misleads every
   reader.
4. **Stale data fails loudly.** A new marker means a consumer querying
   `sequenceParamValue` finds nothing after the upgrade and has to
   migrate. A reused marker would match old records of an incompatible
   shape and fail silently.

The record carries a guideline crosswalk (`sourceSymbol`, the guideline's
own symbol, and `definedIn`), the supplier (`providedBy`, the designer /
TAB / controls partition of Guideline 36 section 3), the placeholder flag,
the value (`val`, `unit`, `resolvedAt`, and `procedure` for a
field-measured value), and its evidence: `valueBasis`, `sourceDocRef`,
`sourcePage`, `sourceLocator`, and `requirementRef` when an owner
requirement dictated the value.

There is no `resolvedBy`, because a record may carry only one `CxRole`
slot (a Dict holds at most one marker of a choice). The supplier is
`providedBy`; confirmation by another party is an `Attestation` on the
record. `SequenceParam.defaultVal` has no slot: a published default is
recorded as `val` with basis `sourceDefaultValue`. `SequenceParamKind` has
no replacement: the logic declares each parameter's type, and the
partition that mattered for commissioning survives as `providedBy`.

Parameter records are per logic revision. A new revision gets its own
records, and a tool copies the unchanged ones forward. That keeps the
replay's parameter gate and `paramSetHash` deterministic.

Completeness is layered. This family cannot force a tool to write a record
for every fill-in the logic has. A recorded placeholder with no value fails
the replay's parameter gate (step 2 below). A parameter the tool never
recorded and the logic never bound fails the engine, because 231 6.6 makes
an unbound parameter in an evaluated expression an error, so the result is
inconclusive by step 6 rule 3.

Glue parameters that only the logic has, and that the commissioning record
has no reason to track, need no record. The logic carries their bindings.

### The value basis

| Member | Marker | Meaning |
|-|-|-|
| `ValueFromDesign` | `designValue` | Stated in the contract documents or the design file. It makes no claim about how the value was captured. |
| `ValueFromOpr` | `oprValue` | Dictated by an owner requirement; `requirementRef` names it |
| `ValueFieldMeasured` | `fieldMeasuredValue` | Measured in the field per the guideline's procedure, e.g. TAB |
| `ValueSourceDefault` | `sourceDefaultValue` | The guideline's or block library's published default; nobody set it for this project |
| `ValueImplementerSet` | `implementerSetValue` | Chosen by the implementer with no document behind it, including field tuning |

### Owed values for a sequence with no logic

For an existing building the sequence often lives only on a drawing. Its
sheet is a `SequenceSource` (the widened doc comment covers a project's own
contract-document sequence, `docId` the sheet number), its headings are
`SequenceSection` records, and each statement is a `SequenceStatement`
whose `narrative` is the drawing's text verbatim, because the project owns
it. Points belong to Haystack and `ph`. A dangling cross-reference on the
drawing is a `CxIssue`.

Parameter evidence is keyed to a logic's parameter names, so a
drawing-only sequence has no key for it. Through 0.6.0 an owed setpoint on
such a sequence was a `SequenceParam` with `placeholder` and no value.
From 0.7.0, **each owed value of a sequence with no executable logic is
tracked as a `CxIssue`** until a logic exists for it. The promise that
owed setpoints are tracked as open items still holds, for every sequence:
as a placeholder `SequenceLogicParam` where there is a logic, and as a
`CxIssue` where there is not. Such an issue names the fill-in through its
existing slots: `sourceRef` to the `SequenceStatement` or `SequenceSection`
that prints it, `assignedRole` the party who owes the value, and
`requirementRef` the requirement it serves, when there is one.

A drawing-only sequence cannot be replayed until a logic exists for it. It
is verified as before: functional test steps generated from its
statements, or `PerformanceCriterion` records on outcomes. A logic a tool
proposes from the drawing makes it replayable. That logic has no honest
`authorRole`, because no `CxRole` is a tool, so `authorRole` stays absent,
`authoredBy` names the proposer in text, and the logic stays unreviewed
until a person signs an `Attestation` on it.

The installed program of an existing building is never modeled. It is the
subject under test, named by the test's `equipRef` and the result's
`subjectRef`.

## Sequence replay verification

Three records, each with one job:

1. **`ReplayCriterion`** (in `cx.requirements`): what the requirement
   demands of a logic's output, stated before any test. A record naming
   `logicOutput` grades that output. A record with no `logicOutput` holds
   defaults only and is never graded.
2. **`SequenceReplayTest`** (a `VerificationActivity` in
   `cx.verification`): one run on one unit (its `equipRef`). It pins the
   logic revision (`logicRef`), which parameter source fed the reference
   (`paramSource`) and a digest of the exact parameter set
   (`paramSetHash`). It names the engine (`referenceEngine`) and flags a
   controller generated from the same logic (`compiledFromLogic`).
3. **`VerificationResult`**: one per graded output per run, with
   `activityRef`, `criterionRef` to that output's `ReplayCriterion`,
   `subjectRef` to the unit, `plane: performancePlane`, `mode: autoMode`,
   the window (`evalStart`, `evalEnd`, new in 0.7.0), `adherencePct`,
   `coveragePct` (new in 0.7.0) and `evidenceRef` to the evidence package.

**Why not `PerformanceCriterion`.** It compares a point to a static band. A
replay compares a point to a time-varying computed reference with time and
value tolerances, and needs the reference's identity.

**Why an activity.** A replay depends on configured trends, a window
covering the season the sequence acts in, and the logic revision in
force: prerequisite, scheduling and identity facts an activity already
holds.

**Ongoing Cx.** A replay anchors like any `VerificationActivity`, on
exactly one of `cxProjectRef` or `ocxProgramRef`. A replay of a
commissioning project is filed under its `CxProject`. A replay the
ongoing-Cx program re-runs (202 section 3) anchors on the `OcxProgram`
instead, which also covers a facility with no `CxProject` at all.
`ocxProgramRef` landed after 0.7.0 and rides the next family release. In
0.7.0 itself, `cxProjectRef` is still required.

**Status rollup.** `activityPassed` when every graded result passed,
`activityFailed` when any failed, otherwise `activityDeferred`: the proof
waits for data, and the re-run is a new activity with `retestOf`.
`CxActivityStatus` has no inconclusive member; issue #15 is where one would
go.

**Which parameters.** The acceptance replay runs on the evidenced design
values (`designParamSource`), proving the building against the design. A
replay on the values read from the controller (`controllerParamSource`) is
a separate diagnostic run, and a mismatch between the two value sets is a
conformance-plane finding, not a replay outcome.

### Which plane

The performance plane. The family defines a plane by its evaluation
mechanism, and performance is "evaluated against telemetry over a span". A
replay reads trended inputs and outputs over a window. It is not
conformance: no record shape is checked with `fits()`. It is not
attestation: no signoff decides it, though `witnessRef` records a
witnessed review. There is no fourth plane. The difference from a band
check is the criterion's form, the way `SpecConstraint` is the conformance
form.

**What a pass proves.** The implementation matched the reference over the
inputs the window exercised. That is verification, in the sense the OBC
working report separates from validation (12.1). It does not prove the
owner's outcome: whether a reset saved energy is a separate question, and
the designer's claim lives in the `BodResponse`. It does not prove branches
the window never exercised; `notes` say which modes were exercised. With
`compiledFromLogic`, a pass tests the translator and runtime, not an
independent implementation.

### The replay contract

Normative. Two independent implementations must compute the same outcome
from this contract alone. The comparison adopts LBNL's funnel
(`lbl-srg/funnel` at commit `a12fa969`) exactly; every choice funnel leaves
open is named.

**Inputs.** One `SequenceReplayTest` T (logic revision L, unit E = T's
`equipRef`, parameter source, window [`evalStart`, `evalEnd`)), the
non-superseded `ReplayCriterion` records whose `requirementRef` is the
requirement and whose `logicRef` is L or a revision L supersedes, and the
trended series.

**Step 1. Resolve criteria.** At most one defaults record (no
`logicOutput`) and at most one per-output record per `logicOutput`; more is
an invalid set and every output is inconclusive. Only outputs with a
per-output record are graded. Each slot takes the per-output value when
set, else the defaults record's, else the built-in default: funnel's for
tolerances (all 0), `adherenceTarget` 100, `minCoverage` 90, `maxGap`
600 s, `referenceStep` 1 s. The last three are judgement values, not
measured.

**Step 2. Parameter gate (policy).** For each distinct `logicParam` among
L's `SequenceLogicParam` records, the effective record for E is the one
with `equipRef` E, else the one with no `equipRef`. Two candidates at the
same level is an invalid set: inconclusive. The parameter is unresolved
when its effective record carries `placeholder` and either has no `val` or
has basis `sourceDefaultValue`. Any unresolved parameter makes every output
inconclusive. This is a policy, not a computability fact: a CDL logic
carries its own bindings and could be simulated anyway. An owed value
blocks the proof instead of being defaulted. The gate is per logic, not
per output: a record cannot tell which outputs a parameter reaches without
the logic's graph, so the policy is fail-closed on purpose.

**Step 3. Build the series.** Time is seconds from `evalStart`.

- Inputs: each input of L from its trend, converted to L's units, held
  between samples (231 5.12(a)).
- Parameters: with `designParamSource`, the effective records' values from
  step 2; with `controllerParamSource`, the values read from the
  controller under test (OBC working report, Scenario 2 step 5).
  `paramSetHash` digests whichever set was used.
- Reference output: the engine's value at every test sample time and on a
  uniform `referenceStep` grid, from `evalStart` to `evalEnd` inclusive.
- Test output: the output's trend samples in the window, in time order,
  equal stamps kept (OBC working report 12.2), converted to the
  criterion's `unit`. A sample is added at `evalStart` and at `evalEnd`
  with the last earlier value when none falls there, because funnel
  requires shared first and last x.
- Indicator: from `indicatorInput` (a Boolean input of L, from its trend)
  or `indicatorPointRef` (a trended Boolean point), held between samples.
  Compare where it is true, or where it is false with `indicatorInverted`.
  With neither slot, always on.
- Suspension, as Scenario 2 step 7 does it: where the indicator is off,
  both reference and test values are set to 0 before the comparison.

**Step 4. Funnel.** Build the funnel around the reference and compare the
test with funnel's `set_tube_size` (`src/tubeSize.c`) and its rectangle
algorithm (`src/compare.c`) at `a12fa969`. Half-width in time: `timeTolAbs`
(`atolx`). Half-height in value: the maximum of `valueTolAbs` (`atoly`),
`valueTolRange` × the zeroed reference's range (`rtoly`) and
`valueTolLocal` × the local reference magnitude (`ltoly`), with funnel's
zero-range fallback to magnitude and its 1e-10 floor. Relative time
tolerances are not offered: one depends on the time origin, the other on
the window length. Each test sample gets funnel's error,
`max(0, y − y_up) − min(0, y − y_low)` against the envelope interpolated at
the sample's time; it is inside when the error is 0.

**Step 5. Weight, coverage and share.** funnel defines no share or pass
threshold, so this step is the family's, matched to the existing slots
("percent of span within criteria", "percent of time in range"). Split the
window at every test, input and indicator sample time.

- A piece is *suspended* when the indicator is off.
- A piece is *missing* when its governing test sample (the latest at or
  before the piece's start) is more than `maxGap` seconds from the next
  test sample or from `evalEnd`, when any input's governing sample is, or
  when an input has no sample at or before the piece.
- Every other piece is *evaluated*, and *inside* when its governing test
  sample is inside.
- `coveragePct` = evaluated time / unsuspended time × 100. `adherencePct` =
  inside time / evaluated time × 100.

A held sample is judged at its own timestamp only. The trend prerequisite
bounds how far a held value can drift from a moving reference unseen.

**Step 6. Outcome, in precedence order.**

1. Invalid criteria set: inconclusive.
2. Parameter gate failed, or an invalid parameter set: inconclusive.
3. The engine could not compute the reference, including an unbound
   parameter (231 6.6): inconclusive.
4. An input of L or the graded output has no sample in the window:
   inconclusive.
5. Unsuspended time is zero: inconclusive.
6. `coveragePct` < `minCoverage`: inconclusive.
7. Pass when `adherencePct` ≥ `adherenceTarget` and, at a target of 100,
   no evaluated test sample has a nonzero error (which makes a 100 target
   identical to funnel's own verdict and catches zero-duration samples).
   Otherwise fail.

Inconclusive wins over a failing share at low coverage. The result still
records the two percentages it computed, and `notes` say what was missing.
A missing trend is not a passing grade, and it is not a failing one
either.

For example, a two-week window in which the supply fan was proven on for
200 hours, with a 30-hour trend outage on the setpoint, evaluates 170 of
200 unsuspended hours: `coveragePct` is 85, below the default
`minCoverage` of 90, so the result is inconclusive by rule 6 whatever its
`adherencePct`.

### Pinning, evidence and re-running

- **`logicHash`** (required) is `"<algorithm>:<lowercase hex>"`, sha256
  unless stated, over the logic **as authored**: one file's bytes, or the
  digest of a manifest of lines `<path relative to the package root>
  <sha256 hex>`, sorted bytewise by path, UTF-8, LF-joined. For CDL, the
  design's `.mo` source, excluding the block library (`libraryName`,
  `libraryVersion`). An exchange export the engine read is named by
  `translator` and `translatorVersion` (231 Table 6-2).
- **`logicUri`**, when set, is absolute. A project-relative path is not a
  `logicUri`.
- **`paramSetHash`** is `"sha256:<hex>"` of lines `<logicParam>=<value>`,
  one per parameter fed to the reference, sorted bytewise, UTF-8,
  LF-joined, with no trailing LF. The value is written as recorded, before
  any conversion to the logic's units: in the `unit` of its
  `SequenceLogicParam` record with `designParamSource`, or as read from
  the controller with `controllerParamSource`. It is written in Zinc
  scalar encoding with its unit, a number in its shortest round-trip
  decimal form with no exponent and no trailing zeros (`65°F`, not
  `65.0°F`). The seed's July replay shows one, computed over its two
  parameter values.
- **Criteria are immutable** once a result references them; a change
  writes a new `ReplayCriterion` with `supersedesRef`.
- **`evidenceRef`** points at the evidence package: input, output and
  indicator series in the converted units, and the parameter set as
  hashed
  (the content of Scenario 2's `reference_input.csv`,
  `controller_output.csv`, `indicator.csv` and parameter file).

With the logic and the evidence package, a second implementation re-runs a
result from step 4 without the producing tool. Re-extracting the series
from the building needs the tool's input binding (which trended point
feeds which logic input), which stays out of this family for now.

## Alignment with Guideline 0, Standard 202, 231 and Guideline 36

- **The replay is a verification activity like any other**: actual
  responses per check (Guideline 0 7.2.10.1g), retest lineage and witness
  signoff (202 Appendix J5.1c), deferred or seasonal (202 16.2.2), ongoing
  Cx (202 section 3) anchored on the project or the program, as above. A
  failure raises a `CxIssue`; an accepted divergence is a
  `RequirementVariance` (202 Appendix O).
- **Trending is a commissioning prerequisite, because 231 excludes it**
  (Appendix A1). The replay needs trends of every input and output,
  recorded as an `ActivityPrerequisite`. CDL can declare trend intent per
  connector through an informative point-list annotation (OBC working
  report 7.7.5).
- **Alarms and fault detection are not replayed against CDL** (Appendix
  A1). `SequenceSectionRole` keeps those roles to describe the guideline;
  their requirements keep functional test or performance-criterion
  verification.
- **No 231 compliance or verification claim.** Conformance classes are
  informative (A1.8) and nothing is certified; `logicStandard` describes a
  format. 231 specifies no normative verification procedure (Appendix A2
  is informative); the replay contract is funnel plus the choices named
  above.
- **Guideline 36 stays a cited source.** A logic names it through
  `sourceRef` and names its block library separately.
- **The change narrows the record, not the process.** Guideline 0 and 202
  ask that sequences be reviewed and tested against the design; they do
  not ask the commissioning record to restate the logic in prose.
  Statements stay as the citation anchors those reviews and tests point
  at.

## The seed example

`cx.content` shows the whole chain on the seed building. OPR-3.3.1
(`@ex-req-sat-reset`) requires the AHU's supply air temperature setpoint to
reset per trim and respond, citing Guideline 36 5.16.2.2, and moves to the
performance plane: it is proven by replay. The functional test step
`@ex-fpt-s04` keeps citing it as supplementary field evidence, because it
exercises branches under forced conditions that trends may never reach. A
failed step raises a `CxIssue`; it does not write the requirement's
`VerificationResult`.

The engineer's design for AHU-1 is one CDL class, extending the multiple-
zone VAV controller of the Modelica Buildings Library 12.1.0 and covering
5.16.2.2 and 5.16.1 (`@ex-seq-logic`). Its two parameter records carry the
guideline's fill-ins: `Max_ClgSAT` (`TSupCoo_max`, designer, from the
controls drawings) and `Max_DSP` (`pMaxSet`, TAB, measured in the field on
July 10).

A June replay ran before TAB measured `Max_DSP`. The logic consumes it, so
the parameter gate made the June result inconclusive: the owed value
blocked the proof. A July retest, after the TAB report, passed with 98.6
percent coverage. The parameter record shows today's resolved state; the
June result's `notes` say what was owed.

## Names

Every new slot and marker name was checked against the slot and global
declarations of the `ph` family and every other library a 4.0.6 Xeto
toolchain ships. None collides. A first draft's `output` collided with ph's
`*output` marker and became `logicOutput`. Reused names keep their
Haystack meaning on purpose: `equipRef` and `siteRef` are ph globals of
type `Ref`, and `val` a global `Obj`. `systemRef` is a `MultiRef` global in
ph, and the family declares it `Ref?`, the narrower single-ref case, as it
already did on `VerificationActivity` and `CxIssue`. `unit: Str?` against
ph's `*unit: Unit` is a collision that predates 0.7.0 (`SequenceParam` and `PerformanceCriterion`
already declared it) and is recorded here rather than fixed in this
release.

## Migration from 0.6.0

| Removed in 0.7.0 | Replaced by |
|-|-|
| `SequenceStatement.trigger`, `.response` | Nothing in this family. Behavior is the logic's (`SequenceLogic`). Prose stays in the source document, or in the narrative where the text is owned. |
| `SequenceInstance` | `SequenceLogic`. `dis`, `sourceRef`, `cxProjectRef`, `systemRef`, `sectionRefs`, `notes` carry over. `equipRef` becomes `equipRefs` (a list). New and required: `logicFormat`, `logicHash`. |
| `SequenceParam` | `SequenceLogicParam`. `symbol` becomes `sourceSymbol`; `dis`, `definedIn`, `providedBy`, `unit`, `placeholder`, `procedure` carry over; `sourceRef` gives way to `logicRef` (the logic's `sourceRef` names the guideline); `defaultVal` becomes `val` with `valueBasis: sourceDefaultValue`; `paramKind` is dropped. New and required: `logicParam`. |
| `SequenceParamValue` | `SequenceLogicParam`, the same record. `(instanceRef, paramRef)` becomes `(logicRef, logicParam, equipRef?)`. `val`, `resolvedAt` carry over. `resolvedBy` folds into `providedBy` (one `CxRole` slot per record); confirmation by another party is an `Attestation`. |
| `SequenceParamKind` and `SetpointParam`, `TuningParam`, `SelectionParam`, `ScheduleParam` (markers `setpointParam`, `tuningParam`, `selectionParam`, `scheduleParam`) | Nothing. The logic declares each parameter's type; the provider partition survives as `providedBy`. |
| `CxEntity` markers `*sequenceParam`, `*sequenceInstance`, `*sequenceParamValue` | `*sequenceLogic`, `*sequenceLogicParam` |
| Seed `@ex-seq-inst`, `@ex-seq-param-maxsat`, `@ex-seq-param-maxdsp`, `@ex-seq-pv-maxsat` | `@ex-seq-logic`, `@ex-seq-lp-maxsat`, `@ex-seq-lp-maxdsp` |

Added in 0.7.0: `SequenceLogic`, `SequenceLogicParam`, the
`ParamValueBasis` and `SequenceLogicFormat` choices (`cx.sequences`);
`ReplayCriterion` (`cx.requirements`); `SequenceReplayTest`, the
`ReplayParamSource` choice, and `VerificationResult.evalStart`, `evalEnd`
and `coveragePct` (`cx.verification`); and the `CxEntity` markers
`*sequenceLogic`, `*sequenceLogicParam`, `*replayCriterion` and
`*sequenceReplayTest`. `@ex-req-sat-reset` moves from `attestationPlane` to
`performancePlane`.

Also in 0.7.0, and not about sequences: all seven lib pragmas drop
`hxSysOnly`, so this repo's own build artifacts can be enabled per project
(see `CONTRIBUTING.md`).

**Record migration recipe**, for anyone holding 0.6.0 records:

1. For each `SequenceInstance` with an executable logic, write a
   `SequenceLogic` from the instance plus the logic's identity.
2. For each `SequenceParam` of that instance's source that has a value for
   the instance or is a placeholder, write a `SequenceLogicParam` with the
   logic's own name for it. That crosswalk (guideline symbol to logic
   parameter) is known only to the tool that holds the logic.
3. A `SequenceInstance` with no executable logic cannot migrate: there is
   no hash to identify. Its owed placeholders become `CxIssue` records (see
   "Owed values for a sequence with no logic" above), and its
   equip-to-section application is lost.
4. Drop `trigger` and `response` from statements.

**Known holders of 0.6.0 sequence records: none.** The one known consumer
of this family holds none. No other consumer is known; that is an absence
of evidence, not a proof.

## Deliberately deferred

- **One statement, several requirements.** `requirementRef` stays single;
  a list would be a further break.
- **Input binding** (which trended point feeds which logic input). A
  result re-runs from the evidence package without it.
- **Capture method** (entered versus extracted). `ParamValueBasis` names
  authority, not capture.
- **As-found values.** Whether `ParamValueBasis` gains an `asFoundValue`
  member.
- **Coverage of behavior.** A typed measure of modes or branches
  exercised. `coveragePct` measures data, not behavior.
- **Static logic checks.** Whether `SequenceLogicParam` values feed a
  conformance-plane criterion, e.g. a design value checked against an OPR
  limit.
- **Statement page provenance.** A statement taken from a drawing cites
  its page only through its source's `docRef`; per-statement
  `sourceDocRef` and `sourcePage` can be added if a consumer needs them.
- **A per-output parameter gate.** It needs the logic's dependency graph,
  which this family does not hold.
- **A project-to-program link.** An activity carries exactly one anchor,
  so a retro-Cx project running under a standing program does not anchor
  its activities on both. The link between that project and its program
  belongs on the project, and this family does not declare it yet.

## Sources

- ANSI/ASHRAE Standard 231, cited by clause number only: 5.6, 5.12(a),
  6.5, 6.6, Table 6-2, Clause 7, Appendix A1, A1.8 and A2.
- LBNL funnel, `lbl-srg/funnel` at `a12fa969`: `README.md`,
  `src/tubeSize.c` (`set_tube_size`) and `src/compare.c`.
- OBC working report: 7.7.5, 12.1, 12.2 and 12.7.3 (Scenario 2).
- Modelica Buildings Library 12.1.0,
  `Buildings.Controls.OBC.ASHRAE.G36.AHUs.MultiZone.VAV.Controller`, for
  the seed's parameter, input and output names.
- ASHRAE Guideline 0, Standard 202 and Guideline 36, at the clause numbers
  this family already cites (see the 0.3.0 rationale).
