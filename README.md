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
| **Elevated Racking Reference Data** | Classifies warehouse racking locations by their location type (Full Serve or Self Serve) |
| **Sales Transaction Data** | Provides historical sales transactions used to calculate article sales history |

## Data Transformation

Before modeling any data, I needed to be sure that the ingested data is standardized and has consistent formatting. The four reports I am pulling from were not ready to function in a relational mode. Power Query was used to clean, standardize, and reduce source data before creating relationships between the tables.

One of the first things I noticed is that some of our **SGF Locations** (our unique 6-digit elevated warehouse racking locations) were missing leading zeros. To correct this, I created custom columns in Power Query to standardize identifiers that were missing leading zeros. 

Here is an example of a custom column in one report that standardizes the SGF locaton:

```powerquery
= Table.AddColumn(
    #"Filtered Rows1",
    "SGFLOCATION_fixed",
    each Text.PadStart(Text.From([SGFLOCATION]), 6, "0")
)
```

I used the `Text.PadStart` function to ensure that every SGF location contains six characters. This function adds leading zeros to any SGF location that is not 6 characters until the required length is reached, ensuring consistent formatting across all source tables. This same standardization was repeated for article numbers, which are 8-digit unique identifiers for our products. 


| Identifier     | Standardized length | Purpose                                  |
| -------------- | ------------------: | ---------------------------------------- |
| SGF Location   |        6 characters | Match warehouse locations across reports |
| Article Number |        8 characters | Match articles across reports            |

After this standardization process, I removed columns that were not needed for the analysis and assigned appropriate data types where necessary. This reduced unnecessary data and further ensured consistent formatting.

## Data Modeling

Now that the data is cleaned, standardized, and ready to work with, I created relationships between each table. Below is the model view of the Power BI that displays the relationships of each table:

<img width="1383" height="1137" alt="FS in SS model view" src="https://github.com/user-attachments/assets/3539b76d-03c2-4df1-8e7f-11faa587b37c" />

Below is a table to describe the relationships:

| One Side                    | Relationship Key    | Many Side                  | Cardinality       |
| --------------------------- | ------------------- | -------------------------- | ----------------- |
| **Article & Stock Data**    | `ArticleNo_fixed`   | **Storage Location Data**  | One-to-Many (1:*) |
| **Location Reference Data** | `SGFLOCATION_fixed` | **Storage Location Data**  | One-to-Many (1:*) |
| **Article & Stock Data**    | `ArticleNo_fixed`   | **Sales Transaction Data** | One-to-Many (1:*) |

The first relationship connects **Article & Stock Data** to **Storage Location Data** using `ArticleNo_fixed`. **Article & Stock Data** occupies the one side because each article is only shown once, while **Storage Location Data** occupies the many side because a single article can be stored in multiple storage/racking locations.

**Location Reference Data** connects to **Storage Location Data** using `SGFLOCATION_fixed`. Each warehouse storage location in only listed once in the reference table; however, those storage locations can appear more than once in the **Storage Location Data** if more than one article is stored in one storage/racking loation.

Finally, **Article & Stock Data** connects to **sales Transaction Data** using `ArticleNo_fixed`. Each article is only listed once in **Article & Stock Data**, while that same article could appear under multiple transactions in **Sales Transaction Data**.

The **Article Summary** table is intentionally disconnected from the other tables in the model. This is because it is a calculated table that uses DAX measures to retrieve information from the underlying tables. I'll break this down further in the next portion.

## DAX Analytical Layer




