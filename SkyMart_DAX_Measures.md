# SkyMart Power BI — DAX Measures Reference

All measures organized page-wise. Copy-paste directly into Power BI's New Measure editor.

---

## 1. Sales Analysis / Executive Overview

```DAX
Total Sales Amount = SUM(Sales[SalesAmount])

Total Profit = SUM(Sales[Profit])

Profit Margin % = DIVIDE([Total Profit], [Total Sales Amount], 0)

Total Orders = COUNTROWS(Sales)
```

### Month over Month (MoM) — Sales

```DAX
Sales Previous Month = 
CALCULATE([Total Sales Amount], DATEADD('Date'[Date], -1, MONTH))

Sales MoM Change = 
[Total Sales Amount] - [Sales Previous Month]

Sales MoM % = 
DIVIDE([Sales MoM Change], [Sales Previous Month], 0)
```

### Month over Month (MoM) — Profit

```DAX
Profit Previous Month = 
CALCULATE([Total Profit], DATEADD('Date'[Date], -1, MONTH))

Profit MoM Change = 
[Total Profit] - [Profit Previous Month]

Profit MoM % = 
DIVIDE([Profit MoM Change], [Profit Previous Month], 0)
```

### Overall Trend Summary

```DAX
Avg Monthly Sales Growth = 
AVERAGEX(
    VALUES('Date'[MonthYear]),
    [Sales MoM %]
)
```

---

## 2. Product and Category Performance

### Contribution %

```DAX
% of Total Sales = 
DIVIDE(
    [Total Sales Amount],
    CALCULATE([Total Sales Amount], ALL(Products)),
    0
)
```

### Ranking

```DAX
Product Rank by Sales = 
RANKX(ALL(Products[ProductName]), [Total Sales Amount], , DESC)

Category Rank by Sales = 
RANKX(ALL(Products[Category]), [Total Sales Amount], , DESC)

Store Rank by Sales = 
RANKX(ALL(Stores[StoreName]), [Total Sales Amount], , DESC)

Region Rank by Sales = 
RANKX(ALL(Stores[Region]), [Total Sales Amount], , DESC)
```

### Top N Flag

```DAX
Top 5 Products Flag = 
IF([Product Rank by Sales] <= 5, "Top 5", "Others")
```

### Top Performer KPI Cards

```DAX
Top Product = 
VAR TopProductTable = 
    TOPN(
        1,
        ALL(Products[ProductName]),
        [Total Sales Amount],
        DESC
    )
RETURN
    CALCULATE(
        SELECTEDVALUE(Products[ProductName]),
        TopProductTable
    )

Top Category = 
VAR TopCategoryTable = 
    TOPN(
        1,
        ALL(Products[Category]),
        [Total Sales Amount],
        DESC
    )
RETURN
    CALCULATE(
        SELECTEDVALUE(Products[Category]),
        TopCategoryTable
    )

Top Product Sales Value = 
VAR TopProductTable = 
    TOPN(1, ALL(Products[ProductName]), [Total Sales Amount], DESC)
RETURN
    CALCULATE([Total Sales Amount], TopProductTable)
```

---

## 3. Returns Analysis

### Base Measures

```DAX
Total Returns = COUNTROWS(Returns)

Return Rate = 
DIVIDE([Total Returns], [Total Orders], 0)

Total Returned Quantity = SUM(Returns[Quantity])

Total Sales Quantity = SUM(Sales[Quantity])

Quantity Return Rate = 
DIVIDE([Total Returned Quantity], [Total Sales Quantity], 0)
```

### Financial Impact

```DAX
Total Return Value = 
SUMX(Returns, Returns[Quantity] * RELATED(Products[UnitPrice]))

Return Value % of Sales = 
DIVIDE([Total Return Value], [Total Sales Amount], 0)
```

### Trend

```DAX
Returns MoM Change = 
VAR CurrentMonth = [Total Returns]
VAR PreviousMonth = 
    CALCULATE([Total Returns], DATEADD('Date'[Date], -1, MONTH))
RETURN
    DIVIDE(CurrentMonth - PreviousMonth, PreviousMonth, 0)
```

### Ranking & Flags

