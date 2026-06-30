# DineDash — Food Delivery Analytics Pipeline

A Databricks data engineering project that builds an end-to-end analytics pipeline for a food delivery platform using the Medallion Architecture (Bronze → Silver → Gold).

---

## Project Structure

```
dinedash/
├── Bronze.ipynb                        # Manual bronze ingestion notebook
├── Silver.ipynb                        # Manual silver transformation notebook
├── Dinedash_DLT_PL.ipynb              # Lakeflow Spark Declarative Pipeline (automated)
├── Order Data Analysis.ipynb           # Exploratory order analysis
├── Orders and Dimension Dataset Analysis.ipynb  # Dimension + orders analysis
├── Clean_Orders.sql                    # Saved SQL query for cleaned orders
├── Dine Dash Metric Dashboard          # AI/BI dashboard
└── README.md
```

---

## Data Sources

All raw data is stored in the Unity Catalog volume `/Volumes/suresh_db/dinedash/source/`.

| File | Format | Description |
|---|---|---|
| `orders/orders_2024_01.json` | JSON | January 2024 orders |
| `orders/orders_2024_02.json` | JSON | February 2024 orders |
| `dimensions/dim_customers.csv` | CSV | Customer master data |
| `dimensions/dim_restaurants.csv` | CSV | Restaurant master data |
| `dimensions/dim_delivery_agents.csv` | CSV | Delivery agent master data |
| `dimensions/dim_locations.csv` | CSV | Location reference data |
| `dimensions/dim_menu_items.csv` | CSV | Menu item catalog |

---

## Architecture: Medallion Layers

### Bronze — Raw Ingestion

Raw data is loaded as-is into Delta tables under `suresh_db.dinedash`, with two added metadata columns: `ingestion_time` and `source_file_name`.

| Table | Source |
|---|---|
| `orders_bronze` | JSON orders files (via `COPY INTO` or streaming) |
| `customers_bronze` | `dim_customers.csv` |
| `restaurants_bronze` | `dim_restaurants.csv` |
| `delivery_agents_bronze` | `dim_delivery_agents.csv` |
| `locations_bronze` | `dim_locations.csv` |
| `menu_items_bronze` | `dim_menu_items.csv` |

### Silver — Cleaned & Normalized

Null/invalid records are filtered out. Tables are enriched by joining fact orders with dimension tables.

| Table | Description |
|---|---|
| `customers_filtered` | Customers with all required fields present (9,999 / 10,050 records) |
| `restaurants_filtered` | Restaurants with valid core fields (150 / 200 records) |
| `locations_filtered` | Locations — all records passed validation |
| `delivery_agents_filtered` | Agents with valid id, name, phone, and rating |
| `cleaned_orders` | Orders passing DQ constraints (valid order_id, customer_id, items, restaurant) |
| `orders_silver` | Fully enriched orders joined with customers, restaurants, agents, and locations |
| `orders_with_agents` | Orders joined with agent name and rating |
| `orders_with_items` | Orders exploded by line item, joined with menu item details |
| `orders_with_locations` | Orders joined with delivery city, state, and area |

### Gold — Aggregated Metrics

Business-level aggregations produced from `orders_silver`.

| Table | Description |
|---|---|
| `total_orders_per_restaurant` | Order count per restaurant |
| `top_selling_items` | Top 10 menu items by total quantity sold |
| `avg_fees_by_city` | Average restaurant rating and tip per city |
| `agent_order_counts` | Total orders handled per delivery agent |

---

## Pipeline Automation

The `Dinedash_DLT_PL` notebook defines a **Lakeflow Spark Declarative Pipeline** that automates the full Bronze → Silver → Gold flow with:
- Streaming ingestion of orders using `STREAM READ_FILES`
- Data quality constraints with `EXPECT ... ON VIOLATION DROP ROW`
- Incremental processing via `CREATE OR REFRESH STREAMING TABLE`

The `Bronze` and `Silver` notebooks contain the equivalent manual SQL steps used for development and exploration.

---

## Catalog & Schema

- **Catalog:** `suresh_db`
- **Schema:** `dinedash`
- **Volume:** `/Volumes/suresh_db/dinedash/source/`

---

## Dashboard

The **Dine Dash Metric Dashboard** visualizes key metrics sourced from the gold layer tables.
