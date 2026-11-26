/********************************************************************************
* Copyright (c) 2025 Contributors to the Eclipse Foundation
*
* See the NOTICE file(s) distributed with this work for additional
* information regarding copyright ownership.
*
* This program and the accompanying materials are made available under the
* terms of the Apache License Version 2.0 which is available at
* https://www.apache.org/licenses/LICENSE-2.0
*
* SPDX-License-Identifier: Apache-2.0
  ********************************************************************************/


# How-to: derive FailureModes and perform FTA (S-CORE tooling)

Scope: written summary only, no repository changes.

Sources:
- `tooling/bazel/rules/rules_score/docs/user_guide/dependability_analysis.rst` ("Performing the Analysis")
- `tooling/.github/skills/score-safety-analysis/SKILL.md`, which is identical to `communication/.github/skills/score-safety-analysis/SKILL.md`
- Process fault models: https://eclipse-score.github.io/process_description/main/process_areas/safety_analysis/guidance/fault_models_guideline.html
- Example: `tooling/bazel/rules/rules_score/examples/seooc/safety_analysis/`

## FMEA vs. FTA (incl. ISO 26262)

### Verified (public sources)
- **Tooling** (`dependability_analysis.rst`):
  - FMEA is inductive and bottom-up. It examines components at their interfaces, asks how they can fail, and traces the effect on the system.
  - FTA is deductive and top-down. It starts from a safety goal or failure mode and traces back to the causes.
  - `rules_score` chains the two: FMEA on the public API produces `FailureMode`s, then FTA decomposes each one into `$RootCause`s.
- **S-CORE process** (safety_analysis workflow and guideline):
  - "Safety Analysis" is an umbrella term for **FMEA and DFA** only. The outputs are `wp__sw_component_fmea` and `wp__feature_fmea`.
  - FTA is not mentioned in the process. It exists only in the tooling.
- **ISO mapping**:
  - S-CORE uses ISO 26262:2018 (2nd edition).
  - The fault-model guideline complies with ISO 26262-6 7.4.10 and ISO 26262-9 8.4.6.
  - The safety-analysis guideline maps ISO 26262-9 8.4.1 to 8.4.10.

### ISO 26262 content (from knowledge, not checked against the text, which is paywalled)
- Part 9 Clause 8 ("Safety analyses") distinguishes two kinds of analysis:
  - **Inductive** (bottom-up): known causes lead to unknown effects. FMEA is the example.
  - **Deductive** (top-down): known effects lead to unknown causes. FTA is the example.
- Part 6 7.4.10 requires a safety-oriented analysis at the SW architectural level per Part 9 Clause 8.
  - It names neither FMEA nor FTA and has no per-ASIL method table.
  - Annex E gives informative guidance.
- Part 4 (system) and Part 5 (hardware) have per-ASIL recommendation tables for deductive vs. inductive analysis.
  - The values could not be verified; web sources disagree.
  - Do not quote them without the standard.

### Practical difference
| | FMEA | FTA |
|---|---|---|
| Question asked | "If X fails, what happens?" | "What can cause Y?" |
| Starting point | Element, interface, fault model | Failure mode or safety-goal violation |
| Result | Failure modes and their effects | Root causes, combined with AND/OR gates |
| Strength | Systematic coverage of each element and fault | Finds combinations of faults (AND) |
| Weakness | Misses combinations of faults | Only as complete as its top event |

### Verified examples (from public sources; the communication repo is not used)

**FMEA: S-CORE process example**