```DAX
Rank by Return Rate = 
RANKX(ALL(Products[ProductName]), [Return Rate], , DESC)

High Return Flag = 
IF([Return Rate] > 0.15, "High Risk", "Normal")
```

### Avg Days to Return (requires ReturnDate + OrderDate columns)

```DAX
Avg Days to Return = 
AVERAGEX(Returns, DATEDIFF(RELATED(Sales[OrderDate]), Returns[ReturnDate], DAY))
```

### Top Return Performer KPI Cards

```DAX
Highest Return Product = 
VAR TopReturnTable = 
    TOPN(
        1,
        ALL(Products[ProductName]),
        [Return Rate],
        DESC
    )
RETURN
    CALCULATE(
        SELECTEDVALUE(Products[ProductName]),
        TopReturnTable
    )

Highest Return Rate Value = 
VAR TopReturnTable = 
    TOPN(1, ALL(Products[ProductName]), [Return Rate], DESC)
RETURN
    CALCULATE([Return Rate], TopReturnTable)
```

### Category / Store Breakdown (why analysis)

```DAX
Category Return Rate = 
DIVIDE(
    CALCULATE([Total Returns]),
    CALCULATE([Total Orders])
)

Return Rate by Store = 
DIVIDE(
    CALCULATE([Total Returns]),
    CALCULATE([Total Orders])
)

Return Count by Reason = COUNTROWS(Returns)
```

---

## 4. Inventory Dashboard

### Base Measures

```DAX
Stock on Hand = SUM(Inventory[QuantityOnHand])

Reorder Level = SUM(Products[ReorderLevel])

Max Stock Level = SUM(Products[MaxStockLevel])
```

### Stock Status

```DAX
Stock Status = 
VAR CurrentStock = [Stock on Hand]
VAR MinLevel = [Reorder Level]
VAR MaxLevel = [Max Stock Level]
RETURN
    SWITCH(
        TRUE(),
        CurrentStock < MinLevel, "Understocked",
        CurrentStock > MaxLevel, "Overstocked",
        "Optimal"
    )
```

### Counts (per Product level)

```DAX
Understocked Items Count = 
COUNTROWS(
    FILTER(
        Products,
        [Stock Status] = "Understocked"
    )
)

Overstocked Items Count = 
COUNTROWS(
    FILTER(
        Products,
        [Stock Status] = "Overstocked"
    )
)

Optimal Stock Items Count = 
COUNTROWS(
    FILTER(
        Products,
        [Stock Status] = "Optimal"
    )
)
```

### Counts (per Product + Store level — use if Inventory_Snapshot is at Product+Store granularity)

```DAX
Understocked Items Count = 
COUNTROWS(
    FILTER(
        Inventory_Snapshot,
        Inventory_Snapshot[QuantityOnHand] < RELATED(Products[ReorderLevel])
    )
)
```

### Stock Coverage & Days of Supply

```DAX
Stock Coverage % = 
DIVIDE([Stock on Hand], [Reorder Level], 0)

Avg Daily Sales = 
DIVIDE([Total Sales Quantity], DISTINCTCOUNT(Sales[OrderDate]), 0)

Days of Stock Remaining = 
DIVIDE([Stock on Hand], [Avg Daily Sales], 0)
```

---

## 5. Shipment Operations

### Calculated Column (in Shipments table — needed before measures)

```DAX
Delivery Delay (Days) = 
DATEDIFF(Shipments[ExpectedDeliveryDate], Shipments[ActualDeliveryDate], DAY)

On Time Flag = 
IF(Shipments[Delivery Delay (Days)] <= 0, "On Time", "Late")
```

### Core Measures

```DAX
Total Shipments = COUNTROWS(Shipments)

On Time Shipments = 
CALCULATE(
    COUNTROWS(Shipments),
    Shipments[On Time Flag] = "On Time"
)

On Time Delivery % = 
DIVIDE([On Time Shipments], [Total Shipments], 0)
```

### If DeliveryStatus column already exists (alternative to above)

```DAX
On Time Shipments = 
CALCULATE(
    COUNTROWS(Shipments),
    Shipments[DeliveryStatus] = "On Time"
)

On Time Delivery % = 
DIVIDE([On Time Shipments], [Total Shipments], 0)
```

### Delay Analysis

