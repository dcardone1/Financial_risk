------------------------------------------------------------------------

editor_options: markdown: wrap: 72 ---

# Credit Risk Analytics --- Data Dictionary

## Project objective

This project represents a financial institution evaluating credit applications and monitoring the subsequent performance of approved loans.

The dataset contains both **pre-application information** used for credit evaluation and **post-origination performance information** used to study repayment behavior and default dynamics.

The main analytical questions are:

1.  Which customer, financial, credit-history and application characteristics are associated with approval?
2.  Which characteristics available at application time are associated with subsequent default?
3.  How does repayment behavior evolve month by month?
4.  Which factors are associated with earlier default?

## Data model

The database is organized into ten relational tables:

- `customers` --- customer master information.
- `financial_profile` --- repeated financial snapshots for each customer.
- `previous_loans` --- historical loans held before the current application.
- `payment_history` --- payment-level history for previous loans.
- `credit_history` --- repeated credit-history snapshots for each customer.
- `loan_applications` --- submitted credit applications and application-time characteristics.
- `loan_decisions` --- underwriting decision and risk-related application-level variables.
- `new_loans` --- loans originated from approved applications.
- `new_payment_history` --- monthly repayment performance for newly originated loans.
- `new_loan_outcomes` --- loan-level summary of subsequent performance and default.

The **analytical unit for underwriting/default modeling is the loan application or originated loan**, depending on the question. Transaction-level tables must be aggregated to the appropriate analytical level before modeling.

## Temporal principle

Variables used to predict credit risk at application time should only use information available **on or before `application_date`**.

Post-origination variables in `new_payment_history` and `new_loan_outcomes` describe subsequent loan performance and must not be used as predictors of the original default event.

------------------------------------------------------------------------

## 1. `customers`

Master table containing one record per customer.

| Column | Type | Key | NULL | Description |
|:---|:---|:---|:---|:---|
| `customer_id` | INTEGER | PK | No | Unique customer identifier. |
| `birth_date` | DATE |  | No | Customer date of birth. Can be used to derive age at application. |
| `gender` | TEXT |  | Yes | Customer gender category. |
| `education` | TEXT |  | Yes | Education level. |
| `marital_status` | TEXT |  | Yes | Marital status. |
| `dependents` | INTEGER |  | Yes | Number of dependents. |

**Relationship:** `customers` 1:N with `financial_profile`, `credit_history`, `previous_loans` and `loan_applications`.

------------------------------------------------------------------------

## 2. `financial_profile`

Time-varying financial information. Each customer has **12 monthly profile records**, identified by `financial_profile_id`.

| Column | Type | Key | NULL | Description |
|:---|:---|:---|:---|:---|
| `financial_profile_id` | INTEGER | PK | No | Unique financial-profile record identifier. |
| `customer_id` | INTEGER | FK | No | References `customers.customer_id`. |
| `profile_date` | DATE |  | No | Date associated with the financial snapshot. |
| `employment_status` | TEXT |  | Yes | Employment status at the snapshot date. |
| `employment_years` | REAL |  | Yes | Years employed at the snapshot date. |
| `monthly_income` | REAL |  | Yes | Estimated monthly income. |
| `monthly_expenses` | REAL |  | Yes | Estimated monthly expenses. |
| `housing_status` | TEXT |  | Yes | Housing situation. |

Potential derived analytical variables include disposable income, expense ratio and income stability.

------------------------------------------------------------------------

## 3. `previous_loans`

Historical loans held by customers before the application being analyzed.

| Column         | Type    | Key | NULL | Description                             |
|:---------------|:--------|:----|:-----|:----------------------------------------|
| `loan_id`      | INTEGER | PK  | No   | Unique identifier for a previous loan.  |
| `customer_id`  | INTEGER | FK  | No   | References `customers.customer_id`.     |
| `loan_date`    | DATE    |     | No   | Date the historical loan was issued.    |
| `loan_amount`  | REAL    |     | No   | Original amount of the historical loan. |
| `term_months`  | INTEGER |     | No   | Original loan term in months.           |
| `loan_status`  | TEXT    |     | No   | Status of the historical loan.          |
| `default_date` | DATE    |     | Yes  | Date of default, when applicable.       |

