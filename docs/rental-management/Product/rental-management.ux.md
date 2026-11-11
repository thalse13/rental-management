---
status: Draft
---

# UX Design Document

## Design Vision

### Product Experience Goals

Rental Management is a back-office extension for Dynamics 365 Business Central. Its UX goal is to make time-based rental management feel like a natural extension of standard sales order handling, rather than a separate application. Rental counter staff and finance users should be able to create a rental quote or order, define its period and pricing, observe the resulting invoicing plan, and post partial or full invoices without leaving Business Central.

The experience is built around three goals evident in the code:

- **Continuity with standard BC.** Rentals are modelled as Sales Lines and Sales Headers (`SCB Rental Line`, `SCB Rental Type`, `SCB Rental Inv. Start Date`, `SCB Rental Return Date`), so users work in familiar sales pages.
- **Trust in the numbers.** Billing is precise (daylight-savings-aware rounding, step-wise price scales, cut-off dates for partial invoicing), and the UI is designed to explain the calculation through invoicing plan pages and descriptive step labels.
- **Protection against accidental changes.** Once a rental is in progress, fields that would silently alter billing are locked, and the system explains why.

### Design Principles

The following principles are inferred from the event subscriber and calculation engine code:

1. **Explain blocks, don't just block.** Every guarded field uses a dedicated message: "It's not allowed to modify field %1 because this %2 is part of a rental in process." Errors name the object and the reason.
2. **Confirm before destructive or ambiguous actions.** Posting a document containing active rentals triggers a `Confirm` dialog ("Active Rental(s) exist on Sales %1 %2. No Rental functionality pertaining to Transfer Orders will be executed… Do you wish to continue?"). Non-GUI sessions receive hard errors instead, so background processes are never silently altered.
3. **Route users to the correct flow.** Where standard posting would bypass rental logic, the error text directs the user: "use Rental Period Posting for full Rental functionality." Users are guided rather than left to guess.
4. **Keep rental logic out of the user's way.** Rental relation lines, transfer orders, and invoicing plan rows are created and maintained automatically by event subscribers. Users see outcomes, not plumbing.
5. **Make time visible.** Calendar pause and speed factors, step durations, chargeable vs. non-chargeable time, and "Time not covered by calendar" rows surface the billing logic as data.
6. **Progressive disclosure through totals.** The invoicing plan uses summation rows ("Totals") per line type, so users see aggregates first and can drill into steps.

### Brand Alignment

The brand is expressed through the publisher identity and assets:

- **Publisher:** Abakion, with logo `Assets/orange-a-216.png`, indicating an orange primary accent.
- **Product name:** "Rental Management", used consistently in the app manifest, page captions, and the `SCB` object prefix (`SCB 920 …`).
- **Help and legal:** Links to `usedynamics.com/rental-management/`, `abakion.com` privacy and licence pages, and `abakion.com/dynamics-365/apps/rental-management/`.
- **Tone:** Professional, precise, ERP-appropriate. Messages are declarative and technical (e.g., "Start-, Cut- and End DateTime must be in sequence"). Labels are mostly noun-phrase captions rather than marketing copy.

Visual brand expression is constrained by Business Central's design system. Abakion inherits BC's colours, typography, and layout, with the orange logo as the only custom visual signal.

## User Research Summary

### Target Users

Inferred from the codebase and the market analysis:

| Persona | Role | Primary Tasks in Code |
|---|---|---|
| **Rental counter clerk** | Sales/rental staff | Creating rental quotes and orders, entering rental start and return dates, selecting items and categories |
| **Rental planner / operations manager** | Operations | Maintaining rental calendars (pauses, speed factors), reviewing rental relation lines |
| **Finance / AR user** | Accounting | Posting rental invoices via Rental Period Posting, reviewing the invoicing plan and posted invoice plan |
| **ERP administrator / consultant** | Technical setup | Configuring the Rental Setup, Rental Categories, Price Scales, and transfer/location defaults |
| **Warehouse / logistics user** | Transfers | Creating and posting transfer shipments and receipts linked to rental orders |

### Key Insights

