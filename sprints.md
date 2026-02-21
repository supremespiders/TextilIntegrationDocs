# WinTech Textile — Sprint Plan

**Project:** WinTech Textile Production Features
**Based on:** Technical & Functional Analysis V2 (February 16, 2026)
**Total Estimate:** ~57 person-days (with 20% contingency: ~68 days)
**Sprint Length:** ~10 working days (2 weeks)
**Number of Sprints:** 7

---

## Sprint Overview

| Sprint | Focus | Phases | Days |
|--------|-------|--------|------|
| Sprint 1 | Foundation + Workflow Core | 1, 2 | 9 |
| Sprint 2 | Formula Integration + Config UI | 3, 4 | 8 |
| Sprint 3 | Talon System + Post Confirmation | 5, 6 | 9 |
| Sprint 4 | KW NEV / IC2S Integration | 7 | 7 |
| Sprint 5 | KW File Exports + RFID Printing | 8, 9 | 7 |
| Sprint 6 | Testing + Reports | 10, 11 | 9 |
| Sprint 7 | Accessory Stock + KW Invoicing + Deployment | 12, 13, 14 | 8 |
| | **Total** | | **57** |

---

## Sprint 1 — Foundation + Workflow Core

**Duration:** 9 days
**Priority:** CRITICAL PATH
**Goal:** Lay the database foundation and implement the core workflow engine that all other features depend on.

### Tasks

#### Phase 1 — Database Schema & Migrations (3 days)

- [ ] Create `Operation` table (master data: Code, Name, Description, IsActive)
- [ ] Create `FamilyWorkflow` table (per-family, one active at a time)
- [ ] Create `FamilyWorkflowStatus` table (ordered statuses in a workflow)
- [ ] Create `FamilyWorkflowStatusOperation` table (operation config: TimeFormulaCode, OrderFormulaCode, PostId)
- [ ] Create `FamilyWorkflowOperationAccessory` table (accessory config: QuantityFormulaCode)
- [ ] Create `BdcWorkflow` table (1:1 with Bdc, immutable snapshot)
- [ ] Create `BdcWorkflowOperation` table (with `IsAutoCompleted` flag)
- [ ] Create `BdcWorkflowOperationAccessory` table
- [ ] Create `GlobalCounter` table (for RFID sequential numbering)
- [ ] Create `ClientMovementTypeMapping` table
- [ ] Create `NevImport` and `NevRoll` tables
- [ ] Create `StoreRodCounter` table (sequential rod counter per Store)
- [ ] Create `ClientFileSendLog` table (file send tracking + dedup)
- [ ] Alter `Bdc`: add `RfidNumber` VARCHAR(20) NULL
- [ ] Alter `User`: add `Matricule`, `FirstName`, `LastName`, `IsActive`
- [ ] Add all required indexes (BdcWorkflow.BdcId UNIQUE, BdcWorkflowOperation composite, FamilyWorkflow active lookup, NevRoll.PaletteNumber, IsAutoCompleted, ClientFileSendLog composite, User.Matricule)
- [ ] Write and validate EF migrations

#### Phase 2 — WorkflowService Core (6 days)

- [ ] Implement `WorkflowService.GetActiveFamilyWorkflow(familyId)`
- [ ] Implement `WorkflowService.InstantiateWorkflow(bdcId, familyWorkflowId)` — creates BdcWorkflow + child records as immutable snapshot
- [ ] Implement `WorkflowService.CompleteOperation(operationId, userId)` — Technical dept (enforced order)
- [ ] Implement `WorkflowService.CompleteOperationByBarcode(barcodeId, userId)` — Textile dept (talon scan)
- [ ] Implement **skip rule & auto-complete logic**: when a later operation is scanned, preceding unscanned operations are auto-completed (`IsAutoCompleted = true`, PostId = NULL, CompletedAt = NULL, CompletedById = NULL)
- [ ] Implement `WorkflowService.GetCurrentStatusFromWorkflow(bdcWorkflowId)` — derives BDC status from operation completion
- [ ] Implement `WorkflowService.GetAutoCompletedOperations(bdcWorkflowId)` — for Anomaly View
- [ ] Auto-create `BdcStatusLog` when all operations under a status complete
- [ ] Hook `WorkflowService.InstantiateWorkflow` into BDC creation (Communicator + manual)
- [ ] Ensure backward compatibility: existing BDCs without workflow continue normally; status-based queries unchanged

