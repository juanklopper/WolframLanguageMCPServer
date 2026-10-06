# Wolfram Data Analysis Notebooks

This folder is a single research project: one CSV dataset, one data dictionary, and the Mathematica (`.nb`) notebooks that analyze it. Notebooks are authored as markdown and converted with the Wolfram Language MCP tools (`WriteNotebook`, `ReadNotebook`, `WolframLanguageEvaluator`).

@DATA_DICTIONARY.md

The data dictionary is the source of truth for how each column is treated. Never infer a column's statistical role from the CSV alone.

## Notebook layout

Heading depth maps to cell style as follows (verified empirically — do not assume standard Markdown-to-Mathematica defaults):

| Markdown | Cell style |
|---|---|
| `#` | Title |
| `##` | Section |
| `###` | Subsection |
| `####` | Subsubsection |

**Every heading cell — Title, Section, Subsection, Subsubsection, at any depth — must be ALL CAPS.**

The first cell of every notebook is a Title cell.

## Required notebook structure

Every analysis notebook follows this skeleton:

1. `#` Title cell — the analysis name, in caps (e.g. `DATA ANALYSIS`).
2. `##` `LOAD DATA` — import the CSV as a `Dataset` using `FileNameJoin[{NotebookDirectory[], "file.csv"}]` (never a hardcoded absolute path), plus a dimensions/columns sanity check.
3. `##` `UNIVARIATE DATA ANALYSIS` — one `###` Subsection per column, in CSV column order. Every analysis begins here; no inferential work precedes it.
4. `##` `COMPARATIVE DATA ANALYSIS` — **stop and ask the user which column is the response (outcome) variable before building this section.** Never assume one. Then one `###` Subsection per column (CSV column order, excluding the response variable and `ID`), each comparing that column side by side across all categories of the response variable. Do **not** create one subsection per response category.

### Within each column subsection (UNIVARIATE)

- One code cell of summary statistics matching the column's role in the data dictionary: `Mean`/`Median`/`StandardDeviation`/`Quartiles`/`Min`/`Max` for continuous; `Counts` (and `KeySort[Counts[...]]` for ordinal) plus proportions for categorical/ordinal.
- One or more visualization cells: `Histogram` + `BoxWhiskerChart` for continuous; `BarChart` for categorical/ordinal.
- Identifier columns (e.g. `ID`) get a uniqueness/completeness check only — no descriptive or distributional statistics.

### Within each column subsection (COMPARATIVE)

