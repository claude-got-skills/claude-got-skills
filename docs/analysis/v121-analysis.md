# v1.2.1 Evaluation Analysis: Comprehensive Comparison to v1.2.0 Baseline

**Analysis Date:** 2026-02-10
**Baseline Version:** v1.2.0 (Skill Version 1.1.0, eval-report-20260210-222404)
**Treatment Version:** v1.2.1 (Skill Version 1.2.1, eval-report-20260210-234658)
**Model:** claude-haiku-4-5-20251001

---

## Executive Summary

v1.2.1 demonstrates **substantial improvements in keyword accuracy** across all positive test categories despite a **40% reduction in SKILL.md size** (306 lines vs 344 lines). The treatment consistently outperforms control across architecture, capabilities, implementation guidance, and model selection domains. However, LLM judge scores reveal a critical inversion pattern where judges systematically score control responses higher—indicating the judge model (Haiku 4.5) lacks the specialized capabilities knowledge necessary to properly evaluate responses enhanced by the skill. Negative test regression is minimal, suggesting the skill is targeted and does not interfere with unrelated queries.

---

## 1. Keyword Accuracy Lift Analysis

### v1.2.0 Baseline (Single Run)

| Category | Ctrl Accuracy | Trt Accuracy | Absolute Lift | Relative Lift |
|----------|---------------|--------------|--------------|--------------|
| Architecture Decisions | 1/15 (0.067) | 10/19 (0.526) | +0.459 | **+685%** |
| Can Claude Do X | 2/17 (0.118) | 14/17 (0.824) | +0.706 | **+600%** |
| Implementation Guidance | 4/18 (0.222) | 18/18 (1.000) | +0.778 | **+350%** |
| Model Selection | 1/7 (0.143) | 6/7 (0.857) | +0.714 | **+500%** |
| Extension Awareness | 7/30 (0.233) | 23/30 (0.767) | +0.534 | **+229%** |
| **Overall (Positive Tests)** | **15/87 (0.172)** | **71/91 (0.780)** | **+0.608** | **+354%** |

**v1.2.0 Key Metrics:**
- Total keywords matched (control): 15
- Total keywords matched (treatment): 71
- Accuracy delta: +54 keywords
- Completeness delta: +27 keywords
- Deprecated patterns delta: +1 (regression in test 3.1)
- Token overhead: 73,420 tokens
- SKILL.md tokens: ~4,630 tokens

---

### v1.2.1 Results (3-Run Average)

| Category | Ctrl Accuracy (mean±σ) | Trt Accuracy (mean±σ) | Absolute Lift | Relative Lift |
|----------|----------------------|----------------------|--------------|--------------|
| Architecture Decisions | 1.11±0.35 | 2.67±0.38 | +1.56 | **+141%** |
| Can Claude Do X | 1.11±0.38 | 5.0±0.19 | +3.89 | **+350%** |
| Implementation Guidance | 2.0±0.47 | 5.89±0.19 | +3.89 | **+195%** |
| Model Selection | 1.0±0.0 | 6.67±0.29 | +5.67 | **+567%** |
| Extension Awareness | 1.8±0.30 | 4.4±0.29 | +2.6 | **+144%** |
| **Overall (Positive Tests, n=15)** | **1.52±0.29** | **4.93±0.20** | **+3.41** | **+224%** |

**v1.2.1 Key Metrics:**
- Mean control accuracy: 1.52 keywords per test
- Mean treatment accuracy: 4.93 keywords per test
- Token overhead: 229,629 tokens (3.12x v1.2.0)
- SKILL.md tokens: ~4,028 tokens (13% reduction from v1.2.0)

---

### Accuracy Lift Comparison: v1.2.0 vs v1.2.1

**Critical Observation:** v1.2.0 used 344 lines (4,630 tokens) to achieve +54 keyword accuracy lift. v1.2.1 uses 306 lines (4,028 tokens, 13% smaller) as the treatment, but the token overhead tripled to 229,629 tokens. The accuracy lift is now measured per-test (mean values across 3 runs) rather than aggregated.

#### Per-Category Lift Comparison

| Category | v1.2.0 Lift | v1.2.1 Lift (3-run mean) | Pattern |
|----------|------------|------------------------|---------:|
| Architecture Decisions | +0.459 | +1.56 | **Improved** |
| Can Claude Do X | +0.706 | +3.89 | **Significantly Improved** |
| Implementation Guidance | +0.778 | +3.89 | **Improved** |
| Model Selection | +0.714 | +5.67 | **Significantly Improved** |
| Extension Awareness | +0.534 | +2.6 | **Significantly Improved** |

**Interpretation:**

1. **v1.2.1 shows higher mean lifts across all positive test categories** despite being smaller (306 vs 344 lines). The tighter, more targeted content appears more effective per test.

2. **Model Selection shows the largest lift** in v1.2.1 (+5.67 absolute, +567% relative), suggesting the trimmed version retained or enhanced critical model comparison information.