Historical loan variables should be aggregated using only loans that were known before the relevant application date.

------------------------------------------------------------------------

## 4. `payment_history`

Payment-level history associated with `previous_loans`.

| Column           | Type    | Key | NULL | Description                          |
|:-----------------|:--------|:----|:-----|:-------------------------------------|
| `payment_id`     | INTEGER | PK  | No   | Unique payment identifier.           |
| `loan_id`        | INTEGER | FK  | No   | References `previous_loans.loan_id`. |
| `payment_date`   | DATE    |     | No   | Payment date.                        |
| `payment_amount` | REAL    |     | Yes  | Amount recorded for the payment.     |
| `days_late`      | INTEGER |     | Yes  | Number of days the payment was late. |

Potential derived variables include average days late, maximum days late, late-payment ratio and payment completion ratio.

------------------------------------------------------------------------

## 5. `credit_history`

Monthly credit-history snapshots for customers.

| Column | Type | Key | NULL | Description |
|:---|:---|:---|:---|:---|
| `credit_history_id` | INTEGER | PK | No | Unique credit-history snapshot identifier. |
| `customer_id` | INTEGER | FK | No | References `customers.customer_id`. |
| `history_date` | DATE |  | No | Date of the credit-history snapshot. |
| `credit_score` | INTEGER |  | Yes | Credit score at the snapshot date. |
| `previous_loans` | INTEGER |  | Yes | Number of previous loans. |
| `previous_defaults` | INTEGER |  | Yes | Number of previous defaults. |
| `late_payments` | INTEGER |  | Yes | Number of historical late payments. |
| `credit_history_years` | REAL |  | Yes | Length of the customer's credit history in years. |
| `avg_days_late` | REAL |  | Yes | Average days late in the available historical payment information. |
| `first_loan_date` | DATE |  | Yes | Date of the customer's first recorded loan. |
| `last_loan_date` | DATE |  | Yes | Date of the customer's most recent recorded loan. |

These are snapshot variables; when joining them to applications, the relevant snapshot must be selected according to the application date.

------------------------------------------------------------------------

## 6. `loan_applications`

Credit applications submitted by customers. This table represents the application-stage analytical population.

| Column | Type | Key | NULL | Description |
|:---|:---|:---|:---|:---|
| `application_id` | INTEGER | PK | No | Unique application identifier. |
| `customer_id` | INTEGER | FK | No | References `customers.customer_id`. |
| `application_date` | DATE |  | No | Date the credit application was submitted. |
| `requested_amount` | REAL |  | No | Amount requested by the customer. |
| `term_months` | INTEGER |  | No | Requested loan term in months. |
| `purpose` | TEXT |  | Yes | Stated purpose of the requested credit. |
| `credit_score` | INTEGER |  | Yes | Credit score available at application. |
| `application_interest_rate` | REAL |  | Yes | Interest rate associated with the application. |
| `monthly_income` | REAL |  | Yes | Monthly income available at application. |
| `monthly_expenses` | REAL |  | Yes | Monthly expenses available at application. |
| `employment_years` | REAL |  | Yes | Employment tenure available at application. |
| `avg_days_late` | REAL |  | Yes | Historical average payment delay available at application. |
| `monthly_application_rate` | REAL |  | Yes | Application-related rate/measure recorded at application time. |
| `estimated_monthly_payment` | REAL |  | Yes | Estimated monthly payment for the requested credit. |
| `debt_to_income` | REAL |  | Yes | Debt-to-income ratio associated with the application. |

`requested_amount`, `term_months`, customer characteristics and financial/credit-history information are candidate predictors. Variables that reflect a decision already made should be treated carefully to avoid target leakage.

------------------------------------------------------------------------

## 7. `loan_decisions`

Underwriting result and application-level risk information associated with each application.

