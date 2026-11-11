# Product Vision

## Executive Summary

### Product Vision

Rental Management is a Microsoft Dynamics 365 Business Central extension published by Abakion that turns standard sales documents into fully tracked equipment and asset rentals. Its vision is to let rental businesses run time-based, partially invoiced, and pause-aware rentals natively inside Business Central, without leaving the ERP for spreadsheets or side systems. The product is delivered as a cloud-target AL extension (`app.json` target `Cloud`, runtime 17.0, application 28.0.0.0, version 29.0.202645.181872) that augments rather than replaces the standard sales, transfer, and posting flows.

### Support for Strategy

The product supports Abakion's strategy of extending Business Central for specialist verticals while preserving upgrade safety. Several signals in the repository confirm this:

- **AppSource compliance as a hard constraint.** `CLAUDE.md` marks AppSource rules as non-negotiable: no breaking schema changes, all user-facing strings localized via `Label` with `Comment`, no `TestPermissions = Disabled` in production, and `includeSourceInSymbolFile: true`.
- **Partner ecosystem integration.** The dependency on `Abakion App Manager Core` (id `9cf9feca-8d2d-49f3-a03d-6ae05f910b2c`) shows the app is one module in a family of Abakion apps sharing a common foundation.
- **Managed delivery pipeline.** Builds run through Azure DevOps and ALOps, with Application Insights telemetry (`westeurope`) for operational visibility.
- **Localization readiness.** The `TranslationFile` feature is enabled, supporting multi-language rollout across Business Central markets.

### Differentiation

The differentiators visible in code are primarily functional depth in time-based pricing and rental lifecycle control:

- **Calendar-driven pause and speed modeling.** Rental calendars carry a `Time Factor` per interval (0% for full pause, fractional for partial pause, above 1 for "upticking"), and the Calculation Engine converts these into chargeable and non-chargeable time per billing step.
- **Price scales with step-wise billing.** Price scale lines define base price factors, max-amount caps, and step durations; `MergePauseIntoScale` interleaves calendar pauses with price steps so each segment carries both properties.
- **Daylight-savings-correct billing.** Dedicated DST handling (rounding to seconds rather than milliseconds, `Daylight Savings` flag on steps and calendar lines) addresses a real pain point for clock-time billing in Europe.
- **Mid-rental partial invoicing.** `InjectCutDateIntoScale` splits steps at the last invoiced date-time, so long rentals can be billed incrementally and posted invoice plans stay consistent with the calendar.
- **Edit protection during active rentals.** `CheckSalesLineFieldModificationError` blocks changes to Quantity, Unit Price, Location, Type, and related fields once a rental has started, preventing billing drift.
- **Transfer-aware rentals.** Rental transfer orders and receipts are linked to the rental; receipts cannot exceed the rented quantity and receipt timing is constrained to on or after rental commencement.

## Product Context

### Market Segment Alignment

This product maps directly to the segment defined in the Market Analysis: **Dynamics 365 Business Central Equipment & Asset Rental Operators** (`rental-management-equipment-rental-erp`). Alignment points:

- **Use case:** Equipment, vehicle, tool, and machinery rental (the "Rental Category" and "Rental Calendar" objects target reusable resources rather than consumables).
- **Platform:** Business Central cloud, matching the segment's technology profile.
- **Geography:** European focus, supported by the `westeurope` telemetry endpoint, Danish-style publisher conventions, `usedynamics.com` help domain, and DST-aware billing.
- **Buyer roles:** The posting procedures, invoicing plan page, and rental calendars serve finance managers and rental counter staff; the Business Central extension structure serves partner consultants and ERP administrators.

### Competitive Position

Within the Business Central ecosystem, rental functionality is typically either absent from standard BC or addressed by third-party AppSource extensions. Rental Management competes on:

- **Depth of pricing model**: calendar-based pauses, speed factors, price scales, and DST handling exceed what generic sales-price logic supports.
- **Native integration**: event subscribers hook into `Sales-Post`, `Sales Line`, `Transfer` and report flows instead of forcing a parallel order-entry process.
- **Partner backing**: Abakion's publisher identity and dependent app family suggest a consultancy-led go-to-market through partners.

Specific named competitors cannot be determined from the repository alone; the competitive position is inferred from functional scope.

### Strategic Fit

The app fits Abakion's Business Central extension portfolio as a vertical module that reuses shared foundations (`Abakion App Manager Core`) and follows a consistent `SCB 920 <Name>` object naming convention. It extends the Business Central sales model into a rental model without forking base objects, which keeps the product maintainable across BC upgrades.

## Ideal Customer Profiles

**Profile 1: Mid-market equipment rental company (primary)**
- Rents construction, industrial, or event equipment on time-based contracts.
- Operates Business Central cloud and has a finance function that issues periodic rental invoices.
- Needs partial invoicing for long-running rentals and clear audit trails for billing disputes.
- Decision maker: finance manager or operations/rental manager. Users: counter staff creating rental orders, AR clerks posting invoices.