1. **Rental data lives on the sales document.** Users never enter rental data in a separate module; it is attached to sales lines. This means the mental model is "a rental is a sales line with time semantics."
2. **Time is the core dimension.** Almost every interaction involves a date-time range, a calendar, or a step. The code treats daylight-savings transitions and sub-hour rounding as first-class concerns, which suggests real user pain around clock changes.
3. **Partial invoicing is a common workflow.** The `Rental Invoiced to Date-Time` field, the `Closed` flag on steps, and `InjectCutDateIntoScale` show that users routinely bill open rentals in parts, so the UI must represent "already invoiced" vs "still open" clearly.
4. **Transfers are part of the rental lifecycle.** Rental orders generate transfer orders; the receipt quantity is capped by the rented quantity and cannot precede the rental start. Logistics users therefore see rental constraints inside standard transfer pages.
5. **Errors are the primary guidance channel.** Because the app extends existing pages, most guidance is delivered at the moment of attempted change rather than through onboarding or tooltips.

### Pain Points Addressed

| Pain Point | Evidence in Code | How the UX Responds |
|---|---|---|
| Edits silently corrupt billed rentals | Blocking of Quantity, Unit Price, Location, Type, Line Discount, etc. once `RentalLineStarted` | Explicit field-level error with field and object name |
| Accidental posting through standard BC | `OnBeforePostSalesDoc` checks for active/prepared rentals | Confirm dialog, or hard error for background/non-GUI contexts |
| Deleting a rental order orphans data | `SalesHeaderOnBeforeDelete` blocks delete while active rental lines exist | "You can't delete … while active associated records exist" |
| Over-receiving transfers | `TransferReceiptQtyErr` caps receipt at rented quantity | Error quoting quantity, UoM, item, and line number |
| Receipt before rental starts | `ReceiptDateTimeErr` | Blocking message stating the rental has not commenced |
| Confusing billing breakdown | Descriptive step labels ("Pause 100%, Price Scale Step 20", "Upticking 10%") | Plain-language descriptions in the invoicing plan |
| Clock change billing errors | Daylight-savings steps with 1000ms rounding | Visible "Daylight Savings" flag and adjustment column |

## Information Architecture

The product is an extension, so its IA is best described as a set of entry points added to Business Central's existing navigation, plus a set of rental-specific objects accessible from setup and from sales documents.

### Structural Map

```
Business Central Client
├── Sales & Receivables
│   ├── Sales Quotes / Sales Orders / Sales Invoices / Credit Memos   (extended)
│   │   ├── Sales Line card — rental fields (SCB Rental Line, SCB Rental Type,
│   │   │   SCB Rental Inv. Start Date, SCB Rental Return Date, Rental Category)
│   │   ├── Rental Relation Lines (per rental line)
│   │   ├── Rental Calendar Lines (per document or line)
│   │   ├── Price Scale Lines (per document or line)
│   │   ├── Rental Invoicing Plan (summary + line detail)
│   │   └── Posted Invoice Plan (history after posting)
│   └── Rental Period Posting (dedicated posting procedure)
├── Logistics
│   ├── Transfer Orders / Shipments / Receipts (extended with rental links)
│   └── Locations (rental-aware location defaults)
├── Rental Setup (administration)
│   ├── Rental Setup card (defaults: Default Rental Category, Scale Price Source)
│   ├── Rental Categories (Duration, base pricing)
│   ├── Price Scales (Base Price, Base Duration, scale steps)
│   ├── Rental Calendars (named calendar templates)
│   └── Permission sets (Rental user, Rental admin)
└── Posted Rental Data
    └── Posted Rental Relation Lines (per posted shipment/invoice line)
```

### Object Grouping Logic

Objects are grouped by lifecycle stage rather than by technical type:

1. **Configuration** — Rental Setup, Rental Category, Price Scale, Rental Calendar.
2. **Transaction** — Sales line extensions, Rental Relation Line, Rental Calendar Line, Price Scale Line.
3. **Planning and billing** — Rental Invoicing Plan, Inv. Plan Line, totals rows.
4. **Logistics** — Transfer order and receipt extensions.
5. **History** — Posted Rental Relation Line, Posted Invoice Plan.

### Navigation Principles

