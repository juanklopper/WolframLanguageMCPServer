# Data Dictionary

## DATA.csv

100 rows, 9 columns. Header row present (`HeaderLines -> 1` on import).

| Column | Name for Plots | Type | Role | Values / Range | Notes | Decode |
|---|---|---|---|---|---|---|
| `ID` | "Subject ID" | Integer | Identifier | 1–100, unique | Row identifier only. Not a measured variable — skip inferential stats, but still include a subsection confirming uniqueness/completeness. | Not applicable |
| `Age` | "Age" | Integer | Continuous | 45–80 | Years. | Not applicable |
| `Chol` | "Total Cholesterol" | Integer | Continuous | 205–255 | Cholesterol, mg/dL. | Not applicable |
| `DBP` | "Diastolic Blood Pressure" | Integer | Continuous | 61–103 | Diastolic blood pressure, mmHg. | Not applicable |
| `Temp` | "Temperature" | Real | Continuous | 98.1–99.5 | Body temperature, °F. | Not applicable |
| `Group` | "Group" | String | Categorical (nominal, 2 levels) | `"Intervention"` (n=40), `"Placebo"` (n=60) | Treatment arm. | Not applicable |
| `Survey` | "Survey Question" | Integer | Ordinal (5 levels) | 1–5 | Likert-style survey score. Treat as ordinal: report counts/mode/median, not just mean. | "Strongly Disagree" for 1, "Disagree" for 2, "Neither" for 3, "Agree" for 4, "Strongly Agree" for 5 |
| `DiabetesMellitus` | "Diabetes Mellitus" | String | Categorical (nominal, 2 levels) | `"Y"`, `"N"` | Not applicable | "Yes" for `"Y"`, "No" for `"N"` |
| `EssentialHypertension` | "Essential Hypertension" | Integer | Categorical (binary) | `0`, `1` | Stored as integer but semantically binary/categorical, not continuous — do not treat as ratio-scale. | "No" for 0, "Yes" for 1 |

### Adding a new dataset

Append a new `## <filename>.csv` section above with the same table format: column name, WL type as imported, statistical role (continuous / ordinal / nominal / identifier), value range or levels, and any notes on how it should be treated (e.g. "binary flag, not continuous" or "free text, exclude from univariate stats").