**Profile 2: Vehicle, AV, or IT asset rental operator**
- Rents vehicles or technical equipment where pauses (maintenance, downtime) and partial-rate periods matter.
- Requires calendar-based pause handling and transfers between locations while rentals remain tracked.
- Decision maker: ERP administrator or finance lead.

**Profile 3: Business Central partner consultancy**
- Implements and maintains Business Central for rental clients.
- Needs a supported, AppSource-compliant extension with localized UI and stable upgrade paths.
- Acts as a channel for the product and as a technical user (developers configuring calendars, price scales, and integration points).

## Product and Module Scope

### In Scope

- Rental order and quote handling on standard Business Central sales documents, with rental-specific line fields and header dates (`SCB Rental Inv. Start Date`, `SCB Rental Return Date`).
- Rental relation lines that track each rental line's document context, start date-time, and invoiced cut-off (`Rental Invoiced to Date-Time`).
- Rental calendars with pause and speed factors, including line- and header-level calendars with fallback.
- Price scales with step-based billing, base price and base duration, and max-amount caps.
- Invoicing plan generation, including Rental, Non-Rental, and Totals line types, and posted invoice plan records.
- Daylight-savings adjustments within calendar and scale calculations.
- Rental-aware sales posting procedures, including active and prepared rental checks before standard posting.
- Transfer order maintenance and transfer receipt/shipment integration for rented items.
- Edit guards on sales lines during active rentals.
- Copy utilities for scale and calendar lines between sales documents.
- Control add-in (`SCBRentalJavaScript`) that triggers an editable list-view toggle in the parent page.
- Localized labels and translation file support.

### Out of Scope

- Consumables, sales of owned inventory without a rental period, and standard sales pricing that does not involve rentals.
- Customer-facing rental portals or self-service booking interfaces (no such objects appear in the repository).
- Fleet maintenance scheduling, asset depreciation, or equipment condition tracking (not present in the codebase).
- Non-Cloud deployments: the manifest targets `Cloud` only.
- Replacement of Business Central's standard posting engine; the extension augments it.
- Rental analytics or dashboards beyond the invoicing plan pages (not evidenced in the provided files).

## Modules Overview

| Module | Responsibility | Representative Objects |
|---|---|---|
| Rental Management core | Rental lifecycle and line handling | `SCB 920 Rental Management` codeunit, `SCB 920 Rental Relation Line` |
| Calculation Engine | Calendar-to-scale conversion, invoicing plan, unit prices, DST | `SCB 920 Calculation Engine` codeunit, `SCB 920 Price Scale`, `SCB 920 Price Scale Line` |
| Rental Calendar | Pause and speed intervals | `SCB 920 Rental Calendar Line` |
| Invoicing Plan | Per-step billing plan and posted plan | `SCB 920 Rental Inv. Plan Line`, posted invoice plan records |
| Event Subscribers | Cascading sales, transfer, posting, and field-validation logic | `SCB 920 Event Subscriber` codeunit, `SCB Publishers` |
| Posting | Rental-specific posting and guards | `SCB 920 GlobalVarsSingleInst`, posting error and confirmation logic |
| Transfer Integration | Rented item transfer and receipt tracking | Transfer event subscribers, `TransferReceiptQtyErr` logic |
| UI Extensions | Page and page-extension customizations, control add-in | `page`, `pageextension`, `SCBRentalJavaScript` |
| Reporting | Rental-related reports including invoicing and deletion of invoiced orders | `report` folder, `Delete Invoiced Sales Orders` subscriber |
| Setup & Permissions | Rental setup, default rental category, permission sets | `SCB 920 Rental Setup`, `permissionset` folder |
| Notifications | App enable check gating rental logic | `SCB 920 Notification Mgmt.` codeunit |

## Features Overview

- **Rental order management**: Create rental lines on sales orders and quotes; rental relation lines are auto-created on insert and kept in sync on modify and delete.
- **Time-based rental pricing**: Base duration and base price drive unit-price calculation across the rental period.
- **Price scales**: Step-wise pricing with per-step price factors and max-amount caps.
- **Rental calendars**: Pause, partial pause, and uptick modeling via time factors; header and line calendars with fallback.
- **Daylight-savings handling**: DST-aware chargeable time and step adjustments.
- **Invoicing plan**: Automatic generation of rental steps, non-rental lines, and totals; regenerated on demand with optional progress dialog.
- **Partial invoicing**: Cut-off date-time splits steps so already-invoiced portions are separated from open ones.
- **Posted invoice plan**: Plan records copied to posted invoices at posting time.
- **Copy utilities**: Copy price scale and calendar lines between sales headers or lines.
- **Rental posting procedures**: Standard posting warns or blocks on rental documents with active or prepared rentals, directing users to rental posting.
- **Edit protection**: Field-level validations block unsafe changes once a rental has started.
- **Transfer order maintenance**: Rental transfer orders are created, maintained, and deleted with their sales orders.
- **Transfer receipt controls**: Receipt quantity cannot exceed rented quantity; receipt timing must be on or after rental start.
- **Document deletion safeguards**: Sales headers with active rental relation lines cannot be deleted.
- **Rental category resolution**: Rental category is resolved from item, variant, and default setup.
- **Localized, telemetry-enabled delivery**: Translation file support and Application Insights integration.