### Deliverables

- All new DB tables created and migrated
- WorkflowService implemented and unit-tested
- BDC creation triggers workflow instantiation (Communicator path)

### Acceptance Criteria

- A new BDC gets a `BdcWorkflow` record with all operations populated
- Completing all operations under a status changes BDC status and creates `BdcStatusLog`
- Existing BDCs without a workflow are unaffected
- Skip rule: scanning operation N auto-completes operations 1…N-1 if unscanned

---

## Sprint 2 — Formula Integration + Config UI

**Duration:** 8 days
**Priority:** CRITICAL PATH
**Goal:** Connect workflows to the Formula API for time/order/accessory calculations, and build the admin UI to configure workflows.

### Tasks

#### Phase 3 — Formula API Adjustments (3 days)

- [ ] Audit Formula API: ensure it can calculate operation time from BDC specific fields
- [ ] Audit Formula API: ensure it can calculate operation order (sequence)
- [ ] Audit Formula API: ensure it can calculate accessory consumption quantities
- [ ] Provide full context to Formula API: specific fields, generic fields, fabric properties, accessory references
- [ ] Integrate Formula API calls into `InstantiateWorkflow` (populate `OrderValue`, `TimeValue`, `QuantityValue`)
- [ ] Validate formula codes during workflow configuration (fail gracefully with meaningful error messages)
- [ ] Performance validation: ensure instantiation scales with BDC volume

#### Phase 4 — Configuration UI (5 days)

- [ ] **Operations Catalog form** — CRUD for the `Operation` entity (Code, Name, Description, IsActive)
- [ ] **Family Workflow Config form** — multi-level editor:
  - Family → Workflow → Statuses (ordered) → Operations (ordered) → Accessories
  - Assign `TimeFormulaCode`, `OrderFormulaCode` per operation
  - Assign `QuantityFormulaCode` per accessory
  - Enforce one active workflow per family
- [ ] **Movement Type Mapping form** — map KW movement codes to internal movement types (admin configurable)
- [ ] **NEV Import Config form** — SFTP settings, monitored folder path, schedule
- [ ] **Personnel Management form** — CRUD for operator records (Matricule, FirstName, LastName, IsActive)

### Deliverables

- Formula API adjusted and validated for all calculation types
- All 5 admin configuration forms functional

### Acceptance Criteria

- Admin can create a workflow for a family with statuses, operations, and accessories
- Formula codes are validated on save; invalid codes show a clear error
- Workflow instantiation populates calculated values (time, order, quantity) correctly
- Personnel management allows creating/editing/deactivating operators

---

## Sprint 3 — Talon System + Post Confirmation

**Duration:** 9 days
**Priority:** HIGH — enables production use
**Goal:** Deliver the full barcode talon flow for textile operators and update technical dept post confirmation to use operations.

### Tasks

#### Phase 5 — Talon System (5 days)

- [ ] **Talon Print Report** — 4.1 × 1.5 cm thermal label per operation, containing:
  - BDC Number + Client
  - Operation Name + Code
  - Barcode encoding `BdcWorkflowOperation.Id` (globally unique PK)
  - Calculated Time
  - Generation Date
  - Print order: scanline / row-major (left-to-right, top-to-bottom, status order then operation order)
- [ ] Talon report triggered when BDC report is printed; user selects printer
- [ ] **Talon Scanning form** (supervisor interface):
  - Scan barcode → display operation info
  - Select worker from dropdown (by name)
  - Confirm button → marks operation complete, assigns worker
  - Recent completions list for verification
- [ ] **BDC Workflow Viewer form** — read-only view of a BDC's full workflow instance (operations, status, completion info)
- [ ] **Anomaly View form**:
  - List all auto-completed (skipped) operations per BDC (`IsAutoCompleted = true`)
  - Allow supervisor to manually mark a skipped operation as completed (retroactively)
  - Allow assigning a worker to the manually completed operation

#### Phase 6 — Post Confirmation Update (4 days)

