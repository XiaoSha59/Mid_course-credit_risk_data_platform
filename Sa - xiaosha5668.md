# Grading Report — Sa (xiaosha5668)

**Domain**: Credit Risk (FinVista Consumer Lending)
**Total: 79 / 100**

---

## Section 1 — Understanding the Problem | 9 / 10

- **1.1** (1 mark): 1/1 — correctly identifies the outdated rules-based scorecard as the root cause, notes that the default rate has risen from 7% to 10%, and explains that this caused unexpected credit losses for the bank
- **1.2** (2 marks): 2/2 — correctly identifies this as a binary classification problem, names `default_flag` as the target variable, and explains that each row represents one loan application rather than one customer, so a customer may appear multiple times
- **1.3** (3 marks): 3/3 — well explained: shows that always predicting no-default achieves 90% accuracy but identifies no risk; correctly defines Gini 0 (random ranking) and Gini 1 (perfect separation); explains why the goal is to rank applicants by risk rather than maximise accuracy
- **1.4** (2 marks): 2/2 — correctly identifies Mistake B (approving a defaulter) as more costly to the bank and Mistake A (rejecting a good borrower) as more costly to the customer; correctly explains the 70% approval constraint
- **1.5** (2 marks): 1/2 — correctly explains that the two tables have different levels of detail (one row per transaction vs one row per customer) and states that transactions must be aggregated to one row per customer before joining

**Incorrect:**
- 1.5 only lists aggregation ideas (total amount, count, etc.) but does not mention an important issue: the transactions table covers the full 18-month history for each customer, but loan applications happen at different points in time — only transactions before each application's date should be used, or else future data leaks into the model. This point is expected

---

## Section 2 — Setup & Data Loading | 3 / 3

**Correct:**
- All four tables loaded with correct column parsing (date columns parsed as datetime)
- Commentary correctly notes the row counts and flags the difference: loans has 10,040 rows for 8,000 customers (some customers applied more than once), and transactions has 479,860 records (one row per transaction)

---

## Section 3 — Exploring the Data | 7 / 10

### 3.1 Data Quality Check | 3 / 4

**Correct:**
- Duplicates checked: 41 duplicate loan applications found
- Customer ID check confirmed that all IDs match across tables (no unmatched IDs)
- Default rate noted as ~10%, flagging class imbalance
- Missing value handling plan is thorough and correct for each column — notably, `months_since_deliq` is correctly treated as informative (missing = no delinquency history, not a data error), and the plan to use a placeholder value of 999 (meaning 'never delinquent')

**Incorrect:**
- The commentary does not explicitly state what columns in each of the four tables have missing values and how many — the expectation is a summary table per table (not just the loans table)

### 3.2 Distribution of Each Column | 2 / 3

**Correct:**
- Correctly identifies `annual_income`, `mortgage_balance`, and `txn_amount` as right-skewed with extreme outliers
- Plans 99th-percentile capping for skewed numeric variables
- Notes `credit_score` is roughly bell-shaped and unlikely to need treatment
- Correctly handles `loan_purpose` missing values as a separate "Unknown" category rather than imputing

**Incorrect:**
- `n_delinquencies_24m` has most values at zero (most customers have zero delinquencies) — this is one of the most important observations to make and is missing here. A binary "ever delinquent" flag would be more useful than the raw count, but this insight is absent

### 3.3 Which Columns are Related to Default? | 3 / 5

**Correct:**
- Categorical analysis is strong: correctly identifies `employment_status` as the strongest categorical predictor (unemployed have much higher default rate), followed by `housing_status` and `loan_purpose`
- Business reasoning for each is clear and correct

**Incorrect:**
- The top 3 picks for numeric variables — `credit_score`, `credit_utilization`, and `annual_income` — are reasonable but miss the strongest numeric predictor: `n_delinquencies_24m` (default rate rises from 7.78% for zero delinquencies up to 50% for 4+), a much sharper rise than `annual_income` shows. `loan_to_income_ratio` and `n_inquiries_12m` are also stronger than `annual_income` individually
- `interest_rate` is not flagged as potentially problematic — the bank sets the interest rate based on its own risk view, so including it as a model input risks the model learning the bank's existing decisions rather than finding independent signals

### 3.4 Which Customer Groups are Riskiest? | 2 / 3

