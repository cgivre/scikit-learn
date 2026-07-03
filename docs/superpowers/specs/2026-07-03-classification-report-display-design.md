# ClassificationReportDisplay — Design

**Date:** 2026-07-03
**Goal:** Add a visual (heatmap) rendering of a classification report to scikit-learn,
analogous to `ConfusionMatrixDisplay`, so users don't need Yellowbrick.
**Target:** Upstream PR to scikit-learn (full contribution standards).

## Motivation

scikit-learn ships `ConfusionMatrixDisplay` but has no visual for classification
reports. Yellowbrick's `ClassificationReport` fills this gap externally. Closed issue
#16880 shows users specifically wanting heatmap colors that reflect classification
performance (precision/recall/f1) rather than raw counts. This Display provides exactly
that.

## Placement & Public API

- New file: `sklearn/metrics/_plot/classification_report.py`
- New class: `ClassificationReportDisplay`
- Exported as `sklearn.metrics.ClassificationReportDisplay`:
  - add `from sklearn.metrics._plot.classification_report import ClassificationReportDisplay`
    to `sklearn/metrics/__init__.py`
  - add `"ClassificationReportDisplay"` to `__all__`
- Mirrors the existing `ConfusionMatrixDisplay` structure exactly (constructor stores
  data as attributes; `from_*` classmethods build it; `plot()` draws).

### Entry points

1. **`ClassificationReportDisplay(report, *, display_labels=None)`** — primary use case.
   - `report`: dict as returned by
     `classification_report(y_true, y_pred, output_dict=True)`.
   - Only the `output_dict` form is accepted (no plain-text-string parsing).
   - `display_labels`: optional row labels override; defaults to the class keys found in
     `report` (in report order), followed by the average rows.
   - Constructor only stores state — no matplotlib import here.

2. **`from_predictions(y_true, y_pred, *, labels=None, target_names=None,
   sample_weight=None, zero_division="warn", display_labels=None, **plot_kwargs)`**
   (classmethod)
   - Computes `report = classification_report(y_true, y_pred, labels=labels,
     target_names=target_names, sample_weight=sample_weight, output_dict=True,
     zero_division=zero_division)`. `zero_division` is passed straight through to
     `classification_report` (same default, `"warn"`).
   - Builds `ClassificationReportDisplay(report, display_labels=...)` and calls `.plot()`.
   - Returns the display.

3. **`from_estimator(estimator, X, y, *, labels=None, target_names=None,
   sample_weight=None, display_labels=None, **plot_kwargs)`** (classmethod)
   - `check_is_fitted` / `is_classifier` validation like `ConfusionMatrixDisplay`.
   - `y_pred = estimator.predict(X)`; delegates to `from_predictions`.
   - If `target_names`/`display_labels` unset, derive from `estimator.classes_`.

## What it draws

`plot(self, *, include_values=True, cmap="viridis", ax=None, colorbar=True,
values_format=None, im_kw=None, text_kw=None)`

### Grid layout

- **Rows** (top to bottom): one per class (in report order), then `macro avg`, then
  `weighted avg`.
- **Metric columns** (color-mapped, shared 0–1 colormap + single colorbar):
  `precision`, `recall`, `f1-score`.
- **Support column**: a 4th, **text-only** column of per-row `support` counts. It is NOT
  color-mapped (support is a count, not a 0–1 score). Rendered as annotations on neutral
  (blank/white) cells so it never distorts the 0–1 color scale. Included when
  `include_values=True`.
- **`accuracy`**: the report's `accuracy` key is a single scalar (not per-class); it is
  **omitted** from the grid. Documented in the docstring. (Optionally surfaced later as a
  figure title/annotation — out of scope for v1.)

### Rendering details

- `im_ = ax.imshow(metric_matrix, cmap=cmap, vmin=0, vmax=1, **im_kw)` over just the
  3 metric columns.
- When `include_values=True`, annotate each P/R/F1 cell with its value formatted by
  `values_format` (default `.2f`), text color chosen for contrast (light on dark cells,
  dark on light), matching `ConfusionMatrixDisplay`'s threshold approach.
- Support annotations use integer formatting.
- Column tick labels: `["precision", "recall", "f1-score", "support"]` (support only if
  shown). Row tick labels: resolved `display_labels`.
- `colorbar=True` adds a colorbar for the 0–1 metric scale.
- Use `sklearn.utils._plotting._validate_style_kwargs` to merge default and user
  `im_kw`/`text_kw`; use `check_matplotlib_support` at the top of `plot`.

## Stored attributes (Display convention)

- `self.report` — the source dict.
- `self.display_labels` — resolved row labels (ndarray).
- After `plot()`: `self.figure_`, `self.ax_`, `self.im_`,
  `self.text_` (ndarray of matplotlib Text for annotated cells, or `None` when
  `include_values=False`).

## Reuse (not reinvention)

- All metric math is delegated to the existing `classification_report`.
- Plotting helpers reused: `check_matplotlib_support`, `_validate_style_kwargs`.
- No new dependencies.

## Testing

New file `sklearn/metrics/_plot/tests/test_classification_report_display.py`, mirroring
`test_confusion_matrix_display.py`:

- Round-trip: `from_predictions(y_true, y_pred)` produces the same displayed values as
  `ClassificationReportDisplay(classification_report(y_true, y_pred, output_dict=True))`.
- `from_estimator` on a fitted classifier matches `from_predictions` with its predictions.
- `display_labels` / `target_names` propagate to row tick labels.
- `include_values=True` creates `text_` with correct P/R/F1 values and the support column;
  `include_values=False` sets `text_=None` and shows no support column.
- `colorbar=False` adds no colorbar.
- Color scale is fixed to `[0, 1]` (`im_.get_clim() == (0, 1)`), independent of support
  magnitudes — the core fix motivating issue #16880.
- Error handling: `from_estimator` on a regressor / unfitted estimator raises; malformed
  `report` dict raises a clear error.
- Binary and multiclass cases.

## Documentation

- Full numpydoc docstring on the class and each `from_*` method, including a runnable
  `>>>` example (guarded like other Display docstrings).
- Changelog fragment under `doc/whats_new/upcoming_changes/` (or current changelog
  mechanism) announcing `metrics.ClassificationReportDisplay`.
- Add to the visualizations API listing (`doc/visualizations.rst` / `doc/api_reference`
  where the other Display classes appear).
- Minimal user-guide prose; extensive narrative is out of scope for v1.

## Out of scope (v1)

- Parsing the plain-text report string.
- Rendering `accuracy` inside the grid.
- Per-column independent colormaps.
- New example gallery script (can follow in a separate PR).
