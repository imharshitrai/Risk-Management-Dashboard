# Borrowing Base and Collateral Risk Dashboard for Asset-Based Lending

A Python and Excel-based credit monitoring tool that models an **asset-based lending (ABL)** portfolio: eligible collateral, advance rates, reserves, borrowing base availability, and credit risk exceptions. The tool ingests borrower-level accounts receivable and inventory data and produces a **borrowing-base-certificate-style Excel workbook** suitable for an ABL analyst or portfolio risk workflow.

---

## Table of Contents

- [Overview](#overview)
- [Why This Project](#why-this-project)
- [Key Concepts Modeled](#key-concepts-modeled)
- [What It Shows](#what-it-shows)
- [Project Structure](#project-structure)
- [Data Inputs](#data-inputs)
- [Methodology](#methodology)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Dashboard Outputs](#dashboard-outputs)
- [Sample Workflow](#sample-workflow)
- [Testing](#testing)
- [Design Decisions](#design-decisions)
- [Limitations](#limitations)
- [Roadmap / Future Enhancements](#roadmap--future-enhancements)
- [Interview Framing](#interview-framing)
- [Resume Bullets](#resume-bullets)
- [License](#license)

---

## Overview

Asset-based lending is a form of secured commercial financing where a borrower's revolving credit availability is tied directly to the value of pledged collateral — typically accounts receivable (AR) and inventory — rather than to enterprise cash flow. Lenders continuously recalculate a **borrowing base**: the eligible collateral value, net of ineligibles, multiplied by contractual advance rates, less reserves, to determine how much the borrower can draw.

This project replicates that monitoring process end-to-end in Python, then renders the results into a polished, formula-verified Excel workbook that mirrors what a bank or non-bank ABL lender's portfolio/credit team would review during a borrowing base certificate (BBC) audit or ongoing field exam.

## Why This Project

ABL credit and portfolio risk roles require fluency in:
- Reading and interpreting a borrowing base certificate
- Understanding AR/inventory eligibility rules and why ineligibles are excluded
- Applying advance rates and reserves to derive net availability
- Monitoring customer concentration and cross-aging risk
- Flagging covenant and availability issues before they become defaults

This project was built to demonstrate that workflow concretely — with real calculations, real Excel formulas (not just static values), and outputs structured the way a credit file or field exam report would be structured.

## Key Concepts Modeled

| Concept | Description |
|---|---|
| **AR Aging** | Buckets invoices into Current, 1–30, 31–60, 61–90, and 90+ days past due |
| **AR Eligibility** | Excludes invoices that fail standard ABL eligibility criteria (see below) |
| **Cross-Aging** | Excludes an entire customer's AR balance if a defined percentage of it is past due, even if individual invoices are current |
| **Inventory Eligibility** | Segments and screens inventory by category and excludes ineligible types |
| **Advance Rates** | Contractual percentages applied to eligible AR and inventory to compute loan value |
| **Reserves** | Dollar or percentage-based holdbacks (e.g., dilution reserve, rent reserve, concentration reserve) that reduce the borrowing base |
| **Borrowing Base (BBC)** | Eligible collateral × advance rate − reserves = maximum availability |
| **Excess Availability** | Borrowing base less outstanding loan balance; a key liquidity/covenant metric |
| **Customer Concentration** | Measures dependency risk when a small number of customers make up a large share of AR |

## What It Shows

- AR aging across current, 1–30, 31–60, 61–90, and 90+ buckets
- AR ineligibility for 90+ day invoices, affiliates, unsupported foreign receivables, disputes, offsets, and cross-aging failures
- Inventory segmentation by raw materials, work-in-process, finished goods, obsolete, slow-moving, consigned, and offsite inventory
- Eligible collateral value, advance rates, reserves, borrowing base, utilization, and excess availability
- Customer concentration exposure, top-customer dependency, and concentration reserves
- Credit monitoring alerts for low availability, borrowing base deficiencies, past-due AR, concentration breaches, and deteriorating trends
- Exception report with recommended credit actions

## Project Structure

```text
data/
  ar_receivables.csv     # Invoice-level AR detail (borrower, customer, invoice date, due date, amount, flags)
  inventory.csv           # SKU/category-level inventory detail (borrower, category, cost, NRV, age, location)
  assumptions.csv         # Advance rates, reserve %, concentration limits, aging thresholds, eligibility rules

src/risk_dashboard/
  abl.py                  # Collateral analytics and borrowing-base calculations
  excel_export.py         # Excel dashboard and borrowing base certificate export
  cli.py                  # Command-line entry point

tests/
  test_abl.py              # Unit tests for eligibility logic, aging, and BBC math
```

## Data Inputs

**`ar_receivables.csv`** — one row per invoice, including borrower ID, customer ID/name, invoice date, due date, invoice amount, and flags for affiliate status, dispute status, foreign domicile, and credit insurance/support, where applicable.

**`inventory.csv`** — one row per SKU or inventory lot, including borrower ID, category (raw materials, WIP, finished goods, etc.), cost basis, net realizable value, age/days-on-hand, and location (owned facility, consigned, or offsite/third-party warehouse).

**`assumptions.csv`** — the rules engine: AR and inventory advance rates, reserve percentages, cross-aging thresholds, customer concentration limits, and the specific eligibility exclusion rules applied by the model. Editing this file changes the calculation without touching code.

## Methodology

1. **Ingest** raw AR and inventory data from CSV.
2. **Age** each AR invoice into standard buckets relative to the reporting date.
3. **Screen for ineligibility** — an invoice or inventory item is excluded from the borrowing base if it fails any rule (e.g., 90+ days past due, affiliate receivable, disputed, offset by a payable, unsupported foreign receivable, or the customer breaches the cross-aging threshold).
4. **Segment inventory** by category and apply category-specific eligibility (obsolete, slow-moving, consigned, and offsite inventory are typically excluded or capped).
5. **Apply advance rates** to eligible AR and eligible inventory separately, per the assumptions file.
6. **Deduct reserves** (e.g., concentration reserve, dilution reserve) from gross availability.
7. **Compute the borrowing base**, compare it to the outstanding balance, and derive **excess (or deficient) availability**.
8. **Evaluate concentration** by ranking customers by eligible AR exposure and flagging breaches of concentration limits.
9. **Generate exceptions** — a rules-based scan across all of the above that produces a prioritized list of credit monitoring alerts and recommended actions.
10. **Export to Excel**, writing both computed values and live formulas so the workbook can be audited or updated by an analyst without needing to rerun Python.

## Quick Start

```bash
python3 -m pip install -e ".[dev]"
python3 -m risk_dashboard.cli --output outputs/abl_dashboard/borrowing_base_dashboard.xlsx
pytest
```

You can also run the installed console command:

```bash
abl-dashboard --output outputs/abl_dashboard/borrowing_base_dashboard.xlsx
```

### Requirements

- Python 3.9+
- Dependencies declared in `pyproject.toml` (installed automatically via `pip install -e ".[dev]"`)

## Configuration

Most credit-policy assumptions live in `data/assumptions.csv` rather than hardcoded in Python, including:

- AR advance rate (e.g., 85% of eligible AR)
- Inventory advance rate, often the lesser of a cost-based or NRV-based percentage
- Cross-aging threshold (e.g., exclude a customer's entire AR balance if 50%+ is over 90 days)
- Customer concentration limit (e.g., flag any customer exceeding 15–20% of eligible AR)
- Reserve percentages (dilution, rent/landlord, concentration)
- Aging bucket cutoffs

This keeps the credit policy transparent and easy to adjust for different borrowers or lending programs without modifying the calculation logic.

## Dashboard Outputs

The generated workbook includes:

| Tab | Contents |
|---|---|
| `Dashboard` | KPI summary, latest exceptions, availability trend, collateral mix, and customer concentration charts |
| `BBC Output` | Borrowing base calculation by borrower and reporting period |
| `AR Aging` | Receivables by aging bucket |
| `AR Eligibility` | Invoice-level eligibility testing |
| `Inventory Eligibility` | SKU/category-level inventory eligibility |
| `Customer Concentration` | Top customer exposure and concentration reserves |
| `Trend Detail` | Availability, utilization, past-due AR, and inventory ineligibility trends over time |
| `Exception Report` | Collateral issues and recommended credit actions |
| `Checks` | Formula-backed model tie-outs to validate the workbook against the Python calculations |

The `Checks` tab is intentionally included so the workbook is self-auditing: an analyst can confirm the Excel formulas reconcile to the underlying Python outputs before relying on the file.

## Sample Workflow

A typical use of this tool mirrors a real ABL field exam or monthly BBC review cycle:

1. Load the borrower's latest AR and inventory data for the reporting period.
2. Run the CLI to regenerate the workbook.
3. Open the `Dashboard` tab to review headline availability, utilization, and any new exceptions.
4. Drill into `AR Eligibility` or `Inventory Eligibility` to understand why specific balances were excluded.
5. Review `Customer Concentration` for dependency risk on top obligors.
6. Check `Trend Detail` for deteriorating availability or rising past-due AR across periods.
7. Use the `Exception Report` to prioritize follow-up questions or covenant conversations with the borrower.

## Testing

`tests/test_abl.py` covers the core calculation logic, including:

- AR aging bucket assignment
- Eligibility rule exclusions (90+ day, affiliate, disputed, foreign, cross-aging)
- Inventory category eligibility and caps
- Advance rate application
- Reserve deduction and borrowing base arithmetic
- Excess availability and concentration calculations

Run the full suite with:

```bash
pytest
```

## Design Decisions

- **CSV-driven assumptions** so credit policy changes don't require code changes.
- **Formula-backed Excel output** (not just pasted values) so the workbook can be reviewed and trusted the way a real credit file would be.
- **A dedicated `Checks` tab** to reconcile Excel outputs back to the Python model, reducing the risk of silent calculation drift.
- **Separation of concerns** between analytics (`abl.py`) and presentation (`excel_export.py`) so the calculation engine can be reused or tested independently of the Excel export.

## Limitations

- Eligibility rules are simplified relative to a real loan agreement's borrowing base definition, which can run many pages and vary by lender and industry.
- The model does not currently incorporate landlord waivers, bank consents, or letter-of-credit sublimits, which affect real-world availability.
- Inventory NRV and appraisal-based advance rates are approximated rather than sourced from third-party appraisals.

## Roadmap / Future Enhancements

- Multi-currency AR support for foreign receivables
- Configurable, per-borrower eligibility rule sets (rather than one global rule set)
- Automated period-over-period covenant testing (e.g., fixed charge coverage triggers tied to availability)
- Web-based dashboard front end in addition to the Excel export

## Interview Framing

> I built a borrowing base dashboard that analyzes AR and inventory collateral to estimate loan availability under an asset-based lending structure. The model applies eligibility rules, advance rates, reserves, customer concentration limits, and aging analysis to produce a borrowing base certificate-style output. I designed it to mirror how an ABL team would monitor collateral value, credit exposure, and risk controls across a borrower portfolio.

## Resume Bullets

- Built a Python and Excel-based borrowing base dashboard to estimate asset-based lending availability using AR aging, inventory eligibility, advance rates, and collateral reserves.
- Analyzed customer concentration, past-due receivables, inventory ineligibility, and excess availability to identify collateral risk and borrowing base deficiencies.
- Created credit monitoring outputs similar to an ABL borrowing base certificate, supporting collateral-based loan structuring and portfolio risk review.

## License

Add your preferred license here (e.g., MIT).
