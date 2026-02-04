# Theme Park Analysis

### by [Vincent Perez](https://www.linkedin.com/in/thevinceperez/)

## Table of Contents
- [Business Problem & Stakeholders](#business-problem)
- [Database Info](#the-database-and-schema)
- [Exploratory Data Analysis](#exploratory-data-analysis)
- [Feature Engineering](#FEATURE-ENGINEERING)
- [Operations Director Analysis & Visuals](#analysis-for-operations-director)
- [Marketing Director Analysis & Visuals](#analysis-for-marketing-director)
- [Final Recommendations](#final-recommendations-for-next-quarter)
- [Ethics and Biases](#ethics-and-biases)
- [Repo Navigation](#repo-navigation)

# Business Problem:

### My client, Supernova Theme Park, has hired me to develop a cross-departmental plan for the Operations and Marketing teams. They want to strategize for the next quarter to improve operational efficiency, guest experience, and marketing effectiveness. They have provided me with Supernova Theme Park’s data to analyze and generate insights.

## The stakeholders are:

### <ins>Primary Stakeholder</ins>
***Park General Manager's*** concerns are: </br>
- Unhappy with previous two quarters. </br>
- Upset with fluctuating revenue streams. </br>
- Uneven guest satisfaction scores. </br>

### <ins>Supporting Stakeholders</ins> </br>
**Operations Director's** concerns include: </br>
- Inconsistent ride availability due to maintenance. </br>
- Long wait times for popular attractions. </br>

**Marketing Director's** concerns: </br>
- Early campaigns say discount packages drive up attendance of price-sensitive guests.

# So Let's Approach The Scenario!

## The Database and Schema:

### We are working with a Star Schema 
Simply put, a __star schema__ is when a central fact table references multiple dimensions tables. </br>

<img width="698" height="326" alt="Star-schema" src="https://github.com/user-attachments/assets/5aaafdc8-d5d0-4ccf-91fc-d0cd25e9fba2" />

A _dimension table_ contains all unique instances, is very verbose, and is usually grouped. </br>

A fact table refers to real-world events, contains measures linked to foreign keys associated with the primary keys in dimension tables, and sometimes includes date and time stamps. </br>

Some benefits are:
- Easier merging of tables when a fact table is shared across dimension tables
- Improved readability

### The Database has 7 Tables

__4 Dimension Tables:__ </br>
- ```dim_attraction``` : includes all unique attractions at Supernova
- ```dim_date``` : includes all unique dates of the database
- ```dim_guest``` : includes all guests and their contact information
- ```dim_ticket``` : includes all ticket types and their pricing </br>
__3 Fact Tables:__ </br>
- ```fact_purchases``` : all instances of guest purchases made during a visit at Supernova
- ```fact_ride_events``` : all details of rides during a visit
- ```fact_visits``` : all separate visits a guest has done

## Exploratory Data Analysis

To find insights, I aggregated on important columns for the stakeholders. </br>

To tackle the **Park General Manager's** concern of rating scores, I checked for satisfaction_ratings. </br>

I calculated the average satisfaction_rating per attraction_name by using the dim_attraction table and joining the fact_ride_events table. </br>

<img width="286" height="211" alt="Screenshot 2025-08-22 at 3 01 46 PM" src="https://github.com/user-attachments/assets/fd4b6929-ea10-4c9f-9fa4-729c7177c905" /> </br>

This aggregation of satisfaction_rating provided insights into how customers felt about the attractions. </br>

Then, I checked the frequency of total guests (47) that utilized promotional codes.

<img width="205" height="125" alt="Screenshot 2025-08-22 at 3 29 07 PM" src="https://github.com/user-attachments/assets/2f0592db-7db5-4fd7-be3f-e1e987466e42" />

Of the 40 guests with promo code data, most of them (33) used promotional offers. This supports the **Marketing Director's** claim of "Promotional Codes drive attendance".

## FEATURE ENGINEERING

I engineered a new column, satisfaction_score, to categorize satisfaction_rating into bins where:
- satisfaction_rating = 5 -> 'Satisfied'
- satisfaction_rating = 4 -> 'Moderately Satisfied'
- satisfaction_rating < 4 -> 'Unsatisfied'

```
ALTER TABLE fact_ride_events ADD COLUMN satisfied_score

UPDATE fact_ride_events
SET satisfied_score =
    CASE WHEN satisfaction_rating = 5 THEN 'Satisfied'
         WHEN satisfaction_rating = 4 THEN 'Moderately Satisfied'
         WHEN satisfaction_rating < 4 THEN 'Unsatisfied'
         ELSE null
         END
```

This allows me to count rides by satisfaction categories and provide insights into customer dissatisfaction. For example, we can identify which rides are negatively or positively affecting overall satisfaction. </br>
This can also guide future outreach to guests who are unsatisfied or moderately satisfied for feedback. </br>


<img width="289" height="185" alt="Screenshot 2025-08-22 at 3 55 49 PM" src="https://github.com/user-attachments/assets/68f83ab2-9648-4224-99f4-6462bf0c4b49" />


I used the new feature to see the top 5 attractions most frequently rated as unsatisfied or moderately satisfied. Of these, two belonged to the 'Water' category. </br>

# __Insights and Recommendations__

## Analysis for Operations Director

The original aggregation of satisfaction_rating showed me a lot about how customers felt about the Water attractions. </br>
Interestingly, the Water category had two of the lowest satisfaction_rating, so I dug deeper into the categories and examined waiting times. </br>

```
SELECT da.category, COUNT(fre.satisfaction_rating) as count_of_visits,
       ROUND(AVG(fre.wait_minutes),2) as average_wait,
       ROUND(AVG(fre.satisfaction_rating),2) as average_rating
FROM fact_ride_events fre
JOIN dim_attraction da ON fre.attraction_id = da.attraction_id
WHERE fre.wait_minutes IS NOT NULL
GROUP BY da.category
```
<img width="414" height="151" alt="Screenshot 2025-08-22 at 3 19 08 PM" src="https://github.com/user-attachments/assets/77e64cf4-dec9-4860-85dd-170cc7c00e4a" />

Not only did the Water category have the highest number of visits among all ride categories (25), it also had the highest average wait time per ride (49.12 minutes) and lowest average rating (2.72). This could be a concern for the **Operations Director.** </br>

### Visuals:

![bar chart for average rating per category](/figures/ratepercat.png "Average Rating")
</br>
![bar chart for average wait per category](/figures/waitpercat.png "Average Wait")
</br>
On these graphs, we can see that Kids rides have the lowest average wait time and the highest guest ratings, suggesting that reducing wait times may lead to improved satisfaction. This is an important consideration for both the **Park General Manager** and the **Operations Director**.

## Analysis for Marketing Director

To further inspect transactions and promotional offers, I examined the fact_purchases table. </br>
By joining the fact_purchase table with the fact_visits table in a CTE, I was able to match visit_id and determine how many purchases were made by guests using promotional offers. I then grouped the data to gain insights into the number of purchases per promotional offer category. </br>

```
--cte for join of fact purchases and fact_visits
  WITH promopurchase AS (
  SELECT *
  FROM fact_purchases fp
  JOIN fact_visits fv ON fv.visit_id = fp.visit_id
)
--promotion code is not null refers to guests who have made purchases,
  SELECT promotion_code,
         COUNT(*) as count_of_purchases
  FROM promopurchase
  WHERE promotion_code IS NOT NULL
  GROUP BY promotion_code
```
<img width="256" height="125" alt="Screenshot 2025-08-22 at 5 03 09 PM" src="https://github.com/user-attachments/assets/42072b4a-5133-4bce-b85b-1db1b1f16254" />

- Guests with promotional offers made the most purchases. (50)
- Guests without promotional made the fewest purchases. (4)

### Visuals

![pie chart with purchase percentages](/figures/piepercent.png "Payment Percentages")
</br>
This pie chart represents the percentages of which promotional offer category makes the most purchases. 
- We can see the SUMMER25 offer provides the majority of purchases (72.2%) within the purchase table,
- VIPDAY second highest percentages (20.4). </br>
</br>
This suggests that promotional offers create opportunities for making purchases outside of the base ticket payment. This could be due to a lesser cost to enter the park, creating opportunities for purchases. </br>

# Final Recommendations for Next Quarter:
As we were able to find out in the analysis, </br>
1. **Operations Director:** There is a necessity to improve wait times all around for Supernova Theme Park, but the Water category is most affected.
   - Have mechanical support on site in case of any ride issues.
   - Provide queues for rides to alert guests 10-15 before their turn.
   - More staff to facilitate ride set up and reduce operational delays.
2. **Marketing Director:** Continue promotional offers, as they drive more attendance and provided for a total of 92.6% purchase rate outside of the base ticket price.
   - Potential to continue promotional offers during fall and winter seasons to keep attendance high.
   - Reach out to guests who visited most frequently to the park to drive future engagement.
3. **Park General Manager:** Avoid customer churn by:
   - Asking for feedback from guests with the lowest average rating.
   - Drive engagement through advertisements for more guests.

## Ethics and Biases:
### Data Cleaning
- 8 Duplicate values removed within fact_ride_events
- Removing 2 Attraction_ID from dim_attractions as they were duplicates of "Galaxy Coaster" and "Pirate Splash!"
- Adjusting all references to Attraction_ID 6 and 7 to 1 and 2 respectively, as they referenced the same ride.
- Fixing casing and trimming whitespace for all values within tables.
### Data Biases
- All data is referring 10 guests, could have a skew in rating because of it.
- Data only referencing a week, more data could provide different analysis.
- Many NULL values within records, had to proceed without imputation by ignoring NULLs.

### REPO NAVIGATION
```
|--data/
|-----themepark.db
|--figures/
|------piepercent.png
|------ratepercat.png
|------waitpercat.png
|--notebooks/
|------plots.ipynb
|--sql/
|------01_eda.sql
|------02_cleaning.sql
|------03_features.sql
|------04_ctes_windows.sql
|------04_extra_queries.sql
|------wiring.sql
|--README.md
```
