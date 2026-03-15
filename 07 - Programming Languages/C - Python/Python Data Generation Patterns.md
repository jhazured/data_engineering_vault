# Python Data Generation Patterns

Patterns for generating realistic test/sample data using Python, pandas, and Faker — commonly used in data engineering projects to populate development environments.

## Reproducible Generation with Seeds

Always set seeds for reproducibility — identical runs produce identical data:

```python
import random
import numpy as np
from faker import Faker

np.random.seed(42)
random.seed(42)
fake = Faker('en_AU')    # Locale-specific fake data
Faker.seed(42)
```

## Class-Based Generator Pattern

Encapsulate all generation logic in a class with dimension and fact generators:

```python
class DataGenerator:
    def __init__(self):
        self.start_date = datetime(2023, 1, 1)
        self.end_date = datetime.now()

        # Shared reference data
        self.vehicle_types = ['TRUCK', 'VAN', 'MOTORCYCLE', 'TRAILER']
        self.statuses = ['PENDING', 'IN_TRANSIT', 'DELIVERED', 'CANCELLED']

    def generate_customers(self) -> pd.DataFrame:
        rows = []
        for i in range(1000):
            customer_type = random.choices(
                ['ENTERPRISE', 'STANDARD', 'BASIC'],
                weights=[0.05, 0.25, 0.70],
            )[0]
            rows.append({
                'customer_id': f'CUST_{str(i + 1).zfill(6)}',
                'customer_name': fake.company(),
                'customer_type': customer_type,
                'credit_limit': round(random.uniform(10_000, 5_000_000), 2),
                'contact_email': fake.email(),
                'created_at': fake.date_time_between(start_date='-3y'),
                'updated_at': fake.date_time_between(start_date='-1y'),
                'is_deleted': False,
                '_loaded_at': datetime.now(),
            })
        return pd.DataFrame(rows)

    def generate_all(self) -> dict:
        """Generate all tables. Returns dict of name → DataFrame."""
        customers = self.generate_customers()
        vehicles = self.generate_vehicles()
        shipments = self.generate_shipments(customers, vehicles)
        return {
            'customers': customers,
            'vehicles': vehicles,
            'shipments': shipments,
        }
```

**Key patterns:**
- `random.choices` with `weights` for realistic distributions (5% enterprise, 70% basic)
- Faker locale (`en_AU`) for region-specific names, addresses, phone numbers
- `_loaded_at` added as ingestion timestamp for downstream incremental loading
- `is_deleted` soft-delete flag matching production source schema

## Dimension-First Generation

Generate dimensions first, then use them as foreign key pools for facts:

```python
def generate_all(self):
    # 1. Generate dimensions
    dim_customer = self.generate_customers()
    dim_vehicle = self.generate_vehicles()
    dim_location = self.generate_locations()
    dim_route = self.generate_routes(dim_location)

    # 2. Generate facts using dimension FKs
    fact_shipments = self.generate_shipments(
        dim_customer, dim_location, dim_vehicle, dim_route
    )

    # 3. Build raw T1 tables from dimensions/facts
    return {
        'customers': self.to_raw_customers(dim_customer),
        'vehicles': self.to_raw_vehicles(dim_vehicle),
        'shipments': self.to_raw_shipments(fact_shipments),
    }
```

## Weighted Random Distributions

Generate realistic business data with appropriate distributions:

```python
# Customer type: mostly basic, few enterprise
customer_type = random.choices(
    ['ENTERPRISE', 'STANDARD', 'BASIC'],
    weights=[0.05, 0.25, 0.70],
)[0]

# Credit limit consistent with tier
credit_ranges = {
    'ENTERPRISE': (1_000_000, 5_000_000),
    'STANDARD':   (100_000,   999_999),
    'BASIC':      (10_000,    99_999),
}
credit_limit = round(random.uniform(*credit_ranges[customer_type]), 2)

# Weekday vs weekend volume
daily_volume = random.randint(50, 200) if date.weekday() < 5 else random.randint(20, 80)

# On-time probability with degradation factors
on_time_prob = 0.85
if date.weekday() >= 5:
    on_time_prob *= 0.9    # Weekends slightly worse
if random.random() > 0.8:
    on_time_prob *= 0.7    # Random disruptions
is_on_time = random.random() < on_time_prob
```

## Unit Conversion at Boundaries

Source systems may use different units. Generate in metric, convert to source units for raw tables:

```python
def to_raw_vehicles(self, vehicles_df):
    """Convert metric dimensions to imperial for raw T1 tables."""
    return pd.DataFrame({
        'vehicle_id': vehicles_df['vehicle_id'],
        'capacity_lbs': (vehicles_df['capacity_kg'] * 2.20462).round(1),
        'fuel_efficiency_mpg': (235.214 / vehicles_df['fuel_efficiency_l_100km']).round(2),
        'current_mileage': (vehicles_df['odometer_km'] * 0.621371).round(1),
        '_loaded_at': datetime.now(),
    })
```

dbt staging then preserves source units; dbt integration converts back to metric.

## Dual Output: CSV + Snowflake

Support both file-based and direct-load workflows:

