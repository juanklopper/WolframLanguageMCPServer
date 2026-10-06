---
name: logistic-regression
description: Fit and diagnose a binary logistic regression model in a Mathematica notebook. Covers LogitModelFit, odds ratios with confidence intervals, the likelihood ratio test, McFadden pseudo R-squared, classification performance with a confusion matrix, and a ROC curve with AUC. Use for predicting or modelling a binary/dichotomous outcome from one or more variables.
---

# Binary logistic regression

Adds a `##` section to an existing analysis notebook. Follow the project's `CLAUDE.md`: ALL CAPS headings, ` ```wl ` fences prefixed `In[]:=`, columns extracted with `Normal[data[All, "..."]]`.

## Selecting the columns

**Read the project's `DATA_DICTIONARY.md` before asking the user anything.** Offer only columns whose role fits:

| Role needed | Dictionary role |
|---|---|
| Outcome | categorical with **exactly two** levels, or binary |
| Predictors | continuous, or categorical (dummy coded — see `linear-regression`) |

Present the fitting candidates by name, with their "Name for Plots" label, and ask the user which column is the binary outcome and which are the predictors. Do not ask blind. Never offer an identifier column as a predictor. If the outcome the user names is continuous, redirect to `linear-regression`.

Then bind the choices once:

```wl
In[]:= (* The only cell carrying project-specific names. Replace both values. *)
outcomeCol = "EssentialHypertension";
predictorCols = {"Age", "Chol"};
```

Everything below refers only to `outcomeCol` and `predictorCols`. Take axis labels from the dictionary's "Name for Plots" entries.

### Coding the outcome

The outcome must be numeric 0/1. Check the dictionary's "Decode" entry and **state in a text cell which level is coded 1**, since every odds ratio is interpreted relative to that choice. If the column holds strings, recode explicitly and say which level was mapped to 1:

```wl
In[]:= positiveLevel = "Y";
outcomeNumeric = Boole[# === positiveLevel] & /@ Normal[data[All, outcomeCol]];
```

For a column already stored as 0/1, use `Round[N@Normal[data[All, outcomeCol]]]` instead.

## Section layout

```
## LOGISTIC REGRESSION OF <OUTCOME>
### MODEL SPECIFICATION
### MODEL FIT
### COEFFICIENTS AND ODDS RATIOS
### MODEL SIGNIFICANCE
### CLASSIFICATION PERFORMANCE
### ROC CURVE
### VISUALIZATION
```

## MODEL SPECIFICATION

State the model in a text cell before fitting:

> $$\mathrm{logit}(\pi) = \ln\left(\frac{\pi}{1-\pi}\right) = \beta_0 + \beta_1 x_1 + \beta_2 x_2$$
>
> where $\pi$ is the probability that <outcome> equals 1.

The outcome goes in the **last** column of the design matrix.

```wl
In[]:= designMatrix = N@Transpose[Append[
    Table[Normal[data[All, c]], {c, predictorCols}],
    outcomeNumeric]];
outcomeValues = Round[designMatrix[[All, -1]]];
Dataset[<|"n" -> Length[designMatrix],
  "Events (coded 1)" -> Total[outcomeValues],
  "Non-events (coded 0)" -> Length[outcomeValues] - Total[outcomeValues],
  "Event rate" -> N@Mean[outcomeValues]|>]
```

Check that the outcome really is binary before fitting, and that there are enough events — roughly ten events per predictor is the usual minimum.

```wl
In[]:= vars = Array[Symbol["x" <> ToString[#]] &, Length[predictorCols]];
model = LogitModelFit[designMatrix, vars, vars];
Normal[model]
```

## MODEL FIT

Use the built-in `LikelihoodRatioIndex` — it **is** McFadden's pseudo $R^2$. Do not build an intercept-only null model by hand; a constant column triggers `LogitModelFit::rank` and returns a degenerate fit.

```wl
In[]:= Dataset[<|
  "Log-likelihood" -> model["LogLikelihood"],
  "Null deviance" -> model["NullDeviance"],
  "Residual deviance" -> model["ResidualDeviance"],
  "McFadden pseudo R-squared" -> model["LikelihoodRatioIndex"],
  "Adjusted pseudo R-squared" -> model["AdjustedLikelihoodRatioIndex"],
  "AIC" -> model["AIC"],
  "BIC" -> model["BIC"]|>]
```

McFadden values are much smaller than ordinary $R^2$; 0.2–0.4 already indicates good fit. Say so when reporting, so the number is not misread.

## COEFFICIENTS AND ODDS RATIOS

```wl
In[]:= model["ParameterTable"]
```

Coefficients are on the log-odds scale. Exponentiate for odds ratios, which is what should be reported.

```wl
In[]:= entries = model["ParameterTableEntries"];
cis = model["ParameterConfidenceIntervals"];
labels = Prepend[predictorCols, "(Intercept)"];
Dataset[AssociationThread[labels -> MapThread[
   <|"Coefficient" -> #1, "Odds ratio" -> Exp[#1],
     "OR 95% CI" -> Exp[#2], "p-value" -> #3|> &,
   {entries[[All, 1]], cis, entries[[All, 4]]}]]]
```

Interpretation: an odds ratio of 1 means no effect. An odds ratio above 1 means the odds of the outcome increase per one-unit rise in that predictor, holding the others constant. **A confidence interval containing 1 means the predictor is not significant** — this is the log-odds interval containing 0, exponentiated.

Report the intercept's odds ratio only if it is meaningful; it is the odds when every predictor is zero, which is often outside the data range.

## MODEL SIGNIFICANCE

The likelihood ratio test compares the fitted model against the intercept-only model.