Source: [Safety Analysis Guideline, "Examples for FMEA and DFA at feature level"](https://eclipse-score.github.io/process_description/main/process_areas/safety_analysis/guidance/safety_analysis_guideline.html#examples-for-fmea-and-dfa-at-feature-level), quoted verbatim, minus the empty `:mitigation_issue:` and the fields `:status: valid` and `:version: 1`:
```rst
.. feat_saf_fmea:: Component 1 Call message not received
   :violates: feat_arc_dyn__mab__dynamic
   :id: feat_saf_fmea__mab__comp1_call_nreceived
   :fault_id: MF_01_01
   :failure_effect: Message is not received. This leads to a unavailability of a safety related functionality of the feature.
   :failure_root_cause: The message is lost due to a communication error.
   :safety_relevant: yes
   :mitigated_by: aou_req__mab__call_not_received
   :sufficient: yes
```
- **Bottom-up reasoning.** The analyst starts from one element of the dynamic architecture view ("flow component 1"), applies fault model MF_01_01 (message not received), and derives the effect: the safety function becomes unavailable.
- **Mitigation.** It is an AoU, because "the feature can't detect if it's not called".
- **Non-applicable fault models are documented with a reason.** The guideline gives two examples:
  - "message unintended sent" is not applicable because the component is ASIL B developed, non-complex and sufficiently tested.
  - "loss of execution" is not applicable because the feature is deterministic.
- **Format.** The process uses sphinx-needs. The tooling equivalent is a `ScoreReq.FailureMode` with guideword `LossOfFunction` (category MF_01_01), plus an AoU as the measure in a `RootCauseAssessment`.
- **Source typos.** The guideline itself writes "FM_01_07" and "FX_01_04", which presumably mean MF_01_07 and EX_01_04.

**FTA: NASA Fault Tree Handbook with Aerospace Applications, v1.1 (Aug 2002), §4.4, Fig. 4-15**
- The S-CORE process has no FTA example; FTA exists only in the tooling.
- Retrieved from a third-party mirror, https://www.mwftr.com/CS2/Fault%20Tree%20Handbook_NASA.pdf. The document identity was checked against its title page. The official NASA copy (hq.nasa.gov) returned 403.

System described in the handbook:
- A signal goes A → B and A → C, then B and C → D, then D → E.
- D needs input from B **or** C, so B and C are redundant.

Top event "no signal to E", as decomposed verbatim by the handbook ("think small", one immediate cause per step):
```
no signal to E
└─ no output from D
   └─ OR
      ├─ (1) input to D but no output from D   [basic failure, internal to D]
      └─ (2) no input to D
         └─ AND                                 [B and C are redundant]
            ├─ (3) no output from B
            │  └─ OR
            │     ├─ (5) input to B but no output from B  [basic failure]
            │     └─ (6) no input to B                     [developed further]
            └─ (4) no output from C                        ["analogous" to (3)]
```
- The handbook leaves the rest "to the reader"; "no input to A" is named as a basic input.
- Rendered in the tooling's FTA format, this would be:
  - `$FailureMode` = "no signal to E"
  - `$IntermediateEvent` = "no output from D", "no input to D", "no output from B"
  - `$OrGate` and `$AndGate` exactly as in the tree above
  - `$RootCause` = (1) and (5)
  - This mapping has not been built or validated with Bazel.
- **What the example shows:**
  - Top-down decomposition in single immediate-cause steps (taking two steps at once would have hidden event 1).
  - An AND gate exactly where redundancy exists.
  - Leaves are basic failures that a measure can be placed on.

**Shape reference in tooling format:** see `tooling/bazel/rules/rules_score/test/fixtures/aou_root_cause/` (`failure_modes.trlc`, `fta.puml`, `assessments.trlc`). It is a CI-tested test fixture with synthetic content, so it shows the syntax only, not a real analysis.

## Phase A: safety reasoning (FMEA + FTA). Do this first; the files only record the result.

### 1. Derive FailureModes (FMEA, bottom-up, per interface)
- Go through the `public_api` class diagram **method by method**. For mw/com this is `architectural_design/static_design/public_api.puml`, TRLC package `mw_com_api`.
- For each method, apply the process fault models and ask: "Can this occur, and would it violate a safety goal or safety requirement?" Dismiss the irrelevant fault models with a short rationale.
  - Message: MF_01_01 not received, 02 too late, 03 too early, 04 not received consistently by all recipients, 05 corrupted, 06 not sent, 07 unintentionally sent
  - Time constraint: CO_01_01 minimum boundary violated, CO_01_02 maximum boundary violated
  - Execution: EX_01_01 wrong result, 02 too slow, 03 too fast, 04 loss of execution, 05 arbitrary execution, 06 incomplete execution (infinite loop)
- Map each fault model to a `Guideword`:
  - Message → LossOfFunction, PartialFunction, Corrupted, UnintendedFunction, Wrong
  - Timing → TooEarly, TooLate, DelayedFunction
  - Execution → Wrong, LossOfFunction, ExceedingFunction, ArbitraryExecution
- Clustering rules:
  - Create one `FailureMode` per (interface, guideword) effect.
  - Methods that share a root cause go together in the `interface` list.
  - If one root cause falls under two guidewords, write two records.
- Write `failureeffect` from the **caller/system perspective** in worst-case terms relative to the safety goal. Good: "returns a stale value that the controller actuates on". Bad: "returns wrong data".
- `safety`: the ASIL of the safety goal that the failure mode can violate.
- `version`: increment it on every content change.
- `rationale` (optional): why the failure mode is relevant.

```trlc
ScoreReq.FailureMode EarlyCleanUp {
    guidewords    = [ScoreReq.Guideword.ExceedingFunction]
    description   = "..."
    failureeffect = "..."
    interface     = [mw_com_api.<ns>_Skeleton_<method>]   // generated ApiMethod record
    version       = 1
    safety        = ScoreReq.Asil.B
}
```
- `ApiMethod` record names follow `<namespace>_<Entity>_<method>`, with `.` replaced by `_`. All overloads of a method share one record, and each free function gets its own record. The `failure_modes` target needs the `architectural_design` target in `deps`.

### 2. Perform the FTA (top-down, one tree per FailureMode)
- Start at the failure mode and decompose it into **actionable** root causes: something you can place a measure on, not a restatement of the failure.
- Choosing gates:
  - Use `$OrGate` by default, when any single cause is enough.
  - Use `$AndGate` only when all causes must occur together, for example a fault plus a failed safety mechanism. This justifies a lower residual risk.
- Use `$IntermediateEvent` for causes that you decompose further.
- Use `$TransferInGate("<Pkg>.<FM>", parent)` to reuse another tree's root causes instead of copying them. Limits:
  - The target must be in the same `fault_trees` target.
  - Cycles are not allowed.
  - The gate cannot name `fm1` of a `$FailureMode` in its own diagram.
  - The inherited root causes' assessment ASIL must be at least the highest ASIL among the failure modes involved.

```plantuml
@startuml
!include fta_metamodel.puml
$FailureMode("Resources freed while in use", "Communication.EarlyCleanUp")   ' fm1..fm8
$OrGate("OG1", "Communication.EarlyCleanUp")
$RootCause("Shared ref-count wrong", "SkeletonRefCountWrong", "OG1")       ' plain identifier alias
$IntermediateEvent("Concurrent access", "IE1", "OG1")
$AndGate("AG1", "IE1")
$RootCause("...", "RcA", "AG1")
$RootCause("...", "RcB", "AG1")
@enduml
```
- Declare nodes top-down: `$FailureMode`, then gates, then events and leaves. Each node's `connection` points to its parent alias.
- A `$RootCause` alias must be a plain identifier, never dotted. The same alias may appear in several trees.
- `puml_cli` generates `fta_events.trlc` in package `<fta_package>` (`mw_com_fta`), with one `RootCause` stub per alias.

### 3. Assess every root cause
- Each `$RootCause` appears in exactly one `RootCauseAssessment.root_causes`. Root causes that share an argument may be grouped in one assessment.
- `measures` (versioned with `@N`) depend on who closes the cause:
  - `CompReq`: the SEooC closes it itself.
  - `AoU`: the integrator or caller must close it.
  - `[]`: the cause does not apply; the rationale says why.
- `safety` must be at least the ASIL of the covered failure modes and at most the ASIL of every measure.
- Every own `AoU` must be listed as a measure somewhere.
- Measures never reference root causes.

## Phase B: wiring and validation (mechanical)
- Use the `failure_modes`, `fault_trees` (`srcs` = puml, `deps` = failure_modes, `fta_package`) and `safety_analysis` targets. `safety_analysis` takes `failure_modes`, `fault_trees`, `assessments` and `safety_measures` (`aous` and comp-req targets).
- Add each new `.puml` to the `fta_files` / root_causes filegroup, in alphabetical order.
- Validate with `bazel test` on the dependable element. Lobster reports uncovered root causes or AoUs as "missing reference to Root Cause Assessments". These are hard failures only when `maturity = "release"`.

Traceability chain:
1. `FailureMode.interface` → `ApiMethod`
2. `FailureMode` → `$FailureMode` fm1..fm8
3. `$RootCause` → `RootCauseAssessment.root_causes`
4. `RootCauseAssessment` → `AoU@N` / `CompReq@N`

## Observed gaps in mw/com (for later)
- 34 `FailureMode`s carry only a commented `// interface = "mw.com..."` string. They are not linked to `mw_com_api` `ApiMethod` records, and `failure_modes` has no `deps` on the arch design.
- Some FailureModes have no FTA, for example `Communication.EarlyCleanUp` (Skeleton Destroy). There are 71 FMs and 65 puml files.
- `assessments.trlc` contains only a `DummyAssessment`; `maturity = "development"` (see #1075).
