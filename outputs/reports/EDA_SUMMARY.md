
# EDA Summary

## Dataset overview
- Train shape: (593,994, 13)
- Test shape: (254,569, 12)
- Target column: loan_paid_back (binary, 79.88% positive)
- ID column: id
- Number of numeric features: 5
- Number of categorical features: 6
- Number of other features: 1
- sample_submission matches test ids: True

## Data quality
- Train duplicate rows: 0 ✓
- Test duplicate rows: 0 ✓
- Train duplicate IDs: 0 ✓
- Test duplicate IDs: 0 ✓
- Columns with missing values in train: 0 ✓
- Columns with missing values in test: 0 ✓
- Strange text value columns: 0 ✓
- Category formatting issue columns: 0 ✓

**Summary:** Dataset is clean with no missing values or duplicates. Quality is excellent for modeling.

## Target analysis
- Imbalance label: **heavily imbalanced** (79.88% positive class)
- Majority-class baseline: 0.7988 (beat this with ROC-AUC > 0.5)
- Recommended validation: **Stratified k-fold** (preserves class ratio in folds)
- Metric focus: **ROC-AUC** (use predict_proba, not binary predictions)

**Implication:** Class imbalance is significant but manageable. Accuracy is a poor metric. ROC-AUC is appropriate.

## Feature insights

### Numeric features (ranked by predictive power)
1. **debt_to_income_ratio** - STRONGEST signal (correlation=0.336, separation=0.837)
   - Right-skewed (skew=1.41), needs log transform
   - Clear separation between positive/negative classes
   
2. **credit_score** - MODERATE signal (correlation=0.235, separation=0.585)
   - Well-distributed, no outliers
   - Class means differ by 32 points (668 vs 687)
   
3. **interest_rate** - WEAK signal (correlation=0.131, separation=0.327)
   - Normal distribution
   - Class means differ slightly (12.88% vs 12.22%)
   
4. **annual_income** - VERY WEAK (correlation=0.006, separation=0.016)
   - Wide range ($6k-$393k), right-skewed
   - Class means essentially identical
   
5. **loan_amount** - NEGLIGIBLE (correlation=0.004, separation=0.009)
   - No predictive value despite high cardinality

### Categorical features (ranked by predictive power)
1. **grade_subgrade** - STRONGEST (target_rate_spread=0.353, 30 categories)
   - Clear ranking signal across grades
   - Recommend frequency or ordinal encoding (not one-hot)
   
2. **employment_status** - ⚠️ SUSPICIOUS (target_rate_spread=0.920)
   - Retired: 99.7%, Unemployed: 7.8%, Employed: 89.4%
   - Unusually high predictive power → LIKELY LEAKAGE
   - See leakage_review.csv for details
   
3. **loan_purpose** - VERY WEAK (target_rate_spread=0.046)
   - Dominated by "Debt consolidation" (54.7%)
   
4. **education_level** - VERY WEAK (target_rate_spread=0.041)
   - Almost no signal across categories
   
5. **marital_status** - NEGLIGIBLE (target_rate_spread=0.009)
   - Single/Married have identical rates (79.88%)
   
6. **gender** - NEGLIGIBLE (target_rate_spread=0.006)
   - Female/Male essentially identical rates

## Train vs test comparison
- Strongly shifted columns: **None** ✓
- Slightly shifted columns: **None** ✓
- All features show stable distributions (KS p-value > 0.15)

**Implication:** No train/test drift detected. Preprocessing pipeline can be identical for train/test.

## Leakage risk - CRITICAL REVIEW NEEDED

### HIGH RISK: employment_status
- "Retired" category: 99.7% positive (16,453 samples)
- Other categories: 89-90% for employed, 7.8% for unemployed
- **Suspicion:** This appears to be employment status *after* loan approval, not at application
- **Action Required:** Verify with domain expert. If post-decision → DROP IMMEDIATELY

### MEDIUM RISK: loan_paid_back
- This is the TARGET VARIABLE accidentally included in features
- **Action:** DROP immediately before training

### LOW RISK: marital_status
- Minimal signal (0.9% spread)
- Safe to keep unless employment_status decision changes overall strategy

### SAFE: All other features
- No suspicious patterns detected

## Highly correlated pairs
- No pairs with correlation > 0.8 detected ✓
- No multicollinearity issues expected

## Recommendations for modeling

### Feature selection (final)
**MUST DROP:**
- id (unique, no signal)
- loan_paid_back (target)

**CONDITIONAL DROP (if employment_status is leakage):**
- employment_status
- marital_status (optional, minimal signal anyway)

**CORE FEATURES (use first):**
- debt_to_income_ratio (log-transformed)
- credit_score
- grade_subgrade (frequency encoded)

**OPTIONAL (add if needed):**
- interest_rate, education_level, loan_purpose, gender
- annual_income, loan_amount (very weak signals)

### Preprocessing pipeline
1. **Drop:** id, loan_paid_back
2. **Handle:** All numeric columns need median imputation (even though no missing detected, for robustness)
3. **Transform numeric:**
   - Log transform: debt_to_income_ratio, annual_income
   - No transforms needed for: credit_score, interest_rate, loan_amount
4. **Encode categorical:**
   - Frequency encoding: grade_subgrade (preserves ranking signal)
   - One-hot encode: loan_purpose, education_level, gender (low cardinality)
5. **No outlier clipping needed** (no extreme outliers detected)

### Validation strategy
- **Method:** Stratified k-fold (5-10 folds)
- **Metric:** ROC-AUC (primary), precision/recall for class-specific analysis
- **No cross-validation data leakage:** Preprocessing fitted only on training fold
- **Test handling:** Apply identical preprocessing, handle unseen categories gracefully

### Model recommendations
1. **Baseline:** Logistic regression with core features (debt_to_income_ratio, credit_score)
2. **Strong models:** LightGBM, XGBoost, CatBoost
3. **Ensemble:** Stack multiple models for final submission
4. **Feature engineering:** Consider interactions (income × debt_ratio)

## Next steps for developer
1. **URGENT:** Review employment_status with domain expert for leakage before building any models
2. **Immediate:** Drop id and loan_paid_back before preprocessing
3. **Build pipeline:** Implement preprocessing to be identical for train/test
4. **Stratified CV:** Use stratified k-fold with ROC-AUC evaluation
5. **Feature validation:** Check that no data leaks through target-dependent encoding
6. **Start simple:** Baseline with 2-3 strongest features, add others incrementally
7. **Watch:** No train/test shift detected, but verify preprocessing handles edge cases safely
