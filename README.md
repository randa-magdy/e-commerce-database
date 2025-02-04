# E-Commerce Database Schema & Queries

## Overview

This project provides a relational database schema for an **E-Commerce System**, along with SQL queries to extract useful reports.

## Database Schema

The database consists of five main tables:

- **Categories**: Stores product categories.
- **Products**: Stores product details and links to categories.
- **Customers**: Stores customer details.
- **Orders**: Stores order details, including total amounts.
- **OrderDetails**: Stores details of each order, linking products to orders.

### ERD Diagram

*(You can generate and include an ERD diagram based on the schema below)*

## Database Schema Script

```sql
CREATE TABLE Categories (
    category_id SERIAL PRIMARY KEY,
    category_name VARCHAR(100) UNIQUE CHECK(CHAR_LENGTH(category_name) > 3)
);

CREATE TABLE Products (
    product_id SERIAL PRIMARY KEY,
    category_id INT,
    name VARCHAR(250) NOT NULL CHECK(CHAR_LENGTH(name) > 3),
    description TEXT,
    price DECIMAL(6, 2) NOT NULL CHECK(price > 0),
    stock_quantity INT NOT NULL DEFAULT 0,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);

CREATE TABLE Customers (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(30) NOT NULL CHECK (CHAR_LENGTH(first_name) > 2),
    last_name VARCHAR(30) NOT NULL CHECK (CHAR_LENGTH(last_name) > 2),
    email VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(225) NOT NULL CHECK (CHAR_LENGTH(password) > 5)
);

CREATE TABLE Orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT,
    order_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES Customers (customer_id)
);

CREATE TABLE OrderDetails (
    order_detail_id SERIAL PRIMARY KEY,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES Orders (order_id),
    FOREIGN KEY (product_id) REFERENCES Products (product_id)
);
```

## SQL Queries

### 1. Daily Revenue Report

Retrieve the total revenue for a specific date.

```sql
SELECT DATE(order_date) AS OrderDate, SUM(total_amount) AS DailyRevenue
FROM Orders
WHERE DATE(order_date) = '2023-10-21'
GROUP BY DATE(order_date);
```

### 2. Monthly Top-Selling Products Report

Retrieve the top-selling products in a given month.

```sql
SELECT DATE_FORMAT(order_date, '%Y-%m') AS Month,
       (SELECT name FROM products P WHERE P.product_id = OD.product_id) AS ProductName,
       SUM(quantity) AS TotalQuantity
FROM orderdetails OD
JOIN orders O ON O.order_id = OD.order_id
WHERE DATE_FORMAT(order_date, '%Y-%m') = '2025-01'
GROUP BY product_id
ORDER BY TotalQuantity DESC;
```

### 3. Customers with Orders Totaling More Than $500 in the Past Month

Retrieve customers who have placed orders totaling more than $500 in the past month.

```sql
SELECT CONCAT(first_name,' ',last_name) AS CustomerName,
       SUM(total_amount) AS TotalAmount,
       order_date AS OrderDate
FROM orders O
JOIN customers C ON O.customer_id = C.customer_id
WHERE order_date < DATE_SUB(NOW(), INTERVAL 1 MONTH)
GROUP BY C.customer_id
HAVING SUM(total_amount) > 500;
```

