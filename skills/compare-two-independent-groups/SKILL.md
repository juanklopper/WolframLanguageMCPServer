---
name: compare-two-independent-groups
description: Compare a numerical variable between two independent groups in a Mathematica notebook. Tests normality and equality of variance first, then runs Student's t-test, Welch's t-test, or the Mann-Whitney U test according to which assumptions hold. Use for two-sample/unpaired/between-subjects comparisons of a continuous outcome.
---

# Comparing a numerical variable between two independent groups

Adds a `##` section to an existing analysis notebook. Follow the project's `CLAUDE.md`: ALL CAPS headings, ` ```wl ` fences prefixed `In[]:=`, columns extracted with `Normal[data[All, "..."]]`.

## Selecting the columns

**Read the project's `DATA_DICTIONARY.md` before asking the user anything.** It is the source of truth for each column's statistical role. Offer only columns whose role fits:

| Role needed | Dictionary role |
|---|---|
| Outcome | continuous |
| Grouping variable | categorical/nominal with **exactly two** levels |

Present the fitting candidates by name, with their "Name for Plots" label, and ask the user to choose one of each. Do not ask blind, and do not offer a column whose role does not fit. If no column has the required role, say so rather than proceeding with the wrong type. If the grouping column has three or more levels, use `compare-multiple-groups` instead.

Then bind the choices once:

```wl
In[]:= (* The only cell carrying project-specific names. Replace both values. *)
outcomeCol = "Chol";
groupCol = "Group";
```

Everything below refers only to `outcomeCol` and `groupCol`, so no other cell changes between projects. Take axis and chart labels from the dictionary's "Name for Plots" entry for each chosen column, and level labels from its "Decode" entry.

## Decision rule

```
Shapiro-Wilk on each group
├── both p ≥ 0.05  →  variances equal? (Conover / VarianceEquivalenceTest)
│   ├── p ≥ 0.05  →  Student's t-test (pooled variance)
│   └── p < 0.05  →  Welch's t-test (unequal variance)
└── either p < 0.05  →  Mann-Whitney U test
```

## Section layout

```
## COMPARISON OF <OUTCOME> BETWEEN <GROUP> CATEGORIES
### HYPOTHESES
### DESCRIPTIVE STATISTICS BY GROUP
### ASSUMPTION TESTS
### HYPOTHESIS TEST
### EFFECT SIZE AND CONFIDENCE INTERVAL
### VISUALIZATION
```

## HYPOTHESES

A text cell, with the null stated in LaTeX. Percent signs stay outside math (see `CLAUDE.md`):

> We test whether mean <outcome> differs between the two groups at the 5% significance level.
>
> $$H_0 \colon \mu_1 = \mu_2$$
>
> $$H_1 \colon \mu_1 \ne \mu_2$$

If the test falls back to Mann-Whitney, restate the hypotheses in terms of distributions/medians rather than means.

## DESCRIPTIVE STATISTICS BY GROUP

```wl
In[]:= outcomeValues = Normal[data[All, outcomeCol]];
levels = DeleteDuplicates[Normal[data[All, groupCol]]];
groups = Table[
   Normal[data[Select[#[groupCol] === lev &], outcomeCol]], {lev, levels}];
{grp1, grp2} = groups;
Dataset[AssociationThread[levels -> Map[
   <|"n" -> Length[#], "Mean" -> N@Mean[#], "SD" -> N@StandardDeviation[#],
     "Median" -> N@Median[#], "Q1" -> Quantile[N@#, 0.25], "Q3" -> Quantile[N@#, 0.75],
     "Min" -> Min[#], "Max" -> Max[#]|> &, groups]]]
```

Confirm `Length[levels] == 2` before continuing.

## ASSUMPTION TESTS

`ShapiroWilkTest` and `VarianceEquivalenceTest` both accept a `"PValue"` property. `VarianceEquivalenceTest` uses the Conover test by default, which does not itself require normality.

```wl
In[]:= normality = AssociationThread[levels -> (ShapiroWilkTest[#, "PValue"] & /@ groups)];
varEquality = VarianceEquivalenceTest[groups, "PValue"];
Dataset[<|"Shapiro-Wilk by group" -> normality,
  "Variance equivalence (Conover)" -> varEquality|>]
```

