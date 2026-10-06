---
name: linear-regression
description: Fit and diagnose a linear regression model in a Mathematica notebook for a continuous outcome. Covers simple and multiple regression, dummy coding of categorical predictors, the full assumption battery (linearity, normality of residuals, homoscedasticity, independence, multicollinearity), influential-point detection, and reporting. Use for predicting or modelling a numerical outcome from one or more variables.
---

# Linear regression

Adds a `##` section to an existing analysis notebook. Follow the project's `CLAUDE.md`: ALL CAPS headings, ` ```wl ` fences prefixed `In[]:=`, columns extracted with `Normal[data[All, "..."]]`.

## Selecting the columns

**Read the project's `DATA_DICTIONARY.md` before asking the user anything.** Offer only columns whose role fits:

| Role needed | Dictionary role |
|---|---|
| Outcome | continuous |
| Predictors | continuous, or categorical (dummy coded — see below) |

Present the fitting candidates by name, with their "Name for Plots" label, and ask the user which column is the outcome and which are the predictors. Do not ask blind. Never offer an identifier column as a predictor. If the outcome the user names is binary, redirect to `logistic-regression`.

Then bind the choices once:

```wl
In[]:= (* The only cell carrying project-specific names. Replace both values. *)
outcomeCol = "Chol";
predictorCols = {"Age", "DBP"};
```

Everything below refers only to `outcomeCol` and `predictorCols`. Take axis labels from the dictionary's "Name for Plots" entries.

## Section layout

```
## LINEAR REGRESSION OF <OUTCOME>
### MODEL SPECIFICATION
### MODEL FIT
### COEFFICIENTS
### MODEL SIGNIFICANCE
### ASSUMPTION DIAGNOSTICS
### INFLUENTIAL OBSERVATIONS
### VISUALIZATION
```

## MODEL SPECIFICATION

State the model in a text cell before fitting:

> $$y = \beta_0 + \beta_1 x_1 + \beta_2 x_2 + \varepsilon$$
>
> where $\varepsilon \sim \mathrm{N}(0, \sigma^2)$.

Build the design matrix with the outcome as the **last** column — that is the layout `LinearModelFit` expects.

```wl
In[]:= designMatrix = N@Transpose[Append[
    Table[Normal[data[All, c]], {c, predictorCols}],
    Normal[data[All, outcomeCol]]]];
Dimensions[designMatrix]
```

```wl
In[]:= vars = Array[Symbol["x" <> ToString[#]] &, Length[predictorCols]];
model = LinearModelFit[designMatrix, vars, vars];
Normal[model]
```

### Categorical predictors

`LinearModelFit` does not dummy-code automatically. Convert a k-level categorical predictor into k−1 indicator columns, holding out a reference level, and say which level is the reference in a text cell.

```wl
In[]:= catCol = "Group";
catLevels = Sort[DeleteDuplicates[Normal[data[All, catCol]]]];
referenceLevel = First[catLevels];
dummies = Transpose[Table[
    Boole[# === lev] & /@ Normal[data[All, catCol]], {lev, Rest[catLevels]}]];
```

Each dummy coefficient is then the mean difference from the reference level, holding other predictors constant.

## MODEL FIT

```wl
In[]:= Dataset[<|
  "R-squared" -> model["RSquared"],
  "Adjusted R-squared" -> model["AdjustedRSquared"],
  "Residual standard error" -> Sqrt[model["EstimatedVariance"]],
  "AIC" -> model["AIC"],
  "BIC" -> model["BIC"],
  "n" -> Length[designMatrix]|>]
```

Prefer adjusted $R^2$ when comparing models with different numbers of predictors.

## COEFFICIENTS

```wl
In[]:= model["ParameterTable"]
```

```wl
In[]:= model["ParameterConfidenceIntervalTable"]
```

The table rows are the intercept followed by the predictors in the order of `predictorCols`. Columns are estimate, standard error, t-statistic, and p-value. Interpret each slope as the expected change in the outcome per one-unit increase in that predictor, holding the others constant.

## MODEL SIGNIFICANCE

```wl
In[]:= model["ANOVATable"]
```

The F-ratio row tests $H_0: \beta_1 = \beta_2 = \cdots = 0$ — whether the model explains more variance than the intercept alone.

## ASSUMPTION DIAGNOSTICS

Five assumptions, each with an explicit check.

### Linearity and homoscedasticity

