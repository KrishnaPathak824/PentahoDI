# Pentaho Data Warehouse Project

The goal of this project is to build a Sales Data Warehouse with Incremental Loading using actual transaction data. The project involves designing and implementing a data pipeline to extract, transform, and load (ETL) sales data into a data warehouse using Pentaho Data Integration (PDI).

### Data Used:

Sales Data: 30 transactions from May 4-6, 2021
[Data Link](https://drive.google.com/file/d/16kWH3qkPTa0yBj1NQTG_8nv_bmW5pcTz/view)

### Technologies Used:

1. Database: PostgreSQL

2. ETL Tool: Pentaho Data Integration (PDI)

3. Data Warehouse Structure: 3-tier architecture (Staging, Core, and Metadata)

## Database Setup (PostgreSQL)

- Download and install PostgreSQL from the official website: https://www.postgresql.org/download/

- Create a database named `sales_warehouse` in the pgAdmin.

![alt text](materials/image-1.png)

- Connect to the `sales_warehouse` and create the necessary schemas: `public`, `staging`, `core`, and `meta`.

![alt text](materials/image.png)

![alt text](materials/image-2.png)

![alt text](materials/image-3.png)

Then, create the source tables in the `public` schema.

```sql
-- Create source sales table in PUBLIC schema
CREATE TABLE public.sales (
    transaction_id INTEGER PRIMARY KEY,
    transactional_date TIMESTAMP NOT NULL,
    product_id VARCHAR(50),
    customer_id INTEGER,
    payment VARCHAR(50),
    credit_card BIGINT,
    loyalty_card VARCHAR(50),
    cost NUMERIC(10, 2),
    quantity INTEGER,
    price NUMERIC(10, 2)
);

-- Create product dimension in SOURCE
CREATE TABLE public.product (
    product_id VARCHAR(50) PRIMARY KEY,
    product_name VARCHAR(100),
    category VARCHAR(50)
);

-- Create payment type in SOURCE
CREATE TABLE public.payment_type (
    payment VARCHAR(50) PRIMARY KEY,
    loyalty_card VARCHAR(50)
);
```

![alt text](materials/image-7.png)

Next, the data was imported using a CSV file into the `public.sales` table.

![alt text](materials/image-6.png)

Then the count of records in the source table was verified.

```sql
SELECT COUNT(*) FROM public.sales;
```

![alt text](materials/image-8.png)

Then, create the necessary tables in the `staging`, `core`, and `meta` schemas.

```sql
-- Staging area
CREATE TABLE staging.sales (
    transaction_id INTEGER PRIMARY KEY,
    transactional_date TIMESTAMP NOT NULL,
    product_id VARCHAR(50),
    customer_id INTEGER,
    payment VARCHAR(50),
    credit_card VARCHAR(50),
    loyalty_card VARCHAR(1),
    cost NUMERIC(10, 2),
    quantity INTEGER,
    price NUMERIC(10, 2)
);
```

![alt text](materials/image-9.png)

```sql
-- Dimension table for payment methods
CREATE TABLE core.dim_payment (
    payment_pk SERIAL PRIMARY KEY,
    payment VARCHAR(50) UNIQUE,
    loyalty_eligible VARCHAR(1)
);

-- Dimension table for products
CREATE TABLE core.dim_product (
    product_pk SERIAL PRIMARY KEY,
    product_id VARCHAR(50) UNIQUE,
    product_name VARCHAR(100)
);

-- Fact table - main warehouse table
CREATE TABLE core.sales (
    transaction_id INTEGER PRIMARY KEY,
    transactional_date TIMESTAMP NOT NULL,
    transactional_date_fk VARCHAR(8),
    product_id VARCHAR(50),
    product_fk INTEGER,
    customer_id INTEGER,
    payment_fk INTEGER,
    credit_card VARCHAR(50),
    loyalty_card VARCHAR(1),
    cost NUMERIC(10, 2),
    quantity INTEGER,
    price NUMERIC(10, 2),
    total_cost NUMERIC(10, 2),
    total_price NUMERIC(10, 2),
    profit NUMERIC(10, 2)
);

-- Create indexes for performance
CREATE INDEX idx_core_sales_date ON core.sales(transactional_date);
CREATE INDEX idx_core_sales_product_fk ON core.sales(product_fk);
CREATE INDEX idx_core_sales_payment_fk ON core.sales(payment_fk);
```

![alt text](materials/image-10.png)

```sql
-- Metadata table to track load history
CREATE TABLE meta.load_history (
    load_id SERIAL PRIMARY KEY,
    table_name VARCHAR(100),
    last_load_date TIMESTAMP,
    current_load_date TIMESTAMP,
    records_loaded INTEGER,
    load_status VARCHAR(20),
    error_message TEXT,
    load_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

![alt text](materials/image-11.png)

## Pentaho Installation and Setup

- Download and install Pentaho Data Integration (PDI).

- Extract the ZIP file and run `spoon.bat`.

Create a new database connection to PostgreSQL in PDI for my database `sales_warehouse`.

![alt text](materials/image-12.png)

![alt text](materials/image-13.png)

![alt text](materials/image-14.png)

![alt text](materials/image-15.png)

**_1. Created transformations for Incremental Load process._**

First, create a transformation to get the last load date from the `core.sales` table.

Create a table input step with the following query to get the last load date.

```sql
SELECT COALESCE(MAX(transactional_date), '2000-01-01'::TIMESTAMP) as LastLoadDate
FROM core.sales
```

![alt text](materials/image-18.png)

- Next, set the variable `v_LastLoadDate` to store the last load date.

![alt text](materials/image-19.png)

**_2. Create the build transformation to extract only new records from the source table based on the last load date._**

First, create a table input step with the following query to extract new records.

```sql
SELECT
    transaction_id,
    transactional_date,
    product_id,
    customer_id,
    payment,
    credit_card,
    loyalty_card,
    cost,
    quantity,
    price
FROM public.sales
WHERE transactional_date > '${v_LastLoadDate}'
ORDER BY transaction_id
```

Then add select values to fix the data types.

![alt text](materials/image-20.png)

Then create a insert/update step to load the new records into the `staging.sales` table.

![alt text](materials/image-21.png)

It is tested and the transformation works fine.

![alt text](materials/image-22.png)

3. **Create the main ETL transformation to load data from staging to core schema.**

```sql
SELECT * FROM staging.sales
```

![alt text](materials/image-23.png)

```sql
UPDATE public.sales
SET payment = 'Unknown'
WHERE payment IS NULL;
```

![alt text](materials/image-24.png)

![alt text](materials/image-25.png)

![alt text](materials/image-26.png)

![alt text](materials/image-27.png)

![alt text](materials/image-28.png)

![alt text](materials/image-29.png)

![alt text](materials/image-30.png)

![alt text](materials/image-31.png)

![alt text](materials/image-32.png)

![alt text](materials/image-33.png)

---

![alt text](materials/image-34.png)

![alt text](materials/image-35.png)

![alt text](materials/image-36.png)

![alt text](materials/image-37.png)

```sql
INSERT INTO public.sales (transaction_id, transactional_date, product_id, customer_id, payment, credit_card, loyalty_card, cost, quantity, price) VALUES
(4414, '2023-05-07 10:00:00', 'P0494', 11, 'visa', '4041593010498829', 'F', 25.00, 2, 30.00),
(4415, '2023-05-07 11:30:00', 'P0221', 12, 'mastercard', '5108753677552345', 'T', 15.50, 1, 20.00),
(4416, '2023-05-07 14:00:00', 'P0625', 13, 'visa', '4041594885335898', 'F', 45.00, 3, 50.00),
(4417, '2023-05-08 09:00:00', 'P0431', 14, 'americanexpress', '374288563442549', 'F', 8.00, 1, 12.00),
(4418, '2023-05-08 16:00:00', 'P0058', 15, 'mastercard', '5108752372298261', 'T', 30.00, 2, 35.00);
```

![alt text](materials/image-38.png)

```sql
SELECT COUNT(*) FROM public.sales;
```

![alt text](materials/image-39.png)

![alt text](materials/image-40.png)

```sql
SELECT * FROM core.sales ORDER BY transaction_id DESC LIMIT 5;
```

![alt text](materials/image-41.png)

```sql
SELECT
    load_id,
    records_loaded,
    load_status,
    load_timestamp
FROM meta.load_history
ORDER BY load_id DESC;
```

![alt text](materials/image-42.png)

```sql
SELECT MAX(transactional_date) as last_load_date FROM core.sales;
```

![alt text](materials/image-43.png)

```sql
INSERT INTO public.sales VALUES
(4419, '2023-10-06 10:00:00', 'P0123', 20, 'esewa', '4041593010498829', 'F', 10.00, 1, 15.00);
```

![alt text](materials/image-44.png)

```sql
SELECT * FROM core.dim_payment ORDER BY payment_pk;
```

![alt text](materials/image-45.png)
