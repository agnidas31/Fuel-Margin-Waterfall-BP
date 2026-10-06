Field-Tested Innovation · US Fuels Pricing · bp

# Creative Solutions: The Fuel Margin Waterfall

How a single cents-per-gallon margin number was broken into the drivers behind it, so pricing and supply teams could see not just *what* the margin was, but *why*.

Agnivesh Das · Senior Data Analytics Engineer · bp

> **Note:** the structure and margin components reflect how the report worked. All values, volumes and examples are illustrative, not real production figures. Items marked *[Confirm]* are my reading of the dashboard labels and need your check.

---

## My Role

As Senior Data Analytics Engineer, I owned the data behind the report end to end:

- **Data preparation.** Brought together pricing, supply cost, invoice and volume data from multiple source systems.
- **Cleaning and transformation.** Aligned everything to a common grain (site × day × fuel grade), volume-weighted every component, and built the logic that splits total margin into its drivers.
- **Reporting.** Built the Tableau waterfall views: monthly, year-to-date, and year-on-year, with filters from channel of trade down to individual site and grade.

---

## The Problem

### A margin number with no story behind it

Fuel margin is reported in **cents per gallon (CPG)**, and small moves matter: one cent across hundreds of millions of gallons is millions of dollars.

But a single margin figure hides everything that drives it. When margin fell, the business couldn't quickly tell whether it was because:

- the market moved,
- the brand premium changed,
- supply terms got worse,
- volume came in differently from what was invoiced,
- or environmental and logistics costs shifted.

Each of those has a different owner and a different fix. Without the breakdown, every margin conversation started with "let's go and find out why."

---

## The Constraint Nobody Was Naming

### Every driver lived in a different system, at a different grain

Breaking margin into drivers wasn't a reporting problem. It was a data problem:

**1. Different sources, different grains.** Market reference prices are published daily by terminal. Invoices are per delivery. Volumes are per site per day. Terms are per contract. None of them lined up naturally.

**2. Averages lie.** A simple average of CPG across sites gives a small site the same weight as a large one. Every component had to be **volume-weighted**, or the parts wouldn't add up to the whole.

**3. The pieces have to reconcile.** A waterfall only works if the bars sum exactly to the total. Any gap between the components and the reported margin would destroy trust in the whole report.

---

## The Fix

### One dataset, one grain, every driver as its own bar

The report decomposes margin into three layers:

**Standard Margin** (what the retail network earns on the fuel itself)

| Component | What it captures *[Confirm]* |
| --------- | ---------------------------- |
| **Market (OPIS low) margin** | Retail price vs. the lowest published rack price in the market: the "baseline" margin anyone could earn |
| **Brand value** | The extra price the bp brand commands at the pump |
| **bp premium** | The extra cost of buying branded supply above the market low (a negative bar) |
| **Terms impact** | The cost of payment and contract terms |
| **Invoice volume variance** | Differences between invoiced and actual delivered volume |
| **Others** | Smaller adjustments |
| **= Standard margin** | |

