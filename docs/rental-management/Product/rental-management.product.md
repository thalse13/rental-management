---
status: Draft
---

# Product Vision

## Executive Summary

### Product Vision

Abakion Rental Management is a rental management app for Microsoft Dynamics 365 Business Central. It lets equipment rental businesses run the full rental lifecycle inside the ERP they already use: rentals are quoted and ordered on standard Business Central sales documents, tracked physically through shipments and transfers, priced by customer and duration, billed on scheduled invoicing plans, and closed through returns, damage handling, and cancellations.

*Source note: Content is drawn from search summaries of abakion.com/apps/rental-management and docs.abakion.com/docs/rental-management. Full-page wording was not retrieved, so feature details should be checked against the complete documentation.*

### Support for Strategy

The product supports a strategy of:

1. **Extending Business Central instead of replacing it.** Rental operations run on standard Business Central sales orders, planning, and warehouse functions, so rental and sales share one system of record.
2. **Making rented assets traceable.** Serial-number tracking, warehouse locations, and transfer orders show where each machine is and what it has earned.
3. **Preventing double-bookings.** Business Central planning and reservations keep rental availability consistent across orders.
4. **Supporting flexible rental pricing.** Prices vary by customer, customer group, currency, variant, rental unit, and duration scale, with rental calendars for pauses and reduced-rate periods.
5. **Serving combined rental and sales businesses.** Rental items can also be sold directly, and rented and sold items can ship together.

### Differentiation

Relative to spreadsheets, standalone rental software, and generic ERP add-ons, the product differentiates on:

* **Native Business Central experience.** Rentals use standard sales orders, shipments, invoices, and posting, so finance and operations teams do not manage a separate system.
* **Rental lifecycle depth.** Shipment swaps a rental item for a rental-fee resource line, and returns, partial returns, lost or damaged items, replacements, suspensions, extensions, and cancellations are modeled explicitly.
* **Flexible pricing and billing.** Duration scales, rental units, rental calendars, and invoicing plans (weekly or monthly) are built in.
* **Rent and sell in one flow.** Rental items can be sold to the same customer, and mixed rental and sales shipments are supported.

## Product Context

### Market Segment Alignment

The product targets equipment rental operators that already use, or are considering, Microsoft Dynamics 365 Business Central. The segment is inferred to include construction, event, and industrial rental businesses with fleets of serial-numbered equipment and both rental and sales activity. Segment sizing and budget ranges are not yet researched and must be validated against the Market Analysis.

| Segment Characteristic                                      | Product Response                                                                       |
| ----------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Business Central as the system of record                    | Rental app built on standard sales orders, planning, and posting                       |
| Individual, serial-numbered machines                        | Serial-number tracking and warehouse locations                                         |
| Rentals with variable durations and customer-specific terms | Duration scales, rental units, and customer or group price rules                       |
| Long or recurring rentals                                   | Rental calendars and scheduled invoicing plans                                         |
| Returns, damage, and changes during rentals                 | Full and partial returns, damage handling, replacements, extensions, and cancellations |
| Mixed rental and sales businesses                           | Rented and sold items on the same order and shipment                                   |

### Competitive Position

Competitive context is inferred and has not been researched:

* **Standalone rental management software:** Often separate systems that must be integrated with accounting. The product competes on native Business Central integration.
* **Generic Business Central or ERP add-ons:** May lack rental-specific logic such as rental calendars, duration scales, and return handling. The product competes on rental depth.
* **Spreadsheets and manual processes:** Common in smaller operators. The product competes on availability accuracy and billing automation.

Current position: the product is an existing app with published documentation. Competitor pricing, market share, and go-to-market data are not yet available and should come from SCOUT's research.

### Strategic Fit

* **Fit with company/project strategy:** The product extends the Business Central ecosystem with a vertical rental app, which fits a focused, platform-based strategy.
* **Fit with market timing:** Not researched. Demand for rental management within Business Central should be validated.
* **Fit with capability:** The published feature set covers the rental lifecycle. Delivery against that scope must be confirmed against the Architecture document and the app source.
* **Risks to fit:** Dependence on the Business Central platform and its release cycle. Tax, deposit, and contract rules may vary by country. Payment and e-signature needs may require partners.

## Ideal Customer Profiles

### ICP A — \[Segment / Industry / Region]

#### Company Profile

