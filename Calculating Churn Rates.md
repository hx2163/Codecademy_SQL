## Calculating Churn Rates

Four months into launching Codeflix, management asks you to look into subscription churn rates. It’s early on in the business and people are excited to know how the company is doing.

The marketing department is particularly interested in how the churn compares between two segments of users. They provide you with a dataset containing subscription data for users who were acquired through two distinct channels.

The dataset provided to you contains one SQL table, subscriptions. Within the table, there are 4 columns:
- id - the subscription id
- subscription_start - the start date of the subscription
- subscription_end - the end date of the subscription
- segment - this identifies which segment the subscription owner belongs to

Codeflix requires a minimum subscription length of 31 days, so a user can never start and end their subscription in the same month.

### Get familiar with the data


1. Take a look at the first 100 rows of data in the subscriptions table. How many different segments do you see?

```mysql
 SELECT *
 FROM subscriptions
 LIMIT 10;
 ```
![churn01](images/churn01.png)

2. Determine the range of months of data provided. Which months will you be able to calculate churn for?

```mysql
 SELECT MIN(subscription_start),
 MAX(subscription_start)
 FROM subscriptions;
```
![churn02](images/churn02.png)

### Calculate churn rate for each segment

3. You’ll be calculating the churn rate for both segments (87 and 30) over the first 3 months of 2017 (you can’t calculate it for December, since there are no subscription_end values yet). To get started, create a temporary table of months.

```mysql
WITH months AS (
  SELECT 
    '2017-01-01' AS first_day, 
    '2017-01-31' AS last_day 
  UNION 
  SELECT 
    '2017-02-01' AS first_day, 
    '2017-02-28' AS last_day 
  UNION 
  SELECT 
    '2017-03-01' AS first_day, 
    '2017-03-31' AS last_day
)
SELECT * FROM months;
```

![churn03](images/churn03.png)

4. Create a temporary table, cross_join, from subscriptions and your months. Be sure to SELECT every column.

```mysql
WITH months AS (
  SELECT 
    '2017-01-01' AS first_day, 
    '2017-01-31' AS last_day 
  UNION 
  SELECT 
    '2017-02-01' AS first_day, 
    '2017-02-28' AS last_day 
  UNION 
  SELECT 
    '2017-03-01' AS first_day, 
    '2017-03-31' AS last_day
),
cross_join AS(
SELECT *
FROM subscriptions
CROSS JOIN months)
SELECT * FROM cross_join
LIMIT 10;
```
![churn04](images/churn04.png)

5.
Create a temporary table, status, from the cross_join table you created. This table should contain:

- id selected from cross_join
- month as an alias of first_day
- is_active_87 created using a CASE WHEN to find any users from segment 87 who existed prior to the beginning of the month. This is 1 if true and 0 otherwise.
- is_active_30 created using a CASE WHEN to find any users from segment 30 who existed prior to the beginning of the month. This is 1 if true and 0 otherwise.

```mysql
WITH months AS (
  SELECT 
    '2017-01-01' AS first_day, 
    '2017-01-31' AS last_day 
  UNION 
  SELECT 
    '2017-02-01' AS first_day, 
    '2017-02-28' AS last_day 
  UNION 
  SELECT 
    '2017-03-01' AS first_day, 
    '2017-03-31' AS last_day
),
cross_join AS(
SELECT *
FROM subscriptions
CROSS JOIN months
),
status AS(
SELECT id,
first_day AS month,
CASE WHEN
  (segment = 87) 
    AND subscription_start < first_day 
    AND (subscription_end > first_day 
         OR subscription_end IS NULL) THEN 1
  ELSE 0
  END AS is_active_87,
  CASE WHEN
(segment = 30)
    AND subscription_start < first_day
    AND (subscription_end > first_day 
         OR subscription_end IS NULL) THEN 1
  ELSE 0
  END AS is_active_30
  FROM cross_join)
SELECT * FROM status
LIMIT 10;
```
![churn05](images/churn05.png)

6. Add an is_canceled_87 and an is_canceled_30 column to the status temporary table. This should be 1 if the subscription is canceled during the month and 0 otherwise.

```mysql
WITH months AS (
  SELECT 
    '2017-01-01' AS first_day, 
    '2017-01-31' AS last_day 
  UNION 
  SELECT 
    '2017-02-01' AS first_day, 
    '2017-02-28' AS last_day 
  UNION 
  SELECT 
    '2017-03-01' AS first_day, 
    '2017-03-31' AS last_day
),
cross_join AS(
SELECT *
FROM subscriptions
CROSS JOIN months
),
status AS(
SELECT id,
first_day AS month,
CASE WHEN
  (segment = 87) AND subscription_start < first_day AND (subscription_end > first_day or subscription_end IS NULL) THEN 1
  ELSE 0
  END AS is_active_87,
CASE WHEN
  (segment = 30) AND subscription_start < first_day AND (subscription_end > first_day or subscription_end IS NULL) THEN 1
  ELSE 0
  END AS is_active_30,
CASE WHEN
  (segment = 87) AND (subscription_end BETWEEN first_day AND last_day) THEN 1
      ELSE 0
  END AS is_canceled_87,
CASE WHEN
  (segment = 30) AND (subscription_end BETWEEN first_day AND last_day) THEN 1
  ELSE 0
  END AS is_canceled_30
FROM cross_join
)
SELECT * FROM status
LIMIT 10;
```
![churn06](images/churn06.png)

