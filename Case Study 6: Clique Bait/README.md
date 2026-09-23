# Clique Bait

## Source

https://8weeksqlchallenge.com/case-study-6/

## (SQL) Lessons Learned

* Calculating funnel fallout rates
* Working with basic ad impressions/click numbers
* Working with indexes to speed up queries, especially when repeatedly scanning through a large table.
* Usage of `MAX` and `SUM` over booleans (which are numerically stored as 0 or 1 in SQLite).
    * For example: `SUM(event_type = 1)` adds 1 to the count if there was a page view for the online store and 0 otherwise: it counts how many page views there were.
* Conditional joins.
* Campaign analysis + insights showcased in an infographic for management reporting.



## Introduction

Clique Bait is not like your regular online seafood store - the founder and CEO Danny, was also a part of a digital data analytics team and wanted to expand his knowledge into the seafood industry!

In this case study - you are required to support Danny’s vision and analyse his dataset and come up with creative solutions to calculate funnel fallout rates for the Clique Bait online store.


## Datasets

This case study contains 5 datasets.

### **Users**

Customers who visit the Clique Bait website are tagged via their `cookie_id`.

| user\_id | cookie\_id | start\_date |
| ----- | ----- | ----- |
| 397 | 3759ff | 2020-03-30 00:00:00 |
| 215 | 863329 | 2020-01-26 00:00:00 |
| 191 | eefca9 | 2020-03-15 00:00:00 |
| 89 | 764796 | 2020-01-07 00:00:00 |
| 127 | 17ccc5 | 2020-01-22 00:00:00 |
| 81 | b0b666 | 2020-03-01 00:00:00 |
| 260 | a4f236 | 2020-01-08 00:00:00 |
| 203 | d1182f | 2020-04-18 00:00:00 |
| 23 | 12dbc8 | 2020-01-18 00:00:00 |
| 375 | f61d69 | 2020-01-03 00:00:00 |

### **Events**

Customer visits are logged in this `events` table at a `cookie_id` level and the `event_type` and `page_id` values can be used to join onto relevant satellite tables to obtain further information about each event.

The sequence\_number is used to order the events within each visit.

| visit\_id | cookie\_id | page\_id | event\_type | sequence\_number | event\_time |
| ----- | ----- | ----- | ----- | ----- | ----- |
| 719fd3 | 3d83d3 | 5 | 1 | 4 | 2020-03-02 00:29:09.975502 |
| fb1eb1 | c5ff25 | 5 | 2 | 8 | 2020-01-22 07:59:16.761931 |
| 23fe81 | 1e8c2d | 10 | 1 | 9 | 2020-03-21 13:14:11.745667 |
| ad91aa | 648115 | 6 | 1 | 3 | 2020-04-27 16:28:09.824606 |
| 5576d7 | ac418c | 6 | 1 | 4 | 2020-01-18 04:55:10.149236 |
| 48308b | c686c1 | 8 | 1 | 5 | 2020-01-29 06:10:38.702163 |
| 46b17d | 78f9b3 | 7 | 1 | 12 | 2020-02-16 09:45:31.926407 |
| 9fd196 | ccf057 | 4 | 1 | 5 | 2020-02-14 08:29:12.922164 |
| edf853 | f85454 | 1 | 1 | 1 | 2020-02-22 12:59:07.652207 |
| 3c6716 | 02e74f | 3 | 2 | 5 | 2020-01-31 17:56:20.777383 |

### **Event Identifier**

The `event_identifier` table shows the types of events which are captured by Clique Bait’s digital data systems.

| event\_type | event\_name |
| ----- | ----- |
| 1 | Page View |
| 2 | Add to Cart |
| 3 | Purchase |
| 4 | Ad Impression |
| 5 | Ad Click |

### **Campaign Identifier**

This table shows information for the 3 campaigns that Clique Bait has ran on their website so far in 2020\.

| campaign\_id | products | campaign\_name | start\_date | end\_date |
| ----- | ----- | ----- | ----- | ----- |
| 1 | 1-3 | BOGOF \- Fishing For Compliments | 2020-01-01 00:00:00 | 2020-01-14 00:00:00 |
| 2 | 4-5 | 25% Off \- Living The Lux Life | 2020-01-15 00:00:00 | 2020-01-28 00:00:00 |
| 3 | 6-8 | Half Off \- Treat Your Shellf(ish) | 2020-02-01 00:00:00 | 2020-03-31 00:00:00 |

### **Page Hierarchy**

This table lists all of the pages on the Clique Bait website which are tagged and have data passing through from user interaction events.

| page\_id | page\_name | product\_category | product\_id |
| ----- | ----- | ----- | ----- |
| 1 | Home Page | null | null |
| 2 | All Products | null | null |
| 3 | Salmon | Fish | 1 |
| 4 | Kingfish | Fish | 2 |
| 5 | Tuna | Fish | 3 |
| 6 | Russian Caviar | Luxury | 4 |
| 7 | Black Truffle | Luxury | 5 |
| 8 | Abalone | Shellfish | 6 |
| 9 | Lobster | Shellfish | 7 |
| 10 | Crab | Shellfish | 8 |
| 11 | Oyster | Shellfish | 9 |
| 12 | Checkout | null | null |
| 13 | Confirmation | null | null |



## Entity Relationship Diagram

![](images/ERD.png)

**Note:**

This ERD was created using DbSchema (version 10.4.0).
