# Radiomics Feature Screen Pipeline
A complete radiomics feature selection pipeline for binary classification tasks, integrating **Mann–Whitney U test**, **Spearman correlation analysis**, and **mRMR (minimum Redundancy Maximum Relevance)** to progressively reduce dimensionality and retain the most informative features.

This pipeline is designed for structured radiomics datasets and is especially suitable for studies in medical imaging, where the number of handcrafted features is usually much larger than the sample size.

---

## Overview

High-dimensional radiomics features often contain a large number of irrelevant, unstable, or redundant variables. Directly using all extracted features for model building may lead to:

- overfitting,
- unstable model performance,
- poor interpretability,
- inflated computational cost.

To address these issues, this project implements a **three-stage feature selection workflow**:

1. **Mann–Whitney U test** for preliminary univariate filtering,
2. **Spearman correlation analysis** for redundancy removal,
3. **mRMR** for final multivariate feature ranking and selection.

The final goal is to obtain a compact subset of robust and informative radiomics features for downstream modeling.

---

## Workflow

The full pipeline is:

**Load data → Missing value handling → Mann–Whitney U filtering → Spearman redundancy reduction → mRMR selection → Export final dataset**

---

## Selection Strategy

### Step 1. Data Loading and Preprocessing

The script reads the input CSV file and assumes the dataset has the following structure:

- **Column 1**: patient ID or case identifier
- **Column 2**: binary class label
- **Column 3 onward**: radiomics features

During preprocessing:

- the label column is converted to numeric values,
- all feature columns are converted to numeric values,
- invalid values are coerced to `NaN`,
- missing values in the label column are filled with the **mean**,
- missing values in feature columns are filled with the **column mean**.

This step ensures that the subsequent statistical analysis can run without interruption due to non-numeric entries or missing values.

---

### Step 2. Mann–Whitney U Test

The first selection stage uses the **Mann–Whitney U test** to identify features that show statistically significant differences between the two classes.

#### Rationale

Radiomics features are often non-normally distributed. Compared with a t-test, the Mann–Whitney U test is a nonparametric alternative that is more appropriate when distributional assumptions are not guaranteed.

#### Procedure

For each feature:

- samples from class 0 and class 1 are extracted,
- a two-sided Mann–Whitney U test is performed,
- the corresponding `p` value is recorded.

Only features with:

```text
p < 0.05
````

are retained for the next step.

#### Purpose

This stage removes features that are unlikely to be associated with the outcome and serves as an initial statistical screening step.

#### Output

* `01_MWU_p_values.csv`
  Contains the `p` values for all original features.

* `02_MWU_significant_features.csv`
  Contains only the statistically significant features retained after Mann–Whitney U filtering.

---

### Step 3. Spearman Correlation Analysis

After univariate filtering, many retained features may still be highly correlated with one another. To address feature redundancy, the second stage applies **Spearman correlation analysis**.

#### Rationale

Highly correlated radiomics features often carry overlapping information. Retaining all of them may:

* increase collinearity,
* reduce model stability,
* make the model less interpretable.

Spearman correlation is used because it is rank-based and more robust for non-normal data.

#### Procedure

* the absolute Spearman correlation coefficient is calculated for every pair of retained features,
* if the absolute correlation exceeds the predefined threshold:

```text
|ρ| > 0.90
```

the pair is considered redundant,

* one feature is removed from the pair.

#### Which feature is removed?

When two features are highly correlated, the pipeline keeps the one with the **smaller Mann–Whitney U test p-value**, because that feature showed stronger univariate discriminative ability in the previous step.

#### Purpose

This stage reduces redundancy and preserves a more diverse and compact feature space before multivariate feature selection.

#### Output

* `03_Spearman_correlation_matrix.csv`
  Full absolute Spearman correlation matrix of retained features.

* `04_Spearman_dropped_features.csv`
  Detailed record of which redundant features were removed and which were kept.

* `05_Features_after_Spearman.csv`
  Feature matrix after redundancy removal.

---

### Step 4. mRMR Feature Selection

The final selection stage applies **mRMR (minimum Redundancy Maximum Relevance)** to rank and select the most informative subset of features.

#### Rationale

Even after significance filtering and redundancy reduction, there may still be too many features for robust model development. mRMR aims to select features that are:

* highly relevant to the outcome,
* minimally redundant with each other.

This makes mRMR especially suitable for radiomics and other high-dimensional biomedical datasets.

#### Implementation Details

This project implements a **self-contained mRMR procedure without relying on `pymrmr`**.

The process includes:

1. **Median imputation** for remaining missing values,
2. **Quantile discretization** of continuous features into bins,
3. **Mutual information calculation**:

   * relevance: mutual information between each feature and the label,
   * redundancy: average mutual information between a candidate feature and already selected features,
4. **Greedy forward selection** until the desired number of features is reached.

#### Supported Criteria

Two mRMR criteria are supported:

* **MID**:
  `score = relevance - redundancy`

* **MIQ**:
  `score = relevance / (redundancy + eps)`

The default setting is:

```text
MID
```

#### Final Number of Features

The pipeline selects the top:

```text
TOP_K = 50
```

features by default, or fewer if the number of candidate features is smaller.

#### Purpose

This stage produces the final compact feature subset for downstream predictive modeling.

#### Output

* `06_mRMR_relevance.csv`
  Mutual information relevance of each feature to the label.

* `07_mRMR_selected_features.txt`
  Final selected feature names.

* `08_mRMR_selection_process.csv`
  Step-by-step mRMR selection record, including relevance, redundancy, and final score.

* `09_mRMR_selected_feature_values.csv`
  Feature matrix containing only the final mRMR-selected features.

---

### Step 5. Final Dataset Export

After mRMR selection, the script generates the final modeling dataset by concatenating:

* the first two original columns,
* the final selected feature subset.

#### Output

* `10_Final_selected_dataset.csv`
  Final dataset for downstream machine learning or statistical modeling.

* `11_feature_count_summary.csv`
  Summary table showing the number of features remaining after each selection stage.

---

## Why This Pipeline Uses Three Selection Stages

This pipeline is designed to balance **statistical significance**, **redundancy control**, and **multivariate relevance**.

### Mann–Whitney U test

Removes features that do not show significant distributional differences between groups.

### Spearman correlation

Removes highly correlated features that may introduce redundancy and instability.

### mRMR

Selects a final subset that maximizes relevance to the label while minimizing overlap between selected features.

Together, these three stages provide a more robust and interpretable feature selection framework than using any single method alone.

---

## Input Requirements

The input file must be a CSV file.

Recommended structure:

| Column   | Description          |
| -------- | -------------------- |
| 1        | Patient ID / Case ID |
| 2        | Binary label         |
| 3 onward | Radiomics features   |

Example:

```text
ID,Label,feature_1,feature_2,feature_3,...
001,0,0.123,4.567,8.910,...
002,1,0.234,3.456,7.890,...
```

### Important Notes

* The label column must represent a **binary classification task**.
* All feature columns should be numeric or convertible to numeric.
* Non-numeric values will be coerced to missing values and imputed automatically.
* If your dataset structure is different, modify the following parameters in the script:

```python
LABEL_COL_INDEX = 1
FEATURE_START_COL_INDEX = 2
```

For example, if the first column is already the label and there is no ID column, you may use:

```python
LABEL_COL_INDEX = 0
FEATURE_START_COL_INDEX = 1
```

---

## Default Parameters

```python
MWU_P_THRESHOLD = 0.05
SPEARMAN_THRESHOLD = 0.90
TOP_K = 50
MRMR_CRITERION = "MID"
N_BINS = 10
```

### Parameter Description

* `MWU_P_THRESHOLD`
  Significance threshold for the Mann–Whitney U test.

* `SPEARMAN_THRESHOLD`
  Redundancy threshold for absolute Spearman correlation.

* `TOP_K`
  Final number of features to be selected by mRMR.

* `MRMR_CRITERION`
  Scoring rule used in mRMR (`MID` or `MIQ`).

* `N_BINS`
  Number of bins used for discretization before mutual information computation.

---

## Output Files

After running the pipeline, the following files are generated in the output directory:

| File                                  | Description                                           |
| ------------------------------------- | ----------------------------------------------------- |
| `01_MWU_p_values.csv`                 | p-values of all features from Mann–Whitney U test     |
| `02_MWU_significant_features.csv`     | statistically significant features after MWU          |
| `03_Spearman_correlation_matrix.csv`  | absolute Spearman correlation matrix                  |
| `04_Spearman_dropped_features.csv`    | details of removed redundant features                 |
| `05_Features_after_Spearman.csv`      | features retained after Spearman filtering            |
| `06_mRMR_relevance.csv`               | mutual information relevance of features to the label |
| `07_mRMR_selected_features.txt`       | final selected feature names                          |
| `08_mRMR_selection_process.csv`       | detailed mRMR selection process                       |
| `09_mRMR_selected_feature_values.csv` | final feature matrix after mRMR                       |
| `10_Final_selected_dataset.csv`       | final modeling dataset                                |
| `11_feature_count_summary.csv`        | summary of feature counts at each stage               |

---

## Example Interpretation of the Pipeline

Suppose the original dataset contains 1,200 radiomics features.

A typical selection process may look like this:

* Original features: **1,200**
* After Mann–Whitney U filtering: **180**
* After Spearman redundancy reduction: **74**
* After mRMR selection: **50**

This means:

* uninformative features were removed first,
* redundant highly correlated features were then eliminated,
* the final subset was optimized for both relevance and complementarity.

---

## Advantages of This Pipeline

* suitable for high-dimensional radiomics data,
* combines univariate and multivariate selection strategies,
* reduces redundancy before advanced feature ranking,
* improves interpretability and reproducibility,
* does not require external `pymrmr`,
* easy to adapt to other binary classification datasets.

---

## Limitations

* the current implementation is intended for **binary classification only**,
* Mann–Whitney U test is not suitable for multi-class labels in this form,
* discretization in mRMR may lead to some information loss,
* threshold choices may need to be adjusted depending on sample size and study design.

---

## Recommended Use Cases

This pipeline is suitable for:

* radiomics-based diagnostic modeling,
* prognosis prediction,
* treatment response classification,
* biomedical feature reduction before machine learning,
* other binary classification tasks involving high-dimensional tabular features.

---

## Future Extensions

Possible future improvements include:

* support for multi-class classification,
* support for regression tasks,
* multiple-testing correction after Mann–Whitney U test,
* cross-validation-based stable feature selection,
* integration with downstream classifiers such as LASSO, SVM, Random Forest, or XGBoost.

---

## Methods Description

If you use this pipeline in your research, please describe the feature selection strategy clearly in the Methods section, including:

* nonparametric univariate screening by Mann–Whitney U test,
* Spearman correlation-based redundancy removal,
* mRMR-based final feature selection.

A typical description may be:

> Features were first screened using the Mann–Whitney U test, and those with p < 0.05 were retained. Redundant features were then removed using Spearman correlation analysis with a threshold of |ρ| > 0.90, keeping the feature with the smaller p-value when a highly correlated pair was identified. Finally, mRMR was applied to select the most relevant and least redundant features for model development.

---

## Citation

If you use this code, this implementation strategy, or a modified version of it in academic work, please cite the original article:
Wang, Y., Zhang, H., Wang, H. et al. *Development of a neoadjuvant chemotherapy efficacy prediction model for nasopharyngeal carcinoma integrating magnetic resonance radiomics and pathomics: a multi-center retrospective study*. BMC Cancer 24, 1501 (2024). https://doi.org/10.1186/s12885-024-13235-0