**Integrated Margin** (value added by bp's own supply chain)
Adds variable logistics and supply & trading contributions on top of standard margin. *[Confirm]*

**Environment** (regulatory and environmental effects)
Lag effects, renewable fuel credits (RINs), and line-space and programme costs. *[Confirm]*

### How it works

1. **Ingest:** Market reference prices, retail prices, supply invoices, contract terms and delivered volumes.
2. **Align:** Map everything to site × day × grade, with site attributes (region, channel of trade, brand) attached.
3. **Weight:** Convert every component into volume-weighted CPG.
4. **Decompose:** Calculate each driver so the components sum exactly to standard, integrated and total margin.
5. **Reconcile:** Check component totals against the finance margin figure every refresh.
6. **Report:** Waterfall views monthly, YTD and year-on-year, filterable from region down to site and grade.

---

## Worked Example

### What a monthly waterfall actually shows

> **Illustrative values only.**

Network view for one month, 500 million gallons:

| Component | CPG |
| --------- | --- |
| Market (OPIS low) margin | +7.0 |
| Brand value | +10.0 |
| bp premium | −8.5 |
| Terms impact | −1.2 |
| Invoice volume variance | +1.3 |
| Others | −0.1 |
| **Standard margin** | **8.5** |

```
8.5 cpg × 500,000,000 gallons = $42.5 million standard margin
```

Now suppose next month standard margin falls to 7.5 CPG. A single number says "margin is down a cent: about $5 million." The waterfall says *which bar moved*. If brand value held but the bp premium widened, the conversation goes to supply, not to the pricing team.

---

## Why This Beat The Old Approach

|                         | Before                                   | With the margin waterfall                          |
| ----------------------- | ---------------------------------------- | -------------------------------------------------- |
| What you see            | One margin number                        | Every driver as its own bar                        |
| Explaining a change     | Manual investigation across systems      | Read straight off the chart                        |
| Weighting               | Inconsistent averages                    | Volume-weighted CPG throughout                     |
| Trust                   | Components didn't always tie to finance  | Components reconcile to the total                  |
| Comparisons             | Ad hoc                                   | Monthly, YTD and year-on-year built in             |
| Drill-down              | Network level                            | Region, channel, brand, site and grade             |

---

## Impact

**One view, three layers:** standard, integrated and environmental margin in one place.

**Every cent explained:** each change in margin traced to a specific driver and owner.

**From network to site:** the same breakdown at every level, for every fuel grade.

The real shift was changing the question from "what was our margin?" to "what moved our margin, and who owns it?"

---
---

# Technical Deep Dive: How the Waterfall Is Built

## 1. Source data

| Source | Grain | Used for |
| ------ | ----- | -------- |
| Market reference prices (e.g. OPIS rack) | Terminal × day × grade | Baseline cost |
| Retail pump prices | Site × day × grade | Revenue per gallon |
| Supply invoices | Delivery | Actual cost, invoiced volume |
| Contract terms | Contract / site | Terms impact |
| Delivered / sold volumes | Site × day × grade | Weighting, volume variance |
| Site master | Site | Region, channel of trade, brand, terminal mapping |

## 2. Align to one grain

Every input is mapped to **site × day × grade**:

- Each site is mapped to its supplying terminal, so it gets the right market reference price.
- Invoices are allocated to delivery dates and grades.
- Missing prices are carried forward from the last valid day, and flagged.

## 3. Volume-weight every component

Each component is stored as **dollars**, not CPG, and converted to CPG only at report time:

```sql
SELECT region,
       month,
       SUM(market_margin_usd)   / SUM(gallons) * 100 AS market_margin_cpg,
       SUM(brand_value_usd)     / SUM(gallons) * 100 AS brand_value_cpg,
       SUM(bp_premium_usd)      / SUM(gallons) * 100 AS bp_premium_cpg,
       SUM(terms_usd)           / SUM(gallons) * 100 AS terms_cpg,
       SUM(volume_variance_usd) / SUM(gallons) * 100 AS volume_variance_cpg,
       SUM(other_usd)           / SUM(gallons) * 100 AS other_cpg,
       SUM(standard_margin_usd) / SUM(gallons) * 100 AS standard_margin_cpg
FROM   margin_components_daily
GROUP  BY region, month;
```

Storing dollars and dividing by total gallons at the last step means the components **always sum to the total** at any level of aggregation. Averaging CPG directly would break that.

## 4. Decompose (illustrative logic)

Per site × day × grade:

```
market_margin   = (retail_price − market_low_price)        × gallons
brand_value     = (brand_reference − market_reference)     × gallons
bp_premium      = −(branded_cost − market_low_price)       × gallons
terms_impact    = −(terms cost per gallon)                 × gallons
volume_variance = (delivered_gallons − invoiced_gallons)   × unit_cost
others          = standard_margin − SUM(all components above)
```

The "others" bar absorbs rounding and small items. It's monitored: if it grows, something upstream has broken.

## 5. Reconcile

Each refresh:

- Total standard margin is compared with the finance figure for the same period.
- The "others" bar is checked against a tolerance.
- Volumes are reconciled to reported sales gallons.

## 6. Report

Tableau workbook with five views:

1. All margin components (monthly)
2. All margin (YTD)
3. Standard margin (YTD, this year vs last year)
4. Environment (YTD)
5. Integrated margin (YTD)

Filters: year, month, channel of trade, branded price type, region, footprint, state, terminal, product type and grade.

---

Created by Agnivesh Das · bp US Fuels · Illustrative values throughout. CPG = cents per gallon. OPIS = Oil Price Information Service (market price benchmark). RINs = Renewable Identification Numbers.
