---
name: repeated-measures-anova
description: Run a one-way repeated measures ANOVA in a Mathematica notebook, where the same subjects are measured three or more times. Checks normality and sphericity (Greenhouse-Geisser), applies a correction when sphericity fails, and falls back to the Friedman test. Use for within-subjects, longitudinal, or repeated-measurement designs with three or more time points.
---

# One-way repeated measures ANOVA

Adds a `##` section to an existing analysis notebook. Follow the project's `CLAUDE.md`: ALL CAPS headings, ` ```wl ` fences prefixed `In[]:=`, columns extracted with `Normal[data[All, "..."]]`.

## Selecting the columns

**Read the project's `DATA_DICTIONARY.md` before asking the user anything.** Offer only columns whose role fits:

| Role needed | Dictionary role |
|---|---|
| Repeated measurements | continuous, **three or more** columns measuring the same quantity |
| Subject identifier | identifier (optional — row position is used if absent) |

Present the fitting candidates by name, with their "Name for Plots" label, and ask the user which columns hold the repeated measurements and in what order. Do not ask blind; order matters because the occasions are plotted and compared in sequence. For exactly two measurements use `compare-two-dependent-groups`.

Then bind the choices once:

```wl
In[]:= (* The only cell carrying project-specific names. Replace these values. *)
timeCols = {"T1", "T2", "T3"};
timeLabels = {"Baseline", "Month 3", "Month 6"};
measureLabel = "Measurement";
```

Everything below refers only to `timeCols`, `timeLabels` and `measureLabel`. Take the labels from the dictionary's "Name for Plots" entries.

## How Mathematica does this

There is no dedicated repeated-measures function. A one-way RM-ANOVA is an **additive two-way ANOVA with subject as a blocking factor**, which the legacy `ANOVA` package computes directly:

```wl
In[]:= ANOVA[longForm, {Subject, Time}, {Subject, Time}]
```

The model spec `{Subject, Time}` requests **main effects only** — no interaction term. That is what makes it a repeated-measures analysis; the subject sum of squares is partitioned out of the error term. The `Time` row of the resulting table is the effect of interest. This has been verified to reproduce a hand-computed RM-ANOVA exactly.

## Decision rule

```
Shapiro-Wilk on the residuals
├── p ≥ 0.05  →  sphericity? (Greenhouse-Geisser ε)
│   ├── ε ≥ 0.75  →  RM-ANOVA, uncorrected
│   └── ε < 0.75  →  RM-ANOVA with Greenhouse-Geisser corrected df
└── p < 0.05  →  Friedman test
```

## Section layout

```
## REPEATED MEASURES ANALYSIS OF <MEASURE>
### HYPOTHESES
### DATA RESHAPING
### DESCRIPTIVE STATISTICS BY TIME POINT
### ASSUMPTION TESTS
### REPEATED MEASURES ANOVA
### POST-HOC PAIRWISE COMPARISONS
### VISUALIZATION
```

## HYPOTHESES

> We test whether mean <measure> differs across the <k> measurement occasions at the 5% significance level.
>
> $$H_0 \colon \mu_1 = \mu_2 = \cdots = \mu_k$$
>
> $H_1$: at least one occasion mean differs from the others.

## DATA RESHAPING

Two shapes are needed: **wide** (one row per subject, one column per occasion) for the sphericity and Friedman computations, and **long** (`{subject, time, value}`) for the ANOVA package.

```wl
In[]:= wide = N@Transpose[Table[Normal[data[All, c]], {c, timeCols}]];
nSubjects = Length[wide];
kTimes = Length[timeCols];
longForm = Flatten[Table[{s, t, wide[[s, t]]}, {s, nSubjects}, {t, kTimes}], 1];
{nSubjects, kTimes, Dimensions[longForm]}
```

Every subject must contribute a value at every occasion — the design must be complete. Check before proceeding:

```wl
In[]:= If[! FreeQ[wide, _Missing | ""],
  Print["Incomplete design - RM-ANOVA requires complete cases"],
  Print["Design is complete"]]
```

## DESCRIPTIVE STATISTICS BY TIME POINT

```wl
In[]:= Dataset[AssociationThread[timeLabels -> Map[
   <|"n" -> Length[#], "Mean" -> Mean[#], "SD" -> StandardDeviation[#],
     "Median" -> Median[#], "Min" -> Min[#], "Max" -> Max[#]|> &,
   Transpose[wide]]]]
```

## ASSUMPTION TESTS

### Normality of residuals

```wl
In[]:= subjectMeans = Mean /@ wide;
timeMeans = Mean /@ Transpose[wide];
grandMean = Mean[Flatten[wide]];
residuals = Flatten[Table[
    wide[[s, t]] - subjectMeans[[s]] - timeMeans[[t]] + grandMean,
    {s, nSubjects}, {t, kTimes}]];
Dataset[<|"Shapiro-Wilk on residuals" -> ShapiroWilkTest[residuals, "PValue"]|>]
```

### Sphericity

Mauchly's test is **not available in Mathematica**. Use the Greenhouse-Geisser epsilon instead, which measures the departure from sphericity directly. It is bounded below by $1/(k-1)$ and above by 1; a value of 1 means sphericity holds exactly.

