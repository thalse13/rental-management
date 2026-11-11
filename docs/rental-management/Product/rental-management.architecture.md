# Architecture Document

## System Overview

### Purpose

Rental Management is a Microsoft Dynamics 365 Business Central extension published by Abakion (publisher ID `Abakion`, app ID `aa6d164b-de93-435c-96c3-8f3029e3840a`, version `29.0.202645.181872`). It adds equipment and asset rental capabilities to standard Business Central sales documents. Rentals are modeled as sales order and quote lines that carry a rental period, a rental category, and a rental calendar. The system computes a time-based invoicing plan (with pauses, speed factors, price scale steps, and daylight-savings adjustments), supports partial invoicing of ongoing rentals, enforces edit protection on lines whose rental has started, and coordinates rental transfer orders and receipts.

The product is a cloud-target AL extension (`"target": "Cloud"`, `"runtime": "17.0"`, `"application": "28.0.0.0"`, `"platform": "28.0.0.0"`) that depends on `Abakion App Manager Core` (`9cf9feca-8d2d-49f3-a03d-6ae05f910b2c`, version 29.0.0.0). The dependency indicates that shared Abakion infrastructure (licensing, notifications, and related foundation objects) lives in a separate app.

### Architecture Style

The implementation is a layered, event-driven extension of the standard Business Central application:

- **Extension model, not a fork.** All integration happens through `[EventSubscriber]` attributes on standard tables (`Sales Line`, `Sales Header`, `Sales Shipment Line`, `Sales Invoice Line`, `Transfer` tables), standard codeunits (`Sales-Post`), and standard reports (`Delete Invoiced Sales Orders`). The base application is not modified.
- **Management codeunit layer.** Business logic is concentrated in `SCB 920 Rental Management`, `SCB 920 Calculation Engine`, `SCB 920 Notification Mgmt.`, and `SCB 920 GlobalVarsSingleInst`, consistent with the `CLAUDE.md` guidance that "Tables stay thin; business rules go in management codeunits."
- **Thin table layer.** Rental data lives in `SCB 920`-prefixed tables (`Rental Relation Line`, `Rental Calendar Line`, `Price Scale`, `Price Scale Line`, `Rental Inv. Plan Line`, `Rental Category`, `Rental Setup`, `Post. Rental Rel. Line`) with minimal triggers.
- **Pipeline/transform pattern for time-based pricing.** The calculation path is a sequence of pure-ish transforms over temporary record buffers: calendar → pause scale lines → price scale steps → merged steps → cut-date injection → daylight-savings merge → amount calculation → plan lines.
- **Feature-gated activation.** Every subscriber guards on `NotificationAppMgt.IsAppEnabled()` before acting, so the extension is a no-op when disabled.

### Key Design Principles

Derived from code, not from an explicit design document:

1. **Non-invasive integration.** Subscribers do nothing for non-rental records (`IsRentalSalesLine` / `IsRentalSalesOrder` checks), and are skipped for temporary records (`Rec.IsTemporary()`) and when `RunTrigger` is false.
2. **Edit protection after commencement.** Once a rental has started (`RentalLineStarted`), changes to quantity, pricing, location, dates, and item identity are blocked with `FieldModificationErr`. Pricing fields are blocked only when the rental has started, while quantity, type, and item reference are blocked in all rental states.
3. **Temporary buffers for calculations.** `TempPauseScaleLine`, `TempPriceScaleLine`, `TempMergedScaleLine`, and `TempCalendarLine` are declared `temporary` so calculation never writes intermediate state to the database.
4. **Idempotent plan regeneration.** `UpdateInvoicingPlan` deletes and rebuilds the plan on every call; this trades compute for consistency.
5. **Localisation by default.** Every user-facing string is a `Label` with a `Comment` (for example `FieldModificationErr`, `DeleteSalesHeaderErr`, `TransferReceiptQtyErr`), satisfying the AppSource requirement in `CLAUDE.md`.
6. **Wide, explicit SWAP handling.** A "SWAP" marker is passed via `xRec."Transaction Type" = 'SWAP'` as a hack to suppress master-data insert and delete during item swaps. This is a documented-in-code hack rather than a first-class contract.

