# Funnel-Analysis-with-SQL
# Funnel Analysis 

**Funnel Analysis** is a marketing model which illustrates the theoretical customer journey towards the purchase of a product or service. Oftentimes, we want to track how many users complete a series of steps and know which steps have the most number of users giving up. 

## 1. Get familar with the tables
#### Four tables
There are 1986 users participating the survey--> 1000 users participate the quiz --> 750 users have the home-try-on --> 495 users make the purchae
```sql
/*
SELECT * FROM survey LIMIT 5;
SELECT * FROM quiz LIMIT 5;
SELECT * FROM home_try_on LIMIT 5;
SELECT * FROM purchase LIMIT 5;
*/
```
#### 1). survey table
question	|user_id	|response|
----------|---------|--------|
|1. What are you looking for?	|005e7f99-d48c-4fce-b605-10506c85aaf7	|Women's Styles|
|3. What's your fit? |005e7f99-d48c-4fce-b605-10506c85aaf7	|Medium|
|4. Which shapes do you like?	|00a556ed-f13e-4c67-8704-27e3573684cd	|Round|
|5. Which colors do you like?	|00a556ed-f13e-4c67-8704-27e3573684cd	|Two-Tone|
|6. What are you looking for?	|00a556ed-f13e-4c67-8704-27e3573684cd	|I'm not sure. Let's skip it.|


#### 2). quiz table
user_id	|style	|fit	|shape	|color|
--------|-------|-----|-------|-----|
|4e8118dc|-bb3d-49bf-85fc-cca8d83232ac|	Women's Styles|	Medium|	Rectangular	Tortoise|
|291f1cca|-e507-48be-b063-002b14906468|	Women's Styles|	Narrow|	Round	Black|
|75122300|-0736-4087-b6d8-c0c5373a1a04|	Women's Styles|	Wide|	Rectangular	Two-Tone|
|75bc6ebd|-40cd-4e1d-a301-27ddd93b12e2|	Women's Styles|	Narrow|	Square	Two-Tone|
|ce965c4d|-7a2b-4db6-9847-601747fa7812|	Women's Styles|	Wide|	Rectangular	Black|

#### 3). home-try-on table
user_id	|number_of_pairs	|address|
--------|-----------------|-------|
|d8addd87-3217-4429-9a01-d56d68111da7|	5 pairs|	145 New York 9a|
|f52b07c8-abe4-4f4a-9d39-ba9fc9a184cc|	5 pairs|	383 Madison Ave|
|8ba0d2d5-1a31-403e-9fa5-79540f8477f9|	5 pairs|	287 Pell St|
|4e71850e-8bbf-4e6b-accc-49a7bb46c586|	3 pairs|	347 Madison Square N|
|3bc8f97f-2336-4dab-bd86-e391609dab97|	5 pairs|	182 Cornelia St|


#### 4). quiz table
user_id	|product_id	|style	|model_name	|color|	price|
--------|-----------|-------|-----------|-----|------|
|00a9dd17-36c8-430c-9d76-df49d4197dcf	|8	|Women's Styles|	Lucy	|Jet Black	|150|
|00e15fe0-c86f-4818-9c63-3422211baa97	|7	|Women's Styles|	Lucy	|Elderflower Crystal	|150|
|017506f7-aba1-4b9d-8b7b-f4426e71b8ca	|4	|Men's Styles|	Dawes	Jet |Black	|150|
|0176bfb3-9c51-4b1c-b593-87edab3c54cb	|10	|Women's Styles|	Eugene Narrow	|Rosewood Tortoise	|95|
|01fdf106-f73c-4d3f-a036-2f3e2ab1ce06	|8	|Women's Styles|	Lucy|Jet Black	|150|


## 2. Steps of the Funnel Analysis