- [ ] Modify Post Confirmation flow (Technical dept): completing a post now marks `BdcWorkflowOperation` records as complete in order
- [ ] UI enforces operation sequence (Technical dept): operator cannot skip to a later operation
- [ ] Ensure BDC status derived from operation completion (not overridden by manual status change)
- [ ] Regression test: all existing post confirmation paths continue to work
- [ ] Update BDC Detail form: add `RfidNumber` display field + "View Workflow" button

### Deliverables

- Talon labels can be printed for any BDC
- Supervisor can scan talons and record completions
- Anomaly View shows all skipped operations
- Technical dept post confirmation works via operations

### Acceptance Criteria

- Scanning a talon barcode correctly identifies the operation and BDC
- Completing all operations under a status automatically advances BDC status
- Skipped operations (auto-completed) appear in Anomaly View with NULL worker and timestamp
- Technical dept: scanning an out-of-order operation is blocked by the UI
- Talon labels print in correct scanline order

---

## Sprint 4 — KW NEV / IC2S Integration

**Duration:** 7 days
**Priority:** MEDIUM — client-specific
**Goal:** Implement the full fabric roll reception flow for Kwantum: NEV import, confrontation, validation, and IC2S response.

### Tasks

#### Phase 7 — KW NEV / IC2S (7 days)

- [ ] **NEV XML Import**: parse KW's weekly NEV file (via SFTP or local folder); create `NevImport` + `NevRoll` records with "ND" location; validate XML format, article existence, roll uniqueness
- [ ] **Roll Label Printing**: print labels for incoming rolls
- [ ] **Confrontation — Case 1 (Over-reception)**: scanned quantity > expected → flag surplus rolls, require manual decision before validation
- [ ] **Confrontation — Case 2 (Under-reception)**: scanned quantity < expected → flag missing rolls, allow closing NEV with partial validation
- [ ] **NEV Validation**: create proper `FabricRoll` records on confirmation
- [ ] **Trigger File Mechanism**: scanning a roll writes a `.txt` trigger file to a monitored folder; Communicator picks it up for downstream processing (decoupled, no direct API call)
- [ ] **NEV Closure**: lock NEV from further changes
- [ ] **IC2S Response Generation**: XML response after NEV closure, includes reception tracking status (full / partial / with surplus); sent via SFTP
- [ ] **NEV Import List form**: view all NEV imports and their status
- [ ] **NEV Import Detail form**: confrontation UI (over/under), validation action, closure

### Deliverables

- Full NEV reception flow operational
- IC2S response generated and sent after NEV closure
- Trigger file mechanism in place for Communicator

### Acceptance Criteria

- KW NEV XML file imports successfully; rolls visible in import detail
- Over-reception: surplus rolls flagged, validation blocked until resolved
- Under-reception: missing rolls flagged, partial closure allowed
- Scanning a roll creates the `.txt` trigger file
- IC2S XML is generated with correct reception status and sent via SFTP

---

## Sprint 5 — KW File Exports + RFID Printing

**Duration:** 7 days
**Priority:** MEDIUM
**Goal:** Deliver all outbound KW movement files with deduplication tracking, and implement RFID label printing for suspended articles.

### Tasks

#### Phase 8 — KW File Exports + FileSendService (4 days)

- [ ] **FileSendService**:
  - `RegisterFile(clientId, fileName, fileType)` → creates `ClientFileSendLog` record
  - `MarkSent(logId)` / `MarkFailed(logId, error)`
  - `HasAlreadySent(clientId, fileName, fileType)` → deduplication check
- [ ] **PI2S Generation**: triggered after cut plan confirmation; contains article, palette, quantity; dedup check applied
- [ ] **SA2S Generation**: triggered on stock adjustments / overconsumption; uses `ClientMovementTypeMapping` for KW codes; dedup check applied
- [ ] **GOMV Generation**: suspended articles (Rideaux Suspendus); includes rod number (per-Store sequential counter from `StoreRodCounter`) + RFID number; triggered at delivery finalization
- [ ] **VOWV Generation**: all non-suspended textile orders (selection by `Family.Type` in code); rod counter = 0 for non-suspended; triggered at delivery finalization
- [ ] **BDC Status Export**: map internal statuses to KW codes 1–15; all active KW orders included regardless of status; scheduled via existing Communicator mechanism
- [ ] Register all file types in `ClientFileSendLog` on generation