## System Context

### External Systems

| System | Role | Evidence in code |
|---|---|---|
| Microsoft Dynamics 365 Business Central (Cloud) | Host platform; provides sales, transfer, posting, and reporting | `target: Cloud`, subscribers on standard objects |
| Abakion App Manager Core | Foundation app providing shared Abakion services | Declared dependency in `app.json` |
| Azure DevOps (`dev.azure.com/abakionscb`) | Source control and build pipeline (definition 426) | `build.url`, `source.repositoryUrl` |
| ALOps (alops.be) | Build orchestration / DevOps tooling | `build.by: "ALOps (www.alops.be)"` |
| Azure Application Insights (West Europe, `westeurope-3`) | Telemetry | `applicationInsightsConnectionString` |
| usedynamics.com / abakion.com | Help, EULA, privacy statement, marketing | `help`, `url`, `EULA`, `privacyStatement` |
| Azure AD (AppSource-signed) | Implicit via BC SaaS authentication | Standard BC cloud model |

No direct HTTP or REST integrations are visible in the reviewed AL files. The JavaScript control add-in (`SCBRentalJavaScript`) communicates only with the Business Central client host through `Microsoft.Dynamics.NAV.InvokeExtensibilityMethod('ControlAddInReady', [])`.

### Users & Actors

- **Rental counter staff / sales clerks.** Create rental quotes and orders, set rental start and return dates, and adjust lines before commencement. Interact with the Sales Order and Sales Quote pages and the rental relation list.
- **Finance / accounts receivable.** Post rental invoices using the rental posting procedure (`SCB 920 Rental Management` posting path), review the invoicing plan page, and handle partial invoicing.
- **Rental / operations managers.** Maintain rental categories, rental calendars (pauses, speed factors), and price scales; run the copy calendar and copy scale utilities.
- **ERP administrators and partner consultants.** Configure `SCB 920 Rental Setup` (default rental category, scale price source), manage permission sets (`permissionset` folder), and install or upgrade the extension.
- **Warehouse / transfer users.** Ship and receive transfer orders linked to rentals; receipts are constrained by the rental quantity and commencement date.
- **Abakion developers and the AI agents referenced in `CLAUDE.md`** (Solution Design, Code Generator, WatchDog reviewer) who maintain the codebase under `app/src/`.

## High-Level Architecture

### Components

The app is organized into the folders listed in `CLAUDE.md`: `codeunit`, `controlAddin`, `enum`, `page`, `pageextension`, `permissionset`, `report`, `table`, `tableextension`.

**Codeunits (business logic)**

- **`SCB 920 Calculation Engine` (codeunit 6083510).** The core rental calculator. Responsibilities:
  - Convert rental calendars into pause/active scale lines (`ConvertCalendarToScale`, `ReturnCalendarAsTempSet`).
  - Build and write the per-step invoicing plan (`UpdateInvoicingPlan`, `MergePauseIntoScale`, `InjectCutDateIntoScale`, `MergeDaylightSavingsAdjustments`, `CalculateScaleAmounts`).
  - Compute unit price factors and average prices (`CollectInvPlanUnitPriceFactor`, `CalcUnitPriceCalendarPrice`).
  - Copy calendar and scale lines between documents (`CopyCalendarLines`, `CopyScaleLines`).
  - Append non-rental sales lines and totals to the plan (`AddNonRentalSalesLinesToInvPlan`, `AddTotalsToInvoicingPlan`).