- The primary entry point for rental work is the sales document, not a separate rental list. Users reach rental data by opening a sales order and drilling into a rental line.
- Setup objects live under administrative navigation, consistent with other BC configuration.
- Posting uses a distinct action ("Rental Period Posting") rather than replacing the standard Post action, so the rental path is an explicit choice.

## User Flows

### Flow 1: Create a Rental Quote or Order

1. User opens **Sales Quote** (or **Sales Order**) and creates a new document with a customer.
2. User adds a sales line and selects an item or resource recognised as a rental category.
3. The `SalesLineOnAfterInsert` subscriber creates a rental relation line automatically, and the line is flagged `SCB Rental Line`.
4. User sets **Rental Inv. Start Date** and **Rental Return Date** on the sales line (or header).
5. The rental return date is validated; `UpdateSalesLineRentalReturnDate` and `UpdateSalesLineRentalLineAmount` recalculate the line amount.
6. User optionally opens **Rental Calendar** to define pauses or speed-ups for the period.
7. User opens **Price Scale** to define step-based pricing.
8. User opens **Rental Invoicing Plan** to review the computed plan, including totals.
9. User converts the quote to an order (standard BC action), and the rental data is carried across via the `CopyScaleLines` and `CopyCalendarLines` helpers.

### Flow 2: Maintain a Rental Calendar

1. User opens the rental document and selects **Rental Calendar** (at header or line level).
2. User adds calendar lines with **Starting Date-Time**, **Ending Date-Time**, **Time Factor**, and **Description**.
3. A Time Factor of 0% represents a full pause, a fraction represents a partial pause, and a value above 100% represents "upticking" (speeding up time).
4. The system fills gaps with "System Inserted" or "Time not covered by calendar" intervals, so the calendar is always complete.
5. User returns to the invoicing plan; the plan refreshes via `UpdateInvoicingPlan`.

### Flow 3: Review the Invoicing Plan

1. User opens the **Rental Invoicing Plan** page for an order.
2. The plan shows rows grouped by **Line Type** (Rental, Non-Rental), each with a summation row labelled "Totals".
3. Each step displays Start Date-Time, Step Duration, Chargeable Time, Non-chargeable Time, Base Price Factor, Applied Amount, and Daylight Savings Adjustment.
4. A cut-off line (from `InjectCutDateIntoScale`) marks the boundary between invoiced and open time.
5. User can see Average Unit Price across the period.

### Flow 4: Partial Invoicing of an Ongoing Rental

1. User opens the sales order and selects **Rental Period Posting**.
2. The system calculates the invoice up to the chosen cut-off date-time, using `GetLatestRentalInvoiceDateTime`.
3. Steps spanning the cut-off are split so the invoiced portion is closed (`Closed = true`).
4. Posted invoice plan rows are written via `InsertPostedInvoicePlan`.
5. The remaining open time stays available for a future invoice.

### Flow 5: Attempt a Standard Post on a Rental Order

1. User selects standard **Post** on a sales order containing active rental lines.
2. `OnBeforePostSalesDoc` detects the rental.
3. **GUI session:** a `Confirm` dialog states that no rental functionality will run for transfer orders and that the document will be deleted if fully posted. User may continue or cancel.
4. **Non-GUI session:** the posting fails with an error directing the user to Rental Period Posting.
5. If the user chooses Rental Period Posting, full rental logic runs.

### Flow 6: Attempt to Modify a Field on an Active Rental

1. User changes Quantity, Unit Price, Location, Type, No., Unit of Measure, Shipment Date, or a discount field on a rental line.
2. The validation trigger runs `CheckSalesLineFieldModificationError`.
3. If the rental has started, an error appears: "It's not allowed to modify field [Field] because this Sales Line is part of a rental in process."
4. The user cannot save the change until the rental is closed, reverted, or reassigned through appropriate means.

### Flow 7: Delete a Rental Order

1. User attempts to delete a sales order with active rental relation lines.
2. Deletion is blocked with "You can't delete [type] [no] while active associated records exist in the Rental Relation Line table."
3. For rental sales orders, transfer orders are maintained (deleted or retained) and the invoicing plan is cleared when the delete is permitted.

