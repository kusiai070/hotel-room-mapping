# Deterministic Hotel Room Mapping & Entity Resolution Engine
## Audited Benchmark Report: 476 Hotels / 2.75M Pairs (Plexe v5 Frame)

**Author:** Julio Arevalo — KusiAI Research Lab  
**Date:** October 2026  
**Status:** Round 3 Production Baseline  
**Harness Hashes:**  
- `evaluate_benchmark.py`: `CE7A0B08DB7523A5`  
- `diagnostico_benchmark.py`: `D972557CFBF12C63`  
- `veredicto_benchmark.py`: `FB13F3961E4E0D37`  

---

### Executive Summary

Resolving hotel room descriptions across multiple bed banks, GDS feeds, and supplier inventories without human intervention is one of the highest-friction challenges in hospitality distribution. A single physical room type routinely appears under 500 to 1,000 distinct descriptions across languages, punctuation formats, and supplier-specific abbreviations.

In production hotel distribution, false merges (Type II errors) carry severe asymmetric costs: mapping a standard room to an oceanfront or club-access suite leads to guest displacement, chargebacks, and corporate SLA penalties.

We present an audited evaluation of a deterministic, zero-GPU entity resolution engine calibrated on the Plexe v5 public benchmark frame. 

**Key Findings:**
1. **False Merge Rate:** 0.82% (vs 1.22% commercial incumbent reference) — a 32.8% reduction in false merges.
2. **Distinct Name Recall:** 7.06% lost matches on distinct names (vs 7.27% commercial incumbent reference).
3. **Execution Throughput:** 27,174 pairs/second on a single CPU thread (2,755,506 pairs evaluated in 101.4 seconds).
4. **Zero GPU Dependencies:** Deterministic rule extraction, bounded syntactic view argument parsing, and corpus IDF token salience without neural embeddings or black-box inference.

---

### Benchmark Setup & Methodology

The engine was evaluated against the public development frame of the Plexe v5 room mapping benchmark (`KushajM/room-mapping-compare`):

- **Hotels:** 476 dev hotels with valid comparative listings.
- **Supplier Listings:** 26,110 distinct hotel room offers across global suppliers.
- **Pairwise Comparisons:** 2,755,506 pairs.
- **Ground Truth:** Official hotel website room catalogs (physical ground truth, not vendor outputs).
- **Baseline Reference:** Commercial incumbent system as published in the benchmark dataset.

---

### Audited Benchmark Results

Measurements taken with frozen harnesses and verified parameters (`TH_AUTO = 0.95`, `MIN_EVIDENCE = 0.15`, `UNKNOWN_CREDIT = 0.0`):

| System / Model | False Merges (Precision Risk) | Lost Matches (Global) | Lost Matches (Distinct Names) |
|---|---|---|---|
| **Commercial Incumbent (Published)** | 1.22% | **4.51%** | 7.27% |
| **KusiAI Engine (Ronda 1)** | 2.36% | 6.94% | 9.42% |
| **KusiAI Engine (Ronda 2)** | 1.94% | 4.86% | 6.60% |
| **KusiAI Engine (Ronda 3 — Current)** | **0.82%** | 5.21% | **7.06%** |

#### Engineering Trade-off Analysis
The engine deliberately trades 0.70 percentage points of global coverage (5.21% vs 4.51%) to eliminate 0.40 percentage points of false merges (0.82% vs 1.22%). In hospitality distribution economics, a missed match simply falls back to manual review or unmapped queue, whereas a false merge directly causes hotel overbooking and booking displacement.

---

### Generalization & Zero-Leakage Cross-Validation

To verify that the token-salience lexicon does not suffer from data leakage or overfitting, the 476 hotels were split into two independent geographic halves:

- **Half A:** 234 hotels (1,350,200 pairs)
- **Half B:** 242 hotels (1,405,306 pairs)

IDF dictionaries were fitted strictly on one half and evaluated on the held-out half:

| Test Set | Training Corpus | False Merges | Lost Matches | Distinct Lost Matches |
|---|---|---|---|---|
| **Half A** | Half B IDF (Cross-Tested) | 0.56% | 5.99% | 7.86% |
| **Half B** | Half A IDF (Cross-Tested) | 1.30% | 4.41% | 6.27% |
| **Half A** | Global IDF (Reference) | 0.52% | 5.78% | 7.59% |
| **Half B** | Global IDF (Reference) | 1.32% | 4.41% | 6.27% |

The cross-tested evaluation confirms stability and proves that the corpus salience mechanism generalizes across unobserved hotel catalogs.

---

### Component Ablation Study

Ablation of Rule 3 (Corpus IDF Salience Filter):

| Configuration | False Merges | Lost Matches | Distinct Lost Matches | Impact |
|---|---|---|---|---|
| **Rule 3 Inactive** (Lexicon only) | 1.62% | 5.01% | 6.80% | Baseline |
| **Rule 3 Active** (Corpus IDF) | **0.82%** | 5.21% | 7.06% | **-0.80 pts false merges** (+0.20 pts lost matches) |
| **Rule 3 Firing Always** | 0.76% | 5.37% | 7.29% | Over-vetoing |

Corpus IDF salience contributes 71% of the total precision gain in Round 3 by vetoing unmapped geographical or distinctive view modifiers (e.g., `Bosphorus view` vs `Taksim square view`) without requiring comprehensive world knowledge dictionaries.

---

### Computational Performance & Infrastructure Economics

- **Total Execution Time:** 101.4 seconds for 2,755,506 pairs.
- **Single-Threaded Throughput:** 27,174 pairs/second.
- **Cost per Comparison:** 36.8 microseconds on commodity x86 CPU.
- **Hardware Footprint:** Zero GPUs, no external embedding API calls, sub-50MB RAM working set.
- **Auditability:** Every merge decision is fully explainable by an explicit feature trace and deterministic rule ID.

---

### Failure Mode Taxonomy & Remaining Challenges

1. **Asymmetric Lexical Residuals:**  
   `grand suite - city view` vs `premium suite - city view` currently results in false merge when modifiers are consumed prior to class categorization. Identified for resolution in Round 3.1.
2. **Property Management System (PMS) Verification:**  
   Empirical validation against real 5-star hotel PMS setups (e.g. Hyatt Regency, 465 keys, 15 room types) proves that 4 commercial room products (`PREMIUM KING`, `PREMIUM KING WITH VIEW`, `PREMIUM KING CLUB ACCESS`, `PREMIUM KING CLUB ACCESS WITH VIEW`) share an identical 29 m2 footprint and King bed. Black-box embedding similarity engines merge these products erroneously; deterministic entity vetoes are strictly required to preserve inventory integrity.
