# Overdrive Repository Summary
## ChainIQ START Hack 2026

---

## 📋 Project Overview

**Overdrive** is a supplier ranking and procurement validation engine developed for the ChainIQ START Hack 2026. It processes procurement requests and returns ranked supplier shortlists with policy evaluation, pricing analysis, escalation triggers, and audit trails.

**Core Purpose:** Automate supplier selection for procurement by scoring suppliers against multiple criteria (risk, quality, ESG, preference status) and validating decisions against historical awards data.

---

## 🏗️ Repository Structure

```
Overdrive/
├── .git/                              # Git repository metadata
├── .gitignore                         # Git ignore rules (excludes .DS_Store, .claude/)
├── .gitmodules                        # Git submodule configuration (if any)
│
├── 📊 DATA DIRECTORY (data/)
│   ├── categories.csv                 # Category taxonomy (L1/L2 classifications)
│   ├── historical_awards.csv          # Historical procurement awards (validation baseline)
│   ├── policies.json                  # Business policies (preferred/restricted suppliers)
│   ├── pricing.csv                    # Supplier pricing by region and category
│   ├── requests.json                  # Sample procurement requests (input)
│   └── suppliers.csv                  # Supplier master data (scores: risk, quality, ESG)
│
├── 🐍 CORE PYTHON MODULES
│   ├── supplier_engine.py             # Main engine (987 lines) - processes requests & ranks suppliers
│   └── scripts/
│       ├── validate_engine.py         # Validation module (271 lines) - compares against historical awards
│       ├── escalation_stats.py        # Escalation analysis (230 lines) - confusion matrix & metrics
│       └── fit_scoring_weights.py     # Weight fitting module (357 lines) - ML model for scoring
│
├── 📦 OUTPUT & CONFIGURATION FILES
│   ├── outputs.json                   # Engine output: ranked suppliers per request (~56K lines)
│   ├── scoring_weights.json           # Fitted ML weights for supplier scoring
│   ├── escalation_report.json         # Escalation analysis results
│   └── validate_report.json           # Validation comparison report
```

---

## 🔧 Core Components

### 1. **supplier_engine.py** — Main Processing Engine
   - **Class:** `SupplierEngine`
   - **Purpose:** Processes a procurement request and returns a ranked supplier shortlist
   - **Key Functions:**
     - Load supplier, pricing, policy, and award data
     - Parse request (category, budget, quantity, country, requirements)
     - Filter suppliers (policy compliance, data residency, capability)
     - Score suppliers using weighted combination of:
       - `risk_score` (0–100) — supplier risk profile
       - `quality_score` (0–100) — quality metrics
       - `esg_score` (0–100) — environmental/social/governance performance
       - `is_preferred` — policy-preferred flag
       - `is_incumbent` — current supplier flag
       - `is_mentioned` — explicitly requested flag
     - Flag escalations (budget issues, policy violations, risk thresholds)
     - Return ranked shortlist with audit trail

   - **Usage:**
     ```python
     from supplier_engine import SupplierEngine
     engine = SupplierEngine()
     result = engine.process(request_dict)
     # Or: python supplier_engine.py  →  writes outputs.json
     ```

   - **Key Mapping:** Country → Region for pricing lookup
     - EU (default), CH, Americas (US/CA/BR/MX), APAC (AU/SG/JP/IN), MEA (UAE/ZA)

### 2. **fit_scoring_weights.py** — ML Weight Optimization
   - **Method:** Pairwise logistic regression (Bradley-Terry model)
   - **Input:** Historical awards CSV (training data)
   - **Output:** `scoring_weights.json` with optimized coefficients
   - **Approach:**
     - For each request with historical award, create (awarded, competitor) pairs
     - Compute feature differences: `diff = features(awarded) - features(competitor)`
     - Fit logistic regression: `P(awarded > competitor) = sigmoid(w · diff)`
     - Positive coefficient = awarded tends to have HIGHER value → subtract in scoring
     - Negative coefficient = awarded tends to have LOWER value → add in scoring

   - **Performance (from scoring_weights.json):**
     - CV Accuracy: 96.2% (±3.5%)
     - Ranking Accuracy: 95.2% (vs 31.2% baseline)
     - +121 correctly ranked requests