- Split the data once, in a code cell directly under the `COMPARATIVE DATA ANALYSIS` Section heading: one filtered `Dataset` per response category, e.g. `interventionData = Select[data, #Group === "Intervention" &]`. Each column subsection then extracts that column from every category subset (`ageColIntervention`, `ageColPlacebo`, …).
- **Summary statistics — side by side in one cell.** Compute the same statistics as UNIVARIATE (matching the column's role) for every response category and show them together in a single `Dataset` table:
  - Continuous: one row per response category, one column per statistic (`Mean`, `Median`, `StandardDeviation`, `Quartiles`, `Min`, `Max`).
  - Categorical/ordinal: one row per level of the column (decoded label), with a count and a proportion column for each response category. Look up missing levels with a default of `0` so every category shows every level. Ordinal columns also get a mode/median table with one row per response category.
- **Visualizations — side by side, not one set per category.**
  - Continuous: one cell with one `Histogram` per response category placed side by side via `GraphicsRow`, then one `BoxWhiskerChart` containing a box for each response category (`ChartLabels` = decoded category names).
  - Categorical/ordinal: one grouped `BarChart` — a bar group per level of the column, one bar per response category, with `ChartLegends` naming the response categories.
- Side-by-side histograms share the same bin spec (reuse the UNIVARIATE bins) and the same `PlotRange`, with the count axis maximum computed across all categories (e.g. `Max[BinCounts[colA, bins], BinCounts[colB, bins]] + 1`) so distributions stay visually comparable.
- Label plots with the column's "Name for Plots" and the response variable's decoded category names (e.g. `"Diabetes Mellitus: Yes"`).

## Code cell conventions

- Code fences are tagged ` ```wl ` and prefixed with `In[]:=` (no cell number, no `Out[]=`) so cells import as **live, unevaluated Input cells** — notebooks are opened and evaluated in Mathematica, never pre-baked with static output.
- Assign each column to a short, descriptive variable (`ageCol`, `groupCounts`, …) rather than repeating `data[All, "..."]` inline.
- Always extract columns with `Normal[data[All, "ColumnName"]]` — the dataset is a `Dataset`, so raw lists are needed for statistics and plotting functions.
- Use the plot label from the data dictionary's "Name for Plots" column for axis and chart labels, and its "Decode" column for category tick labels.

## Mathematical typesetting

All rules below were verified by writing test notebooks and rendering them — do not substitute assumptions about how Markdown or LaTeX "normally" behave.

Use `$...$` for inline math and `$$...$$` for display math. Both convert to a **TeX assistant cell** (`TemplateBox[..., "TeXAssistantTemplate"]` in `TraditionalForm`) — exactly what Insert ▸ TeX (Ctrl-4) produces, editable as TeX and stable through `ReadNotebook`.

- **Never use Wolfram named-character escapes in text cells.** `\[Alpha]`, `\[Mu]` are not interpreted; they render literally as their source. Use `$\alpha$` or a literal Unicode `α`.
- **Never hand-write box structures** (`SuperscriptBox`, `FractionBox`, …). They render, but `ReadNotebook` flattens them to plain text, breaking round-trips.
- Literal Unicode (`α`, `μ`, `σ`, `≤`, `≠`) is fine for a bare symbol in prose. Switch to `$...$` as soon as the expression has structure.

Renders correctly: `\alpha`–`\omega`, `\frac`, `\sqrt`, `\sum` with limits, `\bar`, `\hat`, `\pm`, `\le`, `\ge`, `\ne`, `\sim`, `\colon`, `\left(...\right)`, `\text{}`, `\mathrm{}`, `pmatrix`, `cases`.

Known failures — use the replacement:

| Don't write | Write instead | Failure |
|---|---|---|
| `$95\%$` | `95%` as ordinary text | `\%` breaks delimiter detection; cell renders the literal source `$95\%$` |
| `\operatorname{Var}` | `\mathrm{Var}` | renders letter-spaced as the product `V a r` |
| `\begin{aligned}` | separate display equations | oversized column gaps, inline-size fractions |
| two adjacent `$$` blocks | one per paragraph, prose between | adjacent blocks merge into one cell and render side by side |
| `\mathcal{N}` | `\mathrm{N}` | calligraphic is dropped |

**Unknown commands fail silently.** `$\foobar{x} + \notacmd$` becomes `x +` with an empty `errors` field — a formula can lose a whole term invisibly. After every `WriteNotebook`, call `ReadNotebook` and confirm each formula came back as the LaTeX that was written.

`$$...$$` produces a **Text** cell holding a display-size TeX box, not a `DisplayFormula` cell. That is the default and is sufficient. Only if true `DisplayFormula` styling is needed, post-process: `nb /. Cell[TextData[c_Cell], "Text", ___] :> Cell[First[c], "DisplayFormula"]`.

`ReadNotebook` is lossy in two ways when re-editing: it drops the `In[]:=` prefix, and inline `$...$` returns as `$$...$$`. Re-apply both by hand.

## Inferential analysis

The structure above covers descriptive work only. Hypothesis tests and models live in skills — invoke the matching one rather than improvising:

| Question | Skill |
|---|---|
| Numerical variable, two independent groups | `compare-two-independent-groups` |
| Numerical variable, two paired/dependent measurements | `compare-two-dependent-groups` |
| Numerical variable, three or more independent groups | `compare-multiple-groups` |
| Numerical variable, three or more repeated measurements | `repeated-measures-anova` |
| Association between two categorical variables | `categorical-association` |
| Numerical outcome predicted by one or more variables | `linear-regression` |
| Binary outcome predicted by one or more variables | `logistic-regression` |

## When adding a new dataset

1. Add a `## <filename>.csv` section to `DATA_DICTIONARY.md` describing every column (type, statistical role, range/levels, notes, decode).
2. Build the notebook to the layout above, driven by that data dictionary.
