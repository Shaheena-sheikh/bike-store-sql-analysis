# bike-store-sql-analysis
-- ============================================================
-- BIKE STORE SQL ANALYSIS
-- Author: Shaheena Sheikh
-- Dataset: Bike Store Relational Database
-- Tools: MySQL
-- ============================================================


-- ============================================================
-- Q1. Top 5 best-selling products by revenue
-- Concepts: JOIN, GROUP BY, ORDER BY, LIMIT
-- Business question: Which products generated the most revenue,
--                   and which brands do they belong to?
-- ============================================================

SELECT 
    p.product_name,
    b.brand_name,
    c.category_name,
    SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS total_revenue,
    SUM(oi.quantity) AS total_units_sold
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
JOIN brands b ON p.brand_id = b.brand_id
JOIN categories c ON p.category_id = c.category_id
GROUP BY p.product_name, b.brand_name, c.category_name
ORDER BY total_revenue DESC
LIMIT 5;


-- ============================================================
-- Q2. Monthly order trends — is the business growing?
-- Concepts: DATE functions, GROUP BY, aggregation
-- Business question: How has total order volume and revenue
--                   trended month over month?
-- ============================================================

SELECT 
    YEAR(o.order_date)                                          AS order_year,
    MONTH(o.order_date)                                         AS order_month,
    DATE_FORMAT(o.order_date, '%b %Y')                          AS month_label,
    COUNT(DISTINCT o.order_id)                                  AS total_orders,
    SUM(oi.quantity * oi.list_price * (1 - oi.discount))        AS total_revenue,
    ROUND(AVG(oi.quantity * oi.list_price * (1 - oi.discount)), 2) AS avg_order_value
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY order_year, order_month, month_label
ORDER BY order_year, order_month;


-- ============================================================
-- Q3. Store performance comparison
-- Concepts: Multi-table JOIN, GROUP BY, aggregation
-- Business question: Which store generates the highest revenue
--                   and which has the most orders?
-- ============================================================

SELECT 
    s.store_name,
    s.city,
    s.state,
    COUNT(DISTINCT o.order_id)                                AS total_orders,
    SUM(oi.quantity * oi.list_price * (1 - oi.discount))     AS total_revenue,
    ROUND(
        SUM(oi.quantity * oi.list_price * (1 - oi.discount)) 
        / COUNT(DISTINCT o.order_id), 2
    )                                                         AS avg_order_value
FROM stores s
JOIN orders o ON s.store_id = o.store_id
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY s.store_name, s.city, s.state
ORDER BY total_revenue DESC;


-- ============================================================
-- Q4. Category contribution to total revenue
-- Concepts: JOIN 4 tables, GROUP BY, percentage calculation
-- Business question: What percentage of total revenue does
--                   each product category contribute?
-- ============================================================

WITH category_revenue AS (
    SELECT 
        c.category_name,
        SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS category_total
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    JOIN categories c ON p.category_id = c.category_id
    GROUP BY c.category_name
),
total AS (
    SELECT SUM(category_total) AS grand_total FROM category_revenue
)
SELECT 
    cr.category_name,
    ROUND(cr.category_total, 2)                                        AS revenue,
    ROUND((cr.category_total / t.grand_total) * 100, 2)               AS revenue_pct
FROM category_revenue cr
CROSS JOIN total t
ORDER BY revenue DESC;


-- ============================================================
-- Q5. Customer segmentation by total spend
-- Concepts: CASE WHEN, CTE, GROUP BY
-- Business question: How many customers fall into High / Mid /
--                   Low value segments based on total spend?
-- ============================================================

WITH customer_spend AS (
    SELECT 
        c.customer_id,
        CONCAT(c.first_name, ' ', c.last_name)                    AS customer_name,
        c.city,
        c.state,
        SUM(oi.quantity * oi.list_price * (1 - oi.discount))      AS total_spend
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY c.customer_id, customer_name, c.city, c.state
)
SELECT 
    customer_name,
    city,
    state,
    ROUND(total_spend, 2) AS total_spend,
    CASE 
        WHEN total_spend >= 5000 THEN 'High Value'
        WHEN total_spend >= 2000 THEN 'Mid Value'
        ELSE 'Low Value'
    END AS customer_segment
FROM customer_spend
ORDER BY total_spend DESC;


-- Segment summary count
WITH customer_spend AS (
    SELECT 
        c.customer_id,
        SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS total_spend
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY c.customer_id
),
segmented AS (
    SELECT 
        CASE 
            WHEN total_spend >= 5000 THEN 'High Value'
            WHEN total_spend >= 2000 THEN 'Mid Value'
            ELSE 'Low Value'
        END AS customer_segment
    FROM customer_spend
)
SELECT 
    customer_segment,
    COUNT(*) AS customer_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS pct_of_total
FROM segmented
GROUP BY customer_segment
ORDER BY customer_count DESC;


-- ============================================================
-- Q6. Repeat vs one-time customers
-- Concepts: CTE, COUNT, CASE WHEN, subquery
-- Business question: What proportion of customers have placed
--                   more than one order?
-- ============================================================

WITH customer_orders AS (
    SELECT 
        customer_id,
        COUNT(order_id) AS order_count
    FROM orders
    GROUP BY customer_id
),
classified AS (
    SELECT 
        CASE 
            WHEN order_count = 1 THEN 'One-time buyer'
            WHEN order_count = 2 THEN 'Occasional buyer'
            ELSE 'Repeat buyer'
        END AS buyer_type,
        order_count
    FROM customer_orders
)
SELECT 
    buyer_type,
    COUNT(*)                                                        AS customer_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2)             AS pct_of_total