* **Size:** 10-200 employees — to be validated against Market Analysis (inferred: small to mid-sized rental businesses with multiple locations)
* **Industry:** Equipment rental (construction, event, industrial), inferred
* **Maturity:** Operators already running Microsoft Dynamics 365 Business Central, or planning to adopt it (inferred)

#### Pain Points Addressed by Product

* Rental and sales orders managed in separate systems, causing double entry and reconciliation work
* Overlapping bookings and unclear availability for serial-numbered machines
* Difficulty tracking where each rented item is and when it returns
* Inconsistent pricing across customers, durations, and currencies
* Manual billing for long or recurring rentals, and manual handling of damaged or lost items

#### Key Persona

* **Role:** Rental manager or operations lead, working with a finance or ERP administrator
* **Responsibilities:** Rental orders, shipments, transfers, returns, and billing in Business Central
* **Goals:** Accurate availability, correct pricing, timely billing, and full tracking of each machine
* **Challenges:** Balancing rental and sales workflows, handling returns and damage, and keeping ERP data consistent
* **Buying Criteria:** Native Business Central fit, rental-specific functionality, and support for pricing and invoicing rules

## Product and Module Scope

### In Scope

* Rentals added to standard Business Central sales orders
* Shipment that converts a rental item to a rental-fee resource line
* Transfer orders for physical movement of rental items
* Availability and reservations using Business Central planning
* Serial-number tracking and warehouse locations for rental assets
* Pricing by customer, customer group, currency, variant, rental unit, and duration scale
* Rental calendars for pauses and reduced-rate periods
* Invoicing plans for scheduled (weekly or monthly) billing
* Full and partial returns, lost or damaged items, replacements, suspensions, extensions, and cancellations
* Direct sale of rental items and combined rental and sales shipments

### Out of Scope

* Replacing Business Central core finance, general ledger, or inventory functions
* Standalone operation without Business Central
* Payment acquiring and e-signature services (delegated to partners, inferred)
* Equipment telematics hardware
* Dealer sales, floor-plan financing, and construction project management
* Native desktop applications

## Modules Overview

### Module 1

* **Purpose & Objectives:** Core rental management inside Business Central: rental orders, shipment and transfer, availability and serial tracking, pricing and invoicing plans, and returns and changes, so each rental is booked, tracked, billed, and closed correctly.
* **Key Personas Targeted:** Rental manager, operations and warehouse staff, finance or ERP administrator
* **Dependencies:** Microsoft Dynamics 365 Business Central (planning, reservations, warehouse, serial tracking, sales and posting); Abakion documentation for configuration (scope of published features to be confirmed)

## Features Overview

### Feature 1

* **Quick Description:** Rental availability and lifecycle on Business Central sales orders, covering rental orders, shipment to rental-fee lines, transfers, returns, and serial tracking, so that rentals do not double-book and every machine's status is known.
* **Module:** Module 1 — Core rental management
* **Key Personas Targeted:** Rental manager, operations and warehouse staff

## Functional Requirements

### FR-1: \[Requirement Name]

* **Description:** The system shall let users add rental items to a Business Central sales order, ship them as rental-fee resource lines, and track their physical movement through transfer orders, with availability checked through Business Central planning and reservations.
* **Priority:** Must
* **Acceptance Criteria:**
  * [ ] A rental item can be added to a sales order alongside sold items.
  * [ ] Shipping a rental item creates a rental-fee resource line in place of the item line.
  * [ ] A rental booking that overlaps an existing reservation for the same serial-numbered asset is blocked with a conflict message.

### FR-2: \[Requirement Name]

* **Description:** The system shall price rentals by customer, customer group, currency, variant, rental unit, and duration scale, apply rental calendars and invoicing plans, and handle returns, extensions, and cancellations.
* **Priority:** Must
* **Acceptance Criteria:**
  * [ ] Rental prices resolve correctly for customer, group, currency, variant, rental unit, and duration.
  * [ ] Invoicing plans generate scheduled weekly or monthly billing for a rental.
  * [ ] Partial returns, lost or damaged items, and cancellations update rental status and charges correctly.

## Dependencies

### Other Products or Services

* Microsoft Dynamics 365 Business Central (host platform for sales, planning, warehouse, and posting) (confirmed by sources)
* Payment and e-signature providers, if required by customers (inferred, not in source summaries)

### Shared Services

* Business Central identity, permissions, and company or environment setup (inferred)
* Business Central reporting and posted-document storage for rental invoices (inferred)