# Earthquake-Azure-Databricks-ETL-Pipeline

Earthquake Data ETL Pipeline (Azure Databricks)

An end-to-end batch ETL pipeline that pulls daily earthquake events from the USGS Earthquake API and refines them through a medallion architecture (Bronze → Silver → Gold) in Azure Data Lake Storage Gen2, using Databricks and PySpark.

Architecture
USGS API (GeoJSON)
      │
      ▼
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Bronze  │ ──▶ │  Silver  │ ──▶ │   Gold   │      ADLS Gen2, one container per layer
│ raw JSON │     │ cleaned  │     │ enriched │
└──────────┘     └──────────┘     └──────────┘
      └──────── Databricks Job (3 dependent tasks) ────────┘
Layer	Notebook	What it does	Output
Bronze	Bronze Notebook	Calls the USGS API for the previous day's events and saves the raw GeoJSON features	bronze/{date}_earthquake_data.json
Silver	Silver Notebook	Flattens the nested GeoJSON into columns, handles nulls, and converts Unix millisecond timestamps to timestamps	Parquet in silver/earthquake_events_silver/
Gold	Gold Notebook	Adds a country code via reverse geocoding and classifies significance as Low (<100), Moderate (100–499) or High (500+)	Parquet in gold/earthquake_events_gold/

The three notebooks run as tasks in a single Databricks Job. The Bronze task passes the run dates and storage paths to the later tasks with dbutils.jobs.taskValues, so every task processes the same day.

Tech stack

Python · PySpark · Azure Databricks · Azure Data Lake Storage Gen2 · Parquet · requests · reverse_geocoder

Gold table schema
Column	Description
id	USGS event ID
longitude, latitude, elevation	Event coordinates and depth
title, place_description	Event title and location text
sig, mag, magType	Significance, magnitude and magnitude type
time, updated	Event time and last-update time
country_code	ISO country code from reverse geocoding
sig_class	Low, Moderate or High
Setup
Create an ADLS Gen2 storage account with three containers: bronze, silver and gold.
Give your Databricks cluster access to the storage account, and replace the storage account name in the Bronze notebook with your own.
Install the reverse_geocoder library on the cluster.
Import the three notebooks into your Databricks workspace.
Create a Databricks Job with three tasks named Bronze, Silver and Gold, each depending on the previous one. The task names must match, because the notebooks read task values by these keys.
Run the job manually, or schedule it to run daily.
Possible improvements
Use Delta Lake with MERGE on id so reruns and USGS event updates don't create duplicates
Partition the Silver and Gold tables by event date
Add data-quality checks (schema validation, row counts between layers) and failure alerts
Replace the row-by-row geocoding UDF with a batched pandas UDF for speed
