# WinTech Textile Production Features
## Technical and Functional Analysis - V2

**Document Version:** 2.0
**Date:** February 16, 2026
**Based on:** Client Specification V2 (revised February 2026)

> **Reading guide:** Items marked with <mark>yellow highlight</mark> represent **changes or additions introduced in specification V2**. Time estimates are also highlighted where they have changed.

---

## Executive Summary

This document provides the technical and functional analysis for extending WinTech to support textile production. The project consists of two main feature sets:

1. **Operation-Based Workflow System** (HIGH priority) - A dynamic workflow engine applicable to ALL product families (textile and technical) with operation tracking, time calculation, and worker performance tracing.

2. **KW Client Integration** (MEDIUM priority) - Client-specific XML file exchanges for Kwantum's fabric stock management and reporting needs.

**Key Decisions:**
- Workflow applies to ALL families (smooth transition from current static workflow)
- Step/Sub-step concepts map to existing Status entities (backward compatible)
- Operations are the new atomic work unit under each Status
- "Gamme" (product variant) handled via existing specific fields + Formula API (no new entity)
- KW integration is client-specific (not a general feature)
- <mark>Textile operations use barcode scanning (talon-based) — skipped operations do not block status progression</mark>
- <mark>Technical department operations use Post confirmation — order IS enforced in UI</mark>

**Technology:** .NET Framework 4.8 (no upgrade required)

**Estimated Total Effort:** ~<mark>57</mark>-<mark>68</mark> person-days

---

## 1. Functional Analysis

### 1.1 Feature 1: Operation-Based Workflow System

#### 1.1.1 Overview

Transform WinTech from a static global workflow to a **dynamic per-BDC workflow** where each BDC receives its own workflow instance based on family, product, and specific field values.

**Scope:** ALL product families (Textile: Tenture, etc. AND Technical: Enrouleur, Duo, Vertical, Skylight, PJ)

#### 1.1.2 Current vs Target State

| Aspect | Current | Target |
|--------|---------|--------|
| Workflow definition | Global static chain (WorkflowStep) | Per-family configuration |
| Tracking unit | Status only | Status + Operations |
| Time calculation | None | Per operation via Formula API |
| Accessory tracking | None | Per operation via Formula API |
| Worker assignment | Post-based | Operation-based (with Post) |
| Parallel paths | Hardcoded edge case | Natural via operation ordering |

#### 1.1.3 Workflow Structure

```
Family
  └── FamilyWorkflow (configuration)
        └── Statuses (ordered sequence)
              └── Operations (ordered within status)
                    └── Accessories (consumed by operation)

BDC Created
  └── BdcWorkflow (instance, immutable)
        └── BdcWorkflowOperations (with calculated values)
              └── BdcWorkflowOperationAccessories
```

**Key insight:** "Gamme" (product variant) is NOT a new entity. The combination of specific field values already exists in `CalculatedSpecificField`. Formula API determines which operations apply and their parameters.

#### 1.1.4 Functional Requirements

**FR-WF-01: Workflow Configuration**
- Admin configures workflow per family
- Each workflow has ordered statuses
- Each status has ordered operations
- Each operation has: TimeFormulaCode, OrderFormulaCode
- Each operation can have accessories with QuantityFormulaCode
- Only one active workflow per family

**FR-WF-02: Workflow Generation**
- Triggered when BDC is created (Communicator or manual)
- System identifies family, product, loads active workflow
- For each operation: calls Formula API to calculate OrderValue, TimeValue
- For each accessory: calls Formula API to calculate QuantityValue
- Creates BdcWorkflow + child records (immutable snapshot)

**FR-WF-03: Operation Completion**
- Two completion modes depending on department type:
  - **Real-time (Technical dept):** Worker at Post clicks confirm — operations must be completed in order; UI enforces sequence
  - **Batch via Talon (Textile dept):** Supervisor scans barcode labels — <mark>skipped operations do NOT block status; see FR-TAL-04 for handling</mark>
- Completion marks: IsCompleted, CompletedAt, CompletedById
- When ALL operations under a status complete → BDC status changes
- BdcStatusLog created for audit trail

**FR-WF-04: Backward Compatibility**
- Status-based queries/filters work unchanged
- Reports using Status work unchanged
- Post confirmation works (marks operations complete)
- Existing BDCs without workflow continue normally

#### 1.1.5 Workflow Impact on Existing System

| Area | Impact |
|------|--------|
| BDC creation | Add workflow instantiation step |
| Post confirmation | Mark operations complete in order (Technical) |
| Status display | No change (derived from operations) |
| BdcStatusLog | Preserved (auto-created on status change) |
| Parallel status edge case | Removed (handled naturally by operations) |

---

### 1.2 Feature 2: Talon System (Operation Barcode Labels)

#### 1.2.1 Overview

Physical barcode labels ("talons") for textile operators who don't have PC access. Workers collect talons after completing operations, supervisor scans them to record completion.

