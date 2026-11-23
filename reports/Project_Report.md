# Kristalball Hotel Bar Demand Forecasting and Inventory Optimization

## A. Executive Summary

This project examines hotel bar consumption records to support demand analysis and inventory planning. It validates the source data, explores recorded consumption, engineers historical features, evaluates an observation-level forecast, calculates provisional inventory levels, and compares two replenishment policies in a hypothetical simulation.

The dataset contains 6,575 records across 6 bars, 16 brands, 5 alcohol categories, and 96 bar-brand combinations. Total recorded consumption is 1,968,681.66 ml. A Random Forest outperformed the previous-observation baseline on the chronological test set, with MAE 147.911 ml, RMSE 180.944 ml, and WAPE 49.107%. The 96 provisional recommendations total 166,445 ml of recommended par.

In a 30-day empirical scenario simulation, reorder-up-to-par reduced lost demand from 686,571.51 ml to 42,726.17 ml, a 93.78% simulated reduction. The simulation uses recorded consumption as a daily proxy despite irregular observation intervals; it is not an actual historical backtest. The recommendations and simulation are therefore provisional, not validated operational guidance.

## B. Business Problem

Hotel bars must balance stock availability against the cost and effort of holding inventory. Overstocking can tie up working capital and storage, while insufficient stock can cause stockouts and potentially unfulfilled customer demand. Demand forecasting can help estimate future needs, and inventory replenishment policies can translate demand estimates into reorder and stock targets.

This project evaluates those ideas with the available consumption records. It does not claim that the dataset captures all customer demand or that modeled results quantify financial benefits.

## C. Dataset Overview

| Attribute | Result |
|---|---:|
| Original records | 6,575 |
| Original columns | 8 |
| Bars | 6 |
| Brands | 16 |
| Alcohol categories | 5 |
| Bar-brand combinations | 96 |
| Recorded timestamp range | 2023-01-01 to 2024-01-01 |
| Total recorded consumption | 1,968,681.66 ml |

The eight source columns describe date/time served, bar, alcohol type, brand, opening balance, purchase, consumed quantity, and closing balance. The source CSV was not modified.

## D. Data Cleaning and Validation

The notebook inspected data types and sample records, checked missing values in every source column, checked exact duplicate rows, validated numeric inventory and consumption values, and tested date parsing. It found no missing source values, no exact duplicate rows, no missing or unparseable dates, and no negative values in the checked inventory and consumption fields.

Inventory balance validation used:

> Expected Closing Balance = Opening Balance + Purchase - Consumed

The notebook created a balance difference for comparison and flagged 149 of 6,575 records whose absolute difference exceeded 0.01 ml. The largest absolute difference was 9.25 ml. Flagged records were not silently modified.

The overall dataset has at least one recorded observation on every calendar date in its 2023-01-01 through 2024-01-01 range. However, observations are irregular within each bar-brand series: all 96 combinations have unobserved dates inside their own first-to-last observation span. There were 910 explicit zero-consumption records and no repeated bar-brand-date groups. Missing combination-level dates were not filled with zero because an unrecorded observation does not establish zero demand.

## E. Exploratory Data Analysis

The notebook summarizes numerical and categorical fields and charts recorded consumption by bar, the top 10 brands, and alcohol category. It also charts daily and monthly recorded consumption, observation counts by date, and coverage patterns. Phase 4 adds selected-brand daily trends, weekly and monthly recorded-observation patterns, and the top bar-brand combinations by recorded consumption.

Johnson's Bar had the highest recorded consumption by bar at 344,184.38 ml. Grey Goose had the highest recorded consumption by brand at 158,924.02 ml. These describe recorded quantities, not necessarily complete customer demand.

## F. Feature Engineering

Daily observations were aggregated from the original records by date, bar, and brand. For each bar-brand series, the notebook constructed:

- `Previous_Consumption_ml`: consumption at the preceding recorded observation.
- `Previous_3_Records_Avg_ml`: mean of the three preceding recorded observations, excluding the current target.
- `Days_Since_Previous_Record`: elapsed calendar days since the preceding observation.
- `DayOfWeek`, `Month`, and `IsWeekend`: calendar features for the observation date.

Group-based shifts and rolling calculations prevent values from another bar-brand combination or from the current/future target from entering the historical features. Missing observations were not imputed. Rows lacking the required history were removed to create `model_data`, which contains 6,287 rows and 12 columns across 96 combinations.

## G. Demand Forecasting

The target is consumption at the next **recorded observation** for a bar-brand combination. This is an observation-level, one-step-ahead forecasting exercise, not a validated prediction for every calendar day.

The previous-observation baseline predicts the next value using the most recent recorded consumption. The Random Forest is built in a scikit-learn pipeline with a `ColumnTransformer`, one-hot encoding for bar, brand, and alcohol type, and numerical historical/calendar features. Its preprocessing is fitted on training data.

The split is chronological by unique dates: 288 training dates from 2023-01-06 to 2023-10-21 (5,007 rows) and 72 test dates from 2023-10-22 to 2024-01-01 (1,280 rows). Training and test dates do not overlap.

| Model | MAE (ml) | RMSE (ml) | WAPE |
|---|---:|---:|---:|
| Previous-observation baseline | 176.278 | 238.279 | 58.525% |
| Random Forest | 147.911 | 180.944 | 49.107% |
| Random Forest improvement over baseline | 16.092% | 24.062% | 16.092% |

