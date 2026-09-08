# warehouse-location-optimization
Power BI tool for identifying misplaced warehouse stock using data ingestion, transformation, modeling, and DAX querying

## Business Problem
Our warehouse consists of two primary areas: the **Self Serve warehouse**, where customers can retrieve products themselves, and the **Full Serve warehouse**, where products must be retrieved for customers after purchase. Some products are stored in elevated racking and must be retrieved with a forklift. We will refer to these storage locations as **PALLET locations**.

A common issue we have is when a **PALLET** product intended for Full Serve elevated racking is mistakenly stored in Self Serve racking. Because coworkers cannot operate forklifts on the Self Serve sales floor during customer hours, these products cannot be retrieved until the store closes and the floor is clear of customers.

These **after-hours picks** can delay order fulfillment and create additional work for our customer service team. They can also result in cancelled orders, compensated delivery fees, and an overall negative customer experience. Identifying these misplaced products *before* a customer attempts to purchase them would allow the warehouse team to proactively relocate the inventory to an appropriate Full Serve location.

## Solution
Because there is no reporting that directly indicates these misplaced pallets, I created a Power BI dashboard that identifies these pallets along with additional key information, such as total Full Serve quantity, registered racking locations, and sales history to support relocation decisions.

<img width="824" height="731" alt="image" src="https://github.com/user-attachments/assets/8738a85a-ada5-401b-adaa-4d64e7881087" />

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

Before modeling any data, I needed to be sure that the ingested data was standardized and formatted consistently. The four reports I am pulling from were not ready to function in a relational model. Power Query was used to clean, standardize, and reduce source data before creating relationships between the tables.

One of the first things I noticed is that some of our **SGF Locations** (our unique 6-digit elevated warehouse racking locations) were missing leading zeros. To correct this, I created custom columns in Power Query to standardize identifiers that were missing leading zeros. 

Here is an example of a custom column in one report that standardizes the SGF location:

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

**Location Reference Data** connects to **Storage Location Data** using `SGFLOCATION_fixed`. Each warehouse storage location is only listed once in the reference table; however, those storage locations can appear more than once in the **Storage Location Data** if more than one article is stored in one storage/racking location.

Finally, **Article & Stock Data** connects to **Sales Transaction Data** using `ArticleNo_fixed`. Each article is only listed once in **Article & Stock Data**, while that same article could appear under multiple transactions in **Sales Transaction Data**.

The **Article Summary** table is intentionally disconnected from the other tables in the model. This is because it is a calculated table that uses DAX measures to retrieve information from the underlying tables. I'll break this down further in the next portion.

## DAX Analytical Layer

After the source data was cleaned and relationships were established in the data model, I used DAX to create calculated columns, a calculated summary table, and measures that identify misplaced inventory and provide additional information that our team can use to prioritize relocation decisions.

## 1. Classifying storage locations

The first step was to classify each individual racking location based on both the sales method of the article stored there and the sales method classification of the location itself.

The following calculated column uses `RELATED()` to retrieve the article's **sales method**, or **SM** from the **Article & Stock Data** table and the location classification from the **Location Reference Data** table.

(For reference, **SM1** represents **Self Serve** and **SM2** represents **Full Serve**)

```dax
SM2_LocationType = 
VAR ArticleSM = RELATED('Article & Stock Data'[SALESMETHOD])
VAR LocationSM = RELATED('Location Reference Data'[SM code per SGF location])
RETURN
SWITCH(
    TRUE(),
    ArticleSM = 2 && LocationSM = 1, "SM1",
    ArticleSM = 2 && LocationSM = 2, "FS",
    BLANK()
)
```

So, if an **SM2** article is stored in an **SM1** location, the row is classified as SM1. If an **SM2** article is stored in an **SM2** location, it is classified as `FS`. Any other combination is ignored. This helps the model differentiate between misplaced inventory and correctly stored inventory.

## 2. Creating the Article Summary table

