---
name: compare-two-dependent-groups
description: Compare a numerical variable between two paired/dependent measurements in a Mathematica notebook. Tests normality of the within-pair differences, then runs the paired t-test or the Wilcoxon signed-rank test. Use for before/after, pre/post, matched-pairs, or repeated-measurement comparisons of a continuous outcome at two time points.
---

# Comparing a numerical variable between two dependent (paired) measurements

Adds a `##` section to an existing analysis notebook. Follow the project's `CLAUDE.md`: ALL CAPS headings, ` ```wl ` fences prefixed `In[]:=`, columns extracted with `Normal[data[All, "..."]]`.

## Selecting the columns

**Read the project's `DATA_DICTIONARY.md` before asking the user anything.** Offer only columns whose role fits:

| Role needed | Dictionary role |
|---|---|
| Both measurements | continuous, and the two must be the **same quantity** measured twice |

Present the fitting candidates by name, with their "Name for Plots" label, and ask the user which two columns hold the paired measurements. Do not ask blind. If the dictionary describes the two columns as different quantities, confirm with the user that pairing is intended — a paired test on unrelated measures is meaningless. For three or more occasions use `repeated-measures-anova`.

Then bind the choices once:

```wl
In[]:= (* The only cell carrying project-specific names. Replace both values. *)
firstCol = "PreScore";
secondCol = "PostScore";
```

Everything below refers only to `firstCol` and `secondCol`. Take axis and chart labels from the dictionary's "Name for Plots" entries.

## The critical rule

**Everything is computed on the vector of within-pair differences.** Passing the two measurement vectors as a pair — `LocationTest[{first, second}, ...]` — silently runs the *independent* two-sample test and returns a different, wrong p-value with no warning. Always form the differences first:

```wl
In[]:= diffs = first - second;
```

Rows must be aligned pair-for-pair before subtracting. If the data is in long format, pivot to wide and confirm every subject contributes exactly two measurements.

## Decision rule

```
Shapiro-Wilk on the DIFFERENCES
├── p ≥ 0.05  →  paired t-test (one-sample t on the differences vs 0)
└── p < 0.05  →  Wilcoxon signed-rank test
```

Normality of each measurement separately is irrelevant — only the differences matter.

## Section layout

```
## COMPARISON OF <MEASURE> BETWEEN <TIME 1> AND <TIME 2>
### HYPOTHESES
### PAIRED DESCRIPTIVE STATISTICS
### ASSUMPTION TEST
### HYPOTHESIS TEST
### EFFECT SIZE AND CONFIDENCE INTERVAL
### VISUALIZATION
```

## HYPOTHESES

> We test whether the mean within-pair difference differs from zero at the 5% significance level.
>
> $$H_0 \colon \mu_d = 0$$
>
> $$H_1 \colon \mu_d \ne 0$$

If the test falls back to Wilcoxon signed-rank, restate in terms of the median difference.

## PAIRED DESCRIPTIVE STATISTICS

```wl
In[]:= first = N@Normal[data[All, firstCol]];
second = N@Normal[data[All, secondCol]];
diffs = first - second;
Dataset[<|
  "n pairs" -> Length[diffs],
  "Mean " <> firstCol -> Mean[first], "SD " <> firstCol -> StandardDeviation[first],
  "Mean " <> secondCol -> Mean[second], "SD " <> secondCol -> StandardDeviation[second],
  "Mean difference" -> Mean[diffs], "SD difference" -> StandardDeviation[diffs],
  "Median difference" -> Median[diffs]|>]
```

## ASSUMPTION TEST

```wl
In[]:= Dataset[<|"Shapiro-Wilk on differences" -> ShapiroWilkTest[diffs, "PValue"]|>]
```

```wl
In[]:= Histogram[diffs, Automatic,
  PlotLabel -> "Distribution of Within-Pair Differences",
  FrameLabel -> {"Difference", "Count"}, ImageSize -> 420]
```

A text cell should state which branch the p-value selects.

## HYPOTHESIS TEST

`LocationTest` on the difference vector applies the decision rule automatically — it returns `T` when the differences pass its normality screen and `Signed-Rank` when they do not. `"TestDataTable"` names the chosen test.

```wl
In[]:= LocationTest[diffs, 0, "TestDataTable"]
```

```wl
In[]:= LocationTest[diffs, 0, {"TestStatistic", "PValue", "ShortTestConclusion"}]
```

### Forcing a specific test

`LocationTest` screens normality at 0.025, so it can disagree with Shapiro-Wilk read at 0.05. Force the test when the explicit assumption test says otherwise:

```wl
In[]:= (* paired t-test *)
LocationTest[diffs, 0, "TestDataTable", VerifyTestAssumptions -> None]
```

```wl
In[]:= (* Wilcoxon signed-rank *)
SignedRankTest[diffs, 0, "TestDataTable"]
```

Never call `TTest` directly — it aborts rather than returning a result when its normality screen fails.

## EFFECT SIZE AND CONFIDENCE INTERVAL

Cohen's *d* for paired data divides by the SD of the differences, not a pooled SD.

```wl
In[]:= cohenDz = Mean[diffs]/StandardDeviation[diffs];
ci = Mean[diffs] + {-1, 1} Quantile[StudentTDistribution[Length[diffs] - 1], 0.975] *
   StandardDeviation[diffs]/Sqrt[Length[diffs]];
<|"Cohen's dz" -> cohenDz, "Mean difference" -> Mean[diffs], "95% CI" -> ci|>
```

## VISUALIZATION

Set `measureLabel`, `firstLabel` and `secondLabel` from the data dictionary's "Name for Plots" entries. A paired-line plot shows each subject's trajectory and is far more informative here than side-by-side box plots. Include both.

```wl
In[]:= ListLinePlot[Transpose[{first, second}],
  PlotStyle -> Directive[Opacity[0.4], Gray],
  Ticks -> {{{1, firstLabel}, {2, secondLabel}}, Automatic},
  PlotLabel -> "Individual Paired Trajectories",
  AxesLabel -> {None, measureLabel}, ImageSize -> 400]
```

```wl
In[]:= BoxWhiskerChart[{first, second}, "Outliers",
  ChartLabels -> {firstLabel, secondLabel}, ChartStyle -> "Pastel",
  PlotLabel -> firstLabel <> " vs " <> secondLabel,
  FrameLabel -> {None, measureLabel}, ImageSize -> 380]
```

```wl
In[]:= BoxWhiskerChart[diffs, "Outliers", ChartLabels -> {"Difference"},
  PlotLabel -> "Within-Pair Differences", ImageSize -> 300]
```

## Reporting

Close with a text cell giving the test used, the statistic, degrees of freedom where applicable, the p-value, the effect size, the number of pairs, and a plain-language conclusion tied back to the hypotheses.