MAE is the average absolute prediction error; RMSE penalizes larger errors more strongly; WAPE is the sum of absolute errors relative to total absolute actual consumption. Random Forest improved on all three reported metrics on this test set. WAPE remains relatively high, and irregularly timed observations limit interpretation. The model's most influential feature by impurity-based importance was previous recorded consumption (importance approximately 0.456); this is an association in this fitted model, not evidence of causation.

## H. Inventory Optimization

The notebook creates one provisional recommendation for each of the 96 bar-brand combinations. It uses three explicitly illustrative assumptions: lead time of 3 days, review period of 1 day, and service level of 95%, with the corresponding Z-score calculated using SciPy.

The formulas are:

- Safety Stock = Z-score x Demand Standard Deviation x square root of Lead Time.
- Reorder Point = (Average Demand Proxy x Lead Time) + Safety Stock.
- Recommended Par Level = (Average Demand Proxy x (Lead Time + Review Period)) + Safety Stock.

Recommendations are rounded upward to whole millilitres and checked to be non-negative. Total recommended par is 166,445 ml; the average is approximately 1,733.80 ml per combination. Taylor's Bar / Budweiser has the highest recommended par at 2,081 ml, and Brown's Bar / Coors has the highest safety stock at 736 ml.

The average of irregularly recorded consumption is used only as a provisional daily demand proxy. Observation-count labels describe coverage, not statistical confidence or forecast accuracy. Supplier lead times and actual service-level requirements were not provided, so recommendations must be recalibrated with operational data before use.

## I. Replenishment Simulation

The simulation generates 30 days of demand independently for each combination by sampling with replacement from its recorded consumption observations, with seed 42. Explicit zero-consumption records remain in the sampling pool. The policies start at the recommended par:

1. **No Replenishment:** inventory is depleted by daily scenario demand without new orders.
2. **Reorder-Up-To-Par:** due orders arrive before that day's demand; demand is fulfilled from available inventory; afterward, if inventory position is at or below reorder point and no order is already in the pipeline, an order is placed to restore inventory position to par. Orders arrive after the configured 3-day lead time; orders due beyond the horizon remain outstanding.

| Policy | Total demand (ml) | Fulfilled (ml) | Lost (ml) | Stockout days | Combinations with stockouts | Avg. ending inventory (ml) | Orders |
|---|---:|---:|---:|---:|---:|---:|---:|
| No Replenishment | 853,016.51 | 166,445.00 | 686,571.51 | 2,014 | 96 | 0.00 | 0 |
| Reorder-Up-To-Par | 853,016.51 | 810,290.34 | 42,726.17 | 224 | 88 | 454.44 | 934 |

The reorder-up-to-par policy had 93.78% less simulated lost demand than no replenishment in this scenario. These results are hypothetical scenario outcomes using an observed-consumption daily proxy, not an actual historical backtest. Actual daily demand, lost sales, supplier lead times, and replenishment history were unavailable; the simulation does not establish actual cost savings or operational improvements.

## J. Business Insights and Recommendations

- Johnson's Bar and Grey Goose had the highest recorded consumption totals among bars and brands, respectively.
- Random Forest performed better than the baseline on the chronological test set, but the error levels and observation-level target warrant caution.
- Taylor's Bar / Budweiser had the largest recommended par; Brown's Bar / Coors had the largest safety stock.
- In the stated scenario, reorder-up-to-par reduced simulated lost demand substantially compared with no replenishment, while 88 combinations still had at least one stockout day.
- Use coverage labels and observation-gap statistics to identify where data is thin. Do not interpret a missing record as zero consumption.
- Validate the calculations against daily POS data, real supplier lead times, purchase-order history, and operating procedures before considering implementation.

## K. Limitations and Future Improvements

The project would benefit from:

- Actual daily point-of-sale demand for each bar-brand combination.
- Supplier-specific lead-time distributions and order calendars.
- Purchase-order, receipt, stock-adjustment, and on-hand inventory histories.
- Lost-sales or substitution records to measure unmet demand.
- Occupancy, event, seasonality, and promotion data.
- Unit costs, holding costs, spoilage, and stockout costs for cost-aware optimization.
- Operational review and validation of service-level requirements and recommendations.

Historical observations are irregular, so the demand proxy and empirical sampling assumption may not represent calendar-day demand. The 3-day lead time, 1-day review period, 95% service level, and 30-day simulation horizon are assignment assumptions rather than hotel- or supplier-confirmed inputs. Model validation does not establish production readiness.

## L. Conclusion

The project provides a reproducible workflow from data checks and EDA through one-step-ahead forecasting, provisional par recommendations, and a replenishment scenario comparison. It preserves source records, flags balance discrepancies, and avoids treating missing bar-brand dates as zero. The results are useful for demonstrating analytical methods and identifying data needs, but real operational decisions require validated daily demand, supplier and inventory records, cost inputs, and stakeholder review.

## Deliverables

- Analysis notebook: `notebooks/Kristalball_Hotel_Inventory_Forecasting.ipynb`
- Results workbook: `outputs/Kristalball_Hotel_Inventory_Results.xlsx`
- Project guide: `README.md`
