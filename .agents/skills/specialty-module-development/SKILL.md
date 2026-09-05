---
name: specialty-module-development
description: >-
  Use this skill when developing a new clinical coding specialty module (such as Wound Care,
  Dermatology, Surgery, Oncology, Infusion, or any medical specialty coder), extending an
  existing specialty module, analyzing a clinical chart corpus to understand document structures,
  building document normalizers and clinical extractors, defining deterministic specialty coding
  rules, implementing medical coding UI/WPF workflows, conducting evidence-aware testing and
  independent PonyTail validation, or packaging and releasing a clinical coding module.
---

# Specialty Module Development

This skill defines the standardized, disciplined engineering methodology for building medical coding specialty modules in the SuperbillApp ecosystem. It captures the reusable lessons, document-understanding architecture, verification standards, and release practices learned across production modules.

## Standard Specialty Lifecycle Workflow

```
Corpus → Document Structure → Normalization → Evidence Model → Specialty Rules → Coding → UI/Workflow → Testing → PonyTail Validation → Release
```

---

## 1. Corpus Analysis (Understand Before Coding)

**Strictly Offline Rule:** All clinical corpus analysis, document inspection, and test data parsing must remain 100% local and offline. Zero remote APIs, zero cloud services, zero external LLMs, and zero PHI/PII leakage.

Before writing a single line of extraction or coding logic:
1. **Inspect Real Representative Artifacts:** Examine genuine PDFs, DOCX files, text exports, scan layouts, and clinical spreadsheets from the target specialty corpus.
2. **Identify Document Structural Properties:**
   - Physical page breaks, margins, and layout artifacts.
   - Running headers, footers, clinic logos, and metadata banners.
   - Section delimiters (e.g., SOAP headings, procedure narratives, plan sections, addenda).
   - Template boilerplate vs. provider-entered narrative.
   - Multi-page continuation patterns (e.g., sentences split across page transitions).
3. **Analyze Clinical Granularity:**
   - Single-site vs. multi-site documentation structures (lesions, wounds, anatomical locations).
   - Distributed evidence: related clinical facts documented across non-contiguous sections.
   - Full chart vs. isolated procedure fragment differences: full charts contain surrounding noise, prior history, pathology reports, medication lists, and unrelated visits that must not corrupt coding.

---

## 2. Normalization Layer (Normalize First)

Establish a deterministic document-normalization layer before executing specialty-specific extraction:
- **De-paginate Cleanly:** Detect and bridge sentences broken across page boundaries.
- **Strip Header/Footer Noise:** Identify repeated running headers, footers, page numbering, and clinic metadata banners so they do not interrupt narrative parsing or inject spurious dates.
- **Normalize Whitespace and Formatting:** Standardize Unicode characters, bullets, tabs, and inconsistent line breaks.
- **Preserve Clinical Continuity:** Ensure that physical page breaks never become artificial boundaries for clinical sentences, paragraphs, or evidence blocks.

---

## 3. Architecture: Decouple Understanding from Coding

Keep document understanding and clinical coding cleanly separated into distinct architectural layers:
```
Raw Chart / Paste
       │
       ▼
[Document Normalizer]  ──► Cleaned text with restored cross-page continuity
       │
       ▼
[Clinical Extractor]   ──► Structured Clinical Evidence (Visits, Sites/Wounds, Measurements, Modalities)
       │
       ▼
[Specialty CodeEngine] ──► Deterministic Coding Rules, CPT/HCPCS, Modifiers, ICDs, Review Flags
```
- **Never embed clinical interpretation inside CPT regexes.** First extract *what clinically happened* (anatomical site, procedure performed, depth/stage/dimensions, equipment/technique used, provider roles).
- Then evaluate *specialty coding rules* against the extracted clinical model.

---

## 4. Structural Evidence Association

- **Hierarchical Anchoring:** Associate extracted clinical facts with their specific clinical entities:
  `Visit / Encounter -> Procedure -> Anatomical Site / Lesion / Wound -> Modality / Technique -> Diagnosis`.
- **Avoid Proximity Fallacies:** Do not rely merely on raw character proximity or naive keyword matching when evidence spans across multiple document sections.
- **Distinguish Current vs. Historical Evidence:**
  - Services, procedures, or measurements documented as performed during the current visit must be strictly distinguished from historical references, prior biopsy dates, prior plans, or longitudinal treatment course histories.
  - **Never carry forward prior services or codes** simply because a patient remains in an ongoing episode of care.

---

## 5. Defined Encounter Workflow: One Chart = One Visit

When the product workflow specifies that each pasted chart represents the visit/encounter being coded:
- **No Longitudinal Encounter Engine:** Do not build complex cross-visit encounter-selection machinery that guesses which encounter the user intended.
- **Header Date Handling:** Do not confuse repeated header/footer dates with multiple distinct visits.
- **Material Ambiguities:** If a single document contains genuine, conflicting Dates of Service (DOS) that produce material coding uncertainty, flag the case as `NEEDS_REVIEW` with clear rationale rather than silently guessing.

---

## 6. Requirements Grilling & Decision Discipline

Before implementing or modifying rules, resolve requirements and ambiguities using the disciplined inquiry protocol:
- **KNOWN DECISION = DIRECT:** Proceed immediately using settled architecture and established rules. Do not re-litigate or re-research settled decisions.
- **UNKNOWN MATERIAL FACT = RESEARCH / ASK:** Identify all material unknowns that could affect coding correctness or clinical integrity. Ask all clarifying questions together upfront in a single batch—never serially.
- **NO MATERIAL UNCERTAINTY = PROCEED:** Implement cleanly without delay.

