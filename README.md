# ETL Pipeline - Medallion Architecture (Bronze-Silver-Gold)

## Project Overview

This project demonstrates a **Medallion Architecture ETL pipeline** using **SQL Server Integration Services (SSIS)** and **SQL Server**. The pipeline ingests product sales data from a CSV file and progressively refines it across three data layers, culminating in business intelligence insights about product performance and profitability.

### Objectives
- Extract raw data from CSV source
- Transform and validate data following the medallion pattern
- Load refined data into consumption-ready Gold layer
- Identify top-performing products by sales volume and profitability
- Enable business intelligence and analytics on product metrics

---

## Architecture Overview

### Medallion Architecture Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                     CSV SOURCE FILE                             │
│                  (Raw Product Data)                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
        ┌────────────────────────────────┐
        │    BRONZE LAYER (Raw)          │
        │  ✓ Load raw CSV data as-is     │
        │  ✓ Minimal transformation      │
        │  ✓ Data quality checks         │
        │  ✓ Store with timestamps       │
        └────────────────┬───────────────┘
                         │
                         ▼
        ┌────────────────────────────────┐
        │    SILVER LAYER (Clean)        │
        │  ✓ Data validation & cleansing │
        │  ✓ Remove duplicates           │
        │  ✓ Standardize formats         │
        │  ✓ Join reference tables       │
        │  ✓ Handle nulls & anomalies    │
        └────────────────┬───────────────┘
                         │
                         ▼
        ┌────────────────────────────────┐
        │    GOLD LAYER (Analytics)      │
        │  ✓ Aggregated business views   │
        │  ✓ Top selling products        │
        │  ✓ Profit analysis             │
        │  ✓ Ready for BI tools          │
        └────────────────────────────────┘
```

---

## Layer Descriptions

### **BRONZE LAYER** - Raw Data Ingestion
- **Purpose**: Store raw CSV data with minimal processing
- **Process**:
  - Read CSV file line by line
  - Load all records with data type mapping
  - Add metadata columns (LoadDate, RecordID)
  - Perform basic not-null checks
- **Table**: `Bronze_ProductSales`
- **Columns**: ProductID, ProductName, Category, UnitsSold, UnitPrice, TotalRevenue, Timestamp

### **SILVER LAYER** - Cleaned & Validated Data
- **Purpose**: Create a single source of truth with cleaned, validated data
- **Process**:
  - Remove duplicate records
  - Standardize text fields (trim whitespace, proper casing)
  - Validate numeric fields (prices > 0, units >= 0)
  - Handle missing values with business logic
  - Enrich with calculated columns (Cost, Profit)
- **Table**: `Silver_ProductSales`
- **Additional Tables**: `Silver_Categories`, `Silver_Products`
- **Key Transformations**:
  ```sql
  Profit = (TotalRevenue - (UnitsSold * UnitCost))
  ProfitMargin = (Profit / TotalRevenue) * 100
  ```

### **GOLD LAYER** - Business Analytics & Insights
- **Purpose**: Provide aggregated views for business decisions
- **Views/Tables**:
  1. **Gold_TopSellingProducts** - Products ranked by units sold
  2. **Gold_HighestProfitProducts** - Products ranked by profit amount
  3. **Gold_ProfitMarginAnalysis** - Products ranked by profit margin %
  4. **Gold_CategoryPerformance** - Sales/profit aggregated by category
  5. **Gold_ProductMetrics** - Comprehensive product KPI dashboard

---

## Setup Instructions for Students

### Prerequisites
- SQL Server 2019 or later
- SQL Server Management Studio (SSMS)
- SQL Server Data Tools (SSDT) for Visual Studio
- Sample CSV file with product data

### Step 1: Prepare the CSV File
Ensure your CSV has these columns:
```
ProductID, ProductName, Category, UnitsSold, UnitPrice, TotalRevenue, UnitCost
1, Laptop, Electronics, 50, 999.99, 49999.50, 600.00
2, Coffee Maker, Home, 120, 89.99, 10798.80, 35.00
...
```

### Step 2: Create Database & Bronze Schema
```sql
-- Create database
CREATE DATABASE [ETL_Pipeline]

-- Create Bronze schema and table
USE [ETL_Pipeline]
CREATE SCHEMA Bronze