### Flow 8: Transfer Shipment and Receipt for a Rental

1. Warehouse user creates a **Transfer Order** from a rental sales order.
2. The transfer line carries the rental link; Quantity and Shipment date follow rental data.
3. On **Post Shipment** for a partial quantity, the system validates the sales line and inbound transfer line. Missing links produce `SalesLineNotFoundErr` or `InboundTransferLineNotFoundErr`.
4. On **Post Receipt**, receipt quantity cannot exceed the rented quantity (`TransferReceiptQtyErr`) and the receipt date-time cannot precede the rental start (`ReceiptDateTimeErr`).

### Flow 9: Rental Setup

1. Administrator opens **Rental Setup**.
2. Sets **Default Rental Category** (required by the invoicing plan) and **Scale Price Source** (rental or sales price).
3. Maintains Rental Categories with Duration and base pricing.
4. Defines Price Scales and Rental Calendars as reusable templates.

## Interaction Design

### Interaction Model

The product uses Business Central's standard interaction model: card pages for single records, list pages for collections, subpages for lines, and action bars with grouped actions. Rental interactions extend this model rather than introducing new paradigms.

### Input Patterns

- **Date-time entry.** Start and end values are entered as date-time pairs. The system rounds values to the category `Duration` (`RoundDateTime`), so the UI does not allow arbitrary sub-unit precision in the normal path.
- **Time factor entry.** Entered as a percentage or decimal on calendar lines. The UI labels describe the meaning ("Pause 100%", "Upticking 10%", "Partial Pause 25%").
- **Lookup-driven selection.** Rental categories, price scales, and calendars are selected from lookups tied to setup records.

### Feedback Patterns

| Situation | Feedback Type | Example |
|---|---|---|
| Blocked edit | Error dialog | "It's not allowed to modify field…" |
| Missing prerequisite | Message (non-blocking) | "The Invoice Plan has not been calculated as '%1' has not been defined" |
| Risky action | Confirm dialog | Standard posting on a rental order |
| Copy success | Message | "%1 lines copied and inserted from %2." |
| Empty source | Message | "No lines available in %1." |
| Long operation | Progress dialog | "Updating Invoicing Plan..." |

Messages use Business Central's native `Message`, `Confirm`, `Error`, and `Dialog` types so they render consistently with the host.

### State Feedback

- **Rental in process.** Locked fields become the primary state signal. Users discover the state when an edit is attempted; the app does not add a dedicated status badge in the code shown.
- **Invoicing plan freshness.** An update tag is written by `AddUpdateTagToInvoicePlan`, allowing the page to detect whether the plan is stale.
- **Closed steps.** The `Closed` flag indicates invoiced time, supporting visual differentiation in the plan.

### Interaction Consistency

- Actions that rebuild derived data (invoicing plan, scale lines) are triggered automatically from upstream edits, so users do not need a separate "recalculate" step for routine changes.
- Copying lines always deletes the destination before inserting, so the result is predictable.

## Visual Design Direction

### Foundation

Visual design inherits the Business Central theme: Microsoft's Fluent-based colour palette, typography, spacing, and component styles. Abakion does not ship custom themes, CSS, or fonts in the repository.

### Accent Colour

The only custom visual asset visible is the publisher logo at `Assets/orange-a-216.png`. The orange mark suggests orange as an accent for Abakion-branded surfaces (app tile, about page), but the repository does not define it as a UI colour token.

### Iconography

The app is a logo-only deliverable. Icon use inside pages follows BC's standard icon set. No custom SVG or PNG icons beyond the logo are present in the source.

### Typography

Typography is inherited from BC. Captions and labels are written in sentence case with Title Case for object captions (e.g., "Rental Relation Line", "Rental Invoicing Plan"). Labels use `Comment` attributes for translator context.

### Layout Direction

- Rental data appears in subpages and FastTabs within sales documents, consistent with BC's list-and-detail pattern.
- Invoicing plan layouts group rows by line type, with summation rows visually distinct (labelled "Totals", flagged `Summation = true`).
- Calendar and price scale subpages present ordered rows by start date-time and step number.

### Localisation as a Design Constraint

