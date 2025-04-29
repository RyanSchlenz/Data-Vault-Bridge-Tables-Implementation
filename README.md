# Data-Vault-Bridge-Tables-Implementation

This repository contains modular SQL scripts for implementing Bridge tables in a Data Vault architecture.

## What are Bridge Tables?

Bridge tables are specialized structures within the Business Vault layer of a Data Vault architecture that span across a Hub and one or more associated Links. They improve query performance by pre-joining entities and reducing the number of joins needed for complex queries. Bridge tables act as higher-level fact-less fact tables and contain hash keys from the hubs and links they span.

## Implementation Scripts

Follow these scripts in order for a complete implementation:

1. [`01_as_of_date_setup.sql`](./01_as_of_date_setup.sql) - Creates and populates the AS_OF_DATE table (date dimension)
2. [`02_bridge_table_setup.sql`](./02_bridge_table_setup.sql) - Creates the Bridge table structure
3. [`03_bridge_population.sql`](./03_bridge_population.sql) - Populates the Bridge table
4. [`04_bridge_with_aggregation.sql`](./04_bridge_with_aggregation.sql) - Shows how to create Bridge tables with aggregated values
5. [`05_information_mart_views.sql`](./05_information_mart_views.sql) - Creates information mart views using Bridge tables
6. [`06_verification_queries.sql`](./06_verification_queries.sql) - Verifies that Bridge tables were created and populated correctly
7. [`07_bridge_with_effectivity.sql`](./07_bridge_with_effectivity.sql) - Implements a Bridge table with Effectivity Satellites

## Technical Documentation

For detailed information on the concepts and implementation, see the [Technical Documentation](./docs/bridge_tables_technical_documentation.md).

## Benefits of Bridge Tables

- **Performance Optimization**: Reduce the number of joins required for complex queries
- **Simplified Querying**: Convert complex joins to simple equi-joins
- **Grain Manipulation**: Can materialize grain shifts (e.g., aggregating line items to invoices)
- **Pre-calculated Metrics**: Can include aggregations and computations for faster reporting

## Usage

These scripts are designed to be modular. You can run them sequentially for a complete implementation, or selectively implement certain components based on your needs.

## Requirements

- These scripts are optimized for Snowflake but can be adapted for other database platforms
- Basic understanding of Data Vault concepts (Hubs, Links, Satellites)
- Appropriate permissions to create and populate tables

## Contributing

Feel free to submit issues or pull requests with improvements or extensions to these scripts.
