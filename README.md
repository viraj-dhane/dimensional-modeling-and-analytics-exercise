# Dimensional Modeling and Analytics Exercise
SQL, Snowflake Data Warehouse, AWS S3

---

## Overview

This exercise involes

  - Designing a dimensional model for e-commece platform (e.g. Instacart)
  - i.e. Organize data into facts and dimensions
  - Integrate Snowflake and AWS S3 buckets
  - Develop SQL queries to
      - Create tables in Snowflake data warehouse
      - Insert/ingest data from AWS S3 into tables
      - Create Dimension and Fact Tables

## Dimensional Modeling / Star Schema

![star-schema](https://github.com/user-attachments/assets/bda89a9e-add5-40db-af80-1cdd682f90c0)

## SQL Queries

Creating Stage and File Format

      CREATE STAGE my_stage
      URL = 's3://data-engg-snowflake-data-warehousing/instacart/'
      CREDENTIALS = (AWS_KEY_ID = 'XYZABC' AWS_SECRET_KEY = 'XYaZb0C1');

      CREATE OR REPLACE FILE FORMAT csv_file_format
      TYPE = 'CSV'
      FIELD_DELIMITER = ','
      SKIP_HEADER = 1
      FIELD_OPTIONALLY_ENCLOSED_BY = '"'

Creating Tables and Inserting Data

      -- Aisles Table
      CREATE TABLE aisles (
        aisle_id INTEGER PRIMARY KEY,
        aisle VARCHAR
      );
      COPY INTO aisles (aisle_id, aisle)
      FROM @my_stage/aisles.csv
      FILE_FORMAT = (FORMAT_NAME = 'csv_file_format');

      -- Department Table
      CREATE TABLE departments (
        department_id INTEGER PRIMARY KEY,
        department VARCHAR
      );
      COPY INTO departments (department_id, department)
      FROM @my_stage/departments.csv
      FILE_FORMAT = (FORMAT_NAME = 'csv_file_format');

    -- Products Table
    CREATE OR REPLACE TABLE products (
        product_id INTEGER PRIMARY KEY,
        product_name VARCHAR,
        aisle_id INTEGER,
        department_id INTEGER
    );
    COPY INTO products (product_id, product_name, aisle_id, department_id)
    FROM @my_stage/products.csv
    FILE_FORMAT = (FORMAT_NAME = 'csv_file_format');

    -- Orders Table
    CREATE OR REPLACE TABLE orders (
        order_id INTEGER PRIMARY KEY,
        user_id INTEGER,
        eval_set STRING,
        order_number INTEGER,
        order_dow INTEGER,
        order_hour_of_day INTEGER,
        days_since_prior_order INTEGER
    );
    COPY INTO orders (order_id, user_id, eval_set, order_number, order_dow, order_hour_of_day, days_since_prior_order)
    FROM @my_stage/orders.csv
    FILE_FORMAT = (FORMAT_NAME = 'csv_file_format');

    -- Order Products Table
    CREATE OR REPLACE TABLE order_products (
        order_id INTEGER,
        product_id INTEGER,
        add_to_cart_order INTEGER,
        reordered INTEGER,
        PRIMARY KEY (order_id, product_id)
    );
    COPY INTO order_products (order_id, product_id, add_to_cart_order, reordered)
    FROM @my_stage/order_products.csv
    FILE_FORMAT = (FORMAT_NAME = 'csv_file_format');

Creating Dimension Tables

    CREATE OR REPLACE TABLE dim_users AS (
      SELECT
        user_id
      FROM
        orders
    );

    CREATE OR REPLACE TABLE dim_products AS (
      SELECT
        product_id,
        product_name
      FROM
        products
    );

    CREATE OR REPLACE TABLE dim_aisles AS (
      SELECT
        aisle_id,
        aisle
      FROM
        aisles
    );

    CREATE OR REPLACE TABLE dim_departments AS (
      SELECT
        department_id,
        department
      FROM
        departments
    );

    CREATE OR REPLACE TABLE dim_orders AS (
      SELECT
        order_id,
        order_number,
        order_dow,
        order_hour_of_day,
        days_since_prior_order
      FROM
        orders
    );

Creating Fact Table

    CREATE TABLE fact_order_products AS (
      SELECT
        op.order_id,
        op.product_id,
        o.user_id,
        p.department_id,
        p.aisle_id,
        op.add_to_cart_order,
        op.reordered
      FROM
        order_products op
      JOIN
        orders o ON op.order_id = o.order_id
      JOIN
        products p ON op.product_id = p.product_id
    );

Calculate the total number of products ordered per department:

    SELECT
      d.department,
      COUNT(*) AS total_products_ordered
    FROM
      fact_order_products fop
    JOIN
      dim_departments d ON fop.department_id = d.department_id
    GROUP BY
      d.department;

Find the top 5 aisles with the highest number of reordered products:

    SELECT
      a.aisle,
      COUNT(*) AS total_reordered
    FROM
      fact_order_products fop
    JOIN
      dim_aisles a ON fop.aisle_id = a.aisle_id
    WHERE
      fop.reordered = TRUE
    GROUP BY
      a.aisle
    ORDER BY
      total_reordered DESC
    LIMIT 5;

Calculate the average number of products added to the cart per order by day of the week:

    SELECT
      o.order_dow,
      AVG(fop.add_to_cart_order) AS avg_products_per_order
    FROM
      fact_order_products fop
    JOIN
      dim_orders o ON fop.order_id = o.order_id
    GROUP BY
      o.order_dow;

Identify the top 10 users with the highest number of unique products ordered:

    SELECT
      u.user_id,
      COUNT(DISTINCT fop.product_id) AS unique_products_ordered
    FROM
      fact_order_products fop
    JOIN
      dim_users u ON fop.user_id = u.user_id
    GROUP BY
      u.user_id
    ORDER BY
      unique_products_ordered DESC
    LIMIT 10;