- **`SCB 920 Rental Management` (codeunit referenced from subscribers).** Owns rental relation line lifecycle: `InsertRentalRelationLine`, `ModifyRentalRelationLine`, `DeleteRentalRelationLine`, `FindRentalRelationLine`, `ActiveRentalRelationLineExists`, `IsRentalSalesLine`, `IsRentalSalesOrder`, `RentalLineStarted`, `MaintainTransferOrder`, `FindRentalCategory`, `UpdateSalesLineRentalLineAmount`, `UpdateSalesLineRentalReturnDate`, `RentalReturnDatePostProcessing`, and the per-field validators (`ValidateSalesLineType`, `ValidateSalesLineNo`, `ValidateSalesLineLocationCode`).
- **`SCB 920 Event Subscriber` (codeunit 6083140).** Central subscriber hub. Subscribes to 60+ events across `Sales Line`, `Sales Header`, `Sales Shipment Line`, `Sales Invoice Line`, `Sales-Post`, `Delete Invoiced Sales Orders`, and the transfer tables. Responsibilities:
  - Cascade insert/modify/delete from `Sales Line` into rental relation lines.
  - Guard edits per field with `CheckSalesLineFieldModificationError`.
  - Block unsafe posting via the standard `Sales-Post` path (`OnBeforePostSalesDoc`) and enforce rental posting (`ActiveRetalsExistErr`, `PreparedRetalsExistErr`).
  - Clean up posted rental relation lines when posted shipments or invoices are deleted.
  - Enforce receipt quantity and commencement date rules on transfer lines (`TransferReceiptQtyErr`, `ReceiptDateTimeErr`).
- **`SCB 920 Notification Mgmt.`.** Provides `IsAppEnabled()`, which every subscriber checks first.
- **`SCB 920 GlobalVarsSingleInst`.** Single-instance session state, notably `IsRentalSalesDocumentPosting(SalesHeader)`, used to coordinate rental-driven posting so the standard guard does not fire.
- **`SCB Publishers` (external codeunit).** Publishes integration events such as `OnBeforeSalesLineOnAfterValidateNoInsertMasterData` and `OnBeforeSalesLineOnAfterValidateTypeDeleteMasterData`. These are the hooks used for the SWAP path.

**Tables**

- `SCB 920 Rental Relation Line`: binds a sales line to a rental period (`Starting Date-Time`, `Rental Invoiced to Date-Time`, `Scale Price Source`, `Resource No.`, `Item Quantity`, `Sales Document Type/No.`, `Sales Line No.`).
- `SCB 920 Rental Calendar Line`: calendar intervals with `Starting Date-Time`, `Ending Date-Time`, `Time Factor`, `Description`, `Daylight Savings`, keyed by sales document and line (line 0 = header-level default).
- `SCB 920 Price Scale`: header of a price scale definition, referenced by `Price Scale Code`.
- `SCB 920 Price Scale Line`: per-step rows with `Step`, `Step Duration`, `Base Price Factor`, `Max Amount Base Price Factor`, `Time Factor`, `Chargeable Time`, `Non-chargeable Time`, `Daylight Savings`, `Calculation Price Basis`, `Base Price`, `Base Duration`, `Applied Unit Amount`, `Applied Max Amount`, `Average Unit Price`, `Closed`, `Original Step`.
- `SCB 920 Rental Inv. Plan Line`: the persisted invoicing plan with `Line Type` (Rental / Non-Rental / Totals), `Summation` flag, `Applied Amount`, `Start Date-Time`, `Start Date-Time Suffix`, `Average Unit Price`, `Base Price Factor`.
- `SCB 920 Rental Category`: resource-level definition (`Duration`, `Base Price`, etc.).
- `SCB 920 Rental Setup`: singleton setup with `Default Rental Category`, `Scale Price Source`.
- `SCB 920 Post. Rental Rel. Line`: posted copy of rental relation lines, keyed by `Posted Document Type` (Shipment or Invoice), `Posted Document No.`, `Posted Line No.`.
- Table extensions add `SCB Rental Line`, `SCB Rental Type`, `SCB Rental Return Date`, `SCB Rental Inv. Start Date`, and `SCB Rental Inv. Plan` fields to standard `Sales Header`, `Sales Line`, `Transfer Header`, and `Transfer Line`.

**Pages, page extensions, enums, reports, control add-ins**

- Pages: rental calendar, price scale, invoicing plan (`SCB 920 Rental Inv. Plan Line` list), rental relation list, and rental category card. The invoicing plan page reads the `Summation` rows for document totals.
- Page extensions: attach rental actions and fields to `Sales Order`, `Sales Quote`, `Sales Order Subform`, and similar standard pages.
- Enums: `SCB 920 Invoice Plan Line Type` (`Rental`, `Non-Rental`, possibly `Totals`), `SCB 920 Scale Price Source`, and rental line type (`Item`, `Resource`, etc.) referenced by `SCB Rental Type`.
- Reports: a Delete Invoiced Sales Orders extension (event-subscribed) and likely rental invoicing / plan reports.
- Control add-in `SCBRentalJavaScript`: a JavaScript add-in with `pushEditButton()` (clicks the `editListviewListToggle` button in the parent document) and a startup script that fires `ControlAddInReady`. Its purpose is UI polish for the list view edit toggle on rental pages.

