# E-Commerce Database Schema & Queries

## Overview

This project provides a relational database schema for an **E-Commerce System**, along with SQL queries to extract useful reports.

## Database Schema

The database consists of five main tables:

- **Category**: Stores product categories.
- **Product**: Stores product details and links to categories.
- **Customer**: Stores customer details.
- **Order**: Stores order details, including total amounts.
- **OrderDetails**: Stores details of each order, linking products to orders.

### Relationships Between Entities

| **Relationship**              | **Type**       | **Description**                                                                 |
|-------------------------------|----------------|---------------------------------------------------------------------------------|
| **Category → Product**     | One-to-Many    | One category can have many products.                                            |
| **Customer → Order**        | One-to-Many    | One customer can place many orders.                                             |
| **Order → OrderDetails**     | One-to-Many    | One order can have many order details.                                          |
| **Product → OrderDetails**   | One-to-Many    | One product can appear in many order details.                                   |

### ERD Diagram

![ERD Diagram](./ecommerce-erd-diagram.png) 

## Database Schema Script

```sql
-- Create the database
CREATE DATABASE e-commerce;

-- Create the Category table
CREATE TABLE Category (
    category_id SERIAL PRIMARY KEY,
    category_name VARCHAR(100) UNIQUE
);

-- Create the Product table
CREATE TABLE product (
    product_id SERIAL PRIMARY KEY,
    category_id INT,
    name VARCHAR(250) NOT NULL,
    description TEXT,
    price DECIMAL(6, 2) NOT NULL CHECK(price > 0),
    stock_quantity INT NOT NULL DEFAULT 0,
    FOREIGN KEY (category_id) REFERENCES category(category_id)
);

-- Create the Customer table
CREATE TABLE customer (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(30) NOT NULL,
    last_name VARCHAR(30) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(225) NOT NULL
);

-- Create the Order table
CREATE TABLE "order" (
    order_id SERIAL PRIMARY KEY,
    customer_id INT,
    order_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customer (customer_id)
);

-- Create the OrderDetails table
CREATE TABLE order_details (
    order_detail_id SERIAL PRIMARY KEY,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES "order" (order_id),
    FOREIGN KEY (product_id) REFERENCES product (product_id)
);
```

## Seeded Data Overview

| Table Name         | Number of Records Seeded |
| ------------------ | ------------------------ |
| category           | 100                      |
| product            | 100,000                  |
| customer           | 1,000,000                |
| order              | 5,000,000                |
| order_details      | 10,000,000                |

---

## SQL Queries Before & After Optimization 

### 1. Daily Revenue Report for a specific date

**Query:**

```sql
SELECT DATE(order_date) AS OrderDate, SUM(total_amount) AS DailyRevenue
FROM "order"
WHERE DATE(order_date) = '2025-06-12'
GROUP BY DATE(order_date);
```

**Execution Time Before Optimization:** 363.134 ms

**Execution Time After Optimization:** 8.825 ms

**Optimization Techniques:**

- Rewrite the query using a **Range Condition** for the **order date** in the `WHERE` clause:

```sql
SELECT date_trunc('day', order_date) AS OrderDate, SUM(total_amount) AS DailyRevenue
FROM "order"
WHERE order_date >= '2025-06-12 00:00:00' AND order_date < '2025-06-13 00:00:00'
GROUP BY OrderDate;
```

- Create an **Covering Index** in **(order date)** column on orders table:

```sql
CREATE INDEX idx_order_order_date_total_amount ON "order"(order_date,total_amount);
```

### 2. Monthly Top-Selling Products Report

**Query:**

```sql
SELECT 
    TO_CHAR(O.order_date, 'YYYY-MM') AS Month,
    P.name AS ProductName,
    SUM(OD.quantity) AS TotalQuantity
FROM orderdetails OD
JOIN orders O ON O.order_id = OD.order_id
JOIN products P ON P.product_id = OD.product_id
WHERE TO_CHAR(O.order_date, 'YYYY-MM') = '2025-01'
GROUP BY TO_CHAR(O.order_date, 'YYYY-MM'), P.name, P.product_id
ORDER BY TotalQuantity DESC;
```

**Execution Time Before Optimization:** 16894.256 ms

**Execution Time After Optimization:** 546.125 ms

**Optimization Techniques:**

