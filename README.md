# Olist E-commerce Data Warehouse Project

## Project Overview

This project aims to build a robust and scalable data warehouse for the Olist E-commerce dataset using Google Cloud Platform (GCP) services and dbt (Data Build Tool). The primary goal is to transform raw transactional data into an analytical-ready Star Schema, enabling efficient business intelligence and reporting for key e-commerce metrics.

## Project Goals

*   **Ingest Raw Data:** Efficiently load the Olist raw datasets into Google Cloud Storage (GCS) and then into BigQuery.
*   **Data Cleaning & Transformation:** Apply data cleaning rules and business logic to prepare data for analytical use.
*   **Star Schema Design:** Implement a Star Schema data model optimized for reporting and analytical queries.
*   **Automated ETL/ELT:** Utilize dbt for orchestrating data transformations and ensuring data quality.
*   **Analytical Foundation:** Provide a clean, well-structured dataset for business intelligence tools and data scientists.

## Data Warehouse Architecture: Star Schema

Our data warehouse is built upon a **Star Schema** architecture, chosen for its simplicity, ease of querying, and optimized performance for analytical workloads. This design separates descriptive data (dimensions) from quantitative data (facts), making it intuitive for business users to understand and analyze.

### **1. Core Fact Table: `FactOrders`**

The central table in our schema, `FactOrders`, captures the measurable events and key performance indicators related to each order item.

*   **Grain:** Each row represents a unique item within an order (`order_item_id`).
*   **Key Metrics:**
    *   `price`: The price of the individual item.
    *   `freight_value`: The freight cost associated with the item.
    *   `quantity`: The quantity of the item (typically 1 for individual order items).
    *   `payment_value`: The total value of the payment for the associated order.
    *   `review_score`: The customer's review score for the order.
    *   `order_status`: The current status of the order (e.g., 'delivered', 'shipped').
*   **Foreign Keys (FKs):** Contains foreign keys linking to all relevant dimension tables, allowing us to slice and dice measures by various attributes.

### **2. Dimension Tables (Contextual Data)**

Surrounding the `FactOrders` table are our dimension tables, which provide the descriptive attributes (the "who, what, when, where") for each sales event.

*   **`DimCustomer`**:
    *   **Purpose:** Stores detailed information about each customer.
    *   **Primary Key:** `customer_id`
    *   **Attributes:** `customer_unique_id`, `customer_city`, `customer_state`, and a foreign key to `DimGeolocation` for the customer's location.
    *   **Source:** `olist_customers_dataset.csv`
*   **`DimProduct`**:
    *   **Purpose:** Contains descriptive attributes for each product.
    *   **Primary Key:** `product_id`
    *   **Attributes:** `product_category_name` (standardized and translated), `product_name_lenght`, `product_description_lenght`, `product_photos_qty`, `product_weight_g`, `product_length_cm`, `product_height_cm`, `product_width_cm`.
    *   **Source:** `olist_products_dataset.csv`
*   **`DimSeller`**:
    *   **Purpose:** Holds information about the sellers.
    *   **Primary Key:** `seller_id`
    *   **Attributes:** `seller_city`, `seller_state`, and a foreign key to `DimGeolocation` for the seller's location.
    *   **Source:** `olist_sellers_dataset.csv`
*   **`DimDate`**:
    *   **Purpose:** A comprehensive table containing various time-based attributes.
    *   **Primary Key:** `date_id` (an integer in YYYYMMDD format).
    *   **Attributes:** `full_date`, `day`, `month`, `year`, `quarter`, `day_of_week`, `month_name`, `is_weekend`, `week_of_year`.
    *   **Linked to `FactOrders` multiple times:** For `order_purchase_date`, `order_approved_at_date`, `order_delivered_customer_date`, and `order_estimated_delivery_date`, allowing granular analysis by different points in the order lifecycle.
    *   **Source:** Generated dynamically (e.g., using `GENERATE_DATE_ARRAY` in BigQuery).