#### 1.2.2 Talon Specification

- **Size:** 4.1 × 1.5 cm thermal label
- **Content:**
  - BDC Number + Client
  - Operation Name + Code
  - Barcode (<mark>encodes `BdcWorkflowOperation.Id` — globally unique DB primary key, uniquely identifies the exact operation instance on the exact BDC when scanned</mark>)
  - Calculated Time
  - Generation Date

#### 1.2.3 Functional Requirements

**FR-TAL-01: Talon Printing**
- Printed when BDC report is printed
- Report contains all talons for BDC, <mark>ordered by the scanline (row-major) print order: left-to-right, top-to-bottom — respecting status order then operation order within each status</mark>
- User selects printer (no specific model required)
- Grid format (can be cut/separated)

**FR-TAL-02: Talon Scanning Interface**
- New WinTech form for supervisor
- Scan barcode → Display operation info
- Select worker from dropdown (by name)
- Click Complete → Mark operation done, assign worker
- Recent completions list for verification

**FR-TAL-03: Worker Performance Tracking**
- Data captured: Worker, Timestamp, Expected Time
- Basic reporting: Operations per worker, Total time per worker
- Performance calculation formula: DEFERRED (future phase)

**FR-TAL-04: <mark>Skipped Operations Handling (NEW)</mark>**
- <mark>If an operation barcode is NOT scanned, but the **next** operation's barcode IS scanned:</mark>
  - <mark>The BDC advances to the status corresponding to the last scanned operation</mark>
  - <mark>The skipped operation is automatically marked as completed with: PostId = NULL, CompletedAt = NULL, CompletedById = NULL</mark>
- <mark>A dedicated **Anomaly View** interface allows supervisors to:</mark>
  - <mark>View all non-scanned (auto-completed) operations per BDC</mark>
  - <mark>Manually mark a skipped operation as completed (retroactively)</mark>
  - <mark>Assign a worker to the manually completed operation</mark>

<mark>**FR-TAL-05: Personnel Fields (NEW)**</mark>
- <mark>User entity extended with: Matricule, FirstName, LastName, IsActive — to support future operator identification</mark>
- <mark>Personnel management form (CRUD) added to configuration</mark>
- **Badge scanning / Matricule-based worker selection in scanning interface: DEFERRED** (future iteration once textile line is live and badge readers are in place)

---

### 1.3 Feature 3: KW Client Integration

#### 1.3.1 Overview

Client Kwantum (KW) requires specific XML file exchanges for their fabric stock and delivery tracking. **This is KW-specific, not a general feature.**

#### 1.3.2 NEV/IC2S (Stock Management)

**FR-KW-01: NEV Import**
- KW sends weekly fabric roll inventory (NEV XML file)
- WinTech imports via SFTP or local folder
- Creates fabric roll records with "ND" (not determined) location
- Validation: XML format, article existence, roll uniqueness

**FR-KW-02: Reception Workflow**
- Print roll labels
- Physical reception + barcode scanning
- <mark>Confrontation handles two distinct cases:</mark>
  - <mark>**Case 1 — Over-reception:** Scanned quantity > expected quantity → flag surplus rolls, require manual decision before validation</mark>
  - <mark>**Case 2 — Under-reception:** Scanned quantity < expected quantity → flag missing rolls, can close NEV with partial validation</mark>
- Validation → create proper FabricRoll records
- <mark>Scanning a roll generates a `.txt` trigger file (picked up by Communicator for downstream processing)</mark>
- NEV closure (locks further changes)

**FR-KW-03: IC2S Response**
- Generated after NEV closure
- <mark>Confirms received rolls to KW, including the reception tracking status (full / partial / with surplus)</mark>
- Sent via SFTP

#### 1.3.3 Movement Files

**FR-KW-04: PI2S (Cutting Movements)**
- Generated when KW fabric is cut
- Contains: article, palette, quantity
- Triggered after cut plan confirmation

**FR-KW-05: SA2S (Stock Adjustments)**
- Generated for overconsumption/adjustments
- Contains: article, palette, mutation code, quantity
- Movement type mapping (admin configurable)

**FR-KW-06: Stock Photo**
- <mark>**DEFERRED** — not in scope for this phase</mark>
- <mark>Weekly stock snapshot functionality will be addressed in a future version</mark>

#### 1.3.4 Delivery Reports

**FR-KW-07: GOMV/VOWV**
- GOMV: For "Rideaux Suspendus" family (suspended articles)
- <mark>VOWV: For **all textile non-suspended orders** (not limited to "Stores Bateaux" — selection logic is based on Family.Type in code)</mark>
- Includes rod number (sequential) and RFID number
- <mark>**Rod Number:** Sequential counter **per Store entity** (not global). Each Store has its own counter. Suspended articles use the counter; non-suspended articles receive value = 0.</mark>

