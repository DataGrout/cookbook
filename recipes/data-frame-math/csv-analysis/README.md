# Recipe: CSV / Tabular Data Analysis

> **Difficulty:** Beginner–Intermediate  
> **Credits:** 0 (all math and frame operations are zero-credit)  
> **Time:** ~10 minutes

## What it does

Analyze any tabular dataset, CSV, API response, LC facts, using DataGrout's zero-credit math suite. No LLM inference needed; every step is deterministic and free.

Pipeline:
```
records (any source) -> math.describe -> math.trend -> math.outliers -> prism.chart
```

All math tools accept `cache_ref` so results pipe directly from one step to the next without re-sending data.

## Tools used

| Tool | Purpose | Credits |
|------|---------|---------|
| `math.describe` | Descriptive stats (mean, median, std, percentiles, histogram) | 0 |
| `math.trend` | Linear/polynomial/exponential regression + forecast | 0 |
| `math.window` | Moving averages, cumulative sums, rolling diff | 0 |
| `math.outliers` | IQR or z-score outlier detection | 0 |
| `math.normalize` | Normalize to z-score, min-max, or rank | 0 |
| `math.correlate` | Pearson / Spearman correlation between two series | 0 |
| `math.rank` | Rank rows by a numeric field | 0 |
| `frame.pluck` | Extract a single numeric column from records | 0 |
| `logic.tabulate` | Convert LC facts -> flat records for math tools | 0 |

## The Recipe

### Step 1: Load your data

The math tools accept a `payload` (JSON array of records) or a `cache_ref` from a previous step. If your data is in CSV, read it with the standard library first:

```python
import csv, json
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

with open("sales.csv") as f:
    rows = list(csv.DictReader(f))

# Convert numeric strings
records = [
    {"month": r["month"], "revenue": float(r["revenue"]), "units": int(r["units"])}
    for r in rows
]
```

Or use LC facts as the source, `logic.tabulate` pivots entity attributes into flat records at zero cost:

```python
# If revenue data is stored as LC facts:
records_result = client.perform("data-grout@1/logic.tabulate@1", {
    "namespace": "sales-data",
    "types": ["attribute"],
    "columns": ["month", "revenue", "units"]
})
records = records_result["records"]
```

### Step 2: Descriptive statistics

```python
stats = client.perform("data-grout@1/math.describe@1", {
    "payload": records,
    "field": "revenue"
})

print(f"Mean:   {stats['mean']:.2f}")
print(f"Median: {stats['median']:.2f}")
print(f"Std:    {stats['std']:.2f}")
print(f"P95:    {stats['p95']:.2f}")

# stats["histogram"] is a records array: pipe to prism.chart
```

### Step 3: Trend analysis and forecasting

```python
trend = client.perform("data-grout@1/math.trend@1", {
    "payload": records,
    "x_field": "month",   # or use integer index implicitly
    "y_field": "revenue",
    "model": "linear",
    "forecast": 3          # 3 months ahead
})

print(f"Model: {trend['equation']}")        # e.g. "y = 1420.5x + 48300"
print(f"R²: {trend['r_squared']:.3f}")
print(f"Forecast: {trend['forecast_values']}")
```

### Step 4: Rolling metrics

Moving average smooths noise; cumulative sum tracks total:

```python
ma = client.perform("data-grout@1/math.window@1", {
    "payload": records,
    "field": "revenue",
    "op": "moving_avg",
    "window": 3
})
# ma["values"] = smoothed series

cumsum = client.perform("data-grout@1/math.window@1", {
    "cache_ref": ma["cache_ref"],   # pipe directly — no data re-transfer
    "field": "value",
    "op": "cumsum"
})
```

### Step 5: Outlier detection

```python
outliers = client.perform("data-grout@1/math.outliers@1", {
    "payload": records,
    "field": "revenue",
    "method": "iqr",
    "k": 1.5
})

print(f"Outlier count: {outliers['outlier_count']}")
print(f"Outlier values: {outliers['outlier_values']}")
print(f"Cleaned array has {len(outliers['cleaned_values'])} points")
```