**Permission sets**: one or more `permissionset` objects grant rights to the `SCB 920` tables and codeunits.

### Component interaction (summary)

```
Sales UI / Sales-Post / Transfer flows
        │  (OnAfterInsert, OnAfterModify, OnBeforeValidate, OnBeforePostSalesDoc, ...)
        ▼
SCB 920 Event Subscriber  ──guards──▶ SCB 920 Notification Mgmt. (IsAppEnabled)
        │
        ├──▶ SCB 920 Rental Management ──▶ SCB 920 Rental Relation Line
        │                                 SCB 920 Rental Category
        │                                 transfer orders (MaintainTransferOrder)
        │
        └──▶ SCB 920 Calculation Engine
                 │  temp buffers: Calendar, Pause scale, Price scale, Merged scale
                 ▼
             SCB 920 Rental Inv. Plan Line  (persisted plan)
                 │
                 ▼
             Invoicing plan page / posting (SCB 920 Post. Rental Rel. Line on post)
```

## Data Architecture

### Data Models

**Core entities and relationships** (inferred from the code, with key fields named in source):

```
Sales Header (1) ──< Sales Line (1) ──1 Rental Relation Line
     │                    │
     │                    └── SCB Rental Line, SCB Rental Type,
     │                        SCB Rental Return Date, SCB Rental Inv. Start Date
     │
     ├──< Rental Calendar Line  (keyed Sales Doc Type/No./Line No.; Line 0 = header default)
     ├──< Price Scale Line      (keyed Sales Doc Type/No./Line No.)
     │         └── Price Scale (by Price Scale Code, copied from templates)
     └──< Rental Inv. Plan Line (Line Type: Rental | Non-Rental; Summation rows for totals)

Rental Category (Resource-level)  ◀── Rental Relation Line.Resource No.
Rental Setup (singleton)          ──▶ Default Rental Category, Scale Price Source

Posted Shipment Line ──< Posted Rental Relation Line (Posted Document Type = Shipment)
Posted Invoice Line  ──< Posted Rental Relation Line (Posted Document Type = Invoice)

Transfer Header ──< Transfer Line  (with SCB Rental Inv. Plan etc.)
Transfer Order linked to a rental sales order via MaintainTransferOrder
```

**Key domain types:**

- `Rental Relation Line` key triple: `(Sales Document Type, Sales Document No., Sales Line No.)`. Carries `Starting Date-Time`, `Rental Invoiced to Date-Time`, `Scale Price Source`, `Item Quantity`, `Resource No.`.
- `Rental Calendar Line` key: `(Sales Document Type, Sales Document No., Sales Line No., Starting Date-Time)` (inferred). Fields: `Ending Date-Time`, `Time Factor` (0 = full pause, 0<x<1 = partial pause, >1 = upticking), `Description`, `Daylight Savings`, `Calendar Code`.
- `Price Scale Line` carries both price-step and calendar-merge state: `Step` (increments by 10), `Step Duration`, `Chargeable Time`, `Non-chargeable Time`, `Base Price Factor`, `Max Amount Base Price Factor`, `Time Factor`, `Calculation Price Basis`, `Base Price`, `Base Duration`, `Start Date-Time`, `Closed`, `Original Step`, `Daylight Savings`, `Applied Unit Amount`, `Applied Max Amount`, `Average Unit Price`, `Base 100 Value`.
- `Rental Inv. Plan Line` key includes `Sales Document Type`, `Sales Document No.`, `Sales Line No.`, `Line Type`, `Start Date-Time`, `Start Date-Time Suffix` (used as a collision suffix when two plan rows share the same start), `Summation`. Fields include `Applied Amount`, `Average Unit Price`, `Base Price Factor`, `Description`, `Step Duration`, `Daylight Savings Adjustment`, `Non-chargeable Time`, `Chargeable Time`, `Base 100 Value`.

