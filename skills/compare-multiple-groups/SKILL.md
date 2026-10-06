---
name: compare-multiple-groups
description: Compare a numerical variable across three or more independent groups in a Mathematica notebook. Tests normality and equality of variance, then runs one-way ANOVA with post-hoc pairwise tests, or the Kruskal-Wallis test when assumptions fail. Use for multi-group/one-way/between-subjects comparisons of a continuous outcome.
---

# Comparing a numerical variable across three or more independent groups

Adds a `##` section to an existing analysis notebook. Follow the project's `CLAUDE.md`: ALL CAPS headings, ` ```wl ` fences prefixed `In[]:=`, columns extracted with `Normal[data[All, "..."]]`.

## Selecting the columns

**Read the project's `DATA_DICTIONARY.md` before asking the user anything.** Offer only columns whose role fits:

| Role needed | Dictionary role |
|---|---|
| Outcome | continuous |
| Grouping variable | categorical/nominal or ordinal with **three or more** levels |

Present the fitting candidates by name, with their "Name for Plots" label, and ask the user to choose one of each. Do not ask blind. If the grouping column has exactly two levels, use `compare-two-independent-groups` instead.

Then bind the choices once:

```wl
In[]:= (* The only cell carrying project-specific names. Replace both values. *)
outcomeCol = "Chol";
groupCol = "Survey";
```

Everything below refers only to `outcomeCol` and `groupCol`. Take axis and chart labels from the dictionary's "Name for Plots" entry, and level labels from its "Decode" entry.

## Decision rule

```
Shapiro-Wilk on each group
├── all p ≥ 0.05  →  variances equal? (Conover / VarianceEquivalenceTest)
│   ├── p ≥ 0.05  →  one-way ANOVA  →  post-hoc if significant
│   └── p < 0.05  →  Kruskal-Wallis (or Welch ANOVA, not built in)
└── any p < 0.05  →  Kruskal-Wallis
```

## Section layout

```
## COMPARISON OF <OUTCOME> ACROSS <GROUP> CATEGORIES
### HYPOTHESES
### DESCRIPTIVE STATISTICS BY GROUP
### ASSUMPTION TESTS
### OMNIBUS TEST
### POST-HOC PAIRWISE COMPARISONS
### VISUALIZATION
```

## HYPOTHESES

> We test whether mean <outcome> differs across the <k> groups at the 5% significance level.
>
> $$H_0 \colon \mu_1 = \mu_2 = \cdots = \mu_k$$
>
> $H_1$: at least one group mean differs from the others.

Restate in terms of distributions if the analysis falls back to Kruskal-Wallis.

## GROUPING THE DATA

Two shapes are needed downstream: a **list of groups** for the test functions, and a **two-column long form** for the ANOVA package.

```wl
In[]:= levels = Sort[DeleteDuplicates[Normal[data[All, groupCol]]]];
groups = Table[
   Normal[data[Select[#[groupCol] === lev &], outcomeCol]], {lev, levels}];
longForm = Transpose[{Normal[data[All, groupCol]], Normal[data[All, outcomeCol]]}];
{Length[groups], Length /@ groups}
```

## DESCRIPTIVE STATISTICS BY GROUP

```wl
In[]:= Dataset[AssociationThread[levels -> Map[
   <|"n" -> Length[#], "Mean" -> N@Mean[#], "SD" -> N@StandardDeviation[#],
     "Median" -> N@Median[#], "Min" -> Min[#], "Max" -> Max[#]|> &, groups]]]
```

## ASSUMPTION TESTS

```wl
In[]:= normality = AssociationThread[levels -> (ShapiroWilkTest[#, "PValue"] & /@ groups)];
varEquality = VarianceEquivalenceTest[groups, "PValue"];
Dataset[<|"Shapiro-Wilk by group" -> normality,
  "Variance equivalence (Conover)" -> varEquality|>]
```

State in a text cell which branch these p-values select.

## OMNIBUS TEST