### Step 6: Correlation between two series

```python
corr = client.perform("data-grout@1/math.correlate@1", {
    "payload": records,
    "x_field": "revenue",
    "y_field": "units",
    "method": "both"
})

print(f"Pearson:  {corr['pearson']:.3f}  ({corr['pearson_interpretation']})")
print(f"Spearman: {corr['spearman']:.3f}")
```

### Step 7: Normalize for comparison

```python
# Z-score normalize to compare series on the same scale
normalized = client.perform("data-grout@1/math.normalize@1", {
    "payload": records,
    "field": "revenue",
    "method": "zscore"
})
# normalized["records"] has a "value" column with z-scores
```

## Full pipeline: zero-credit CSV analysis

```python
import csv
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

with open("sales.csv") as f:
    records = [
        {"month": i + 1, "revenue": float(r["revenue"])}
        for i, r in enumerate(csv.DictReader(f))
    ]

# 1. Stats overview
stats = client.perform("data-grout@1/math.describe@1", {
    "payload": records, "field": "revenue"
})

# 2. Trend + 2-month forecast
trend = client.perform("data-grout@1/math.trend@1", {
    "cache_ref": stats["cache_ref"],
    "y_field": "revenue", "model": "linear", "forecast": 2
})

# 3. Flag anomalous months
outliers = client.perform("data-grout@1/math.outliers@1", {
    "cache_ref": stats["cache_ref"],
    "field": "revenue", "method": "zscore", "threshold": 2.5
})

print(f"Mean revenue: ${stats['mean']:,.0f}")
print(f"Trend: {trend['equation']}  (R²={trend['r_squared']:.3f})")
print(f"Next 2 months forecast: {trend['forecast_values']}")
print(f"Anomalous months: {outliers['outlier_indices']}")
```

Total credits: **0**.

## Using sequences for synthetic data or baselines

```python
# Generate 24-month baseline at 5% monthly growth (geometric)
baseline = client.perform("data-grout@1/math.sequence@1", {
    "type": "geometric",
    "count": 24,
    "start": 10000,
    "ratio": 1.05
})

# Correlate actual revenue against geometric baseline
corr = client.perform("data-grout@1/math.correlate@1", {
    "x": baseline["values"],
    "y": [r["revenue"] for r in records],
    "method": "pearson"
})
print(f"Fit to 5% growth curve: {corr['pearson']:.3f}")
```

## Ranking rows

```python
ranked = client.perform("data-grout@1/math.rank@1", {
    "payload": records,
    "field": "revenue",
    "method": "dense",
    "ascending": False   # rank 1 = highest revenue
})
# ranked["records"] has "revenue" and "rank" columns
```

## From LC facts to analysis

When your data lives in a logic cell namespace, `logic.tabulate` is the zero-credit bridge:

```python
# Tabulate all attribute facts for entities whose names start with "q1_"
table = client.perform("data-grout@1/logic.tabulate@1", {
    "namespace": "sales-data",
    "types": ["attribute"],
    "entities": ["q1_revenue", "q2_revenue", "q3_revenue", "q4_revenue"],
    "columns": ["entity", "value"]
})

# Pipe immediately to math.describe via cache_ref
stats = client.perform("data-grout@1/math.describe@1", {
    "cache_ref": table["cache_ref"],
    "field": "value"
})
```

## See also

- [zero-credit-pipeline](../../../patterns/zero-credit-pipeline/) — pipeline design without spending credits
- [cache-ref-composition](../../../patterns/cache-ref-composition/) — piping data between steps
- [flow/trend-analysis-pipeline](../../flow/trend-analysis-pipeline/) — filter -> group -> trend pipeline
- [competitive-intel-pipeline](../../combined/competitive-intel-pipeline/) — research -> assert -> tabulate -> analyze
