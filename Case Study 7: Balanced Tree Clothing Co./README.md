# Balanced Tree Clothing Co.

## Source

https://8weeksqlchallenge.com/case-study-7/

## (SQL) Lessons Learned

* 


## Introduction

Balanced Tree Clothing Company prides themselves on providing an optimised range of clothing and lifestyle wear for the modern adventurer!

Danny, the CEO of this trendy fashion company has asked you to assist the team’s merchandising teams analyse their sales performance and generate a basic financial report to share with the wider business.

## Datasets

This case study contains 4 datasets, though 2 of them are building blocks for one of the tables that we rebuild later in a bonus question. So effectively, there are only 2 relevant tables.

### **Product Details**

`balanced_tree.product_details` includes all information about the entire range that Balanced Clothing sells in their store.

| product\_id | price | product\_name | category\_id | segment\_id | style\_id | category\_name | segment\_name | style\_name |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- |
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

### **Product Sales**

`balanced_tree.sales` contains product level information for all the transactions made for Balanced Tree including quantity, price, percentage discount, member status, a transaction ID and also the transaction timestamp.

| prod\_id | qty | price | discount | member | txn\_id | start\_txn\_time |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| c4a632 | 4 | 13 | 17 | t | 54f307 | 2021-02-13 01:59:43.296 |
| 5d267b | 4 | 40 | 17 | t | 54f307 | 2021-02-13 01:59:43.296 |
| b9a74d | 4 | 17 | 17 | t | 54f307 | 2021-02-13 01:59:43.296 |
| 2feb6b | 2 | 29 | 17 | t | 54f307 | 2021-02-13 01:59:43.296 |
| c4a632 | 5 | 13 | 21 | t | 26cc98 | 2021-01-19 01:39:00.3456 |
| e31d39 | 2 | 10 | 21 | t | 26cc98 | 2021-01-19 01:39:00.3456 |
| 72f5d4 | 3 | 19 | 21 | t | 26cc98 | 2021-01-19 01:39:00.3456 |
| 2a2353 | 3 | 57 | 21 | t | 26cc98 | 2021-01-19 01:39:00.3456 |
| f084eb | 3 | 36 | 21 | t | 26cc98 | 2021-01-19 01:39:00.3456 |
| c4a632 | 1 | 13 | 21 | f | ef648d | 2021-01-27 02:18:17.1648 |

### **Product Hierarcy & Product Price**

Thes tables are used only for the bonus question where we will use them to recreate the `balanced_tree.product_details` table.

**`balanced_tree.product_hierarchy`**

| id | parent\_id | level\_text | level\_name |
| ----- | ----- | ----- | ----- |
| 1 |   | Womens | Category |
| 2 |   | Mens | Category |
| 3 | 1 | Jeans | Segment |
| 4 | 1 | Jacket | Segment |
| 5 | 2 | Shirt | Segment |
| 6 | 2 | Socks | Segment |
| 7 | 3 | Navy Oversized | Style |
| 8 | 3 | Black Straight | Style |
| 9 | 3 | Cream Relaxed | Style |
| 10 | 4 | Khaki Suit | Style |
| 11 | 4 | Indigo Rain | Style |
| 12 | 4 | Grey Fashion | Style |
| 13 | 5 | White Tee | Style |
| 14 | 5 | Teal Button Up | Style |
| 15 | 5 | Blue Polo | Style |
| 16 | 6 | Navy Solid | Style |
| 17 | 6 | White Striped | Style |
| 18 | 6 | Pink Fluro Polkadot | Style |

**`balanced_tree.product_prices`**

| id | product\_id | price |
| ----- | ----- | ----- |
| 7 | c4a632 | 13 |
| 8 | e83aa3 | 32 |
| 9 | e31d39 | 10 |
| 10 | d5e9a6 | 23 |
| 11 | 72f5d4 | 19 |
| 12 | 9ec847 | 54 |
| 13 | 5d267b | 40 |
| 14 | c8d436 | 10 |
| 15 | 2a2353 | 57 |
| 16 | f084eb | 36 |
| 17 | b9a74d | 17 |
| 18 | 2feb6b | 29 |


## Entity Relationship Diagram

![](images/ERD.png)

**Note:**

This ERD was created using DbSchema (version 10.4.0).
