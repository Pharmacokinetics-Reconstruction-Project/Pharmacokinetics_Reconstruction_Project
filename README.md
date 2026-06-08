# PK Reconstruction Project

> Recover numerical pharmacokinetic data from published time–concentration figures, one click at a time.

A lot of useful pharmacokinetic (PK) data lives only inside figures in journal articles — the underlying tables of (time, concentration, standard error) are never published. This repository is a small, focused tool for getting that data back: open the figure in a notebook, click each observed marker, optionally click each one-sided error-bar endpoint, and write a tidy CSV.

The point isn't to be magical — it's to be reliable. Six worked examples (with both the source figure and the resulting CSV) are included so you can see the full input → output before doing your own.

---

## What's in this repo

```
PK_Reconstruction_Project/
├── pk_figure_to_csv.ipynb     # the extraction notebook
├── example_images/            # six published PK figures
├── extracted_data/            # the corresponding CSVs (raw + cleaned)
├── source_papers/             # PDFs of the original journal articles
├── requirements.txt
├── LICENSE
└── README.md
```

## Quick start

```bash
git clone <this repo>
cd PK_Reconstruction_Project
pip install -r requirements.txt
jupyter notebook pk_figure_to_csv.ipynb
```

Then in the notebook:

1. **Configure** — point `IMAGE_PATH` at your figure and set `OUTPUT_CSV` for the destination.
2. **Calibrate axes** (3 clicks):
   - Click the *origin* of the plot.
   - Click a known x-axis endpoint (you also tell the cell what x value that point corresponds to).
   - Click a known y-axis endpoint (and tell the cell its y value).
3. **Extract each trajectory** — for every curve in the figure, run `extractor.start_trajectory("<name>")`, click each observed marker, then `extractor.finish_trajectory()`. Right-click undoes the last click.
4. **(Optional) Extract one-sided error bars** — run `extractor.start_se_extraction("<name>")`, click each error-bar endpoint (any order — they auto-pair with the closest data point in x), then `extractor.finish_se_extraction()`. The notebook stores `SE = |endpoint_y − mean_y|`, in the same units as `y`.
5. **Save** — `extractor.save_csv(OUTPUT_CSV)` writes a long-form CSV with columns `trajectory, x, y, se`.
6. **Verify** — the round-trip cell reads the CSV back and re-plots it so you can sanity-check against the original figure.

## CSV output format

Long-form, one row per observed point:

```
trajectory,x,y,se
blank,0.49,0.05,0.030
blank,1.00,0.16,0.051
blank,2.02,0.22,0.074
...
filled,0.49,0.02,0.018
filled,1.00,0.05,0.014
...
```

- `trajectory` — whatever string you passed to `start_trajectory(...)`. Use it to distinguish curves on the same figure (e.g. dose level, sub-population, marker style).
- `x`, `y` — the data coordinates of each observed marker, in the units of the original plot's axes.
- `se` — the standard-error magnitude (always positive). `NaN` if you didn't capture an error bar for that point.

## Worked examples

Each row of the table below corresponds to one figure that ships with the repo. The CSVs in `extracted_data/` were produced by running this exact notebook on the matching image; `<drug>_cleaned.csv` is a lightly post-processed version (small rounding fixes near the origin and similar manual touch-ups).

| Drug                | Figure                                | Extracted CSV (raw)                   | Cleaned                                     | Source paper |
|---------------------|---------------------------------------|---------------------------------------|---------------------------------------------|--------------|
| 5-Fluorouracil      | `example_images/5fu_human.png`        | `extracted_data/5fu.csv`              | `extracted_data/5fu_cleaned.csv`            | `source_papers/capcitabine_5fu_hepatic_dysfunction.pdf` |
| Acetaminophen       | `example_images/acetaminophen_human.png` | `extracted_data/acetaminophen.csv`  | `extracted_data/acetaminophen_cleaned.csv`  | `source_papers/acetaminophen_and_oxycodone_paper.pdf` |
| Capecitabine        | `example_images/capecitabine_human.png` | `extracted_data/capecitabine.csv`   | `extracted_data/capecitabine_cleaned.csv`   | `source_papers/capcitabine_5fu_hepatic_dysfunction.pdf` |
| Cisplatin           | `example_images/cisplatin_human.png`  | `extracted_data/cisplatin.csv`        | `extracted_data/cisplatin_cleaned.csv`      | `source_papers/cisplatin_paper.pdf` |
| Midazolam           | `example_images/midazolam_human.png`  | `extracted_data/midazolam.csv`        | `extracted_data/midazolam_cleaned.csv`      | `source_papers/midazolam_paper.pdf` |
| Oxycodone           | `example_images/oxycodone_human.png`  | `extracted_data/oxycodone.csv`        | `extracted_data/oxycodone_cleaned.csv`      | `source_papers/acetaminophen_and_oxycodone_paper.pdf` |

If you reuse the extracted numbers in your own work, please cite the original source paper, not this repo.

## Design notes

**Click-based by choice.** Earlier prototypes of this project included an automatic extractor that used color clustering and line-connectivity heuristics. It worked tolerably on figures with clean, well-separated trajectories but produced unreliable output on dense or muted ones — exactly the cases where you'd most want the data. The click flow is slower per figure but is independent of marker style, color saturation, and crowding. For the kind of one-off extraction this tool is built for, that trade is the right one.

**Linear axes only.** The notebook assumes both axes are linear. If you have a log-axis figure, transform the tick values yourself before calibrating, or extend `pixel_to_data` in the class. Log-scale support used to be built in and was removed at user request to simplify the API and avoid `origin=(0, 0)` edge cases.

**Calibration: 3-click and 4-click variants.**
- `calibrate_corners(x_end, y_end, origin=(0, 0))` — the default. Three clicks total: the origin, a known x-axis endpoint, a known y-axis endpoint. Fast when the origin is a single visible corner of the plot.
- `calibrate(x_refs, y_refs)` — the backup. Four clicks: two known x-axis points, two known y-axis points. Use it when the origin isn't visible or when the x-axis spine doesn't extend back to the data origin.

Both methods set the same internal references, so the rest of the workflow is identical.

**Standard errors.** Many PK papers draw a one-sided error bar at each marker — the SE is symmetric, so only one side is shown to save space. The notebook captures just the magnitude (`|endpoint_y − mean_y|`); the direction of the bar (up vs. down) is discarded, since for a symmetric SE it carries no information. Points you skip get `NaN` in the `se` column.

## Backend notes

The notebook uses `%matplotlib widget` (the `ipympl` backend) so clicks on the inline figure are captured. If `ipympl` isn't installed or you'd prefer a separate window, swap the first cell's magic to `%matplotlib qt` or `%matplotlib tk` — the rest of the workflow works the same.

## Contributing

Pull requests are welcome, in particular:

- Log-axis support (with a clean API that doesn't make `origin=(0, 0)` awkward on linear axes).
- An automatic extraction mode that's actually robust — e.g. via shape-template matching rather than color clustering.
- A `pip install`-able package version of the extractor class, separate from the notebook, so it can be embedded in other tools.
- Additional worked examples (figure + paper + extracted CSV).

For new examples, please include the source figure under `example_images/`, the matching raw and cleaned CSVs under `extracted_data/`, and either the PDF or a citation under `source_papers/`.

## License

[MIT](LICENSE) for the code and the extracted CSVs.

The figures under `example_images/` and the PDFs under `source_papers/` are reproduced for research-reproducibility purposes and remain the property of their original authors and publishers. Cite the source paper if you use the extracted data in derivative work.