```wl
In[]:= ggEpsilon[w_] := Module[{s, k, dbar, gm, rowMeans, num, den},
   s = Covariance[N@w]; k = Length[s];
   dbar = Mean[Diagonal[s]]; gm = Mean[Flatten[s]]; rowMeans = Mean /@ s;
   num = k^2 (dbar - gm)^2;
   den = (k - 1) (Total[Flatten[s^2]] - 2 k Total[rowMeans^2] + k^2 gm^2);
   num/den];
epsilon = ggEpsilon[wide];
Dataset[<|"Greenhouse-Geisser epsilon" -> epsilon,
  "Lower bound" -> N[1/(kTimes - 1)],
  "Sphericity acceptable" -> epsilon >= 0.75|>]
```

State in a text cell which branch of the decision rule these results select.

## REPEATED MEASURES ANOVA

```wl
In[]:= Needs["ANOVA`"];
rmResult = ANOVA[longForm, {Subject, Time}, {Subject, Time}];
ANOVA /. rmResult
```

Rows are `Subject`, `Time`, `Error`, `Total`; columns are `DF`, `SumOfSq`, `MeanSq`, `FRatio`, `PValue`. **Report the `Time` row.** The `Subject` row confirms that between-subject variance was partitioned out and is not itself of interest.

```wl
In[]:= CellMeans /. rmResult
```

### Greenhouse-Geisser correction

When $\varepsilon < 0.75$, multiply both degrees of freedom by epsilon and recompute the p-value from the same F ratio. The F statistic is unchanged; only its reference distribution narrows.

```wl
In[]:= anovaTable = First[ANOVA /. rmResult];
{dfTime, ssTime, msTime, fTime, pTime} = anovaTable[[2]];
{dfError, ssError, msError} = anovaTable[[3, 1 ;; 3]];
ggCorrected = <|
   "F" -> fTime,
   "df (uncorrected)" -> {dfTime, dfError},
   "df (GG corrected)" -> {dfTime epsilon, dfError epsilon},
   "p (uncorrected)" -> pTime,
   "p (GG corrected)" -> 1 - CDF[FRatioDistribution[dfTime epsilon, dfError epsilon], fTime]|>
```

## FRIEDMAN TEST

Use when the residuals fail normality. Mathematica has **no built-in Friedman test**; the implementation below ranks within each subject across occasions and has been verified against a hand-computed reference case.

```wl
In[]:= rankRow[l_] := Module[{o = Ordering[l], r = ConstantArray[0., Length[l]],
    t = Tally[Sort[l]], pos = 1},
   Do[With[{c = t[[k, 2]]},
     Do[r[[o[[pos + j - 1]]]] = pos + (c - 1)/2., {j, c}]; pos += c], {k, Length[t]}];
   r];
friedman[w_] := Module[{n = Length[w], k = Length[First[w]], ranks, rankSums, chi},
   ranks = rankRow /@ w;
   rankSums = Total[ranks];
   chi = 12/(n k (k + 1)) Total[rankSums^2] - 3 n (k + 1);
   <|"ChiSquare" -> chi, "df" -> k - 1,
     "PValue" -> 1 - CDF[ChiSquareDistribution[k - 1], chi],
     "RankSums" -> rankSums|>];
friedman[wide]
```

## POST-HOC PAIRWISE COMPARISONS

Run only if the omnibus test is significant. Compare occasions pairwise as dependent samples with a Bonferroni-adjusted threshold.

```wl
In[]:= pairs = Subsets[Range[kTimes], {2}];
adjusted = 0.05/Length[pairs];
Dataset[Table[
  With[{d = wide[[All, pr[[1]]]] - wide[[All, pr[[2]]]]},
   <|"Pair" -> timeLabels[[pr]],
     "Mean difference" -> Mean[d],
     "p" -> LocationTest[d, 0, "PValue"],
     "Significant" -> LocationTest[d, 0, "PValue"] < adjusted|>], {pr, pairs}]]
```

Use `SignedRankTest[d, 0, "PValue"]` in place of `LocationTest` when following a Friedman test.

## VISUALIZATION

```wl
In[]:= ListLinePlot[wide, PlotStyle -> Directive[Opacity[0.3], Gray],
  Ticks -> {Thread[{Range[kTimes], timeLabels}], Automatic},
  PlotLabel -> "Individual Subject Trajectories",
  AxesLabel -> {None, measureLabel}, ImageSize -> 450]
```

```wl
In[]:= BoxWhiskerChart[Transpose[wide], "Outliers", ChartLabels -> timeLabels,
  ChartStyle -> "Pastel", PlotLabel -> "Distribution by Occasion",
  FrameLabel -> {"Occasion", measureLabel}, ImageSize -> 450]
```

```wl
In[]:= errors = StandardDeviation[#]/Sqrt[Length[#]] & /@ Transpose[wide];
ListPlot[MapThread[Around, {timeMeans, errors}],
  Joined -> True, PlotMarkers -> Automatic,
  Ticks -> {Thread[{Range[kTimes], timeLabels}], Automatic},
  PlotLabel -> "Mean by Occasion with Standard Error",
  AxesLabel -> {None, "Mean " <> measureLabel}, ImageSize -> 450]
```

## Reporting

Close with a text cell giving the test used, the F ratio with its degrees of freedom (stating whether they are Greenhouse-Geisser corrected and the epsilon value), the p-value, which pairs differed post-hoc, the number of subjects, and a plain-language conclusion tied to the hypotheses.