- Apply Denormalization to Reduce Joins by Combining Data:
  
    **Steps to Apply Denormalization:**
  
    **Step 1**: Create a **Denormalized Table**:
      
    ```sql
    CREATE TABLE denormalized_orders_products(
    	order_id INT , 
    	product_id INT ,
    	product_name VARCHAR(250) NOT NULL,
    	order_date TIMESTAMP NOT NULL,
    	quantity INT NOT NULL CHECK (quantity > 0)
    );
    ```
    **Step 2**: Filling the Denormalized Table:
    
    ```sql
    INSERT INTO denormalized_orders_products (order_id, product_id, product_name, order_date, quantity)
    SELECT O.order_id, OD.product_id, P.name AS product_name, O.order_date, OD.quantity
    FROM order_details OD
    JOIN "order" O ON O.order_id = OD.order_id
    JOIN product P ON P.product_id = OD.product_id;
	```

    **Step 3**: Query the Denormalized Table:
    ```sql
    SELECT 
    TO_CHAR(order_date, 'YYYY-MM') AS Month,
    product_name, SUM(quantity) AS TotalQuantity
    FROM denormalized_orders_products
    WHERE order_date >= '2025-01-01 00:00:00' AND order_date < '2025-02-01 00:00:00'
    GROUP BY TO_CHAR(order_date, 'YYYY-MM'), product_name
    ORDER BY TotalQuantity DESC
    limit 100;
    ```

- Create a **Covering Index** that includes **(order_date,product_name,quantity)**:
    
```sql
CREATE INDEX idx_orderdate_productname_productquantity ON denormalized_orders_products(order_date,product_name,quantity) 
```

### 3. Customers with Orders Totaling More Than \$500 in the Past Month

**Query:**

```sql
SELECT 
    C.first_name || ' ' || C.last_name AS CustomerName,
    SUM(O.total_amount) AS TotalAmount
FROM "order" O
JOIN customer C ON O.customer_id = C.customer_id
WHERE O.order_date >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month' AND O.order_date <  DATE_TRUNC('month', CURRENT_DATE)
GROUP BY C.customer_id, C.first_name, C.last_name
HAVING SUM(O.total_amount) > 500;
```

**Execution Time Before Optimization:** 4896.671 ms

**Execution Time After Optimization:** 17.112 ms

**Optimization Techniques:**

- Rewrite the query to use a Materialized CTA:
```sql
WITH qualifying_orders AS MATERIALIZED (
    SELECT
        customer_id,
        SUM(total_amount) AS total_spent
    FROM "order"
    WHERE order_date >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 month')
          AND order_date < DATE_TRUNC('month', CURRENT_DATE)
    GROUP BY customer_id
    HAVING SUM(total_amount) > 500
)

SELECT
    C.first_name || ' ' || C.last_name AS CustomerName,
    O.total_spent AS TotalAmount
FROM customer C
INNER JOIN qualifying_orders O ON C.customer_id = O.customer_id;
```

- Create a **Covering Index** for **(customer_id, order_date,total_amount)** columns in orders table :
```sql
CREATE INDEX idx_order_date_customer_amount ON "order"(order_date, customer_id, total_amount);
```

### 4. Total Number of Products in Each Category

**Query:**

```sql
SELECT C.category_id, C.category_name, COUNT(P.product_id) AS total_products
FROM category C LEFT JOIN product P ON C.category_id=P.category_id
GROUP BY C.category_id , C.category_name
ORDER BY C.category_id;
```

**Execution Time Before Optimization:** 57.891 ms

**Execution Time After Optimization:** 13.402 ms

**Optimization Techniques:**

- Rewrite the query to use a subquery that filters products first:
```sql
  SELECT c.category_id, c.category_name, 
       COALESCE(p.product_count, 0) AS total_products
FROM category c
LEFT JOIN (
    SELECT category_id, COUNT(product_id) AS product_count
    FROM product
    GROUP BY category_id
) p ON c.category_id = p.category_id
```

- Create an **Index** for **(category_id)** column in the products table:
```sql
CREATE INDEX idx_product_category_id ON product(category_id);
```

### 5. Top Customers by Total Spending

**Query:**

```sql
SELECT C.customer_id, C.first_name || ' ' || C.last_name AS full_name, SUM(O.total_amount) AS total_spending
FROM customer C JOIN "order" O ON C.customer_id=O.customer_id
GROUP By C.customer_id
ORDER BY total_spending DESC
LIMIT 10;
```

**Execution Time Before Optimization:** 5003.110 ms

**Execution Time After Optimization:** 0.826 ms

**Optimization Techniques:**

- Apply Materialized View:
```sql
CREATE MATERIALIZED VIEW mv_customer_spending AS
SELECT 
    C.customer_id,
    C.first_name || ' ' || C.last_name AS full_name,
    SUM(O.total_amount) AS total_spending
FROM customer C
JOIN "order" O ON C.customer_id = O.customer_id
GROUP BY C.customer_id, C.first_name, C.last_name;

SELECT customer_id, full_name, total_spending
FROM mv_customer_spending
ORDER BY total_spending DESC
LIMIT 10;
```