#### Phase 9 — RFID Printing (3 days)

- [ ] **RFID Enable Check** (in code, no Formula API): `bdc.Family` type is suspended AND fabric quantity from `CalculatedSpecificField` — enable/disable "Print RFID" button on Packaging form accordingly
- [ ] **RFID Number Generation**: sequential 9-digit counter from `GlobalCounter`; admin-configurable starting value (one-time alignment with Winsat legacy counter)
- [ ] **2-Label Rule**: if article's `nombre de cintres` (from `CalculatedSpecificField`) = 2, generate and print 2 RFID labels for that BDC
- [ ] **RFID Printing**: send ZPL to Zebra ZT411 printer; store `RfidNumber` on Bdc record
- [ ] **Reprint Permission**: reprinting generates a new number and requires explicit permission
- [ ] Packaging form: add "Print RFID" button with enable/disable logic

> **Pending from client:** ZPL label template (visual layout + RFID write command) — implement once received.

### Deliverables

- PI2S, SA2S, GOMV, VOWV, BDC Status files generated and tracked
- No duplicate file sends (dedup enforced via `ClientFileSendLog`)
- RFID labels print to Zebra ZT411 for suspended articles

### Acceptance Criteria

- PI2S generated after cut plan confirmation; `ClientFileSendLog` record created
- Attempting to send the same file twice is blocked by dedup check
- RFID button disabled on non-suspended BDCs
- RFID button prints 2 labels when `nombre de cintres` = 2
- RFID counter starts at the configured value (Winsat alignment)
- Rod counter increments per Store (GOMV) and is 0 for VOWV

---

## Sprint 6 — Testing + Reports

**Duration:** 9 days
**Priority:** REQUIRED before go-live
**Goal:** Full integration testing across all features, UAT support, and delivery of worker performance reports.

### Tasks

#### Phase 10 — Integration Testing & UAT (5 days)

- [ ] End-to-end test: BDC creation → workflow instantiation → operation completion → status change → BdcStatusLog
- [ ] End-to-end test: talon print → talon scan → worker assignment → status advancement
- [ ] End-to-end test: skip rule → auto-complete → Anomaly View → manual retroactive completion
- [ ] End-to-end test: Technical dept post confirmation with enforced operation order
- [ ] End-to-end test: NEV import → confrontation → validation → IC2S response
- [ ] End-to-end test: PI2S / SA2S / GOMV / VOWV / Status export with dedup validation
- [ ] End-to-end test: RFID label generation and printing (2-label rule)
- [ ] Regression test: existing BDCs, reports, status filters, post confirmation
- [ ] Regression test: KW technical invoicing unchanged
- [ ] Performance test: workflow instantiation with large BDC volume
- [ ] UAT support: assist client during acceptance testing, fix reported issues

#### Phase 11 — Reports (4 days)

- [ ] **Worker Performance Report**: operations per worker (count), total calculated time per worker, by date range
- [ ] **Operation Summary Report**: operations completed per BDC, completion time vs. calculated time
- [ ] Validate Talon Print report rendering and label dimensions (4.1 × 1.5 cm) on actual thermal printer
- [ ] Ensure all existing reports using Status continue to work unchanged

### Deliverables

- All features validated end-to-end
- Worker performance and operation summary reports available
- UAT sign-off from client

### Acceptance Criteria

- All acceptance criteria from Sprints 1–5 pass in integration
- Worker performance report shows correct totals per worker
- No regression in existing report or workflow behavior
- Client UAT feedback addressed

---

## Sprint 7 — Accessory Stock + KW Invoicing + Deployment

**Duration:** 8 days
**Goal:** Deliver the two newest feature additions (Accessory Stock Management and KW Textile Invoicing), then deploy to production.

### Tasks

#### Phase 13 — Accessory Stock Management (3 days)

> **Note:** Detailed UI layout, movement types, and export format to be communicated by client before implementation.