`LocationEquivalenceTest` with its default `VerifyTestAssumptions -> Automatic` implements the decision rule: it returns **K-Sample T** (one-way ANOVA) when the groups pass its normality screen, and **Kruskal-Wallis** when they do not. `"TestDataTable"` names the chosen test.

```wl
In[]:= LocationEquivalenceTest[groups, "TestDataTable"]
```

```wl
In[]:= LocationEquivalenceTest[groups, {"TestStatistic", "PValue", "DegreesOfFreedom"}]
```

There is **no built-in `KruskalWallisTest` function** — it exists only as a method inside `LocationEquivalenceTest`. Likewise `Method -> "KruskalWallis"` is rejected as invalid and silently falls back to `Automatic`. The only supported override is:

```wl
In[]:= (* force one-way ANOVA *)
LocationEquivalenceTest[groups, "TestDataTable", VerifyTestAssumptions -> None]
```

If Shapiro-Wilk read at 0.05 disagrees with the automatic choice (`LocationEquivalenceTest` screens at 0.025), report both results and note the discrepancy rather than hand-rolling a Kruskal-Wallis statistic — a hand-rolled version will disagree with the built-in on tie correction.

### Full ANOVA table

`LocationEquivalenceTest` reports only the statistic and p-value. For a conventional ANOVA table with sums of squares and mean squares, use the legacy `ANOVA` package:

```wl
In[]:= Needs["ANOVA`"];
anovaResult = ANOVA[longForm, {group}, {group}];
ANOVA /. anovaResult
```

```wl
In[]:= CellMeans /. anovaResult
```

The table columns are `DF`, `SumOfSq`, `MeanSq`, `FRatio`, `PValue`, with rows for the factor, `Error`, and `Total`. Its `FRatio` matches the `K-Sample T` statistic from `LocationEquivalenceTest`.

## POST-HOC PAIRWISE COMPARISONS

Run only when the omnibus test is significant; say so explicitly in a text cell if it is not.

```wl
In[]:= anovaPost = ANOVA[longForm, {group}, {group},
   PostTests -> {Tukey, Bonferroni}];
PostTests /. anovaPost
```

The result lists the group-index pairs that differ significantly, e.g. `{{1, 2}, {1, 3}, {2, 3}}`. **An empty list `{""}` means no pair reached significance** — which is what you get whenever the omnibus test was not significant. Map indices back to `levels` when reporting. Available post-hoc tests: `Tukey`, `Bonferroni`, `Duncan`, `Dunnett`, `StudentNewmanKeuls`.

After Kruskal-Wallis, do pairwise Mann-Whitney tests with a Bonferroni-adjusted threshold instead:

```wl
In[]:= pairs = Subsets[Range[Length[groups]], {2}];
adjusted = 0.05/Length[pairs];
Dataset[Table[
  With[{p = MannWhitneyTest[{groups[[pr[[1]]]], groups[[pr[[2]]]]}, 0, "PValue"]},
   <|"Pair" -> levels[[pr]], "p" -> p, "Significant" -> p < adjusted|>], {pr, pairs}]]
```

## VISUALIZATION

Set `outcomeLabel` and `groupLabel` from the data dictionary's "Name for Plots" entries, and `levelLabels` from its "Decode" entry (falling back to `levels` when there is no decode).

```wl
In[]:= BoxWhiskerChart[groups, "Outliers", ChartLabels -> levelLabels,
  ChartStyle -> "Pastel", PlotLabel -> outcomeLabel <> " by " <> groupLabel,
  FrameLabel -> {groupLabel, outcomeLabel}, ImageSize -> 500]
```

```wl
In[]:= means = N@Mean /@ groups;
errors = N[StandardDeviation[#]/Sqrt[Length[#]] & /@ groups];
BarChart[MapThread[Around, {means, errors}], ChartLabels -> levelLabels,
  PlotLabel -> "Group Means with Standard Error",
  FrameLabel -> {groupLabel, "Mean " <> outcomeLabel}, ImageSize -> 450]
```

## Reporting

Close with a text cell giving the omnibus test used, its statistic, degrees of freedom, p-value, which pairs differed in post-hoc testing (mapped back to level names), and a plain-language conclusion tied to the hypotheses.