| Column | Type | Key | NULL | Description |
|:---|:---|:---|:---|:---|
| `application_id` | INTEGER | PK/FK | No | References `loan_applications.application_id`. One decision record per application. |
| `decision` | TEXT |  | No | Underwriting decision, such as approved or rejected. |
| `approval_probability` | REAL |  | Yes | Probability-like score associated with approval. |
| `credit_score` | INTEGER |  | Yes | Credit score used/available for the decision. |
| `previous_defaults` | INTEGER |  | Yes | Number of previous defaults available to the decision process. |
| `late_payment_ratio` | REAL |  | Yes | Historical proportion of payments that were late. |
| `default_ratio` | REAL |  | Yes | Historical default proportion associated with the customer. |
| `debt_to_income` | REAL |  | Yes | Debt-to-income ratio used in the decision context. |
| `requested_amount` | REAL |  | Yes | Requested amount associated with the application. |
| `monthly_income` | REAL |  | Yes | Monthly income associated with the application. |

`loan_decisions` is linked 1:1 to `loan_applications`.

For a predictive model of approval, `decision` is the natural outcome. For a default model, decision-related fields should be evaluated carefully because some may be downstream of the underwriting process.

------------------------------------------------------------------------

## 8. `new_loans`

Loans actually originated from approved applications.

| Column | Type | Key | NULL | Description |
|:---|:---|:---|:---|:---|
| `loan_id` | INTEGER | PK | No | Unique identifier for the newly originated loan. |
| `customer_id` | INTEGER | FK | No | References `customers.customer_id`. |
| `application_id` | INTEGER | FK | No | References `loan_applications.application_id`. |
| `loan_date` | DATE |  | No | Origination date of the loan. |
| `loan_amount` | REAL |  | No | Amount actually originated. |
| `term_months` | INTEGER |  | No | Loan term in months. |
| `annual_interest_rate` | REAL |  | No | Annual interest rate applied to the loan. |
| `loan_status` | TEXT |  | No | Current final/status classification in the generated data. |
| `default_flag` | INTEGER |  | No | Binary indicator: `1` = default, `0` = no default. |
| `default_date` | DATE |  | Yes | Date on which the loan defaulted, when applicable. |

The generated dataset contains 19,841 originated loans, of which 5,532 defaulted.

------------------------------------------------------------------------

## 9. `new_payment_history`

Monthly payment-level performance for newly originated loans.

| Column | Type | Key | NULL | Description |
|:---|:---|:---|:---|:---|
| `loan_id` | INTEGER | FK | No | References `new_loans.loan_id`. |
| `customer_id` | INTEGER | FK | No | Customer associated with the loan. |
| `payment_number` | INTEGER | Composite key\* | No | Sequential payment/month number for the loan. |
| `scheduled_payment_date` | DATE |  | No | Scheduled payment date. |
| `actual_payment_date` | DATE |  | Yes | Actual payment date. Missing when no payment was made, including default records. |
| `scheduled_payment_amount` | REAL |  | No | Scheduled installment amount. |
| `actual_payment_amount` | REAL |  | No | Actual amount paid. |
| `interest_amount` | REAL |  | No | Interest component of the scheduled payment. |
| `principal_amount` | REAL |  | No | Principal component of the scheduled payment. |
| `ending_balance` | REAL |  | No | Outstanding balance after the payment event. |
| `days_late` | INTEGER |  | Yes | Number of days late. Missing for default-event rows. |
| `payment_status` | TEXT |  | No | Payment classification: `On time`, `Late`, `Severely late`, or `Defaulted`. |

\* The generated data validate uniqueness of the combination `loan_id + payment_number`.

This table is strictly **post-origination performance information** and should not be used as an original application-time predictor.

------------------------------------------------------------------------

## 10. `new_loan_outcomes`

Loan-level summary of subsequent repayment performance.

