# warehouse-location-optimization
Power BI tool for identifying misplaced warehouse stock using data ingestion, transformation, modeling, and DAX querying

## Business Problem
Our warehouse consists of two primary areas: the **Self Serve warehouse**, where customers can retrieve products themselves, and the **Full Serve warehouse**, where products must be retrieved for customers after purchase. Some products are stored in elevated racking and must be retrieved with a forklift. We will refer to these storage locations as **PALLET locations**.

A common issue we have is when a **PALLET** product intended for Full Serve elevated racking is mistakenly stored in Self Serve racking. Because coworkers cannot operate forklifts on the Self Serve sales floor during customer hours, these products cannot be retrieved until the store closes and the floor is clear of customers.

These **after-hours picks** can delay order fulfillment and create additional work for our customer service team. They can also result in cancelled orders, compensated delivery fees, and an overall negative customer experience. Identifying these misplaced products *before* a customer attempts to purchase them would allow the warehouse team to proactively relocate the inventory to an appropriate Full Serve location.

## Solution
Because there is no reporting that directly indicates these misplaced pallets, I created a Power BI dashboard that identifies these pallets along with additional key information, such as total Full Serve quantity, registered racking locations, and sales history to support relocation decisions.

While there was previously an Excel tool that served the same purpose, that tool required manually copying and pasting data from multiple reports and was prone to breaking. This Power BI is set up to refresh every day at 6:30 AM and is connected to reporting that also refreshes every day. This eliminates the manual data preparation required by the Excel tool.

Coworkers and leadership are now able to simply access the Power BI dashboard, and make educated decisions on what products to bring back to the Full Serve racking.

## Data Ingestion
This dashboard combines data from four separate reports that provide information about article sales method (whether it belongs in Full Serve or Self Serve), inventory locations, stock quantities, and historical sales activity. These reports are stored as Excel files in SharePoint and are set to auto export every day to their SharePoint folders, replacing the file from the previous day and allowing the Power BI to ingest the updated data every morning. Power BI connects to these SharePoint folders as its data sources, which allows the semantic model to receive the updated data.

(Historical sales data, forecasting, and data source table names have been redacted for data privacy reasons)

### Source Data

The dashboard combines four Excel-based reports which are stored in SharePoint and updated daily. For confidentiality, the internal report names have been replaced with descriptive names throughout this project.

| Source | Purpose |
| --- | --- |
| **Article & Stock Data** | Provides article information, sales method, available stock, and primary sales location |
| **Storage Location Data** | Provides every sales and racking location in the store along with what is stored there |
| **Elevated Racking Reference Data** | Classifies warehouse racking locations by their location (Full Serve or Self Serve) |
| **Sales Transaction Data** | Provides historical sales transactions used to calculate article sales history |

## Data Transformation

Before modeling any data, I needed to be sure that the ingested data is standardized and has consistent formatting.

One of the first things I noticed is that some of our article numbers (our unique 8-digit article idenfitiers) were missing leading zeros. 
