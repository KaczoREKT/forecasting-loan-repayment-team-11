# Preprocessing recommendations

## ⚠️ CRITICAL: Address leakage risk first

### employment_status
- **Issue:** Suspiciously high predictive power (Retired: 99.7%, Unemployed: 7.8%)
- **Risk:** Likely post-decision status, not pre-application feature
- **Decision:** Verify with domain expert. If post-approval → DROP
- **Until clarified:** DO NOT INCLUDE in models

### loan_paid_back
- This IS the target variable
- **Action:** DROP IMMEDIATELY

---

## Drop (definite)
- **id** - unique identifier, no signal
- **loan_paid_back** - target column

## Drop or Keep (conditional)
- **employment_status** - pending leakage review (see above)
- **marital_status** - minimal signal (keep initially, consider dropping if needed)

---

## Numeric imputation
- **Method:** Median imputation (robust to skewed distributions)
- **Apply to:** annual_income, debt_to_income_ratio, credit_score, loan_amount, interest_rate
- **Note:** No missing values currently, but imputation pipelines should handle them for robustness

### Numeric transformation
- **Log transform (y = log(x + 1)):**
  - `debt_to_income_ratio` - skewness=1.41, strongest signal, needs normalization
  - `annual_income` - skewness=1.72, wide range, improves tree models
  
- **No transform needed:**
  - `credit_score` - normal distribution around 682
  - `interest_rate` - normal distribution around 12.4%
  - `loan_amount` - nearly uniform, no skewness

---

## Categorical imputation
- **Method:** Create 'Missing' category (for future-proofing if test has missing values)
- **Apply to:** gender, education_level, loan_purpose, grade_subgrade

**Note:** Currently zero missing values in training data, but test may differ.

---

## Categorical encoding

### One-hot encoding (low cardinality features)
- **loan_purpose** (8 categories)
  - Dominated by "Debt consolidation" (54.7%)
  - One-hot is safe and interpretable
  
- **education_level** (5 categories)
  - Balanced categories, no ordinal relationship
  - One-hot is standard approach
  
- **gender** (3 categories)
  - Binary + Other category
  - One-hot works well for tree models

### Frequency or target encoding (high cardinality)
- **grade_subgrade** (30 categories)
  - Likely ordinal structure (C < D < E, 1 < 2 < 3 < 4 < 5)
  - **Recommended:** Ordinal encoding if grade hierarchy applies
  - **Alternative:** Frequency encoding to reduce dimensionality vs. one-hot (30 features)
  - **Why not one-hot:** Creates too many features, one-hot doesn't preserve grade ordering
  - **Do NOT one-hot:** Would create 29 additional features with minimal gain

---

## Transform summary

### Numeric scaling
- **Optional:** StandardScaler for linear models (logistic regression, SVM)
- **Not needed:** Tree-based models (LightGBM, XGBoost, Random Forest) don't require scaling

### Outlier clipping
- **Needed?** No - no extreme outliers detected
- **annual_income:** Range $6k-$393k is wide but not extreme; log transform handles this
- **All other features:** Within reasonable bounds

---

## Missing value handling - edge cases
- **If test set has missing values:**
  - Apply median from training fold (computed on train only, not test)
  - Apply same 'Missing' category label for categorical
  - Document test missing rate in model report
  
- **If test has new categories:**
  - Frequency encoding: assign frequency=0 or small value
  - One-hot: create 'Unknown' bucket or drop feature

---

## Validation strategy

### Preprocessing integrity
1. **Fit on training fold ONLY** - never on test/validation
2. **Apply to validation/test** - use fitted transformers
3. **Handle unseen categories:**
   - Assign rare category frequency for frequency encoding
   - Use 'Unknown' bucket for one-hot encoding

### Data leakage prevention
- **DO NOT:** Use target for encoding (target encoding without proper CV)
- **DO NOT:** Fit preprocessing on full train+test combined
- **DO:** Fit separate preprocessing in each k-fold iteration

---

## Implementation order

1. **Remove leakage:**
   - Drop id, loan_paid_back
   - Verify employment_status is safe (domain review)

2. **Handle missing (imputation):**
   - Numeric: median imputation
   - Categorical: 'Missing' label

3. **Transform numeric:**
   - Log transform debt_to_income_ratio, annual_income

4. **Scale (optional for tree models):**
   - StandardScaler if using logistic regression
   - Skip if using LightGBM/XGBoost

5. **Encode categorical:**
   - grade_subgrade: frequency encoding
   - Others: one-hot encoding

6. **Feature selection (if needed):**
   - Start with core features (debt_to_income_ratio, credit_score, grade_subgrade)
   - Add others incrementally, evaluate ROC-AUC
   - Drop low-signal features (loan_purpose, education_level, gender)

---

## Validation plan

### Cross-validation setup
- **Method:** Stratified k-fold (k=5 or 10)
- **Reason:** Preserves 79.88% positive class ratio in each fold
- **Metric:** ROC-AUC (primary), log loss, precision-recall

### Train/test prediction
- **Train:** Use predict_proba() for ROC-AUC calculation
- **Test:** Same preprocessing pipeline as train
- **Edge cases:** Handle unseen categories, new missing patterns

### Monitoring
- **Train vs validation:** Watch for overfitting
- **Train vs test:** No drift expected (verified in EDA)
- **Class balance:** Verify stratification is maintained