- Create **Index** for **(total_spending)** columns in the mv_customer_spending table:

```sql
 CREATE INDEX idx_mv_spending ON mv_customer_spending(total_spending DESC);
```

### 6. Most Recent Orders with Customer Information (1000 Orders)

**Query:**

```sql
SELECT o.order_id, o.order_date ,o.total_amount, c.customer_id, c.first_name ||' '|| c.last_name AS full_name, c.email
FROM "order" o JOIN customer c ON o.customer_id = c.customer_id
ORDER BY o.order_date DESC 
LIMIT 1000;
```

**Execution Time Before Optimization:** 1014.651 ms

**Execution Time After Optimization:** 0.981 ms

**Optimization Techniques:**

- Rewrite the query to use a subquery that filters orders first:
  
```sql
SELECT o.order_id, o.order_date ,o.total_amount, c.customer_id, c.first_name ||' '|| c.last_name AS full_name, c.email 
from customer c 
join (
	select order_id , order_date , total_amount, customer_id
	from "order"
	order by order_date desc
	limit 1000
) o on c.customer_id = o.customer_id

```

- Create an **Index** for **(order_date)** column in the order table:

```sql
CREATE INDEX idx_order_order_date ON "order"(order_date);
```

### 7. List of Products with Low Stock (Less than 10)

**Query:**

```sql
SELECT product_id, name , stock_quantity FROM product
WHERE stock_quantity < 10 ;
```

**Execution Time Before Optimization:** 19.709 ms

**Execution Time After Optimization:** 3.975 ms

**Optimization Techniques:**

- Create an **Index** for **(stock_quantity)** column in the products table:

```sql
CREATE INDEX idx_product_stock_quantity ON product(stock_quantity);
```

### 8. Revenue Generated by Each Product Category

**Query:**

```sql
SELECT c.category_id, c.category_name, SUM(od.quantity * od.unit_price) AS total_revenue
FROM category c
JOIN product p ON c.category_id = p.category_id 
JOIN order_details od ON od.product_id=p.product_id  
GROUP BY c.category_id;
```

**Execution Time Before Optimization:** 4898.515 ms

**Execution Time After Optimization:** 0.338 ms

**Optimization Techniques:**

- Apply Materialized View:
```sql
CREATE MATERIALIZED VIEW mv_category_revenue AS
SELECT 
    c.category_id,
    c.category_name,
    COALESCE(SUM(od.quantity * od.unit_price), 0) AS total_revenue,
    COUNT(DISTINCT od.order_id) AS order_count,
    COUNT(DISTINCT p.product_id) AS product_count
FROM category c
LEFT JOIN product p ON c.category_id = p.category_id
LEFT JOIN order_details od ON od.product_id = p.product_id
GROUP BY c.category_id, c.category_name;

SELECT * FROM mv_category_revenue 
ORDER BY total_revenue DESC;
```
---

## Final Optimization Summary

| Query Description                                               | Time Before Optimization | Optimization Techniques                                                                                                                                     | Time After Optimization |
| --------------------------------------------------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------- |
| Daily Revenue Report for a Specific Date                        | 363.134 ms               | • Replaced `DATE(order_date)` filter with range condition<br>• Added composite index on `(order_date, total_amount)`                                        | 8.825 ms                |
| Monthly Top-Selling Products Report                             | 16894.256 ms             | • Applied denormalization to reduce joins<br>• Rewrote query using date range filtering<br>• Added covering index on `(order_date, product_name, quantity)` | 546.125 ms              |
| Customers with Orders Totaling More Than $500 in the Past Month | 4896.671 ms              | • Used MATERIALIZED CTE to pre-aggregate data<br>• Added covering index on `(order_date, customer_id, total_amount)`                                        | 17.112 ms               |
| Total Number of Products in Each Category                       | 57.891 ms                | • Rewrote query using pre-aggregated subquery<br>• Added index on `product(category_id)`                                                                    | 13.402 ms               |
| Top Customers by Total Spending                                 | 5003.110 ms              | • Created materialized view for customer spending<br>• Indexed `total_spending` column in MV                                                                | 0.826 ms                |
| Most Recent Orders with Customer Information (1000 Orders)      | 1014.651 ms              | • Limited and ordered orders in subquery before join<br>• Added index on `order(order_date)`                                                                | 0.981 ms                |
| Products with Low Stock (Less Than 10)                          | 19.709 ms                | • Added index on `product(stock_quantity)`                                                                                                                  | 3.975 ms                |
| Revenue Generated by Each Product Category                      | 4898.515 ms              | • Created materialized view for category revenue aggregation                                                                                                | 0.338 ms                |

