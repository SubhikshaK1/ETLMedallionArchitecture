<h1>ETL Medallion Architecture</h1>
### Main Pipeline Logic: 
The primary flow (Source to Bronze to Silver to Gold) focuses on building a Star Schema using a sales dataset. In this main flow, the Silver layer uses a standard MERGE (Upsert) which effectively acts like an SCD Type 1 (overwriting old data). 
The Slowly Changing Dimensions (SCD) implementation is handled as a separate to dive deep, self-contained module rather than being integrated into the main end-to-end sales data pipeline.

### Source
The Source Layer represents the point of origin for the data pipeline. While typically an external system in production, it is simulated here within the Databricks environment to demonstrate end-to-end ELT.
The source acts as the Point A in the ETL journey. It is usually an OLTP (Online Transactional Processing) system, such as a SQL Server or a third-party API, optimized for high-frequency writes rather than analytical queries 

_Key Characteristics:_<br>
Normalized Format: OLTP sources use normalization to reduce redundancy and ensure data integrity.<br>
Volatile State: Data is frequently updated or deleted; it is the "raw" starting point before any data lake layers are applied <br>
The Data Engineer's Role: We do not typically create the OLTP source; we treat it as the input to create an OLAP (Online Analytical Processing) model.

1. Table Creation<br>
The source is established as a standard SQL table containing mixed data types to represent a complex real-world "One Big Table" (OBT) scenario<br>
2. Initial Data Load<br>
The first "historical" batch represents the existing state of the business before the pipeline was active.
3. Simulating Incremental Growth<br>
To test the pipeline's Incremental Loading capabilities, a secondary INSERT block is used to add new transactions with later dates (e.g., '2024-07-02')

The Source layer provides the raw data that will travel through the Medallion Architecture:<br>
Source: The raw, transactional input<br>

### <b>Bronze layer</b>
It serves as the initial entry point for data into our Medallion Architecture, moving data from the source system into a managed Delta Lake environment.<br>
The Bronze Layer acts as the landing zone for all raw data coming from external sources (APIs, CRM systems, On-premise DBs).<br>

Key Objectives:<br>
Data Preservation: Retain the full history of data in its original raw state. No heavy transformations are applied here to ensure we can re-process data if business logic changes.<br>
Source of Truth: It serves as the "System of Record".<br>
Incremental Ingestion: Instead of reloading all data every time, the Bronze layer uses Watermarking (incremental loads) to capture only the new records since the last successful run.<br>

The Bronze Notebook implements the following steps:<br>
1. Dynamic Watermarking (Python)<br>
To achieve Incremental Loading, the code first determines the last_load_date.<br>
Logic: It checks if bronze_table exists. If yes, it fetches the MAX(order_date). If no, it defaults to a historical date (1990-01-01).
2. The Extraction View (bronze_source)<br>
The code bridges Python variables into Spark SQL by creating a Temporary View. This view filters the source data to only include records newer than our watermark.<br>
3. Data Persistence (Delta Lake)<br>
The final step materializes the data into the managed bronze_table. This moves the data from a temporary memory state (View) to a physical, versioned Delta table.<br>

### <b>Silver Layer</b>
The Silver Layer is the "Quality Zone." Here, raw data from the Bronze layer is refined, standardized, and merged into a clean, queryable format.<br>
The primary goal of the Silver layer is to provide a "Single Source of Truth" for enterprise data.<br>

Key Objectives:
Data Cleansing: Standardizing formats (e.g., date formats, case sensitivity).<br>
Enrichment: Adding calculated columns (timestamps, metadata).<br>
Evolutionary Consistency: Ensuring that updates to existing records are handled gracefully through UPSERT operations.<br>

The Silver Notebook focuses on the transition from Bronze to Silver using Spark SQL and Delta Lake's MERGE capability.<br>

1. Transformation Logic<br>
Before saving, the data is enriched with business logic to ensure consistency.<br>
Standardization: Converting customer_name to uppercase for uniform reporting.<br>
Audit Trails: Adding a processDate using current_timestamp() to track exactly when a record was processed.<br>

2. The MERGE (UPSERT) Operation<br>
To avoid duplicate entries for the same order_id, the notebook uses a MERGE statement. This ensures that if an order is modified in the source, it is updated in Silver; otherwise, it is inserted as new.

### <b>Gold Layer</b>
The Gold Layer is the "Knowledge Zone." It contains high-level, aggregated data that is directly consumed by PowerBI, Tableau, or Data Science models.<br>
While Bronze and Silver are generally organized around technical structures, Gold is organized around Business Processes.<br>

Key Objectives:<br>
Performance: Data is often pre-aggregated to allow for sub-second query speeds in BI tools.
Business Logic: Complex joins between different entities (e.g., Orders joined with Customers and Products) are materialized here.<br>
Simplified Access: Instead of 50 tables, a business user might only see 5 "Gold" tables representing key metrics.<br>

The Gold Notebook transforms the normalized Silver data into actionable insights.<br>
1. Aggregation & Metrics<br>
In this layer, we shift from individual transactions to summary statistics.<br>
Example: Calculating total sales per day or most popular products.<br>

2. Data Denormalization<br>
Unlike the Source or Silver layers, Gold often "flattens" data. Instead of looking up a customer_id, the Gold table includes the full customer_name_upper and contact details directly in the reporting table to reduce the need for expensive runtime joins.
<img width="1706" height="906" alt="image" src="https://github.com/user-attachments/assets/b3568c2b-af59-4e34-b011-cd3c30f783eb" />

![Data Modelling Databricks zip file](./DataModelling.dbc) includes Source, Bronze, Silver, Gold, SCD Notebooks. 
The same is also uploaded as individual HTML files.