The source reports can contain multiple records for the same articles, but the final dashboard is designed to only display each article once. To avoid data complications, I created a calculated `Article Summary` table using `SUMMARIZE()` to ensure there is one row per article number and corresponding article name.

```dax
Article Summary = 
SUMMARIZE(
    Article & Stock Data,
    Article & Stock Data[ArticleNo_fixed],
    Article & Stock Data[ARTNAME_UNICODE]
)
```

Now, measures will be evaluated against each article in the summary table and can retrieve information on them from the other source tables.

## 3. Retrieving article attributes

The following two measures are then built into the Article Summary calculated table to retrieve the **Sales Method** and the **Primary Location (SLID)** of each article. This will tell us whether the article lives in **SM1** or in **SM2**, and then where the article lives on the sales floor if it is not a **PALLET** article.

```DAX
SM = 
VAR CurrentArticle =
    SELECTEDVALUE('Article Summary'[ArticleNo_fixed])
RETURN
CALCULATE(
    MAX(Article & Stock Data[SALESMETHOD]),
    FILTER(
        ALL(Article & Stock Data),
        Article & Stock Data[ArticleNo_fixed] = CurrentArticle
    )
)
```

```DAX
Primary Location = 
VAR CurrentArticle =
    SELECTEDVALUE('Article Summary'[ArticleNo_fixed])
RETURN
CALCULATE(
    MIN(Article & Stock Data[H_SLID]),
    FILTER(
        ALL(Article & Stock Data),
        Article & Stock Data[ArticleNo_fixed] = CurrentArticle
    )
)
```
Several measures here use the same `CurrentArticle` pattern. `SELECTEDVALUE()` retrieves the article that is being evaluated one at a time from the `Article Summary` table, while `CALCULATE()` and `FILTER()` use that article to retrieve its corresponding values from the underlying source data. In this case, the **Sales Method** and the **Primary Location (SLID)**.

## 4. Identifying misplaced articles

Now we get to identify the problem that inspired this dashboard: **SM2** pallets in **SM1** elevated racking locations.

First, we create a calculated column in our **Storage Location Data** table:

```DAX
IsMisplaced = 
VAR ArticleSM = RELATED(Article & Stock Data[SALESMETHOD])
VAR LocationSM = RELATED('Location Reference Data'[SM code per SGF location])
RETURN
IF(ArticleSM = 2 && LocationSM = 1, 1, 0)
```
This column evaluates each row of the **Storage Location Data** table and looks for any **SM2** articles that are stored in **SM1** locations. If this is the case, the measure returns 1, and if not, it returns 0.

Then, we create a measure in our Article Summary calculated table:

```DAX
Is Misplaced Article = 
VAR CurrentArticle =
    SELECTEDVALUE('Article Summary'[ArticleNo_fixed])
RETURN
IF(
    CALCULATE(
        COUNTROWS(Storage Location Data),
        FILTER(
            ALL(Storage Location Data),
            Storage Location Data[ArticleNo_fixed] = CurrentArticle &&
            Storage Location Data[IsMisplaced] = 1
        )
    ) > 0,
    1,
    0
)
```
This checks every article number with at least one storage location row where `IsMisplaced = 1`.

## 5. Combining multiple locations into one field

Since one article can sometimes be stored in several storage locations, this next measure's function uses `CONCATENATEX()` to list every storage location of the given article and separate them with commas rather than displaying one row for every location.

```DAX
SGF Locations = 
VAR CurrentArticle =
    SELECTEDVALUE('Article Summary'[ArticleNo_fixed])
VAR LocationTable =
    DISTINCT(
        SELECTCOLUMNS(
            FILTER(
                ALL(Storage Location Data),
                Storage Location Data[ArticleNo_fixed] = CurrentArticle &&
                NOT ISBLANK(Storage Location Data[SGF Location Display])
            ),
            "Loc", Storage Location Data[SGF Location Display]
        )
    )
RETURN
IF(
    ISBLANK(CurrentArticle),
    BLANK(),
    CONCATENATEX(
        LocationTable,
        [Loc],
        ", ",
        [Loc],
        ASC
    )
)
```