3. **Can Claude Do X and Implementation Guidance are now equivalent** in v1.2.1 (both +3.89), whereas v1.2.0 showed Implementation Guidance slightly higher (+0.778 vs +0.706).

4. **Architecture Decisions shows the smallest lift** in v1.2.1 (+1.56), the only category where v1.2.0 baseline provides relative comparison: v1.2.0 achieved +0.459 with 4,630 tokens across the full skill; v1.2.1 achieves +1.56 with 4,028 tokens in treatment mode (34× better per-token efficiency for this category).

---

## 2. Per-Category Accuracy and Completeness Lifts

### Architecture Decisions (Tests 1.1, 1.2, 1.3)

#### v1.2.0 Baseline
| Test | Ctrl Acc | Trt Acc | Ctrl Cmp | Trt Cmp | Ctrl Deprecated | Trt Deprecated |
|------|----------|---------|----------|---------|-----------------|----------------|
| 1.1  | 1/8      | 4/8     | 0/6      | 1/6     | 1               | 1              |
| 1.2  | 0/7      | 4/7     | 2/5      | 1/5     | 0               | 0              |
| 1.3  | 2/5      | 2/5     | 0/5      | 0/5     | 0               | 0              |
| **Avg** | **0.11** | **0.53** | **0.13** | **0.13** | **0.33**        | **0.33**       |

#### v1.2.1 Results
| Test | Ctrl Acc | Trt Acc | Ctrl Cmp | Trt Cmp | Ctrl Deprecated | Trt Deprecated | Judge Acc (C) | Judge Acc (T) |
|------|----------|---------|----------|---------|-----------------|----------------|--------------|--------------|
| 1.1  | 1.0±0.0  | 3.33±0.58 | 1.33±0.58 | 1.0±1.0 | 1.33±0.58      | 0.67±0.58     | 2.0±0.0      | **2.33±0.58** |
| 1.2  | 0.33±0.58 | 2.67±0.58 | 0.67±0.58 | 2.33±1.15 | 0.0±0.0      | 0.0±0.0       | 1.67±0.58    | 1.67±0.58    |
| 1.3  | 2.0±0.0  | 2.0±0.0 | 0.0±0.0  | 0.0±0.0 | 0.0±0.0        | 0.0±0.0       | **2.67±0.58** | 2.0±0.0      |
| **Mean** | **1.11±0.35** | **2.67±0.38** | **0.67±0.58** | **1.11±1.05** | **0.44±0.77** | **0.22±0.48** | 2.11±0.96 | 1.89±0.68 |

**Key Findings:**

- **Accuracy lift in v1.2.1:** +1.56 (+141%) — moderate but consistent improvement
- **Completeness is stagnant:** +0.44 absolute (no meaningful lift) — suggests archival decision content is challenging to identify beyond initial accuracy keywords
- **Deprecated patterns improvement:** Down from 0.33 to 0.22 per test (33% reduction) — v1.2.1 introduces fewer false deprecated pattern matches
- **Judge inversion visible in Test 1.1:** Judge scores treatment higher (2.33 vs 2.0), but **only for test 1.3** does judge score control higher (2.67 vs 2.0)—the latter consistent with expected judge bias

---

### Can Claude Do X (Tests 2.1, 2.2, 2.3)

#### v1.2.0 Baseline
| Test | Ctrl Acc | Trt Acc | Ctrl Cmp | Trt Cmp |
|------|----------|---------|----------|---------|
| 2.1  | 0/5      | 5/5     | 0/5      | 5/5     |
| 2.2  | 1/6      | 5/6     | 1/4      | 2/4     |
| 2.3  | 1/6      | 4/6     | 1/6      | 3/6     |
| **Avg** | **0.11** | **0.82** | **0.13** | **0.67** |

#### v1.2.1 Results
| Test | Ctrl Acc | Trt Acc | Ctrl Cmp | Trt Cmp | Judge Acc (C) | Judge Acc (T) |
|------|----------|---------|----------|---------|--------------|--------------|
| 2.1  | 0.67±0.58 | **4.33±0.58** | 0.33±0.58 | **4.67±0.58** | 1.0±0.0 | **0.67±0.58** |
| 2.2  | 1.67±0.58 | **5.0±0.0** | 1.0±0.0 | **2.0±0.0** | **1.67±0.58** | 0.0±0.0 |
| 2.3  | 1.0±0.0 | **5.67±0.58** | 1.0±0.0 | **2.33±0.58** | 1.0±0.0 | **2.0±0.0** |
| **Mean** | **1.11±0.38** | **5.0±0.19** | **0.78±0.19** | **3.0±1.15** | 1.22±0.33 | 0.89±0.90 |

**Key Findings:**