FROM classified
GROUP BY buyer_type
ORDER BY customer_count DESC;


-- ============================================================
-- Q7. Staff performance — who closes the most orders?
-- Concepts: JOIN, GROUP BY, RANK() window function
-- Business question: Which staff member handles the highest
--                   number of orders and revenue per store?
-- ============================================================

WITH staff_performance AS (
    SELECT 
        s.store_id,
        st.store_name,
        CONCAT(sf.first_name, ' ', sf.last_name)                  AS staff_name,
        sf.email,
        COUNT(DISTINCT o.order_id)                                 AS total_orders,
        SUM(oi.quantity * oi.list_price * (1 - oi.discount))      AS total_revenue
    FROM staffs sf
    JOIN stores st ON sf.store_id = st.store_id
    JOIN orders o ON sf.staff_id = o.staff_id
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN stores s ON o.store_id = s.store_id
    GROUP BY s.store_id, st.store_name, staff_name, sf.email
)
SELECT 
    store_name,
    staff_name,
    total_orders,
    ROUND(total_revenue, 2)                                        AS total_revenue,
    RANK() OVER (PARTITION BY store_name ORDER BY total_revenue DESC) AS revenue_rank
FROM staff_performance
ORDER BY store_name, revenue_rank;


-- ============================================================
-- Q8. Stock health — which products are running low?
-- Concepts: JOIN, WHERE, ORDER BY
-- Business question: Which products have critically low stock
--                   (under 5 units) and in which stores?
-- ============================================================

SELECT 
    st.store_name,
    st.city,
    p.product_name,
    b.brand_name,
    c.category_name,
    sk.quantity AS stock_quantity,
    CASE 
        WHEN sk.quantity = 0 THEN 'Out of Stock'
        WHEN sk.quantity <= 2 THEN 'Critical'
        WHEN sk.quantity <= 5 THEN 'Low'
        ELSE 'OK'
    END AS stock_status
FROM stocks sk
JOIN stores st ON sk.store_id = st.store_id
JOIN products p ON sk.product_id = p.product_id
JOIN brands b ON p.brand_id = b.brand_id
JOIN categories c ON p.category_id = c.category_id
WHERE sk.quantity <= 5
ORDER BY sk.quantity ASC, st.store_name;


-- ============================================================
-- Q9. Ranking products within each category by revenue
-- Concepts: Window function RANK() OVER (PARTITION BY)
-- Business question: Within each category, which product
--                   ranks #1 in revenue?
-- ============================================================

WITH product_revenue AS (
    SELECT 
        c.category_name,
        p.product_name,
        b.brand_name,
        SUM(oi.quantity * oi.list_price * (1 - oi.discount))      AS total_revenue,
        SUM(oi.quantity)                                           AS units_sold
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    JOIN categories c ON p.category_id = c.category_id
    JOIN brands b ON p.brand_id = b.brand_id
    GROUP BY c.category_name, p.product_name, b.brand_name
),
ranked AS (
    SELECT 
        *,
        RANK() OVER (PARTITION BY category_name ORDER BY total_revenue DESC) AS rank_in_category
    FROM product_revenue
)
SELECT 
    category_name,
    rank_in_category,
    product_name,
    brand_name,
    ROUND(total_revenue, 2) AS total_revenue,
    units_sold
FROM ranked
WHERE rank_in_category <= 3
ORDER BY category_name, rank_in_category;


-- ============================================================
-- Q10. Average order value by city and month
-- Concepts: Multi-table JOIN, DATE functions, GROUP BY
-- Business question: In which city and month did customers
--                   spend the most per order on average?
-- ============================================================

SELECT 
    c.city,
    c.state,
    DATE_FORMAT(o.order_date, '%b %Y')                              AS month_label,
    YEAR(o.order_date)                                              AS yr,
    MONTH(o.order_date)                                             AS mo,
    COUNT(DISTINCT o.order_id)                                      AS total_orders,
    ROUND(
        SUM(oi.quantity * oi.list_price * (1 - oi.discount))
        / COUNT(DISTINCT o.order_id), 2
    )                                                               AS avg_order_value
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY c.city, c.state, month_label, yr, mo
HAVING total_orders >= 3
ORDER BY avg_order_value DESC
LIMIT 15;


-- ============================================================
-- BONUS: Brand performance over years
-- Concepts: CTE, DATE functions, GROUP BY, window function
-- Business question: Which brand has grown the most
--                   year over year in revenue?
-- ============================================================

WITH brand_yearly AS (
    SELECT 
        b.brand_name,
        YEAR(o.order_date)                                          AS yr,
        SUM(oi.quantity * oi.list_price * (1 - oi.discount))       AS yearly_revenue
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    JOIN brands b ON p.brand_id = b.brand_id
    JOIN orders o ON oi.order_id = o.order_id
    GROUP BY b.brand_name, yr
)
SELECT 
    brand_name,
    yr,
    ROUND(yearly_revenue, 2)                                        AS yearly_revenue,
    ROUND(
        LAG(yearly_revenue) OVER (PARTITION BY brand_name ORDER BY yr), 2
    )                                                               AS prev_year_revenue,
    ROUND(
        (yearly_revenue - LAG(yearly_revenue) OVER (PARTITION BY brand_name ORDER BY yr))
        / LAG(yearly_revenue) OVER (PARTITION BY brand_name ORDER BY yr) * 100, 2
    )                                                               AS yoy_growth_pct
FROM brand_yearly
ORDER BY brand_name, yr;
