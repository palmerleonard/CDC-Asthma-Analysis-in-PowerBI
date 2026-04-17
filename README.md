# 🫁 Pediatric Asthma ETL Pipeline
### CDC CSV → Claude-Generated XLSX → Power BI Dashboard

[![Data Source](https://img.shields.io/badge/Source-CDC%20NCHS-blue)](https://www.cdc.gov/nchs/hus/topics/asthma.htm)
[![Tool](https://img.shields.io/badge/Transform-Claude%20AI-blueviolet)](https://claude.ai)
[![Output](https://img.shields.io/badge/Dashboard-Power%20BI-F2C811?logo=powerbi&logoColor=black)](https://powerbi.microsoft.com)
[![Status](https://img.shields.io/badge/Status-Complete-brightgreen)]()

---

## 📋 Table of Contents

- [Overview](#overview)
- [Pipeline Architecture](#pipeline-architecture)
- [Stage 1 — Source Data (CDC CSV)](#stage-1--source-data-cdc-csv)
- [Stage 2 — AI-Assisted Transform (Claude → XLSX)](#stage-2--ai-assisted-transform-claude--xlsx)
- [Stage 3 — Power BI Load & Visualization](#stage-3--power-bi-load--visualization)
- [DAX Measures Reference](#dax-measures-reference)
- [Data Dictionary](#data-dictionary)
- [Key Findings](#key-findings)
- [File Structure](#file-structure)

---

## Overview

This repository documents the end-to-end ETL (Extract, Transform, Load) pipeline used to analyze **CDC National Health Interview Survey (NHIS)** data on asthma prevalence among U.S. children under 18, covering selected years from **1997 to 2022**.

The pipeline was built using **Claude AI** as the transformation engine — replacing a traditional Python/dbt/SQL stack with a conversational, AI-assisted workflow that produced a clean, Power BI-ready Excel workbook and an interactive dashboard.

| Stage | Tool | Input | Output |
|-------|------|-------|--------|
| Extract | CDC / NCHS | Raw `.csv` download | Structured survey data |
| Transform | Claude AI | CDC `.csv` | Clean `.xlsx` workbook |
| Load | Power BI Desktop | `.xlsx` workbook | Interactive dashboard |

---

## Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ETL PIPELINE OVERVIEW                        │
└─────────────────────────────────────────────────────────────────────┘

  ┌──────────────┐     Extract      ┌──────────────┐
  │  CDC / NCHS  │ ───────────────► │  Raw CSV     │
  │  Website     │                  │  1,300 rows  │
  └──────────────┘                  └──────┬───────┘
                                           │
                                           │  Upload to Claude AI
                                           ▼
                                    ┌──────────────┐
                                    │  Claude AI   │
                                    │  Transform   │
                                    │  + Validate  │
                                    └──────┬───────┘
                                           │
                              ┌────────────┼────────────┐
                              │            │            │
                              ▼            ▼            ▼
                         Raw Data     Pivot Tables   KPI Cards
                         (Sheet 1)    (Sheets 2–3)  (Sheet 4)
                              │            │            │
                              └────────────┼────────────┘
                                           │
                                    ┌──────▼───────┐
                                    │  Clean XLSX  │
                                    │  Workbook    │
                                    │  1,048 rows  │
                                    └──────┬───────┘
                                           │
                                           │  Get Data → Excel
                                           ▼
                                    ┌──────────────┐
                                    │  Power Query │
                                    │  Type Cast   │
                                    │  + Filter    │
                                    └──────┬───────┘
                                           │
                                           ▼
                                    ┌──────────────┐
                                    │  Power BI    │
                                    │  Data Model  │
                                    │  + DAX       │
                                    └──────┬───────┘
                                           │
                                           ▼
                                    ┌──────────────┐
                                    │  Dashboard   │
                                    │  6 Visuals   │
                                    │  3 Slicers   │
                                    └──────────────┘
```

---

## Stage 1 — Source Data (CDC CSV)

### Source
**National Center for Health Statistics (NCHS)**
Health, United States — Table: *Asthma among children younger than 18 years, by selected characteristics: United States, selected years 1997–2022*

**Download URL:**
```
https://www.cdc.gov/nchs/hus/topics/asthma.htm
```

### Raw File Profile

| Property | Value |
|----------|-------|
| File name | `Asthma_in_children_younger_than_age_18__by_selected_characteristics__United_States.csv` |
| Total rows | 1,300 |
| Rows with valid estimates | 1,048 |
| Rows suppressed / missing | 252 (flagged `- - -`, years 1997–2002) |
| Year range | 1997–2022 |
| Estimate type | Percent of children, crude |

### Raw Schema

```
HUS_YEAR        — Publication year of Health, United States report
HUS_SHORT_NAME  — Dataset short code (ASTHCH)
INDICATOR       — Full indicator description
PANEL_NUM       — Panel index (1 = Current asthma, 2 = Asthma attack)
PANEL           — Panel label
UNIT_NUM        — Unit index
UNIT            — "Percent of children, crude"
STUB_NAME_NUM   — Demographic dimension index
STUB_NAME_ORDER — Sort order within dimension
STUB_NAME       — Demographic dimension (Total, Sex, Age group, Race, etc.)
STUB_LABEL_NUM  — Sub-group index
STUB_LABEL_ORDER— Sort order within sub-group
STUB_LABEL      — Sub-group label (Male, Female, 0-4 years, White only, etc.)
YEAR_NUM        — Year index (1–26)
YEAR            — Survey year (1997–2022)
AGE_NUM         — Age code
AGE             — Age label
ESTIMATE        — Percent estimate (nullable — suppressed for unstable estimates)
SE              — Standard error (nullable)
FLAG            — Suppression flag ("- - -" = not available)
FOOTNOTE_ID_LIST— Footnote codes
FOOTNOTE_LIST   — Footnote descriptions
```

### Data Quality Notes

- Estimates **before 2003 are fully suppressed** for most sub-groups due to redesigned survey methodology in 2003
- `FLAG = "- - -"` rows were **excluded** from all analysis (252 rows dropped)
- No duplicate records detected
- Standard errors are provided for confidence interval construction

---

## Stage 2 — AI-Assisted Transform (Claude → XLSX)

### Why Claude Instead of a Traditional ETL Script?

This pipeline replaces a conventional Python/pandas transform script with **Claude AI (claude-sonnet-4-6)** running interactively. Claude was given the raw CDC CSV and asked to:

1. Parse and profile the data structure
2. Filter to valid estimates only (`ESTIMATE` not null)
3. Reshape into analysis-ready pivot tables
4. Compute derived metrics (disparities, differences)
5. Produce a fully formatted, Power BI-ready Excel workbook with zero formula errors

The transformation logic that Claude applied is equivalent to the following Python pseudocode:

```python
import pandas as pd

# ── EXTRACT ──────────────────────────────────────────────────────────────────
df = pd.read_csv("cdc_asthma_children.csv")

# ── TRANSFORM ────────────────────────────────────────────────────────────────

# 1. Drop suppressed rows
df = df[df["ESTIMATE"].notna() & (df["FLAG"] != "- - -")]
# 1,300 → 1,048 rows

# 2. Cast types
df["YEAR"]        = df["YEAR"].astype(int)
df["ESTIMATE"]    = df["ESTIMATE"].astype(float)
df["SE"]          = pd.to_numeric(df["SE"], errors="coerce")

# 3. Select and rename columns for Power BI
df_clean = df[[
    "PANEL",        → "Panel"
    "STUB_NAME",    → "Stub_Name"
    "STUB_LABEL",   → "Stub_Label"
    "YEAR",         → "Year"
    "ESTIMATE",     → "Estimate_Pct"
    "SE",           → "SE"
]].copy()
df_clean["Source"] = "CDC NHIS"

# 4. Build national trend pivot
trend = df_clean[
    (df_clean["Stub_Name"] == "Total")
].pivot_table(index="Year", columns="Panel", values="Estimate_Pct")
trend["Difference_pp"] = trend["Current asthma"] - trend["Asthma attack in past 12 months"]

# 5. Build 2022 demographic snapshot
demo_2022 = df_clean[
    (df_clean["Year"] == 2022) &
    (df_clean["Panel"] == "Current asthma") &
    (df_clean["Stub_Name"].isin(["Sex","Age group","Race","Poverty level"]))
]

# ── LOAD ─────────────────────────────────────────────────────────────────────
with pd.ExcelWriter("Pediatric_Asthma_PowerBI.xlsx", engine="openpyxl") as writer:
    df_clean.to_excel(writer,  sheet_name="RawData_PowerBI",       index=False)
    trend.to_excel(writer,     sheet_name="National_Trend",        index=True)
    demo_2022.to_excel(writer, sheet_name="Demographic_Breakdowns",index=False)
```

### Output Workbook Structure

The Claude-generated `.xlsx` file contains **5 sheets**:

#### Sheet 1 — `RawData_PowerBI` *(Primary Import Table)*

This is the **single source of truth** for Power BI. All 1,048 valid rows, clean column names, correct data types.

| Column | Type | Description |
|--------|------|-------------|
| `Panel` | Text | `"Current asthma"` or `"Asthma attack in past 12 months"` |
| `Stub_Name` | Text | Demographic dimension (Total, Sex, Age group, Race, Poverty level) |
| `Stub_Label` | Text | Sub-group within dimension (Male, Female, 0-4 years, etc.) |
| `Year` | Whole Number | Survey year (2003–2022) |
| `Estimate_Pct` | Decimal | Asthma prevalence as % of children |
| `SE` | Decimal | Standard error of estimate |
| `Source` | Text | `"CDC NHIS"` (static audit column) |

#### Sheet 2 — `National_Trend`

Year-by-year pivot of national Current Asthma vs. Attack Rate, with a live `=B-C` difference column. Includes an embedded Excel line chart.

#### Sheet 3 — `Demographic_Breakdowns`

2022 snapshot across all 4 demographic dimensions with a disparity column (`Estimate − National Total`) and an embedded bar chart.

#### Sheet 4 — `KPI_Summary`

8 pre-computed headline KPI cards ready to replicate as Power BI Card visuals.

#### Sheet 5 — `PowerBI_Setup_Guide`

Step-by-step instructions for importing, writing DAX measures, building visuals, and applying the dashboard theme.

### Validation

The workbook passed formula validation with **0 errors** across 20 formulas:

```json
{
  "status": "success",
  "total_errors": 0,
  "total_formulas": 20,
  "error_summary": {}
}
```

---

## Stage 3 — Power BI Load & Visualization

### Step 1 — Connect to the XLSX

```
Power BI Desktop
  → Home
  → Get Data
  → Excel Workbook
  → [Select Pediatric_Asthma_PowerBI_Light.xlsx]
  → Navigator: ✅ RawData_PowerBI
  → Transform Data
```

### Step 2 — Power Query Type Casting

In the Power Query Editor, enforce the following column types before closing:

| Column | Power Query Type |
|--------|-----------------|
| `Panel` | Text |
| `Stub_Name` | Text |
| `Stub_Label` | Text |
| `Year` | Whole Number |
| `Estimate_Pct` | Decimal Number |
| `SE` | Decimal Number |
| `Source` | Text |

```
→ Close & Apply
```

### Step 3 — Create DAX Measures

In the Data Model view, create a **Measures table** and add the following:

```dax
-- Average prevalence across current filter context
Avg Asthma % = AVERAGE(RawData_PowerBI[Estimate_Pct])

-- All-time maximum rate
Max Rate = MAX(RawData_PowerBI[Estimate_Pct])

-- Current asthma panel only
Current Only =
    CALCULATE(
        [Avg Asthma %],
        RawData_PowerBI[Panel] = "Current asthma"
    )

-- Asthma attack panel only
Attack Only =
    CALCULATE(
        [Avg Asthma %],
        RawData_PowerBI[Panel] = "Asthma attack in past 12 months"
    )

-- Gap between current and attack rates
Panel Gap = [Current Only] - [Attack Only]

-- Year-over-year change (requires Date table or Year column slicer)
YoY Change =
    VAR CurrentYear = MAX(RawData_PowerBI[Year])
    VAR PriorYear   = CurrentYear - 1
    VAR CurrentVal  = CALCULATE([Avg Asthma %], RawData_PowerBI[Year] = CurrentYear)
    VAR PriorVal    = CALCULATE([Avg Asthma %], RawData_PowerBI[Year] = PriorYear)
    RETURN CurrentVal - PriorVal

-- Disparity vs. national total (for demographic visuals)
Disparity vs Total =
    VAR SubGroupRate = [Avg Asthma %]
    VAR NationalRate =
        CALCULATE(
            [Avg Asthma %],
            RawData_PowerBI[Stub_Name] = "Total"
        )
    RETURN SubGroupRate - NationalRate
```

### Step 4 — Build the 6 Dashboard Visuals

#### Visual 1 — National Trend (Area Chart)
```
Visual type : Area Chart
X-axis      : Year
Y-axis      : Estimate_Pct
Legend      : Panel
Filter      : Stub_Name = "Total"
Purpose     : Shows both Current Asthma and Attack Rate trends 2003–2022
```

#### Visual 2 — Age Group Lines (Line Chart)
```
Visual type : Line Chart
X-axis      : Year
Y-axis      : Estimate_Pct
Legend      : Stub_Label
Filter      : Stub_Name = "Age group"
Purpose     : Compares 0-4, 5-9, and 10-17 year age bands over time
```

#### Visual 3 — Sex Disparity (Stacked Area Chart)
```
Visual type : Area Chart
X-axis      : Year
Y-axis      : Estimate_Pct
Legend      : Stub_Label
Filter      : Stub_Name = "Sex"
Purpose     : Highlights the persistent male-female prevalence gap
```

#### Visual 4 — Race Disparity (Clustered Bar Chart)
```
Visual type : Clustered Bar Chart
X-axis      : Estimate_Pct
Y-axis      : Stub_Label
Legend      : Panel
Filter      : Stub_Name = "Race"
Year slicer : Connected (snapshot year)
Purpose     : Side-by-side current vs. attack rate by racial group
```

#### Visual 5 — Poverty Gradient (Horizontal Bar Chart)
```
Visual type : Bar Chart (Horizontal)
X-axis      : Estimate_Pct
Y-axis      : Stub_Label
Filter      : Stub_Name = "Poverty level"
Sort        : By Estimate_Pct descending
Year slicer : Connected (snapshot year)
Purpose     : Illustrates the socioeconomic gradient in asthma burden
```

#### Visual 6 — KPI Cards
```
Card 1 : [Current Only]     — filtered to Year = 2022
Card 2 : [Max Rate]         — all-time peak
Card 3 : [Attack Only]      — filtered to Year = 2022
Card 4 : [Disparity vs Total] — filtered to Black only, Year = 2022
Card 5 : [YoY Change]       — most recent available year
```

### Step 5 — Add Slicers

| Slicer | Field | Type | Effect |
|--------|-------|------|--------|
| Asthma Type | `Panel` | Tile / Dropdown | Filters all 6 visuals simultaneously |
| Snapshot Year | `Year` | Slider or Dropdown | Drives bar chart snapshots + KPI cards |
| Demographic Dimension | `Stub_Name` | Dropdown | Drill into one demographic at a time |

### Step 6 — Apply Theme

Set canvas and visual colors to match the dashboard palette:

| Element | Hex | Usage |
|---------|-----|-------|
| Primary | `#2E75B6` | Main line/bar color, headers |
| Alert | `#C00000` | Attack rate, high-prevalence values |
| Amber | `#B8660A` | Elevated values (8–12%) |
| Positive | `#375623` | Declining trend, low values |
| Purple | `#7030A0` | Racial disparity highlights |
| Canvas BG | `#FFFFFF` | Standard white background |
| Alt rows | `#F5F7FA` | Alternating table rows |

```
Power BI Desktop
  → View
  → Themes
  → Customize current theme
  → Paste hex values above into the color slots
```

---

## DAX Measures Reference

| Measure | Description | Use In |
|---------|-------------|--------|
| `Avg Asthma %` | Average prevalence in filter context | All visuals |
| `Max Rate` | All-time peak rate | KPI Card |
| `Current Only` | Current asthma panel average | KPI Card, trend |
| `Attack Only` | Attack panel average | KPI Card, trend |
| `Panel Gap` | Difference between panels | KPI Card |
| `YoY Change` | Year-over-year delta | KPI Card |
| `Disparity vs Total` | Sub-group minus national average | Demographic bars |

---

## Data Dictionary

### Demographic Dimensions (`Stub_Name`)

| Value | Sub-groups |
|-------|-----------|
| `Total` | Younger than 18 years |
| `Sex` | Male, Female |
| `Age group` | 0-4 years, 5-9 years, 10-17 years, 5-17 years |
| `Race` | White only, Black only, American Indian and Alaska Native only, Asian only, Native Hawaiian or Other Pacific Islander only, Two or more races |
| `Race and Hispanic origin` | All races Hispanic, All races non-Hispanic, Black only non-Hispanic, White only non-Hispanic |
| `Health insurance status` | Insured, Medicaid, Private, Uninsured |
| `Poverty level` | Below 100% FPL, 100%-199% FPL, 200%-399% FPL, 400% FPL or more |

### Panels (`Panel`)

| Value | Definition |
|-------|-----------|
| `Current asthma` | Child currently has asthma (diagnosis + still has it) |
| `Asthma attack in past 12 months` | Had at least one asthma attack/episode in past 12 months |

---

## Key Findings

The dashboard surfaces the following evidence-based insights from the CDC data:

- 📉 **National rate declined** from a peak of **9.6%** (2009) to **6.2%** (2022) — a 3.4 percentage point reduction
- 👦 **Males** consistently show higher rates than females across all years (~2–3pp gap)
- 🎒 **School-age children (5-17)** bear a disproportionate burden vs. toddlers (0-4 years)
- ⚕️ **Black children** peaked at **14.3%** in 2009 vs. 8.4% for White children — a persistent ~1.7x disparity
- 🏠 **Poverty gradient is clear**: children below 100% FPL show rates 56% higher than those at 400%+ FPL (7.5% vs. 4.8% in 2022)
- 📋 **Attack rate (~3.2%)** runs consistently at roughly half the current asthma rate, indicating significant uncontrolled disease burden

---

## File Structure

```
📁 pediatric-asthma-etl/
│
├── 📄 README.md                              ← This file
│
├── 📁 data/
│   ├── raw/
│   │   └── Asthma_in_children_younger_than_18_CDC.csv   ← Source file
│   └── processed/
│       └── Pediatric_Asthma_PowerBI_Light.xlsx          ← Claude-generated workbook
│
├── 📁 dashboard/
│   └── Pediatric_Asthma_Dashboard.pbix                  ← Power BI file
│
└── 📁 docs/
    └── Pediatric_Asthma_Dashboard.pdf                   ← Dashboard preview
```

---

## Data Source & Citation

> National Center for Health Statistics. Health, United States, 2022.
> *Table: Asthma among children younger than age 18, by selected characteristics: United States, selected years 1997–2022.*
> Hyattsville, MD: National Center for Health Statistics; 2023.
> https://www.cdc.gov/nchs/hus/

---

*ETL pipeline designed and executed using [Claude AI](https://claude.ai) — Anthropic's AI assistant.*
*Dashboard built in Microsoft Power BI Desktop.*
