---
status: Draft
targetDate: 2026-12-31
period: Q4 FY26
tshirt: M
---

# Integrate Rental management with Fixed asset module of business central

## Release Information
**Product/Module:** Rental Management for Business Central (Fixed Asset integration)
**Version:** 29.0.202645.181872

## Headline
Rental Management now links rented items to Business Central fixed assets, giving every asset a complete rental history.

## Summary
This release connects the Rental Management extension with the standard Fixed Asset module in Dynamics 365 Business Central. Each rental item is mapped one-to-one to a fixed asset, and the asset's record shows its rental transactions in a dedicated rental entry table. Fixed asset functionality itself is unchanged.

## Problem Statement
Companies that rent out equipment track the physical asset in the Fixed Asset module but track its rental activity separately in Rental Management. Finance and operations staff must switch between records to see how often an asset was rented, what it earned, and where it has been, which slows depreciation, utilisation, and audit reviews.

## Solution
Each rental item is mapped to exactly one fixed asset. Rental transactions for that item are written to a rental entry table on the fixed asset, so the asset record shows its full rental history alongside its standard book value and depreciation data. Existing fixed asset processes, postings, and depreciation logic remain untouched.

## Key Features
- One-to-one mapping between a rental item and a fixed asset
- Rental entry table on the fixed asset card, listing transactions from the linked rental item
- Standard fixed asset functionality left unchanged
- Rental transactions appear automatically as rentals are invoiced

## Customer Benefits
- Full rental and financial history visible on each fixed asset in one place
- Faster utilisation and profitability analysis per asset
- Cleaner audit trail linking physical assets to their rental

---
> ⚠️ This response was truncated because it exceeded the token limit. Use **Fit to limit** for a complete compressed version, or click **Continue generating** to extend it.