# pandas & Polars for Data Engineering

Practical reference for DataFrame libraries used in data engineering pipelines — when to reach for each tool, core operations, performance techniques, and migration patterns.

See also: [[Python Core Patterns for Data Engineering]], [[Python Data Generation Patterns]], [[PySpark Core Concepts]]

---

## 1. When to Use What

| Criterion | pandas | Polars | PySpark | DuckDB |
|-----------|--------|--------|---------|--------|
| **Best for** | Prototyping, small-medium data, wide ecosystem | Performance-critical local work, lazy evaluation | Distributed/cluster processing | SQL on files, ad-hoc analytics |
| **Data size** | Up to ~5 GB (single node memory) | Up to ~50 GB (single node, streaming) | TB+ across a cluster | Up to ~100 GB (single node) |
| **Execution** | Eager only | Eager or lazy | Lazy (DAG) | Lazy (query planner) |
| **Threading** | Single-threaded | Multi-threaded (Rust) | Distributed (JVM) | Multi-threaded (C++) |
| **API style** | Method chaining, index-based | Expression-based, no index | DataFrame + SQL | SQL-first, optional DataFrame |
| **Ecosystem** | Largest — scikit-learn, plotting, connectors | Growing — Arrow-native interchange | Spark MLlib, Delta Lake | Excellent Parquet/CSV support |
| **Install** | `pip install pandas` | `pip install polars` | `pip install pyspark` | `pip install duckdb` |

**Rule of thumb:** start with pandas for exploration, move to Polars when performance matters on a single machine, reach for PySpark only when data truly requires distribution, and use DuckDB when SQL is more natural than DataFrame code.

---

## 2. pandas Fundamentals

### Core structures

```python
import pandas as pd

# Series (1D) and DataFrame (2D)
s = pd.Series([10, 20, 30], name="amount")
df = pd.DataFrame({"id": [1, 2, 3], "amount": [10, 20, 30]})
```

### Reading data

```python
df = pd.read_csv("data.csv", dtype={"id": str})
df = pd.read_parquet("data.parquet")
df = pd.read_sql("SELECT * FROM orders", con=engine)
df = pd.read_json("data.json", lines=True)
```

### Indexing and filtering

```python
df.loc[df["status"] == "ACTIVE"]           # Label-based, returns matching rows
df.iloc[0:10]                               # Position-based slice
df.query("amount > 100 and status == 'ACTIVE'")  # Readable string filter
```

### Grouping and aggregation

```python
summary = (
    df.groupby("region")
    .agg(
        total=("amount", "sum"),
        avg_amount=("amount", "mean"),
        order_count=("order_id", "nunique"),
    )
    .reset_index()
)
```

### Joins and concatenation

```python
merged = pd.merge(orders, customers, on="customer_id", how="left")
stacked = pd.concat([df_jan, df_feb, df_mar], ignore_index=True)
```

### Reshape

```python
wide = df.pivot_table(index="date", columns="region", values="amount", aggfunc="sum")
long = pd.melt(wide.reset_index(), id_vars="date", var_name="region", value_name="amount")
```

### Method chaining

```python
result = (
    pd.read_parquet("events.parquet")
    .query("event_type == 'purchase'")
    .assign(amount_usd=lambda x: x["amount"] * x["fx_rate"])
    .groupby("customer_id")
    .agg(total_usd=("amount_usd", "sum"))
    .reset_index()
    .sort_values("total_usd", ascending=False)
)
```

---

## 3. pandas for ETL

### Reading from multiple sources

```python
from pathlib import Path

# Glob and concatenate
files = sorted(Path("raw/").glob("orders_*.parquet"))
df = pd.concat([pd.read_parquet(f) for f in files], ignore_index=True)
```

### Cleaning

```python
df = (
    df
    .dropna(subset=["customer_id"])              # Remove rows missing key field
    .fillna({"discount": 0.0, "notes": ""})      # Default missing values
    .replace({"status": {"CANCELLED": "CANCELED"}})  # Standardise values
    .astype({"order_date": "datetime64[ns]", "amount": "float64"})
)
```

### Deduplication and derived columns

```python
df = df.drop_duplicates(subset=["order_id"], keep="last")

df = df.assign(
    order_year=lambda x: x["order_date"].dt.year,
    amount_category=lambda x: pd.cut(x["amount"], bins=[0, 100, 1000, float("inf")],
                                      labels=["small", "medium", "large"]),
)
```

### Writing results

```python
df.to_parquet("output/orders_clean.parquet", index=False, engine="pyarrow")
df.to_sql("orders_clean", con=engine, if_exists="replace", index=False, method="multi")
```

### Chunked reading for large files

```python
chunks = pd.read_csv("huge_file.csv", chunksize=100_000)
for chunk in chunks:
    cleaned = transform(chunk)
    cleaned.to_parquet(f"output/part_{i}.parquet", index=False)
```

---

## 4. pandas Performance

### Vectorised operations over apply