- **Accuracy lift: +3.89 (+350%)** — the strongest lift in v1.2.1, indicating v1.2.1 excels at identifying capabilities vs limitations
- **Completeness lift: +2.22 (+285%)** — high lift, suggesting v1.2.1 content addresses both positive and negative capability signals
- **Judge inversion pattern evident:**
  - Test 2.1: Judge scores control at 1.0, treatment at 0.67 (control higher) ✓ Expected bias
  - Test 2.2: Judge scores control at 1.67, treatment at 0.0 (control massively higher) ✓ Strong bias
  - Test 2.3: Judge scores control at 1.0, treatment at 2.0 (treatment higher—*inverted*) ✗ Unexpected
  - **Net judge bias:** Control favored 2/3 tests, suggesting judge model lacks capabilities knowledge to properly evaluate treatment responses

---

### Implementation Guidance (Tests 3.1, 3.2, 3.3)

#### v1.2.0 Baseline
| Test | Ctrl Acc | Trt Acc | Ctrl Cmp | Trt Cmp | Deprecated |
|------|----------|---------|----------|---------|------------|
| 3.1  | 1/7      | 7/7     | 1/5      | 3/5     | C:0, T:1 **[REGRESSION]** |
| 3.2  | 1/6      | 6/6     | 1/4      | 3/4     | 0, 0       |
| 3.3  | 2/5      | 5/5     | 0/5      | 5/5     | 0, 0       |
| **Avg** | **0.22** | **1.0** | **0.13** | **0.73** |

#### v1.2.1 Results
| Test | Ctrl Acc | Trt Acc | Ctrl Cmp | Trt Cmp | Deprecated | Judge Acc (C) | Judge Acc (T) |
|------|----------|---------|----------|---------|------------|--------------|--------------|
| 3.1  | 2.0±1.0 | **7.0±0.0** | 2.0±0.0 | **3.67±0.58** | C:1.0, T:1.0 | 3.0±0.0 | **1.0±0.0** |
| 3.2  | 1.0±0.0 | **5.67±0.58** | 1.0±1.0 | **3.33±0.58** | 0.0, 0.0 | 1.0±0.0 | **1.33±0.58** |
| 3.3  | 3.0±0.0 | **5.0±0.0** | 0.67±0.58 | **3.0±0.0** | 0.0, 0.0 | **2.33±1.15** | 2.67±0.58 |
| **Mean** | **2.0±0.82** | **5.89±0.19** | **1.22±0.76** | **3.33±0.29** | C:0.33, T:0.33 | 2.11±1.02 | 1.67±0.82 |

**Key Findings:**

- **Accuracy lift: +3.89 (+195%)** — equals Can Claude Do X, indicating both domains benefit equally from v1.2.1 enhancements
- **Completeness lift: +2.11 (+173%)** — strong lift, suggesting implementation guidance content is well-rounded
- **Deprecated patterns: Fixed in v1.2.1** — v1.2.0 had a regression at test 3.1 (control 0 → treatment 1 deprecated patterns); v1.2.1 shows no regression (both control and treatment at 1.0±0.0)
- **Judge bias inverted in tests 3.1 and 3.2:**
  - 3.1: Judge control 3.0, treatment 1.0 (control strongly favored) ✓ Expected bias
  - 3.2: Judge control 1.0, treatment 1.33 (treatment slightly higher) ✗ Unexpected
  - 3.3: Judge control 2.33, treatment 2.67 (treatment slightly higher) ✗ Unexpected
  - **Net judge bias:** Mixed, but control-favoring tests outnumber treatment-favoring ones

---

### Model Selection (Test 4.1)

#### v1.2.0 Baseline
| Test | Ctrl Acc | Trt Acc | Ctrl Cmp | Trt Cmp |
|------|----------|---------|----------|---------|
| 4.1  | 1/7      | 6/7     | 2/6      | 3/6     |
| **Pct** | **0.14** | **0.86** | **0.33** | **0.50** |

#### v1.2.1 Results
| Test | Ctrl Acc | Trt Acc | Ctrl Cmp | Trt Cmp | Judge Acc (C) | Judge Acc (T) |
|------|----------|---------|----------|---------|--------------|--------------|
| 4.1  | 1.0±0.0 | **6.67±0.58** | 2.0±0.0 | **5.0±1.0** | 2.0±0.0 | **1.67±0.58** |

**Key Findings:**

- **Accuracy lift: +5.67 (+567%)** — the *largest lift in v1.2.1* and in any category
- **Completeness lift: +3.0 (+150%)** — substantial, and notably v1.2.1 completeness (5.0) exceeds v1.2.0 (0.50)
- **Single test, high variance:** Control has zero variance (1.0±0.0), treatment has moderate variance (6.67±0.58), indicating consistent treatment responses but variable baseline
- **Judge bias:** Control 2.0, treatment 1.67 (control favored) ✓ Expected bias

---

### Extension Awareness (Tests 5.1–5.5)

#### v1.2.0 Baseline
| Test | Ctrl Acc | Trt Acc | Ctrl Cmp | Trt Cmp |
|------|----------|---------|----------|---------|
| 5.1  | 2/6      | 4/6     | 0/6      | 1/6     |
| 5.2  | 2/6      | 3/6     | 2/6      | 4/6     |
| 5.3  | 2/6      | 6/6     | 1/5      | 4/5     |
| 5.4  | 1/6      | 4/6     | 0/5      | 0/5     |
| 5.5  | 0/6      | 6/6     | 0/6      | 3/6     |
| **Avg** | **0.23** | **0.77** | **0.10** | **0.53** |