**Correct:**
- Correctly identifies unemployed customers (31.6% default rate) as the highest-risk group
- Correctly identifies renters (2,935 loans, 10.9% default rate) as the second group by combination of volume and elevated default rate
- Correctly explains why volume matters: the "Other" housing category has a higher default rate (15.5%) but only 291 loans, so it has less business impact overall

**Further advice:**
- The answer would be stronger if it estimated the money at risk for each group: default rate × number of loans × average loan amount gives a rough pound figure. That would make it easier to compare whether unemployed borrowers or renters represent the bigger financial exposure

---

## Section 4 — Cleaning & Joining | 10 / 12

**Correct:**
- 41 duplicates removed correctly
- `annual_income` capped at the 99th percentile (£192,570) — correct approach
- Transactions aggregated to one row per customer with financially meaningful features
- Missing value strategy is detailed and correct: median-within-group for `annual_income`, median for `months_at_address`, placeholder value 999 (meaning 'never delinquent') for `months_since_deliq`, Unknown category for `loan_purpose`
- All four tables joined correctly; explanation cell covers the final shape of `master` and each decision

**Incorrect:**
- The written explanation describes imputation as the first cleaning step, but the code actually caps `annual_income` before filling missing values — the explanation does not match the order of operations in the code
- Only 2 transaction ratio features created (`loan_to_income_ratio` and `utilization_x_delinquency`) — no richer transaction-derived features appear in the cleaning step

---

## Section 5 — Feature Engineering | 9 / 15

### WoE encoding (9 marks): 6 / 9

**Correct:**
- WoE and IV computed for 3 categorical columns, meeting the minimum requirement: `employment_status` (IV=0.554, very strong), `housing_status` (IV=0.033, weak), `loan_purpose` (IV=0.035, weak)
- Detail tables showing WoE per category are present. Note: the formula used (`ln(p_bad / p_good)`) matches the template, but the standard industry convention is the opposite — `ln(p_good / p_bad)`, where non-defaulters are in the numerator. With the template's formula, a positive WoE means a riskier bin; with the standard convention it would mean a safer bin. Both produce the same IV and the model trains correctly either way, but be aware the sign interpretation differs from most credit risk textbooks. Sorry for my mistake :<.

**Incorrect:**
- No numeric IV computed — binning numeric columns into equal-frequency bins and computing IV for each would have identified the strongest numeric predictors (e.g. `loan_to_income` and `credit_score`). This is missing entirely
- Logistic Regression uses `OneHotEncoder` instead of WoE in the pipeline — the WoE encoding computed here is not actually used in Model 1

**Further advice:**
- Encoding more categorical columns would strengthen the analysis — `education` and `region` are straightforward additions

### New features (6 marks): 3 / 6

**Correct:**
- `loan_to_income_ratio` is a well-known ratio feature
- `utilization_x_delinquency` combines two known risk factors sensibly

**Incorrect:**
- Only 2 ratio features created — useful additions would be a spending acceleration ratio (recent vs previous 3-month spend) and a cash-advance flag. These transaction-based features are entirely absent
- The explanation cell for each feature is short — it names the feature but does not fully explain what a high value signals about the borrower's financial behaviour

---

## Section 6 — Training Models | 10 / 15

### Model 1 — Logistic Regression

**Correct:**
- Pipeline set up with imputation and scaling
- `class_weight="balanced"` used to handle class imbalance
- Written explanation covers why logistic regression was chosen, main limitation (assumes linear relationship)
- Cross-validation Gini reported as 0.5464; validation Gini reported as 0.5668

**Incorrect:**
- Model 1 pipeline uses `OneHotEncoder` rather than the WoE transformer computed in Section 5 — this means the WoE encoding work in Section 5 is not actually applied to the model, which is a significant gap
- No hyperparameter tuning for Model 1 — using `GridSearchCV` over `C`, `penalty`, and `class_weight` can produce a notably better result
- The explanation states that `class_weight="balanced"` was applied but does not explain why Gini (rather than accuracy) is the right metric to use when class weighting is applied

### Model 2 — Random Forest

**Correct:**
- `class_weight="balanced"` used
- Validation Gini 0.6335 is a meaningful improvement over the Logistic Regression baseline (0.5668)
- Explanation covers the key advantage (non-linear patterns and interactions) and the main limitation (interpretability)
- Model correctly selected as the better performer (Gini 0.6335 vs 0.5668)