**FR-KW-08: BDC Status Export**
- Map internal statuses to KW codes (1-15)
- XML export on schedule (existing Communicator mechanism)
- <mark>All active KW orders are included in the export regardless of their current status; each order's status is mapped to the KW code (1–15) per the mapping table</mark>

**FR-KW-09: <mark>File Send Tracking (NEW)</mark>**
- <mark>Each generated file (PI2S, SA2S, IC2S, GOMV, VOWV, Status) must be tracked in a dedicated log table</mark>
- <mark>Deduplication rule: each file is sent **only once** per generation cycle; no duplicate sends</mark>
- <mark>Tracking fields: FileName, FileType, GeneratedAt, SentAt, Status (Pending/Sent/Failed), ErrorMessage</mark>
- <mark>New table: `ClientFileSendLog`</mark>

---

### 1.4 Feature 4: RFID Printing

#### 1.4.1 Overview

Print packaging labels with RFID tags for certain textile families during the Emballage (packaging) operation.

#### 1.4.2 Functional Requirements

**FR-RFID-01: RFID Enable Trigger**
- <mark>RFID printing applies to **suspended articles** only (Rideaux Suspendus family)</mark>
- <mark>Determination is made in code by checking: (1) `bdc.Family` type is suspended, and (2) fabric quantity from `CalculatedSpecificField` — no Formula API call required</mark>
- <mark>The "Print RFID" button on the packaging form is enabled/disabled based on this check</mark>