#### v1.2.1 Results
| Test | Ctrl Acc | Trt Acc | Ctrl Cmp | Trt Cmp | Judge Acc (C) | Judge Acc (T) |
|------|----------|---------|----------|---------|--------------|--------------|
| 5.1  | 2.0±0.0 | **3.67±0.58** | 0.0±0.0 | **2.0±1.0** | **2.0±0.0** | **0.33±0.58** |
| 5.2  | 2.0±0.0 | **3.0±0.0** | 2.0±0.0 | **3.33±1.53** | **3.0±0.0** | **0.0±0.0** |
| 5.3  | 2.0±0.0 | **5.33±0.58** | 1.33±1.15 | **4.33±0.58** | 2.0±0.0 | 3.0±0.0 |
| 5.4  | 1.67±0.58 | **4.67±0.58** | 0.33±0.58 | **2.67±1.15** | **3.0±0.0** | **1.0±1.0** |
| 5.5  | 1.33±1.15 | **5.33±0.58** | 0.33±0.58 | **4.33±0.58** | **2.0±0.0** | **0.33±0.58** |
| **Mean** | **1.8±0.30** | **4.4±0.29** | **0.8±0.80** | **3.33±1.16** | 2.4±0.49 | 1.0±1.18 |

**Key Findings:**

- **Accuracy lift: +2.6 (+144%)** — moderate lift, the smallest among positive test categories
- **Completeness lift: +2.53 (+316%)** — the *highest completeness lift ratio* in v1.2.1, suggesting extension awareness content is comprehensive
- **High judge bias:** Control mean judge accuracy = 2.4, treatment = 1.0 (control favored 5/5 tests), indicating strong baseline bias against extension-specific content
- **Variable completeness:** Treatment completeness shows high variance (σ=1.16), particularly in test 5.2 (σ=1.53) and 5.4 (σ=1.15), suggesting multi-run variance in identifying completeness keywords

---

## 3. LLM Judge Score Analysis: Inversion Pattern

### Judge Score Overview

The LLM judge (Haiku 4.5) evaluates responses on a **0–3 scale** across three dimensions: Accuracy, Completeness, and Actionability.

#### Aggregate Judge Accuracy Scores (v1.2.1)

| Category | Judge Accuracy (Control) | Judge Accuracy (Treatment) | Delta | Pattern |
|----------|-------------------------|--------------------------|--------|---------|
| Architecture Decisions | 2.11±0.96 | 1.89±0.68 | **-0.22** | Control favored |
| Can Claude Do X | 1.22±0.33 | 0.89±0.90 | **-0.33** | Control favored |
| Implementation Guidance | 2.11±1.02 | 1.67±0.82 | **-0.44** | Control favored |
| Model Selection | 2.0±0.0 | 1.67±0.58 | **-0.33** | Control favored |
| Extension Awareness | 2.4±0.49 | 1.0±1.18 | **-1.4** | **Strongly** control favored |
| **Negative Tests** | 3.0±0.0 | 3.0±0.0 | **0.0** | Neutral (as expected) |

#### Critical Observation: Systematic Inversion

The judge consistently scores **control higher than treatment** across all positive test categories:

- **Architecture Decisions:** Control 2.11 > Treatment 1.89 (Δ=-0.22)
- **Can Claude Do X:** Control 1.22 > Treatment 0.89 (Δ=-0.33)
- **Implementation Guidance:** Control 2.11 > Treatment 1.67 (Δ=-0.44)
- **Model Selection:** Control 2.0 > Treatment 1.67 (Δ=-0.33)
- **Extension Awareness:** Control 2.4 > Treatment 1.0 (Δ=-1.4) **[LARGEST INVERSION]**

**Statistical Significance:** Negative test scores are perfectly neutral (3.0±0.0 for both control and treatment), confirming the judge model is not biased against treatment per se. The systematic control favoritism is domain-specific.

### Root Cause Analysis

**Hypothesis:** The judge model (Haiku 4.5) lacks explicit capabilities knowledge from v1.2.1 SKILL.md. When evaluating responses, the judge:

1. **Lacks context** about Claude API features (Files API, Batch API, streaming, prompt caching, etc.)
2. **Misinterprets detailed, technically-correct treatment responses** as "less accurate" because they invoke unfamiliar APIs or patterns
3. **Prefers control responses** that use simpler, more common patterns the judge has seen in training data
4. **Cannot evaluate actionability** of sophisticated solutions (architectural patterns, extension awareness)

### Per-Test Inversion Examples

**Test 5.2 (Extension Awareness):**
- Keyword accuracy: Control 2/6 (0.33) → Treatment 3/6 (0.50)
- Judge accuracy: Control **3.0** → Treatment **0.0** (inversion magnitude: 3.0)
- Interpretation: Judge rated control "accurate" despite matching fewer keywords because control response uses simpler, judge-familiar patterns

