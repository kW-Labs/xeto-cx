# 0.2.0 Design Rationale — CFR, EBCx, Ongoing Cx, and Verification Results

xeto-cx 0.1.0 modeled the new-construction commissioning arc. 0.2.0
extends the ontology across the rest of the building lifecycle, anchored
clause-by-clause in ANSI/ASHRAE/IES Standard 202-2018.

## Why the CFR is a subtype, not a new document model

Standard 202 section 3 defines the Current Facility Requirements (CFR) as
"a written document that details the current functional requirements of an
existing facility and the expectations for how it should be used and
operated, including goals, measurable performance criteria, cost
considerations, benchmarks, success criteria, and supporting information."
Set that beside the OPR definition in the same section and the structures
are near-identical — what differs is the anchor (a project vs a facility)
and the lifecycle. The Foreword makes the relationship explicit: "The OPR
may transition to the Current Facility Requirements (CFR)."

So 0.2.0 models both as subtypes of one abstract `RequirementsDoc`, and
`Requirement.docRef` points at the container without caring which subtype
owns it (this renames 0.1.0's `oprRef` — a breaking change taken
deliberately, while the cost is near zero). Requirement-level lineage
across the transition uses the existing `supersedesRef`; document-level
lineage is `Cfr.sourceOprRefs`, a list because one facility CFR absorbs
requirements from successive project OPRs over decades.

## The EBCx lifecycle

Existing Building Cx (202 section 3) arrives as vocabulary, not as a
separate library: new `CxPhase` values (planning, investigation,
implementation, persistence) and a `CxProjectType` choice covering new
construction, renovation, retro-commissioning, and re-commissioning. A
separate "EBCx lib" would structurally claim that existing buildings are
an add-on; the definitions in section 3 say the opposite.

`OcxProgram` is the one genuinely new entity: 202 defines Ongoing Cx as
"a continuation of the Cx well into occupancy and operations to
continually improve the operation and performance of a facility to meet
current and evolving CFR or OPR." A program is facility-anchored and
perpetual — deliberately a sibling of `CxProject`, not a subtype, because
a bounded engagement and a standing program have different lifecycle
semantics.

## Verification results as records

Standard 202 records verification in reports written at moments in time.
But its own definition of Ongoing Cx is continuous re-verification — and
a check that re-runs needs somewhere to write its answer. `VerificationResult`
is that record: one check run, append-only, with a three-way outcome.
`outcomeInconclusive` is deliberate: an honest contract never conflates "no
data" with either "passed" or "failed". Failing results feed the existing
issue machinery (`CxIssue` with `sourceRef`); how a tool schedules checks
or gates phases on results is application behavior, out of ontology scope.

## Owner-accepted variance

Appendix O(a): the owner "may choose to accept performance that is at
variance with the OPR, either permanently or until schedule and budget
constraints allow for correction," documented with impacts, and "the OPR
must be updated to match the revised expectations." `RequirementVariance`
records exactly that — the contract-amendment primitive that keeps
document versioning honest.

## The document taxonomy

The Cx Plan (section 7) and Systems Manual (section 14, containing the
Facility Guide, section 3) complete the standard's required document set
as thin subtypes of an abstract `CxDoc`. One deliberate inversion:
Appendix L instructs practitioners to "insert final copy" of the OPR,
BOD, and other documents into the Systems Manual — embedded photocopies.
`SystemsManual.componentRefs` points at the live versioned records
instead. A Systems Manual that references living documents cannot go
stale the way a binder of copies does; that is the point of modeling
these documents as data in the first place.

## Out of scope, on purpose

Progress/final Cx report record types (generated artifacts — wait for a
real consumer), Appendix L's full section tree (a refs list suffices),
phase-gate evaluation semantics (application behavior), and any notion of
how documents are parsed or extracted (tooling concern, not ontology).