**Incorrect:**
- No hyperparameter tuning — `n_estimators=100` is the default; searching over depth, sample sizes, and max features typically yields a better Gini score
- The explanation does not mention what hyperparameters were considered or why the defaults were kept
- No third model or ensemble attempted

### Model selection

**Correct:**
- Random Forest correctly chosen based on higher Gini on both cross-validation and validation
- Justification notes that Random Forest scores higher but is harder to explain to a business audience than Logistic Regression

---

## Section 7 — Evaluating Your Model | 6 / 8

**Correct:**
- ROC curve produced
- Score distribution plot shows partial overlap between defaulters and non-defaulters — interpretation is correct (Gini 0.6335, KS 0.4942)
- Feature importance chart produced and key features discussed; notes transaction-based features also contribute
- Written commentary explains why the top features matter: loan-to-income ratio, credit score, and credit utilisation reflect repayment capacity, and transaction-based features (average transaction amount, total spending, cash stress rate) add information about recent spending behaviour

**Incorrect:**
- No calibration plot — a calibration chart checks whether predicted probabilities match what actually happens: for example, applicants given a 20% default probability should default roughly 20% of the time in practice. A Brier score (a single number summarising this accuracy — lower is better) should accompany the chart. This is one of the required evaluation plots and is missing

---

## Section 8 — Business Impact | 9 / 12

**Correct:**
- `expected_loss()` function correctly written
- At 70% approval rate: losses cut from £2,384,267 (approve all) to £470,513 (model policy) — an **80.27% reduction**, well above the 25% target
- Efficiency curve plotted across approval rates 50% to 100% with a vertical line at 70%
- Model saved to `model.pkl` and reload verified with Gini unchanged at 0.6335

**Incorrect:**
- The written explanation is missing — there is no cell explaining what the numbers mean for FinVista or giving a recommendation to the Risk Committee. The only output is a code-generated line: "Recommendation: Deploy the Random Forest model." A written answer should state the EL reduction figure and explain in one or two sentences why the model should be adopted

---

## Section 9 — Drift in Production | 5 / 5

**9.1** (2 marks): 2/2

**Correct:**
- Proposes monitoring key feature distributions (loan-to-income ratio, credit score, annual income) using PSI and histogram comparison
- Mentions predefined thresholds for triggering review

**9.2** (3 marks): 3/3

**Correct:**
- Proposes monthly/quarterly recomputation of ROC AUC, Gini, KS, and calibration using live outcome data
- Correctly identifies target drift as the case where customers in the same predicted-risk band start defaulting at a different rate than expected

---

## Extension | 8 / 10

### Fairness Check (5 marks): 3 / 5

**Correct:**
- Actual code computes approval rates by region at the 70% policy and produces a bar chart showing the breakdown across Southeast, Midwest, Northeast, West, and Southwest — with rates ranging from roughly 64% to 73%
- Commentary correctly notes the differences are modest and do not show strong evidence of systematic regional disadvantage
- Correctly identifies that region can act as a proxy for socioeconomic background (poorer areas may correlate with higher default rates) and flags the regulatory risk — shows genuine awareness of fairness concerns

**Incorrect:**
- The analysis only looks at approval rates — a stronger analysis would also compute the actual default rate per region and compare it to the approval rate to check whether the gap in approvals matches the gap in actual risk. Without showing the actual default rates by region, it is not possible to say whether any differences are justified

### SHAP Explanations (5 marks): 5 / 5

**Correct:**
- Three customers chosen covering the full risk range: high-risk (PD=0.97), medium-risk (PD=0.18), low-risk (PD~0.00)
- SHAP factors described with correct directions for each customer (e.g. unemployment and low income driving up risk for the high-risk customer, high credit score and low loan-to-income ratio driving down risk for the low-risk customer)
- All explanations are intuitive and consistent with credit risk principles

---

## Summary

- Section 1 — Understanding the Problem: 9 / 10
- Section 2 — Setup & Data Loading: 3 / 3
- Section 3 — Exploring the Data: 7 / 10
- Section 4 — Cleaning & Joining: 10 / 12
- Section 5 — Feature Engineering: 9 / 15
- Section 6 — Training Models: 10 / 15
- Section 7 — Evaluating Your Model: 6 / 8
- Section 8 — Business Impact: 9 / 12
- Section 9 — Drift Detection: 5 / 5
- Extension: 8 / 10

**Total: 76 / 100**
