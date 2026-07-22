# 0.3.0 — Standards Alignment: design rationale

0.3.0 aligns the ontology against the full commissioning standards library —
ASHRAE Guideline 0-2019, Standard 202-2018, Guideline 1.4-2019, Guideline
32-2018, Guideline 36-2024, Standards 62.1-2016 and 170-2013, and the BCxA
New Construction, EBCx, and Ongoing Cx best-practice guides — and adds two
new libraries. The additions follow one test: is this an *artifact* of the
commissioning process (a record someone signs, verifies, or audits), not
process prose?

## Per-mode acceptance criteria (cx.requirements)

Real OPRs state space criteria as a room-type matrix: setpoint band,
tolerance, and adherence percentage, per occupancy mode, per quantity —
drybulb temperature, humidity, CO2, air changes, pressurization, with the
governing standard cited per row. Three additions make that matrix
expressible without a new record family:

- `OccupancyMode` (occupied / standby / unoccupied) on `PerformanceCriterion`.
  Standby is a real ASHRAE 170 mode the model previously could not state.
- `PressureRelationship` (positive / negative / neutral) for the 170-2013
  Table 7-1 pressure-relationship column, which is sometimes specified
  without numeric limits.
- `Requirement.citation` for external standard/code provenance, and
  `Requirement.spaceTypeKey` for vendor-neutral space-type scoping using the
  62.1/170 occupancy-category vocabulary — a scope key that survives tool
  and ontology boundaries where spec qnames cannot.

## Sequences of operation (cx.sequences)

Guideline 36-2024 structures a sequence as: a parameter fill-in contract
partitioned by provider (designer-scheduled, TAB field-measured, controls
computed — section 3), required points (section 4), and per-equipment logic
with citable subsection ids (section 5), plus built-in testing/commissioning
override hooks. It does not yet publish functional tests (they await a
future addendum), so a commissioning record system must *generate* FPTs
from sequence atoms. cx.sequences models exactly that anatomy:

- `SequenceSource` and `SequenceSection` carry identity and citation only —
  published prose is never embedded, and `stableAnchor` marks the ids
  Guideline 36 holds stable across addenda (to the third level).
- `SequenceStatement` is the atomic checkable behavior — one statement, one
  possible "no" — with optional verbatim text only where the institution
  owns it, and a `requirementRef` edge into the traceability spine.
- `SequenceParam` / `SequenceInstance` / `SequenceParamValue` make the
  fill-in contract first-class: a placeholder parameter is itself a
  requirement, and an instance's unresolved placeholders are auditable.

## Standard 202 / Guideline 0 alignment (all libs)

- **CxA → CxP.** 202-2018 Addendum a replaced "Commissioning Authority"
  with "Commissioning Provider" throughout; `CxRoleCxp { cxpRole }` follows
  (breaking rename from 0.2.0, per the pre-1.0 version policy). New roles:
  designer, construction manager, vendor/manufacturer, occupant.
- **Predesign exists.** The OPR is developed before design (202 section
  6.2.1, Guideline 0 section 5); the phase taxonomy now starts there. EBCx
  gains an initial-assessment phase (BCxA EBCx practice).
- **Acceptance is a record.** 202 requires formal Owner acceptance of every
  major deliverable (section 4.3); Guideline 0 Appendix H defines the
  recommend-then-accept flow. `AcceptancePlan` + `AcceptanceRecord` capture
  it, with roles carried by the referenced attestations.
- **Reports are documents.** `CxReport` (preliminary/final, 202 section 17)
  and `CxProgressReport` (recurring, including the End-of-Warranty report
  of 202 section 4.2.2) join the CxDoc taxonomy.
- **Tests record evidence per step.** Guideline 0 section 7.2.10.1g says
  check boxes alone "should be avoided" — `TestStepResult` records observed
  response per step per run, and activities carry retest lineage, test
  conditions, and deferred/seasonal scheduling.
- **Training is verified work.** `TrainingProgram`, `TrainingSession` (a
  VerificationActivity), and `TrainingAttendance` are the training records
  202 section 15 and Guideline 1.4 Part 5 require the Systems Manual to
  retain.
- **Submittal review is an activity** with a conforms / nonconforming /
  outstanding outcome (202 section 11); the record set is the submittal
  register.
- **Issues carry the 202 Appendix K field set** — found date, due date,
  root cause, recommendation, corrective action, typed impact areas, and
  avoided cost (Guideline 0 Appendix G) — and `RequirementVariance` gains
  the same typed impact areas its citation (202 Appendix O(a)) mandates.
- **The Systems Manual index is typed.** `SystemsManualEntry` rows assign
  living documents to the six parts shared by 202 section 14.2.3 and
  Guideline 1.4 section 4.3.2, ordered by spec section, per system where
  applicable — Guideline 1.4 explicitly endorses indexing over embedding.
- **Requirements can carry the Guideline 0 OPR content categories** (section
  5.2.2.4 items a-ac) as a typed choice, plus the Appendix J "Key OPR" flag.

## Measures and savings (cx.measures)

EBCx lives on the master list of findings: each finding's measure carries
implementation economics and the owner's selection decision, and savings
are stated per category (electric, gas, steam, water, demand, maintenance,
operational) on a basis that hardens from estimated through adjusted to
verified. `CxMeasure` + `SavingsEstimate` model that, keeping the finding
itself a `cx.issues::CxIssue` — the measure is the solution, not the
deficiency. Savings records are append-only so the estimate-to-verified
trail stays auditable.

## Deliberately deferred

Documentation-responsibility matrices (Guideline 0 Appendix D), meeting and
lessons-learned records, sampling plans and failure-escalation rules,
diagnostic/monitoring plans for ongoing Cx, KPI and benchmark-result
records (Guideline 32), M&V plan structure, warranty records, and the
operational-change log (Guideline 1.4 Appendix C) are recorded as roadmap
candidates — each is artifact-shaped but none was needed to close the
0.3.0 drivers.
