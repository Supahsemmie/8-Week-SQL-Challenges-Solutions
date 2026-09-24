# **Fresh Segments**

[https://8weeksqlchallenge.com/case-study-8/](https://8weeksqlchallenge.com/case-study-8/)

Queries written in (DB browser for) SQlite. 

## Case Study Questions

*The following questions can be considered key business questions that are required to be answered for the Fresh Segments team.*
*Most questions can be answered using a single query however some questions are more open ended and require additional thought and not just a coded solution!*

### Data Exploration and Cleansing

1. *Update the `fresh_segments.interest_metrics` table by modifying the `month_year` column to be a date data type with the start of the month*

**Query:**

```sql
UPDATE interest_metrics
SET 
    month_year = CASE 
                    WHEN length(_month) = 1 THEN concat(_year, '-0', _month, '-01') --add zero for single digit month: 7 -> 07
                    WHEN length(_month) = 2 THEN concat(_year, '-', _month, '-01') --do not add zero for double digit month: 10 -> 10
                 END
```

**Result (first 10 rows):**

| **\_month** | **\_year** | **month_year** | **interest_id** | **composition** | **index_value** | **ranking** | **percentile_ranking** |
| ----------- | ---------- | -------------- | --------------- | --------------- | --------------- | ----------- | ---------------------- |
| 7           | 2018       | 2018-07-01     | 32486           | 11.89           | 6.19            | 1           | 99.86                  |
| 7           | 2018       | 2018-07-01     | 6106            | 9.93            | 5.31            | 2           | 99.73                  |
| 7           | 2018       | 2018-07-01     | 18923           | 10.85           | 5.29            | 3           | 99.59                  |
| 7           | 2018       | 2018-07-01     | 6344            | 10.32           | 5.1             | 4           | 99.45                  |
| 7           | 2018       | 2018-07-01     | 100             | 10.77           | 5.04            | 5           | 99.31                  |
| 7           | 2018       | 2018-07-01     | 69              | 10.82           | 5.03            | 6           | 99.18                  |
| 7           | 2018       | 2018-07-01     | 79              | 11.21           | 4.97            | 7           | 99.04                  |
| 7           | 2018       | 2018-07-01     | 6111            | 10.71           | 4.83            | 8           | 98.9                   |
| 7           | 2018       | 2018-07-01     | 6214            | 9.71            | 4.83            | 8           | 98.9                   |
| 7           | 2018       | 2018-07-01     | 19422           | 10.11           | 4.81            | 10          | 98.63                  |

**Note:**

SQLite does not have a `date` datatype, but instead uses `TEXT`. SQLite’s `ALTER TABLE` functionalities are also notoriously lacking, so I have changed the datatype using DB browser for SQLite instead. The alternative is dropping and rebuilding the table with the correct datatypes.

I reckon the point of this exercise is to keep in mind what datatypes your columns are. In other SQL implementations, one needs to be strict about tracking datatypes. SQLite however has flexible typing so even if one does not change the data type of `month_year` at all, future queries will run just fine.

2. *What is the count of records in the `fresh_segments.interest_metrics` for each `month_year` value sorted in chronological order (earliest to latest) with the null values appearing first?*

**Query:**

```sql
SELECT 
    month_year,
    COUNT(*) AS record_count
FROM interest_metrics
GROUP BY month_year
ORDER BY month_year
```

**Result:**

| **month_year** | **record_count** |
| -------------- | ---------------- |
| NULL           | 1194             |
| 2018-07-01     | 729              |
| 2018-08-01     | 767              |
| 2018-09-01     | 780              |
| 2018-10-01     | 857              |
| 2018-11-01     | 928              |
| 2018-12-01     | 995              |
| 2019-01-01     | 973              |
| 2019-02-01     | 1121             |
| 2019-03-01     | 1136             |
| 2019-04-01     | 1099             |
| 2019-05-01     | 857              |
| 2019-06-01     | 824              |
| 2019-07-01     | 864              |
| 2019-08-01     | 1149             |

3. *What do you think we should do with these null values in the `fresh_segments.interest_metrics`*

**Answer:**

The `NULL` values correspond to rows without any information on the `month_year` nor the `interest_id`. This makes these rows unsuitable for a lot of analysis *unless* we can deduce their values from the other columns.

Every `index_value` corresponds to how many times larger the `composition` value is of a specific interest in a specific month and year compared to the average. Hence, if we divide the composition by the index_value, then we get that specific average back.

If all averages are known, then this information can partially identify the missing date and interest values. Not completely unfortunately, because several interest/date combinations can have the exact same average. 

Furthermore, from this table alone we cannot deduce the averages for interest/date combinations that we have not seen yet, and for those we have seen already, we already know the interest/dates. 

It seems therefore suitable to filter out the `NULL` rows in future analysis if they are not relevant (e.g. looking at how well every `interest_id` does on average over all months). However, for some analysis it might still be useful, like if we want to know what the (average) `composition` values are for all rows that have rank 100 or below (how well are the top 100 doing?).

In conclusion, we do not remove the `NULL` rows but we will filter them out on a case-by-case basis.

4. *How many `interest_id` values exist in the `fresh_segments.interest_metrics` table but not in the `fresh_segments.interest_map` table? What about the other way around?*

**Query:**

```sql
SELECT 
    COUNT(DISTINCT CASE
        WHEN interest_id IS NOT NULL AND id IS NULL 
        THEN interest_id
    END) AS "Interests in metrics but not maps",
    COUNT(DISTINCT CASE
        WHEN interest_id IS NULL AND id IS NOT NULL 
        THEN id
    END) AS "Interests in maps but not metrics"
FROM interest_metrics
FULL JOIN interest_map ON interest_id = id
```

**Result:**

| **Interests in metrics but not maps** | **Interests in maps but not metrics** |
| ------------------------------------- | ------------------------------------- |
| 0                                     | 7                                     |

**Learned:**

When I use the `DISTINCT` keyword to count distinct interest ids, I have to actually return the `interest_id` and `id` after the `THEN` keyword, rather than writing `THEN 1` as I am used to. 

This is because if you are counting based on a condition, then the result of the `CASE` is irrelevant: you are just counting how many not-`NULL` results there are. But in this case, we need to keep in mind that the results (the `id`) also need to be distinct, and so we need to return the actual `id` values so that the `DISTINCT` keyword can be applied afterwards.

5. *Summarise the `id` values in the `fresh_segments.interest_map` by its total record count in this table*

**Note:**

I am going to assume that the intention is to count how many times every `id` value from `interest_map` occurs in `interest_metrics`. Otherwise, this would be a trivial question, since every `id` only occurs exactly once in `interest_map`.

**Query:**


```sql
SELECT 
    id,
    interest_name,
    COUNT(*) AS amount
FROM interest_metrics
LEFT JOIN interest_map ON interest_id = id
WHERE id IS NOT NULL
GROUP BY id
```

**Result (first 10 rows):**

| **id** | **interest_name**         | **amount** |
| ------ | ------------------------- | ---------- |
| 1      | Fitness Enthusiasts       | 12         |
| 2      | Gamers                    | 11         |
| 3      | Car Enthusiasts           | 10         |
| 4      | Luxury Retail Researchers | 14         |
| 5      | Brides & Wedding Planners | 14         |
| 6      | Vacation Planners         | 14         |
| 7      | Motorcycle Enthusiasts    | 11         |
| 8      | Business News Readers     | 13         |
| 12     | Thrift Store Shoppers     | 14         |
| 13     | Advertising Professionals | 13         |

6. *What sort of table join should we perform for our analysis and why? Check your logic by checking the rows where `interest_id = 21246` in your joined output and include all columns from `fresh_segments.interest_metrics` and all columns from `fresh_segments.interest_map` except from the `id` column.*

**Answer:**

We should use a `LEFT JOIN`, with `interest_metrics` on the left. This is because:

1. We keep all the `NULL` rows as mentioned in question 1.3;
2. Every non-`NULL` `interest_id` exists in `interest_maps` as seen in question 1.4, so we only need to join the interest information for every interest in `interest_metrics`. 

Below is the table join result that we save as a view called `interests`.

**Query:**

```sql
SELECT
    _month,
    _year,
    month_year,
    interest_id,
    composition,
    index_value,
    ranking,
    percentile_ranking,
    interest_name,
    interest_summary,
    created_at,
    last_modified
FROM interest_metrics
LEFT JOIN interest_map ON interest_id = id
```

**Result (first 10 rows):**

| **\_month** | **\_year** | **month_year** | **interest_id** | **composition** | **index_value** | **ranking** | **percentile_ranking** | **interest_name**                          | **interest_summary**                                                                                                            | **created_at**      | **last_modified**   |
| ----------- | ---------- | -------------- | --------------- | --------------- | --------------- | ----------- | ---------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- | ------------------- | ------------------- |
| 7           | 2018       | 2018-07-01     | 32486           | 11.89           | 6.19            | 1           | 99.86                  | Vacation Rental Accommodation Researchers  | People researching and booking rentals accommodations for vacations.                                                            | 2018-06-29 12:55:03 | 2018-06-29 12:55:03 |
| 7           | 2018       | 2018-07-01     | 6106            | 9.93            | 5.31            | 2           | 99.73                  | Luxury Second Home Owners                  | High income individuals with more than one home.                                                                                | 2017-03-27 16:59:29 | 2018-05-23 11:30:12 |
| 7           | 2018       | 2018-07-01     | 18923           | 10.85           | 5.29            | 3           | 99.59                  | Online Home Decor Shoppers                 | Consumers shopping online for home decor available for delivery.                                                                | 2018-04-19 18:25:02 | 2018-04-19 18:25:02 |
| 7           | 2018       | 2018-07-01     | 6344            | 10.32           | 5.1             | 4           | 99.45                  | Hair Care Shoppers                         | Consumers researching trends and purchasing hair and beauty products.                                                           | 2017-05-15 13:04:55 | 2018-05-31 22:11:37 |
| 7           | 2018       | 2018-07-01     | 100             | 10.77           | 5.04            | 5           | 99.31                  | Nutrition Conscious Eaters                 | Consumer reading about healthy eating options.                                                                                  | 2016-05-26 14:57:59 | 2018-05-23 11:30:13 |
| 7           | 2018       | 2018-07-01     | 69              | 10.82           | 5.03            | 6           | 99.18                  | Healthy Eaters                             | People researching healthy eating options.                                                                                      | 2016-05-26 14:57:59 | 2018-05-23 11:30:12 |
| 7           | 2018       | 2018-07-01     | 79              | 11.21           | 4.97            | 7           | 99.04                  | Luxury Travel Researchers                  | Consumers reading online reviews of luxury travel options.                                                                      | 2016-05-26 14:57:59 | 2018-05-23 11:30:12 |
| 7           | 2018       | 2018-07-01     | 6111            | 10.71           | 4.83            | 8           | 98.9                   | Wine Lovers                                | Consumers researching wine and purchasing alcohol online. These consumers are more likely to purchase wine and visit vineyards. | 2017-03-27 16:59:29 | 2017-12-07 12:35:47 |
| 7           | 2018       | 2018-07-01     | 6214            | 9.71            | 4.83            | 8           | 98.9                   | Home Remodelers                            | People researching techniques and resources for home remodels.                                                                  | 2017-03-27 16:59:29 | 2018-05-23 11:30:12 |
| 7           | 2018       | 2018-07-01     | 19422           | 10.11           | 4.81            | 10          | 98.63                  | Home Design and Living Publication Readers | People reading publications focused on design and living at home.                                                               | 2018-05-08 11:55:03 | 2018-05-08 11:55:03 |

And below is a sanity check by checking the rows where `interest_id = 21246` and seeing that the row with NULL valued `month_year` is correctly kept in-tact and joined.

**Query:**

```sql
SELECT
    _month,
    _year,
    month_year,
    interest_id,
    composition,
    index_value,
    ranking,
    percentile_ranking,
    interest_name,
    interest_summary,
    created_at,
    last_modified
FROM interest_metrics
LEFT JOIN interest_map ON interest_id = id
WHERE interest_id = 21246
```

**Result:**

| **\_month** | **\_year** | **month_year** | **interest_id** | **composition** | **index_value** | **ranking** | **percentile_ranking** | **interest_name**                | **interest_summary**                                  | **created_at**      | **last_modified**   |
| ----------- | ---------- | -------------- | --------------- | --------------- | --------------- | ----------- | ---------------------- | -------------------------------- | ----------------------------------------------------- | ------------------- | ------------------- |
| 7           | 2018       | 2018-07-01     | 21246           | 2.26            | 0.65            | 722         | 0.96                   | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11 17:50:04 | 2018-06-11 17:50:04 |
| 8           | 2018       | 2018-08-01     | 21246           | 2.13            | 0.59            | 765         | 0.26                   | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11 17:50:04 | 2018-06-11 17:50:04 |
| 9           | 2018       | 2018-09-01     | 21246           | 2.06            | 0.61            | 774         | 0.77                   | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11 17:50:04 | 2018-06-11 17:50:04 |
| 10          | 2018       | 2018-10-01     | 21246           | 1.74            | 0.58            | 855         | 0.23                   | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11 17:50:04 | 2018-06-11 17:50:04 |
| 11          | 2018       | 2018-11-01     | 21246           | 2.25            | 0.78            | 908         | 2.16                   | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11 17:50:04 | 2018-06-11 17:50:04 |
| 12          | 2018       | 2018-12-01     | 21246           | 1.97            | 0.7             | 983         | 1.21                   | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11 17:50:04 | 2018-06-11 17:50:04 |
| 1           | 2019       | 2019-01-01     | 21246           | 2.05            | 0.76            | 954         | 1.95                   | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11 17:50:04 | 2018-06-11 17:50:04 |
| 2           | 2019       | 2019-02-01     | 21246           | 1.84            | 0.68            | 1109        | 1.07                   | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11 17:50:04 | 2018-06-11 17:50:04 |
| 3           | 2019       | 2019-03-01     | 21246           | 1.75            | 0.67            | 1123        | 1.14                   | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11 17:50:04 | 2018-06-11 17:50:04 |
| 4           | 2019       | 2019-04-01     | 21246           | 1.58            | 0.63            | 1092        | 0.64                   | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11 17:50:04 | 2018-06-11 17:50:04 |
| NULL        | NULL       | NULL           | 21246           | 1.61            | 0.68            | 1191        | 0.25                   | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11 17:50:04 | 2018-06-11 17:50:04 |

7. *Are there any records in your joined table where the `month_year` value is before the `created_at` value from the `fresh_segments.interest_map` table? Do you think these values are valid and why?*

**Answer:**

Filtering rows for this criteria directly gives the following query:

**Query:**

```sql
SELECT 
    month_year,
    created_at
FROM interests
WHERE month_year < created_at
```

**Result (first 5 results):**

| **month_year** | **created_at**      |
| -------------- | ------------------- |
| 2018-07-01     | 2018-07-06 14:35:04 |
| 2018-07-01     | 2018-07-17 10:40:03 |
| 2018-07-01     | 2018-07-06 14:35:04 |
| 2018-07-01     | 2018-07-06 14:35:03 |
| 2018-07-01     | 2018-07-06 14:35:04 |

For these 5 records, the values are valid, because they were all created in July 2018 which matches their `month_year`. We get these results because `month_year` is set to be the first day of that month. Hence, interests that were created in that month will always be *after* the date in the `month_year` column. 

To more directly check if there are any possibly invalid records, we have to check if there are any records for which the interest was created in the month after `month_year`:

**Query:**

```sql
SELECT 
    COUNT(*) AS amount
FROM interests
WHERE month_year < date(created_at, 'start of month')
```

**Result:**

| **amount** |
| ---------- |
| 0          |

By transforming all `created_at` times to the date of the start of the month, we now actually filter for records where the `month_year` is before the interest creation date’s month. It can be seen that there are no such cases in this dataset.

### Interest Analysis

1. *Which interests have been present in all `month_year` dates in our dataset?*

We first confirm that every interest occurs at most once per `month_year`:

**Query:**

```sql
WITH amounts AS (
    SELECT 
        interest_id,
        month_year,
        COUNT(*) AS interest_amnt
    FROM interests
    WHERE interest_id IS NOT NULL
    GROUP BY interest_id, month_year
)
SELECT COUNT(*) AS amount
FROM amounts
WHERE interest_amnt > 1
```

**Result:**

| **amount** |
| ---------- |
| 0          |

Then, we count how many distinct `month_year` exist in the dataset and we find all the interests that occur that often in the dataset (excluding counting rows with `NULL` dates such as for example one row for interest 21246).

**Query:**

```sql
WITH total_months AS (
    SELECT COUNT(DISTINCT month_year) AS total_months
    FROM interests
),
interest_counts AS (
    SELECT 
        interest_id,
        COUNT(*) AS interest_count 
    FROM interests
    WHERE month_year IS NOT NULL --Only consider interests with known dates
    GROUP BY interest_id
)
SELECT interest_id AS id_present_in_all_months
FROM interest_counts
CROSS JOIN total_months
WHERE interest_count = total_months
```

**Result (first 10 rows):**

| **id_present_in_all_months** |
| ---------------------------- |
| 100                          |
| 10008                        |
| 10009                        |
| 10010                        |
| 101                          |
| 102                          |
| 10249                        |
| 10250                        |
| 10251                        |
| 10284                        |

2. *Using this same `total_months` measure - calculate the cumulative percentage of all records starting at 14 months - which `total_months` value passes the 90% cumulative percentage value?*

**Query:**

```sql
WITH RECURSIVE total_months AS (
    SELECT COUNT(DISTINCT month_year) AS total_months
    FROM interests
),
total_interests AS (
    SELECT COUNT(DISTINCT interest_id) AS total_interests
    FROM interests
),
interest_counts AS (
    SELECT 
        interest_id,
        COUNT(*) AS interest_count 
    FROM interests
    WHERE month_year IS NOT NULL --Only consider interests with known dates
    GROUP BY interest_id
),
--Create rows for 1 to 14 months
all_months AS (
    SELECT
        total_months 
    FROM total_months
    
    UNION ALL
    
    SELECT 
        total_months - 1
    FROM all_months
    WHERE total_months > 1
)
SELECT 
    total_months,
    ROUND(
        100 * CAST(
            COUNT(
                CASE 
                    WHEN interest_count >= total_months 
                    THEN 1 
                END) 
            AS REAL
        ) / total_interests,
        1
    ) AS cumulative_percentage
FROM all_months
CROSS JOIN total_interests
CROSS JOIN interest_counts
GROUP BY total_months
ORDER BY total_months DESC
```

**Result:**

| **total_months** | **cumulative_percentage** |
| ---------------- | ------------------------- |
| 14               | 39.9                      |
| 13               | 46.8                      |
| 12               | 52.2                      |
| 11               | 60.0                      |
| 10               | 67.1                      |
| 9                | 75.0                      |
| 8                | 80.6                      |
| 7                | 88.1                      |
| 6                | 90.8                      |
| 5                | 94.0                      |
| 4                | 96.7                      |
| 3                | 97.9                      |
| 2                | 98.9                      |
| 1                | 100.0                     |

**Answer:**

We can see that the 90% cumulative percentage threshold is first passed when the `total_months` is 6. Hence, 90% of all interests have records in at least 6 months from the dataset. 

3. *If we were to remove all `interest_id` values which are lower than the `total_months` value we found in the previous question - how many total data points would we be removing?*

**Query:**

```sql
WITH interest_counts AS (
    SELECT 
        interest_id,
        COUNT(*) AS interest_count 
    FROM interests
    WHERE month_year IS NOT NULL --Only consider interests with known dates
    GROUP BY interest_id
)
SELECT 
    COUNT(
        CASE 
            WHEN interest_count < 6
            THEN 1
        END
    ) AS data_points_removed
FROM interests
JOIN interest_counts USING (interest_id)
```

**Result:**

| **data_points_removed** |
| ----------------------- |
| 400                     |

**Note:**

We have to be careful to count the total data points that are removed, and **not** the total amount of interests that are removed. This is why the final `SELECT` statement takes its data from the `interests` table rather than the `interest_counts` CTE.

4. *Does this decision make sense to remove these data points from a business perspective? Use an example where there are all 14 months present to a removed `interest` example for your arguments - think about what it means to have less months present from a segment perspective.*

**Answer:**

Yes. When a significant amount of months are lacking (as is the case for all removed interests: they have less than 6 months of data) then within that segment of (related) interest(s) there will either be many gaps between months or large gaps over time.

In that case, looking at trends in compositions for example will not be very fruitful, nor will averages be very representative of the full time period. When all 14 months are present, then you can make a graph that follows the metrics you are interested in on a monthly granularity instead. 

5. *After removing these interests - how many unique interests are there for each month?*

**Query:**

```sql
WITH interest_counts AS (
    SELECT 
        interest_id,
        COUNT(*) AS interest_count 
    FROM interests
    WHERE month_year IS NOT NULL --Only consider interests with known dates
    GROUP BY interest_id
)
SELECT 
    month_year,
    COUNT(DISTINCT interest_id) AS unique_interests
FROM interests
JOIN interest_counts USING (interest_id)
WHERE 
    interest_count >= 6 
    AND month_year IS NOT NULL
GROUP BY month_year
```

**Result:**

| **month_year** | **unique_interests** |
| -------------- | -------------------- |
| 2018-07-01     | 709                  |
| 2018-08-01     | 752                  |
| 2018-09-01     | 774                  |
| 2018-10-01     | 853                  |
| 2018-11-01     | 925                  |
| 2018-12-01     | 986                  |
| 2019-01-01     | 966                  |
| 2019-02-01     | 1072                 |
| 2019-03-01     | 1078                 |
| 2019-04-01     | 1035                 |
| 2019-05-01     | 827                  |
| 2019-06-01     | 804                  |
| 2019-07-01     | 836                  |
| 2019-08-01     | 1062                 |

### Segment Analysis

1. *Using our filtered dataset by removing the interests with less than 6 months worth of data, which are the top 10 and bottom 10 interests which have the largest composition values in any `month_year`? Only use the maximum composition value for each interest but you must keep the corresponding `month_year`*

**Answer:**

First we create a view called `filtered_interests` that removes interests with less than 6 months worth of data as the result of the following query:

**Query:**

```sql
WITH interest_counts AS (
    SELECT 
        interest_id,
        COUNT(*) AS interest_count 
    FROM interests
    GROUP BY interest_id
)
SELECT 
    _month,
    _year,
    month_year,
    interest_id,
    composition,
    index_value,
    ranking,
    percentile_ranking,
    interest_name,
    interest_summary,
    created_at,
    last_modified
FROM interests
JOIN interest_counts USING (interest_id)
WHERE interest_count >= 6
```

**Result (first 5 rows):**

| **\_month** | **\_year** | **month_year** | **interest_id** | **composition** | **index_value** | **ranking** | **percentile_ranking** | **interest_name**                         | **interest_summary**                                                  | **created_at**      | **last_modified**   |
| ----------- | ---------- | -------------- | --------------- | --------------- | --------------- | ----------- | ---------------------- | ----------------------------------------- | --------------------------------------------------------------------- | ------------------- | ------------------- |
| 7           | 2018       | 2018-07-01     | 32486           | 11.89           | 6.19            | 1           | 99.86                  | Vacation Rental Accommodation Researchers | People researching and booking rentals accommodations for vacations.  | 2018-06-29 12:55:03 | 2018-06-29 12:55:03 |
| 7           | 2018       | 2018-07-01     | 6106            | 9.93            | 5.31            | 2           | 99.73                  | Luxury Second Home Owners                 | High income individuals with more than one home.                      | 2017-03-27 16:59:29 | 2018-05-23 11:30:12 |
| 7           | 2018       | 2018-07-01     | 18923           | 10.85           | 5.29            | 3           | 99.59                  | Online Home Decor Shoppers                | Consumers shopping online for home decor available for delivery.      | 2018-04-19 18:25:02 | 2018-04-19 18:25:02 |
| 7           | 2018       | 2018-07-01     | 6344            | 10.32           | 5.1             | 4           | 99.45                  | Hair Care Shoppers                        | Consumers researching trends and purchasing hair and beauty products. | 2017-05-15 13:04:55 | 2018-05-31 22:11:37 |
| 7           | 2018       | 2018-07-01     | 100             | 10.77           | 5.04            | 5           | 99.31                  | Nutrition Conscious Eaters                | Consumer reading about healthy eating options.                        | 2016-05-26 14:57:59 | 2018-05-23 11:30:13 |

Now we use this view to answer the question.

**Query:**

```sql
WITH rankings AS (
    SELECT
        month_year,
        interest_id,
        interest_name,
        composition,
        ROW_NUMBER() OVER (PARTITION BY interest_id ORDER BY composition DESC) AS ranking
    FROM filtered_interests
),
top_10 AS (
    SELECT 
        month_year,
        interest_id,
        interest_name,
        composition AS max_comp
    FROM rankings
    WHERE ranking = 1
    ORDER BY max_comp DESC
    LIMIT 10
),
bottom_10 AS (
    SELECT 
        month_year,
        interest_id,
        interest_name,
        composition AS max_comp
    FROM rankings
    WHERE ranking = 1
    ORDER BY max_comp ASC
    LIMIT 10
)
SELECT
    'Top 10' AS category,
    month_year,
    interest_id,
    interest_name,
    max_comp
FROM top_10

UNION ALL 

SELECT 
    'Bottom 10' AS category,
    month_year,
    interest_id,
    interest_name,
    max_comp
FROM bottom_10
```

**Result:**

| **category** | **month_year** | **interest_id** | **interest_name**                 | **max_comp** |
| ------------ | -------------- | --------------- | --------------------------------- | ------------ |
| Top 10       | 2018-12-01     | 21057           | Work Comes First Travelers        | 21.2         |
| Top 10       | 2018-07-01     | 6284            | Gym Equipment Owners              | 18.82        |
| Top 10       | 2018-07-01     | 39              | Furniture Shoppers                | 17.44        |
| Top 10       | 2018-07-01     | 77              | Luxury Retail Shoppers            | 17.19        |
| Top 10       | 2018-10-01     | 12133           | Luxury Boutique Hotel Researchers | 15.15        |
| Top 10       | 2018-12-01     | 5969            | Luxury Bedding Shoppers           | 15.05        |
| Top 10       | 2018-07-01     | 171             | Shoe Shoppers                     | 14.91        |
| Top 10       | 2018-07-01     | 4898            | Cosmetics and Beauty Shoppers     | 14.23        |
| Top 10       | 2018-07-01     | 6286            | Luxury Hotel Guests               | 14.1         |
| Top 10       | 2018-07-01     | 4               | Luxury Retail Researchers         | 13.97        |
| Bottom 10    | 2018-08-01     | 33958           | Astrology Enthusiasts             | 1.88         |
| Bottom 10    | 2018-10-01     | 37412           | Medieval History Enthusiasts      | 1.94         |
| Bottom 10    | 2019-03-01     | 19599           | Dodge Vehicle Shoppers            | 1.97         |
| Bottom 10    | 2018-07-01     | 19635           | Xbox Enthusiasts                  | 2.05         |
| Bottom 10    | 2018-10-01     | 19591           | Camaro Enthusiasts                | 2.08         |
| Bottom 10    | 2019-08-01     | 37421           | Budget Mobile Phone Researchers   | 2.09         |
| Bottom 10    | 2019-01-01     | 42011           | League of Legends Video Game Fans | 2.09         |
| Bottom 10    | 2018-07-01     | 22408           | Super Mario Bros Fans             | 2.12         |
| Bottom 10    | 2019-08-01     | 34085           | Oakland Raiders Fans              | 2.14         |
| Bottom 10    | 2019-02-01     | 36138           | Haunted House Researchers         | 2.18         |

**Note:**

A trade-off has to be made in the choice of window function: 

- If we use `rank()`, then any ties in composition values will yield multiple rows for the same interest, which means the final top 10 or bottom 10 will not be guaranteed to be 10 distinct interests.
- If we use `ROW_NUMBER()`, then we will get 10 distinct interests, but ties will be decided arbitrarily and the corresponding `month_year` will be chosen arbitrarily too. 

In my opinion the second scenario is preferable, so I have decided to use `ROW_NUMBER` instead.

2. *Which 5 interests had the lowest average `ranking` value?*

**Query:**

```sql
WITH avg_rankings AS ( 
    SELECT 
        interest_id,
        interest_name,
        AVG(ranking) AS avg_ranking
    FROM filtered_interests
    GROUP BY interest_id
)
SELECT
    interest_id,
    interest_name,
    ROUND(avg_ranking, 1) AS avg_ranking
FROM avg_rankings
ORDER BY avg_ranking ASC
LIMIT 5
```

**Result:**

| **interest_id** | **interest_name**              | **avg_ranking** |
| --------------- | ------------------------------ | --------------- |
| 41548           | Winter Apparel Shoppers        | 1.0             |
| 42203           | Fitness Activity Tracker Users | 4.1             |
| 115             | Mens Shoe Shoppers             | 5.9             |
| 171             | Shoe Shoppers                  | 9.4             |
| 4               | Luxury Retail Researchers      | 11.9            |

3. *Which 5 interests had the largest standard deviation in their `percentile_ranking` value?*

**Query:**

```sql
WITH stdev_percentile_rankings AS ( 
    SELECT 
        interest_id,
        interest_name,
        stats_stddev(percentile_ranking) AS stdev_percentile_ranking
    FROM filtered_interests
    GROUP BY interest_id
)
SELECT
    interest_id,
    interest_name,
    ROUND(stdev_percentile_ranking, 2) AS stdev_percentile_ranking
FROM stdev_percentile_rankings
ORDER BY stdev_percentile_ranking DESC
LIMIT 5
```

**Result:**

| **interest_id** | **interest_name**                      | **stdev_percentile_ranking** |
| --------------- | -------------------------------------- | ---------------------------- |
| 23              | Techies                                | 30.18                        |
| 20764           | Entertainment Industry Decision Makers | 28.97                        |
| 38992           | Oregon Trip Planners                   | 28.32                        |
| 43546           | Personalized Gift Shoppers             | 26.24                        |
| 10839           | Tampa and St Petersburg Trip Planners  | 25.61                        |

**Note:**

I used the [sqlean extension](https://github.com/nalgeon/sqlean) for the `stats_stddev` standard deviation functionality.

4. *For the 5 interests found in the previous question - what was minimum and maximum `percentile_ranking` values for each interest and its corresponding `year_month` value? Can you describe what is happening for these 5 interests?*

**Query:**

```sql
WITH stdev_percentile_rankings AS ( 
    SELECT 
        interest_id,
        stats_stddev(percentile_ranking) AS stdev_percentile_ranking
    FROM filtered_interests
    GROUP BY interest_id
),
five_largest_stdev AS (
    SELECT
        interest_id,
        ROUND(stdev_percentile_ranking, 2) AS stdev_percentile_ranking
    FROM stdev_percentile_rankings
    ORDER BY stdev_percentile_ranking DESC
    LIMIT 5
),
--rankings (within interests) of the percentile rankings (between interests): pr_rankings
pr_rankings AS (
    SELECT
        month_year,
        interest_id,
        interest_name,
        percentile_ranking,
        ROW_NUMBER() OVER (PARTITION BY interest_id ORDER BY percentile_ranking DESC) AS max_pr_ranking, 
        ROW_NUMBER() OVER (PARTITION BY interest_id ORDER BY percentile_ranking ASC) AS min_pr_ranking
    FROM filtered_interests
),
min_and_max AS (
    SELECT
        CASE 
            WHEN max_pr_ranking = 1 THEN 'Maximum'
            ELSE 'Minimum'
        END AS category,
        month_year,
        interest_id,
        interest_name,
        percentile_ranking
    FROM pr_rankings p
    WHERE 
        max_pr_ranking = 1 OR min_pr_ranking = 1
)
SELECT 
    category,
    month_year,
    interest_id,
    interest_name,
    percentile_ranking
FROM five_largest_stdev
JOIN min_and_max USING (interest_id)
```

**Result:**

| **category** | **month_year** | **interest_id** | **interest_name**                      | **percentile_ranking** |
| ------------ | -------------- | --------------- | -------------------------------------- | ---------------------- |
| Maximum      | 2018-07-01     | 23              | Techies                                | 86.69                  |
| Minimum      | 2019-08-01     | 23              | Techies                                | 7.92                   |
| Maximum      | 2018-07-01     | 20764           | Entertainment Industry Decision Makers | 86.15                  |
| Minimum      | 2019-08-01     | 20764           | Entertainment Industry Decision Makers | 11.23                  |
| Maximum      | 2018-11-01     | 38992           | Oregon Trip Planners                   | 82.44                  |
| Minimum      | 2019-07-01     | 38992           | Oregon Trip Planners                   | 2.2                    |
| Maximum      | 2019-03-01     | 43546           | Personalized Gift Shoppers             | 73.15                  |
| Minimum      | 2019-06-01     | 43546           | Personalized Gift Shoppers             | 5.7                    |
| Maximum      | 2018-07-01     | 10839           | Tampa and St Petersburg Trip Planners  | 75.03                  |
| Minimum      | 2019-03-01     | 10839           | Tampa and St Petersburg Trip Planners  | 4.84                   |

**Answer:**

For all these interests, the minimum takes place months after the maximum. In other words, these interests have fallen off significantly in terms of percentile rankings.

The reason we honed in on these 5 interests in particular is because of their maximal standard deviation.

5. *How would you describe our customers in this segment based off their composition and ranking values? What sort of products or services should we show to these customers and what should we avoid?*

**Answer:**

Let’s look back at our answers from the previous four questions and look for customer patterns:

1. The top 10 for maximal composition values is filled mostly with luxury products, cosmetics, travel and otherwise expensive products. The bottom 10 is filled mostly with sports, vehicles and video gaming related interests.
2. The top 5 best average rankings are mostly clothing/shoes related, with luxury/fitness interests too. Interestingly, the “Winter Apparel Shoppers” interest has average rank 1: it’s always the highest ranked interest for any month it appears in. 
3. The five interests with the highest standard deviation have all seen a substantial decline in percentile ranking over the last period. These interests consist of US-based trip planners, tech/entertainment and personalized gift shoppers. 

All in all, I would suggest showing the customers more luxury, clothing/shoes and travel items (though not necessarily travel to places in the US). In contrast, I would avoid showing products that are more techy such as video games or sports/vehicle related.

### Index Analysis

*The `index_value` is a measure which can be used to reverse calculate the average composition for Fresh Segments’ clients.*
*Average composition can be calculated by dividing the `composition` column by the `index_value` column rounded to 2 decimal places.*

**Note:**

For these next 5 questions, we choose to once again use the full `interests` joined table, rather than the `filtered_interests`. This is because the next 5 questions look for metrics on a monthly basis, rather than metrics over the entire dataset like before (such as average rank, minimal and maximal compositions of interests etc.). On a monthly basis, it does not matter if an interest only appears in a couple months: for the months that they are in, they can contribute to the average and for others they just do not.

Technically question 2 has no monthly metric, but it looks for the highest frequency interest from the answer of question 1. We will be building that answer from the answer from question 1, and that answer will use the full `interests` table, so question 2 will use that table too. 

1. *What is the top 10 interests by the average composition for each month?*

**Query:**

```sql
WITH avg_comps AS (
    SELECT
        month_year,
        interest_id,
        interest_name,
        ROUND(
            composition / index_value,
            2
        ) AS avg_comp
    FROM interests
    WHERE month_year IS NOT NULL
),
rankings AS (
    SELECT 
        month_year,
        interest_id,
        interest_name,
        rank() OVER (PARTITION BY month_year ORDER BY avg_comp DESC) AS avg_comp_ranking,
        avg_comp
    FROM avg_comps
)
SELECT 
    month_year,
    interest_id,
    interest_name,
    avg_comp_ranking,
    avg_comp
FROM rankings
WHERE avg_comp_ranking <= 10
```

**Result (first 10 rows):**

| **month_year** | **interest_id** | **interest_name**             | **ranking** | **avg_comp** |
| -------------- | --------------- | ----------------------------- | ----------- | ------------ |
| 2018-07-01     | 6324            | Las Vegas Trip Planners       | 1           | 7.36         |
| 2018-07-01     | 6284            | Gym Equipment Owners          | 2           | 6.94         |
| 2018-07-01     | 4898            | Cosmetics and Beauty Shoppers | 3           | 6.78         |
| 2018-07-01     | 77              | Luxury Retail Shoppers        | 4           | 6.61         |
| 2018-07-01     | 39              | Furniture Shoppers            | 5           | 6.51         |
| 2018-07-01     | 18619           | Asian Food Enthusiasts        | 6           | 6.1          |
| 2018-07-01     | 6208            | Recently Retired Individuals  | 7           | 5.72         |
| 2018-07-01     | 21060           | Family Adventures Travelers   | 8           | 4.85         |
| 2018-07-01     | 21057           | Work Comes First Travelers    | 9           | 4.8          |
| 2018-07-01     | 82              | HDTV Researchers              | 10          | 4.71         |

**Note:**

- These top 10 interests are *only* for the interests that appear in this clients’ database. There might be other interests that Fresh Segments tracks with a higher average composition for some months that are simply missing in this dataset. 
- We save this resulting table as a view called `monthly_avg_comp_rankings` for later use.

2. *For all of these top 10 interests - which interest appears the most often?*

**Query:**

```sql
WITH interest_freqs AS (
    SELECT
        interest_id,
        interest_name,
        COUNT(*) AS interest_freq
    FROM monthly_avg_comp_rankings
    GROUP BY interest_id
),
freq_rankings AS (
    SELECT 
        interest_id,
        interest_name,
        rank() OVER (ORDER BY interest_freq DESC) AS freq_ranking,
        interest_freq
    FROM interest_freqs
)
SELECT 
    interest_id,
    interest_name,
    interest_freq AS interest_frequency
FROM freq_rankings
WHERE freq_ranking = 1
```

**Result:**

| **interest_id** | **interest_name**        | **interest_frequency** |
| --------------- | ------------------------ | ---------------------- |
| 5969            | Luxury Bedding Shoppers  | 10                     |
| 6065            | Solar Energy Researchers | 10                     |
| 7541            | Alabama Trip Planners    | 10                     |

3. *What is the average of the average composition for the top 10 interests for each month?*

**Query:**

```sql
SELECT
    month_year,
    ROUND(
        avg(avg_comp),
        2
    ) AS average_avg_comp
FROM monthly_avg_comp_rankings
GROUP BY month_year
```

**Result:**

| **month_year** | **average_avg_comp** |
| -------------- | -------------------- |
| 2018-07-01     | 6.04                 |
| 2018-08-01     | 5.95                 |
| 2018-09-01     | 6.9                  |
| 2018-10-01     | 7.07                 |
| 2018-11-01     | 6.62                 |
| 2018-12-01     | 6.65                 |
| 2019-01-01     | 6.32                 |
| 2019-02-01     | 6.58                 |
| 2019-03-01     | 6.12                 |
| 2019-04-01     | 5.75                 |
| 2019-05-01     | 3.54                 |
| 2019-06-01     | 2.43                 |
| 2019-07-01     | 2.76                 |
| 2019-08-01     | 2.63                 |

**Note:**

The average composition is an average over *clients*, and exists for each interest. The average of the average composition is an average over *interests* and their average composition metric (which is just another number). We only average over the top 10 interests per month here.

4. *What is the 3 month rolling average of the max average composition value from September 2018 to August 2019 and include the previous top ranking interests in the same output shown below.*

*Required output for question 4:*

| **month_year** | **interest_name**             | **max_index_composition** | **3_month_moving_avg** | **1_month_ago**                   | **2_months_ago**                  |
| -------------- | ----------------------------- | ------------------------- | ---------------------- | --------------------------------- | --------------------------------- |
| 2018-09-01     | Work Comes First Travelers    | 8.26                      | 7.61                   | Las Vegas Trip Planners: 7.21     | Las Vegas Trip Planners: 7.36     |
| 2018-10-01     | Work Comes First Travelers    | 9.14                      | 8.20                   | Work Comes First Travelers: 8.26  | Las Vegas Trip Planners: 7.21     |
| 2018-11-01     | Work Comes First Travelers    | 8.28                      | 8.56                   | Work Comes First Travelers: 9.14  | Work Comes First Travelers: 8.26  |
| 2018-12-01     | Work Comes First Travelers    | 8.31                      | 8.58                   | Work Comes First Travelers: 8.28  | Work Comes First Travelers: 9.14  |
| 2019-01-01     | Work Comes First Travelers    | 7.66                      | 8.08                   | Work Comes First Travelers: 8.31  | Work Comes First Travelers: 8.28  |
| 2019-02-01     | Work Comes First Travelers    | 7.66                      | 7.88                   | Work Comes First Travelers: 7.66  | Work Comes First Travelers: 8.31  |
| 2019-03-01     | Alabama Trip Planners         | 6.54                      | 7.29                   | Work Comes First Travelers: 7.66  | Work Comes First Travelers: 7.66  |
| 2019-04-01     | Solar Energy Researchers      | 6.28                      | 6.83                   | Alabama Trip Planners: 6.54       | Work Comes First Travelers: 7.66  |
| 2019-05-01     | Readers of Honduran Content   | 4.41                      | 5.74                   | Solar Energy Researchers: 6.28    | Alabama Trip Planners: 6.54       |
| 2019-06-01     | Las Vegas Trip Planners       | 2.77                      | 4.49                   | Readers of Honduran Content: 4.41 | Solar Energy Researchers: 6.28    |
| 2019-07-01     | Las Vegas Trip Planners       | 2.82                      | 3.33                   | Las Vegas Trip Planners: 2.77     | Readers of Honduran Content: 4.41 |
| 2019-08-01     | Cosmetics and Beauty Shoppers | 2.73                      | 2.77                   | Las Vegas Trip Planners: 2.82     | Las Vegas Trip Planners: 2.77     |

**Query:**

```sql
WITH avg_comp_rankings AS (
    SELECT 
        month_year,
        interest_name,
        rank() OVER (PARTITION BY month_year ORDER BY avg_comp DESC) AS avg_comp_ranking,
        avg_comp
    FROM monthly_avg_comp_rankings
),
max_avg_comps AS (
    SELECT
        month_year,
        interest_name,
        avg_comp,
        lag(avg_comp) OVER (ORDER BY month_year ASC) AS prev_avg_comp,
        lag(interest_name) OVER (ORDER BY month_year ASC) AS prev_interest_name,
        lag(avg_comp, 2) OVER (ORDER BY month_year ASC) AS second_prev_avg_comp,
        lag(interest_name, 2) OVER (ORDER BY month_year ASC) AS second_prev_interest_name
    FROM avg_comp_rankings
    WHERE avg_comp_ranking = 1
)
SELECT 
    month_year,
    interest_name,
    avg_comp AS max_index_composition,
    ROUND(
        CAST(
            avg_comp + prev_avg_comp + second_prev_avg_comp AS REAL
        ) / 3.0,
        2
    ) AS "3_month_moving_avg",
    concat(prev_interest_name, ': ', prev_avg_comp) AS "1_month_ago",
    concat(second_prev_interest_name, ': ', second_prev_avg_comp) AS "2_months_ago"
FROM max_avg_comps
WHERE 
    month_year >= '2018-09-01'
    AND month_year <= '2019-08-01'
```

**Result:**

| **month_year** | **interest_name**             | **max_index_composition** | **3_month_moving_avg** | **1_month_ago**                   | **2_months_ago**                  |
| -------------- | ----------------------------- | ------------------------- | ---------------------- | --------------------------------- | --------------------------------- |
| 2018-09-01     | Work Comes First Travelers    | 8.26                      | 7.61                   | Las Vegas Trip Planners: 7.21     | Las Vegas Trip Planners: 7.36     |
| 2018-10-01     | Work Comes First Travelers    | 9.14                      | 8.2                    | Work Comes First Travelers: 8.26  | Las Vegas Trip Planners: 7.21     |
| 2018-11-01     | Work Comes First Travelers    | 8.28                      | 8.56                   | Work Comes First Travelers: 9.14  | Work Comes First Travelers: 8.26  |
| 2018-12-01     | Work Comes First Travelers    | 8.31                      | 8.58                   | Work Comes First Travelers: 8.28  | Work Comes First Travelers: 9.14  |
| 2019-01-01     | Work Comes First Travelers    | 7.66                      | 8.08                   | Work Comes First Travelers: 8.31  | Work Comes First Travelers: 8.28  |
| 2019-02-01     | Work Comes First Travelers    | 7.66                      | 7.88                   | Work Comes First Travelers: 7.66  | Work Comes First Travelers: 8.31  |
| 2019-03-01     | Alabama Trip Planners         | 6.54                      | 7.29                   | Work Comes First Travelers: 7.66  | Work Comes First Travelers: 7.66  |
| 2019-04-01     | Solar Energy Researchers      | 6.28                      | 6.83                   | Alabama Trip Planners: 6.54       | Work Comes First Travelers: 7.66  |
| 2019-05-01     | Readers of Honduran Content   | 4.41                      | 5.74                   | Solar Energy Researchers: 6.28    | Alabama Trip Planners: 6.54       |
| 2019-06-01     | Las Vegas Trip Planners       | 2.77                      | 4.49                   | Readers of Honduran Content: 4.41 | Solar Energy Researchers: 6.28    |
| 2019-07-01     | Las Vegas Trip Planners       | 2.82                      | 3.33                   | Las Vegas Trip Planners: 2.77     | Readers of Honduran Content: 4.41 |
| 2019-08-01     | Cosmetics and Beauty Shoppers | 2.73                      | 2.77                   | Las Vegas Trip Planners: 2.82     | Las Vegas Trip Planners: 2.77     |

**Note:**

- One column is named `max_index_composition` in the required result, even though what we are calculating is actually the maximum average composition. I’m not really sure where that other naming comes from.
- My output’s second row has a 3 month moving average of 8.2 rather than the intended notation of 8.20 with the extra zero. In SQLite I could output the values as text instead and force the extra zero to be there, but I did not find that to be important enough to change the query for.
- The usage of `lag()` here implicitly assumes that there are no month gaps in our data: it looks at the previous maximal average composition, not the previous month. In our data though, every month between September 2018 and August 2019 is included so this is not a problem. 

5. *Provide a possible reason why the max average composition might change from month to month? Could it signal something is not quite right with the overall business model for Fresh Segments?*

**Answer:**

One reason that the maximum average composition might be changing (becoming smaller) each month is that there is a more general trend online in customers from all client businesses interacting less with ads. For example, ad blockers could be more on the rise, and so less customers interact with ads, and hence the composition values (and with that the maximum average composition values) all go down.

Fresh Segments aims to give business insight on client customer list information. Global trends like these are out of control for Fresh Segments and so it is not necessarily a sign that there is something wrong with their business model. 

If we had data on the same metrics for other digital marketing agencies like Fresh Segments, and their numbers were substantially higher, then one could start looking more closely at the overall business model for Fresh Segments. 

