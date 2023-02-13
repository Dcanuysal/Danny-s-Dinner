# Danny's-Dinner
Analysis of data
---
CREATE SCHEMA dannys_diner;
SET search_path = dannys_diner;

CREATE TABLE sales (
  "customer_id" VARCHAR(1),
  "order_date" DATE,
  "product_id" INTEGER
);

INSERT INTO sales
  ("customer_id", "order_date", "product_id")
VALUES
  ('A', '2021-01-01', '1'),
  ('A', '2021-01-01', '2'),
  ('A', '2021-01-07', '2'),
  ('A', '2021-01-10', '3'),
  ('A', '2021-01-11', '3'),
  ('A', '2021-01-11', '3'),
  ('B', '2021-01-01', '2'),
  ('B', '2021-01-02', '2'),
  ('B', '2021-01-04', '1'),
  ('B', '2021-01-11', '1'),
  ('B', '2021-01-16', '3'),
  ('B', '2021-02-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-07', '3');
 

CREATE TABLE menu (
  "product_id" INTEGER,
  "product_name" VARCHAR(5),
  "price" INTEGER
);

INSERT INTO menu
  ("product_id", "product_name", "price")
VALUES
  ('1', 'sushi', '10'),
  ('2', 'curry', '15'),
  ('3', 'ramen', '12');
  

CREATE TABLE members (
  "customer_id" VARCHAR(1),
  "join_date" DATE
);

INSERT INTO members
  ("customer_id", "join_date")
VALUES
  ('A', '2021-01-07'),
  ('B', '2021-01-09');
  
  -- 1. What is the total amount each customer spent at the restaurant?
--
SELECT customer_id,
SUM(price)
FROM Sales as s
INNER JOIN menu as m ON s.product_id = m.product_id
GROUP BY customer_id

-- 2. How many days has each customer visited the restaurant?
---
SELECT customer_id,
COUNT(DISTINCT order_date)
FROM SALES
GROUP BY customer_id

-- 3. What was the first item from the menu purchased by each customer?
---
WITH CTE AS(
SELECT customer_id,order_date,product_name,
RANK() OVER(PARTITION BY customer_id ORDER BY order_date ASC) AS rnk
FROM Sales as s
INNER JOIN menu as m ON s.product_id = m.product_id
) SELECT customer_id,product_name
FROM CTE
WHERE rnk = 1

-- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
---
SELECT product_name,
COUNT(order_date) as orders
FROM Sales as S
INNER JOIN menu as m ON s.product_id = m.product_id
GROUP BY product_name
ORDER BY COUNT(order_date) DESC
LIMIT 1

-- 5. Which item was the most popular for each customer?
---
WITH CTE AS(
SELECT 
product_name,
customer_id,
COUNT(order_date) as orders,
RANK() OVER(PARTITION BY customer_id ORDER BY COUNT(order_date) DESC) AS rnk
FROM Sales as S
INNER JOIN menu as m ON s.product_id = m.product_id
GROUP BY product_name, customer_id
	)
	SELECT customer_id,product_name
	FROM CTE
	WHERE rnk = 1
 
 -- 6. Which item was purchased first by the customer after they became a member?
--
WITH CTE AS (
SELECT 
S.customer_id,
order_date,
join_date,
product_name,
RANK() OVER(PARTITION BY S.customer_id ORDER BY order_date) AS rnk
FROM SALES AS S
INNER JOIN members as MEM ON MEM.customer_id = S.customer_id
INNER JOIN menu as m ON s.product_id = m.product_id
WHERE order_date >= join_date
GROUP BY S.customer_id,order_date,join_date,product_name
	)
	SELECT customer_id,product_name
	FROM CTE
WHERE rnk =1

-- 7. Which item was purchased just before the customer became a member?
--
WITH CTE AS (
SELECT 
S.customer_id,
order_date,
join_date,
product_name,
RANK() OVER(PARTITION BY S.customer_id ORDER BY order_date DESC) AS rnk
FROM SALES AS S
INNER JOIN members as MEM ON MEM.customer_id = S.customer_id
INNER JOIN menu as m ON s.product_id = m.product_id
WHERE order_date < join_date
GROUP BY S.customer_id,order_date,join_date,product_name
	)
	SELECT customer_id,product_name
	FROM CTE
WHERE rnk =1

-- 8. What is the total items and amount spent for each member before they became a member?
--
SELECT 
S.customer_id,	
COUNT(product_name) as total_items,
SUM(price) as amount_spent
FROM SALES AS S
INNER JOIN members as MEM ON MEM.customer_id = S.customer_id
INNER JOIN menu as m ON s.product_id = m.product_id
WHERE order_date < join_date
GROUP BY S.customer_id

-- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
--
SELECT customer_id,
SUM(CASE
WHEN product_name = 'sushi' THEN price * 10 * 2
ELSE price * 10
END) as points
FROM menu as M
INNER JOIN SALES AS S ON S.product_id = m.product_id
GROUP BY customer_id

-- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
--
WITH
 first_week_program
AS
 (
  SELECT
   s.customer_id, 
   CASE WHEN m.product_name = 'sushi' AND
    s.order_date BETWEEN ms.join_date + CAST(-1 || 'day' AS INTERVAL) and 
    ms.join_date + cast(6 || 'day' AS INTERVAL) THEN m.price*20
    WHEN product_name = 'sushi' OR
    s.order_date BETWEEN ms.join_date + CAST(-1 || 'day' AS INTERVAL) 
   AND
   ms.join_date + CAST(6 || 'day' AS INTERVAL) THEN m.price*20
   ELSE m.price*10 END AS points
  FROM dannys_diner.members ms
  LEFT JOIN dannys_diner.sales s 
  ON s.customer_id = ms.customer_id
  LEFT JOIN dannys_diner.menu m 
  ON s.product_id = m.product_id
  WHERE s.order_date <= '20210131'
 )
SELECT customer_id, SUM(points) points
FROM first_week_program
GROUP BY customer_id;

--BONUS QUESTIONS-(JOIN ALL THE THINGS)
--
WITH summary AS
(
SELECT s.customer_id, s.order_date, m.product_name, m.price,
CASE
 WHEN ms.join_date > s.order_date THEN 'N'
 WHEN ms.join_date <= s.order_date THEN 'Y'
 ELSE 'N' END AS member
FROM dannys_diner.sales AS s
LEFT JOIN dannys_diner.menu AS m
 ON s.product_id = m.product_id
LEFT JOIN dannys_diner.members AS ms
 ON s.customer_id = ms.customer_id
  )
  
 SELECT *, CASE WHEN member = 'N' THEN NULL ELSE RANK() OVER(PARTITION BY customer_id,member ORDER BY order_date)
 END AS ranking
 FROM summary