## 6. Calculating total quantity currently in Full Serve

Often because of space constraints, we are forced to store some **SM2** article in **SM1** locations. This is not an issue if these articles have sales locations on the floor that are consistently restocked. So while it is important for us to see all misplaced articles, it is crucial that we identify any **SM2** articles that have little or no stock accessible in **SM2**. This last measure displays the total quantity of each misplaced article that is available in Full Serve:

```DAX
QTY in FS = 
VAR CurrentArticle =
    SELECTEDVALUE('Article Summary'[ArticleNo_fixed])

VAR AvailStock =
    CALCULATE(
        MAX(Article & Stock Data[AVAIL_STOCK]),
        FILTER(
            ALL(Article & Stock Data),
            Article & Stock Data[ArticleNo_fixed] = CurrentArticle
        )
    )

VAR SGFStock =
    CALCULATE(
        MAX(Article & Stock Data[SGF_STOCK]),
        FILTER(
            ALL(Article & Stock Data),
            Article & Stock Data[ArticleNo_fixed] = CurrentArticle
        )
    )

VAR FSQty =
    CALCULATE(
        SUM(Storage Location Data[Qty]),
        FILTER(
            ALL(Storage Location Data),
            Storage Location Data[ArticleNo_fixed] = CurrentArticle &&
            Storage Location Data[SGF Location Display] = "FS"
        )
    )

RETURN
COALESCE(AvailStock, 0) - COALESCE(SGFStock, 0) + COALESCE(FSQty, 0)
```

This formula takes the total amount of stock in our store and subtracts the quantity stored in racking locations. It then adds back on the amount of that stock that is available in **SM2** racking locations.

For example, if we have 10 total pieces of an article as available stock with 7 of those pieces stored in the racking, this initial calculation leaves 3 pieces of stock on the floor. But if 2 of those racked pieces are stored in Full Serve (**SM2**) racking, that means those 2 pieces are added back, giving us 5 total pieces available in SM2.

**10 - 7  + 2 = 5**

## Using the Dashboard

<img width="824" height="731" alt="image" src="https://github.com/user-attachments/assets/8738a85a-ada5-401b-adaa-4d64e7881087" />

The final dashboard, as shown at the beginning, brings all of these calculations together into a single view of misplaced articles. For each article, coworkers can see its primary sales location, the total quantity available in Full Serve, and registered racking locations.

Not every misplaced article requires immediate action. As shown in the dashboard, I have sorted the `QTY in FS` column in ascending order. This brings all articles that have **0 stock in Full Serve** to the top of the list, showing us the most urgent candidates for relocation before the store opens. 

While this dashboard also includes sales history and forecasting for each article, this portion was redacted for data privacy reasons. The additional sales data provides even more context when making relocation decisions, allowing us to consider both **availability and expected demand** when deciding which pallets should be moved.

## Skills Demonstrated

This project shows the use of Power BI to transform several operational reports into an automated inventory analysis tool

**Power Query**
- SharePoint folder data ingestion
- Data cleaning and transformation
- Custom columns
- Identifier standardization with `Text.PadStart()`
- Data type management
- Column reduction

**Data Modeling**
- One-to-many relationships
- Article and location-based relationship keys
- Calculated summary table
- Relational data modeling

**DAX**
- Calculated columns
- Measures
- Variables (`VAR`)
- `RELATED()`
- `SUMMARIZE()`
- `SELECTEDVALUE()`
- `CALCULATE()`
- `FILTER()`
- `ALL()`
- `IF()` and `SWITCH()`
- `COUNTROWS()`
- `DISTINCT()`
- `SELECTCOLUMNS()`
- `CONCATENATEX()`
- `ISBLANK()` / `BLANK()`
- `COALESCE()`
- Aggregations including `SUM()`, `MIN()`, and `MAX()`