---

## 7. Controlled & Deterministic Coding Logic

- **Deterministic Rules:** Specialty coding rules must evaluate clinical evidence deterministically.
- **Mandatory Review Triggers:** Missing mandatory technical specifications, conflicting documentation, or unsupported services must set `Status = NeedsReview` with specific clinical explanations.
- **Strictly Scoped Coding Output:** Only produce billable codes substantiated by documented clinical evidence. Prohibit unauthorized or mutually exclusive codes.
- **Auditability:** Every generated code must link to its supporting clinical evidence excerpt.

---

## 8. UI & Workflow Architecture (WPF / Native Windows)

- **Single Source of Truth:**
  - Avoid duplicate, competing representations of coding data in memory or UI.
  - The editable coding log/grid is the authoritative source of truth.
  - Summary views, diagnosis panels, and review statuses must project/derive directly from the authoritative grid (e.g., CPT row ICD fields project dynamically into the unique ICD diagnosis summary).
- **Patient-Scoped Persistence:**
  - Workbook, database, or queue updates must be strictly scoped by a patient/visit unique key.
  - Saving or re-finalizing a patient must update that patient's existing records in place—never duplicate, overwrite, reorder, or disturb other queue records.
- **Native Windows & WPF Conventions:**
  - Use native WPF Commands (`ICommand` / CommunityToolkit `[RelayCommand]`) and bindings.
  - Implement native WPF Access Keys / Mnemonics with underscores (e.g., `_Analyze Chart`, `_Load Inventory`, `Code and _Continue`). Pressing `Alt` must reveal mnemonic underlines.
  - Keyboard shortcuts (e.g., `Ctrl+Enter`, `Ctrl+O`, `Esc`) must invoke the same commands as mouse clicks; do not create duplicate shortcut-specific business logic.

---

## 9. Comprehensive Testing Discipline

Do not stop at simple happy-path unit tests. Implement multi-layered automated tests:
1. **Isolated Fragments vs. Full Charts:** Verify that the engine processes both minimal procedure notes and complete, multi-page patient charts.
2. **Evidence-Aware Acceptance:** A full chart containing additional valid clinical evidence may legitimately produce different or enriched coding compared to a bare fragment. Compare available evidence explicitly.
3. **Multi-Site & Multi-Lesion Handling:** Validate multi-site documentation, distinct anatomic locations, modifier application, and unit calculations.
4. **Historical vs. Current Disambiguation:** Verify that historical procedure mentions do not trigger billable codes for today's visit.
5. **UI & State Synchronization:** Test dynamic grid-to-summary updates, row additions, row deletions, queue completion states, patient reopening, and re-finalization.
6. **Failure & Boundary Cases:** Blank inputs, missing clinical parameters, unsupported modalities, and malformed inputs.
7. **Runtime & Interactive Verification:** Execute automated UI/scripted verification of the actual running application where practical.

---

## 10. Independent PonyTail Validation

Every specialty module implementation or major feature must undergo independent PonyTail review before release:
- **Mandatory Review Artifacts:** PonyTail must inspect actual changed files, actual git diffs, actual test suites, and actual runtime execution logs.
- **Discipline:** Known decisions remain direct; material facts are verified against ground-truth evidence.
- **Severity Classification:**
  - **CRITICAL:** Compliance, clinical coding correctness, data corruption, or system crash issues. (Blocks release).
  - **MATERIAL:** Architectural drift, competing sources of truth, broken UI bindings, or missing test coverage. (Blocks release until resolved).
  - **MINOR:** Cosmetic spacing, phrasing, or non-blocking enhancements.
  - **PASS:** Formally verifies that all requirements and tests are satisfied.

---

## 11. Anti-AI-Slop Coding Discipline

All implementation code must adhere to strict quality standards:
- **Smallest Clean Change:** Solve the exact requirement cleanly. Do not write speculative code or over-engineer.
- **Leverage Existing Architecture:** Reuse existing models, services, helpers, and conventions within `SuperbillApp.Core`.
- **Zero Duplicate State / Logic:** Never create shadow properties or duplicated validation logic.
- **No Unnecessary Abstractions:** Do not introduce speculative interfaces, generic factories, or excessive scaffolding for hypothetical future features.
- **Direct & Readable:** Keep methods focused, readable, and free of verbose, obvious commentary. If a major refactor seems necessary, identify the concrete blocker first.

---

## 12. Release & Packaging Discipline

When releasing a validated specialty module:
1. **Freeze Code:** No new features, refactoring, or rule tweaks during packaging.
2. **Clean Build:** Build in net8.0-windows configuration; ensure 0 warnings and 0 errors.
3. **Run Full Test Suite:** Execute all unit, integration, and workflow tests.
4. **Deploy Minimal Package:**
   - Include only the executable, required application and core DLLs, third-party dependency DLLs, runtime JSON configs, and assets.
   - Include the portable launcher script (`.bat`) referencing `%~dp0`.
   - Strictly exclude `.pdb` debug files, scratch scripts, temporary outputs, test data, and logs.
5. **Verify Package:** Extract the ZIP to an isolated temporary directory and verify that the executable launches cleanly.
6. **Clean Git Release:** Review `git status` and `git diff`. Stage only intended source/test/project files. Exclude all scratch, zip, and test artifacts. Commit with a clear descriptive message and push to the configured remote branch.
