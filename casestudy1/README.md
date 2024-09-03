-- 1. What is the total amount each customer spent at the restaurant?

SELECT SUM(menu.price) AS total_amount, sales.customer_id
FROM dannys_diner.sales sales
	JOIN dannys_diner.menu menu
   ON sales.product_id = menu.product_id
GROUP BY sales.customer_id

-- 2. How many days has each customer visited the restaurant?

SELECT customer_id, COUNT(DISTINCT sales.order_date) AS total_visits
FROM dannys_diner.sales sales
GROUP BY customer_id

-- 3. What was the first item from the menu purchased by each customer?

WITH rank_cte AS (
SELECT customer_id, menu.product_id, product_name, order_date,
ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date ASC) AS rank_item
FROM dannys_diner.menu menu
JOIN dannys_diner.sales sales
	ON menu.product_id = sales.product_id
)

SELECT customer_id, product_name
FROM rank_cte 
WHERE rank_item = 1

-- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

SELECT 
  menu.product_name,
  COUNT(sales.product_id) AS most_purchased_item
FROM dannys_diner.sales
INNER JOIN dannys_diner.menu
  ON sales.product_id = menu.product_id
GROUP BY menu.product_name
ORDER BY most_purchased_item DESC
LIMIT 1

-- 5. Which item was the most popular for each customer?

WITH cte AS(
SELECT 
  sales.customer_id,
  menu.product_name,
  COUNT(sales.product_id) AS purchased_count,
  RANK() OVER ( PARTITION BY sales.customer_id
               ORDER BY COUNT(sales.product_id) DESC ) AS rank
FROM dannys_diner.sales
INNER JOIN dannys_diner.menu
  ON sales.product_id = menu.product_id
GROUP BY sales.customer_id, menu.product_name
ORDER BY customer_id, purchased_count
)

SELECT customer_id, product_name, purchased_count
FROM cte
WHERE rank = 1


-- 6. Which item was purchased first by the customer after they became a member?

WITH rank_cte AS (
SELECT sales.customer_id, menu.product_id, product_name, order_date,
ROW_NUMBER() OVER (PARTITION BY sales.customer_id ORDER BY order_date ASC) AS rank_item
FROM dannys_diner.menu menu
JOIN dannys_diner.sales sales
	ON menu.product_id = sales.product_id
JOIN dannys_diner.members
  ON sales.customer_id = members.customer_id
WHERE order_date > join_date
)


SELECT customer_id, product_name
FROM rank_cte 
WHERE rank_item = 1


-- 7. Which item was purchased just before the customer became a member?

WITH rank_cte AS (
SELECT sales.customer_id, menu.product_id, product_name, order_date,
ROW_NUMBER() OVER (PARTITION BY sales.customer_id ORDER BY order_date DESC) AS rank_item
FROM dannys_diner.menu menu
JOIN dannys_diner.sales sales
	ON menu.product_id = sales.product_id
JOIN dannys_diner.members
  ON sales.customer_id = members.customer_id
WHERE order_date < join_date
)


SELECT customer_id, product_name
FROM rank_cte 
WHERE rank_item = 1

-- 8. What is the total items and amount spent for each member before they became a member?

SELECT sales.customer_id,
COUNT(product_name) AS total_items_count,
SUM(price) AS total_amount_spent
FROM dannys_diner.menu menu
JOIN dannys_diner.sales sales
	ON menu.product_id = sales.product_id
JOIN dannys_diner.members
  ON sales.customer_id = members.customer_id
WHERE order_date < join_date
GROUP BY sales.customer_id
ORDER BY sales.customer_id

-- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

WITH cte AS (
SELECT sales.customer_id, menu.product_name,
SUM(price) AS total_amount_spent
FROM dannys_diner.menu menu
JOIN dannys_diner.sales sales
	ON menu.product_id = sales.product_id
GROUP BY sales.customer_id, menu.product_name
ORDER BY sales.customer_id

),
points_cte AS ( 
  SELECT customer_id, product_name,
  CASE 
      WHEN product_name = 'curry' OR product_name = 'ramen' THEN total_amount_spent * 10
      WHEN product_name = 'sushi' THEN total_amount_spent * 20
  END AS points
  FROM cte
)

SELECT customer_id, SUM(points) AS total_points
FROM points_cte
GROUP BY customer_id
ORDER BY customer_id


-- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

WITH cte AS(
SELECT join_date + INTERVAL '6 days' AS add_6_days, join_date, sale.order_date, sale.customer_id, price, menu.product_name, menu.product_id
FROM dannys_diner.sales sale
INNER JOIN dannys_diner.members member
	ON sale.customer_id = member.customer_id
INNER JOIN dannys_diner.menu menu
  ON menu.product_id = sale.product_id
), 


points_cte AS ( 
SELECT customer_id, product_name,
CASE WHEN order_date BETWEEN join_date AND add_6_days THEN price * 20
	 WHEN product_id = 1 THEN price * 20
	 ELSE price * 10
END AS points
FROM cte
WHERE order_date <= '2021-01-31'
)

SELECT customer_id, SUM(points) as total_points 
FROM points_cte
GROUP BY customer_id
ORDER BY customer_id




