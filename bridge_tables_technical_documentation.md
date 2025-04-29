# Data Vault: Bridge Tables

## Modules

- [1. Introduction](#1-introduction)
- [2. Bridge Table Types](#2-bridge-table-types)
- [3. AS_OF_DATE Tables](#3-as_of_date-tables)
- [4. Standard Bridge Tables](#4-standard-bridge-tables)
- [5. Bridge Tables with Aggregation](#5-bridge-tables-with-aggregation)
- [6. Bridge Tables with Effectivity Satellites](#6-bridge-tables-with-effectivity-satellites)
- [7. Implementation Examples](#7-implementation-examples)
- [8. Best Practices](#8-best-practices)

## 1. Introduction

Bridge tables are specialized structures within the Business Vault layer of a Data Vault architecture. They span across a Hub and one or more associated Links, acting as higher-level fact-less fact tables containing hash keys from the hubs and links they span. Their primary purpose is to improve query performance by reducing the number of required joins for complex queries to simple equi-joins.

As described by Scalefree, "Bridge tables act as a higher-level fact-less fact table and contains hash keys from the hubs and links it spans. Another performance improvement can be achieved by moving resource intensive computations to the bridge table."

### Key Benefits

- **Performance Optimization**: Pre-join entities to reduce the number of joins required
- **Simplified Querying**: Convert complex multi-table joins to simple equi-joins
- **Grain Manipulation**: Can materialize grain shifts (e.g., aggregating transaction details)
- **Pre-calculated Metrics**: Can include aggregations and computations at the appropriate grain

## 2. Bridge Table Types

### Standard Bridge Tables

Standard Bridge tables contain hash keys from the Hub and Links they span, providing a simplified view across multiple entities. They do not typically contain descriptive data from Satellites.

### Aggregation Bridge Tables

Bridge tables can contain aggregated values added to the structure and loaded using GROUP BY statements. This results in a bridge table that has a higher grain than the links included in the table. These are particularly useful for reporting at different levels of granularity.

### Bridge Tables with Effectivity

These Bridge tables incorporate time-based aspects by joining with Effectivity Satellites, which track when relationships begin and end. This allows for accurate point-in-time querying of relationships between entities.

## 3. AS_OF_DATE Tables

Similar to Point-in-Time tables, Bridge tables often work with an AS_OF_DATE table that serves as a date dimension for temporal analysis. This is particularly important for Bridge tables that incorporate effectivity.

### Structure

```sql
CREATE TABLE queryassistance.as_of_date (
    as_of DATE NOT NULL,              -- Primary date field for point-in-time reference
    year SMALLINT NOT NULL,           -- Year component
    month SMALLINT NOT NULL,          -- Month number (1-12)
    month_name CHAR(10),              -- Month name 
    day_of_month SMALLINT NOT NULL,   -- Day of month (1-31)
    day_of_week VARCHAR(9) NOT NULL,  -- Day of week (0-6)
    day_name CHAR(10),                -- Name of day
    week_of_year SMALLINT NOT NULL,   -- Week number within year
    day_of_year SMALLINT NOT NULL,    -- Day number within year
    month_lastday SMALLINT NOT NULL,  -- Flag for last day of month
    week_lastday SMALLINT NOT NULL,   -- Flag for last day of week
    week_firstday SMALLINT NOT NULL   -- Flag for first day of week
);
```

## 4. Standard Bridge Tables

### Structure

A standard Bridge table contains hash keys from the Hub and Links it spans:

```sql
CREATE TABLE business_vault.bridge_customer_order (
    bridge_customer_order_id INT IDENTITY(1,1), -- Surrogate key for the bridge table
    dv_hashkey_hub_customer BINARY(20),         -- Hash key for the customer hub
    dv_hashkey_link_customer_order BINARY(20),  -- Hash key for the customer-order link
    dv_hashkey_hub_order BINARY(20),            -- Hash key for the order hub
    snapshotdate DATE,                          -- The date when this snapshot was taken
    load_date TIMESTAMP_LTZ(0)                  -- Load timestamp
);
```

### How Bridge Tables Work

Bridge tables pre-join entities to reduce the number of joins required for complex queries. A standard bridge table might span from a customer to their orders, pre-joining the customer hub, customer-order link, and order hub. This allows for simpler reporting queries that don't require multiple joins.

## 5. Bridge Tables with Aggregation

### Structure

An aggregation bridge table adds aggregated metrics to the standard structure:

```sql
CREATE TABLE business_vault.bridge_customer_order_aggregated (
    bridge_customer_order_id INT IDENTITY(1,1), -- Surrogate key
    dv_hashkey_hub_customer BINARY(20),         -- Customer hash key
    snapshot_date DATE,                         -- Snapshot date
    
    -- Aggregated metrics
    order_count INT,                            -- Count of orders
    total_order_value DECIMAL(18,2),            -- Sum of order values
    avg_order_value DECIMAL(18,2),              -- Average order value
    last_order_date DATE,                       -- Date of last order
    
    load_date TIMESTAMP_LTZ(0)                  -- Load timestamp
);
```

### Use Cases

Aggregation bridge tables are commonly used to:
- Roll up transaction details to header level
- Pre-compute metrics for reporting
- Implement different grains for analysis

A common business example is aggregating invoice line items to the invoice level. While line items would be stored in the Raw Data Vault (finer grain), the bridge table could aggregate these to invoice level (coarser grain) for reporting.

## 6. Bridge Tables with Effectivity Satellites

### Structure

A bridge table with effectivity incorporates temporal aspects:

```sql
CREATE TABLE business_vault.bridge_customer_order_effectivity (
    bridge_id INT IDENTITY(1,1),                -- Surrogate key
    dv_hashkey_hub_customer BINARY(20),         -- Customer hash key
    dv_hashkey_link_customer_order BINARY(20),  -- Link hash key
    dv_hashkey_hub_order BINARY(20),            -- Order hash key
    
    -- Effectivity dates
    relationship_start_date TIMESTAMP_NTZ(9),   -- When relationship started
    relationship_end_date TIMESTAMP_NTZ(9),     -- When relationship ended (null if active)
    is_current BOOLEAN,                         -- Flag for current relationship
    
    snapshot_date DATE,                         -- Snapshot date
    load_date TIMESTAMP_LTZ(0)                  -- Load timestamp
);
```

### How Effectivity Works with Bridge Tables

Effectivity satellites track when relationships between entities begin and end. By incorporating this data into bridge tables, you can accurately represent the state of relationships at specific points in time. This is particularly valuable for handling changing relationships in a historical context.

## 7. Implementation Examples

### Example 1: Creating a Standard Bridge Table

A basic bridge table spanning customer to order:

```sql
CREATE TABLE business_vault.bridge_customer_order AS
SELECT 
    CURRENT_DATE AS snapshot_date,
    c.dv_hashkey_hub_customer,
    co.dv_hashkey_link_customer_order,
    o.dv_hashkey_hub_order,
    CURRENT_TIMESTAMP AS load_date
FROM
    raw_vault.hub_customer c
JOIN raw_vault.link_customer_order co
    ON c.dv_hashkey_hub_customer = co.dv_hashkey_hub_customer
JOIN raw_vault.hub_order o
    ON o.dv_hashkey_hub_order = co.dv_hashkey_hub_order;
```

### Example 2: Creating an Aggregation Bridge Table

```sql
CREATE TABLE business_vault.bridge_customer_orders_aggregated AS
SELECT 
    CURRENT_DATE AS snapshot_date,
    c.dv_hashkey_hub_customer,
    COUNT(DISTINCT o.dv_hashkey_hub_order) AS order_count,
    SUM(os.order_amount) AS total_order_value,
    AVG(os.order_amount) AS avg_order_value,
    MAX(os.order_date) AS last_order_date,
    CURRENT_TIMESTAMP AS load_date
FROM
    raw_vault.hub_customer c
JOIN raw_vault.link_customer_order co
    ON c.dv_hashkey_hub_customer = co.dv_hashkey_hub_customer
JOIN raw_vault.hub_order o
    ON o.dv_hashkey_hub_order = co.dv_hashkey_hub_order
JOIN raw_vault.sat_order os
    ON o.dv_hashkey_hub_order = os.dv_hashkey_hub_order
GROUP BY
    c.dv_hashkey_hub_customer;
```

### Example 3: Bridge Table with Effectivity

```sql
CREATE TABLE business_vault.bridge_customer_order_effectivity AS
SELECT 
    CURRENT_DATE AS snapshot_date,
    c.dv_hashkey_hub_customer,
    co.dv_hashkey_link_customer_order,
    o.dv_hashkey_hub_order,
    eff.start_date AS relationship_start_date,
    eff.end_date AS relationship_end_date,
    CASE WHEN eff.end_date IS NULL THEN TRUE ELSE FALSE END AS is_current,
    CURRENT_TIMESTAMP AS load_date
FROM
    raw_vault.hub_customer c
JOIN raw_vault.link_customer_order co
    ON c.dv_hashkey_hub_customer = co.dv_hashkey_hub_customer
JOIN raw_vault.hub_order o
    ON o.dv_hashkey_hub_order = co.dv_hashkey_hub_order
JOIN raw_vault.eff_sat_customer_order eff
    ON co.dv_hashkey_link_customer_order = eff.dv_hashkey_link_customer_order;
```

## 8. Best Practices

### Design Principles

1. **Focus on Performance**: Create bridge tables to address specific performance bottlenecks
2. **Appropriate Grain**: Choose the right level of aggregation for the business need
3. **Clearly Define Relationships**: Document the entities and links that the bridge table spans
4. **Include Zero Records**: Consider including zero/unknown records to handle NULL values in joins

### Maintenance

1. **Refresh Strategy**: Determine how often bridge tables need to be refreshed
2. **Monitor Usage**: Track which bridge tables are most frequently used
3. **Version Control**: Maintain scripts that build bridge tables in version control
4. **Documentation**: Document the structure, purpose, and refresh cycle of each bridge table

### Performance Considerations

1. **Indexing**: Properly index bridge tables based on common query patterns
2. **Partitioning**: Consider partitioning large bridge tables by date
3. **Materialization**: Decide between materialized tables and views based on size and refresh needs
4. **Query Monitoring**: Monitor queries that use bridge tables to identify optimization opportunities

### Integration with Other Business Vault Components

1. **Combine with PIT Tables**: Consider combining bridge tables with Point-in-Time tables for complex temporal queries
2. **Information Mart Integration**: Design bridge tables with downstream information mart requirements in mind
3. **Business Calculations**: Use bridge tables to implement consistent business calculations across the enterprise