Add a text cell stating, in words, which branch of the decision rule these p-values select.

## HYPOTHESIS TEST

`LocationTest` with its default `VerifyTestAssumptions -> Automatic` already implements the decision rule above — it checks normality and variance equality internally and selects Student's t, Welch's t, or Mann-Whitney. The `"TestDataTable"` property reports **which test was chosen**, so it is self-documenting.

```wl
In[]:= LocationTest[groups, 0, "TestDataTable"]
```

```wl
In[]:= LocationTest[groups, 0, {"TestStatistic", "PValue", "ShortTestConclusion"}]
```

Confirm the test named in the table matches the branch selected by the explicit assumption tests, and say so in a text cell.

### Forcing a specific test

`LocationTest` uses a 0.025 internal threshold for its normality screen, so it can occasionally disagree with Shapiro-Wilk read at 0.05. When the explicit assumption tests and the automatic choice disagree, force the correct test:

```wl
In[]:= (* Student's t, pooled variance *)
LocationTest[groups, 0, "TestDataTable", VerifyTestAssumptions -> None]
```

```wl
In[]:= (* Mann-Whitney U *)
MannWhitneyTest[groups, 0, "TestDataTable"]
```

Welch's test has no forcing switch — `Method -> "Welch"` is silently ignored, and `VerifyTestAssumptions -> None` returns the pooled test. Compute it directly when it must be forced:

```wl
In[]:= welch[a_, b_] := Module[{na, nb, va, vb, tStat, df},
   na = Length[a]; nb = Length[b];
   va = Variance[N@a]; vb = Variance[N@b];
   tStat = (Mean[N@a] - Mean[N@b])/Sqrt[va/na + vb/nb];
   df = (va/na + vb/nb)^2/((va/na)^2/(na - 1) + (vb/nb)^2/(nb - 1));
   <|"t" -> tStat, "df" -> df,
     "PValue" -> 2 (1 - CDF[StudentTDistribution[df], Abs[tStat]])|>];
welch[grp1, grp2]
```

Never call `TTest` directly — it aborts with `TTest::nortst` whenever its normality screen fails, rather than returning a result.

## EFFECT SIZE AND CONFIDENCE INTERVAL

Report Cohen's *d* alongside a parametric test, or the rank-biserial correlation alongside Mann-Whitney.

```wl
In[]:= pooledSD = Module[{n1 = Length[grp1], n2 = Length[grp2]},
   Sqrt[((n1 - 1) Variance[N@grp1] + (n2 - 1) Variance[N@grp2])/(n1 + n2 - 2)]];
cohenD = (Mean[N@grp1] - Mean[N@grp2])/pooledSD;
ci = Module[{n1 = Length[grp1], n2 = Length[grp2], se, tc},
   se = pooledSD Sqrt[1/n1 + 1/n2];
   tc = Quantile[StudentTDistribution[n1 + n2 - 2], 0.975];
   (Mean[N@grp1] - Mean[N@grp2]) + {-1, 1} tc se];
<|"Cohen's d" -> cohenD, "Mean difference" -> Mean[N@grp1] - Mean[N@grp2],
  "95% CI" -> ci|>
```

## VISUALIZATION

Set `outcomeLabel` and `groupLabel` from the data dictionary's "Name for Plots" entries.

```wl
In[]:= BoxWhiskerChart[groups, "Outliers",
  ChartLabels -> levels, ChartStyle -> "Pastel",
  PlotLabel -> outcomeLabel <> " by " <> groupLabel,
  FrameLabel -> {groupLabel, outcomeLabel}, ImageSize -> 400]
```

```wl
In[]:= binSpec = {Min[outcomeValues], Max[outcomeValues],
   (Max[outcomeValues] - Min[outcomeValues])/12};
Histogram[groups, binSpec, ChartLegends -> levels,
  PlotLabel -> "Distribution by " <> groupLabel,
  FrameLabel -> {outcomeLabel, "Count"}, ImageSize -> 450]
```

## Reporting

Close the section with a text cell giving the test used, the test statistic, degrees of freedom where applicable, the p-value, the effect size, and a plain-language conclusion referred back to the hypotheses. State the significance level explicitly.
