# Olist E-Commerce Data Pipeline

An end-to-end data pipeline built on the Olist Brazilian e-commerce dataset — covering database design, data quality enforcement, cleaning, and delivery-performance analysis in MySQL and Python.

## Project Overview

This project takes the raw Olist dataset (orders, customers, products, order items, reviews, payments, geolocation) and builds it into a properly structured relational database, then cleans and analyzes it to answer a focused question: **how does delivery performance affect customer satisfaction, and where does it break down?**

**Pipeline stages:** Data collection → MySQL storage & schema design → Data cleaning → Analysis → Visualization

## Tech Stack

- **MySQL** — relational storage, schema design, constraint enforcement
- **Python** — pandas, SQLAlchemy, matplotlib
- **Jupyter Notebook** — cleaning and analysis workflow

## Database Design Decisions

A few deliberate schema decisions were made while building this out, rather than accepting the raw dataset's structure as-is:

### 1. Geolocation table — index instead of primary key
The geolocation table has duplicate rows for its zip-code-prefix column, so it can't serve as a primary key. It was set up as an **index** instead, kept for fast lookups without forcing a false uniqueness constraint.

### 2. Composite key on `order_items`
Neither `order_id` nor `order_item_id` is unique on its own — an order can contain multiple items, each numbered starting from 1 within that order. The two columns together form the natural **composite primary key**.

### 3. Orphaned rows surfaced by foreign key enforcement
While adding a foreign key between `orders` and `customers`, constraint validation surfaced **13 orphaned records** (0.01% of orders) referencing customer IDs that didn't exist in the customers table. After confirming no recoverable customer data existed for these rows, they were removed — an issue that would have gone completely undetected in a pandas-only cleaning pipeline, since pandas has no concept of a foreign key to violate.

### 4. Customers–geolocation relationship — evaluated, not enforced
A foreign key between `customers` and `geolocation` was considered but not implemented. The geolocation table has incomplete zip code coverage and no natural unique key, so enforcing this constraint would have meant dropping otherwise-valid customer records to satisfy a supplementary reference table. An index was used instead, and the coverage gap is documented here rather than silently patched over.

## Data Cleaning Highlights

Beyond standard null-checking, cleaning focused on catching *disguised* data quality issues that simple `.isnull()` checks miss:

- **Hidden blank categories**: `product_category_name` showed 0 nulls via `.isnull()`, but 610 rows (1.85%) were empty strings rather than true nulls — standardized to proper `NaN`/labeled `unknown`.
- **Impossible zero values**: 6 products had `0` for weight and/or dimensions — physically impossible for a real item. Investigated individually: 4 were matched to a near-identical product by category and dimensions and imputed with its weight (3100g); the remaining 2 had no reliable reference and were left null.
- **Orphaned payment records**: 3 order items (1 order) were marked `delivered` with zero matching rows in `order_payments` — confirmed as a genuine source-data gap via direct lookup, not a join error. Left null rather than estimated.
- **Incomplete delivery trail**: 8 orders marked `delivered` had no `order_delivered_customer_date` — 7 had a full trail through carrier handoff with only the final timestamp missing; documented rather than imputed.
- **Verified merge integrity**: joining `order_items` → `orders` → `products` → `customers` → `payments` (aggregated) → `reviews` (deduplicated) preserved the exact original row count (112,650), confirming no accidental row duplication from one-to-many relationships.

## Key Findings

### 1. Delivery is fast, and Olist's estimates are conservative
Average delivery time is **~12 days** (median 10), but the *estimated* delivery date is padded well beyond that — **93.4% of orders arrive on time or early**, with an average buffer of 12 days ahead of the estimate.

### 2. Late delivery rate varies more than 6x by state
| State | Late Rate |
|---|---|
| AL (Alagoas) | 20.84% |
| MA (Maranhão) | 18.00% |
| SP (São Paulo) | 4.40% |
| AM (Amazonas) | 3.07% |

Northeastern states consistently show the highest delay rates, while São Paulo — despite handling ~42% of all order volume — has one of the lowest, suggesting delivery reliability tracks proximity to major logistics hubs more than order volume.

### 3. Delivery speed and delivery reliability are different things
Furniture categories (office, living room, bedroom) take the longest to ship (13–20 days vs. a 12-day average) — expected, given bulkier logistics. But they aren't the least *reliable*: categories like `audio` and `artigos_de_natal` (holiday items) show disproportionately high late rates (10–11.6%) despite average shipping speeds, pointing to category-specific fulfillment issues rather than a simple size effect.

### 4. Late delivery is strongly linked to customer dissatisfaction
This was the strongest finding in the analysis:

| Delivery Timing | Avg. Review Score |
|---|---|
| Early (7+ days) | 4.23 |
| On-time (0–7 days early) | 4.11 |
| Late (0–7 days) | 2.68 |
| Very Late (7+ days) | 1.70 |

![Review score by delivery timing](review_score_by_delivery_timing.png)

The drop is a **threshold effect**, not a gradual slide — crossing from on-time to even slightly late causes a steep score drop (4.11 → 2.68), and scores continue falling the later an order gets. This suggests customers react less to exact delivery speed and more to whether Olist met its own promised date at all.

## Setup

1. Clone this repo
2. Copy `.env.example` to `.env` and fill in your own MySQL credentials
3. Run `schema.sql` against your MySQL instance to recreate the table structure:
   ```bash
   mysql -u root -p olist < schema.sql
   ```
4. Open `olist.ipynb` and run through the notebook

## Repo Structure

```
olist-ecommerce-pipeline/
├── schema.sql          # Table structures, keys, and constraints
├── olist.ipynb          # Cleaning and analysis notebook
├── review_score_by_delivery_timing.png
├── .env.example         # Template for required environment variables
├── .gitignore
└── README.md
```

## Status

Core pipeline complete: schema design, data cleaning, and delivery-performance analysis. Revenue/sales-trend analysis is a possible future extension.
