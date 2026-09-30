# FIFA 2 Data Cleaning Challenge

A complete, reproducible data-cleaning workflow for the FIFA player dataset. This project starts with a raw CSV export, audits its columns and missing values, standardizes inconsistent text and units, converts financial fields into numeric values, handles missing `Hits`, rebuilds the cleaned dataset, validates the result, and saves the final CSV.

> This README translates the original Jupyter Notebook workflow into a clean, GitHub-ready guide. The sequence is kept from start to finish, while a few fragile expressions from the notebook are made safer and more reusable.

## Table of Contents

- [Project Overview](#project-overview)
- [Dataset](#dataset)
- [Cleaning Goals](#cleaning-goals)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [How to Run](#how-to-run)
- [Step-by-Step Workflow](#step-by-step-workflow)
  - [1. Import Libraries](#1-import-libraries)
  - [2. Load and Inspect the Raw Dataset](#2-load-and-inspect-the-raw-dataset)
  - [3. Audit Missing Values](#3-audit-missing-values)
  - [4. Select Columns That Need Cleaning](#4-select-columns-that-need-cleaning)
  - [5. Clean Club and Contract](#5-clean-club-and-contract)
  - [6. Standardize Height](#6-standardize-height)
  - [7. Standardize Weight](#7-standardize-weight)
  - [8. Convert Joined to a Standard Date](#8-convert-joined-to-a-standard-date)
  - [9. Convert Value, Wage, and Release Clause](#9-convert-value-wage-and-release-clause)
  - [10. Clean Star Ratings](#10-clean-star-ratings)
  - [11. Handle Missing Hits and K-Values](#11-handle-missing-hits-and-k-values)
  - [12. Validate the Cleaning Subset](#12-validate-the-cleaning-subset)
  - [13. Select the Remaining Analysis Columns](#13-select-the-remaining-analysis-columns)
  - [14. Merge the Cleaned Columns Back](#14-merge-the-cleaned-columns-back)
  - [15. Reorder Columns](#15-reorder-columns)
  - [16. Final Validation and Export](#16-final-validation-and-export)
- [Expected Output](#expected-output)
- [Important Notes](#important-notes)
- [Possible Extensions](#possible-extensions)

## Project Overview

The raw FIFA dataset contains player information, club information, physical measurements, contract details, ratings, financial values, and performance statistics. Several columns contain values stored as text with symbols or mixed units, including:

- Newline characters and formatting artifacts in club names.
- Tildes in contract ranges, such as `2018 ~ 2022`.
- Heights stored as both centimetres and feet/inches.
- Weights stored as both kilograms and pounds.
- Dates stored as strings such as `Jul 1, 2004`.
- Currency values using the euro symbol and suffixes such as `M` and `K`.
- Star ratings such as `4★`.
- Hits values such as `1.6K` and missing values.

The final result is a cleaner, analysis-ready table indexed by player `ID`.

## Dataset

The notebook uses the FIFA 21 player dataset. Place the raw CSV in the project data directory and update the path in the loading cell if necessary.

The raw dataset used in the notebook contains **77 columns**. The cleaned output contains **66 columns**, with URL fields and other low-value columns excluded from the analysis table.

## Cleaning Goals

1. Load and inspect the raw data.
2. Understand the dataset structure and missing values.
3. Isolate columns that require transformation.
4. Remove formatting characters without damaging meaningful text.
5. Convert mixed physical units to consistent units.
6. Convert human-readable financial values to numeric values.
7. Convert ratings and hit counts into numeric values.
8. Fill missing `Hits` with zero.
9. Merge the cleaned subset with the remaining useful columns.
10. Validate and export the finished dataset.

## Project Structure

```text
.
├── data/
│   ├── fifa.csv                 # Raw input dataset
│   └── fifa2_cleaned_data.csv   # Generated output dataset
├── Fifa_2_Data_Cleaning_Challenge.ipynb
└── README.md
```

## Requirements

- Python 3.8+
- pandas
- numpy
- matplotlib
- seaborn

Install the dependencies with:

```bash
pip install pandas numpy matplotlib seaborn
```

## How to Run

1. Clone or download this repository.
2. Put the raw FIFA CSV file at `data/fifa.csv`.
3. Open the notebook or copy the code below into a Python script.
4. Run the cells in order.
5. Find the cleaned dataset at `data/fifa2_cleaned_data.csv`.

The path is deliberately written with `pathlib`, so the workflow works on Windows, macOS, and Linux.

## Step-by-Step Workflow

### 1. Import Libraries

The first step loads the packages used for data manipulation, numerical work, and visual inspection.

```python
import re
from pathlib import Path
from datetime import datetime

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

%matplotlib inline
```

### 2. Load and Inspect the Raw Dataset

Load the raw file and inspect the first five rows. The original notebook calls this object `fifa_dirty` because it is the uncleaned source table.

```python
DATA_DIR = Path("data")
RAW_FILE = DATA_DIR / "fifa.csv"
OUTPUT_FILE = DATA_DIR / "fifa2_cleaned_data.csv"

fifa_dirty = pd.read_csv(RAW_FILE)
fifa_dirty.head()
```

The raw table should contain 77 columns:

```python
print(f"Rows: {len(fifa_dirty):,}")
print(f"Columns: {len(fifa_dirty.columns)}")
print(fifa_dirty.columns.tolist())
```

### 3. Audit Missing Values

The original notebook divides the columns into four groups to make the wide dataset easier to inspect. This is useful when working with a table that contains many fields.

```python
fifa_unclean_1 = fifa_dirty.iloc[:, 0:20]
fifa_unclean_2 = fifa_dirty.iloc[:, 20:40]
fifa_unclean_3 = fifa_dirty.iloc[:, 40:60]
fifa_unclean_4 = fifa_dirty.iloc[:, 60:]

print("Missing values in columns 1–20")
display(fifa_unclean_1.isnull().sum())

print("Missing values in columns 21–40")
display(fifa_unclean_2.isnull().sum())

print("Missing values in columns 41–60")
display(fifa_unclean_3.isnull().sum())

print("Missing values in columns 61–77")
display(fifa_unclean_4.isnull().sum())
```

The notebook identifies substantial missingness in `Loan Date End` and `Hits`. `Loan Date End` is not used in the cleaned analysis table, while missing `Hits` values are filled with zero later in the workflow.

A quick visual inspection of the last group is also useful:

```python
fifa_unclean_4.head()
```

### 4. Select Columns That Need Cleaning

The following 12 columns contain the main formatting and type inconsistencies addressed in the notebook:

```python
columns_to_clean = [
    "ID", "Club", "Contract", "Height", "Weight", "Joined",
    "Value", "Wage", "Release Clause", "W/F", "SM", "IR", "Hits"
]

fifa_to_clean = fifa_dirty[columns_to_clean]
fifa_cleaning = fifa_to_clean.copy()
fifa_cleaning.head()
```

### 5. Clean Club and Contract

The raw `Club` field contains line breaks and other display artifacts. The safest approach is to normalize whitespace rather than remove individual letters such as `c` or `m`, which could corrupt legitimate club names.

```python
def clean_text(value):
    """Normalize whitespace and return a clean string."""
    if pd.isna(value):
        return value
    value = str(value).replace("\\n", " ")
    return re.sub(r"\\s+", " ", value).strip()

fifa_cleaning["Club"] = fifa_cleaning["Club"].apply(clean_text)
```

Contract ranges use a tilde as a separator. Replace it with a hyphen for readability:

```python
fifa_cleaning["Contract"] = (
    fifa_cleaning["Contract"]
    .astype("string")
    .str.replace("~", "-", regex=False)
    .str.replace(r"\\s+", " ", regex=True)
    .str.strip()
)

fifa_cleaning[["Club", "Contract"]].head()
```

### 6. Standardize Height

The notebook renames `Height` and `Weight` before cleaning them. Although the original notebook names the weight column `Weight(cm)`, the correct unit is kilograms, so this README uses the clearer name `Weight(kg)`.

```python
fifa_cleaning = fifa_cleaning.rename(
    columns={
        "Height": "Height(cm)",
        "Weight": "Weight(kg)"
    }
)

fifa_cleaning["Height(cm)"].unique()[:20]
```

The raw height column includes values such as `170cm` and `6'2"`. Convert both formats to centimetres:

```python
def convert_height(value):
    """Convert centimetres or feet/inches to centimetres."""
    if pd.isna(value):
        return np.nan

    value = str(value).strip()

    if value.endswith("cm"):
        return float(value[:-2])

    feet_inches = re.fullmatch(r"(\d+)'(\d+)", value)
    if feet_inches:
        feet, inches = map(int, feet_inches.groups())
        return round((feet * 12 + inches) * 2.54, 2)

    return pd.to_numeric(value, errors="coerce")

fifa_cleaning["Height(cm)"] = fifa_cleaning["Height(cm)"].apply(convert_height)
fifa_cleaning["Height(cm)"].unique()[:20]
```

### 7. Standardize Weight

The raw weight column contains both kilograms and pounds. Convert all values to kilograms:

```python
def convert_weight(value):
    """Convert kilograms or pounds to kilograms."""
    if pd.isna(value):
        return np.nan

    value = str(value).strip().lower()

    if value.endswith("kg"):
        return float(value[:-2])

    if value.endswith("lbs"):
        # The original notebook uses 0.45 kg per pound.
        return round(float(value[:-3]) * 0.45, 2)

    return pd.to_numeric(value, errors="coerce")

fifa_cleaning["Weight(kg)"] = fifa_cleaning["Weight(kg)"].apply(convert_weight)
fifa_cleaning["Weight(kg)"].unique()[:20]
```

### 8. Convert Joined to a Standard Date

The `Joined` field is stored as text in the format `Mon DD, YYYY`. Convert it to a standard ISO-style date (`YYYY-MM-DD`).

```python
fifa_cleaning["Joined"] = pd.to_datetime(
    fifa_cleaning["Joined"],
    format="%b %d, %Y",
    errors="coerce"
).dt.strftime("%Y-%m-%d")

fifa_cleaning[["ID", "Joined"]].head()
```

### 9. Convert Value, Wage, and Release Clause

The financial columns use a euro symbol and suffixes:

- `M` means millions.
- `K` means thousands.
- A value without a suffix is treated as a plain number.

The notebook first removes the euro symbol and then expands the suffix. The function below performs both steps safely and returns numeric values.

```python
def full_figure(value):
    """Convert values such as €103.5M or €560K to numeric amounts."""
    if pd.isna(value):
        return np.nan

    value = str(value).strip().replace("€", "").replace(",", "")
    value = value.replace(" ", "")

    if not value:
        return np.nan

    multiplier = 1
    if value.upper().endswith("M"):
        multiplier = 1_000_000
        value = value[:-1]
    elif value.upper().endswith("K"):
        multiplier = 1_000
        value = value[:-1]

    return int(float(value) * multiplier)

for column in ["Value", "Wage", "Release Clause"]:
    fifa_cleaning[column] = fifa_cleaning[column].apply(full_figure)

fifa_cleaning[["Value", "Wage", "Release Clause"]].head()
```

The resulting columns are numeric and can be used directly for calculations, sorting, grouping, and visualization.

### 10. Clean Star Ratings

`W/F`, `SM`, and `IR` contain values such as `4★`. Remove the star symbol and convert the result to a numeric type.

```python
for column in ["W/F", "SM", "IR"]:
    fifa_cleaning[column] = (
        fifa_cleaning[column]
        .astype("string")
        .str.replace("★", "", regex=False)
        .str.strip()
    )
    fifa_cleaning[column] = pd.to_numeric(
        fifa_cleaning[column],
        errors="coerce"
    )

fifa_cleaning[["W/F", "SM", "IR"]].head()
```

### 11. Handle Missing Hits and K-Values

Inspect the missing values in `Hits` before deciding how to handle them:

```python
fifa_cleaning["Hits"].isnull().sum()
```

The original notebook finds 2,595 missing `Hits` values and fills them with zero so that the remaining player records are not discarded.

```python
fifa_cleaning["Hits"] = fifa_cleaning["Hits"].fillna(0)
fifa_cleaning["Hits"].unique()[:20]
```

Some values are displayed in thousands, such as `1.6K`. The same numeric conversion helper can standardize them:

```python
fifa_cleaning["Hits"] = fifa_cleaning["Hits"].apply(full_figure)
fifa_cleaning["Hits"] = pd.to_numeric(
    fifa_cleaning["Hits"],
    errors="coerce"
).fillna(0).astype(int)

fifa_cleaning["Hits"].unique()[:20]
```

### 12. Validate the Cleaning Subset

Use a heatmap to visually check the cleaned subset for missing values:

```python
plt.figure(figsize=(14, 4))
sns.heatmap(
    fifa_cleaning.isnull(),
    cbar=False,
    yticklabels=False,
    cmap="viridis"
)
plt.title("Missing Values in the Cleaned Subset")
plt.show()
```

Set `ID` as the index of the cleaned subset. This makes it ready to merge with the remaining analysis columns.

```python
fifa_cleaning.set_index("ID", inplace=True)
fifa_cleaning.head()
```

### 13. Select the Remaining Analysis Columns

The original dataset contains URL fields and other columns that are not needed for the analysis table. Keep the useful player, rating, position, and performance fields, while leaving the already-cleaned columns out of this second subset.

```python
remaining_columns = [
    "LongName", "Nationality", "Age", "↓OVA", "POT",
    "Preferred Foot", "BOV", "Best Position",
    "Attacking", "Crossing", "Finishing", "Heading Accuracy",
    "Short Passing", "Volleys", "Skill", "Dribbling", "Curve",
    "FK Accuracy", "Long Passing", "Ball Control", "Movement",
    "Acceleration", "Sprint Speed", "Agility", "Reactions", "Balance",
    "Power", "Shot Power", "Jumping", "Stamina", "Strength", "Long Shots",
    "Mentality", "Aggression", "Interceptions", "Vision", "Penalties",
    "Composure", "Defending", "Marking", "Standing Tackle",
    "GK Kicking", "GK Positioning", "GK Reflexes", "Total Stats", "Base Stats",
    "A/W", "D/W", "PAC", "SHO", "PAS", "DRI", "DEF", "PHY"
]

fifa_cleaned = fifa_dirty[["ID"] + remaining_columns].copy()
print(f"Remaining analysis columns: {len(fifa_cleaned.columns)}")
fifa_cleaned.head()
```

The cleaned subset contains 12 columns, while the remaining analysis subset contains 55 columns including `ID`.

### 14. Merge the Cleaned Columns Back

Merge both subsets using the stable player identifier, `ID`.

```python
fifa2_cleaned_data = pd.merge(
    fifa_cleaning.reset_index(),
    fifa_cleaned,
    on="ID",
    how="inner"
)

fifa2_cleaned_data.head()
```

Set `ID` as the final index:

```python
fifa2_cleaned_data.set_index("ID", inplace=True)
fifa2_cleaned_data.head()
```

### 15. Reorder Columns

Put the most useful identifying, club, contract, physical, financial, and rating columns first. Keep the remaining performance fields after them.

```python
priority_columns = [
    "LongName", "Nationality", "Club", "Contract", "Age",
    "Height(cm)", "Weight(kg)", "Joined", "Value", "Wage",
    "Release Clause", "W/F", "SM", "IR", "Hits", "↓OVA", "POT",
    "Preferred Foot", "BOV", "Best Position"
]

remaining_order = [
    column for column in fifa2_cleaned_data.columns
    if column not in priority_columns
]

new_index = priority_columns + remaining_order
fifa2_cleaned_data = fifa2_cleaned_data.reindex(columns=new_index)
fifa2_cleaned_data.head()
```

### 16. Final Validation and Export

Check the final shape, data types, duplicate IDs, and missing values before exporting.

```python
print(f"Final rows: {len(fifa2_cleaned_data):,}")
print(f"Final columns: {len(fifa2_cleaned_data.columns)}")
print(f"Duplicate IDs: {fifa2_cleaned_data.index.duplicated().sum()}")

missing_values = fifa2_cleaned_data.isnull().sum()
display(missing_values[missing_values > 0])
```

Create a final heatmap of missing values:

```python
plt.figure(figsize=(16, 5))
sns.heatmap(
    fifa2_cleaned_data.isnull(),
    cbar=False,
    yticklabels=False,
    cmap="viridis"
)
plt.title("Missing Values in the Final FIFA Dataset")
plt.ylabel("Player ID")
plt.show()
```

Save the cleaned dataset. The index is written as a regular column so the output can be loaded easily by other tools.

```python
DATA_DIR.mkdir(parents=True, exist_ok=True)
fifa2_cleaned_data.to_csv(OUTPUT_FILE, index=True, index_label="ID")
print(f"Saved cleaned data to: {OUTPUT_FILE}")
```

## Expected Output

After the workflow runs successfully, the output file should be available at:

```text
data/fifa2_cleaned_data.csv
```

The final table should have:

- `ID` as its index or exported identifier column.
- Cleaned club names and contract ranges.
- Height in centimetres.
- Weight in kilograms.
- Dates in `YYYY-MM-DD` format.
- Numeric `Value`, `Wage`, and `Release Clause` fields.
- Numeric `W/F`, `SM`, and `IR` ratings.
- Numeric `Hits`, with missing values represented as zero.
- Player, nationality, rating, position, and performance columns.
- URL columns removed from the analysis table.

## Important Notes

- **Use the correct raw filename.** If your source file has another name, update `RAW_FILE`.
- **Currency units are preserved as numeric amounts.** The values are expanded into their base units; for example, `€103.5M` becomes `103500000`.
- **Weight is named `Weight(kg)`.** The original notebook used `Weight(cm)`, but kilograms are the correct unit for that field.
- **Missing `Hits` are filled with zero.** This follows the notebook’s decision to retain the associated player records rather than drop them.
- **Club-name cleaning is intentionally conservative.** Removing every occurrence of characters such as `c` or `m` can corrupt names, so this version removes formatting artifacts and normalizes whitespace only.
- **The cleaned output is analysis-ready, not necessarily publication-ready.** You may still want to standardize column names, document units in a data dictionary, or add validation tests for production use.

## Possible Extensions

- Add automated tests for every conversion function.
- Convert `Joined` to a true pandas datetime column instead of a formatted string.
- Standardize all column names to `snake_case`.
- Add a data dictionary describing every field and unit.
- Produce summary statistics for age, value, wage, and overall rating.
- Investigate outliers in height, weight, value, and hits.
- Build visualizations comparing player value, wage, nationality, position, and club.
- Save the cleaned dataset as both CSV and Parquet for efficient downstream analysis.

## License and Attribution

This README documents the FIFA data-cleaning workflow represented in the supplied Jupyter Notebook. Confirm the original dataset's licensing and attribution requirements before redistributing the raw or cleaned data.