Residuals against fitted values should show no pattern and constant vertical spread.

```wl
In[]:= fitted = model["PredictedResponse"];
residuals = model["FitResiduals"];
ListPlot[Transpose[{fitted, residuals}],
  PlotLabel -> "Residuals vs Fitted Values",
  AxesLabel -> {"Fitted", "Residual"},
  Epilog -> {Red, Line[{{Min[fitted], 0}, {Max[fitted], 0}}]},
  ImageSize -> 450]
```

A funnel shape indicates heteroscedasticity; curvature indicates a missing nonlinear term. Mathematica has no built-in Breusch-Pagan test, so assess this graphically and describe what the plot shows.

### Normality of residuals

```wl
In[]:= Dataset[<|"Shapiro-Wilk on residuals" -> ShapiroWilkTest[residuals, "PValue"]|>]
```

```wl
In[]:= QuantilePlot[residuals, NormalDistribution[0, StandardDeviation[residuals]],
  PlotLabel -> "Normal Q-Q Plot of Residuals",
  AxesLabel -> {"Theoretical Quantiles", "Sample Quantiles"}, ImageSize -> 400]
```

```wl
In[]:= Histogram[residuals, Automatic, PlotLabel -> "Distribution of Residuals",
  FrameLabel -> {"Residual", "Count"}, ImageSize -> 400]
```

### Independence of residuals

```wl
In[]:= Dataset[<|"Durbin-Watson d" -> model["DurbinWatsonD"]|>]
```

Values near 2 indicate no autocorrelation; below 1 or above 3 are cause for concern. This matters mainly for time-ordered data.

### Multicollinearity

```wl
In[]:= vifs = model["VarianceInflationFactors"];
Dataset[AssociationThread[Prepend[predictorCols, "(Intercept)"] -> vifs]]
```

**The first element corresponds to the intercept — ignore it.** Read VIFs for the predictors from the second element onward. A VIF above 5 warrants comment; above 10 indicates serious multicollinearity. Omit this check entirely for simple regression with one predictor.

## INFLUENTIAL OBSERVATIONS

```wl
In[]:= cooks = model["CookDistances"];
threshold = 4./Length[designMatrix];
influential = Flatten[Position[cooks, d_ /; d > threshold]];
Dataset[<|"Threshold (4/n)" -> threshold,
  "Count exceeding" -> Length[influential],
  "Row indices" -> influential|>]
```

```wl
In[]:= ListPlot[cooks, Filling -> Axis, PlotLabel -> "Cook's Distance by Observation",
  AxesLabel -> {"Observation", "Cook's D"},
  Epilog -> {Red, Dashed, Line[{{0, threshold}, {Length[cooks], threshold}}]},
  ImageSize -> 450]
```

Report influential points; do not remove them without a substantive reason, and state the reason if you do.

## VISUALIZATION

Set `outcomeLabel` and `predictorLabels` from the dictionary's "Name for Plots" entries.

For simple regression (one predictor), plot the fit with confidence bands:

```wl
In[]:= bands = model["MeanPredictionBands"];
Show[
 ListPlot[designMatrix[[All, {1, -1}]], PlotStyle -> Opacity[0.5]],
 Plot[Evaluate[{model[x], bands /. First[vars] -> x}], {x,
   Min[designMatrix[[All, 1]]], Max[designMatrix[[All, 1]]]},
  PlotStyle -> {Red, {Gray, Dashed}, {Gray, Dashed}}],
 PlotLabel -> "Fitted Line with 95% Mean Prediction Bands",
 AxesLabel -> {First[predictorLabels], outcomeLabel}, ImageSize -> 450]
```

For multiple regression, plot observed against predicted:

```wl
In[]:= Show[
 ListPlot[Transpose[{fitted, designMatrix[[All, -1]]}], PlotStyle -> Opacity[0.6]],
 Plot[x, {x, Min[fitted], Max[fitted]}, PlotStyle -> {Red, Dashed}],
 PlotLabel -> "Observed vs Predicted",
 AxesLabel -> {"Predicted " <> outcomeLabel, "Observed " <> outcomeLabel},
 ImageSize -> 420]
```

## Reporting

Close with a text cell giving the fitted equation with coefficient estimates, each coefficient's confidence interval and p-value, $R^2$ and adjusted $R^2$, the overall F-test result, the sample size, an explicit statement of which assumptions held and which did not, and a plain-language interpretation of each significant predictor in the units of the data.