```python
# Slow — Python-level row iteration
df["total"] = df.apply(lambda row: row["qty"] * row["price"], axis=1)

# Fast — vectorised NumPy operation
df["total"] = df["qty"] * df["price"]
```

### Categorical dtype for low-cardinality columns

```python
df["status"] = df["status"].astype("category")    # 8 unique values in 1M rows → major memory saving
df["region"] = df["region"].astype("category")
```

### Memory reduction — downcast numerics

```python
df["qty"] = pd.to_numeric(df["qty"], downcast="integer")       # int64 → int16 if values fit
df["price"] = pd.to_numeric(df["price"], downcast="float")     # float64 → float32
```

### query() and eval() for readable expressions

```python
df.query("amount > 100 and region in @valid_regions")     # @ references Python variables
df.eval("margin = revenue - cost", inplace=True)          # Computed column without temporaries
```

### PyArrow backend (pandas 2.0+)

```python
df = pd.read_parquet("data.parquet", dtype_backend="pyarrow")
# Benefits: nullable dtypes, lower memory, faster string operations, better interop with Arrow-based tools
```

---

## 5. Polars Fundamentals

### DataFrame vs LazyFrame

```python
import polars as pl

# Eager — executes immediately
df = pl.DataFrame({"id": [1, 2, 3], "amount": [10, 20, 30]})

# Lazy — builds query plan, executes on .collect()
lf = pl.scan_parquet("data.parquet")
result = lf.filter(pl.col("amount") > 100).collect()
```

### Expressions — the core abstraction

```python
df = df.with_columns(
    pl.col("amount").cast(pl.Float64),
    pl.lit("USD").alias("currency"),
    pl.when(pl.col("amount") > 1000)
      .then(pl.lit("high"))
      .otherwise(pl.lit("normal"))
      .alias("tier"),
)
```

### Select, filter, group_by

```python
result = (
    pl.scan_parquet("orders.parquet")
    .filter(pl.col("status") == "ACTIVE")
    .group_by("region")
    .agg(
        pl.col("amount").sum().alias("total"),
        pl.col("amount").mean().alias("avg"),
        pl.col("order_id").n_unique().alias("order_count"),
    )
    .sort("total", descending=True)
    .collect()
)
```

### Join and scan

```python
orders = pl.scan_parquet("orders.parquet")
customers = pl.scan_parquet("customers.parquet")

joined = (
    orders.join(customers, on="customer_id", how="left")
    .collect()
)
```

---

## 6. Polars Advantages

| Feature | Detail |
|---------|--------|
| **Multi-threaded** | All operations parallelised across CPU cores by default |
| **Lazy optimisation** | Predicate pushdown, projection pushdown, slice pushdown — Polars only reads the columns/rows it needs |
| **No index** | Simpler mental model — no `reset_index()`, no `set_index()`, no alignment bugs |
| **Rust core** | Native performance without JIT compilation or C extensions |
| **Streaming** | `collect(streaming=True)` processes data in batches for out-of-core datasets |
| **Consistent API** | Same expressions work in select, filter, group_by, and join contexts |

### Streaming for large data

```python
result = (
    pl.scan_csv("huge_file.csv")
    .filter(pl.col("year") >= 2024)
    .group_by("category")
    .agg(pl.col("revenue").sum())
    .collect(streaming=True)      # Process in chunks — doesn't need full dataset in memory
)
```

---

## 7. Common Operations Comparison

| Operation | pandas | Polars |
|-----------|--------|--------|
| **Read Parquet** | `pd.read_parquet("f.parquet")` | `pl.read_parquet("f.parquet")` |
| **Lazy read** | N/A | `pl.scan_parquet("f.parquet")` |
| **Filter rows** | `df[df["x"] > 10]` or `df.query("x > 10")` | `df.filter(pl.col("x") > 10)` |
| **Select columns** | `df[["a", "b"]]` | `df.select("a", "b")` |
| **New column** | `df.assign(c=lambda x: x["a"] + x["b"])` | `df.with_columns((pl.col("a") + pl.col("b")).alias("c"))` |
| **Group + agg** | `df.groupby("k").agg(total=("v", "sum"))` | `df.group_by("k").agg(pl.col("v").sum().alias("total"))` |
| **Join** | `pd.merge(a, b, on="k", how="left")` | `a.join(b, on="k", how="left")` |
| **Window function** | `df["rn"] = df.groupby("k")["v"].rank()` | `df.with_columns(pl.col("v").rank().over("k").alias("rn"))` |
| **Pivot** | `df.pivot_table(index="d", columns="k", values="v")` | `df.pivot(on="k", index="d", values="v")` |
| **Sort** | `df.sort_values("v", ascending=False)` | `df.sort("v", descending=True)` |
| **Write Parquet** | `df.to_parquet("out.parquet")` | `df.write_parquet("out.parquet")` |

---

## 8. Integration Patterns

### pandas <-> Polars conversion

```python
# Polars → pandas
pdf = polars_df.to_pandas()

# pandas → Polars
pldf = pl.from_pandas(pandas_df)
```

