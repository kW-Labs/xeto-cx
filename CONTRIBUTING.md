# xeto-cx — Open Commissioning Ontology

Xeto schema libraries for the building commissioning process. Open source
(AFL-3.0), vendor-neutral, built to Project Haystack conventions with the
intent to upstream the core as `ph.cx`.

## Repository Structure

```
cx/                 → root vocabulary (phases, roles, disciplines, planes, modes, CxProject, OcxProgram, document taxonomy, acceptance records, tag vocabulary)
cx.requirements/    → OPR/CFR/BOD contract layer + traceability spine
cx.verification/    → verification activities, checklists, tests, training, attestations
cx.issues/          → deficiency + observation interchange shapes
cx.measures/        → improvement measures + savings (EBCx findings economics)
cx.sequences/       → control sequences of operation as commissioning artifacts
cx.content/         → seed reference content (instance data)
```

## Dependency Rules

- All libs depend on `sys` + `hx`. The `hx` dep exists solely because the `hxSysOnly` lib-meta tag (which lets these libs load boot-basis in Haxall/SkySpark, available to every project with zero per-project enablement) is *defined by* the hx lib. It is expected to drop away if/when the core upstreams into the `ph.*` family.
- **No dep on `ph`**: ph is not boot-basis loadable, and a `hxSysOnly` lib with a non-boot dep breaks at boot. All ph spec references are qname *strings* (`targetSpec: "ph::Ahu"`), resolved at project-namespace level by consuming tools — never structural imports.
- Family libs additionally depend on `cx` (never on each other, except `cx.content` which depends on `cx`, `cx.requirements`, `cx.verification`, `cx.issues`, `cx.measures`, `cx.sequences`)
- **Never** depend on vendor libs (`kw.*` or anyone else's) — this repo must stand alone

## Xeto Conventions (verified against SkySpark 4.0.5 / Haxall 4.0.5 toolchain)

- **Top-level specs PascalCase**; slot names camelCase. Lowercase top-level defs are rejected.
- **Choices use the taxonomy form** (see `doc.xeto::Choices`):
  `CxPhase: Choice` then `CxPhaseDesign: CxPhase { designPhase }` — one PascalCase value spec per choice value, each carrying its marker. Do NOT use the inline form (`Choice { a, b }`); validation semantics differ.
- **Choice values on instances are bare markers** (`@x: CxProject { designPhase }`), never `slot: value` form — a bare identifier in value position parses as a spec ref and fails.
- **Multi-select choices** use `<multiChoice>` on the slot (`disciplines: CxDiscipline? <multiChoice>`), not lists.
- **Choice-value markers are namespaced defensively** (`designPhase` not `design`, `controlsRole` not `controls`) — generic names collide with ph global tags (`design` and `controls` already exist in ph) and with each other across choices.
- **No Maybe on parameterized lists**: `List<of:Ref>?` fails to parse on this toolchain. Use plain `List?` with an `// of: X` comment.
- **List-of-refs instance syntax**: `sourceOprRefs: { @ex-opr, @ex-other }` — braces around comma-separated refs (verified on SkySpark 4.0.5).
- **Date instance syntax**: `startedAt: Date "2026-07-01"` — explicit type prefix + quoted ISO date (verified on SkySpark 4.0.5).
- **DateTime instance syntax**: `ranAt: DateTime "2026-06-01T08:00:00-06:00 Denver"` — explicit type prefix + quoted ISO datetime with tz suffix (verified on SkySpark 4.0.5).
- **Flags are Markers, not Bool** (`requiresPhoto: Marker?`) — presence semantics is the Haystack idiom, and bare `true` doesn't parse in instance data anyway.
- **Global tags** are declared as `*name: Marker` slots on the abstract `CxEntity` base in `cx/cx.xeto` — the root lib owns the tag vocabulary (same architecture as ph's `entity.xeto`).
- **Flat records + refs, not embedded lists**: checklist items point at templates via `templateRef`, criteria point at requirements via `requirementRef`. Haystack-idiomatic and Folio-friendly.
- `lib.xeto` contains only the pragma. One `.xeto` content file per lib until size demands splitting.
- No semicolons in spec bodies. `//` comments directly above a spec become its doc.

## Validate Locally Before Pushing

Copy libs into a SkySpark/Haxall source tree and compile:

```bash
SS=/path/to/skyspark-4.0.5
for d in cx cx.requirements cx.verification cx.issues cx.measures cx.sequences cx.content; do
  rm -rf "$SS/src/xeto/$d"; cp -r "$d" "$SS/src/xeto/$d"
done
cd "$SS" && bin/xeto build cx cx.requirements cx.verification cx.issues cx.measures cx.sequences cx.content
```

Clean output is `Compiled xetolib [...]` per lib; any `ERROR:` line is a failure.
Clean up `$SS/src/xeto/cx*` and `$SS/lib/xeto/cx*` afterward so the install
stays pristine. CI runs the same compile with Haxall 4.0.5 on every push.

## Design Principles

- **Three verification planes** (`conformancePlane` / `performancePlane` / `attestationPlane`) — every Requirement declares how it is verified. Never blur them: over-claiming machine-checkability costs the standard its credibility.
- **The narrative is sacred.** `Requirement.narrative` is the text the owner signs. Structured criteria are its projection, never its replacement.
- **`SpecConstraint` fields are xeto's own constraint vocabulary** (slot, minVal/maxVal, unit, marker) so compiling a requirement to a validatable spec is mechanical — no interpretation layer.
- **Schema here, content elsewhere.** `cx.content` carries just enough example data to demonstrate the model. Full checklist/test libraries are the domain of tools and providers.
- **Versions advance in minor steps pre-1.0** (breaking changes are allowed with a minor bump and explicit release notes); `1.0.0` waits until the first real-project validation completes.
