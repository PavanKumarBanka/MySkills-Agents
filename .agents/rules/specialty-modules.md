# Clinical Specialty Module Development Rule

When tasked with creating, extending, refactoring, testing, validating, or releasing any medical coding specialty module (such as Wound Care, Dermatology, Surgery, Oncology, Infusion, E/M, or any clinical coder):

1. **Activate Skill:** Load and strictly follow the instructions in [specialty-module-development](file:///.agents/skills/specialty-module-development/SKILL.md).
2. **Follow Standard Lifecycle:**
   `Corpus → Document Structure → Normalization → Evidence Model → Specialty Rules → Coding → UI/Workflow → Testing → PonyTail Validation → Release`
3. **Strictly Offline:** Keep all chart analysis, document inspection, and test data strictly offline and local. Zero external APIs, zero cloud services, and zero PHI/PII leakage.
4. **Decouple Understanding from Rules:** Build the document normalizer and structured clinical extractor before applying specialty coding logic. Never bury document understanding in regexes.
5. **Single Source of Truth:** In UI/WPF workflows, ensure the editable coding grid is the single authoritative source of truth. Projections (e.g., ICD summaries) derive dynamically from the grid.
6. **Anti-AI-Slop:** Make the smallest clean change. Do not introduce speculative abstractions, duplicate state, or unnecessary interfaces.
