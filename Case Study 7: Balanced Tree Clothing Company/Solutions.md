# **Balanced Tree Clothing Co.**

[https://8weeksqlchallenge.com/case-study-7/](https://8weeksqlchallenge.com/case-study-7/)

Queries written in (DB browser for) SQlite. 

# **Case study questions and answers**

### **High Level Sales Analysis**

1. *What was the total quantity sold for all products?*

**Query:**

```sql
SELECT SUM(qty) AS total_quantity
FROM sales  |
```

**Result:**

| total\_quantity |
| :---: |
| 45216 |

2. *What is the total generated revenue for all products before discounts?*

**Query:**

```sql
SELECT SUM(price) AS total_revenue_before_discounts
FROM sales |
```

**Result:**

| *total\_revenue\_before\_discounts* |
| :---: |
| 429290 |

3. *What was the total discount amount for all products?*

**Query:**

```sql
SELECT 
    SUM(
        price * CAST(discount AS REAL)/100
    )    AS total_discount
FROM sales |
```

**Result:**

| total\_discount |
| :---: |
| 52096.34 |

### **Transaction Analysis**

1. *How many unique transactions were there?*

**Query:**

```sql
SELECT COUNT(DISTINCT txn_id) AS unique_transactions
FROM sales |
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
FROM products_per_txn |
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
FROM revenue_per_txn |
```

**Result:**

| 25th percentile\_rpt | 50th percentile\_rpt | 75th percentile\_rpt |
| :---: | :---: | :---: |
| 116.16 | 150.5 | 185.15 |

**Note:**

In challenge 4 (Data Bank), I had manually calculated percentile statistics since SQLite lacks the base functionality. For this exercise however I found an extension for sqlite that gets us there: [https://github.com/nalgeon/sqlite-stats](https://github.com/nalgeon/sqlite-stats).

4. *What is the average discount value per transaction?*

**Query:**

```sql
WITH discount_per_transaction AS (
    SELECT 
        txn_id,
        MAX(discount) AS discount_value
    FROM sales
    GROUP BY txn_id
)
SELECT 
    ROUND(
        AVG(discount_value),
        1
    ) AS avg_discount_per_txn
FROM discount_per_transaction |
```

**Result:**

| avg\_discount\_per\_txn |
| :---: |
| 12.1 |

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
FROM members_txn_percentage |
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
FROM revenues |
```

**Result:**

| average\_member\_revenue | average\_non\_member\_revenue |
| :---: | :---: |
| 151.23 | 150.35 |

### 

### **Product Analysis**

1. *What are the top 3 products by total revenue before discount?*

**Query:**

```sql
SELECT 
    product_name,
    SUM(s.price) AS revenue_before_discount
FROM sales s
JOIN product_details ON product_id = prod_id
GROUP BY product_name
ORDER BY SUM(s.price) DESC 
LIMIT 3 |
```

**Result:**

| product\_name | revenue\_before\_discount |
| :---: | :---: |
| Blue Polo Shirt \- Mens | 72276 |
| Grey Fashion Jacket \- Womens | 68850 |
| White Tee Shirt \- Mens | 50720 |

2. *What is the total quantity, revenue and discount for each segment?*

**Query:**

```sql
SELECT 
    segment_name,
    SUM(qty) AS quantity,
    SUM(
        s.price 
        * 
        CAST(100 - discount AS REAL) / 100
    ) AS revenue,
    SUM(
        s.price
        *
        CAST(discount AS REAL)/100
    ) AS discount
FROM sales s
JOIN product_details ON product_id = prod_id
GROUP BY segment_name |
```

**Result:**

| segment\_name | quantity | revenue | discount |
| :---: | :---: | :---: | :---: |
| Jacket | 11385 | 106633.36 | 14647.64 |
| Jeans | 11349 | 60470.92 | 8393.08 |
| Shirt | 11265 | 118855.69 | 16560.31 |
| Socks | 11217 | 91233.69 | 12495.31 |

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
ON t.sales = p.quantity AND t.segment_name = p.segment_name |
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
        CAST(100 - discount AS REAL) / 100
    ) AS revenue,
    SUM(
        s.price
        *
        CAST(discount AS REAL)/100
    ) AS discount
FROM sales s
JOIN product_details ON product_id = prod_id
GROUP BY category_name |
```

**Result:**

| category\_name | quantity | revenue | discount |
| :---: | :---: | :---: | :---: |
| Mens | 22482 | 210089.38 | 29055.62 |
| Womens | 22734 | 167104.28 | 23040.72 |

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
ON t.sales = p.quantity AND t.category_name = p.category_name |
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
            CAST(100 - discount AS REAL) / 100
        ) --product revenue
        * 
        100 / segment_revenue, --percentage of total segment revenue
        1
    ) AS revenue_percentage_split
