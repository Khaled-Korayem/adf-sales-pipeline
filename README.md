# Azure Data Factory — Sales Data Cleaning Pipeline

An end-to-end **Azure Data Factory** project that reads raw `sales.csv` from **Azure Data Lake Storage Gen2**, applies multi-step data cleaning and sorting transformations via a **Mapping Data Flow**, and writes the cleaned output back to a separate ADLS Gen2 container.

---

## Project Structure

```
adf-sales-pipeline/
├── linkedService/
│   └── AzureDataLakeGen2.json          # Linked Service to ADLS Gen2
├── dataset/
│   ├── SourceSalesCSV.json             # Source dataset (raw container)
│   └── SinkCleanedSalesCSV.json        # Sink dataset (processed container)
├── dataflow/
│   └── SalesCleaningDataFlow.json      # Mapping Data Flow (5 transformations)
├── pipeline/
│   └── SalesCleaningPipeline.json      # Orchestration pipeline
├── output/
│   └── sales_cleaned_dummy.csv         # Dummy file used for schema import
├── docs/
│   ├── dataflow_screenshot.png         # Data Flow canvas screenshot
│   ├── pipeline_run_screenshot.png     # Pipeline run monitor screenshot
│   └── output_screenshot.png          # Output file in ADLS Gen2 screenshot
├── sales.csv                           # Original raw input file
└── README.md
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Azure Data Lake Storage Gen2                       │
│                     Account: korayemstorage                         │
│                                                                     │
│  ┌───────────────────────────┐    ┌────────────────────────────┐   │
│  │  File System: raw         │    │  File System: processed     │   │
│  │  Folder: sales-data       │    │  Folder: sales-cleaned      │   │
│  │  File:   sales.csv        │    │  File:   sales_cleaned.csv  │   │
│  └─────────────┬─────────────┘    └──────────────▲─────────────┘   │
└────────────────┼──────────────────────────────────┼────────────────┘
                 │                                  │
                 ▼                                  │
┌────────────────────────────────────────────────────────────────────┐
│                    Azure Data Factory                              │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  Pipeline: SalesCleaningPipeline                            │  │
│  │                                                             │  │
│  │  ┌───────────────────────────────────────────────────────┐  │  │
│  │  │  Mapping Data Flow: SalesCleaningDataFlow             │  │  │
│  │  │                                                       │  │  │
│  │  │  Source → T1 → T2 → T3 → T4 → Sink                  │  │  │
│  │  │                                                       │  │  │
│  │  └───────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow — Transformation Steps

```
[Source: SourceSalesCSV]
      │   220 rows read from raw/sales-data/sales.csv
      ▼
┌─────────────────────┐
│  T1: RemoveNullRows │  Filter — drop rows where order_id, order_date,
└─────────┬───────────┘  customer_id, product, quantity, unit_price are NULL
          │   86 rows remaining  (134 dropped)
          ▼
┌──────────────────────────┐
│ T2: RemoveNegativeValues │  Filter — drop rows where
└─────────┬────────────────┘  quantity ≤ 0 or unit_price ≤ 0
          │   45 rows remaining  (41 dropped)
          ▼
┌──────────────────────────────┐
│ T3: NormalizeAndRecalculate  │  DerivedColumn:
└─────────┬────────────────────┘  · category      → 'Electronics'
          │                       · customer_id   → CUST001 format
          │                       · order_date    → YYYY-MM-DD format
          │                       · total_amount  → quantity × unit_price
          ▼
┌──────────────────────────┐
│ T4: SortByDateAndOrderID │  Sort:
└─────────┬────────────────┘  · order_date ASC (primary)
          │                   · order_id ASC   (secondary)
          ▼
[Sink: SinkCleanedSalesCSV]
      45 rows written to processed/sales-cleaned/sales_cleaned.csv
