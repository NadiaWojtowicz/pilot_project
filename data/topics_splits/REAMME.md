# ABM Initialization Data: Kandidattest and Newspaper Articles

## Overview

This repository contains empirically grounded data files used to initialize agent opinions
in an agent-based model (ABM) of opinion dynamics. 
The purpose of these files is **not** to model opinion change directly, but to 
**seed agents with realistic initial opinion configurations** that preserve empirical dependencies 
between political issues and sentiments.

Two distinct empirical sources are used:

1. **Kandidattest survey data** (dense, individual-level opinions)
2. **Newspaper articles** (sparse, discursive co-occurrence of issues and sentiments)

These sources can be used:
- as **separate agent populations in separate simulation worlds**, or
- **combined within the same population** to study heterogeneous initialization.

All opinion dynamics after initialization are governed exclusively by the ABM rules (Paul’s model);
empirical structures are used **only at time t = 0**.

---

## Files and Their Roles

### 1. `cleaned_data_no_none .csv`

**Purpose:**  
Preprocessed article-level data serving as the *empirical input* for article-based ABM initialization.

**Structure:**
- Rows: 48 newspaper articles
- Columns:
  - `file`: article identifier
  - `text`: original Danish article text
  - `eng_text`: English translation (used for LLM-based classification)
  - `pred_topic_1` … `pred_topic_10`: binary indicators (0/1) of whether an issue is mentioned

**Key properties:**
- Articles may mention **multiple issues**
- Articles typically mention **1–3 issues**
- No sentiment aggregation or probabilities are stored here

**Role in pipeline:**  
This file is **not used directly in the ABM**. It is a reproducibility and transparency checkpoint
showing how article-level classifications were derived before aggregation into probabilistic structures.

---

### 2. `kandidattest_abm_init_complete.json`

**Purpose:**  
ABM initialization derived from **kandidattest survey data**, where each respondent answers all issue questions.

**Conceptual meaning:**  
Each survey respondent represents a **complete belief vector** across all topics. This allows estimation of:
- marginal probabilities of sentiments per issue
- conditional probabilities between issue–sentiment pairs
- correlations between issues

**Structure (hierarchical JSON):**
- Conditioning state: `(topic_i, sentiment_i)`
- Stored quantities:
  - `marginal_probability`:  
    \[
    P(T_i = s_i)
    \]
  - `conditionals`: full conditional distributions  
    \[
    P(T_j = s_j \mid T_i = s_i)
    \]
  - `correlations`: Pearson correlations between issue–sentiment pairs

**Important note on sentiment space:**  
Neutral sentiment has probability 0 in this dataset. This is **not a modeling choice**, but a direct consequence of the empirical survey responses. As a result, the sentiment space for the kandidattest-based initialization is effectively **binary (positive / negative)**.

**ABM use:**  
Agents are initialized by:
1. Sampling one issue–sentiment pair from the marginal distribution
2. Sequentially sampling remaining issues using the conditional probabilities

This procedure preserves empirical dependencies without freezing correlations dynamically.

---

### 3. `articles_abm_init_complete_smoothed.csv`

**Purpose:**  
ABM initialization derived from **newspaper articles**, capturing how issues and sentiments co-occur in public discourse.

**Conceptual meaning:**  
Articles do **not** represent complete belief vectors. 
Instead, they provide **sparse signals** of which issue–sentiment pairs tend to appear together
in media framing.

**Structure:**
Each row corresponds to a conditional relationship:
- `topic_given`
- `sentiment_given`
- `marginal_probability`:  
  \[
  P(T_i = s_i)
  \]
- `topic_predicted`
- `sentiment_predicted`
- `conditional_probability`
- `conditional_probability_smoothed`
- `n_observations`: number of articles contributing to the estimate
- `smoothed`: indicator that smoothing was applied

---

## Sparsity and Zero Probabilities in Articles

### Why zeros appear

Unlike survey data, articles:
- mention only a subset of issues
- do not cover all issue pairs
- often omit issues entirely

As a result, many conditional probabilities are empirically:
\[
P(T_j = s_j \mid T_i = s_i) = 0
\]
because certain issue–sentiment combinations **never co-occurred** in the same article.

This sparsity is **structural**, not noise.

---

### Why zeros are a problem for ABM initialization

During agent initialization, opinions are sampled sequentially. If all conditional probabilities for a given issue are zero, sampling becomes impossible and the ABM fails.

---

## Smoothing Procedure (Articles Only)

To ensure valid probability distributions for sampling, **Laplace smoothing** is applied.

### Original conditional probability:
\[
P(T_j = s_j \mid T_i = s_i)
= \frac{\text{count}(T_i = s_i \cap T_j = s_j)}
       {\text{count}(T_i = s_i)}
\]

### Smoothed version (α = 0.01):
\[
P(T_j = s_j \mid T_i = s_i)
= \frac{\text{count}(T_i = s_i \cap T_j = s_j) + \alpha}
       {\text{count}(T_i = s_i) + 3\alpha}
\]

(where 3 corresponds to the number of sentiment categories)

**Properties of smoothing:**
- Prevents zero-probability vectors
- Preserves dominance of observed outcomes
- Does not introduce strong artificial correlations
- Ensures probabilities sum to 1

**Files:**
- `articles_abm_init_complete.csv`: unsmoothed (contains zeros; not ABM-safe)
- `articles_abm_init_complete_smoothed.csv`: smoothed (used for ABM)

---

## Correlations vs. Conditional Sampling

- In **kandidattest data**, correlations are meaningful due to complete belief vectors.
- In **article data**, correlations are not estimated; only conditional co-occurrence is used.

In both cases:
- Empirical dependencies are used **only for initialization**
- All persuasion, convergence, polarization, and tie dynamics emerge **endogenously** from the ABM

---

## Simulation Scenarios Supported

1. **Separate worlds**
   - Population A initialized from kandidattest
   - Population B initialized from articles

2. **Combined population**
   - Agents initialized from both sources
   - Heterogeneous empirical grounding

---

## Summary

These files provide:
- empirically grounded initial conditions
- preservation of real-world issue dependencies
- a clean separation between data and dynamics

They are designed to support transparent, reproducible, and theoretically defensible agent-based simulations.