7.
Create a status_aggregate temporary table that is a SUM of the active and canceled subscriptions for each segment, for each month.

The resulting columns should be:
- sum_active_87
- sum_active_30
- sum_canceled_87
- sum_canceled_30

```mysql
WITH months AS (
  SELECT 
    '2017-01-01' AS first_day, 
    '2017-01-31' AS last_day 
  UNION 
  SELECT 
    '2017-02-01' AS first_day, 
    '2017-02-28' AS last_day 
  UNION 
  SELECT 
    '2017-03-01' AS first_day, 
    '2017-03-31' AS last_day
),
cross_join AS(
SELECT *
FROM subscriptions
CROSS JOIN months
),
status AS(
SELECT id,
first_day AS month,
CASE WHEN
  (segment = 87) AND subscription_start < first_day AND (subscription_end > first_day or subscription_end IS NULL) THEN 1
  ELSE 0
  END AS is_active_87,
CASE WHEN
  (segment = 30) AND subscription_start < first_day AND (subscription_end > first_day or subscription_end IS NULL) THEN 1
  ELSE 0
  END AS is_active_30,
CASE WHEN
  (segment = 87) AND (subscription_end BETWEEN first_day AND last_day) THEN 1
      ELSE 0
  END AS is_canceled_87,
CASE WHEN
  (segment = 30) AND (subscription_end BETWEEN first_day AND last_day) THEN 1
  ELSE 0
  END AS is_canceled_30
FROM cross_join),
status_aggregate AS (
  SELECT 
    month, 
    SUM(is_active_87) AS sum_active_87, 
    SUM(is_canceled_87) AS sum_canceled_87,
    SUM(is_active_30) AS sum_active_30, 
    SUM(is_canceled_30) AS sum_canceled_30
  FROM status 
  GROUP BY month
)
SELECT * FROM status_aggregate
LIMIT 10;
```
![churn07](images/churn07.png)

8. Calculate the churn rates for the two segments over the three month period. Which segment has a lower churn rate?

```mysql
WITH months AS (
  SELECT 
    '2017-01-01' AS first_day, 
    '2017-01-31' AS last_day 
  UNION 
  SELECT 
    '2017-02-01' AS first_day, 
    '2017-02-28' AS last_day 
  UNION 
  SELECT 
    '2017-03-01' AS first_day, 
    '2017-03-31' AS last_day
),
cross_join AS(
SELECT *
FROM subscriptions
CROSS JOIN months
),
status AS(
SELECT id,
first_day AS month,
CASE WHEN
  (segment = 87) AND subscription_start < first_day AND (subscription_end > first_day or subscription_end IS NULL) THEN 1
  ELSE 0
  END AS is_active_87,
CASE WHEN
  (segment = 30) AND subscription_start < first_day AND (subscription_end > first_day or subscription_end IS NULL) THEN 1
  ELSE 0
  END AS is_active_30,
CASE WHEN
  (segment = 87) AND (subscription_end BETWEEN first_day AND last_day) THEN 1
      ELSE 0
  END AS is_canceled_87,
CASE WHEN
  (segment = 30) AND (subscription_end BETWEEN first_day AND last_day) THEN 1
  ELSE 0
  END AS is_canceled_30
FROM cross_join),
status_aggregate AS (
  SELECT 
    month, 
    SUM(is_active_87) AS sum_active_87, 
    SUM(is_canceled_87) AS sum_canceled_87,
    SUM(is_active_30) AS sum_active_30, 
    SUM(is_canceled_30) AS sum_canceled_30
  FROM status 
  GROUP BY month
)
SELECT month,
1.0 * sum_canceled_87 / sum_active_87 AS churn_87,
1.0 * sum_canceled_30 / sum_active_30 AS churn_30
FROM status_aggregate;
```
![churn08](images/churn08.png)

9. How would you modify this code to support a large number of segments?

```mysql
WITH months AS (
  SELECT 
    '2017-01-01' AS first_day, 
    '2017-01-31' AS last_day 
  UNION 
  SELECT 
    '2017-02-01' AS first_day, 
    '2017-02-28' AS last_day 
  UNION 
  SELECT 
    '2017-03-01' AS first_day, 
    '2017-03-31' AS last_day
),
cross_join AS(
SELECT *
FROM subscriptions
CROSS JOIN months
),
status AS(
SELECT id,
first_day AS month,
  segment,
CASE WHEN
  subscription_start < first_day AND (subscription_end > first_day or subscription_end IS NULL) THEN 1
  ELSE 0
  END AS is_active,
CASE WHEN
  (subscription_end BETWEEN first_day AND last_day) THEN 1
      ELSE 0
  END AS is_canceled
FROM cross_join),
status_aggregate AS (
  SELECT 
    month,
  segment,
    SUM(is_active) AS sum_active, 
    SUM(is_canceled) AS sum_canceled
  FROM status 
  GROUP BY month, segment
  ORDER BY segment
)
SELECT month,
segment,
1.0 * sum_canceled / sum_active AS churn
FROM status_aggregate;
```

![churn09](images/churn09.png)
