
# Developer brief

## ⚠️ CRITICAL: Leakage Risk - employment_status

**DO NOT USE employment_status WITHOUT REVIEW.** The data shows:
- "Retired" → 99.7% positive rate (16k samples) - extremely suspicious
- "Unemployed" → 7.8% positive rate (62k samples)
- "Employed" → 89.4% positive rate (451k samples)

This extreme stratification by employment suggests **post-decision status** rather than pre-application feature. If this is filled in *after* loan approval (e.g., after checking employment at first payment), it would leak the outcome.

**Action:** Verify with domain expert that employment_status is captured **at time of application**, not after approval. If post-decision → **DROP immediately**.

## Drop first (definite)
- **id** - unique identifier, no predictive value
- **loan_paid_back** - target column, accidentally in features

## Drop OR keep (pending review)
- **employment_status** - *see critical leakage risk above*
- **marital_status** - minimal signal (0.9% variation) but safe to keep initially

## Numeric preprocessing
- Median imputation (no missing values currently)
- **Log transform for:**
  - `debt_to_income_ratio` - skewness=1.41, correlation=0.336 (strongest signal)
  - `annual_income` - skewness=1.72, correlation=0.006 (weak but improves tree models)
- **No clip needed** - no extreme outliers detected

## Categorical preprocessing
- **One-hot encode (low cardinality):**
  - loan_purpose (8 categories, 0.0462 signal spread)
  - education_level (5 categories, 0.0412 signal spread)
  - gender (3 categories, minimal signal)
  
- **Better encoding for high cardinality:**
  - grade_subgrade (30 categories, 0.353 signal spread) → **Use frequency or ordinal encoding** instead of one-hot to preserve ranking signal

## Feature priority for modeling
1. **MUST INCLUDE:** debt_to_income_ratio, credit_score (only features with correlation > 0.2)
2. **TRY EARLY:** grade_subgrade (highest categorical signal), interest_rate
3. **LOW IMPACT:** loan_purpose, education_level, gender, annual_income, loan_amount
4. **EXCLUDE:** id, loan_paid_back, and employment_status (pending leakage review)

## Validation
- **Stratified k-fold** (heavily imbalanced: 79.88% positive)
- **Metric: ROC-AUC** (competition metric)
- Train/test shift: **None detected** - all features stable across train/test
- No unseen categories expected

## Key risks & mitigations
| Risk | Mitigation |
|------|-----------|
| employment_status leakage | Domain expert review REQUIRED before use |
| Class imbalance (79.88% positive) | Stratified validation, ROC-AUC metric, consider SMOTE if needed |
| Weak features diluting signal | Start with debt_to_income_ratio + credit_score, add others incrementally |
| High cardinality in grade_subgrade | Use frequency encoding or target encoding instead of one-hot |

## Modeling steps
1. **Baseline:** Logistic regression with debt_to_income_ratio + credit_score (log-transformed)
2. **Add features:** grade_subgrade (frequency encoded)
3. **Upgrade model:** LightGBM or XGBoost for non-linear relationships
4. **Feature engineering:** Consider interactions (e.g., income × debt_ratio)
5. **Before submission:** Verify all preprocessing is identical for test set, no data leakage
