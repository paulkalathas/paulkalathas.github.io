---
layout: post
title: Creating a Logistics Dataset with SQL for Data Analysis
image: "/posts/logistics-analysis.png"
tags: [Python, Primes, Data Science]
---
# Introduction

Managing and analyzing logistics data efficiently requires structured datasets that integrate multiple data sources. This SQL query constructs a comprehensive dataset by combining various logistics-related tables, ensuring data integrity and completeness. The resulting dataset, FullDataset, is optimized for analysis in tools like Power BI.

Understanding the SQL Query

This SQL query retrieves and cleans logistics data by joining multiple tables in logisticsdb. It consolidates information about customer orders, freight costs, warehouse capacities, and customer details.

Selected Columns

The query retrieves the following key attributes:

1. Order Information: Order_ID, Order_Date, Origin_Port, Destination_Port, Carrier, Service_Level

2. Customer & Product Details: Customer, Verified_Customers, Product_ID, Plant_Code

3. Freight Costs & Transport Details: Minimum_Cost, Freight_Rate, Transport_Mode, Transport_Days

4. Warehouse Data: Daily_Capacity, Cost_Per_Unit, Port

---
# Table Joins and Data Cleaning

# Joining Freight Rates (freightrates):

The freightrates table contains freight cost details for shipments. The join is performed using Origin_Port and Destination_Port from orderlist, ensuring accurate freight cost mapping.


```SQL
LEFT JOIN logisticsdb.freightrates f 
    ON o.Origin_Port = f.orig_port_cd 
    AND o.Destination_Port = f.dest_port_cd
```

> Data Cleaning:

Ensures all orders have freight rate data.

rate and minimum cost are retained for cost analysis.

#Integrating Warehouse Capacity and Costs (warehousecapacities & warehousecosts)

The warehouse-related tables provide data about storage and handling costs. The query aggregates values using MAX() to avoid duplicates while maintaining correctness.

```SQL
LEFT JOIN (
    SELECT 
        w.Plant_Code,  
        MAX(w.Daily_Capacity) AS Daily_Capacity,  
        MAX(wc.Cost_Per_Unit) AS Cost_Per_Unit  
    FROM logisticsdb.warehousecapacities w
    LEFT JOIN logisticsdb.warehousecosts wc 
        ON w.Plant_Code = wc.Plant_Code  
    GROUP BY w.Plant_Code
) AS w 
    ON o.Plant_Code = w.Plant_Code
```

> Data Cleaning:

Uses MAX() to avoid multiple records per plant.

Ensures Plant_Code is correctly mapped across both tables.

# Assigning Ports to Orders (plantports)

The plantports table contains port locations associated with plants. To handle multiple ports, GROUP_CONCAT() is used, concatenating multiple port names into a single field.

```SQL
LEFT JOIN (
    SELECT 
        pp.Plant_Code, 
        GROUP_CONCAT(DISTINCT pp.Port SEPARATOR ', ') AS Port  
    FROM logisticsdb.plantports pp
    GROUP BY pp.Plant_Code
) AS l 
    ON o.Plant_Code = l.Plant_Code
```

> Data Cleaning:

Uses GROUP_CONCAT() to merge multiple port names into a single field.

Ensures each plant has its corresponding port(s) in a readable format.

# Handling Customer Data (vmicustomers)

To ensure correct customer mapping, the query first tries to match Customer directly. If a customer match is unavailable, it defaults to Plant_Code.

```SQL
LEFT JOIN logisticsdb.vmicustomers vm
    ON o.Customer = vm.Customer  
LEFT JOIN logisticsdb.vmicustomers vm2
    ON o.Plant_Code = vm2.Plant_Code;
```

> Data Cleaning:

Uses COALESCE(vm.Customer, vm2.Customer, 'Unknown Customer') to fill missing values.

Ensures all records have a valid customer name.

# Full SQL Query

```SQL
SELECT 
    o.Order_ID, 
    o.Order_Date, 
    o.Origin_Port, 
    o.Destination_Port, 
    o.Carrier, 
    o.Service_Level, 
    o.Ship_ahead_day_count, 
    o.Ship_Late_Day_count, 
    o.Customer,  
    COALESCE(vm.Customer, vm2.Customer, 'Unknown Customer') AS Verified_Customers,  
    o.Product_ID, 
    o.Plant_Code,  
    o.Unit_quantity, 
    o.Weight, 
    f.orig_port_cd, 
    f.dest_port_cd, 
    f.`minimum cost` AS Minimum_Cost, 
    f.rate AS Freight_Rate, 
    f.mode_dsc AS Transport_Mode, 
    f.tpt_day_cnt AS Transport_Days,
    w.Daily_Capacity, 
    w.Cost_Per_Unit, 
    l.Port
FROM logisticsdb.orderlist o
LEFT JOIN logisticsdb.freightrates f 
    ON o.Origin_Port = f.orig_port_cd 
    AND o.Destination_Port = f.dest_port_cd
LEFT JOIN (
    SELECT 
        w.Plant_Code,  
        MAX(w.Daily_Capacity) AS Daily_Capacity,  
        MAX(wc.Cost_Per_Unit) AS Cost_Per_Unit  
    FROM logisticsdb.warehousecapacities w
    LEFT JOIN logisticsdb.warehousecosts wc 
        ON w.Plant_Code = wc.Plant_Code  
    GROUP BY w.Plant_Code
) AS w 
    ON o.Plant_Code = w.Plant_Code
LEFT JOIN (
    SELECT 
        pp.Plant_Code, 
        GROUP_CONCAT(DISTINCT pp.Port SEPARATOR ', ') AS Port  
    FROM logisticsdb.plantports pp
    GROUP BY pp.Plant_Code
) AS l 
    ON o.Plant_Code = l.Plant_Code
LEFT JOIN logisticsdb.vmicustomers vm
    ON o.Customer = vm.Customer  
LEFT JOIN logisticsdb.vmicustomers vm2
    ON o.Plant_Code = vm2.Plant_Code;
```

# Conclusion

This SQL query efficiently structures logistics data by merging multiple sources, cleaning inconsistencies, and ensuring referential integrity. The FullDataset is now optimized for business analysis, performance tracking, and visualization in Power BI or other analytical tools.

With this dataset, analysts can:

1. Track freight costs per shipment.

2. Analyze warehouse capacities and costs.

3. Optimize supply chain operations.

4. Improve customer service and delivery efficiency.

