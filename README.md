# ETLMedallionArchitecture
Source<br>
The Source Layer represents the point of origin for the data pipeline. While typically an external system in production, it is simulated here within the Databricks environment to demonstrate end-to-end ELT.

📖 Theory: The Source System<br>
The source acts as the Point A in the ETL journey. It is usually an OLTP (Online Transactional Processing) system, such as a SQL Server or a third-party API, optimized for high-frequency writes rather than analytical queries 

Key Characteristics:<br>
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

<b>Bronze layer<b>
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

Code Reference: ```python
if spark.catalog.tableExists('datamodelling.bronze.bronze_table'):
last_load_date = spark.sql("SELECT MAX(order_date) FROM ...").collect()[0][0]
else:
last_load_date = "1990-01-01"


2. The Extraction View (bronze_source)
The code bridges Python variables into Spark SQL by creating a Temporary View. This view filters the source data to only include records newer than our watermark.

Code Reference:

Python
spark.sql(f"SELECT * FROM datamodelling.default.source_data WHERE order_date > '{last_load_date}'") \
     .createOrReplaceTempView("bronze_source")
3. Data Persistence (Delta Lake)
The final step materializes the data into the managed bronze_table. This moves the data from a temporary memory state (View) to a physical, versioned Delta table.

Code Reference:

SQL
CREATE OR REPLACE TABLE datamodelling.bronze.bronze_table AS 
SELECT * FROM bronze_source
🛠 Project Structure
Source: Contains the initial raw data setup.

Bronze (Current): Handles incremental ingestion and historical raw storage.

Silver: Applies cleaning and UPSERT logic.

Gold: Aggregates data for business insights.

🚀 Evolution: From DBAs to Modern Data Engineering
The notebook notes a significant shift in methodology:

Historical: Historically, this logic was handled via Stored Procedures managed by DBAs in on-premise OLTP systems.

Modern: We now use Spark SQL and Python in the cloud (Databricks), allowing for better scalability and integration with Delta Lake features like Time Travel.
