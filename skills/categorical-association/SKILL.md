---
name: categorical-association
description: Test the association between two categorical variables in a Mathematica notebook. Builds an observed contingency table with margins and a table of expected values, verifies the expected-count requirement, then runs a chi-squared test of independence with Cramer's V and standardized residuals. Use for cross-tabulation, contingency table, or chi-squared independence questions.
---

# Testing association between two categorical variables

Adds a `##` section to an existing analysis notebook. Follow the project's `CLAUDE.md`: ALL CAPS headings, ` ```wl ` fences prefixed `In[]:=`, columns extracted with `Normal[data[All, "..."]]`.

## Selecting the columns

**Read the project's `DATA_DICTIONARY.md` before asking the user anything.** Offer only columns whose role fits:

| Role needed | Dictionary role |
|---|---|
| Both variables | categorical/nominal, ordinal, or binary |

Present the fitting candidates by name, with their "Name for Plots" label and level count, and ask the user which two to cross-tabulate. Do not ask blind. A column stored as an integer but marked binary or categorical in the dictionary qualifies — do not exclude it for being numeric, and do not include a continuous column.

Then bind the choices once:

```wl
In[]:= (* The only cell carrying project-specific names. Replace both values. *)
rowVar = "Group";
colVar = "DiabetesMellitus";
```

Everything below refers only to `rowVar` and `colVar`. Take axis labels from the dictionary's "Name for Plots" entries and level labels from its "Decode" entries — a column coded `0`/`1` should be displayed with its decoded names, not its raw codes.

## Mathematica has no contingency-table chi-squared test

This must be computed manually. Both candidate built-ins fail:

- `PearsonChiSquareTest` is a **goodness-of-fit** test against a distribution. Passing it a contingency table raises `PearsonChiSquareTest::rctnln` and returns unevaluated.
- `IndependenceTest` requires two data vectors, not a table, and tests a different hypothesis.

The formulas below are standard and have been verified numerically.

## Section layout

```
## ASSOCIATION BETWEEN <VARIABLE 1> AND <VARIABLE 2>
### HYPOTHESES
### OBSERVED CONTINGENCY TABLE
### EXPECTED VALUES
### ASSUMPTION CHECK
### CHI-SQUARED TEST OF INDEPENDENCE
### EFFECT SIZE AND RESIDUALS
### VISUALIZATION
```

## HYPOTHESES

> We test whether <variable 1> and <variable 2> are independent at the 5% significance level.
>
> $H_0$: the two variables are independent.
>
> $H_1$: the two variables are associated.

The test statistic, stated in a text cell, is

$$\chi^2 = \sum_{i=1}^{r}\sum_{j=1}^{c} \frac{(O_{ij} - E_{ij})^2}{E_{ij}}$$

with $(r-1)(c-1)$ degrees of freedom, where $E_{ij} = \frac{R_i C_j}{n}$.

## OBSERVED CONTINGENCY TABLE

Set `rowLabels` and `colLabels` from the dictionary's "Decode" entries, falling back to `rowLevels`/`colLevels` when a column has no decode.

```wl
In[]:= rowLevels = Sort[DeleteDuplicates[Normal[data[All, rowVar]]]];
colLevels = Sort[DeleteDuplicates[Normal[data[All, colVar]]]];
pairsData = Transpose[{Normal[data[All, rowVar]], Normal[data[All, colVar]]}];
observed = Table[Count[pairsData, {r, c}], {r, rowLevels}, {c, colLevels}];
rowTotals = Total /@ observed;
colTotals = Total[observed];
n = Total[rowTotals];
Grid[Join[
  {Join[{""}, colLabels, {"Total"}]},
  Table[Join[{rowLabels[[i]]}, observed[[i]], {rowTotals[[i]]}], {i, Length[rowLevels]}],
  {Join[{"Total"}, colTotals, {n}]}],
 Frame -> All, Background -> {None, {{LightGray, None}}}]
```

Row and column percentages are often more interpretable than raw counts:

```wl
In[]:= Grid[Join[
  {Join[{"Row %"}, colLabels]},
  Table[Join[{rowLabels[[i]]}, N[100 observed[[i]]/rowTotals[[i]]]], {i, Length[rowLevels]}]],
 Frame -> All]
```

## EXPECTED VALUES