FROM sales s
JOIN product_details ON prod_id = product_id
JOIN segment_revenue USING (segment_name)
GROUP BY segment_name, product_name |
```

**Result:**

| segment\_name | product\_name | revenue\_percentage\_split |
| :---: | :---: | :---: |
| Jacket | Grey Fashion Jacket \- Womens | 56.7 |
| Jacket | Indigo Rain Jacket \- Womens | 19.5 |
| Jacket | Khaki Suit Jacket \- Womens | 23.7 |
| Jeans | Black Straight Jeans \- Womens | 57.9 |
| Jeans | Cream Relaxed Jeans \- Womens | 18.1 |
| Jeans | Navy Oversized Jeans \- Womens | 24.1 |
| Shirt | Blue Polo Shirt \- Mens | 53.4 |
| Shirt | Teal Button Up Shirt \- Mens | 9.2 |
| Shirt | White Tee Shirt \- Mens | 37.5 |
| Socks | Navy Solid Socks \- Mens | 44.4 |
| Socks | Pink Fluro Polkadot Socks \- Mens | 35.2 |
| Socks | White Striped Socks \- Mens | 20.4 |

7. *What is the percentage split of revenue by segment for each category?*

**Query:**

```sql
WITH category_revenue AS (
    SELECT
        category_name,
        SUM(
            s.price 
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
            CAST(100 - discount AS REAL) / 100
        ) --segment revenue
        * 
        100 / category_revenue, --percentage of total category revenue
        1
    ) AS revenue_percentage_split
FROM sales s
JOIN product_details ON prod_id = product_id
JOIN category_revenue USING (category_name)
GROUP BY category_name, segment_name |
```

**Result:**

| category\_name | segment\_name | revenue\_percentage\_split |
| :---: | :---: | :---: |
| Mens | Shirt | 56.6 |
| Mens | Socks | 43.4 |
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
            CAST(100 - discount AS REAL) / 100
        ) AS total_revenue
    FROM sales s
    JOIN product_details ON prod_id = product_id
)
SELECT
    category_name,
    ROUND(SUM(
        s.price 
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
GROUP BY category_name |
```

**Result:**