### 3. **validate_engine.py** — Historic Validation
   - **Purpose:** Run supplier engine over all requests and compare against historical_awards.csv
   - **Comparison Dimensions:**
     1. `winner_match` — Our rank-1 == historically awarded?
     2. `winner_in_shortlist` — Awarded supplier in our shortlist?
     3. `escalation_match` — Did we flag escalations iff history says needed?
     4. `status_vs_awarded` — If awarded in history, status ≠ cannot_proceed?
   - **Output:**
     - Console: summary table + disagreement breakdown
     - `validate_report.json`: machine-readable results

### 4. **escalation_stats.py** — Escalation Analysis
   - **Purpose:** Confusion matrix analysis for escalation decisions
   - **Matrix:**
     ```
                        History: escalated | History: not escalated
     We: escalated      TP                | FP
     We: not escalated  FN                | TN
     ```
   - **Metrics:** Precision, Recall, F1, Accuracy, Specificity
   - **Breakdowns:** FPs by rule fired, FNs by historical escalation reason
   - **Output:** Console + detailed JSON report

---

## 📊 Data Files

### suppliers.csv
- **Fields:** supplier_id, name, risk_score, quality_score, esg_score, capability_tags
- **Purpose:** Supplier master data with scored metrics

### pricing.csv
- **Fields:** supplier_id, category_l2, region, unit_price, currency
- **Purpose:** Supplier unit pricing by category and geographic region

### policies.json
- **Structure:**
  ```json
  {
    "preferred_suppliers": [{"supplier_id": "...", "category_l2": "..."}],
    "restricted_suppliers": [{"supplier_id": "...", "category_l2": "...", "scope": "..."}]
  }
  ```
- **Purpose:** Business policies for preferred/restricted supplier combinations

### historical_awards.csv
- **Fields:** request_id, supplier_id, awarded (True/False), escalation_required, escalation_reason
- **Purpose:** Ground truth for validation and ML training

### categories.csv
- **Fields:** category_l1, category_l2, description
- **Purpose:** Category taxonomy for procurement classification

### requests.json
- **Structure:** Array of procurement requests with fields like:
  - quantity, unit_of_measure, budget_amount, currency
  - delivery_country, required_by_date
  - preferred_supplier_stated, data_residency_required, esg_requirement
- **Purpose:** Input procurement requests to be processed

---

## 📈 Output Files

### outputs.json (~56K lines)
- **Structure:** Array of processed requests with:
  - `request_id` — unique identifier
  - `processed_at` — ISO timestamp
  - `request_interpretation` — parsed request details
  - `validation` — completeness checks & detected issues
  - `candidate_pool` — filtered suppliers meeting criteria
  - `scoring_results` — ranked suppliers with scores and audit
  - `shortlist` — final ranked recommendation
  - `escalations` — triggered escalation reasons (budget, policy, risk, etc.)
  - `status` — overall result (success, warning, cannot_proceed)

### scoring_weights.json
- **Fitted ML Weights:**
  ```json
  {
    "method": "pairwise logistic regression (Bradley-Terry)",
    "training_pairs": 417,
    "cv_accuracy": {"mean": 0.9616, "std": 0.0352},
    "ranking_accuracy": {
      "baseline": {"correct": 59, "total": 189, "pct": 31.2},
      "fitted_weights": {"correct": 180, "total": 189, "pct": 95.2}
    },
    "coefficients": {
      "risk_score": -4.25713,      // Lower risk is better
      "quality_score": 4.597243,   // Higher quality is better
      "esg_score": -0.201009,      // Minor ESG impact
      "is_preferred": -0.304739,   // Preferred status preferred
      "is_incumbent": 0.004008,    // Negligible incumbent preference
      "is_mentioned": 0.729038     // Mentions matter significantly
    }
  }
  ```