| Column | Type | Key | NULL | Description |
|:---|:---|:---|:---|:---|
| `loan_id` | INTEGER | PK/FK | No | References `new_loans.loan_id`. |
| `customer_id` | INTEGER | FK | No | Customer associated with the loan. |
| `application_id` | INTEGER | FK | No | Original application associated with the loan. |
| `loan_date` | DATE |  | No | Loan origination date. |
| `loan_amount` | REAL |  | No | Originated loan amount. |
| `term_months` | INTEGER |  | No | Loan term in months. |
| `default_flag` | INTEGER |  | No | Binary outcome: `1` = default, `0` = no default. |
| `default_date` | DATE |  | Yes | Default date, when applicable. |
| `months_to_default` | INTEGER/REAL |  | Yes | Number of months from origination to default, when applicable. |
| `first_late_payment_date` | DATE |  | Yes | Date of first late payment, when applicable. |
| `last_actual_payment_date` | DATE |  | Yes | Date of the last recorded actual payment. |
| `total_late_payments` | INTEGER |  | No | Total number of late-payment events. |
| `total_days_late` | INTEGER |  | No | Total accumulated days late. |
| `payment_completion_ratio` | REAL |  | Yes | Proportion of scheduled payment obligations completed. |
| `late_payment_ratio` | REAL |  | Yes | Proportion of payment records classified as late. |
| `recovery_ratio` | REAL |  | Yes | Recovery-related ratio associated with the loan outcome. |

`new_loan_outcomes` provides loan-level variables useful for portfolio-performance and time-to-default analysis. Variables such as `months_to_default` are outcome variables and should not be used to predict the same default event.

------------------------------------------------------------------------

## Relationships

``` text
customers
 ├── 1:N ── financial_profile
 ├── 1:N ── credit_history
 ├── 1:N ── previous_loans
 │            └── 1:N ── payment_history
 └── 1:N ── loan_applications
                 ├── 1:1 ── loan_decisions
                 └── 0:1 ── new_loans
                              ├── 1:N ── new_payment_history
                              └── 1:1 ── new_loan_outcomes
```

## Important analytical distinctions

### Application-time predictors

Candidate predictors for an original credit-risk model include information from:

- `customers`
- the appropriate `financial_profile` snapshot
- the appropriate `credit_history` snapshot
- historical `previous_loans`
- historical `payment_history`
- `loan_applications`

These variables must be aligned temporally to the application date.

### Post-origination performance

The following tables describe what happened after a loan was granted:

- `new_loans`
- `new_payment_history`
- `new_loan_outcomes`

They are appropriate for portfolio monitoring, repayment behavior analysis and time-to-default/hazard analysis, but they contain information that occurs after origination.

## Data quality characteristics

The dataset intentionally contains missing values and realistic imperfections. Missing values should be handled analytically rather than automatically removed.

The completed global validation found no structural consistency failures. The generated dataset contains:

- 20,000 customers.
- 240,000 financial-profile records.
- 16,637 previous loans.
- 386,370 historical payment records.
- 240,000 credit-history snapshots.
- 28,688 loan applications.
- 28,688 loan decisions.
- 19,841 newly originated loans.
- 589,774 new-loan payment records.
- 19,841 loan outcomes.

The observed default rate among newly originated loans is approximately **27.88%**.

`existing_monthly_debt` and related operational quantities used during generation should be interpreted as **approximations**, not audited financial-accounting measures.

## Modeling guidance

The dataset supports at least two complementary analytical paths:

1.  **Default prediction at origination**
    - Unit: originated loan/application.
    - Predictors: information available at application time.
    - Outcome: `default_flag`.
    - Candidate method: logistic regression, with possible tree-based models as a comparison.
2.  **Time-to-default / hazard analysis**
    - Unit: originated loan.
    - Time variable: months from `loan_date` to default or censoring.
    - Event: default.
    - Supporting data: `new_payment_history` and `new_loan_outcomes`.
    - Candidate methods: Kaplan--Meier curves and Cox proportional-hazards modeling.

The two analyses should remain conceptually separate to avoid leakage between pre-origination prediction and post-origination monitoring.