```DAX
Average Delay Days = 
AVERAGEX(
    FILTER(Shipments, Shipments[Delivery Delay (Days)] > 0),
    Shipments[Delivery Delay (Days)]
)

Late Shipments Count = 
CALCULATE(COUNTROWS(Shipments), Shipments[On Time Flag] = "Late")
```

---

## 6. Customer Satisfaction

### Base Measures

```DAX
Average Satisfaction Score = 
AVERAGE(CustomerSurvey[Rating])

Total Responses = 
COUNTROWS(CustomerSurvey)
```

### Region / Category Breakdown (same measure, use on different visual axes)

```DAX
Satisfaction by Region = 
CALCULATE(
    [Average Satisfaction Score]
)

Satisfaction by Category = 
CALCULATE(
    [Average Satisfaction Score]
)

Avg Rating by Question = 
CALCULATE([Average Satisfaction Score])
```

### Top / Bottom KPI Cards

```DAX
Lowest Satisfaction Region = 
VAR LowestTable = 
    TOPN(1, ALL(Stores[Region]), [Average Satisfaction Score], ASC)
RETURN
    CALCULATE(SELECTEDVALUE(Stores[Region]), LowestTable)

Highest Satisfaction Region = 
VAR HighestTable = 
    TOPN(1, ALL(Stores[Region]), [Average Satisfaction Score], DESC)
RETURN
    CALCULATE(SELECTEDVALUE(Stores[Region]), HighestTable)
```

### Satisfaction Level Label

```DAX
Satisfaction Level = 
SWITCH(
    TRUE(),
    [Average Satisfaction Score] >= 4, "Satisfied",
    [Average Satisfaction Score] >= 3, "Neutral",
    "Dissatisfied"
)
```

---

## 7. Budget vs Actual

### Base Measures

```DAX
Total Budget = SUM(Budget[TargetAmount])

Actual Sales = SUM(Sales[SalesAmount])
```

### Variance

```DAX
Variance = [Actual Sales] - [Total Budget]

Variance % = 
DIVIDE([Variance], [Total Budget], 0)

Budget Achievement % = 
DIVIDE([Actual Sales], [Total Budget], 0)
```

### Status Flag

```DAX
Budget Status = 
SWITCH(
    TRUE(),
    [Variance %] >= 0, "Meeting/Exceeding Target",
    [Variance %] >= -0.10, "Slightly Below Target",
    "Below Target"
)
```

### Cumulative / YTD Tracking

```DAX
Cumulative Actual = 
CALCULATE(
    [Actual Sales],
    FILTER(
        ALL('Date'[Date]),
        'Date'[Date] <= MAX('Date'[Date])
    )
)

Cumulative Budget = 
CALCULATE(
    [Total Budget],
    FILTER(
        ALL('Date'[Date]),
        'Date'[Date] <= MAX('Date'[Date])
    )
)
```

### Best / Worst Performer KPI Cards

```DAX
Best Performing Region = 
VAR BestTable = 
    TOPN(1, ALL(Stores[Region]), [Variance %], DESC)
RETURN
    CALCULATE(SELECTEDVALUE(Stores[Region]), BestTable)

Worst Performing Region = 
VAR WorstTable = 
    TOPN(1, ALL(Stores[Region]), [Variance %], ASC)
RETURN
    CALCULATE(SELECTEDVALUE(Stores[Region]), WorstTable)
```

> ⚠️ Note: Budget table relates to Sales indirectly — link via **Date table**, **Region (from Stores)**, and **Category (from Products)** using many-to-one relationships, since Budget is at a different granularity than Sales.

---

## Notes on Column Dependencies

Some measures assume specific columns exist in your data. Adjust names to match your actual files:

| Measure needs | Table.Column |
|---|---|
| Reorder threshold | `Products[ReorderLevel]`, `Products[MaxStockLevel]` |
| Return reason | `Returns[ReturnReason]` |
| Return date | `Returns[ReturnDate]`, `Sales[OrderDate]` |
| Shipment dates | `Shipments[ExpectedDeliveryDate]`, `Shipments[ActualDeliveryDate]` |
| Survey rating | `CustomerSurvey[Rating]` |
| Budget target | `Budget[TargetAmount]` |
| Unit price | `Products[UnitPrice]` |