## Functional Requirements

**FR-1: Rental line creation**
The system shall create a rental relation line automatically when a rental sales line is inserted, and delete it when the sales line is deleted.

**FR-2: Rental line synchronization**
The system shall update the corresponding rental relation line and maintain transfer orders when a rental sales line is modified.

**FR-3: Rental calendar conversion**
The system shall convert rental calendar entries for a given date range into scale steps, emitting "Time not covered by calendar" intervals where no entry exists.

**FR-4: Calendar time factors**
The system shall compute chargeable time from each calendar step's time factor, supporting 0% (full pause), fractional (partial pause), and above-1 (uptick) values.

**FR-5: Pause merging**
The system shall merge calendar-derived pause steps with price scale steps so each billing segment carries both pause classification and price properties.

**FR-6: Cut-date splitting**
The system shall split any billing step crossing the last invoiced date-time into two steps meeting at that date-time.

**FR-7: Daylight-savings adjustment**
The system shall represent daylight-savings hour adjustments with second-level precision and exclude them from non-chargeable time.

**FR-8: Invoicing plan generation**
The system shall regenerate the invoicing plan for a rental order, including Rental, Non-Rental, and Totals line types, when triggered by the user or by posting.

**FR-9: Invoicing plan prerequisites**
The system shall refuse to calculate an invoicing plan when the rental invoice start date or rental return date is undefined, and display an explanatory message.

**FR-10: Posted plan persistence**
The system shall copy the invoicing plan to the posted invoice plan when a rental order is posted as an invoice.

**FR-11: Active rental posting guard**
The system shall warn or block standard sales posting for documents with active or prepared rentals, requiring the rental posting procedure for full functionality.

**FR-12: Document deletion guard**
The system shall prevent deletion of a sales header while active rental relation lines exist for it.

**FR-13: Active rental edit protection**
The system shall block changes to Quantity, Unit Price, Location Code, Type, Unit of Measure, and related fields on rental sales lines once the rental has started.

**FR-14: Transfer receipt quantity guard**
The system shall prevent receipt of more rented items than the rental quantity on a transfer line.

**FR-15: Transfer receipt timing guard**
The system shall prevent receipt of rental items before the rental commencement date-time.

**FR-16: Scale and calendar copy**
The system shall copy price scale and calendar lines between sales documents, preserving destination filters and reporting counts to the user.

**FR-17: Localization**
The system shall present all user-facing text as localized Labels with comments.

**FR-18: Telemetry**
The system shall send operational telemetry to the configured Application Insights instance.

## Dependencies

**Platform dependencies**
- Microsoft Dynamics 365 Business Central: runtime 17.0, application 28.0.0.0, platform 28.0.0.0.
- Standard BC modules used by the event subscribers: Sales (`Sales-Post`, `Sales Line`, `Sales Header`, `Sales Shipment Line`, `Sales Invoice Line`), Transfer (`Transfer Line`, `Transfer Receipt`), and reports including `Delete Invoiced Sales Orders`.

**Application dependencies**
- `Abakion App Manager Core` (id `9cf9feca-8d2d-49f3-a03d-6ae05f910b2c`), version 29.0.0.0, publisher Abakion. Required for the app to be enabled (`NotificationAppMgt.IsAppEnabled()` gates all subscribers).
- Abakion's `SCB Publishers` codeunit, referenced by subscribers for SWAP handling and the sales-line integration hooks.

**Build and delivery dependencies**
- Azure DevOps (`dev.azure.com/abakionscb`) for source control and build pipeline (definition 426).
- ALOps (www.alops.be) as build tooling.
- Application Insights (instrumentation key and `westeurope` ingestion endpoint).

**Tooling and standards**
- AL language toolchain with `TranslationFile` feature enabled.
- Generic AL standards referenced in `CLAUDE.md` (`al-standards.md` in `abakion/app-excellence-workflows`).
- Object ID ranges `6082563–6083562` and `6223181–6224180`, reserved per `app.json`.

**Assumptions**
- The `SCB Publishers` codeunit and `SCB 920 GlobalVarsSingleInst` codeunit are defined in files not included in the provided excerpts; their behavior is inferred from call sites.
- Help and marketing content are hosted at `usedynamics.com` and `abakion.com`; their content was not analyzed.