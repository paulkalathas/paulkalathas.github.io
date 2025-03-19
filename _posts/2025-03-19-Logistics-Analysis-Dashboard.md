---
layout: post
title: Logistics Analysis Dashboard
image: "/posts/Power-BI-Dashboard.png"
tags: [Make, API, Data Science]
---
Introduction
---

> Dashboard Link: "img/posts/Logistics-Dataset-Analysis.pbit"

Each DAX query below is structured with a detailed explanation of its purpose, calculations, and where to use it in Power BI.

1️⃣ Total Freight Cost
🔹 Purpose: Calculates the total freight cost of all shipments.
📌 Used in: KPI Card

```DAX
Total Freight Cost = SUM(FullDataset[Minimum_Cost])
```
🔍 Explanation:

The SUM function adds up all values in the Minimum_Cost column.
This represents the total freight cost of all orders.


2️⃣ Average Freight Cost per Shipment
🔹 Purpose: Calculates the average freight cost per shipment.
📌 Used in: KPI Card

```DAX
Avg Freight Cost per Shipment = 
DIVIDE(
    SUM(FullDataset[Minimum_Cost]), 
    COUNT(FullDataset[Order_ID]), 
    0
)
```
🔍 Explanation:

SUM(FullDataset[Minimum_Cost]) calculates total freight cost.
COUNT(FullDataset[Order_ID]) counts the number of shipments.
DIVIDE(..., 0) safely divides to prevent errors if shipments = 0.


3️⃣ Order Fulfillment %
🔹 Purpose: Calculates the percentage of orders that were shipped on time.
📌 Used in: KPI Card

```DAX
Order Fulfillment % = 
DIVIDE(
    COUNTROWS(FILTER(FullDataset, FullDataset[Ship_Late_Day_count] = 0)), 
    COUNTROWS(FullDataset), 
    0
) * 100
```
🔍 Explanation:

FILTER(FullDataset, Ship_Late_Day_count = 0): Finds orders that were not late.
COUNTROWS(...): Counts the number of on-time orders.
DIVIDE(..., 0) * 100: Converts the result into a percentage.


4️⃣ Late Shipment %
🔹 Purpose: Calculates the percentage of orders shipped late.
📌 Used in: KPI Card

```DAX
Late Shipment % = 
DIVIDE(
    COUNTROWS(FILTER(FullDataset, FullDataset[Ship_Late_Day_count] > 0)), 
    COUNTROWS(FullDataset), 
    0
) * 100
```
🔍 Explanation:

Similar to Order Fulfillment %, but filters for orders with delays (Ship_Late_Day_count > 0).
Calculates the % of late shipments.


5️⃣ Warehouse Utilization %
🔹 Purpose: Measures warehouse usage based on current vs. available capacity.
📌 Used in: Gauge Chart

```DAX
Warehouse Utilization % = 
VAR TotalCapacity = SUM(FullDataset[Daily_Capacity])
VAR UsedCapacity = SUM(FullDataset[Unit_quantity])
VAR Utilization = DIVIDE(UsedCapacity, TotalCapacity, 0) * 100

RETURN 
IF(Utilization > 100, 100, Utilization)  -- Ensures max is 100%
```
🔍 Explanation:

SUM(Daily_Capacity): Finds total warehouse capacity.
SUM(Unit_quantity): Finds used warehouse space.
DIVIDE(..., 0) * 100: Converts to percentage.
IF(Utilization > 100, 100, Utilization): Caps utilization at 100%.
6️⃣ Transport Mode Count
🔹 Purpose: Counts shipments by transport mode (Air, Ground, etc.).
📌 Used in: Donut Chart

```DAX
Transport Mode Count = COUNT(FullDataset[Transport_Mode])
```
🔍 Explanation:

Counts the number of shipments per transport mode (AIR, GROUND).
Used in a Pie/Donut Chart for transport mode breakdown.


7️⃣ Top 5 Customers by Order Volume
🔹 Purpose: Identifies top 5 customers based on order count.
📌 Used in: Table or Bar Chart

```DAX
Top 5 Customers = 
VAR TopCustomers =
    TOPN(
        5, 
        SUMMARIZE(
            FullDataset, 
            FullDataset[Customer], 
            "Order Count", COUNT(FullDataset[Order_ID])
        ), 
        "Order Count", 
        DESC
    )
RETURN
    TopCustomers
```
🔍 Explanation:

SUMMARIZE(...) groups data by Customer and counts orders.
TOPN(5, ...) returns the top 5 customers based on highest order count.


8️⃣ Total Orders by Customer
🔹 Purpose: Displays total order count per customer.
📌 Used in: Bar Chart

```DAX
Total Order Count by Customer = COUNT(FullDataset[Order_ID])
```
🔍 Explanation:

Counts all orders in the dataset.
Used in Bar Chart visualization to compare order volume per customer.