*   **`DimGeolocation`**:
    *   **Purpose:** Provides geographic coordinates and city/state information for zip code prefixes.
    *   **Primary Key:** `geolocation_zip_code_prefix`
    *   **Attributes:** `geolocation_lat`, `geolocation_lng`, `geolocation_city`, `geolocation_state`.
    *   **Linked to:** `DimCustomer` and `DimSeller`.
    *   **Source:** `olist_geolocation_dataset.csv`

### **3. Entity-Relationship Diagram (ERD)**

Below is the conceptual ERD illustrating the relationships between our Fact and Dimension tables in the Star Schema.

```mermaid
erDiagram
    %% Dimensions Tables
    DimCustomer {
        VARCHAR customer_id PK "Unique identifier for each customer"
        VARCHAR customer_unique_id "Anonymized identifier for unique customer"
        VARCHAR customer_zip_code_prefix FK "Links to DimGeolocation"
        VARCHAR customer_city
        VARCHAR customer_state
    }
    
    DimProduct {
        VARCHAR product_id PK "Unique identifier for each product"
        VARCHAR product_category_name "Category of the product (e.g., 'electronics')"
        INT product_name_lenght
        INT product_description_lenght
        INT product_photos_qty
        INT product_weight_g
        INT product_length_cm
        INT product_height_cm
        INT product_width_cm
    }
    
    DimSeller {
        VARCHAR seller_id PK "Unique identifier for each seller"
        VARCHAR seller_zip_code_prefix FK "Links to DimGeolocation"
        VARCHAR seller_city
        VARCHAR seller_state
    }
    
    DimDate {
        INT date_id PK "Surrogate key for date (YYYYMMDD)"
        DATE full_date "Full date"
        INT day "Day of the month"
        INT month "Month number"
        INT year "Year"
        INT quarter "Quarter of the year"
        VARCHAR day_of_week "Name of the day (e.g., 'Monday')"
        VARCHAR month_name "Name of the month"
        BOOLEAN is_weekend "Flag for weekend"
        INT week_of_year "Week number"
    }
    
    DimGeolocation {
        VARCHAR geolocation_zip_code_prefix PK "Zip code prefix"
        DECIMAL geolocation_lat "Latitude"
        DECIMAL geolocation_lng "Longitude"
        VARCHAR geolocation_city "City"
        VARCHAR geolocation_state "State"
    }
    
    %% Fact Table
    FactOrders {
        VARCHAR order_item_id PK "Unique identifier for each item within an order"
        VARCHAR order_id PK "Unique identifier for the order (part of composite PK)"
        VARCHAR customer_id FK "References DimCustomer"
        VARCHAR product_id FK "References DimProduct"
        VARCHAR seller_id FK "References DimSeller"
        INT order_purchase_date_id FK "References DimDate for purchase date"
        INT order_approved_at_date_id FK "References DimDate for approval date"
        INT order_delivered_customer_date_id FK "References DimDate for customer delivery date"
        INT order_estimated_delivery_date_id FK "References DimDate for estimated delivery date"
        DECIMAL price "Price of the item"
        DECIMAL freight_value "Freight value for the item"
        INT quantity "Quantity of the item (usually 1 per order_item_id)"
        VARCHAR order_status "Current status of the order (e.g., 'delivered')"
        DECIMAL payment_value "Total payment value for the order"
        VARCHAR payment_type "Method of payment (e.g., 'credit_card')"
        INT payment_installments "Number of payment installments"
        INT review_score "Customer review score for the order"
    }
    
    %% Relationships
    FactOrders ||--o{ DimCustomer : "placed by"
    FactOrders ||--o{ DimProduct : "includes"
    FactOrders ||--o{ DimSeller : "fulfilled by"
    FactOrders ||--o{ DimDate : "purchased on"
    FactOrders ||--o{ DimDate : "approved on"
    FactOrders ||--o{ DimDate : "delivered on"
    FactOrders ||--o{ DimDate : "estimated delivery on"
    
    DimCustomer }|--o{ DimGeolocation : "located at"
    DimSeller }|--o{ DimGeolocation : "located at"