```python
def save_csv(datasets: dict, output_dir='sample_data'):
    Path(output_dir).mkdir(parents=True, exist_ok=True)
    for name, df in datasets.items():
        path = Path(output_dir) / f'{name}.csv'
        df.to_csv(path, index=False)
        print(f"  {name}: {len(df):,} rows -> {path}")


def load_snowflake(datasets: dict):
    from snowflake.connector.pandas_tools import write_pandas

    conn = snowflake.connector.connect(
        account=os.getenv('SF_ACCOUNT'),
        user=os.getenv('SF_USER'),
        password=os.getenv('SF_PASSWORD'),
        role=os.getenv('SF_ROLE', 'ENGINEER'),
        warehouse=os.getenv('SF_WAREHOUSE'),
        database=os.getenv('SF_DATABASE'),
        schema=os.getenv('SF_SCHEMA'),
    )

    try:
        for name, df in datasets.items():
            table = name.upper()
            success, _, nrows, _ = write_pandas(
                conn, df,
                table_name=table,
                database=os.getenv('SF_DATABASE'),
                schema=os.getenv('SF_SCHEMA'),
                chunk_size=10000,
                auto_create_table=True,
            )
            print(f"  {'OK' if success else 'FAIL'} {table}: {nrows:,} rows")
    finally:
        conn.close()
```

## CLI Interface with argparse

Provide flexible usage via command-line flags:

```python
def main():
    parser = argparse.ArgumentParser(description='Generate sample data')
    parser.add_argument('--load', action='store_true',
                        help='Load directly into Snowflake after generation')
    parser.add_argument('--no-csv', action='store_true',
                        help='Skip CSV output (only with --load)')
    parser.add_argument('--output-dir', default='sample_data',
                        help='CSV output directory')
    args = parser.parse_args()

    if args.no_csv and not args.load:
        parser.error('--no-csv requires --load')

    generator = DataGenerator()
    datasets = generator.generate_all()

    if not args.no_csv:
        save_csv(datasets, args.output_dir)
    if args.load:
        load_snowflake(datasets)
```

**Usage:**
```bash
python data/generate_sample_data.py              # CSVs only
python data/generate_sample_data.py --load       # CSVs + Snowflake
python data/generate_sample_data.py --load --no-csv  # Snowflake only
```

## CSV Loading Script

A separate loader reads CSVs and bulk-loads via `write_pandas`:

```python
SAMPLE_DIR = PROJECT_ROOT / 'sample_data'

# Auto-discover CSV files: customers.csv → CUSTOMERS table
TABLES = {p.stem: p.stem.upper() for p in sorted(SAMPLE_DIR.glob('*.csv'))}

for stem, table in TABLES.items():
    df = pd.read_csv(SAMPLE_DIR / f'{stem}.csv')
    success, _, nrows, _ = write_pandas(
        conn, df,
        table_name=table,
        chunk_size=10000,
        auto_create_table=True,
    )
```

## Environment Variable Validation

Fail fast with clear messages:

```python
required = ['SF_ACCOUNT', 'SF_USER', 'SF_PASSWORD', 'SF_DATABASE', 'SF_SCHEMA']
missing = [v for v in required if not os.getenv(v)]
if missing:
    logger.error(f"Missing environment variables: {missing}")
    logger.error("Run: set -a && source .env && set +a")
    sys.exit(1)
```

## Python Project Patterns

### Requirements File Splitting

Split dependencies by use case to keep CI fast:

```
requirements.txt              # Full stack (ingestion + UI + ML)
requirements-agent-only.txt   # Slim: retriever + connector only (~100MB)
requirements-test.txt         # CI: pytest + parsing libs (no torch/ML)
requirements-dev.txt          # Full + pytest
```

### Dependency Verification Script

Check dependencies and connectivity before running pipelines:

```python
def verify_setup():
    # Required packages
    for pkg in ['snowflake-connector-python', 'pandas']:
        try:
            __import__(pkg.replace('-', '_'))
            print(f"  OK: {pkg}")
        except ImportError:
            print(f"  MISSING: {pkg}")

    # Optional packages (graceful degradation)
    for pkg in ['unstructured', 'pypdf', 'langchain_core']:
        try:
            __import__(pkg)
        except ImportError:
            print(f"  OPTIONAL: {pkg} (needed for ingestion only)")

    # Connectivity smoke test
    try:
        conn = snowflake.connector.connect(...)
        conn.cursor().execute("SELECT 1")
        print("  OK: Snowflake connection")
    except Exception as e:
        print(f"  FAIL: Snowflake: {e}")
```

### .env.example Convention

Provide a template with placeholder values (committed to git):

```bash
# .env.example — copy to .env and fill in real values
SNOWFLAKE_ACCOUNT=xxxxxxx-xxxxxxx
SNOWFLAKE_USER=your_username
SNOWFLAKE_PASSWORD=your_password
SNOWFLAKE_WAREHOUSE=TRN_DEV_CENTRAL_WH
SNOWFLAKE_DATABASE=KNOWLEDGE_DB
CORTEX_MODEL=mistral-large2          # Optional, defaults to mistral-large2
CHUNK_MAX_CHARS=2000                  # Tune per document type
```

## Faker Cheat Sheet

| Method | Output |
|--------|--------|
| `fake.company()` | "Smith-Jones Pty Ltd" |
| `fake.email()` | "jsmith@example.com" |
| `fake.phone_number()` | "+61 4 1234 5678" |
| `fake.address()` | "42 Collins St, Melbourne VIC 3000" |
| `fake.postcode()` | "3000" |
| `fake.name()` | "John Smith" |
| `fake.date_between(start_date='-3y', end_date='today')` | date object |
| `fake.date_time_between(start_date='-1y', end_date='now')` | datetime object |
| `fake.sentence()` | "The quick brown fox jumps." |
| `fake.word()` | "brake" |
