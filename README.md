# Kristalball Hotel Bar Demand Forecasting and Inventory Optimization

This project analyzes hotel bar consumption records, evaluates an observation-level demand forecasting model, creates provisional inventory recommendations, and compares two replenishment policies in a reproducible scenario simulation. It is an academic/business analysis; it is not deployed or production-ready.

## Business problem

Hotel bars need to balance inventory availability with the cost and operational burden of holding stock. Overstocking ties up inventory, while stockouts can leave customer demand unfulfilled. This project explores whether historical consumption records can support demand analysis and provisional reorder guidance. The underlying data is irregularly recorded, so the analysis does not treat an unobserved bar-brand-date as zero consumption.

## Objectives

- Inspect data structure, quality, inventory balances, and observation coverage.
- Explore recorded consumption by bar, brand, alcohol type, and time.
- Engineer historical and calendar features without using future observations.
- Compare a previous-observation baseline with a Random Forest using chronological validation.
- Derive provisional safety stock, reorder points, and par levels using configurable illustrative assumptions.
- Compare no-replenishment and reorder-up-to-par policies in a hypothetical scenario simulation.
- Provide results in an executive dashboard and an Excel workbook.

## Dataset overview

The source file is `data/Consumption Dataset - Dataset.csv`. It contains 6,575 records and 8 original columns, covering 6 bars, 16 brands, 5 alcohol categories, and 96 bar-brand combinations. Recorded timestamps span 2023-01-01 through 2024-01-01. Total recorded consumption is **1,968,681.66 ml**.

The source columns are date/time served, bar, alcohol type, brand, opening balance, purchase, consumed, and closing balance. The CSV is retained unchanged.

## Technology stack

- Python
- Jupyter Notebook
- pandas and NumPy
- Matplotlib and Seaborn
- scikit-learn
- SciPy
- openpyxl

## Project workflow

1. **Data validation and EDA:** inspect records, missing values, duplicates, negative values, date parsing, inventory balance arithmetic, consumption distributions, and observation frequency.
2. **Demand analysis and feature engineering:** aggregate recorded data by date, bar, and brand; review coverage; construct historical and calendar features.
3. **Forecasting:** compare a previous-observation baseline with a preprocessing and Random Forest pipeline using a chronological split.
4. **Inventory recommendations:** estimate demand statistics from recorded observations and calculate provisional safety stock, reorder points, and par levels.
5. **Replenishment simulation:** sample 30-day demand scenarios with replacement from each combination's recorded consumption observations and compare two policies.
6. **Executive dashboard and Excel export:** summarize the existing analysis and export its results.
7. **Final documentation and validation:** document methods and limitations and execute the notebook from a fresh kernel.

The full analysis is in [the forecasting notebook](notebooks/Kristalball_Hotel_Inventory_Forecasting.ipynb).

## Forecasting approach and results

The target is `Daily_Consumption_ml` at the next recorded observation for a bar-brand combination. Features include previous recorded consumption, the average of the previous three records, elapsed days since the previous record, weekday, month, weekend status, bar, brand, and alcohol type. Lag and rolling calculations are grouped by bar and brand and use only earlier observations.

The evaluation holds out the latest 20% of unique dates: training covers 2023-01-06 through 2023-10-21 (5,007 rows, 288 dates), and testing covers 2023-10-22 through 2024-01-01 (1,280 rows, 72 dates). The split is chronological, not random.

| Model | MAE (ml) | RMSE (ml) | WAPE |
|---|---:|---:|---:|
| Previous-observation baseline | 176.278 | 238.279 | 58.525% |
| Random Forest | 147.911 | 180.944 | 49.107% |
| Random Forest improvement | 16.092% | 24.062% | 16.092% |

Random Forest performed better on this chronological test set. This is an observation-level, one-step-ahead evaluation, not validation of a complete daily demand forecast. WAPE remains relatively high; the result does not establish production readiness.

## Inventory optimization approach and results

The project creates 96 provisional recommendations. Recorded observation averages are used only as a **daily demand proxy**, not as a validated daily demand rate. The illustrative assumptions are a 3-day lead time, 1-day review period, and 95% service level. The service-level Z-score is calculated with `scipy.stats.norm.ppf`.