The expected count in each cell under independence is the row total times the column total, divided by the grand total.

```wl
In[]:= expected = N@Outer[Times, rowTotals, colTotals]/n;
Grid[Join[
  {Join[{""}, colLabels]},
  Table[Join[{rowLabels[[i]]}, NumberForm[#, {5, 2}] & /@ expected[[i]]],
   {i, Length[rowLevels]}]],
 Frame -> All, Background -> {None, {{LightGray, None}}}]
```

## ASSUMPTION CHECK

The chi-squared approximation requires that no expected count falls below 1 and that at most 20% of cells have an expected count below 5.

```wl
In[]:= expectedFlat = Flatten[expected];
Dataset[<|
  "Minimum expected count" -> Min[expectedFlat],
  "Cells with expected < 5" -> Count[expectedFlat, x_ /; x < 5],
  "Percent of cells with expected < 5" -> N[100 Count[expectedFlat, x_ /; x < 5]/Length[expectedFlat]],
  "Chi-squared appropriate" -> Min[expectedFlat] >= 1 && N[Count[expectedFlat, x_ /; x < 5]/Length[expectedFlat]] <= 0.2|>]
```

Report the outcome in a text cell. This project's datasets are large enough that Fisher's exact test is not needed; if the check ever fails, say so explicitly rather than substituting a different test silently.

## CHI-SQUARED TEST OF INDEPENDENCE

```wl
In[]:= chiSq = Total[Flatten[(observed - expected)^2/expected]];
dfChi = (Length[rowLevels] - 1) (Length[colLevels] - 1);
pValue = 1 - CDF[ChiSquareDistribution[dfChi], chiSq];
Dataset[<|"Chi-squared" -> chiSq, "Degrees of freedom" -> dfChi,
  "p-value" -> pValue, "Significant at 0.05" -> pValue < 0.05|>]
```

## EFFECT SIZE AND RESIDUALS

Cramér's V measures association strength on a 0–1 scale, independent of sample size.

```wl
In[]:= cramersV = Sqrt[chiSq/(n Min[Length[rowLevels] - 1, Length[colLevels] - 1])];
Dataset[<|"Cramer's V" -> cramersV|>]
```

Standardized residuals identify which cells drive an association. Values beyond $\pm 2$ mark cells contributing substantially more than expected.

```wl
In[]:= stdResiduals = (observed - expected)/Sqrt[expected];
Grid[Join[
  {Join[{""}, colLabels]},
  Table[Join[{rowLabels[[i]]}, NumberForm[#, {4, 2}] & /@ stdResiduals[[i]]],
   {i, Length[rowLevels]}]],
 Frame -> All]
```

Only interpret residuals when the omnibus test is significant.

## VISUALIZATION

Set `rowLabel` and `colLabel` from the dictionary's "Name for Plots" entries.

```wl
In[]:= BarChart[observed, ChartLayout -> "Grouped",
  ChartLabels -> {rowLabels, colLabels},
  ChartLegends -> colLabels, ChartStyle -> "Pastel",
  PlotLabel -> "Observed Counts",
  FrameLabel -> {rowLabel, "Count"}, ImageSize -> 450]
```

```wl
In[]:= BarChart[N[observed/rowTotals], ChartLayout -> "Stacked",
  ChartLabels -> rowLabels, ChartLegends -> colLabels,
  ChartStyle -> "Pastel", PlotLabel -> "Proportion within Each Row Category",
  FrameLabel -> {rowLabel, "Proportion"}, ImageSize -> 450]
```

A mosaic-style view of the residuals makes the driving cells obvious:

```wl
In[]:= MatrixPlot[stdResiduals,
  ColorFunction -> "TemperatureMap", ColorFunctionScaling -> False,
  FrameTicks -> {Thread[{Range[Length[rowLevels]], rowLabels}],
    Thread[{Range[Length[colLevels]], colLabels}]},
  PlotLegends -> Automatic, PlotLabel -> "Standardized Residuals",
  ImageSize -> 380]
```

## Reporting

Close with a text cell giving the chi-squared statistic, degrees of freedom, p-value, Cramér's V, the sample size, confirmation that the expected-count requirement was met, and a plain-language conclusion. When significant, name the cells whose standardized residuals exceeded $\pm 2$ and describe the direction of the association.