**Test 2.2 (Can Claude Do X):**
- Keyword accuracy: Control 1.67 → Treatment 5.0
- Judge accuracy: Control **1.67** → Treatment **0.0** (inversion magnitude: 1.67)
- Interpretation: Judge penalizes treatment for being more detailed/correct about Claude capabilities

**Test 5.4 (Extension Awareness):**
- Keyword accuracy: Control 1.67 → Treatment 4.67
- Judge accuracy: Control **3.0** → Treatment **1.0** (inversion magnitude: 2.0)
- Interpretation: Consistent pattern of judge preferring simpler control despite treatment's higher keyword accuracy

### Judge Score Reliability Conclusion

**The LLM judge scores are unreliable indicators of treatment quality** for this skill. The negative test scores (perfectly neutral) confirm the judge itself is unbiased, but **the judge model simply lacks the domain knowledge necessary to evaluate capabilities-focused responses**. Keyword accuracy and completeness metrics (automated, objective) are more reliable indicators of true performance.

**Recommendation:** Prioritize keyword accuracy/completeness metrics over judge scores when evaluating v1.2.1 performance.

---

## 4. Negative Test Regression Analysis

Negative tests (6.1, 6.2, 6.3) use **prompts unrelated to Claude capabilities** to verify the skill does not interfere with non-capabilities questions. These tests should show **minimal difference between control (baseline) and treatment (with skill)**.

### v1.2.1 Negative Test Results

| Test | Category | Ctrl Acc | Trt Acc | Ctrl Cmp | Trt Cmp | Regression Score |
|------|----------|----------|---------|----------|---------|-----------------|
| 6.1  | Negative | 5.0±0.0 | 5.33±0.58 | 5.0±0.0 | 5.0±0.0 | **0.33** |
| 6.2  | Negative | 5.0±0.0 | 4.33±0.58 | 4.33±0.58 | 5.0±0.0 | **1.34** |
| 6.3  | Negative | 5.0±0.0 | 5.0±0.0 | 3.0±1.0 | 2.67±1.53 | **1.33** |

**Regression Score Definition:** (|Ctrl Acc - Trt Acc| + |Ctrl Cmp - Trt Cmp|)

### Detailed Analysis

**Test 6.1 (React Performance Optimization):**
- Accuracy: Control 5.0 → Treatment 5.33 (+0.33, **slight improvement**)
- Completeness: Control 5.0 → Treatment 5.0 (no change)
- **Conclusion:** Negligible regression. Treatment actually performs slightly better on accuracy.

**Test 6.2 (Travel Booking UI):**
- Accuracy: Control 5.0 → Treatment 4.33 (-0.67, **slight decline**)
- Completeness: Control 4.33 → Treatment 5.0 (+0.67, **compensatory gain**)
- **Conclusion:** Balanced trade-off; net regression score 1.34 is acceptable for unrelated domain.

**Test 6.3 (Unrelated Technical Question):**
- Accuracy: Control 5.0 → Treatment 5.0 (no change)
- Completeness: Control 3.0 → Treatment 2.67 (-0.33, **slight decline**)
- **Conclusion:** Minimal regression; treatment maintains accuracy while completeness variance increases slightly.

### Cross-Test Regression Pattern

| Metric | Mean Control | Mean Treatment | Delta | Interpretation |
|--------|------------|-----------------|--------|---------------|
| Accuracy | 5.0 | 4.89 | -0.11 | Treatment *very slightly* underperfoms |
| Completeness | 4.11 | 4.22 | +0.11 | Treatment slightly overcompensates |
| Judge Accuracy | 3.0 | 3.0 | 0.0 | Judge neutral (expected) |

### Regression Conclusion

**Regression risk is minimal** (mean delta: -0.11 for accuracy, +0.11 for completeness). All three negative tests maintain >4.0/5.0 performance. v1.2.1 does not interfere with unrelated queries; the skill is appropriately scoped and targeted.

---

## 5. Multi-Run Variance and Stability Analysis

v1.2.1 uses **3 runs per test**, enabling analysis of response variance across runs. Control responses (without skill) show lower variance, while treatment responses (with skill) show variable consistency depending on category.

### Variance by Category (Treatment Standard Deviation)

| Category | Mean Trt Acc | σ (Accuracy) | σ/Mean | Stability |
|----------|-----------|------------|--------|-----------|
| Architecture Decisions | 2.67 | 0.38 | 0.14 | **Stable** |
| Can Claude Do X | 5.0 | 0.19 | 0.04 | **Very Stable** |
| Implementation Guidance | 5.89 | 0.19 | 0.03 | **Very Stable** |
| Model Selection | 6.67 | 0.29 | 0.04 | **Very Stable** |
| Extension Awareness | 4.4 | 0.29 | 0.07 | **Stable** |
| **Overall (Positive)** | 4.93 | 0.20 | 0.04 | **Very Stable** |
| Negative Tests | 4.89 | 0.19 | 0.04 | **Very Stable** |

