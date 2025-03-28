# Unified Power BI Semantic Model: Advanced Business Analytics with DAX & M

## Table of Contents

- [The Need for a Comprehensive Data Model](#the-need-for-comprehensive-data-model)

- [Tools](#tools)

- [Tables](#tables)
  - [Date Table](#date-table)
  - [Sales Table](#sales-table)
  - [SO Table](#so-table-sales-orders)
  - [Purchases Table](#purchases-table)
  - [PO Table](#po-table-purchase-orders)

...

## The Need for a Comprehensive Data Model

Company faced significant challenges due to fragmented and inconsistent data scattered across multiple sources, such as sales records, purchase orders, inventory details, customer interactions, and financial accounts. This fragmentation hindered the company’s ability to generate timely, accurate, and comprehensive insights necessary for strategic decision-making. Establishing a unified, structured, and comprehensive data analytics model became crucial to streamline data integration, ensure data integrity, and empower Suspensia with actionable insights for improved business performance and informed decision-making.

## Tools

- PowerBI
- SQL Server

## Tables

Below is a list of tables loaded and structured within Power BI. Each table serves a distinct purpose and is either linked to SQL servers, loaded as-is from flat files or transformed through Power Query (M language). Some tables are created dynamically (e.g., Date Table), while others are merged or cleaned versions of raw exports from the company’s accounting system or supporting databases.

### Date Table

The Date Table serves as the foundational time dimension for the entire data model, enabling consistent and accurate time-based analytics and comparisons. It provides a standardized way to analyze data across different time periods such as yearly, quarterly, monthly, weekly, and daily views.

#### **Source:**
Created dynamically using **Power Query (M language)** within Power BI. This method ensures the date table automatically updates to include current dates.

#### **Columns Included:**
| Column Name   | Description                                      | Example        |
|---------------|--------------------------------------------------|----------------|
| `Date`        | Individual date entries                          | 2024-01-01     |
| `Year`        | Year number                                      | 2024           |
| `Quarter`     | Quarter name                                     | Q1, Q2, Q3, Q4 |
| `QuarterNo`   | Quarter number                                   | 1, 2, 3, 4     |
| `Month`       | Month name                                       | January        |
| `MonthNo`     | Month number                                     | 1              |
| `Week`        | Week number within the year                      | 1-52           |
| `WeekNo`      | Week number aligned to fiscal or calendar weeks  | 1-52           |
| `Day`         | Day of the month                                 | 1-31           |
| `DayOfWeekNo` | Numeric representation of the weekday            | 1 (Monday)     |

Below is the M code to dynamically generate a Date Table within Power BI, which automatically updates from January 1, 2018, to the current date:


```m language
let
    //Variables
    Start_Date = #date(2018, 1, 1),
    End_Date = DateTime.Date(DateTime.LocalNow()),
    Duration_VAR = Duration.Days(Duration.From(End_Date-Start_Date))+1,
    Dates = List.Dates(Start_Date,Duration_VAR,#duration(1,0,0,0)),
    #"Converted to Table" = Table.FromList(Dates, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    #"Renamed Columns" = Table.RenameColumns(#"Converted to Table",{{"Column1", "Date"}}),
    #"Changed Type" = Table.TransformColumnTypes(#"Renamed Columns",{{"Date", type date}}),
    #"Inserted Year" = Table.AddColumn(#"Changed Type", "Year", each Date.Year([Date]), Int64.Type),
    #"Inserted QuarterNo" = Table.AddColumn(#"Inserted Year", "QuarterNo", each Date.QuarterOfYear([Date]), Int64.Type),
    #"Custom Column for Quarter" = Table.AddColumn(#"Inserted QuarterNo", "Quarter", each "Q"&Text.From([QuarterNo])),
    #"Inserted Month" = Table.AddColumn(#"Custom Column for Quarter", "Month", each Text.Start(Date.MonthName([Date]),3), type text),
    #"Inserted MonthNo" = Table.AddColumn(#"Inserted Month", "MonthNo", each Date.Month([Date]), Int64.Type),
    #"Inserted WeekNo" = Table.AddColumn(#"Inserted MonthNo", "WeekNo", each Date.WeekOfYear([Date]), Int64.Type),
    #"Custom Column for Week" = Table.AddColumn(#"Inserted WeekNo", "Week", each "Week "&Text.From([WeekNo])),
    #"Inserted Day" = Table.AddColumn(#"Custom Column for Week", "Day", each Text.Start(Date.DayOfWeekName([Date]),3), type text),
    #"Inserted DayOfWeekNo" = Table.AddColumn(#"Inserted Day", "DayOfWeekNo", each Date.DayOfWeek([Date]), Int64.Type)
in
    #"Inserted DayOfWeekNo"
```

---

### Sales Table

#### Purpose
The **Sales Table** is the central repository capturing detailed transactional information related to product sales. It enables comprehensive analysis of sales performance, revenue generation, profitability, and customer purchasing behaviors over time. The CSV files containing sales data are automatically exported to the company server through scheduled exports from the accounting software, ensuring timely and accurate data integration.

#### Source and Creation Method
- **Source:** Multiple CSV files (`SALES_SUS_2018_2023.csv`, `SALES_SUS_2024.csv`, `SALES_SUS_2025.csv`)
- **Creation Method:** Imported, merged, cleaned, and structured using Power Query (M language).

#### (M) Code to Form Sales Table

The following M code is used to load, clean, and prepare sales data from three yearly CSV exports. Each step is explained below.

```powerquery-m
let
    // Combine multiple yearly CSV tables into one unified sales table
    Source = Table.Combine({SALES_SUS_2018_2023, SALES_SUS_2024, SALES_SUS_2025}),

    // Remove unnecessary or unused columns to simplify the model
    #"Removed Columns" = Table.RemoveColumns(Source,{
        "Apply to Invoice Number", "Progress Billing Invoice", "Ship By", "Quote", "Quote #", 
        "Quote Good Thru Date", "Ship Via", "Ship Date", "Date Due", "Sales Tax ID", 
        "Invoice Note", "Note Prints After Line Items", "Statement Note", 
        "Stmt Note Prints Before Ref", "Internal Note", "Beginning Balance Transaction", 
        "AR Date Cleared in Bank Rec", "Number of Distributions", "Invoice/CM Distribution", 
        "Apply to Invoice Distribution", "Apply To Sales Order", "Apply to Proposal", 
        "Serial Number", "SO/Proposal Distribution", "Weight", "Stocking Quantity", 
        "Stocking Unit Price", "Return Authorization", "Receipt Number", 
        "Voided by Transaction", "Retainage Percent", "Recur Number", "Recur Frequency", 
        "Ship to Name", "Ship to Address-Line One", "Ship to Address-Line Two", 
        "Discount Amount", "Discount Date", "Displayed Terms", "Accounts Receivable Account", 
        "Accounts Receivable Amount", "SO/Proposal Number", "GL Date Cleared in Bank Rec", 
        "Tax Type", "UPC / SKU", "Inv Acnt Date Cleared In Bank Rec", 
        "COS Acnt Date Cleared In Bank Rec", "U/M ID", "U/M No. of Stocking Units", 
        "Job ID", "Sales Tax Agency ID", "Transaction Period", "Transaction Number", 
        "Description", "Inventory Account"
    }),

    // Rename 'Item ID' to 'Sold AS (Item ID)' for clarity and consistency
    #"Item ID Renamed Sold AS" = Table.RenameColumns(#"Removed Columns",{{"Item ID", "Sold AS (Item ID)"}}),

    // Invert the sign of 'Amount' to convert negative values (used in accounting) to positives
    #"Amount Multiplied by (-1)" = Table.TransformColumns(#"Item ID Renamed Sold AS", {
        {"Amount", each _ * -1, Currency.Type}
    }),

    // Rename columns to match semantic model naming conventions
    #"Renamed Columns" = Table.RenameColumns(#"Amount Multiplied by (-1)", {
        {"Amount", "Sales_Amount"},
        {"Quantity", "Sales_Quantity"},
        {"Unit Price", "Sales_Unit Price"}
    }),

    // Ensure 'Cost of Sales Amount' is explicitly typed as a number
    #"Changed Type" = Table.TransformColumnTypes(#"Renamed Columns", {
        {"Cost of Sales Amount", type number}
    }),

    // Replace empty strings in item IDs with the literal string \"(blank)\"
    #"Blanks replaced with \"(blank)\"" = Table.ReplaceValue(#"Changed Type", "", "(blank)", Replacer.ReplaceValue, {"Sold AS (Item ID)"})
in
    #"Blanks replaced with \"(blank)\""
```

#### Detailed Column Descriptions

| Column Name             | Data Type | Description                                               | Example           |
|-------------------------|-----------|-----------------------------------------------------------|-------------------|
| `Customer ID`           | Text      | Unique identifier for the customer                        | CUST1001          |
| `Customer Name`         | Text      | Name of the customer                                      | ABC Corp.         |
| `Invoice/CM #`          | Text      | Invoice number or Credit Memo reference                   | INV-2023-10001    |
| `Date`                  | Date      | Date of the sales transaction                             | 2024-01-20        |
| `Credit Memo`           | Text      | Indicates if the transaction is a credit memo             | Yes / No          |
| `Drop Ship`             | Text      | Indicates if item was drop-shipped directly               | Yes / No          |
| `Sold AS (Item ID)`     | Text      | Identifier for the sold item                              | ITEM56789         |
| `Sales_Quantity`        | Number    | Quantity of the item sold                                 | 150               |
| `G/L Account`           | Text      | General Ledger account associated with sales revenue      | 4000-Sales Rev.   |
| `Sales_Unit Price`      | Currency  | Selling price per unit of the item                        | 49.99             |
| `Sales_Amount`          | Currency  | Total sales amount (Quantity × Unit Price)                | 7498.50           |
| `Cost of Sales Account` | Text      | G/L account linked to cost of sales                       | 5000-Cost of Sales|
| `Cost of Sales Amount`  | Currency  | Total cost of the items sold                              | 4000.00           |

#### Usage and Analytical Value
This table forms the backbone of sales analytics, allowing the following analyses:

- **Revenue Analysis:** Evaluating total revenue by period, item, customer, and region.
- **Profitability Analysis:** Determining margins, profitability per item, and cost management effectiveness.
- **Trend Analysis:** Monitoring sales trends to support forecasting and strategic planning.
- **Customer Insights:** Understanding customer buying behavior and segmentation.

#### Relationships (Brief Overview)
- **Linked to the Date Table** via the `Date` column for robust temporal analysis.
- **Linked to Item Information Table (`Item_Info`)** via `Sold AS (Item ID)` for enriched item-level analytics.
- **Linked to Customer Information** via `Customer ID` for customer demographic and segmentation insights.
- **Linked to General Ledger Accounts (`COA_CONS`)** via `G/L Account` to enable detailed financial analytics.

---

### SO Table (Sales Orders)

#### Purpose
The **SO Table** (Sales Orders) contains detailed records of open or historical sales orders, proposals, and quotes received from customers. It is primarily used to track future commitments, analyze sales pipelines, manage fulfillment expectations, and assess sales potential that hasn't yet been invoiced. This complements the Sales Table, which tracks completed transactions.

Sales Order data is exported from the accounting software to the company server in CSV format on a scheduled basis.

#### Source and Creation Method
- **Source Files:** `SO_SUS_2018_2023.csv`, `SO_SUS_2024.csv`, `SO_SUS_2025.csv`
- **Creation Method:** The CSVs are combined and transformed using Power Query (M). Unnecessary columns are removed, numeric values are type-corrected, and fields are renamed for clarity.

#### (M) Code to Form Sales Table

```m-language
let
    // Combine multiple year-based SO files into one table
    Source = Table.Combine({SO_SUS_2018_2023, SO_SUS_2024, SO_SUS_2025}),

    // Remove unnecessary or irrelevant columns from the combined data
    #"Removed Columns" = Table.RemoveColumns(Source, {
        "Ship By", "Proposal", "Proposal Accepted", "Closed", "Quote #",
        "Ship to Name", "Ship to Address-Line One", "Ship to Address-Line Two",
        "Ship to City", "Ship Via", "Sales Tax ID", "Invoice Note",
        "Note Prints After Line Items", "Statement Note", "Stmt Note Prints Before Ref",
        "Internal Note", "Number of Distributions", "SO/Proposal Distribution",
        "Weight", "U/M No. of Stocking Units", "Stocking Quantity", "Stocking Unit Price",
        "Sales Tax Agency ID", "Discount Amount", "Transaction Period",
        "Transaction Number", "U/M ID", "Tax Type", "Sales Order/Proposal #",
        "Ship to State", "Ship to Zipcode", "Ship to Country", "Customer PO",
        "Displayed Terms", "Accounts Receivable Account", "Accounts Receivable Amount",
        "UPC / SKU", "Job ID", "Description"
    }),

    // Ensure Amount and Unit Price columns are interpreted as Currency data types
    #"Changed Type" = Table.TransformColumnTypes(#"Removed Columns", {
        {"Amount", Currency.Type},
        {"Unit Price", Currency.Type}
    }),

    // Convert Amounts to negative values (likely to align with accounting standards or match invoice logic)
    #"Amount Multiplied by (-1)" = Table.TransformColumns(#"Changed Type", {
        {"Amount", each _ * -1, Currency.Type}
    }),

    // Rename columns to match semantic naming conventions used in the model
    #"Renamed Columns" = Table.RenameColumns(#"Amount Multiplied by (-1)", {
        {"Item ID", "Sold As (Item ID)"},
        {"Amount", "SO_Amount"},
        {"Quantity", "SO_Quantity"},
        {"Unit Price", "SO_Unit Price"}
    }),

    // Replace blank values in the Item ID column with the string "(blank)"
    #"Blanks replaced with \"(blank)\"" = Table.ReplaceValue(#"Renamed Columns", "", "(blank)", Replacer.ReplaceValue, {"Sold As (Item ID)"})
in
    #"Blanks replaced with \"(blank)\""
```

#### Detailed Column Descriptions

| Column Name             | Data Type | Description                                                | Example           |
|-------------------------|-----------|------------------------------------------------------------|-------------------|
| `Customer ID`           | Text      | Unique customer identifier                                 | CUST2034          |
| `Customer Name`         | Text      | Name of the customer placing the order                     | Beta Industries   |
| `Date`                  | Date      | Date when the sales order was entered                      | 2024-03-10        |
| `Drop Ship`             | Logical   | Indicates whether this is a drop-shipped order             | TRUE              |
| `Sales Representative ID` | Text    | Identifier of the responsible sales representative         | REP0003           |
| `Item ID` (renamed to `Sold As (Item ID)`) | Text | Identifier of the ordered item                 | ITEM32991         |
| `Quantity` (renamed to `SO_Quantity`) | Number | Quantity ordered                                 | 80                |
| `Unit Price` (renamed to `SO_Unit Price`) | Currency | Price per unit ordered                         | 45.00             |
| `Amount` (renamed to `SO_Amount`) | Currency | Total sales order value for the line item                 | 3600.00           |
| `G/L Account`           | Int64     | Financial ledger account linked to the order               | 4000              |

#### Usage and Analytical Value
- **Backlog Analysis:** Identifies committed revenue not yet realized.
- **Sales Pipeline Monitoring:** Tracks pending and open orders.
- **Demand Planning:** Helps forecast future inventory needs based on pending orders.
- **Performance Monitoring:** Evaluates quoting and proposal effectiveness over time.

#### Relationships (Brief Overview)
- **Linked to the Date Table** via the `Date` field for tracking open orders over time.
- **Linked to the Item Table (`Item_Info`)** via `Sold As (Item ID)` to analyze product-level demand.
- **Linked to Customer Information** via `Customer ID` to evaluate outstanding commitments by customer.

This table complements completed sales data by providing visibility into expected future business and supporting operational and financial forecasting.

---

### Purchases Table

#### Purpose
The **Purchases Table** records historical procurement transactions, capturing detailed information about goods or services acquired from vendors. It supports analysis of purchasing behavior, cost tracking, vendor performance, and inventory planning.

The table serves as a counterpart to the Sales Table and is essential for cost control, margin analysis, and operational decision-making.

#### Source and Creation Method
- **Source Files:** `PJ_SUS_2018_2024.csv`, `PJ_SUS_2025.csv`
- **Creation Method:** Combined and transformed using Power Query (M). Extraneous columns are removed, numeric types are enforced, and standard naming conventions are applied.

#### (M) Code to Form Purchases Table

Below is a breakdown of the M code used to generate and transform the `Purchases` table in Power BI.

```powerquery-m
let
    // Step 1: Combine purchase records from two CSV sources (multiple years)
    Source = Table.Combine({PJ_SUS_2018_2024, PJ_SUS_2025}),

    // Step 2: Explicitly define data types for relevant columns
    #"Changed Type" = Table.TransformColumnTypes(Source,{
        {"Credit Memo", type logical},
        {"Date", type date},
        {"Drop Ship", type logical},
        {"Waiting on Bill", type logical},
        {"Date Due", type date},
        {"Discount Date", type date},
        {"Discount Amount", type number},
        {"Accounts Payable Account", type text},
        {"Accounts Payable Amount", Currency.Type},
        {"Note Prints After Line Items", type logical},
        {"Applied To Purchase Order", type logical},
        {"Number of Distributions", Int64.Type},
        {"Invoice/CM Distribution", Int64.Type},
        {"Apply to Invoice Distribution", Int64.Type},
        {"PO Distribution", Int64.Type},
        {"Quantity", type number},
        {"Stocking Quantity", type number},
        {"U/M No. of Stocking Units", Int64.Type},
        {"G/L Account", type text},
        {"GL Date Cleared in Bank Rec", type date},
        {"Unit Price", Currency.Type},
        {"Stocking Unit Price", type number},
        {"UPC / SKU", type text},
        {"Weight", type number},
        {"Amount", Currency.Type},
        {"Transaction Number", Int64.Type},
        {"Transaction Period", Int64.Type},
        {"Used for Reimbursable Expense", type logical}
    }),

    // Step 3: Remove unnecessary columns to clean and reduce the dataset
    #"Removed Columns" = Table.RemoveColumns(#"Changed Type",{
        "Apply to Invoice Number", "Customer SO #", "Waiting on Bill", "Customer ID", "Customer Invoice #",
        "Ship to Address-Line One", "Ship to Address-Line Two", "Ship to City", "Ship to State",
        "Ship to Country", "Date Due", "Discount Date", "Discount Amount", "Ship Via", "P.O. Note",
        "Note Prints After Line Items", "Beginning Balance Transaction", "AP Date Cleared in Bank Rec",
        "Applied To Purchase Order", "Number of Distributions", "Apply to Invoice Distribution", "PO Number",
        "PO Distribution", "Stocking Quantity", "Serial Number", "U/M ID", "U/M No. of Stocking Units",
        "GL Date Cleared in Bank Rec", "Stocking Unit Price", "UPC / SKU", "Weight", "Job ID",
        "Used for Reimbursable Expense", "Transaction Period", "Transaction Number", "Displayed Terms",
        "Return Authorization", "Row Type", "Recur Number", "Recur Frequency", "Ship to Name",
        "Description", "Accounts Payable Amount", "Invoice/CM Distribution", "Ship to Zipcode",
        "Accounts Payable Account"
    }),

    // Step 4: Rename key columns for semantic clarity and consistency
    #"Renamed Columns" = Table.RenameColumns(#"Removed Columns",{
        {"Amount", "Purchase_Amount"},
        {"Quantity", "Purchase_Quantity"},
        {"Unit Price", "Purchase_Unit Price"}
    })
in
    #"Renamed Columns"
```

#### Detailed Column Descriptions

| Column Name             | Data Type | Description                                                | Example           |
|-------------------------|-----------|------------------------------------------------------------|-------------------|
| `Vendor ID`             | Text      | Unique identifier for the vendor                           | VEND1021          |
| `Vendor Name`           | Text      | Name of the vendor or supplier                             | Acme Suppliers     |
| `Invoice/CM #`          | Text      | Invoice or Credit Memo number                              | PJ-2023-00213      |
| `Date`                  | Date      | Date of the purchase transaction                           | 2024-01-05         |
| `Credit Memo`           | Text      | Indicates if the transaction is a credit memo              | Yes / No           |
| `Drop Ship`             | Logical   | True if goods were drop-shipped directly                   | TRUE               |
| `Item ID` (renamed to `Purchased As (Item ID)`) | Text | Identifier of the purchased item         | ITEM89021          |
| `Purchase_Quantity`     | Number    | Number of units purchased                                  | 120                |
| `Purchase_Unit Price`   | Currency  | Cost per unit purchased                                    | 22.50              |
| `Purchase_Amount`       | Currency  | Total purchase amount (Quantity × Unit Price)              | 2700.00            |
| `G/L Account`           | Text      | General Ledger account for the purchase transaction        | 5000-Cost of Sales |

#### Usage and Analytical Value
- **Cost Analysis:** Evaluates cost per item and purchasing trends.
- **Vendor Analytics:** Tracks performance and purchase volume by vendor.
- **Margin Monitoring:** Used in combination with Sales data to calculate gross profit.
- **Inventory Planning:** Informs demand forecasting and restocking cycles.

#### Relationships (Brief Overview)
- **Linked to the Date Table** via `Date` for time-based analysis.
- **Linked to the Item Table (`Item_Info`)** via `Purchased As (Item ID)` to understand item-level purchase behavior.
- **Linked to Vendor dimension** via `Vendor ID` for supplier-based analysis.
- **Linked to COA Table (`COA_CONS`)** via `G/L Account` for financial reporting and cost classification.

---

### PO Table(Purchase Orders)

#### Purpose
The **Purchase Orders Table** tracks procurement intents — orders placed with vendors for future delivery. Unlike the Purchases Table (which reflects completed transactions), this table captures **open commitments** and allows the business to manage expected inventory inflows, vendor reliability, and purchasing trends.

CSV files are automatically exported from the accounting system and uploaded to the company server as part of a scheduled integration routine.

#### Source and Creation Method
- **Source Files:** `POJ_2018_2024.csv`, `POJ_2025.csv`
- **Creation Method:** Combined using Power Query (M language), followed by removal of irrelevant fields and renaming for clarity and consistency.

### M Code to Form PO Table

The following M code loads, cleans, and transforms the Purchase Orders data from two annual sources (`POJ_2018_2024` and `POJ_2025`). This prepares it for use in Power BI’s semantic model.

```powerquery-m
let
    // Step 1: Combine two PO data sources into one unified table
    Source = Table.Combine({POJ_2018_2024, POJ_2025}),

    // Step 2: Remove unnecessary columns to keep only relevant fields for analysis
    #"Removed Columns" = Table.RemoveColumns(Source,{
        "Remit to Address Line 1", "Remit to Address Line 2", "Remit to City", "Remit to State",
        "Remit to Zip", "Remit to Country", "Closed", "P.O. Good Thru Date", "Customer SO #",
        "Customer Invoice #", "Customer ID", "Ship to Name", "Ship to Address-Line One",
        "Ship to Address-Line Two", "Ship to City", "Ship to State", "Ship to Zipcode",
        "Ship to Country", "Discount Amount", "Displayed Terms", "Accounts Payable Account",
        "Accounts Payable Amount", "Ship Via", "P.O. Note", "Internal Note",
        "Note Prints After Line Items", "Number of Distributions", "PO Distribution",
        "Stocking Quantity", "U/M ID", "U/M No. of Stocking Units", "Stocking Unit Price",
        "UPC / SKU", "Weight", "Job ID", "Transaction Period", "Transaction Number",
        "Description"
    }),

    // Step 3: Assign correct data types to key columns
    #"Changed Type" = Table.TransformColumnTypes(#"Removed Columns",{
        {"Amount", Currency.Type},
        {"Unit Price", Currency.Type},
        {"Quantity", Int64.Type},
        {"PO #", type text}
    }),

    // Step 4: Rename columns to match the semantic model's naming conventions
    #"Renamed Columns" = Table.RenameColumns(#"Changed Type",{
        {"Unit Price", "PO_Unit Price"},
        {"Amount", "PO_Amount"},
        {"Quantity", "PO_Quantity"}
    })
in
    #"Renamed Columns"
```

#### Detailed Column Descriptions

| Column Name       | Data Type | Description                                                | Example          |
|-------------------|-----------|------------------------------------------------------------|------------------|
| `Vendor ID`       | Text      | Unique identifier for the vendor                          | VEND0041         |
| `Vendor Name`     | Text      | Name of the vendor company                                | Global Parts Inc.|
| `PO #`            | Text      | Unique Purchase Order number                              | PO-2024-00891    |
| `Date`            | Date      | Date the purchase order was issued                        | 2024-03-15       |
| `Drop Ship`       | Logical   | Indicates if the order is to be shipped directly to customer | TRUE          |
| `Item ID`         | Text      | Identifier of the product ordered                         | ITEM47592        |
| `PO_Quantity`     | Number    | Number of units ordered                                   | 200              |
| `PO_Unit Price`   | Currency  | Unit cost of the ordered item                             | 11.25            |
| `PO_Amount`       | Currency  | Total order value (Quantity × Unit Price)                 | 2250.00          |
| `G/L Account`     | Int64     | General Ledger account used for PO financial tracking     | 5000             |

#### Usage and Analytical Value
- **Open Order Monitoring:** Understand what inventory is on the way and from which vendors.
- **Procurement Planning:** Measure purchasing frequency and lead times.
- **Vendor Performance:** Evaluate if suppliers fulfill POs as promised.
- **Cash Flow Planning:** Anticipate financial commitments before they're invoiced.

#### Relationships (Brief Overview)
- **Linked to the Date Table** via `Date` for time-based PO tracking.
- **Linked to the Item Table (`Item_Info`)** via `Item ID` to analyze product-level demand.
- **Linked to Vendor dimension** via `Vendor ID` to review supplier-based activity.
- **Linked to the COA Table (`COA_CONS`)** via `G/L Account` to associate POs with financial accounts.