CREATE TABLE Bronze_ProductSales (
    BronzeID INT IDENTITY(1,1) PRIMARY KEY,
    ProductID INT,
    ProductName NVARCHAR(255),
    Category NVARCHAR(100),
    UnitsSold INT,
    UnitPrice DECIMAL(10,2),
    TotalRevenue DECIMAL(15,2),
    UnitCost DECIMAL(10,2),
    LoadDate DATETIME DEFAULT GETDATE()
)
```

### Step 3: Create SSIS Package (Bronze Load)
1. Open SQL Server Data Tools
2. Create new SSIS Project
3. Create Data Flow Task:
   - **Source**: Flat File Connection to CSV
   - **Destination**: OLE DB Destination to Bronze_ProductSales
   - **Data Conversion**: Map CSV columns to table columns
   - **Error Handling**: Redirect bad rows to error table

### Step 4: Create Silver Schema & Transformations
```sql
-- Create Silver schema
CREATE SCHEMA Silver

-- Silver cleaned products table
CREATE TABLE Silver_ProductSales (
    SilverID INT IDENTITY(1,1) PRIMARY KEY,
    ProductID INT UNIQUE,
    ProductName NVARCHAR(255),
    Category NVARCHAR(100),
    UnitsSold INT CHECK (UnitsSold >= 0),
    UnitPrice DECIMAL(10,2) CHECK (UnitPrice > 0),
    UnitCost DECIMAL(10,2),
    TotalRevenue DECIMAL(15,2),
    Profit DECIMAL(15,2),
    ProfitMargin DECIMAL(5,2),
    CreatedDate DATETIME DEFAULT GETDATE(),
    UpdatedDate DATETIME
)

-- Silver Categories reference table
CREATE TABLE Silver_Categories (
    CategoryID INT IDENTITY(1,1) PRIMARY KEY,
    CategoryName NVARCHAR(100) UNIQUE,
    CreatedDate DATETIME DEFAULT GETDATE()
)
```

### Step 5: Create SSIS Package (Silver Transformation)
1. Create new Data Flow Task for Silver load
2. **Transformations**:
   - **Lookup Transform**: Deduplicate ProductID
   - **Derived Column**: Calculate Profit and ProfitMargin
   - **Data Conversion**: Ensure correct data types
   - **Conditional Split**: Validate business rules (price > 0, units >= 0)
   - **OLE DB Destination**: Load to Silver_ProductSales

### Step 6: Create Gold Schema & Analytics Views
```sql
-- Create Gold schema
CREATE SCHEMA Gold

-- Top Selling Products
CREATE VIEW Gold_TopSellingProducts AS
SELECT TOP 10
    ProductID,
    ProductName,
    Category,
    UnitsSold,
    TotalRevenue,
    RANK() OVER (ORDER BY UnitsSold DESC) AS SalesRank
FROM Silver_ProductSales
WHERE UnitsSold > 0
ORDER BY UnitsSold DESC

-- Highest Profit Products
CREATE VIEW Gold_HighestProfitProducts AS
SELECT TOP 10
    ProductID,
    ProductName,
    Category,
    Profit,
    ProfitMargin,
    RANK() OVER (ORDER BY Profit DESC) AS ProfitRank
FROM Silver_ProductSales
WHERE Profit > 0
ORDER BY Profit DESC

-- Profit Margin Analysis (Identify inefficient products)
CREATE VIEW Gold_ProfitMarginAnalysis AS
SELECT
    ProductID,
    ProductName,
    Category,
    Profit,
    ProfitMargin,
    CASE 
        WHEN ProfitMargin < 10 THEN 'Low Margin (Review)'
        WHEN ProfitMargin BETWEEN 10 AND 30 THEN 'Medium Margin'
        ELSE 'High Margin'
    END AS MarginStatus
FROM Silver_ProductSales
ORDER BY ProfitMargin ASC

-- Category Performance Summary
CREATE VIEW Gold_CategoryPerformance AS
SELECT
    Category,
    COUNT(DISTINCT ProductID) AS ProductCount,
    SUM(UnitsSold) AS TotalUnitsSold,
    SUM(TotalRevenue) AS TotalRevenue,
    SUM(Profit) AS TotalProfit,
    AVG(ProfitMargin) AS AvgProfitMargin
FROM Silver_ProductSales
GROUP BY Category