### validate_report.json
- Comparison of engine results vs. historical awards
- Summary stats on winner matching, shortlist inclusion, escalation alignment
- Detailed disagreement breakdown by type

### escalation_report.json
- Confusion matrix for escalation decisions
- Precision/Recall/F1/Accuracy metrics
- False Positive analysis (which rules fire incorrectly)
- False Negative analysis (which historical escalations we missed)

---

## 🚀 How to Use

### Run Main Engine
```bash
python supplier_engine.py
# Reads: data/requests.json, data/suppliers.csv, data/pricing.csv, etc.
# Outputs: outputs.json (all requests processed with recommendations)
```

### Fit Scoring Weights (ML Training)
```bash
python scripts/fit_scoring_weights.py
# Reads: data/historical_awards.csv + suppliers data
# Outputs: scoring_weights.json (optimized weights)
# Trains pairwise logistic regression on historical procurement decisions
```

### Validate Against Historical Data
```bash
python scripts/validate_engine.py
# Compares engine output against historical_awards.csv on 4 dimensions
# Outputs: validate_report.json + console summary
```

### Analyse Escalation Decisions
```bash
python scripts/escalation_stats.py
# Confusion matrix: our escalations vs. history
# Outputs: escalation_report.json + console confusion matrix + metrics
```

---

## 🎯 Key Insights

1. **Scoring Model:** Uses ML-trained weighted scoring with 95.2% ranking accuracy
2. **Policy Integration:** Enforces preferred/restricted suppliers per category
3. **Geographic Awareness:** Pricing varies by region (EU, CH, Americas, APAC, MEA)
4. **Risk Management:** Escalations for budget insufficiency, policy violations, high-risk suppliers
5. **Validation Ready:** Full historical comparison to measure model accuracy
6. **Audit Trail:** Every decision logged with reason and score breakdown

---

## 📝 Dependencies (Inferred)
- Python 3.8+
- `numpy` — numerical operations
- `scikit-learn` — logistic regression, cross-validation, scaling
- Standard library: `json`, `csv`, `pathlib`, `datetime`, `collections`

---

## 🔄 Data Flow

```
requests.json
    ↓
[SupplierEngine.process()]
    ├─ Load: suppliers.csv, pricing.csv, policies.json
    ├─ Parse request → category, budget, country
    ├─ Filter → policy-compliant suppliers
    ├─ Score → weighted combination (risk, quality, ESG, preferred, incumbent, mentioned)
    ├─ Rank → sorted by score
    └─ Flag escalations → budget, policy, risk issues
    ↓
outputs.json (with rankings & escalations)
    ↓
[scripts/validate_engine.py / scripts/escalation_stats.py]
    ├─ Compare vs. historical_awards.csv
    ├─ Measure accuracy (winner match, shortlist, escalations)
    └─ Generate validate_report.json & escalation_report.json
```

---

## 🧠 Machine Learning Pipeline

1. **Historical Data:** historical_awards.csv contains past procurement decisions (awarded supplier per request)
2. **Feature Extraction:** For each request, extract supplier features (risk, quality, ESG, preferred, incumbent, mentioned)
3. **Pairwise Training:** Create (awarded, non-awarded) pairs and compute feature differences
4. **Logistic Regression:** Fit model to learn which features drive winning decisions
5. **Fitted Weights:** Output coefficients that optimize ranking accuracy (31.2% → 95.2%)
6. **Integration:** Drop fitted weights into supplier_engine.py for production scoring

---

## ✅ Validation Metrics

- **Winner Match:** Does our rank-1 recommendation match historical award?
- **Shortlist Rate:** Is historical award within our top-N shortlist?
- **Escalation Precision:** Do we flag escalations when history says needed? (Avoid false alarms)
- **Escalation Recall:** Do we catch all historical escalations?
- **Overall Accuracy:** Consensus across all four validation dimensions

---

**Last Updated:** March 19, 2026  
**Repository:** https://github.com/ArjunSingh48/Overdrive.git