```

---

## Transformations Detail

### T1 — RemoveNullRows (Filter)
```
!isNull(order_id) &&
!isNull(order_date) &&
!isNull(customer_id) &&
!isNull(product) &&
!isNull(quantity) &&
!isNull(unit_price)
```

### T2 — RemoveNegativeValues (Filter)
```
toInteger(quantity) > 0 &&
toDouble(unit_price) > 0
```

### T3 — NormalizeAndRecalculate (DerivedColumn)

**category**
```
'Electronics'
```

**customer_id**
```
'CUST' + right('00' + toString(
  toInteger(regexReplace(upper(customer_id),'[^0-9]',''))
), 3)
```

**order_date** (handles 4 different input formats)
```
iif(
  locate('/', order_date) > 0 && length(order_date) == 10,
  toString(toDate(order_date, 'yyyy/MM/dd'), 'yyyy-MM-dd'),
  iif(
    locate('/', order_date) > 0,
    toString(toDate(order_date, 'MM/dd/yyyy'), 'yyyy-MM-dd'),
    iif(
      locate('-', order_date) > 0,
      toString(toDate(order_date, 'yyyy-MM-dd'), 'yyyy-MM-dd'),
      iif(
        locate(' ', order_date) > 0,
        toString(toDate(order_date, 'MMMM d yyyy'), 'yyyy-MM-dd'),
        order_date
      )
    )
  )
)
```

**total_amount**
```
toInteger(quantity) * toDouble(unit_price)
```

### T4 — SortByDateAndOrderID (Sort)
```
Condition 1: order_date ASC
Condition 2: order_id   ASC
```

---

## Pipeline Run Results

| Metric | Value |
|---|---|
| Pipeline name | SalesCleaningPipeline |
| Data Flow | SalesCleaningDataFlow |
| Status | Succeeded |
| Storage account | korayemstorage (ADLS Gen2) |
| Source | raw/sales-data/sales.csv |
| Sink | processed/sales-cleaned/sales_cleaned.csv |
| Input rows | 220 |
| Rows after T1 | 86 |
| Rows after T2 | 45 |
| Output rows | 45 |
| Rows removed | 175 |

### Data quality issues fixed

| Issue | Raw example | Cleaned value |
|---|---|---|
| NULL order_date | NaN | Row removed |
| NULL customer_id | NaN | Row removed |
| Negative quantity | -1 | Row removed |
| Negative unit_price | -100 | Row removed |
| Inconsistent category | Elec, electronics | Electronics |
| Inconsistent customer_id | cust002, C-003, 4 | CUST002, CUST003, CUST004 |
| Mixed date format (slash) | 01/02/2024 | 2024-01-02 |
| Mixed date format (text) | March 3 2024 | 2024-03-03 |
| Mixed date format (slash-year) | 2024/04/05 | 2024-04-05 |
| Wrong Total Amount | Miscalculated | Recomputed qty × price |
| Unsorted data | Random order | Sorted by date then order_id |

---

## Screenshots

### Data Flow Canvas
![Data Flow Canvas](docs/dataflow_screenshot.png)

### Pipeline Run — Monitor Tab
![Pipeline Run](docs/pipeline_run_screenshot.png)

### Output File in ADLS Gen2
![Output File](docs/output_screenshot.png)

---

## Output Sample

```
order_id,order_date,customer_id,product,category,quantity,unit_price,total_amount
15,2024-01-01,CUST002,Phone,Electronics,3,200.0,600
38,2024-01-01,CUST004,Headphones,Electronics,3,500.0,1500
73,2024-01-01,CUST002,Monitor,Electronics,1,1000.0,1000
75,2024-01-01,CUST001,Laptop,Electronics,3,500.0,1500
107,2024-01-02,CUST001,Headphones,Electronics,3,1000.0,3000
72,2024-01-02,CUST004,Phone,Electronics,1,200.0,200
64,2024-03-03,CUST004,Phone,Electronics,3,500.0,1500
89,2024-03-03,CUST002,Laptop,Electronics,2,50.0,100
21,2024-04-05,CUST003,Monitor,Electronics,1,50.0,50
```

---

## How to Deploy

### Prerequisites
- Azure subscription
- Azure Data Factory instance
- Azure Data Lake Storage Gen2 account with:
  - File system: `raw` → folder: `sales-data` → file: `sales.csv`
  - File system: `processed` → folder: `sales-cleaned`

### Steps

1. Clone this repository
```bash
git clone https://github.com/YOUR_USERNAME/adf-sales-pipeline.git
```

2. Update the Linked Service with your storage account details in `linkedService/AzureDataLakeGen2.json`:
```json
"url": "https://YOUR_STORAGE_ACCOUNT.dfs.core.windows.net"
```

3. Import ADF artifacts into your Data Factory via ADF Studio → Manage → Git configuration, or manually create each resource using the JSON files as reference.

4. Upload `sales.csv` to your ADLS Gen2 `raw/sales-data/` path.

5. Publish all in ADF Studio → trigger the pipeline → monitor in the Monitor tab.

6. Download the output from `processed/sales-cleaned/sales_cleaned.csv`.

---

## Technologies Used

- **Azure Data Factory** — Orchestration and Mapping Data Flow
- **Azure Data Lake Storage Gen2** — Source (raw) and Sink (processed) storage
- **ADF Mapping Data Flow** — Filter, DerivedColumn, Sort transformations
- **ADF Expression Language** — Data transformation expressions
