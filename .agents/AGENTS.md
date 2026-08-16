# Workspace Rules for My Room Ledger

## 🔒 STRICT IMMUTABILITY RULE FOR REFERENCE REPOSITORY

- **`ref_repo/` IS STRICTLY READ-ONLY AND LOCKED.**
- **NEVER** modify, edit, delete, reformat, or write any files inside `ref_repo/` under any circumstances.
- `ref_repo/` is an external reference artifact only.
- All code changes, schema edits, and documentation updates must be made strictly in the main repository files (`MASTER.md`, `docs/technical/architecture.md`, `docs/technical/database-schema.md`, `docs/technical/api-contracts.md`, `docs/*`, etc.).

---

## 📐 MANDATORY DOCUMENTATION MODULARIZATION & GLOSSARY RULE

- **Modular Document Placement**: All new rules, acceptance criteria, non-functional requirements, API contracts, or domain logic MUST be placed into their dedicated document with explicit cross-referencing:
  - Business Rules & Logic Invariants $\rightarrow$ [`docs/governance/business-rules.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/docs/governance/business-rules.md) (cited as `BR-xx`)
  - Functional Requirements $\rightarrow$ [`docs/stages/02-functional-requirements.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/docs/stages/02-functional-requirements.md) (cited as `FR-xx`)
  - Testable Acceptance Criteria $\rightarrow$ [`docs/stages/acceptance-criteria.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/docs/stages/acceptance-criteria.md) (cited as `AC-xx`)
  - Non-Functional Requirements $\rightarrow$ [`docs/stages/03-non-functional-requirements.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/docs/stages/03-non-functional-requirements.md) (cited as `NFR-xx`)
  - High-Level Summary Index $\rightarrow$ [`MASTER.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/MASTER.md) (concise summaries linking to `BR-xx` / `FR-xx`, never bloated).
- **Mandatory Glossary Maintenance (`docs/governance/glossary.md`)**:
  - Whenever a new abbreviation, technical term, domain concept, or document reference is introduced or used, [`docs/governance/glossary.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/docs/governance/glossary.md) MUST be updated immediately.
  - The glossary must index what each term means, where it is referenced, its abbreviations, and its authoritative document source.
- **Document Minimization & Modular Consolidation Rule**:
  - Before creating ANY new document, carefully review all existing project documentation and determine whether the new requirement, rule, decision, or information can be appropriately incorporated into an existing document.
  - Do **NOT** create a new document for every new requirement or update. Prefer updating an existing relevant document when it provides an appropriate place for the information.
  - Avoid unnecessary document duplication and fragmentation.
  - Do not force unrelated requirements into existing documents simply to avoid creating a file. If, after deep review of existing docs, their purpose, scope, and structure, a requirement represents a distinct concern that genuinely provides value as a dedicated file, you may recommend and create a new document.