**FR-RFID-02: RFID Number Generation**
- Sequential counter (9-digit)
- <mark>Starting number configured once by admin (to align with where Winsat's legacy counter ended — one-time migration configuration)</mark>
- Stored in Bdc.RfidNumber

**FR-RFID-03: RFID Printing**
- During packaging operation (Emballage)
- "Print RFID" button on packaging form
- Sends to RFID printer
- Reprint requires permission (generates new number)
- <mark>**2-label rule:** If the article's `nombre de cintres` (from CalculatedSpecificField) = 2, the system generates and prints **2 RFID labels** for that BDC</mark>

**FR-RFID-04: Pending Client Information**
- ~~Printer model~~ **Answered: Zebra ZT411**
- ZPL label template (visual layout + RFID write command for the packaging label)

---

### <mark>1.5 Feature 5: Accessory Stock Management (NEW)</mark>

#### <mark>1.5.1 Overview</mark>

<mark>Accessories consumed by workflow operations will have their own stock management system, mirroring the existing FabricRoll stock model. Stock is reduced automatically when an operation that uses the accessory is completed, and follows the same main stock / temp stock / movement / overconsumption pattern as fabric.</mark>

#### <mark>1.5.2 Functional Requirements</mark>

<mark>**FR-ACC-01: Accessory Stock Entry**</mark>
- <mark>Ability to enter accessory stock (manual entry or import — details TBD)</mark>
- <mark>Stock organized into main stock and temp stock (same model as FabricRoll)</mark>

<mark>**FR-ACC-02: Stock Reduction on Operation Completion**</mark>
- <mark>When a workflow operation is marked complete, the calculated accessory quantities (`BdcWorkflowOperationAccessory.QuantityValue`) are deducted from stock</mark>
- <mark>Follows the same stock movement logic as fabric cuts</mark>

<mark>**FR-ACC-03: Overconsumption**</mark>
- <mark>If actual quantity used exceeds calculated quantity, an overconsumption record is created</mark>
- <mark>Same pattern as existing fabric overconsumption</mark>

<mark>**FR-ACC-04: Stock Export**</mark>
- <mark>Accessory stock movements exportable (format and trigger TBD — to be communicated by client)</mark>

> <mark>**Note:** Detailed design (UI layout, movement types, export format) will be communicated by client. Scope confirmed; estimate based on parity with existing FabricRoll stock system.</mark>

---

### <mark>1.6 Feature 6: KW Textile Invoicing (NEW)</mark>

#### <mark>1.6.1 Overview</mark>

<mark>KW textile orders follow a different billing logic than KW technical orders. The existing `ImportedPrice` table (already in WinTech) holds prices imported from KW's official price files. For textile, these imported prices are the **billing base** — the inverse of technical invoicing where the internal matrix price is the base.</mark>

#### <mark>1.6.2 Invoicing Logic Comparison</mark>

| | Technical KW (existing) | <mark>Textile KW (new)</mark> |
|---|---|---|
| Billing base | Internal matrix price (PreInvoice) | <mark>Imported KW price (ImportedPrice)</mark> |
| Reference / comparison | ImportedPrice | <mark>Internal matrix price (PreInvoice)</mark> |
| Block if missing | No | <mark>Yes — invoicing blocked if no ImportedPrice found</mark> |

#### <mark>1.6.3 Functional Requirements</mark>

<mark>**FR-INV-01: Price Resolution**</mark>
- <mark>For KW textile BDCs: billing price is looked up from `ImportedPrice` by article code (ProductPriceCode)</mark>
- <mark>If no matching `ImportedPrice` found → invoicing is blocked with an explicit error message</mark>
- <mark>Internal matrix prices (PreInvoice lines) are generated as a reference/comparison column only</mark>

<mark>**FR-INV-02: Simulation (Reversed)**</mark>
- <mark>Existing simulation for technical: shows Windeco matrix price and compares to imported client price</mark>
- <mark>New simulation for KW textile: shows imported KW price as base and compares to the Windeco calculated price — reversed presentation</mark>

<mark>**FR-INV-03: Invoice Report**</mark>
- <mark>A KW-specific textile invoice report will extend or replace the existing `InvoiceKwantum` report</mark>
- <mark>Report template design: **TBD — to be communicated by client**</mark>

<mark>**FR-INV-04: XML Invoice (Communicator)**</mark>
- <mark>A modified XML invoice file is sent to KW via Communicator</mark>
- <mark>Each line includes: article price codes, unit prices, quantities</mark>
- <mark>Exact XML format / example file: **TBD — to be communicated by client**</mark>
- <mark>Sent via existing Communicator SFTP mechanism (new file type added)</mark>

#### <mark>1.6.4 Notes on Existing Infrastructure</mark>

- <mark>`ImportedPrice` entity already exists and is linked to BDC via `BdcId` and to `ProductPrice` via `ProductPriceId`</mark>
- <mark>`InvoiceKwantum.cs` report class already exists — will be adapted for textile variant</mark>
- <mark>KW client branch already present in `InvoiceService.cs` (ClientId 1 and 413) — textile switch will be added here based on Family type</mark>
- <mark>No new DB table required; logic change only + new invoice report + new Communicator file type</mark>

#### <mark>1.6.5 Open Items for Next Meeting</mark>

- <mark>Highlight difference between technique invoice and textil invoice</mark>

---

## 2. Technical Analysis

### 2.1 Database Schema Changes

#### 2.1.1 New Entities

**Operation** (Master Data)
| Column | Type | Description |
|--------|------|-------------|
| Id | int PK | |
| Code | VARCHAR(50) UNIQUE | Operation code |
| Name | VARCHAR(255) | Display name |
| Description | VARCHAR(1024) NULL | |
| IsActive | bool | Default true |

**FamilyWorkflow** (Configuration)
| Column | Type | Description |
|--------|------|-------------|
| Id | int PK | |
| FamilyId | int FK | Reference to Family |
| Name | VARCHAR(255) | Workflow name |
| IsActive | bool | Only one active per family |
| CreatedAt | DateTime | |

**FamilyWorkflowStatus** (Status in Workflow)
| Column | Type | Description |
|--------|------|-------------|
| Id | int PK | |
| FamilyWorkflowId | int FK | |
| StatusId | int FK | Reference to existing Status |
| Order | int | Sequence order |

**FamilyWorkflowStatusOperation** (Operation Config)
| Column | Type | Description |
|--------|------|-------------|
| Id | int PK | |
| FamilyWorkflowStatusId | int FK | |
| OperationId | int FK | |
| TimeFormulaCode | VARCHAR(100) | Formula for time calculation |
| OrderFormulaCode | VARCHAR(100) | Formula for execution order |
| PostId | int FK NULL | Assigned workstation |

**FamilyWorkflowOperationAccessory** (Accessory Config)
| Column | Type | Description |
|--------|------|-------------|
| Id | int PK | |
| FamilyWorkflowStatusOperationId | int FK | |
| AccessoryId | int FK | |
| QuantityFormulaCode | VARCHAR(100) | Formula for quantity |

**BdcWorkflow** (Instance per BDC)
| Column | Type | Description |
|--------|------|-------------|
| Id | int PK | |
| BdcId | int FK UNIQUE | 1:1 with Bdc |
| FamilyWorkflowId | int FK | Source workflow |
| CreatedAt | DateTime | |

**BdcWorkflowOperation** (Operation Instance)
| Column | Type | Description |
|--------|------|-------------|
| Id | int PK | |
| BdcWorkflowId | int FK | |
| OperationId | int FK | |
| StatusId | int FK | Which status this belongs to |
| OrderValue | int | Calculated sequence |
| TimeValue | decimal NULL | Calculated time (minutes) |
| IsCompleted | bool | Default false |
| CompletedAt | DateTime NULL | |
| CompletedById | int FK NULL | Reference to User (worker) |
| <mark>IsAutoCompleted</mark> | <mark>bool DEFAULT false</mark> | <mark>True if completed via skip rule (no scan)</mark> |

**BdcWorkflowOperationAccessory** (Accessory Instance)
| Column | Type | Description |
|--------|------|-------------|
| Id | int PK | |
| BdcWorkflowOperationId | int FK | |
| AccessoryId | int FK | |
| QuantityValue | decimal NULL | Calculated quantity |

**ClientMovementTypeMapping** (KW-specific)
| Column | Type | Description |
|--------|------|-------------|
| Id | int PK | |
| ClientId | int FK | |
| MovementTypeId | int FK | |
| ClientCode | VARCHAR(10) | KW's code |

**NevImport** (KW-specific)
| Column | Type | Description |
|--------|------|-------------|
| Id | int PK | |
| Reference | VARCHAR(50) | NEV reference |
| ClientId | int FK | Always KW |
| ImportDate | DateTime | |
| IsClosed | bool | |
| FileName | VARCHAR(255) | |

**NevRoll** (KW-specific)
| Column | Type | Description |
|--------|------|-------------|
| Id | int PK | |
| NevImportId | int FK | |
| ArticleCode | VARCHAR(10) | |
| PaletteNumber | int | |
| Quantity | decimal | Meters |
| IsValidated | bool | |
| FabricRollId | int FK NULL | Created after validation |

**GlobalCounter** (Sequences)
| Column | Type | Description |
|--------|------|-------------|
| Id | int PK | |
| CounterName | VARCHAR(50) UNIQUE | e.g., "RFID" |
| CurrentValue | int | |
| PadLength | int | Zero-padding |

<mark>**StoreRodCounter** (NEW — KW-specific, rod number per store)</mark>
| Column | Type | Description |
|--------|------|-------------|
| <mark>Id</mark> | <mark>int PK</mark> | |
| <mark>StoreId</mark> | <mark>int FK UNIQUE</mark> | <mark>Reference to Store entity</mark> |
| <mark>CurrentValue</mark> | <mark>int</mark> | <mark>Sequential rod counter for this store</mark> |

<mark>**ClientFileSendLog** (NEW — KW-specific, file tracking)</mark>
| Column | Type | Description |
|--------|------|-------------|
| <mark>Id</mark> | <mark>int PK</mark> | |
| <mark>ClientId</mark> | <mark>int FK</mark> | |
| <mark>FileName</mark> | <mark>VARCHAR(255)</mark> | |
| <mark>FileType</mark> | <mark>VARCHAR(50)</mark> | <mark>e.g., PI2S, SA2S, IC2S, GOMV, VOWV, Status</mark> |
| <mark>GeneratedAt</mark> | <mark>DateTime</mark> | |
| <mark>SentAt</mark> | <mark>DateTime NULL</mark> | |
| <mark>Status</mark> | <mark>VARCHAR(20)</mark> | <mark>Pending / Sent / Failed</mark> |
| <mark>ErrorMessage</mark> | <mark>VARCHAR(1024) NULL</mark> | |

#### 2.1.2 Modified Entities

**Bdc**
- Add: `RfidNumber` VARCHAR(20) NULL

**Family**
- <mark>No new field needed — RFID eligibility determined in code from existing Family type and CalculatedSpecificField</mark>

<mark>**User** (UPDATED — personnel fields added)</mark>
- <mark>Add: `Matricule` VARCHAR(50) NULL UNIQUE</mark>
- <mark>Add: `FirstName` VARCHAR(100) NULL</mark>
- <mark>Add: `LastName` VARCHAR(100) NULL</mark>
- <mark>Add: `IsActive` bool DEFAULT true</mark>

#### 2.1.3 Indexes

- `BdcWorkflow.BdcId` - UNIQUE
- `BdcWorkflowOperation(BdcWorkflowId, StatusId, OrderValue)` - Query optimization
- `FamilyWorkflow(FamilyId, IsActive)` - Active workflow lookup
- `NevRoll.PaletteNumber` - Scanning lookup
- <mark>`BdcWorkflowOperation.IsAutoCompleted` - Anomaly view filtering</mark>
- <mark>`ClientFileSendLog(ClientId, FileType, Status)` - Dedup and send queue lookup</mark>
- <mark>`User.Matricule` - Operator lookup by badge scan</mark>

---

### 2.2 Service Layer

#### 2.2.1 WorkflowService (New)

**Responsibilities:**
- Load active workflow for family
- Instantiate BdcWorkflow from configuration
- Complete operations
- Calculate status from operation completion
- <mark>Apply skip rule: auto-complete preceding operations when out-of-order scan is detected</mark>

**Key Methods:**
```
GetActiveFamilyWorkflow(familyId) → FamilyWorkflow
InstantiateWorkflow(bdcId, familyWorkflowId) → BdcWorkflow
CompleteOperation(operationId, userId) → void
CompleteOperationByBarcode(barcodeId, userId) → CompleteOperationResult  [NEW]
GetCurrentStatusFromWorkflow(bdcWorkflowId) → Status
GetAutoCompletedOperations(bdcWorkflowId) → List<BdcWorkflowOperation>  [NEW]
```

#### 2.2.2 Formula API Adjustments

**Task:** Ensure Formula API handles all calculation requirements for the workflow system.

**Requirements:**
- Calculate operation time from BDC specific fields
- Calculate operation order (sequence)
- Calculate accessory consumption quantities
- Ensure performance scales with number of BDCs processed
- Provide all needed context: specific fields, generic fields, fabric properties, accessory references

**Estimated Effort:** 2-3 days for adjustments and validation

#### 2.2.3 NevService (New, KW-specific)

**Responsibilities:**
- Parse NEV XML files
- <mark>Handle two confrontation cases: over-reception and under-reception</mark>
- Generate IC2S response <mark>with reception status</mark>
- SFTP upload/download
- <mark>Write `.txt` trigger file on roll scan action</mark>

<mark>**FileSendService** (New, KW-specific)</mark>
- <mark>Responsibilities: Track all outbound files, enforce deduplication, log send results</mark>
- <mark>Key methods:</mark>
  - <mark>`RegisterFile(clientId, fileName, fileType)` → ClientFileSendLog</mark>
  - <mark>`MarkSent(logId)` → void</mark>
  - <mark>`MarkFailed(logId, error)` → void</mark>
  - <mark>`HasAlreadySent(clientId, fileName, fileType)` → bool (dedup check)</mark>

---

### 2.3 WinTech UI Changes

#### 2.3.1 New Configuration Forms

| Form | Purpose |
|------|---------|
| Operations Catalog | CRUD for Operation entity |
| Family Workflow Config | Multi-level editor: Family → Statuses → Operations → Accessories |
| Movement Type Mapping | KW movement code mapping |
| NEV Import Config | SFTP settings, schedule |
| <mark>Personnel Management</mark> | <mark>CRUD for operator records (Matricule, FirstName, LastName, IsActive)</mark> |

#### 2.3.2 New Production Forms

| Form | Purpose |
|------|---------|
| Talon Scanning | Scan barcodes, assign workers (by name), mark complete |
| BDC Workflow Viewer | Read-only view of BDC's workflow instance |
| NEV Import List | View NEV imports, status |
| NEV Import Detail | Confrontation (over/under), validation, closure |
| <mark>Anomaly View</mark> | <mark>List auto-completed (skipped) operations; allow manual completion + worker assignment</mark> |

#### 2.3.3 Modified Forms

| Form | Changes |
|------|---------|
| BDC Detail | Add: RFID number, "View Workflow" button |
| Packaging Form | Add: "Print RFID" button (<mark>enabled based on in-code check: suspended family + fabric quantity from CalculatedSpecificField</mark>) |
| Post Confirmation | Logic: mark operations complete in order (Technical dept — UI enforces sequence) |

#### 2.3.4 New Reports

| Report | Purpose |
|--------|---------|
| Talon Print | <mark>Operation barcode labels (4.1 × 1.5 cm), printed in scanline (row-major) order</mark> |
| Worker Performance | Operations per worker, time per worker |

---

### 2.4 Communicator Changes

#### 2.4.1 BDC Creation Extension

- After creating BDC, call WorkflowService.InstantiateWorkflow
- Log errors if workflow instantiation fails

#### 2.4.2 KW File Processing (KW-specific)

| Task | Trigger | Notes |
|------|---------|-------|
| NEV Import | Scheduled/manual | |
| IC2S Generation | NEV closure | <mark>Includes reception status in response</mark> |
| PI2S Generation | Cut plan confirmation | <mark>Dedup check via ClientFileSendLog</mark> |
| SA2S Generation | Stock adjustment | <mark>Dedup check via ClientFileSendLog</mark> |
| ~~Stock Photo~~ | ~~Weekly (Sunday)~~ | <mark>**DEFERRED**</mark> |
| GOMV Generation | Delivery finalization | <mark>Suspended articles only; rod counter per Store</mark> |
| VOWV Generation | Delivery finalization | <mark>All non-suspended textile orders; rod counter per Store</mark> |
| BDC Status Export | Scheduled | |

<mark>**Trigger File Handling (NEW)**</mark>
- <mark>When a roll is scanned during NEV reception, a `.txt` trigger file is written to a monitored folder</mark>
- <mark>Communicator picks up this file and initiates downstream processing</mark>
- <mark>This decouples WinTech UI from Communicator without direct API calls</mark>

---

## 3. .NET Version

**Decision:** Remain on .NET Framework 4.8

**Rationale:**
- DevExpress WinForms compatibility
- Migration effort not justified for this scope
- .NET Framework 4.8 has indefinite support via Windows
- Formula API already on .NET 10 (HTTP integration, no runtime conflicts)

**Impact:** None. All required libraries support .NET Framework 4.8.

---

## 4. Risks and Dependencies

### 4.1 Technical Risks

| Risk | Probability | Mitigation |
|------|-------------|------------|
| Formula errors block BDC creation | Medium | Validate formulas during config, good error messages |
| Technical dept disruption during migration | Low | Thorough testing, gradual rollout |
| RFID printer integration | Low | Need testing on printer model |
| Performance with many operations | Medium | Optimize queries, use caching |
| <mark>Winsat counter alignment (RFID start value)</mark> | <mark>Low</mark> | <mark>One-time admin config; document the procedure clearly</mark> |
| <mark>Confrontation edge cases (partial NEV)</mark> | <mark>Medium</mark> | <mark>Both over/under cases covered; manual override available</mark> |

### 4.2 Dependencies

| Dependency | Status |
|------------|--------|
| Formula API availability | Available |
| RFID printer specs | Pending client |
| KW SFTP access | To be configured |
| Admin to configure workflows | Client responsibility |
| <mark>Winsat legacy RFID counter end value</mark> | <mark>To be provided by client (one-time)</mark> |

---

## 5. Questions for Client

| # | Question | Impact | Status |
|---|----------|--------|--------|
| 1 | RFID printer model? | RFID implementation | <mark>**Answered: Zebra ZT411**</mark> |
| 1b | ZPL label template for the RFID packaging label (visual layout + RFID write command)? | RFID implementation | Pending |
| 2 | New fields needed on Client/Fabric/Product tables? | Database schema | Pending |
| 3 | KW Textile invoice XML format (example file)? | Feature 5 invoicing | Pending |
| 4 | KW Textile invoice report layout/template? | Feature 5 invoice report | Pending |

*Note: Formula configurations, status lists, and accessory lists will be configured by client admin during development.*

---

## 6. Implementation Roadmap

### 6.1 Phase Breakdown

| Phase | Description | Est. Days |
|-------|-------------|-----------|
| **Phase 1: Foundation** | Database schema, entity models, migrations (<mark>+User fields, +ClientFileSendLog, +StoreRodCounter, +IsAutoCompleted</mark>) | <mark>3</mark> |
| **Phase 2: Workflow Core** | WorkflowService, workflow instantiation, completion logic, <mark>skip rule & auto-complete logic</mark> | <mark>6</mark> |
| **Phase 3: Formula Integration** | Formula API adjustments, operation calculation (time, order, accessories) | <mark>3</mark> |
| **Phase 4: Config UI** | Operations catalog, Family workflow config, <mark>Personnel management form</mark> | <mark>5</mark> |
| **Phase 5: Talon System** | Talon report (<mark>scanline order</mark>), scanning interface (<mark>Matricule-based worker selection</mark>), <mark>Anomaly View form</mark> | <mark>5</mark> |
| **Phase 6: Post Confirmation** | Modify confirmation flow for operations (Technical dept, enforced order) | <mark>4</mark> |
| **Phase 7: KW NEV/IC2S** | NEV import, confrontation (<mark>over + under cases</mark>), IC2S export (<mark>with status</mark>), <mark>trigger file mechanism</mark> | <mark>7</mark> |
| **Phase 8: KW Files** | PI2S, SA2S, GOMV (<mark>per-store rod counter</mark>), VOWV (<mark>all non-suspended</mark>), Status export, <mark>FileSendService + dedup</mark> | <mark>4</mark> |
| **Phase 9: RFID** | RFID printing (<mark>Formula-driven, 2-label rule, Winsat start config</mark>) | <mark>3</mark> |
| **Phase 10: Testing** | Integration testing, UAT support | <mark>5</mark> |
| **Phase 11: Reports** | Worker performance, operation summary | <mark>4</mark> |
| **Phase 12: Deployment** | Migration, deployment, hypercare | <mark>2</mark> |
| <mark>**Phase 13: Accessory Stock Management**</mark> | <mark>Stock entry, stock reduction on operation completion, overconsumption, movements, export (details TBD)</mark> | <mark>3</mark> |
| <mark>**Phase 14: KW Textile Invoicing**</mark> | <mark>Billing logic switch (textile vs technical), block guard, simulation reversal, invoice report (once template received), XML invoice via Communicator (once format received)</mark> | <mark>3</mark> |

**Total: ~<mark>57</mark> person-days**

### 6.2 Priority Order

1. **Phases 1-3:** Foundation + Workflow Core + Formula (CRITICAL PATH)
2. **Phases 4-6:** Config UI + Talon + Post Confirmation (enables production use)
3. **Phases 7-8:** KW Integration (client-specific, can be parallel)
4. **Phase 9:** RFID (depends on client specs)
5. **Phases 10-12:** Testing, Reports, Deployment

### 6.3 Timeline Estimate

**With 1 developer:**
- ~<mark>57</mark> working days

**With contingency (20%):** ~<mark>68</mark> days

---

## 7. Summary

### 7.1 What We're Building

| Component | Scope | Effort |
|-----------|-------|--------|
| Workflow/Operations System | All families, core architecture change | LARGE (<mark>15 days</mark>) |
| Talon System (<mark>+ Anomaly View</mark>) | Textile department, printing + scanning + anomaly management | MEDIUM (<mark>13 days</mark>) |
| KW Integration (<mark>+ file dedup + trigger file</mark>) | One client, XML files | MEDIUM (<mark>11 days</mark>) |
| RFID Printing (<mark>code-driven check, 2-label rule, Zebra ZT411</mark>) | Suspended textile articles | SMALL (<mark>3 days</mark>) |
| <mark>Accessory Stock Management</mark> | <mark>All families using accessories</mark> | <mark>SMALL (3 days — details TBD)</mark> |
| <mark>KW Textile Invoicing</mark> | <mark>KW client, textile orders</mark> | <mark>SMALL (3 days — details TBD)</mark> |
| Supporting (config, reports, testing) | Various | MEDIUM (<mark>9 days</mark>) |

### 7.2 What We're NOT Building (Deferred)

- SWIBTIME integration
- Aléas (production incidents)
- Operations not linked to BDC
- Performance calculation formula
- ~~Accessory stock management (beyond existing)~~ <mark>NOW IN SCOPE — see Feature 5</mark>
- <mark>Stock Photo (weekly snapshot) — moved to future phase</mark>

### 7.3 Key Success Criteria

1. Textile production can run with talon-based workflow
2. Technical production continues working (smooth transition)
3. KW receives their required XML files
4. BDC status tracking preserved and accurate
5. Worker performance data captured for reporting
6. <mark>Skipped operations are visible and manageable via Anomaly View</mark>
7. <mark>Files are sent to KW without duplicates</mark>

---

## Appendix A: Entity Relationship Diagram

```
Family ─────────────────────────────────────────┐
  │                                              │
  └── FamilyWorkflow (1:M)                       │
        │                                        │
        └── FamilyWorkflowStatus (1:M)           │
              │                                  │
              ├── Status (M:1) ←─────────────────┤
              │                                  │
              └── FamilyWorkflowStatusOperation  │
                    │                            │
                    ├── Operation (M:1)          │
                    │                            │
                    └── FamilyWorkflowOperationAccessory
                          │
                          └── Accessory (M:1)

Bdc ──────────────────────────────────────────────────────┐
  │                                                        │
  └── BdcWorkflow (1:1)                                    │
        │                                                  │
        └── BdcWorkflowOperation (1:M)                     │
              │                                            │
              ├── Operation (M:1)                          │
              ├── Status (M:1) ←───────────────────────────┤
              ├── User [CompletedBy] (M:1)                 │
              │                                            │
              └── BdcWorkflowOperationAccessory (1:M)      │
                    │                                      │
                    └── Accessory (M:1)                    │

[KW-Specific]
Client (KW) ── NevImport (1:M) ── NevRoll (1:M) ── FabricRoll (1:1)
            ── ClientMovementTypeMapping (1:M)
            ── ClientFileSendLog (1:M)  [NEW]

Store ── StoreRodCounter (1:1)  [NEW]
```

---

## Appendix B: Summary of Changes from V2 Specification

The following items are new or updated relative to V1 specification:

| # | Change | Section |
|---|--------|---------|
| 1 | Barcode encodes BdcWorkflowOperation.Id (globally unique PK) | FR-TAL (talon content) |
| 2 | Talon print order follows scanline / row-major logic | FR-TAL-01 |
| 3 | Skipped operations do NOT block status in textile workflow | FR-TAL-04 |
| 4 | Auto-completed skipped ops: PostId=NULL, CompletedAt=NULL | FR-TAL-04 |
| 5 | Anomaly View: IN SCOPE — view/manually complete skipped ops + assign worker | FR-TAL-04 |
| 6 | Multiple operators per Post; identified by Matricule | FR-TAL-02, FR-TAL-05 |
| 7 | Personnel DB: User entity extended with Matricule, FirstName, LastName, IsActive | FR-TAL-05 |
| 8 | Roll scan generates .txt trigger file for Communicator | FR-KW-02 |
| 9 | Confrontation handles 2 cases: over-reception AND under-reception | FR-KW-02 |
| 10 | IC2S response includes reception tracking status | FR-KW-03 |
| 11 | File deduplication rule: each file sent only once per cycle | FR-KW-09 |
| 12 | New table: ClientFileSendLog (structured file send tracking) | FR-KW-09, DB schema |
| 13 | Stock Photo: DEFERRED to future phase | FR-KW-06 |
| 14 | RFID trigger: in-code check (suspended Family + fabric quantity CalculatedSpecificField) — no Formula API | FR-RFID-01 |
| 15 | RFID counter: configure starting number to align with Winsat legacy counter | FR-RFID-02 |
| 16 | 2 RFID labels printed if article's nombre de cintres = 2 | FR-RFID-03 |
| 17 | VOWV: applies to ALL non-suspended textile orders (not only Stores Bateaux) | FR-KW-07 |
| 18 | Rod number: sequential counter per Store entity (not global) | FR-KW-07 |
| 19 | <mark>Feature 5 added: Accessory Stock Management — main/temp stock, reduction on operation completion, overconsumption, movements, export</mark> | FR-ACC-01 to 04 |
| 20 | <mark>Feature 6 added: KW Textile Invoicing — billing base = ImportedPrice, simulation reversed vs technical, block if no price found</mark> | FR-INV-01 to 04 |

---

**END OF DOCUMENT**