- [ ] **Stock Entry** (FR-ACC-01): manual entry of accessory stock; organized into main stock and temp stock (same model as FabricRoll)
- [ ] **Stock Reduction on Operation Completion** (FR-ACC-02): when a `BdcWorkflowOperation` is marked complete, deduct `BdcWorkflowOperationAccessory.QuantityValue` from accessory stock; follow same movement logic as fabric cuts
- [ ] **Overconsumption** (FR-ACC-03): if actual quantity exceeds calculated quantity, create overconsumption record (same pattern as existing fabric overconsumption)
- [ ] **Stock Export** (FR-ACC-04): export accessory stock movements (format and trigger per client spec)

#### Phase 14 — KW Textile Invoicing (3 days)

> **Note:** Invoice report template and XML invoice format to be communicated by client before implementation.

- [ ] **Price Resolution** (FR-INV-01): for KW textile BDCs, billing price from `ImportedPrice` by `ProductPriceCode`; block invoicing with explicit error if no `ImportedPrice` found; generate internal matrix price (PreInvoice) as reference/comparison only
- [ ] **Reversed Simulation** (FR-INV-02): new simulation view showing imported KW price as base and Windeco calculated price as comparison (inverse of current technical simulation)
- [ ] **Invoice Report** (FR-INV-03): KW-specific textile invoice report (extend/replace existing `InvoiceKwantum` — template pending client)
- [ ] **XML Invoice via Communicator** (FR-INV-04): new Communicator file type for KW textile invoice XML; each line includes article price codes, unit prices, quantities; sent via existing SFTP mechanism (format pending client)
- [ ] Update `InvoiceService.cs` KW branch (ClientId 1 and 413): add textile switch based on `Family.Type`

#### Phase 12 — Deployment (2 days)

- [ ] Final pre-deployment regression run
- [ ] Apply DB migrations to production environment
- [ ] Configure admin starting value for RFID counter (align with Winsat legacy)
- [ ] Configure SFTP credentials for KW file exchange
- [ ] Configure NEV monitored folder path
- [ ] Client admin configures family workflows (client responsibility)
- [ ] Deploy WinTech application update
- [ ] Deploy Communicator update
- [ ] Monitor production during hypercare period; address critical issues

### Deliverables

- Accessory stock management operational
- KW textile invoicing logic in place (report and XML pending client specs)
- Production deployment complete

### Acceptance Criteria

- Completing an operation with accessories reduces accessory stock correctly
- Overconsumption record created when actual > calculated accessory quantity
- KW textile BDC invoicing uses `ImportedPrice` as billing base
- Invoicing blocked with clear error when no `ImportedPrice` found
- Production environment running new version with all migrations applied
- RFID counter aligned with Winsat legacy end value

---

## Deferred (Out of Scope for This Release)

| Item | Reason |
|------|--------|
| Stock Photo (weekly KW snapshot) | Moved to future phase |
| Performance calculation formula | Future iteration |
| Badge scanning / Matricule-based worker selection in scanning | Deferred until textile line is live and badge readers are in place |
| SWIBTIME integration | Not in scope |
| Aléas (production incidents) | Not in scope |
| Operations not linked to BDC | Not in scope |

---

## Open Items (Blocking or Pending)

| # | Item | Blocking |
|---|------|---------|
| 1 | ZPL label template for Zebra ZT411 RFID packaging label | Sprint 5 — RFID label printing |
| 2 | KW Textile invoice XML format (example file) | Sprint 7 — FR-INV-04 |
| 3 | KW Textile invoice report layout/template | Sprint 7 — FR-INV-03 |
| 4 | Accessory stock UI layout, movement types, export format | Sprint 7 — FR-ACC |
| 5 | Winsat legacy RFID counter end value | Sprint 5 — RFID counter start config |
| 6 | New fields needed on Client/Fabric/Product tables (if any) | Sprint 1 — DB schema |

---

## Key Success Criteria

1. Textile production runs with talon-based barcode workflow
2. Technical production continues working (smooth transition, no regression)
3. KW receives all required XML files without duplicates
4. BDC status tracking is preserved, accurate, and derived from operation completion
5. Worker performance data is captured for reporting
6. Skipped operations are visible and manageable via Anomaly View
7. RFID labels print correctly for suspended articles on Zebra ZT411
