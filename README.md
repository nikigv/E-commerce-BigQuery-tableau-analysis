Overview 
![Screenshot 2025-02-07 at 06 22 17](https://github.com/user-attachments/assets/036da5ac-a7c7-408e-89b3-637e62e75d66)


This README provides a detailed explanation of the analysis approach, methodology, and insights derived from the e-commerce dataset. The objective was to explore the data, clean and transform it for analysis, and generate meaningful insights focusing on customer behavior, sales performance, customer lifetime value (CLV), and churn prediction.

Data Exploration and Cleaning

Initial Data Exploration: I began by exploring the various tables in the dataset including orders, products, users, and customer transactions.

Product Sales Analysis:

A products table was created with total sales per product by grouping by product_id and aggregating sales for each year-month.

Duplicates were removed based on order_id, product_id, and status = 'Complete' to ensure accurate data.

The cleaned data was joined with the products table to get product names, providing a clear view of each product's total sales.

Query Summary:

Grouped data by product_id, sale_year_month, and calculated the SUM(sale_price) for completed orders.

Duplicates were handled using ROW_NUMBER and filtering out non-unique rows.

WITH duplicates AS (
  SELECT order_id, product_id, sale_price, created_at, status, ROW_NUMBER() OVER (PARTITION BY order_id, product_id ORDER BY created_at) AS row_num
  FROM bigquery-public-data.thelook_ecommerce.order_items
  WHERE status = 'Complete'
)
SELECT product_id, p.name AS product_name, FORMAT_DATE('%Y-%m', DATE(created_at)) AS sale_year_month, SUM(sale_price) AS total_sale_price
FROM (SELECT oi.* FROM bigquery-public-data.thelook_ecommerce.order_items oi
      JOIN duplicates d ON oi.order_id = d.order_id AND oi.product_id = d.product_id WHERE d.row_num = 1) clean_data
JOIN bigquery-public-data.thelook_ecommerce.products p ON clean_data.product_id = p.id
GROUP BY product_id, p.name, sale_year_month
ORDER BY sale_year_month, product_id;

Order Transaction Table:

Cleaned order transaction data with order_id, user_id, product_id, and sale_price.

Grouped by order_id, user_id, and other fields to ensure accurate representation of sales transactions.

Customer Segmentation:

Customers were categorized into three segments: New, Returning, and VIP, based on their order count and total spend.

A customer_order_summary was created to summarize order history, including first and last purchase dates.

Query Summary:

Grouped data by customer and calculated order_count, total_spend, and segmented customers based on spend behavior.

WITH customer_order_summary AS (
  SELECT u.id, u.first_name, u.last_name, u.country, u.city, COUNT(DISTINCT oi.order_id) AS order_count, SUM(oi.sale_price) AS total_spend, MIN(oi.created_at) AS first_order_date, MAX(oi.created_at) AS last_order_date
  FROM bigquery-public-data.thelook_ecommerce.order_items AS oi
  JOIN bigquery-public-data.thelook_ecommerce.users AS u ON oi.user_id = u.id
  GROUP BY u.id, u.first_name, u.last_name, u.country, u.city
),
customer_segments AS (
  SELECT id, first_name, last_name, order_count, total_spend, first_order_date, last_order_date, country, city, CASE WHEN order_count = 1 THEN 'New' WHEN order_count > 1 AND total_spend > 400 THEN 'VIP' ELSE 'Returning' END AS customer_segment
  FROM customer_order_summary
)
SELECT id, first_name, last_name, order_count, total_spend, customer_segment, country, city FROM customer_segments
ORDER BY customer_segment, total_spend DESC;

CLV and Churn Analysis

Customer Lifetime Value (CLV): Calculated CLV by summing up sales per customer, representing their total lifetime value.

Churn Status: Churned customers were identified based on their last purchase date. If a customer had not made a purchase in the last 6 months, they were marked as churned.

Query Summary:

Grouped data by user_id, calculated CLV, and determined churn status by comparing the last purchase date to the current date.

SELECT user_id, SUM(sale_price) AS clv, COUNT(DISTINCT FORMAT_DATE('%Y-%m', DATE(oi.created_at))) AS total_months, MIN(DATE(created_at)) AS first_purchase_date, MAX(DATE(created_at)) AS last_purchase_date, CASE WHEN DATE_DIFF(CURRENT_DATE(), MAX(DATE(created_at)), MONTH) > 6 THEN 'Churned' ELSE 'Active' END AS churn_status
FROM clean_data oi
JOIN bigquery-public-data.thelook_ecommerce.products p ON oi.product_id = p.id
GROUP BY user_id
ORDER BY clv DESC;

Insights Derived

Revenue Trend:

Total revenue peaked in 2024 at $4,215,356, showing an upward trend until 2024 before a slight decline to $1,144,951.

Best Selling Products:

The North Face Men's Jacket was the top seller, with total sales of $39,732. Darla had the lowest sales.

Average Order Value (AOV):

The average order value across all years was $86. High AOV for Outerwear and Coats was $148, while the lowest was for Swimwear at $58.3.

Customer Segmentation:

62% of customers were new, 34% were returning, and 4% were VIPs. VIP customers were primarily from France, Spain, and the USA.

Churn and CLV:

CLV increased until 2024, reaching $303,504,932, and dropped to $75,566,780 in recent periods.

Churn distribution was 3.26%, with churned customers mostly from China and returning customers from China as well.

Tableau Dashboard

I created a Tableau dashboard by loading four main tables: order_transaction, CLV and Churn, customer_segment, and products. Relationships were established using user_id and product_id where applicable. Key visualizations include:

Revenue Trend: Yearly sales trends showed the peak in 2024 and a recent decline.

Customer 360: Customer segments, churn, and CLV were highlighted.

Anomalies Dashboard: A line chart visualized the spike in returns, which peaked significantly in 2025.

Box Plot of Order Values: This showed outliers in order values, with average orders around $150 and outliers at $1,204.

Conclusion

This analysis provided insights into sales performance, customer behavior, and anomalies in the e-commerce dataset. The Tableau dashboard allowed interactive exploration of key metrics, customer segments, and trends, providing actionable insights for decision-maki