All user-facing text is localised through `Label` declarations, and the `TranslationFile` feature is enabled. Designs must allow for text expansion, right-to-left considerations where BC supports them, and date/number format variation across locales. Date-time displays rely on BC's regional formatting.

## Component Patterns

### Pattern Inventory

The following components are inferred from the code and BC's extensibility model:

| Component | Implementation | Usage |
|---|---|---|
| **Rental fields on sales lines** | Page and table extensions adding `SCB Rental Line`, `SCB Rental Type`, `SCB Rental Inv. Start Date`, `SCB Rental Return Date` | Sales Quote, Order, Invoice, Credit Memo line subpages |
| **Rental Relation Line** | Table with page for drill-down | Per rental line tracking of dates, invoiced cut-off, resource, scale source |
| **Rental Calendar Line** | Table with editable subpage | Pauses, speed factors, descriptions, daylight-savings flags |
| **Price Scale Line** | Table with editable subpage | Step, Step Duration, Base Price Factor, Max Amount, Time Factor |
| **Rental Invoicing Plan** | Read-oriented list page with summation rows | Plan review and billing preview |
| **Posted Invoice Plan** | Read-only list | Historical billing record |
| **Rental Setup** | Card page (singleton, `Get('')`) | Global defaults |
| **Rental Category** | Card and list | Duration and base pricing per category |
| **Dialog-based progress** | `Dialog` with fixed text label | Long plan rebuilds |
| **Copy actions** | Code-driven copy with `Message` result | Copy calendars and scales between documents |

### Dialog and Message Patterns

- **Error:** Terminal, used for invariant violations and blocked edits.
- **Confirm:** Used where the user can proceed deliberately (standard posting on rental documents).
- **Message:** Used for non-blocking info, including copy results and missing prerequisites.
- **Progress dialog:** Used for invoicing-plan rebuilds when `ShowDialog` is true.

### Naming and Caption Patterns

Captions follow BC's noun-phrase convention. Step descriptions generated by the calculation engine ("Time not covered by calendar", "Price Scale Step %1", "Pause 100%, Price Scale Step %1") are written as short phrases suitable for a description column.

### Data Patterns

- Summation rows are identified by a boolean flag (`Summation`) and a fixed description ("Totals"), so they can be styled or filtered out.
- Line types are an enum (`SCB 920 Invoice Plan Line Type`) with values Rental and Non-Rental, which drive grouping.

## Accessibility

Accessibility is largely determined by Business Central's platform, since Abakion's UI is built on standard pages, dialogs, and actions. The repository does not contain explicit accessibility declarations, so the following is an inference with recommendations.

### Inherited Platform Behaviour

- Keyboard navigation, screen-reader labelling, and focus order come from BC's page framework.
- Captions and tooltips are sourced from `Caption` and `ToolTip` properties on pages and fields, which should be provided for every rental field.

### Text and Messaging

- Error and confirm messages state the object and field involved, which supports screen-reader users who depend on message content rather than visual context.
- Dialogs use plain language and avoid colour-only signalling.

### Recommended Practices

- Every rental field should carry a `ToolTip` explaining its billing effect (e.g., Time Factor, Daylight Savings).
- Summation rows should be distinguishable by text, not only by styling.
- The Upticking and Partial Pause labels should be reviewed for clarity to users unfamiliar with the terminology.
- Translator comments (`Comment` attributes) should be kept current so localized messages remain accurate for assistive technology users.

### Open Questions

- Whether any custom page controls (the `SCBRentalJavaScript` control add-in) include ARIA attributes. The visible script only triggers a standard edit toggle button, so behaviour depends on the host page markup.

## Responsive Design

Business Central is a desktop-first web client with a responsive layout that adapts to tablet and narrow widths through its own rules. Rental Management does not define breakpoints or responsive CSS in the repository.

### Layout Adaptation

- Rental subpages and invoicing plan lists collapse to fewer visible columns on narrower widths, following BC's standard behaviour. Users should be able to scroll horizontally and use the Show More Fields affordance.
- Card pages (Rental Setup, Rental Category) stack fields vertically at narrow widths.

### Input Considerations

- Date-time entry on touch devices relies on the BC client's native date and time pickers.
-