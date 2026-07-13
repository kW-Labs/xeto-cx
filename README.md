# xeto-cx — Open Commissioning Ontology

Machine-readable [Xeto](https://xeto.dev) schema libraries for the building
commissioning process: Owner's Project Requirements, Basis of Design,
verification activities, deficiencies, and the traceability spine that
connects them.

**The core idea:** the OPR becomes a validatable contract, not a Word
document. Requirements are typed data. The machine-checkable subset compiles
to xeto specs, so design data, submittals, and the as-built model can be
verified with `fits()`. Commissioning becomes, literally, making the building
fit its spec — and the commissioning record becomes a portable asset the
owner keeps, instead of a PDF trapped in a proprietary tool.

## Status

**Early draft (0.2.0) — seeking collaborators.** These libraries are being
proven on real commissioning projects. The long-term intent is to contribute
the core (`cx`, `cx.requirements`) to the
[Project Haystack](https://project-haystack.org) library family as `ph.cx`,
with stewardship transferred to a Haystack working group. The vendor-neutral
`cx.*` naming, ph library conventions, and AFL-3.0 license (identical to
Project Haystack's) are deliberate: no single vendor should own the format
your commissioning record lives in.

This repo is stewarded by [kW Engineering](https://kw-engineering.com) and is
not (yet) affiliated with Project Haystack.

## Libraries

| Library | Purpose | Upstream ambition |
|---|---|---|
| `cx` | Root vocabulary: phases, roles, disciplines, verification planes/modes, the CxProject entity, document taxonomy + OcxProgram, and the tag vocabulary | ph.cx candidate |
| `cx.requirements` | OPR/CFR/BOD document structures, Requirement, acceptance criteria, variance records, and the traceability spine | ph.cx candidate |
| `cx.verification` | Verification activities: checklists, PVT/FPT test scripts, contractor testing, start-ups, TAB, field observations, attestations | community-open |
| `cx.issues` | Deficiency + field-observation interchange shapes | community-open |
| `cx.content` | Seed reference content as instance data: an example OPR requirement set, an AHU installation checklist, an FPT script, and an EBCx continuation (CFR, OCx program, verification results, variance, Systems Manual) | illustrative only |

## The three verification planes

A real OPR is only partly machine-checkable, and this ontology is honest
about that. Every `Requirement` declares its **verification plane**:

| Plane | Example requirement | Evaluated by |
|---|---|---|
| `conformance` | "All AHUs shall include heat recovery"; "CHW design ΔT ≥ 12°F" | `fits()` against typed records, via criteria that use xeto's own constraint vocabulary (`SpecConstraint`) |
| `performance` | "Space temps 70–74°F at 95% adherence during occupied hours" | telemetry evaluation over a span (`PerformanceCriterion`) |
| `attestation` | "Operators trained on the BAS before turnover" | human signoff with evidence (`ProcessRequirement` + `Attestation`) |

## The whole lifecycle: OPR to CFR to Ongoing Cx

Standard 202's Foreword names two transitions at turnover: the OPR may
transition to the Current Facility Requirements (CFR), and the Cx Plan to
an Ongoing Cx (OCx) Plan. This ontology models both. `Opr` and `Cfr` are
subtypes of one abstract `RequirementsDoc` — a `Requirement` never knows
which owns it, so the same requirement (and its verification criteria)
survives from design intent to operating contract. `OcxProgram` anchors
continuous verification against the current CFR, and every check —
machine or human — writes a `VerificationResult` with a three-way
outcome: `outcomePass`, `outcomeFail`, or `outcomeInconclusive`, because an honest record
never reports "no data" as either compliance or violation. When the
owner formally accepts performance at variance with a requirement
(Standard 202 Appendix O), a `RequirementVariance` record preserves the
amendment trail. The Systems Manual, which Appendix L assembles by
inserting final copies of project documents, becomes an index of references to
the living records instead.

## The traceability spine

Every record points back to the requirement it serves:

```
OPR Requirement ──▶ BOD Response ──▶ Design Review Comment
      ▲                                      │
      │            requirementRef(s)         ▼
  CxIssue ◀── Verification Activity / Checklist Item / Test Step
```

The commissioning traceability matrix — a core ASHRAE Guideline 0
deliverable — becomes a query over this graph instead of a hand-maintained
spreadsheet.

## Example

```xeto
@opr-3-1-1: Requirement {
  dis: "AHU heat recovery"
  docRef: @my-opr
  reqId: "OPR-3.1.1"
  narrative: "All air handling units serving more than 5,000 cfm outdoor air shall include exhaust air heat recovery."
  conformancePlane
  reqRequired
  targetSpec: "ph::Ahu"
}

@opr-3-1-1-c: SpecConstraint {
  requirementRef: @opr-3-1-1
  slot: "heatRecovery"
  requiredMarker: "heatRecovery"
}
```

A conforming tool compiles that constraint into a project-scoped xeto spec
extending `ph::Ahu` and validates every AHU record against it with
`fits()`/`fitsExplain()`.

## Relationship to ASHRAE Guideline 0 / Standard 202

These libraries encode the *artifacts* of the commissioning process that
Guideline 0 and Standard 202 define in prose — they implement the process as
data, they do not replace or reinterpret the standards.

## Building

Requires a Xeto toolchain ([Haxall](https://haxall.io) 4.0.5+ or
SkySpark 4.0.5+):

```bash
# copy the cx* directories into {fan.home}/src/xeto/, then:
xeto build cx cx.requirements cx.verification cx.issues cx.content
```

CI compiles every push with Haxall — see `.github/workflows/build.yml`.

## Contributing

Issues and PRs welcome — especially from commissioning providers, building
owners, and tool vendors. Schema conventions are in `CONTRIBUTING.md`. If you
represent an owner who wants to require open-format commissioning
deliverables, or a tool that wants to read/write these shapes, open an issue
— that conversation is the whole point.

## License

[Academic Free License 3.0](LICENSE) — the same license as Project Haystack.