-- Comprehensive Metrics Dashboard
CREATE TABLE Gold_ProductMetrics AS
SELECT
    ProductID,
    ProductName,
    Category,
    UnitsSold,
    TotalRevenue,
    Profit,
    ProfitMargin,
    CASE WHEN UnitsSold > (SELECT AVG(UnitsSold) FROM Silver_ProductSales) 
         THEN 'Above Average' ELSE 'Below Average' END AS SalesPerformance,
    CASE WHEN Profit > (SELECT AVG(Profit) FROM Silver_ProductSales WHERE Profit > 0)
         THEN 'High Profit' ELSE 'Needs Review' END AS ProfitStatus
FROM Silver_ProductSales
```

### Step 7: Create SSIS Package (Gold Load)
1. Create Data Flow Task that executes views/populates tables
2. **Source**: OLE DB Source from Silver_ProductSales
3. **Transformations**: Aggregate transforms for category summaries
4. **Destination**: OLE DB Destination to Gold tables

### Step 8: Execute Master Package
Create a master SSIS package that calls all three packages in sequence:
```
Execute -> Bronze Package
   ↓ (on success)
Execute -> Silver Package
   ↓ (on success)
Execute -> Gold Package
```

---

## Expected Downstream Outputs

### Reports & Dashboards to Create

**1. Top 10 Selling Products Report**
- Shows ProductName, Category, UnitsSold, Revenue
- Helps with inventory planning & sales focus

**2. Profitability Analysis Report**
- Shows ProductName, Profit Amount, Profit Margin %
- Identifies high-margin vs. loss-making products
- Flags products with margin < 10%

**3. Category Performance Dashboard**
- Total sales, revenue, and profit by category
- Average profit margin per category
- Category contribution to overall profit

**4. Product Exceptions Report**
- Products with negative profit (review pricing)
- Products with zero or very low sales
- Products with price inconsistencies

---

## Testing Checklist

- [ ] Bronze load: All 100% of CSV rows loaded
- [ ] Silver load: No duplicates, all validations pass
- [ ] Gold views: Return expected results with correct aggregations
- [ ] Top 10 products display correctly
- [ ] Profit calculations verified manually
- [ ] Date/timestamp columns populated
- [ ] Error logs captured for failed records
- [ ] Package execution time < 2 minutes

---

## File Structure
```
ETL_Pipeline/
├── SSIS_Packages/
│   ├── ETL_Bronze.dtsx
│   ├── ETL_Silver.dtsx
│   ├── ETL_Gold.dtsx
│   └── ETL_Master.dtsx
├── SQL_Scripts/
│   ├── 01_CreateDatabase.sql
│   ├── 02_CreateBronzeSchema.sql
│   ├── 03_CreateSilverSchema.sql
│   ├── 04_CreateGoldViews.sql
│   └── 05_ValidationQueries.sql
├── SampleData/
│   └── ProductSales_Sample.csv
└── README.md
```

---

## Key Metrics to Monitor

| Metric | Bronze | Silver | Gold |
|--------|--------|--------|------|
| Record Count | Source Count | After Dedup | Aggregated |
| Data Quality | % Complete | % Valid | 100% |
| Processing Time | Extract | Transform | Load |
| Error Rate | Log All | Reject Bad | None Expected |

---

## Next Steps

1. **Implement**: Follow steps 1-8 above
2. **Test**: Execute master package with sample data
3. **Validate**: Query Gold views and verify results
4. **Document**: Add business logic notes for each transformation
5. **Schedule**: Set up SQL Agent job for daily/weekly execution
6. **Monitor**: Track package execution logs and data quality metrics

---

## Troubleshooting

**Common Issues**:
- *CSV encoding error*: Save CSV as UTF-8
- *Column mapping mismatch*: Verify CSV column order matches SSIS mapping
- *Duplicate key violation*: Check Silver uniqueness constraints
- *Package execution timeout*: Increase timeout in SSIS package settings
- *Zero rows in Gold*: Verify Silver data populated successfully

**Debug Steps**:
1. Check Data Viewer in SSIS packages
2. Query Bronze/Silver tables to verify data presence
3. Review SSIS execution logs in SQL Agent
4. Validate CSV file format and encoding

---

**Created**: 2026-10-05  
**Version**: 1.0  
**Author**: Data Engineering Team  
**Last Updated**: 2026-10-05