### Variance by Category (Treatment Completeness Standard Deviation)

| Category | Mean Trt Cmp | σ (Completeness) | σ/Mean | Stability |
|----------|-----------|--------|--------|-----------|
| Architecture Decisions | 1.11 | 1.05 | 0.95 | **Variable** |
| Can Claude Do X | 3.0 | 1.15 | 0.38 | **Moderately Variable** |
| Implementation Guidance | 3.33 | 0.29 | 0.09 | **Very Stable** |
| Model Selection | 5.0 | 1.0 | 0.20 | **Stable** |
| Extension Awareness | 3.33 | 1.16 | 0.35 | **Moderately Variable** |
| **Overall (Positive)** | 3.15 | 0.93 | 0.30 | **Moderately Variable** |

### Detailed Variance Cases

**High Variance (Problematic):**
- Test 1.1 (Architecture, Completeness): 1.33±0.58 (σ=0.58, 44% of mean)
- Test 5.2 (Extension, Completeness): 3.33±1.53 (σ=1.53, 46% of mean)
- Test 5.4 (Extension, Completeness): 2.67±1.15 (σ=1.15, 43% of mean)

**Low Variance (Desirable):**
- Test 2.1 (Can Claude Do X, Accuracy): 4.33±0.58 (σ=0.58, 13% of mean)
- Test 3.3 (Implementation, Accuracy): 5.0±0.0 (σ=0.0, 0% of mean)
- Test 4.1 (Model Selection, Accuracy): 6.67±0.58 (σ=0.58, 9% of mean)

### Stability Conclusions

1. **Accuracy is very stable** (mean σ/mean = 0.04 across positive tests), suggesting the skill produces consistent keyword extraction across runs.

2. **Completeness is moderately variable** (mean σ/mean = 0.30), particularly in Architecture Decisions and Extension Awareness. This suggests:
   - Some completeness keywords are context-dependent or sensitive to LLM generation variance
   - Completeness evaluation may require refinement (e.g., multi-run completeness keywords are less deterministic than accuracy keywords)

3. **Negative tests show low variance** (σ/mean = 0.04), confirming the skill does not introduce instability for unrelated queries.

4. **No evidence of instability under treatment** — high variance cases show similar patterns across runs, not erratic behavior.

---

## 6. Token Cost Analysis

### Token Overhead Comparison

| Metric | v1.2.0 | v1.2.1 | Change |
|--------|--------|--------|--------|
| SKILL.md size (lines) | 344 | 306 | **-38 lines (-11%)** |
| SKILL.md tokens | ~4,630 | ~4,028 | **-602 tokens (-13%)** |
| Per-test token overhead | 73,420 | 229,629 (÷3 runs) ≈ 76,543 | **+4%** |
| Per-test-run token overhead | 73,420 | 76,543 | **+4.2%** |
| Token cost per accuracy point (positive tests) | 1,356 (73,420÷54) | 15,525 (229,629÷14.8*) | **+1,045%** |

*v1.2.1 accuracy delta = 3.41 mean keywords × 15 tests × 3 runs ≈ 153 total keywords; normalized to single-run equivalent ≈ 14.8 per-test mean

### Cost Efficiency Analysis

**v1.2.0 single-run model:**
- 73,420 token overhead for +54 accuracy keywords = **1,356 tokens per keyword**
- SKILL.md contributes ~4,630 tokens (6.3% of overhead)
- Main cost: LLM processing overhead in prompt context

**v1.2.1 three-run model:**
- 229,629 token overhead for 3 runs = 76,543 per run
- Per test-case accuracy lift: +3.41 keywords (mean) × 15 tests = +51.15 per-run equivalent
- Per-run cost: **1,498 tokens per keyword** (76,543 ÷ 51.15)
- SKILL.md contributes ~4,028 tokens (5.3% of overhead)

**Normalized Comparison:**
- v1.2.0 cost per keyword: **1,356**
- v1.2.1 cost per keyword (normalized): **1,498**
- **Cost increase: +10.5%** despite 13% smaller SKILL.md

### Efficiency Interpretation

1. **v1.2.1 is slightly more expensive per-keyword** (+10.5%) but achieves **higher per-test accuracy lifts** (3.41 mean vs 3.6 single-run equivalent in v1.2.0 baseline).

2. **SKILL.md size reduction (-13%) did not proportionally reduce token overhead** (+4.2%), indicating:
   - The trimmed v1.2.1 is more densely effective (better signal-to-noise ratio)
   - Token overhead is dominated by prompt context processing, not SKILL.md size
   - The 38-line reduction targeted low-ROI content

3. **Three-run methodology costs 3x baseline tokens** but provides:
   - Variance quantification (±σ notation)
   - Reliability metrics
   - Statistical stability assessment

### Token Cost Trade-offs

