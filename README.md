# 🛒 Market Basket & Spatial Anchor SKU Analysis

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/12SKQgDxLjCuZoMwfDZCbH2A-jbWn4Ir4?usp=sharing)
![Python](https://img.shields.io/badge/Python-3.10+-blue?logo=python&logoColor=white)
![pandas](https://img.shields.io/badge/pandas-2.0+-150458?logo=pandas&logoColor=white)
![GeoPandas](https://img.shields.io/badge/GeoPandas-0.14+-139C5A?logo=python&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)

A data science pipeline that identifies product affinity patterns at the micro-geographic level and translates them into concrete field sales actions. Applied FP-Growth association rule mining combined with a custom SKU Viability Scoring engine to uncover high-gravity **Anchor SKUs** — products whose presence in a basket reliably predicts the purchase of another — and maps geographic brand presence to guide targeted distribution.

> **Note:** All data in this repository is synthetically generated to replicate real-world FMCG transaction and geographic patterns. No proprietary or confidential data is used.

---

## 📋 Table of Contents

- [Business Problem](#-business-problem)
- [From Data Science to Field Action](#-from-data-science-to-field-action)
- [The Four Personas](#-the-four-personas)
- [SKU Viability Scoring](#-sku-viability-scoring)
- [Geographic Micro-Segmentation](#-geographic-micro-segmentation)
- [Pipeline Architecture](#-pipeline-architecture)
- [Outputs](#-outputs)
- [Tech Stack](#-tech-stack)
- [How to Run](#-how-to-run)

---

## 📌 Business Problem

Field sales teams covering hundreds of barangays (the smallest administrative unit in the Philippines) need a clear, prioritized answer to a deceptively simple question: **"Which products should I be pitching together, in this specific neighborhood, right now?"**

Aggregate sales reporting at the city or provincial level answers this poorly. A nationally popular SKU dominates every report, while genuinely valuable local opportunities — a coffee-and-creamer pairing that's exploding in one barangay, or a detergent-and-softener bundle that's quietly building loyalty in another — get buried in the noise.

This pipeline solves that by running statistical product-affinity analysis **independently within every branch-barangay combination**, then translating the raw statistics into a sales-ready action framework that requires no data science background to execute.

---

## 🎯 From Data Science to Field Action

The pipeline's defining feature is that it does not stop at producing statistical output. Raw Lift and Confidence scores mean little to a field representative standing in front of a sari-sari store owner. The **Persona Classification layer** converts every discovered rule into one of four concrete sales directives — answering not just *"what is statistically associated"* but *"what should I do about it."*

```
Raw Transaction Data
        ↓
FP-Growth Statistical Discovery (Support, Confidence, Lift)
        ↓
SKU Viability Scoring (is the Anchor product actually healthy here?)
        ↓
Persona Classification (translate math → sales directive)
        ↓
Field-Ready Excel Output (sorted by branch, sortable by priority)
```

This synthesis layer is what separates an academic exercise from a deployable business tool.

---

## 🎭 The Four Personas

Every discovered association rule is automatically classified into one of four business personas, each carrying a distinct field action:

| Persona | Statistical Trigger | What It Means | Field Action |
|---|---|---|---|
| **A: Perfect Pair** | Lift ≥ 2.0 & Confidence ≥ 0.70 | A near-symmetric, highly reliable pairing | Mandatory bundling — never pitch one without the other |
| **B: Anchor Pull** | Lift ≥ 1.5 & Confidence ≥ 0.50 | A strong anchor product reliably pulling a secondary item | Protect the Anchor SKU's shelf presence to prevent basket collapse |
| **C: Hidden Gem** | Lift ≥ 2.5 & Support ≤ 0.10 | Explosive statistical pull, but still low overall adoption | High-priority sampling and targeted expansion — this is an early signal worth chasing |
| **Standard Association** | All others | Coincidental or expected co-purchase behavior | No special promotional budget required |

This classification means a regional sales manager can filter directly to **Persona C: Hidden Gem** rows across their territory and immediately see where the next expansion opportunity is — without ever looking at a raw lift score.

---

## 📐 SKU Viability Scoring

Lift and Confidence answer *"are these two products statistically linked?"* — but they say nothing about whether the Anchor product itself is healthy in that location. A product that spiked in a single bulk order can produce a misleadingly strong association rule.

The **Viability Score** answers a separate, equally important question: *"How embedded is this SKU in this specific barangay's actual buying habits?"* It is a composite of three weighted pillars:

| Pillar | Weight | What It Measures | Business Meaning |
|---|---|---|---|
| **Stickiness** | 40% | Purchase frequency + recency over the trailing 3 months | Is the product bought consistently, or was it a one-time event? |
| **Reliability** | 35% | Total volume normalized against the market maximum | Is sales volume meaningful relative to top performers? |
| **Adoption** | 25% | Number of distinct stores carrying the product | Is the product spread across many outlets, or concentrated in one? |

By appending Viability Scores to both sides of every association rule, the output reveals a strategic dimension that pure MBA cannot: a **high-viability Anchor paired with a low-viability Consequent** is the single highest-value opportunity in the dataset — a proven, trusted product that can be used as leverage to introduce an underperforming but statistically linked SKU into a store's inventory.

---

## 🗺️ Geographic Micro-Segmentation

Running market basket analysis at a city or provincial scale produces rules dominated by universally popular products — a top-seller appears in the majority of all transactions everywhere, anchoring every rule and drowning out genuinely local insight.

This pipeline instead runs FP-Growth **independently within every branch × barangay combination**. The result is a set of association rules that reflect actual neighborhood-level consumer behavior: one barangay may show a strong coffee-and-creamer affinity, while a few kilometers away the dominant pairing is detergent-and-fabric-softener. These are different local market signals requiring different sales conversations — visible only at this resolution.

A complementary **geospatial heatmap** (built with GeoPandas and Folium) visualizes average SKU Viability across all barangays, surfacing geographic clusters of strong consumer embeddedness versus distribution gaps — directly informing route prioritization and expansion targeting decisions.

---

## 🏗️ Pipeline Architecture

```
Raw Transaction Data (3-Month Rolling Window, SQL Server)
        │
        ▼
┌─────────────────────────────┐
│  Data Validation             │  ← Drop records missing barcode, receipt ID,
│                               │    or geographic resolution
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│  SKU Viability Scoring       │  ← Vectorized Pandas composite score
│  (Stickiness/Reliability/    │    per (Barangay × SKU)
│   Adoption)                  │
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│  Per-Barangay FP-Growth      │  ← Statistical pre-pruning →
│  Market Basket Analysis      │    transaction encoding → rule mining
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│  Symmetrical Deduplication   │  ← Retain only the higher-confidence
│                               │    direction per mirrored pair
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│  Viability Score Synthesis   │  ← Append antecedent & consequent
│                               │    viability via O(1) lookup
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│  Persona Classification      │  ← Translate raw metrics into
│                               │    field sales directives
└─────────────────────────────┘
        │
        ▼
   Field-Ready Excel Output → Sales Team Action
```

---

## 📤 Outputs

A single-sheet Excel workbook, deliberately column-ordered for field sales usability — geography first for territory filtering, persona second for prioritization, then SKU details:

| Column | Description |
|---|---|
| `branch` / `Province` / `Municipality` / `Barangay` | Geographic filtering hierarchy |
| `Persona_Classification` | The actionable sales directive |
| `antecedent_description` | The Anchor SKU (human-readable) |
| `antecedent_viability` | Health score of the Anchor in this barangay |
| `consequent_description` | The pulled/dependent SKU |
| `consequent_viability` | Health score of the dependent SKU in this barangay |
| `support` / `confidence` / `lift` | Underlying statistical metrics for analyst review |

A companion interactive HTML heatmap (`sku_viability_map.html`) visualizes average SKU viability across all barangays for territory planning.

---

## 🛠️ Tech Stack

| Library | Purpose |
|---|---|
| `mlxtend` | FP-Growth frequent itemset mining and association rule extraction |
| `pandas` / `NumPy` | Vectorized viability scoring and data manipulation |
| `GeoPandas` | Geographic data handling for spatial analysis |
| `Folium` | Interactive heatmap visualization |
| `matplotlib` | Persona distribution and statistical synthesis charts |
| `tqdm` | Progress tracking across barangay-level processing loops |
| `SQLAlchemy` + `pyodbc` | Production SQL Server data extraction |
| `openpyxl` | Excel output generation |

---

## ▶️ How to Run

**In Google Colab (recommended):**

Click the **Open in Colab** badge at the top of this README. Run all cells top to bottom via **Runtime → Run all**. Every section includes methodology notes explaining the business reasoning behind each step.

**Locally:**

```bash
git clone https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git
cd YOUR_REPO_NAME
pip install mlxtend geopandas folium pandas numpy matplotlib openpyxl tqdm
jupyter notebook market_basket_sku_analysis.ipynb
```

> The notebook uses synthetically generated transaction data with injected ground-truth product affinities, requiring no external database connection to run in demo mode.

---

*Built with Python 3 · pandas · mlxtend (FP-Growth) · GeoPandas · Folium · matplotlib · openpyxl*