**Domain constants and conventions visible in code:**

- Time is represented as `DateTime` and `Duration` (milliseconds in AL). The engine uses `Round(..., 1)` for normal rounding and `Round(..., 1000)` for daylight-savings steps, rounding to seconds so the one-hour DST shift is represented exactly.
- Base durations come from the rental category (`RentalCategory.Duration`) and rounded via `RoundDateTime(...)`.
- Base 100 values use `100000` as the scaling constant (`Base 100 Value / (TotalDuration / 100000)`), an internal normalization.
- Step numbers are multiples of 10 (`Step += 10`), with `InjectCutDateIntoScale` inserting `+1` splits.

### Data Flow

**1. Rental creation (sales line insert/modify)**

```
User enters rental sales line
   → Sales Line OnAfterInsert → RentalManagement.InsertRentalRelationLine
   → Sales Line OnAfterModify → ModifyRentalRelationLine + MaintainTransferOrder(2)
   → Sales Line OnAfterValidate (Quantity / Shipment Date)
        → UpdateSalesLineRentalLineAmount / UpdateSalesLineRentalReturnDate
```

**2. Invoicing plan generation**

```
UpdateInvoicingPlan(SalesHeader)
   1. Delete existing non-summation plan rows for the header
   2. Validate rental inv start/return dates on header
   3. Resolve base duration from Rental Category (default from Rental Setup)
   4. Compute ToDateTime = RoundDateTime(Return Date, BaseDuration)
   5. Compute CutDateTime = RoundDateTime(GetLatestRentalInvoiceDateTime, BaseDuration)
   6. GenerateUnitPriceCalendar + CalcUnitPriceCalendarPrice → overall base unit price
   7. ConvertCalendarToScale → TempPauseScaleLine (pause/active steps + "Time not covered")
   8. CreateTempScaleDataSet → TempPriceScaleLine (price scale steps)
   9. MergePauseIntoScale(pause, price, merged)
  10. InjectCutDateIntoScale(merged, CutDateTime)        // split at invoiced boundary
  11. MergeDaylightSavingsAdjustments(merged)
  12. CalculateScaleAmounts(merged)                      // amounts, base 100, avg unit price
  13. RentalInvPlanLine.UpdateFromScaleLineSet(merged)   // persist rental rows
  14. AddNonRentalSalesLinesToInvPlan                    // non-rental sales lines
  15. AddTotalsToInvoicingPlan                           // Summation rows
  16. AddUpdateTagToInvoicePlan                          // freshness tag
  17. MarkSuspenseEntriesAsPosted(SalesHeader, CutDateTime)
```

**3. Rental posting**

```
Sales-Post OnBeforePostSalesDoc
   → if active rental and not rental posting → Error or Confirm
Sales-Post OnBeforeDeleteAfterPosting
   → UpdateInvoicingPlan (refresh cut date)
   → InsertPostedInvoicePlan(SalesInvoiceHeader."No.")
   → MaintainTransferOrder(3)  // try-delete transfer orders
   → Delete rental relation lines for each rental sales line
Sales Shipment Line / Sales Invoice Line OnAfterDelete
   → Delete corresponding Post. Rental Rel. Line rows
```

**4. Transfer receipt**

```
Transfer Receipt Line validations
   → quantity cannot exceed rented quantity (TransferReceiptQtyErr)
   → receipt date/time must be on or after rental commencement (ReceiptDateTimeErr)
Partial transfer shipment
   → looks up sales line and inbound transfer line; errors if missing (SalesLineNotFoundErr, InboundTransferLineNotFoundErr)
```

**5. Edit protection**

```
Sales Line OnBeforeValidate (field X)
   → if rental line and rental started (for pricing / dates) or always (for identity fields)
        → CheckSalesLineFieldModificationError → Error(FieldModificationErr)
```

### Storage Strategy

- **Persistent storage** is standard Business Central SQL. All AL tables in the `SCB 920` prefix are persistent except the `temporary` buffers used in calculation.
-