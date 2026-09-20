# Olist E-Commerce Data Pipeline

An end-to-end data pipeline built on the Olist Brazilian e-commerce dataset — covering database design, data quality enforcement, cleaning, and analysis in MySQL and Python.

## Project Overview

This project takes the raw Olist dataset (orders, customers, products, order items, reviews, geolocation, etc.) and builds it into a properly structured relational database, then cleans and analyzes it using Python (pandas, SQLAlchemy).

**Pipeline stages:** Data collection → MySQL storage & schema design → Data cleaning → Analysis → Visualization

## Tech Stack

- **MySQL** — relational storage, schema design, constraint enforcement
- **Python** — pandas, SQLAlchemy
- **Jupyter Notebook** — analysis and cleaning workflow

## Database Design Decisions

A few deliberate schema decisions were made while building this out, rather than just accepting the raw dataset's structure as-is:

### 1. Geolocation table — index instead of primary key
The geolocation table has duplicate rows for its zip-code-prefix column, so it can't serve as a primary key. It was set up as an **index** instead, kept for fast lookups without forcing a false uniqueness constraint.

### 2. Composite key on `order_items`
Neither `order_id` nor `order_item_id` is unique on its own — an order can contain multiple items, each numbered starting from 1 within that order. The two columns together form the natural **composite primary key**.

### 3. Orphaned rows surfaced by foreign key enforcement
While adding a foreign key between `orders` and `customers`, constraint validation surfaced **13 orphaned records** (0.01% of orders) referencing customer IDs that didn't exist in the customers table. After confirming no recoverable customer data existed for these rows, they were removed.

This is a good example of why enforcing referential integrity at the database level matters: these 13 rows would have gone completely undetected in a pandas-only cleaning pipeline, since pandas has no concept of a foreign key constraint to violate.

### 4. Customers–geolocation relationship — evaluated, not enforced
A foreign key between `customers` and `geolocation` was considered but ultimately **not implemented**. The geolocation table has incomplete zip code coverage and no natural unique key, so enforcing this constraint would have meant dropping otherwise-valid customer records just to satisfy a supplementary reference table. An index was used instead, and the coverage gap is documented here rather than silently patched over.

## Data Cleaning Notes

- Checked all ID-style columns (32-character hex strings) for null values, empty strings, and whitespace-only formatting issues separately — confirmed the only real issue was genuine nulls, not disguised blanks.
- Verified foreign key relationships hold in pandas as well as MySQL, as a belt-and-suspenders check.

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
├── olist.ipynb          # Main analysis and cleaning notebook
├── .env.example         # Template for required environment variables
├── .gitignore
└── README.md
```

## Status

🚧 In progress — cleaning and analysis stages underway, visualization layer to follow.