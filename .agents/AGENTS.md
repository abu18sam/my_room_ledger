# Workspace Rules for My Room Ledger

## 🔒 STRICT IMMUTABILITY RULE FOR REFERENCE REPOSITORY

- **`ref_repo/` IS STRICTLY READ-ONLY AND LOCKED.**
- **NEVER** modify, edit, delete, reformat, or write any files inside `ref_repo/` under any circumstances.
- `ref_repo/` is an external reference artifact only.
- All code changes, schema edits, and documentation updates must be made strictly in the main repository files (`MASTER.md`, `architecture.md`, `database-schema.md`, `api-contracts.md`, `docs/*`, etc.).

---

## 📐 MANDATORY DOCUMENTATION MODULARIZATION & GLOSSARY RULE

- **Modular Document Placement**: All new rules, acceptance criteria, non-functional requirements, API contracts, or domain logic MUST be placed into their dedicated document with explicit cross-referencing:
  - Business Rules & Logic Invariants $\rightarrow$ [`docs/business-rules.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/docs/business-rules.md) (cited as `BR-xx`)
  - Functional Requirements $\rightarrow$ [`docs/02-functional-requirements.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/docs/02-functional-requirements.md) (cited as `FR-xx`)
  - Testable Acceptance Criteria $\rightarrow$ [`docs/acceptance-criteria.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/docs/acceptance-criteria.md) (cited as `AC-xx`)
  - Non-Functional Requirements $\rightarrow$ [`docs/03-non-functional-requirements.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/docs/03-non-functional-requirements.md) (cited as `NFR-xx`)
  - High-Level Summary Index $\rightarrow$ [`MASTER.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/MASTER.md) (concise summaries linking to `BR-xx` / `FR-xx`, never bloated).
- **Mandatory Glossary Maintenance (`docs/glossary.md`)**:
  - Whenever a new abbreviation, technical term, domain concept, or document reference is introduced or used, [`docs/glossary.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/docs/glossary.md) MUST be updated immediately.
  - The glossary must index what each term means, where it is referenced, its abbreviations, and its authoritative document source.