```wl
In[]:= lrStat = model["LikelihoodRatioStatistic"];
lrDf = Length[predictorCols];
Dataset[<|"Likelihood ratio statistic" -> lrStat,
  "Degrees of freedom" -> lrDf,
  "p-value" -> 1 - CDF[ChiSquareDistribution[lrDf], lrStat]|>]
```

## CLASSIFICATION PERFORMANCE

`"PredictedResponse"` gives fitted probabilities. Note that `"EstimatedResponse"` is **not** a valid property and raises `FittedModel::notprop`. For a prediction at new covariate values, call the model directly — `model[60, 230]` — rather than substituting with `/.`, which does not work on a `FittedModel`.

```wl
In[]:= probabilities = model["PredictedResponse"];
predictedClass = Boole[# >= 0.5] & /@ probabilities;
```

Build the confusion matrix against `Round`ed outcomes. Matching real-valued `0.`/`1.` against integer patterns silently produces a matrix of zeros.

```wl
In[]:= confusion = Table[Count[Transpose[{outcomeValues, predictedClass}], {a, b}],
   {a, {0, 1}}, {b, {0, 1}}];
Grid[{{"", "Predicted 0", "Predicted 1"},
   {"Actual 0", confusion[[1, 1]], confusion[[1, 2]]},
   {"Actual 1", confusion[[2, 1]], confusion[[2, 2]]}}, Frame -> All]
```

```wl
In[]:= {tn, fp, fn, tp} = Flatten[confusion];
Dataset[<|
  "Accuracy" -> N[(tp + tn)/Total[Flatten[confusion]]],
  "Sensitivity (recall)" -> N[tp/(tp + fn)],
  "Specificity" -> N[tn/(tn + fp)],
  "Positive predictive value" -> N[tp/(tp + fp)],
  "Negative predictive value" -> N[tn/(tn + fn)]|>]
```

Compare accuracy against the event rate — a model that never beats always predicting the majority class has no practical value regardless of its p-values.

## ROC CURVE

Mathematica has no built-in ROC function for `FittedModel`. Sweep the threshold and integrate by trapezoid; this has been verified to return approximately 0.5 for random scores.

```wl
In[]:= rocPoints[yv_, pv_] := Module[{thresholds, pos, neg},
   pos = Total[yv]; neg = Length[yv] - pos;
   thresholds = Prepend[Append[Sort[DeleteDuplicates[pv], Greater], -0.01], 1.01];
   Table[{N[Count[Transpose[{yv, pv}], {0, q_ /; q >= t}]/neg],
     N[Count[Transpose[{yv, pv}], {1, q_ /; q >= t}]/pos]}, {t, thresholds}]];
auc[pts_] := Total[Table[
    (pts[[i + 1, 1]] - pts[[i, 1]]) (pts[[i + 1, 2]] + pts[[i, 2]])/2,
    {i, Length[pts] - 1}]];
roc = rocPoints[outcomeValues, probabilities];
areaUnderCurve = auc[roc];
Dataset[<|"AUC" -> areaUnderCurve|>]
```

```wl
In[]:= Show[
 ListLinePlot[roc, PlotStyle -> Thick,
  AxesLabel -> {"1 - Specificity", "Sensitivity"}],
 Plot[x, {x, 0, 1}, PlotStyle -> {Gray, Dashed}],
 PlotLabel -> "ROC Curve (AUC = " <> ToString[NumberForm[areaUnderCurve, {4, 3}]] <> ")",
 AspectRatio -> 1, PlotRange -> {{0, 1}, {0, 1}}, ImageSize -> 400]
```

An AUC of 0.5 is chance; 0.7–0.8 is acceptable discrimination, above 0.8 is good.

## VISUALIZATION

Set `outcomeLabel` and `predictorLabels` from the dictionary's "Name for Plots" entries.

For a single continuous predictor, plot the fitted probability curve against the observed outcomes, holding the other predictors at their means:

```wl
In[]:= otherMeans = Mean /@ Transpose[designMatrix[[All, 2 ;; -2]]];
Show[
 ListPlot[Transpose[{designMatrix[[All, 1]], outcomeValues}],
  PlotStyle -> Directive[Opacity[0.4], PointSize[Medium]]],
 Plot[model @@ Prepend[otherMeans, x], {x,
   Min[designMatrix[[All, 1]]], Max[designMatrix[[All, 1]]]},
  PlotStyle -> Red],
 PlotLabel -> "Fitted Probability vs " <> First[predictorLabels],
 AxesLabel -> {First[predictorLabels], "P(" <> outcomeLabel <> " = 1)"},
 PlotRange -> {{Automatic, Automatic}, {-0.05, 1.05}}, ImageSize -> 450]
```

Holding the other predictors at their means is what makes the curve a valid slice of the fitted surface — state that in the text cell. With a single predictor, `otherMeans` is empty and `model @@ {x}` is just `model[x]`.

```wl
In[]:= Histogram[{Pick[probabilities, outcomeValues, 0],
   Pick[probabilities, outcomeValues, 1]}, Automatic,
  ChartLegends -> {"Actual 0", "Actual 1"},
  PlotLabel -> "Predicted Probability by Actual Outcome",
  FrameLabel -> {"Predicted probability", "Count"}, ImageSize -> 450]
```

Good separation between the two histograms is the visual counterpart of a high AUC.

## Reporting

Close with a text cell giving each predictor's odds ratio with its confidence interval and p-value, the likelihood ratio test result, McFadden's pseudo $R^2$ (noting that it is not comparable to ordinary $R^2$), accuracy, sensitivity, specificity, AUC, the sample size and event count, and a plain-language interpretation of each significant predictor in the units of the data.