Arrow is the zero-copy interchange format — both libraries use it internally, so conversion is fast.

### pandas with Snowflake

```python
from snowflake.connector.pandas_tools import write_pandas

write_pandas(conn, df, table_name="ORDERS", auto_create_table=True, chunk_size=10_000)
```

See [[Python Data Generation Patterns]] for full Snowflake loading examples.

### Polars with DuckDB

```python
import duckdb

# Query a Polars DataFrame directly — zero-copy via Arrow
result = duckdb.sql("SELECT region, SUM(amount) FROM polars_df GROUP BY region").pl()

# Scan Parquet with DuckDB, return Polars DataFrame
result = duckdb.sql("SELECT * FROM 'data/*.parquet' WHERE year = 2025").pl()
```

### pandas with PySpark

```python
# PySpark → pandas (collects to driver — be careful with large data)
pdf = spark_df.toPandas()

# pandas → PySpark
spark_df = spark.createDataFrame(pandas_df)
```

See [[PySpark Core Concepts]] for distributed DataFrame patterns.

---

## 9. Testing DataFrames

### pandas assertions

```python
import pandas as pd
from pandas.testing import assert_frame_equal

def test_transform_adds_derived_column():
    input_df = pd.DataFrame({"qty": [2, 3], "price": [10.0, 20.0]})
    result = transform(input_df)

    expected = pd.DataFrame({"qty": [2, 3], "price": [10.0, 20.0], "total": [20.0, 60.0]})
    assert_frame_equal(result, expected)
```

### Polars assertions

```python
import polars as pl
from polars.testing import assert_frame_equal

def test_polars_filter():
    df = pl.DataFrame({"status": ["ACTIVE", "INACTIVE", "ACTIVE"], "amount": [10, 20, 30]})
    result = df.filter(pl.col("status") == "ACTIVE")

    expected = pl.DataFrame({"status": ["ACTIVE", "ACTIVE"], "amount": [10, 30]})
    assert_frame_equal(result, expected)
```

### Parametrised test patterns

```python
import pytest

@pytest.mark.parametrize("input_status,expected_count", [
    ("ACTIVE", 2),
    ("INACTIVE", 1),
    ("UNKNOWN", 0),
])
def test_filter_by_status(input_status, expected_count, sample_df):
    result = filter_by_status(sample_df, input_status)
    assert len(result) == expected_count
```

### Fixture factories

```python
@pytest.fixture
def make_orders():
    """Factory fixture — call with overrides to get a test DataFrame."""
    def _make(n=5, status="ACTIVE", **overrides):
        data = {
            "order_id": range(1, n + 1),
            "status": [status] * n,
            "amount": [100.0] * n,
        }
        data.update(overrides)
        return pd.DataFrame(data)
    return _make

def test_dedup(make_orders):
    df = make_orders(n=3, order_id=[1, 1, 2])
    result = df.drop_duplicates(subset=["order_id"])
    assert len(result) == 2
```

See [[Python Testing with pytest]] for broader testing patterns.

---

## 10. Migration Path — pandas to Polars

### Incremental approach

1. **Keep pandas for IO** — read with pandas, convert to Polars for transforms, convert back for writing to connectors that only support pandas (Snowflake `write_pandas`, SQLAlchemy).
2. **Migrate transforms first** — replace `groupby/apply` chains with Polars expressions for immediate speed wins.
3. **Switch to native Polars IO** — once stable, use `pl.scan_parquet` / `pl.read_csv` directly.

### Key API differences

| pandas | Polars | Notes |
|--------|--------|-------|
| `inplace=True` | Not supported | Polars always returns a new DataFrame — assign the result |
| `df.index` | No index | Use regular columns; pass join keys explicitly |
| `df.apply(func, axis=1)` | `df.with_columns(expr)` | Write expressions instead of row-wise Python functions |
| `df.groupby()` | `df.group_by()` | Underscore in method name |
| `df["col"]` returns Series | `df["col"]` returns Series | Similar, but Polars Series has different methods |
| `pd.concat([...])` | `pl.concat([...])` | Same idea, different import |
| `df.rename(columns={...})` | `df.rename({"old": "new"})` | No `columns` keyword |
| `NaN` for missing floats | `null` for all types | Polars uses Arrow nulls — consistent across dtypes |
| `dtype: object` for strings | `dtype: Utf8` / `String` | Polars has proper string type, not object |

### Common pitfall — expressions vs lambdas

```python
# pandas habit — works but slow in Polars
df.with_columns(pl.col("name").apply(lambda s: s.upper()))

# Polars way — native, fast
df.with_columns(pl.col("name").str.to_uppercase())
```

The general rule: if you reach for `apply` or `map_elements` in Polars, look for a built-in expression first. Polars expressions run in Rust; Python lambdas force row-by-row Python execution.

---

**Related:** [[DuckDB for Data Engineering]] | [[PySpark Core Concepts]] | [[Python Core Patterns for Data Engineering]]