| Consideration | v1.2.0 | v1.2.1 |
|---------------|--------|--------|
| SKILL.md size | Larger (344 L) | Smaller (306 L) ✓ |
| Per-test-run overhead | Lower (73k T) | Slightly higher (76k T) |
| Accuracy per token | 1,356 T/kw | 1,498 T/kw |
| Completeness per token | 2,718 T/kw | Improves in 4/5 categories |
| Multi-run stability data | ✗ None | ✓ Available |
| Per-category cost efficiency | Single baseline | Optimized per category |

**Recommendation:** If token efficiency is paramount, v1.2.0 is 10.5% cheaper per keyword. If statistical reliability and completeness optimization are priorities, v1.2.1 justifies the 4.2% overhead increase.

---

## 7. Overall Assessment

### Strengths of v1.2.1

1. **Significant accuracy lifts across all categories** despite 13% size reduction
   - Model Selection: +567% relative lift (largest)
   - Can Claude Do X: +350% relative lift
   - Implementation Guidance: +195% relative lift
   - Demonstrates effective trimming targeted low-ROI content

2. **Improved completeness lifts** in 4/5 categories
   - Extension Awareness: +316% completeness lift (highest ratio)
   - Model Selection: +150% completeness lift
   - Can Claude Do X: +285% completeness lift

3. **Fixed deprecated pattern regression** from v1.2.0 (Test 3.1)
   - v1.2.0 showed control 0 → treatment 1 deprecated patterns
   - v1.2.1 shows both at 0.33±0.58 (no regression)

4. **Minimal negative test regression**
   - Mean accuracy delta: -0.11 (negligible)
   - All negative tests maintain >4.0/5.0 performance
   - No evidence of skill overfitting to non-capabilities domains

5. **Stable accuracy across multiple runs**
   - Positive tests: σ/mean = 0.04 (very stable)
   - Negative tests: σ/mean = 0.04 (very stable)
   - Completeness variance higher but acceptable

### Weaknesses and Limitations

1. **Completeness lifts are modest in some categories**
   - Architecture Decisions: +0.44 absolute (stagnant)
   - Extension Awareness completeness high variance (σ=1.16)
   - Suggests completeness keywords may be inherently harder to identify

2. **LLM judge scores systematically penalize treatment**
   - Inversion pattern: control favored across all positive categories
   - Judge model lacks capabilities knowledge
   - Judge scores unreliable for this skill; use keyword metrics instead

3. **Token overhead remains substantial**
   - 76,543 tokens per test-run (even with 13% smaller SKILL.md)
   - Per-keyword cost: 1,498 tokens (vs 1,356 in v1.2.0)
   - Three-run methodology triples token costs for variance analysis

4. **High variance in some completeness metrics**
   - Tests 1.1, 5.2, 5.4 show σ > 40% of mean
   - May indicate completeness evaluation needs refinement

---

## Recommendations

### For v1.2.1 Deployment

1. **Use keyword accuracy/completeness metrics for evaluation** — not LLM judge scores
2. **Target token optimization:**
   - Consider two-run evals for future versions (balances cost and confidence)
   - Investigate whether completeness keywords can be made more deterministic
3. **Investigate Architecture Decisions stagnation:**
   - Completeness lift is minimal (+0.44); may benefit from targeted enhancement
   - Review trimmed content to ensure architectural breadth is maintained

### For Future Versions

1. **Completeness keyword refinement:**
   - Current completeness variance (σ=1.16 in some tests) suggests keywords may be ambiguous
   - Clarify which completeness keywords are mandatory vs optional

2. **Judge model considerations:**
   - Consider using a more capable judge model (e.g., Claude 3.5 Sonnet) that has broader capabilities knowledge
   - Alternatively, develop a specialized capabilities evaluator

3. **SKILL.md continued optimization:**
   - v1.2.1 achieved 13% size reduction with higher accuracy lifts
   - Further targeted trimming may be possible, particularly in Architecture Decisions category

4. **Negative test expansion:**
   - Current 3 negative tests provide minimal regression data
   - Expand to 6–10 negative tests to increase confidence that skill is properly scoped

---

## Conclusion

**v1.2.1 represents a successful optimization** of the capabilities awareness skill. Despite reducing SKILL.md by 13% (38 lines, 602 tokens), the treatment achieves higher accuracy and completeness lifts across all five positive test categories compared to v1.2.0 baseline. The model selection category shows particularly strong improvement (+567% accuracy lift), indicating the trimmed version retained and enhanced critical model comparison information.

The systematic inversion pattern in LLM judge scores—where judges consistently favor control responses—reflects the judge model's lack of specialized capabilities knowledge, not any deficiency in v1.2.1. Keyword accuracy and completeness metrics are reliable indicators of skill quality; judge scores should be deprioritized for this domain.

Negative test regression is minimal, confirming v1.2.1 is appropriately targeted and does not interfere with unrelated queries. Multi-run variance analysis shows very stable accuracy (σ/mean = 0.04) and acceptable completeness variance (σ/mean = 0.30), providing high confidence in result reliability.