| category\_name | revenue\_percentage\_split |
| :---: | :---: |
| Mens | 55.7 |
| Womens | 44.3 |

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
GROUP BY product_name |
```

**Result:**

| product\_name | transaction\_penetration |
| :---: | ----- |
| Black Straight Jeans \- Womens | 0.4984 |
| Blue Polo Shirt \- Mens | 0.5072 |
| Cream Relaxed Jeans \- Womens | 0.4972 |
| Grey Fashion Jacket \- Womens | 0.51 |
| Indigo Rain Jacket \- Womens | 0.5 |
| Khaki Suit Jacket \- Womens | 0.4988 |
| Navy Oversized Jeans \- Womens | 0.5096 |
| Navy Solid Socks \- Mens | 0.5124 |
| Pink Fluro Polkadot Socks \- Mens | 0.5032 |
| Teal Button Up Shirt \- Mens | 0.4968 |
| White Striped Socks \- Mens | 0.4972 |
| White Tee Shirt \- Mens | 0.5072 |

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
LIMIT 1 |
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

| /\* \================================
   Monthly report for Balanced Tree
   \================================ \*/ 

DROP TABLE IF EXISTS monthly\_sales;

CREATE TEMP TABLE monthly\_sales AS
SELECT \*
FROM sales
WHERE
    \--Choose reporting period, set to January as default.
    date(start\_txn\_time) \>= '2021-01-01' \--Start of reporting period
    AND date(start\_txn\_time) \< '2021-02-01'; \--End of reporting period
   
/\* \================================
   High level sales analysis
   \================================ \*/
   
/\* \======================================================================
   Question 1:
   What was the total quantity sold for all products?
   Question 2:
   What is the total generated revenue for all products before discounts?
   Question 3:
   What was the total discount amount for all products?
   \====================================================================== \*/

SELECT 
    SUM(qty) AS total\_quantity,
    SUM(price) AS total\_revenue\_before\_discounts,
    SUM(
        price \* CAST(discount AS REAL)/100
    )    AS total\_discount
FROM monthly\_sales;

/\* \================================
   Transaction analysis
   \================================ \*/
   
/\* \======================================================================
   Questions that can all be answered on the txn\_id level
   
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
   \====================================================================== \*/
   
WITH products\_per\_txn AS (
    SELECT 
        txn\_id, 
        COUNT(DISTINCT prod\_id) AS unique\_products
    FROM monthly\_sales
    GROUP BY txn\_id
),
revenue\_per\_txn AS (
    SELECT 
        txn\_id,
        SUM(price) 
        \* 
        CAST(
            100 \- discount AS REAL
        ) / 100 AS revenue
    FROM monthly\_sales
    GROUP BY txn\_id
),
discount\_per\_transaction AS (
    SELECT 
        txn\_id,
        MAX(discount) AS discount\_value
    FROM monthly\_sales
    GROUP BY txn\_id
),
total\_transactions AS (
    SELECT COUNT(DISTINCT txn\_id) AS total\_transactions
    FROM monthly\_sales
),
members\_txn\_percentage AS (
    SELECT 
        ROUND(
            CAST(
                100 \* COUNT(DISTINCT txn\_id) AS REAL
            ) / total\_transactions
            ,
            1
        ) AS members\_pct
    FROM monthly\_sales
    CROSS JOIN total\_transactions
    WHERE member \= 't'
)
SELECT
    COUNT(DISTINCT txn\_id) AS unique\_transactions,
    ROUND(
        AVG(unique\_products),
        2
    ) AS average\_unique\_products,
    ROUND(percentile\_25(r.revenue), 2) AS "25th percentile\_rpt",
    ROUND(median(r.revenue), 2) "50th percentile\_rpt",
    ROUND(percentile\_75(r.revenue), 2) "75th percentile\_rpt",
    ROUND(
        AVG(discount\_value),
        1
    ) AS avg\_discount\_per\_txn,
    concat(
        members\_pct,
        '/',
        100 \- members\_pct
    ) AS members\_vs\_non\_members\_split
FROM products\_per\_txn 
JOIN revenue\_per\_txn r USING (txn\_id)
JOIN discount\_per\_transaction USING (txn\_id)
CROSS JOIN members\_txn\_percentage;



/\* \======================================================================
   Question that has to be answered on the (txn\_id, member) level
   
   Question 6:
   What is the average revenue for member 
   transactions and non-member transactions?
   \====================================================================== \*/
   
WITH revenues AS (
    SELECT 
        txn\_id,
        member,
        SUM(price) 
        \* 
        CAST(
            100 \- discount AS REAL
        ) / 100 AS revenue
    FROM monthly\_sales
    GROUP BY txn\_id, member
)
SELECT 
    ROUND(
        AVG(
            CASE 
                WHEN member \= 't'
                THEN revenue
            END
        ),
        2
    ) AS average\_member\_revenue,
    ROUND(
        AVG(
            CASE 
                WHEN member \= 'f'
                THEN revenue
            END
        ),
        2
    ) AS average\_non\_member\_revenue
FROM revenues;

/\* \================================
   Product analysis
   \================================ \*/
   
/\* \======================================================================
   Question 1:
   What are the top 3 products by total revenue before discount?
   \====================================================================== \*/
   
SELECT 
    product\_name,
    SUM(s.price) AS revenue\_before\_discount
FROM monthly\_sales s
JOIN product\_details ON product\_id \= prod\_id
GROUP BY product\_name
ORDER BY SUM(s.price) DESC 
LIMIT 3;

/\* \======================================================================
   Question 2:
   What is the total quantity, revenue and discount for each segment?
   Question 3:
   What is the top selling product for each segment?
   \====================================================================== \*/

WITH product\_quantity AS (
    SELECT 
        segment\_name,
        product\_name,
        SUM(qty) AS quantity
    FROM monthly\_sales s
    JOIN product\_details ON product\_id \= prod\_id
    GROUP BY segment\_name, product\_name
),
top\_selling AS (
    SELECT 
        segment\_name,
        MAX(quantity) AS sales
    FROM product\_quantity
    GROUP BY segment\_name
)   
SELECT 
    t.segment\_name,
    SUM(qty) AS quantity,
    SUM(
        s.price 
        \* 
        CAST(100 \- discount AS REAL) / 100
    ) AS revenue,
    SUM(
        s.price
        \*
        CAST(discount AS REAL)/100
    ) AS discount,
    p.product\_name AS top\_selling\_product,
    sales AS product\_sales
FROM monthly\_sales s
JOIN product\_details ON product\_id \= prod\_id
JOIN top\_selling t USING (segment\_name)
JOIN product\_quantity p 
ON t.sales \= p.quantity AND t.segment\_name \= p.segment\_name
GROUP BY t.segment\_name;
  
/\* \======================================================================
   Question 4:
   What is the total quantity, revenue and discount for each category?
   Question 5:
   What is the top selling product for each category?
   Question 8:
   What is the percentage split of total revenue by category?
   \====================================================================== \*/

WITH product\_quantity AS (
    SELECT 
        category\_name,
        product\_name,
        SUM(qty) AS quantity
    FROM monthly\_sales s
    JOIN product\_details ON product\_id \= prod\_id
    GROUP BY category\_name, product\_name
),
top\_selling AS (
    SELECT 
        category\_name,
        MAX(quantity) AS sales
    FROM product\_quantity
    GROUP BY category\_name
),
total\_revenue AS (
    SELECT
        SUM(
            s.price 
            \* 
            CAST(100 \- discount AS REAL) / 100
        ) AS total\_revenue
    FROM monthly\_sales s
    JOIN product\_details ON prod\_id \= product\_id
)
SELECT 
    t.category\_name,
    SUM(qty) AS quantity,
    SUM(
        s.price 
        \* 
        CAST(100 \- discount AS REAL) / 100
    ) AS revenue,
    SUM(
        s.price
        \*
        CAST(discount AS REAL)/100
    ) AS discount,
    ROUND(SUM(
        s.price 
        \* 
        CAST(100 \- discount AS REAL) / 100
    ) \--category revenue
        \* 
        100 / total\_revenue, \--percentage of total revenue
        1
    ) AS revenue\_percentage\_split,
    p.product\_name AS top\_selling\_product,
    sales AS product\_sales
FROM monthly\_sales s
JOIN product\_details ON product\_id \= prod\_id
JOIN top\_selling t USING (category\_name)
JOIN product\_quantity p 
CROSS JOIN total\_revenue
ON t.sales \= p.quantity AND t.category\_name \= p.category\_name
GROUP BY t.category\_name;

/\* \======================================================================
   Question 6:
   What is the percentage split of revenue by product for each segment?
   \====================================================================== \*/
   
WITH segment\_revenue AS (
    SELECT
        segment\_name,
        SUM(
            s.price 
            \* 
            CAST(100 \- discount AS REAL) / 100
        ) AS segment\_revenue
    FROM monthly\_sales s
    JOIN product\_details ON prod\_id \= product\_id
    GROUP BY segment\_name
)
SELECT 
    segment\_name,
    product\_name,
    ROUND(
        SUM(
            s.price 
            \* 
            CAST(100 \- discount AS REAL) / 100
        ) \--product revenue
        \* 
        100 / segment\_revenue, \--percentage of total segment revenue
        1
    ) AS revenue\_percentage\_split
FROM monthly\_sales s
JOIN product\_details ON prod\_id \= product\_id
JOIN segment\_revenue USING (segment\_name)
GROUP BY segment\_name, product\_name;

/\* \======================================================================
   Question 7:
   What is the percentage split of revenue by segment for each category?
   \====================================================================== \*/

WITH category\_revenue AS (
    SELECT
        category\_name,
        SUM(
            s.price 
            \* 
            CAST(100 \- discount AS REAL) / 100
        ) AS category\_revenue
    FROM monthly\_sales s
    JOIN product\_details ON prod\_id \= product\_id
    GROUP BY category\_name
)
SELECT 
    category\_name,
    segment\_name,
    ROUND(
        SUM(
            s.price 
            \* 
            CAST(100 \- discount AS REAL) / 100
        ) \--segment revenue
        \* 
        100 / category\_revenue, \--percentage of total category revenue
        1
    ) AS revenue\_percentage\_split
FROM monthly\_sales s
JOIN product\_details ON prod\_id \= product\_id
JOIN category\_revenue USING (category\_name)
GROUP BY category\_name, segment\_name;

/\* \======================================================================
   Question 9:
   What is the total transaction "penetration" for each product? 
   \====================================================================== \*/
   
WITH total\_transactions AS (
    SELECT 
        COUNT(DISTINCT txn\_id) AS total\_transactions
    FROM monthly\_sales
)
SELECT 
    product\_name,
    ROUND(
        CAST(
            COUNT(DISTINCT txn\_id) AS REAL
        ) / total\_transactions,
        4
    ) AS transaction\_penetration
FROM monthly\_sales s
JOIN product\_details ON prod\_id \= product\_id
CROSS JOIN total\_transactions 
WHERE qty \> 0 \--failsafe in case some rows include a product\_name without any actual sales
GROUP BY product\_name;

/\* \======================================================================
   Question 10:
   What is the most common combination of at least
   1 quantity of any 3 products in a 1 single transaction?
   \====================================================================== \*/

WITH triples AS (
    SELECT 
        txn\_id,
        s1.prod\_id AS product\_1,
        s2.prod\_id AS product\_2,
        s3.prod\_id AS product\_3,
        concat(p1.product\_name, ', ', p2.product\_name, ', ', p3.product\_name) AS triple
    FROM monthly\_sales s1
    JOIN monthly\_sales s2 USING (txn\_id)
    JOIN monthly\_sales s3 USING (txn\_id)
    JOIN product\_details p1 ON 
        s1.prod\_id \= p1.product\_id
    JOIN product\_details p2 ON 
        s2.prod\_id \= p2.product\_id
    JOIN product\_details p3 ON 
        s3.prod\_id \= p3.product\_id
    WHERE 
        \--Ordering removes duplicate triples
        product\_1 \< product\_2
        AND 
        product\_2 \< product\_3
        AND s1.qty \> 0
        AND s2.qty \> 0
        AND s3.qty \> 0
)
SELECT
    triple,
    COUNT(\*) AS Amount
FROM triples 
GROUP BY triple
ORDER BY Amount DESC 
LIMIT 1 |
| :---- |

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
    AND product_id IS NOT NULL -- Filter all rows from base case and recursion that are no longer needed |
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

