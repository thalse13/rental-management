---
status: Draft
targetDate: 2026-12-31
period: Q4 FY26
tshirt: M
---

# Rental item availability

## Release Information
**Product/Module:** Rental Management (Dynamics 365 Business Central extension)
**Version:** 29.0.202645.181872

## Headline
Rental Management adds a rental availability matrix so counter staff can see at a glance when each rental item is free to rent.

## Summary
This release introduces a matrix window showing rental item availability across dates and calendar periods. Location codes indicate when an item is available or unavailable, so users can plan rentals without leaving Business Central. The view is built on the existing rental calendar and sales order model.

## Problem Statement
Rental staff currently check item availability by opening individual sales orders or rental lines one at a time. This is slow, error-prone, and makes it hard to spot double bookings or idle capacity across a period. Staff cannot quickly answer "when is this item free?" when quoting a customer.

## Solution
A new availability matrix window displays rental items as rows and dates or calendar periods as columns. Each cell shows whether the item is available or rented at the relevant location code, based on the rental period and calendar already stored on rental lines. Users can switch between day, week, and month views to match their planning horizon.

## Key Features
- Matrix view of rental items by date or calendar period
- Location code indicators showing availability per cell
- Day, week, and month period switching
- Drill-down from a matrix cell to the underlying rental line or sales order
- Availability driven by existing rental calendars, so no duplicate data entry

## Customer Benefits
- Quote rentals faster by seeing free periods in one view
- Avoid double bookings through clear visibility of committed periods
- Improve asset utilisation by spotting idle capacity
- Reduce reliance on manual checks across multiple orders

## Customer Quote (Imagined)
> "Our counter

---
> ⚠️ This response was truncated because it exceeded the token limit. Use **Fit to limit** for a complete compressed version, or click **Continue generating** to extend it.