- Safety stock = Z-score x observed-demand standard deviation x square root of lead time.
- Reorder point = average demand proxy x lead time + safety stock.
- Recommended par = average demand proxy x (lead time + review period) + safety stock.

Recommendation quantities are rounded upward to whole millilitres and are non-negative. Total recommended par is **166,445 ml**, averaging approximately **1,733.80 ml** per combination. These are provisional calculations: actual supplier lead times, operational service requirements, and validated daily POS demand were not supplied. Coverage labels describe observation counts, not statistical confidence or forecast accuracy.

## Replenishment simulation approach and results

The simulation runs for 30 days with random seed 42. For each bar-brand combination, daily scenario demand is sampled with replacement from its historical recorded consumption values, retaining explicit zero-consumption observations. This is a hypothetical scenario using an observed-consumption daily proxy, not a reconstruction of actual daily demand or an actual historical backtest.

| Policy | Total demand (ml) | Fulfilled (ml) | Lost (ml) | Stockout days | Combinations with stockouts | Orders |
|---|---:|---:|---:|---:|---:|---:|
| No Replenishment | 853,016.51 | 166,445.00 | 686,571.51 | 2,014 | 96 | 0 |
| Reorder-Up-To-Par | 853,016.51 | 810,290.34 | 42,726.17 | 224 | 88 | 934 |

In this particular simulation scenario, reorder-up-to-par reduced simulated lost demand by **93.78%** relative to no replenishment. This result is not evidence of actual cost savings or operational improvement.

## Project structure

```text
Kristalball-Hotel-Inventory-Optimization/
├── data/
│   └── Consumption Dataset - Dataset.csv
├── notebooks/
│   └── Kristalball_Hotel_Inventory_Forecasting.ipynb
├── reports/
│   └── Project_Report.md
├── outputs/
│   └── Kristalball_Hotel_Inventory_Results.xlsx
├── .venv/  # local virtual environment; ignored by Git
├── README.md
└── .gitignore
```

The notebook creates charts and includes the analyses, dashboard, and Excel export. The workbook contains Executive Summary, Model Comparison, Inventory Recommendations, Simulation Results, Daily Simulation Details, Bar Performance, Brand Performance, and Observation Coverage sheets.

## Setup on Windows PowerShell

Run these commands from the project root:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install pandas numpy matplotlib seaborn scikit-learn scipy ipykernel openpyxl jupyter nbclient
jupyter notebook
```

If PowerShell blocks activation for the current terminal, use a process-scoped policy and activate again:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process
.\.venv\Scripts\Activate.ps1
```

In Jupyter, open `notebooks/Kristalball_Hotel_Inventory_Forecasting.ipynb` and run the notebook from the first cell. Keep the notebook's working directory at the project root so its relative data and output paths resolve. The notebook runs forecasting during Phase 5, then uses those results in later phases.

## Outputs

- Excel workbook: [outputs/Kristalball_Hotel_Inventory_Results.xlsx](outputs/Kristalball_Hotel_Inventory_Results.xlsx)
- Final report: [reports/Project_Report.md](reports/Project_Report.md)
- Full analysis, charts, and executed results: [notebooks/Kristalball_Hotel_Inventory_Forecasting.ipynb](notebooks/Kristalball_Hotel_Inventory_Forecasting.ipynb)

## Assumptions and limitations

- Observations are irregular by bar-brand combination. Although the overall dataset has at least one record on each calendar date in its range, all 96 combinations have unobserved dates within their individual observation spans.
- Missing bar-brand dates are not filled with zero. Only explicit recorded zero-consumption values are zero.
- There are 149 inventory-balance inconsistencies exceeding the notebook's 0.01 ml tolerance; the records are flagged, not silently corrected.
- The forecasting target is consumption at a recorded observation point. Sparse and irregular recording limits interpretation as calendar-day demand.
- Inventory lead time, review period, service level, and the simulation horizon are illustrative assumptions.
- Actual daily POS demand, supplier lead times, purchase-order history, lost-sales records, occupancy and event data, and cost information are unavailable.
- The scenario simulation is hypothetical and is not an actual historical backtest.
- Recommendations and model results require operational validation with appropriate source data before any real-world use.