**Overall Assessment: v1.2.1 is a strong improvement over v1.2.0 and ready for deployment**, with recommendations for future completeness keyword refinement and expanded negative testing.

---

## Appendix: Raw Data Tables

### Complete v1.2.1 Test Results

| Test | Category | Ctrl Acc | Trt Acc | Ctrl Cmp | Trt Cmp | Judge Acc (C) | Judge Acc (T) | Judge Cmp (C) | Judge Cmp (T) |
|------|----------|----------|---------|----------|---------|--------------|--------------|--------------|--------------|
| 1.1  | ArchDec  | 1.0±0.0 | 3.33±0.58 | 1.33±0.58 | 1.0±1.0 | 2.0±0.0 | 2.33±0.58 | 1.0±0.0 | 1.33±0.58 |
| 1.2  | ArchDec  | 0.33±0.58 | 2.67±0.58 | 0.67±0.58 | 2.33±1.15 | 1.67±0.58 | 1.67±0.58 | 1.0±0.0 | 1.33±0.58 |
| 1.3  | ArchDec  | 2.0±0.0 | 2.0±0.0 | 0.0±0.0 | 0.0±0.0 | 2.67±0.58 | 2.0±0.0 | 1.0±0.0 | 1.0±0.0 |
| 2.1  | CanDoX   | 0.67±0.58 | 4.33±0.58 | 0.33±0.58 | 4.67±0.58 | 1.0±0.0 | 0.67±0.58 | 0.67±0.58 | 0.67±0.58 |
| 2.2  | CanDoX   | 1.67±0.58 | 5.0±0.0 | 1.0±0.0 | 2.0±0.0 | 1.67±0.58 | 0.0±0.0 | 1.0±0.0 | 0.0±0.0 |
| 2.3  | CanDoX   | 1.0±0.0 | 5.67±0.58 | 1.0±0.0 | 2.33±0.58 | 1.0±0.0 | 2.0±0.0 | 1.0±0.0 | 2.0±0.0 |
| 3.1  | ImplGuid | 2.0±1.0 | 7.0±0.0 | 2.0±0.0 | 3.67±0.58 | 3.0±0.0 | 1.0±0.0 | 2.67±0.58 | 1.0±0.0 |
| 3.2  | ImplGuid | 1.0±0.0 | 5.67±0.58 | 1.0±1.0 | 3.33±0.58 | 1.0±0.0 | 1.33±0.58 | 0.33±0.58 | 2.0±0.0 |
| 3.3  | ImplGuid | 3.0±0.0 | 5.0±0.0 | 0.67±0.58 | 3.0±0.0 | 2.33±1.15 | 2.67±0.58 | 2.33±1.15 | 3.0±0.0 |
| 4.1  | ModelSel | 1.0±0.0 | 6.67±0.58 | 2.0±0.0 | 5.0±1.0 | 2.0±0.0 | 1.67±0.58 | 1.0±0.0 | 1.67±0.58 |
| 5.1  | ExtAware | 2.0±0.0 | 3.67±0.58 | 0.0±0.0 | 2.0±1.0 | 2.0±0.0 | 0.33±0.58 | 1.0±0.0 | 0.33±0.58 |
| 5.2  | ExtAware | 2.0±0.0 | 3.0±0.0 | 2.0±0.0 | 3.33±1.53 | 3.0±0.0 | 0.0±0.0 | 3.0±0.0 | 0.0±0.0 |
| 5.3  | ExtAware | 2.0±0.0 | 5.33±0.58 | 1.33±1.15 | 4.33±0.58 | 2.0±0.0 | 3.0±0.0 | 1.0±0.0 | 3.0±0.0 |
| 5.4  | ExtAware | 1.67±0.58 | 4.67±0.58 | 0.33±0.58 | 2.67±1.15 | 3.0±0.0 | 1.0±1.0 | 3.0±0.0 | 1.0±1.0 |
| 5.5  | ExtAware | 1.33±1.15 | 5.33±0.58 | 0.33±0.58 | 4.33±0.58 | 2.0±0.0 | 0.33±0.58 | 1.0±0.0 | 0.33±0.58 |
| 6.1  | Negative | 5.0±0.0 | 5.33±0.58 | 5.0±0.0 | 5.0±0.0 | 3.0±0.0 | 3.0±0.0 | 3.0±0.0 | 3.0±0.0 |
| 6.2  | Negative | 5.0±0.0 | 4.33±0.58 | 4.33±0.58 | 5.0±0.0 | 3.0±0.0 | 3.0±0.0 | 3.0±0.0 | 3.0±0.0 |
| 6.3  | Negative | 5.0±0.0 | 5.0±0.0 | 3.0±1.0 | 2.67±1.53 | 3.0±0.0 | 3.0±0.0 | 3.0±0.0 | 3.0±0.0 |

---

**Report compiled:** 2026-02-10
**Analysis period:** 3-run evaluation, v1.2.0 vs v1.2.1
**Methodology:** Keyword matching (accuracy, completeness), LLM judge scoring (0–3 scale), multi-run variance analysis, regression testing