### Step 1 : Calculate the number of users complete each steps
```sql
WITH step_1 AS (
SELECT 
CASE 
  WHEN question LIKE '1.%' THEN 1
  WHEN question LIKE '2.%' THEN 2
  WHEN question LIKE '3.%' THEN 3
  WHEN question LIKE '4.%' THEN 4
  WHEN question LIKE '5.%' THEN 5
  END AS Question,
COUNT (DISTINCT user_id) AS counts 
  FROM survey
GROUP BY question),

step_2 AS (

SELECT tb1.Question, 
  tb1.counts AS count_1, 
   tb2.counts AS count_2
FROM step_1 as tb1 
LEFT JOIN step_1 as tb2
ON tb1.Question - tb2.Question  = 1), 

step_3 AS (
SELECT Question AS Question, count_1 AS responses,
IFNULL(ROUND(100.0 * count_1/count_2, 2), 100) AS 'Percent Completing this Question'
FROM step_2)
SELECT * FROM step_3
;
```

### Output
Question_Number|	Response_Number|	Percent Completing this Question (%)|
---------|-------|-----|
|1	|500|	100|
|2	|475|	95.0|
|3	|380|	80.0|
|4	|361|	95.0|
|5	|270|	74.0|

### Step 2: Use left join function to combine the tables 
### Functions used: 
- left join 
- substr()

```sql

SELECT SUBSTR(quiz.user_id, 1, 8) as user_id,
CASE WHEN home_try_on.user_id IS NOT NULL THEN 'True' ELSE 'False' END AS 'is_home_try_on',
CASE 
  WHEN home_try_on.number_of_pairs LIKE '3%' THEN 3 
  WHEN home_try_on.number_of_pairs LIKE '5%' THEN 5
  ELSE 'NULL'
END AS number_of_pairs,
CASE WHEN purchase.user_id IS NOT NULL THEN 'True' ELSE 'False' END AS 'is_purchase'
FROM quiz
LEFT JOIN home_try_on
ON quiz.user_id = home_try_on.user_id
LEFT JOIN purchase
ON home_try_on.user_id = purchase.user_id
LIMIT 10;
```
### Output

user_id|	is_home_try_on|	number_of_pairs|	is_purchase|
-----|----|-------|-----|
4e8118dc|	True|	3|	False|
291f1cca|	True|	3|	True|
75122300|	False|	NULL|	False|
75bc6ebd|	True|	5|	False|
ce965c4d|	True|	3|	True|

### Step 3: Calculate the percentage of users from the previous step to the current step
```sql
WITH prev AS (
SELECT SUBSTR(quiz.user_id, 1, 8) as user_id,
CASE WHEN home_try_on.user_id THEN 1 ELSE 0 END AS 'is_home_try_on',
CASE 
  WHEN home_try_on.number_of_pairs LIKE '3%' THEN 3 
  WHEN home_try_on.number_of_pairs LIKE '5%' THEN 5
END AS number_of_pairs,
CASE WHEN purchase.user_id  THEN 1 ELSE 0 END AS 'is_purchase'
FROM quiz
LEFT JOIN home_try_on
ON quiz.user_id = home_try_on.user_id
LEFT JOIN purchase
ON home_try_on.user_id = purchase.user_id),

aggegate AS (
SELECT number_of_pairs, 
COUNT(DISTINCT user_id) AS num_users, 
SUM(is_home_try_on) AS num_home_try_on, 
SUM(is_purchase) AS num_purchase
FROM prev 
WHERE number_of_pairs IS NOT NULL
GROUP BY number_of_pairs)

SELECT number_of_pairs,
ROUND(100.0 * num_home_try_on/num_users, 2) AS quiz_to_home_try_pct,
ROUND(100.0 *num_purchase/num_home_try_on, 2) AS home_try_to_purchase_pct
FROM aggegate;
```

### Output

|number_of_pairs|	num_users|	num_purchase|	quiz_to_home_try_pct(%)|home_try_to_purchase_pct(%)
|-|-----|-----|-------|------|
|3|	379|	123|	62.27|	52.12|
|5|	371|	175|	59.57|	79.19|


### Conclusion 
From the process above, a slightly less users choose the number_of_pairs of 5, but if the users choose number_of_pairs of 5, they are more likely to make the purchase. Thus, we can take actions to increase the number of peole choosing number_of_pairs of 5 to increase purchase rate.
