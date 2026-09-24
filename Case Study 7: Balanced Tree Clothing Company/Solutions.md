# **Balanced Tree Clothing Company**

[https://8weeksqlchallenge.com/case-study-7/](https://8weeksqlchallenge.com/case-study-7/)

Queries written in (DB browser for) SQlite. 

# **Case study questions and answers**

### **High Level Sales Analysis**

1. *What was the total quantity sold for all products?*

**Query:**

```sql
SELECT SUM(qty) AS total_quantity
FROM sales
```

**Result:**

| total\_quantity |
| :---: |
| 45216 |

2. *What is the total generated revenue for all products before discounts?*

**Query:**

```sql
SELECT SUM(price * qty) AS total_revenue_before_discounts
FROM sales
```

**Result:**

| *total\_revenue\_before\_discounts* |
| :---: |
| 1289453 |

3. *What was the total discount amount for all products?*

**Query:**

```sql
SELECT 
    SUM(
        price * qty * CAST(discount AS REAL)/100
    )    AS total_discount
FROM sales
```

**Result:**

| total\_discount |
| :---: |
| 156229.14 |

### **Transaction Analysis**

1. *How many unique transactions were there?*

**Query:**

```sql
SELECT COUNT(DISTINCT txn_id) AS unique_transactions
FROM sales
```

**Result:**

| unique\_transactions |
| :---: |
| 2500 |

2. *What is the average unique products purchased in each transaction?*

**Query:**

```sql
WITH products_per_txn AS (
    SELECT 
        txn_id, 
        COUNT(DISTINCT prod_id) AS unique_products
    FROM sales
    GROUP BY txn_id
)
SELECT 
    ROUND(
        AVG(unique_products),
        2
    ) AS average_unique_products
FROM products_per_txn
```

**Result:**

| average\_unique\_products |
| :---: |
| 6.04 |

3. *What are the 25th, 50th and 75th percentile values for the revenue per transaction?*

**Query:**

```sql
WITH revenue_per_txn AS (
    SELECT 
        txn_id,
        SUM(price) 
        *
        qty
        *
        CAST(
            100 - discount AS REAL
        ) / 100 AS revenue
    FROM sales
    GROUP BY txn_id
)
SELECT 
    ROUND(percentile_25(revenue), 2) AS "25th percentile_rpt",
    ROUND(median(revenue), 2) "50th percentile_rpt",
    ROUND(percentile_75(revenue), 2) "75th percentile_rpt"
FROM revenue_per_txn
```

**Result:**

| 25th percentile\_rpt | 50th percentile\_rpt | 75th percentile\_rpt |
| :---: | :---: | :---: |
| 229.35 | 414.2 | 628.32 |

**Note:**

* I will assume "revenue" to mean total revenue, so the discount values must be deducted from the gross revenue per transaction.
* In challenge 4 (Data Bank), I had manually calculated percentile statistics since SQLite lacks the base functionality. For this exercise however I found an extension for sqlite that gets us there: [https://github.com/nalgeon/sqlite-stats](https://github.com/nalgeon/sqlite-stats).

4. *What is the average discount value per transaction?*

**Query:**

```sql
WITH discount_per_transaction AS (
	SELECT 
		txn_id,
		SUM(price * qty) * CAST(discount AS REAL) / 100  AS discount_value
	FROM sales
	GROUP BY txn_id
)
SELECT 
	ROUND(
		AVG(discount_value),
		1
	) AS avg_discount_per_txn
FROM discount_per_transaction
```

**Result:**

| avg\_discount\_per\_txn |
| :---: |
| 62.5 |

5. *What is the percentage split of all transactions for members vs non-members?*

**Query:**

```sql
WITH total_transactions AS (
    SELECT COUNT(DISTINCT txn_id) AS total_transactions
    FROM sales
),
members_txn_percentage AS (
    SELECT 
        ROUND(
            CAST(
                100 * COUNT(DISTINCT txn_id) AS REAL
            ) / total_transactions
            ,
            1
        ) AS members_pct
    FROM sales
    CROSS JOIN total_transactions
    WHERE member = 't'
)
SELECT 
    concat(
        members_pct,
        '/',
        100 - members_pct
    ) AS members_vs_non_members_split
FROM members_txn_percentage
```

**Result:**

| members\_vs\_non\_members\_split |
| :---: |
| 60.2/39.8 |

6. *What is the average revenue for member transactions and non-member transactions?*

**Query:**

```sql
WITH revenues AS (
    SELECT 
        txn_id,
        member,
        SUM(price) 
        *
        qty
        *
        CAST(
            100 - discount AS REAL
        ) / 100 AS revenue
    FROM sales
    GROUP BY txn_id, member
)
SELECT 
    ROUND(
        AVG(
            CASE 
                WHEN member = 't'
                THEN revenue
            END
        ),
        2
    ) AS average_member_revenue,
    ROUND(
        AVG(
            CASE 
                WHEN member = 'f'
                THEN revenue
            END
        ),
        2
    ) AS average_non_member_revenue
FROM revenues
```

**Result:**

| average\_member\_revenue | average\_non\_member\_revenue |
| :---: | :---: |
| 458.03 | 455.81 |

### 

### **Product Analysis**

1. *What are the top 3 products by total revenue before discount?*

**Query:**

```sql
SELECT 
    product_name,
    SUM(s.price * qty) AS revenue_before_discount
FROM sales s
JOIN product_details ON product_id = prod_id
GROUP BY product_name
ORDER BY SUM(s.price) DESC 
LIMIT 3
```

**Result:**

| product\_name | revenue\_before\_discount |
| :---: | :---: |
| Blue Polo Shirt \- Mens | 217683 |
| Grey Fashion Jacket \- Womens | 209304 |
| White Tee Shirt \- Mens | 152000 |

2. *What is the total quantity, revenue and discount for each segment?*

**Query:**

```sql
SELECT 
	segment_name,
	SUM(qty) AS quantity,
	SUM(
		s.price 
		*
		qty
		*
		CAST(100 - discount AS REAL) / 100
	) AS revenue,
	SUM(
		s.price
		*
		qty
		*
		CAST(discount AS REAL)/100
	) AS discount
FROM sales s
JOIN product_details ON product_id = prod_id
GROUP BY segment_name
```

**Result:**

| segment\_name | quantity | revenue | discount |
| :---: | :---: | :---: | :---: |
| Jacket | 11385 | 322705.54 | 44277.46 |
| Jeans | 11349 | 183006.03 | 25343.97 |
| Shirt | 11265 | 356548.73 | 49594.27 |
| Socks | 11217 | 270963.56 | 37013.44 |

3. *What is the top selling product for each segment?*

**Query:**

```sql
WITH product_quantity AS (
    SELECT 
        segment_name,
        product_name,
        SUM(qty) AS quantity
    FROM sales s
    JOIN product_details ON product_id = prod_id
    GROUP BY segment_name, product_name
),
top_selling AS (
    SELECT 
        segment_name,
        MAX(quantity) AS sales
    FROM product_quantity
    GROUP BY segment_name
)
--Join back
SELECT 
    t.segment_name,
    product_name,
    sales
FROM top_selling t
JOIN product_quantity p 
ON t.sales = p.quantity AND t.segment_name = p.segment_name
```

**Result:**

| segment\_name | product\_name | sales |
| :---: | :---: | :---: |
| Jacket | Grey Fashion Jacket \- Womens | 3876 |
| Jeans | Navy Oversized Jeans \- Womens | 3856 |
| Shirt | Blue Polo Shirt \- Mens | 3819 |
| Socks | Navy Solid Socks \- Mens | 3792 |

4. *What is the total quantity, revenue and discount for each category?*

**Query:**

```sql
SELECT 
    category_name,
    SUM(qty) AS quantity,
    SUM(
        s.price 
        *
		qty
		*
        CAST(100 - discount AS REAL) / 100
    ) AS revenue,
    SUM(
        s.price
        *
		qty
		*
        CAST(discount AS REAL)/100
    ) AS discount
FROM sales s
JOIN product_details ON product_id = prod_id
GROUP BY category_name
```

**Result:**

| category\_name | quantity | revenue | discount |
| :---: | :---: | :---: | :---: |
| Mens | 22482 | 627512.29 | 86607.71 |
| Womens | 22734 | 505711.57 | 69621.43 |

5. *What is the top selling product for each category?*

**Query:**

```sql
WITH product_quantity AS (
    SELECT 
        category_name,
        product_name,
        SUM(qty) AS quantity
    FROM sales s
    JOIN product_details ON product_id = prod_id
    GROUP BY category_name, product_name
),
top_selling AS (
    SELECT 
        category_name,
        MAX(quantity) AS sales
    FROM product_quantity
    GROUP BY category_name
)
--Join back
SELECT 
    t.category_name,
    product_name,
    sales
FROM top_selling t
JOIN product_quantity p 
ON t.sales = p.quantity AND t.category_name = p.category_name
```

**Result:**

| category\_name | product\_name | sales |
| :---: | :---: | :---: |
| Mens | Blue Polo Shirt \- Mens | 3819 |
| Womens | Grey Fashion Jacket \- Womens | 3876 |

6. *What is the percentage split of revenue by product for each segment?*

**Query:**

```sql
WITH segment_revenue AS (
	SELECT
		segment_name,
		SUM(
			s.price 
			* 
			qty
			*
			CAST(100 - discount AS REAL) / 100
		) AS segment_revenue
	FROM sales s
	JOIN product_details ON prod_id = product_id
	GROUP BY segment_name
)
SELECT 
	segment_name,
	product_name,
	ROUND(
		SUM(
			s.price 
			* 
			qty
			*
			CAST(100 - discount AS REAL) / 100
		) --product revenue
		* 
		100 / segment_revenue, --percentage of total segment revenue
		1
	) AS revenue_percentage_split
FROM sales s
JOIN product_details ON prod_id = product_id
JOIN segment_revenue USING (segment_name)
GROUP BY segment_name, product_name
```

**Result:**

| segment\_name | product\_name | revenue\_percentage\_split |
| :---: | :---: | :---: |
| Jacket | Grey Fashion Jacket \- Womens | 57.0 |
| Jacket | Indigo Rain Jacket \- Womens | 19.4 |
| Jacket | Khaki Suit Jacket \- Womens | 23.6 |
| Jeans | Black Straight Jeans \- Womens | 58.1 |
| Jeans | Cream Relaxed Jeans \- Womens | 17.8 |
| Jeans | Navy Oversized Jeans \- Womens | 24.0 |
| Shirt | Blue Polo Shirt \- Mens | 53.5 |
| Shirt | Teal Button Up Shirt \- Mens | 9.0 |
| Shirt | White Tee Shirt \- Mens | 37.5 |
| Socks | Navy Solid Socks \- Mens | 44.2 |
| Socks | Pink Fluro Polkadot Socks \- Mens | 35.6 |
| Socks | White Striped Socks \- Mens | 20.2 |

7. *What is the percentage split of revenue by segment for each category?*

**Query:**

```sql
WITH category_revenue AS (
	SELECT
		category_name,
		SUM(
			s.price 
			*
			qty
			*
			CAST(100 - discount AS REAL) / 100
		) AS category_revenue
	FROM sales s
	JOIN product_details ON prod_id = product_id
	GROUP BY category_name
)
SELECT 
	category_name,
	segment_name,
	ROUND(
		SUM(
			s.price 
			*
		    qty
		    *	
			CAST(100 - discount AS REAL) / 100
		) --segment revenue
		* 
		100 / category_revenue, --percentage of total category revenue
		1
	) AS revenue_percentage_split
FROM sales s
JOIN product_details ON prod_id = product_id
JOIN category_revenue USING (category_name)
GROUP BY category_name, segment_name
```

**Result:**

| category\_name | segment\_name | revenue\_percentage\_split |
| :---: | :---: | :---: |
| Mens | Shirt | 56.8 |
| Mens | Socks | 43.2 |
| Womens | Jacket | 63.8 |
| Womens | Jeans | 36.2 |

8. *What is the percentage split of total revenue by category?*

**Query:**

```sql
WITH total_revenue AS (
	SELECT
		SUM(
			s.price 
			*
			qty
			*
			CAST(100 - discount AS REAL) / 100
		) AS total_revenue
	FROM sales s
	JOIN product_details ON prod_id = product_id
)
SELECT
	category_name,
	ROUND(
		SUM(
			s.price 
			*
			qty
			*
			CAST(100 - discount AS REAL) / 100
		) --category revenue
		* 
		100 / total_revenue, --percentage of total revenue
		1
	) AS revenue_percentage_split
FROM sales s
JOIN product_details ON prod_id = product_id
CROSS JOIN total_revenue
GROUP BY category_name
```

**Result:**

| category\_name | revenue\_percentage\_split |
| :---: | :---: |
| Mens | 55.4 |
| Womens | 44.6 |

9. *What is the total transaction “penetration” for each product? (hint: penetration \= number of transactions where at least 1 quantity of a product was purchased divided by total number of transactions)*

**Query:**

```sql
WITH total_transactions AS (
    SELECT 
        COUNT(DISTINCT txn_id) AS total_transactions
    FROM sales
)
SELECT 
    product_name,
    CAST(
        COUNT(DISTINCT txn_id) AS REAL
    ) / total_transactions AS transaction_penetration
FROM sales s
JOIN product_details ON prod_id = product_id
CROSS JOIN total_transactions 
WHERE qty > 0 --failsafe in case some rows include a product_name without any actual sales
GROUP BY product_name
ORDER BY transaction_penetration DESC
```

**Result:**

| product\_name | transaction\_penetration |
| :---: | ----- |
| Navy Solid Socks \- Mens | 0.5124 |
| Grey Fashion Jacket \- Womens | 0.51 |
| Navy Oversized Jeans \- Womens | 0.5096 |
| White Tee Shirt \- Mens | 0.5072 |
| Blue Polo Shirt \- Mens | 0.5072 |
| Pink Fluro Polkadot Socks \- Mens | 0.5032 |
| Indigo Rain Jacket \- Womens | 0.5 |
| Khaki Suit Jacket \- Womens | 0.4988 |
| Black Straight Jeans \- Womens | 0.4984 |
| White Striped Socks \- Mens | 0.4972 |
| Cream Relaxed Jeans \- Womens | 0.4972 |
| Teal Button Up Shirt \- Mens | 0.4968 |



10. *What is the most common combination of at least 1 quantity of any 3 products in a 1 single transaction?*

**Query:**

```sql
WITH triples AS (
    SELECT 
        txn_id,
        s1.prod_id AS product_1,
        s2.prod_id AS product_2,
        s3.prod_id AS product_3,
        concat(p1.product_name, ', ', p2.product_name, ', ', p3.product_name) AS triple
    FROM sales s1
    JOIN sales s2 USING (txn_id)
    JOIN sales s3 USING (txn_id)
    JOIN product_details p1 ON 
        s1.prod_id = p1.product_id
    JOIN product_details p2 ON 
        s2.prod_id = p2.product_id
    JOIN product_details p3 ON 
        s3.prod_id = p3.product_id
    WHERE 
        --Ordering removes duplicate triples
        product_1 < product_2
        AND 
        product_2 < product_3
        AND s1.qty > 0
        AND s2.qty > 0
        AND s3.qty > 0
)
SELECT
    triple,
    COUNT(*) AS Amount
FROM triples 
GROUP BY triple
ORDER BY Amount DESC 
LIMIT 1 
```

**Result:**

| triple | Amount |
| :---: | :---: |
| White Tee Shirt \- Mens, Grey Fashion Jacket \- Womens, Teal Button Up Shirt \- Mens | 352 |

**Learned:**

* **Self-joining multiple times** to create permutations.  
* **Filtering by ordering the products** to remove duplicate triples and only keep the combinations.

### **Reporting Challenge**

*Write a single SQL script that combines all of the previous questions into a scheduled report that the Balanced Tree team can run at the beginning of each month to calculate the previous month’s values.*

*Imagine that the Chief Financial Officer (which is also Danny) has asked for all of these questions at the end of every month.*

*He first wants you to generate the data for January only \- but then he also wants you to demonstrate that you can easily run the same analysis for February without many changes (if at all).*

*Feel free to split up your final outputs into as many tables as you need \- but be sure to explicitly reference which table outputs relate to which question for full marks :)*

**Script:**

<details>
<summary> Click to expand answer! </summary>

```sql
/* ================================
   Monthly report for Balanced Tree
   ================================ */ 

DROP TABLE IF EXISTS monthly_sales;

CREATE TEMP TABLE monthly_sales AS
SELECT *
FROM sales
WHERE
	--Choose reporting period, set to January as default.
	date(start_txn_time) >= '2021-01-01' --Start of reporting period
	AND date(start_txn_time) < '2021-02-01'; --End of reporting period
   
/* ================================
   High level sales analysis
   ================================ */
   
/* ======================================================================
   Question 1:
   What was the total quantity sold for all products?
   Question 2:
   What is the total generated revenue for all products before discounts?
   Question 3:
   What was the total discount amount for all products?
   ====================================================================== */

SELECT 
	SUM(qty) AS total_quantity,
	SUM(price * qty) AS total_revenue_before_discounts,
	SUM(
		price * qty * CAST(discount AS REAL)/100
	)	AS total_discount
FROM monthly_sales;

/* ================================
   Transaction analysis
   ================================ */
   
/* ======================================================================
   Questions that can all be answered on the txn_id level
   
   Question 1:
   How many unique transactions were there?
   Question 2:
   What is the average unique products purchased in each transaction?
   Question 3:
   What are the 25th, 50th and 75th percentile values 
   for the revenue per transaction?
   Question 4:
   What is the average discount value per transaction?
   Question 5:
   What is the percentage split of all 
   transactions for members vs non-members?
   ====================================================================== */
   
WITH products_per_txn AS (
    SELECT 
        txn_id, 
        COUNT(DISTINCT prod_id) AS unique_products
    FROM monthly_sales
    GROUP BY txn_id
),
revenue_per_txn AS (
    SELECT 
        txn_id,
        SUM(price) 
        *
		qty
		*
        CAST(
            100 - discount AS REAL
        ) / 100 AS revenue
    FROM monthly_sales
    GROUP BY txn_id
),
discount_per_transaction AS (
    SELECT 
        txn_id,
        SUM(price * qty) * CAST(discount AS REAL) / 100  AS discount_value
    FROM monthly_sales
    GROUP BY txn_id
),
total_transactions AS (
    SELECT COUNT(DISTINCT txn_id) AS total_transactions
    FROM monthly_sales
),
members_txn_percentage AS (
    SELECT 
        ROUND(
            CAST(
                100 * COUNT(DISTINCT txn_id) AS REAL
            ) / total_transactions
            ,
            1
        ) AS members_pct
    FROM monthly_sales
    CROSS JOIN total_transactions
    WHERE member = 't'
)
SELECT
	COUNT(DISTINCT txn_id) AS unique_transactions,
    ROUND(
        AVG(unique_products),
        2
    ) AS average_unique_products,
	ROUND(percentile_25(r.revenue), 2) AS "25th percentile_rpt",
    ROUND(median(r.revenue), 2) "50th percentile_rpt",
    ROUND(percentile_75(r.revenue), 2) "75th percentile_rpt",
	ROUND(
        AVG(discount_value),
        1
    ) AS avg_discount_per_txn,
	concat(
        members_pct,
        '/',
        100 - members_pct
    ) AS members_vs_non_members_split
FROM products_per_txn 
JOIN revenue_per_txn r USING (txn_id)
JOIN discount_per_transaction USING (txn_id)
CROSS JOIN members_txn_percentage;



/* ======================================================================
   Question that has to be answered on the (txn_id, member) level
   
   Question 6:
   What is the average revenue for member 
   transactions and non-member transactions?
   ====================================================================== */
   
WITH revenues AS (
    SELECT 
        txn_id,
        member,
        SUM(price) 
        *
		qty 
		*
        CAST(
            100 - discount AS REAL
        ) / 100 AS revenue
    FROM monthly_sales
    GROUP BY txn_id, member
)
SELECT 
    ROUND(
        AVG(
            CASE 
                WHEN member = 't'
                THEN revenue
            END
        ),
        2
    ) AS average_member_revenue,
    ROUND(
        AVG(
            CASE 
                WHEN member = 'f'
                THEN revenue
            END
        ),
        2
    ) AS average_non_member_revenue
FROM revenues;

/* ================================
   Product analysis
   ================================ */
   
/* ======================================================================
   Question 1:
   What are the top 3 products by total revenue before discount?
   ====================================================================== */
   
SELECT 
	product_name,
	SUM(s.price * qty) AS revenue_before_discount
FROM monthly_sales s
JOIN product_details ON product_id = prod_id
GROUP BY product_name
ORDER BY revenue_before_discount DESC 
LIMIT 3;

/* ======================================================================
   Question 2:
   What is the total quantity, revenue and discount for each segment?
   Question 3:
   What is the top selling product for each segment?
   ====================================================================== */

WITH product_quantity AS (
    SELECT 
        segment_name,
        product_name,
        SUM(qty) AS quantity
    FROM monthly_sales s
    JOIN product_details ON product_id = prod_id
    GROUP BY segment_name, product_name
),
top_selling AS (
    SELECT 
        segment_name,
        MAX(quantity) AS sales
    FROM product_quantity
    GROUP BY segment_name
)   
SELECT 
    t.segment_name,
    SUM(qty) AS quantity,
    SUM(
        s.price 
        *
		qty 
		*
        CAST(100 - discount AS REAL) / 100
    ) AS revenue,
    SUM(
        s.price
        *
		qty 
		*
        CAST(discount AS REAL)/100
    ) AS discount,
    p.product_name AS top_selling_product,
    sales AS product_sales
FROM monthly_sales s
JOIN product_details ON product_id = prod_id
JOIN top_selling t USING (segment_name)
JOIN product_quantity p 
ON t.sales = p.quantity AND t.segment_name = p.segment_name
GROUP BY t.segment_name;
  
/* ======================================================================
   Question 4:
   What is the total quantity, revenue and discount for each category?
   Question 5:
   What is the top selling product for each category?
   Question 8:
   What is the percentage split of total revenue by category?
   ====================================================================== */

WITH product_quantity AS (
    SELECT 
        category_name,
        product_name,
        SUM(qty) AS quantity
    FROM monthly_sales s
    JOIN product_details ON product_id = prod_id
    GROUP BY category_name, product_name
),
top_selling AS (
    SELECT 
        category_name,
        MAX(quantity) AS sales
    FROM product_quantity
    GROUP BY category_name
),
total_revenue AS (
    SELECT
        SUM(
            s.price 
            *
			qty
			*
            CAST(100 - discount AS REAL) / 100
        ) AS total_revenue
    FROM monthly_sales s
    JOIN product_details ON prod_id = product_id
)
SELECT 
    t.category_name,
    SUM(qty) AS quantity,
    SUM(
        s.price 
        *
		qty
		*
        CAST(100 - discount AS REAL) / 100
    ) AS revenue,
    SUM(
        s.price
        *
		qty
		*
        CAST(discount AS REAL)/100
    ) AS discount,
	ROUND(SUM(
        s.price 
        *
		qty
		*
        CAST(100 - discount AS REAL) / 100
    ) --category revenue
        * 
        100 / total_revenue, --percentage of total revenue
        1
    ) AS revenue_percentage_split,
    p.product_name AS top_selling_product,
    sales AS product_sales
FROM monthly_sales s
JOIN product_details ON product_id = prod_id
JOIN top_selling t USING (category_name)
JOIN product_quantity p 
CROSS JOIN total_revenue
ON t.sales = p.quantity AND t.category_name = p.category_name
GROUP BY t.category_name;

/* ======================================================================
   Question 6:
   What is the percentage split of revenue by product for each segment?
   ====================================================================== */
   
WITH segment_revenue AS (
    SELECT
        segment_name,
        SUM(
            s.price 
            *
			qty
			*
            CAST(100 - discount AS REAL) / 100
        ) AS segment_revenue
    FROM monthly_sales s
    JOIN product_details ON prod_id = product_id
    GROUP BY segment_name
)
SELECT 
    segment_name,
    product_name,
    ROUND(
        SUM(
            s.price 
            *
			qty
			*
            CAST(100 - discount AS REAL) / 100
        ) --product revenue
        * 
        100 / segment_revenue, --percentage of total segment revenue
        1
    ) AS revenue_percentage_split
FROM monthly_sales s
JOIN product_details ON prod_id = product_id
JOIN segment_revenue USING (segment_name)
GROUP BY segment_name, product_name;

/* ======================================================================
   Question 7:
   What is the percentage split of revenue by segment for each category?
   ====================================================================== */

WITH category_revenue AS (
    SELECT
        category_name,
        SUM(
            s.price 
            *
			qty
			*
            CAST(100 - discount AS REAL) / 100
        ) AS category_revenue
    FROM monthly_sales s
    JOIN product_details ON prod_id = product_id
    GROUP BY category_name
)
SELECT 
    category_name,
    segment_name,
    ROUND(
        SUM(
            s.price 
            *
			qty
			*
            CAST(100 - discount AS REAL) / 100
        ) --segment revenue
        * 
        100 / category_revenue, --percentage of total category revenue
        1
    ) AS revenue_percentage_split
FROM monthly_sales s
JOIN product_details ON prod_id = product_id
JOIN category_revenue USING (category_name)
GROUP BY category_name, segment_name;

/* ======================================================================
   Question 9:
   What is the total transaction “penetration” for each product? 
   ====================================================================== */
   
WITH total_transactions AS (
    SELECT 
        COUNT(DISTINCT txn_id) AS total_transactions
    FROM monthly_sales
)
SELECT 
    product_name,
    ROUND(
		CAST(
			COUNT(DISTINCT txn_id) AS REAL
		) / total_transactions,
		4
	) AS transaction_penetration
FROM monthly_sales s
JOIN product_details ON prod_id = product_id
CROSS JOIN total_transactions 
WHERE qty > 0 --failsafe in case some rows include a product_name without any actual sales
GROUP BY product_name
ORDER BY transaction_penetration DESC;

/* ======================================================================
   Question 10:
   What is the most common combination of at least
   1 quantity of any 3 products in a 1 single transaction?
   ====================================================================== */

WITH triples AS (
    SELECT 
        txn_id,
        s1.prod_id AS product_1,
        s2.prod_id AS product_2,
        s3.prod_id AS product_3,
        concat(p1.product_name, ', ', p2.product_name, ', ', p3.product_name) AS triple
    FROM monthly_sales s1
    JOIN monthly_sales s2 USING (txn_id)
    JOIN monthly_sales s3 USING (txn_id)
    JOIN product_details p1 ON 
        s1.prod_id = p1.product_id
    JOIN product_details p2 ON 
        s2.prod_id = p2.product_id
    JOIN product_details p3 ON 
        s3.prod_id = p3.product_id
    WHERE 
        --Ordering removes duplicate triples
        product_1 < product_2
        AND 
        product_2 < product_3
        AND s1.qty > 0
        AND s2.qty > 0
        AND s3.qty > 0
)
SELECT
    triple,
    COUNT(*) AS Amount
FROM triples 
GROUP BY triple
ORDER BY Amount DESC 
LIMIT 1
```

</details>

**Results (for the month January):**

**High level sales analysis:**

1\. *What was the total quantity sold for all products?*  
2\. *What is the total generated revenue for all products before discounts?*  
3\. *What was the total discount amount for all products?*

| total\_quantity | total\_revenue\_before\_discounts | total\_discount |
| :---: | :---: | :---: |
| 14788 | 420672 | 51589.1 |

**Transaction analysis:**

1\. *How many unique transactions were there?*  
2\. *What is the average unique products purchased in each transaction?*  
3\. *What are the 25th, 50th and 75th percentile values for the revenue per transaction?*  
4\. *What is the average discount value per transaction?*  
5\. *What is the percentage split of all transactions for members vs non-members?*

| unique\_transactions | average\_unique\_products | 25th percentile\_rpt | 50th percentile\_rpt | 75th percentile\_rpt | avg\_discount\_per\_txn | members\_vs\_non\_members\_split |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 828 | 5.99 | 226.27 | 417.3 | 617.76 | 62.3 | 59.4/40.6 |

6\. *What is the average revenue for member transactions and non-member transactions?*

| average\_member\_revenue | average\_non\_member\_revenue |
| :---: | :---: |
| 455.41 | 441.84 |

**Product analysis:** 

1\. *What are the top 3 products by total revenue before discount?*

| product\_name | revenue\_before\_discount |
| :---: | :---: |
| Grey Fashion Jacket \- Womens | 70200 |
| Blue Polo Shirt \- Mens | 69198 |
| White Tee Shirt \- Mens | 50240 |

2\. *What is the total quantity, revenue and discount for each segment?*  
3\. *What is the top selling product for each segment?*

| segment\_name | quantity | revenue | discount | top\_selling\_product | product\_sales |
| :---: | :---: | :---: | :---: | :---: | :---: |
| Jacket | 3750 | 106778.62 | 14871.38 | Grey Fashion Jacket \- Womens | 1300 |
| Jeans | 3777 | 60294.32 | 8482.68 | Cream Relaxed Jeans \- Womens | 1282 |
| Shirt | 3690 | 115409.62 | 16228.38 | White Tee Shirt \- Mens | 1256 |
| Socks | 3571 | 86600.34 | 12006.66 | Navy Solid Socks \- Mens | 1264 |

4\. *What is the total quantity, revenue and discount for each category?*  
5\. *What is the top selling product for each category?*  
8\. *What is the percentage split of total revenue by category?*

| category\_name | quantity | revenue | discount | revenue\_percentage\_split | top\_selling\_product | product\_sales |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Mens | 7261 | 202009.96 | 28235.04 | 54.7 | Navy Solid Socks \- Mens | 1264 |
| Womens | 7527 | 167072.94 | 23354.06 | 45.3 | Grey Fashion Jacket \- Womens | 1300 |

6\. *What is the percentage split of revenue by product for each segment?*

| segment\_name | product\_name | revenue\_percentage\_split |
| :---: | :---: | :---: |
| Jacket | Grey Fashion Jacket \- Womens | 57.7 |
| Jacket | Indigo Rain Jacket \- Womens | 19.1 |
| Jacket | Khaki Suit Jacket \- Womens | 23.2 |
| Jeans | Black Straight Jeans \- Womens | 57.6 |
| Jeans | Cream Relaxed Jeans \- Womens | 18.6 |
| Jeans | Navy Oversized Jeans \- Womens | 23.7 |
| Shirt | Blue Polo Shirt \- Mens | 52.6 |
| Shirt | Teal Button Up Shirt \- Mens | 9.2 |
| Shirt | White Tee Shirt \- Mens | 38.2 |
| Socks | Navy Solid Socks \- Mens | 46.1 |
| Socks | Pink Fluro Polkadot Socks \- Mens | 34.0 |
| Socks | White Striped Socks \- Mens | 19.9 |

7\. *What is the percentage split of revenue by segment for each category?*

| category\_name | segment\_name | revenue\_percentage\_split |
| :---: | :---: | :---: |
| Mens | Shirt | 57.1 |
| Mens | Socks | 42.9 |
| Womens | Jacket | 63.9 |
| Womens | Jeans | 36.1 |

9\. *What is the total transaction “penetration” for each product?*

| product\_name | transaction\_penetration |
| :---: | :---: |
| Cream Relaxed Jeans \- Womens | 0.5217 |
| Grey Fashion Jacket \- Womens | 0.5205 |
| Navy Oversized Jeans \- Womens | 0.5109 |
| Navy Solid Socks \- Mens | 0.5072 |
| White Tee Shirt \- Mens | 0.5024 |
| Blue Polo Shirt \- Mens | 0.4988 |
| Teal Button Up Shirt \- Mens | 0.4964 |
| Black Straight Jeans \- Womens | 0.4928 |
| Indigo Rain Jacket \- Womens | 0.4915 |
| Khaki Suit Jacket \- Womens | 0.4855 |
| White Striped Socks \- Mens | 0.4819 |
| Pink Fluro Polkadot Socks \- Mens | 0.4783 |

10\. *What is the most common combination of at least 1 quantity of any 3 products in a 1 single transaction?*

| triple | Amount |
| :---: | :---: |
| White Tee Shirt \- Mens, Grey Fashion Jacket \- Womens, Teal Button Up Shirt \- Mens | 125 |


**Learned:**

Using temporary tables to re-use a subset of data over many queries.

### **Bonus Challenge**

*Use a single SQL query to transform the `product_hierarchy` and `product_prices` datasets to the `product_details` table.*

*Hint: you may want to consider using a recursive CTE to solve this problem\!*

**Query:**

```sql
WITH RECURSIVE product_details2 AS (
    SELECT 
        id, -- i.e. style_id 
        product_id,
        price,
        parent_id,
        NULL AS category_id,
        NULL AS segment_id,
        id AS style_id,
        NULL AS category_name,
        NULL AS segment_name,
        NULL AS style_name
    FROM product_prices
    FULL JOIN product_hierarchy h USING (id)
    
    UNION ALL
    
    SELECT 
        p.parent_id, -- parent i.e. segment_id on loop 1 and category_id on loop 2
        p.product_id,
        p.price,
        h1.parent_id, -- grandparent i.e. category_id on loop 1 and NULL on loop 2
        p.parent_id, -- category on loop 2
        p.id, -- segment on loop 2
        p.style_id, -- style on every loop 
        h1.level_text,
        h2.level_text,
        h3.level_text
    FROM product_details2 p
    LEFT JOIN product_hierarchy h1 ON p.parent_id = h1.id -- Current parent becomes the next "normal" id
    LEFT JOIN product_hierarchy h2 ON p.id = h2.id -- Find segment name in final loop
    LEFT JOIN product_hierarchy h3 ON p.style_id = h3.id -- Find style name
    WHERE p.parent_id IS NOT NULL            -- Loops twice: once from id to parent, then from parent to grandparent            -- (parents of grandparents are always NULL so the loop stops there)
)
SELECT 
    product_id,
    price,
    concat(
        style_name,
        ' ',
        segment_name,
        ' - ',
        category_name
    ) AS product_name,
    category_id,
    segment_id,
    style_id,
    category_name,
    segment_name,
    style_name
FROM product_details2
WHERE 
    parent_id IS NULL 
    AND product_id IS NOT NULL -- Filter all rows from base case and recursion that are no longer needed 
```

**Result:**

| product\_id | price | product\_name | category\_id | segment\_id | style\_id | category\_name | segment\_name | style\_name |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| c4a632 | 13 | Navy Oversized Jeans \- Womens | 1 | 3 | 7 | Womens | Jeans | Navy Oversized |
| e83aa3 | 32 | Black Straight Jeans \- Womens | 1 | 3 | 8 | Womens | Jeans | Black Straight |
| e31d39 | 10 | Cream Relaxed Jeans \- Womens | 1 | 3 | 9 | Womens | Jeans | Cream Relaxed |
| d5e9a6 | 23 | Khaki Suit Jacket \- Womens | 1 | 4 | 10 | Womens | Jacket | Khaki Suit |
| 72f5d4 | 19 | Indigo Rain Jacket \- Womens | 1 | 4 | 11 | Womens | Jacket | Indigo Rain |
| 9ec847 | 54 | Grey Fashion Jacket \- Womens | 1 | 4 | 12 | Womens | Jacket | Grey Fashion |
| 5d267b | 40 | White Tee Shirt \- Mens | 2 | 5 | 13 | Mens | Shirt | White Tee |
| c8d436 | 10 | Teal Button Up Shirt \- Mens | 2 | 5 | 14 | Mens | Shirt | Teal Button Up |
| 2a2353 | 57 | Blue Polo Shirt \- Mens | 2 | 5 | 15 | Mens | Shirt | Blue Polo |
| f084eb | 36 | Navy Solid Socks \- Mens | 2 | 6 | 16 | Mens | Socks | Navy Solid |
| b9a74d | 17 | White Striped Socks \- Mens | 2 | 6 | 17 | Mens | Socks | White Striped |
| 2feb6b | 29 | Pink Fluro Polkadot Socks \- Mens | 2 | 6 | 18 | Mens | Socks | Pink Fluro Polkadot |

**Learned:**

Recursively constructing a product dataset from a hierarchy table.
