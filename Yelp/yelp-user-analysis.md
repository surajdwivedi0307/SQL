# Comprehensive Yelp User Dataset SQL Analysis

## Table of Contents
1. [Introduction](#introduction)
2. [Dataset Schema](#dataset-schema)
3. [Analysis Categories](#analysis-categories)
4. [Detailed Queries and Analysis](#detailed-queries)
5. [Best Practices](#best-practices)

## Introduction
This document provides detailed SQL queries for analyzing the Yelp user dataset using BigQuery. The queries are categorized by complexity level and include explanations and business value.

## Dataset Schema
```json
{
    "user_id": "string (22 chars)",
    "name": "string",
    "review_count": "integer",
    "yelping_since": "date (YYYY-MM-DD)",
    "friends": "array of strings (user_ids)",
    "useful": "integer",
    "funny": "integer",
    "cool": "integer",
    "fans": "integer",
    "elite": "array of integers (years)",
    "average_stars": "float",
    "compliment_hot": "integer",
    "compliment_more": "integer",
    "compliment_profile": "integer",
    "compliment_cute": "integer",
    "compliment_list": "integer",
    "compliment_note": "integer",
    "compliment_plain": "integer",
    "compliment_cool": "integer",
    "compliment_funny": "integer",
    "compliment_writer": "integer",
    "compliment_photos": "integer"
}
```

## Detailed Queries and Analysis

### Basic Level Queries

#### 1. User Engagement Analysis
**Question:** Who are the most active reviewers on the platform?
**Business Value:** Identify power users and potential brand ambassadors
```sql
SELECT 
    name,
    review_count,
    fans,
    average_stars,
    DATE(yelping_since) as member_since,
    DATE_DIFF(CURRENT_DATE(), DATE(yelping_since), DAY) as days_on_platform
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
ORDER BY review_count DESC
LIMIT 20;
```

#### 2. User Retention Analysis
**Question:** What's the distribution of user tenure on the platform?
**Business Value:** Understanding user retention patterns
```sql
-- Step 1: Create a Common Table Expression (CTE) named `user_tenure` to calculate user tenure
WITH user_tenure AS (
    SELECT 
        -- Calculate the number of years a user has been on the platform
        EXTRACT(YEAR FROM CURRENT_DATE()) - EXTRACT(YEAR FROM DATE(yelping_since)) AS years_on_platform,
        -- Count the number of users with the same number of years on the platform
        COUNT(*) AS user_count
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    -- Group by the number of years a user has been on the platform
    GROUP BY years_on_platform
)

-- Step 2: Use the CTE to calculate the percentage distribution of users based on their tenure
SELECT 
    -- Include the years of user tenure
    years_on_platform,
    -- Include the count of users for each tenure
    user_count,
    -- Calculate the percentage of users for each tenure relative to the total number of users
    ROUND(user_count * 100.0 / SUM(user_count) OVER(), 2) AS percentage
FROM user_tenure
-- Sort the results by years on the platform in ascending order
ORDER BY years_on_platform;

```

### Intermediate Level Queries

#### 3. Elite User Handling 
```sql
-- Part 1: Handle the 'elite' years treatment separately and calculate user metrics

SELECT 
    user_id,
    name,
    
    -- Convert the 'elite' years into an array of 4-digit years
    -- Remove the decimal, split the years into 4-digit substrings, and create an array
    ARRAY(
        SELECT SUBSTR(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', ''), pos, 4)
        FROM UNNEST(GENERATE_ARRAY(1, LENGTH(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '')) - 3, 4)) AS pos
    ) AS elite_years_split,
    
    -- Calculate the number of years a user has been elite (number of 4-digit years)
    ARRAY_LENGTH(
        ARRAY(
            SELECT SUBSTR(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', ''), pos, 4)
            FROM UNNEST(GENERATE_ARRAY(1, LENGTH(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '')) - 3, 4)) AS pos
        )
    ) AS elite_years_count
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
WHERE elite IS NOT NULL AND elite > 0;  -- Filter users with elite years
```

#### 4. Elite User Analysis
**Question:** What characteristics define elite users compared to regular users?
**Business Value:** Understanding valuable user segments
```sql
-- Step 1: Create a Common Table Expression (CTE) to split the elite years into valid 4-digit years
WITH elite_years AS (
    SELECT 
        -- Convert the elite value to a string and remove the decimal point
        REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '') AS elite_years_raw,
        
        -- Extract user-specific data
        user_id,
        name,
        review_count,
        fans,
        average_stars,
        useful,
        funny,
        cool,
        compliment_hot,
        compliment_more,
        compliment_profile,
        compliment_cute,
        compliment_list,
        compliment_note,
        compliment_plain,
        compliment_cool,
        compliment_funny,
        compliment_writer,
        compliment_photos,
        
        -- Split the elite years string into an array of 4-digit substrings
        ARRAY(SELECT SUBSTR(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', ''), pos, 4)
              FROM UNNEST(GENERATE_ARRAY(1, LENGTH(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '')) - 3, 4)) AS pos) AS elite_years_split
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    WHERE elite IS NOT NULL AND elite > 0 -- Filter users with elite years
)

-- Step 2: Create user segments based on the presence of elite years
, user_segments AS (
    SELECT 
        -- Classify users as 'Elite' if they have any valid 4-digit years, otherwise 'Regular'
        CASE 
            WHEN ARRAY_LENGTH(elite_years_split) > 0 THEN 'Elite' 
            ELSE 'Regular' 
        END AS user_type,
        
        -- Count the number of users in each segment
        COUNT(*) AS user_count,
        
        -- Calculate the average number of reviews written by users in each segment
        AVG(review_count) AS avg_reviews,
        
        -- Calculate the average number of fans per user in each segment
        AVG(fans) AS avg_fans,
        
        -- Calculate the average star rating given by users in each segment
        AVG(average_stars) AS avg_rating,
        
        -- Calculate the average total number of votes ('useful', 'funny', 'cool') received by users in each segment
        AVG(useful + funny + cool) AS avg_vote_count,
        
        -- Calculate the average total number of compliments received by users in each segment
        AVG(
            compliment_hot + compliment_more + compliment_profile + 
            compliment_cute + compliment_list + compliment_note + 
            compliment_plain + compliment_cool + compliment_funny + 
            compliment_writer + compliment_photos
        ) AS avg_compliments
    FROM elite_years
    GROUP BY user_type
)

-- Step 3: Use the CTE to calculate the percentage of users in each segment
SELECT 
    *,
    -- Calculate the percentage of users in each segment relative to the total number of users
    ROUND(user_count * 100.0 / SUM(user_count) OVER(), 2) AS percentage_of_users
FROM user_segments;

```
#### 5. Elite User Correlation
```sql
-- Part 1: Handle the 'elite' years treatment separately and calculate user metrics
WITH elite_years_processed AS (
    SELECT 
        user_id,
        name,
        
        -- Convert the 'elite' years into an array of 4-digit years
        -- Remove the decimal, split the years into 4-digit substrings, and create an array
        ARRAY(
            SELECT SUBSTR(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', ''), pos, 4)
            FROM UNNEST(GENERATE_ARRAY(1, LENGTH(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '')) - 3, 4)) AS pos
        ) AS elite_years_split,
        
        -- Calculate the number of years a user has been elite (number of 4-digit years)
        ARRAY_LENGTH(
            ARRAY(
                SELECT SUBSTR(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', ''), pos, 4)
                FROM UNNEST(GENERATE_ARRAY(1, LENGTH(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '')) - 3, 4)) AS pos
            )
        ) AS elite_years_count,
        
        -- Other necessary user information
        review_count,    -- Number of reviews written by the user
        fans,            -- Number of fans the user has
        useful + funny + cool AS total_votes_given,  -- Total votes given by the user
        compliment_hot + compliment_more + compliment_profile + 
        compliment_cute + compliment_list + compliment_note + 
        compliment_plain + compliment_cool + compliment_funny + 
        compliment_writer + compliment_photos AS total_compliments,  -- Total compliments received by the user
        average_stars,   -- Average star rating given by the user
        
        -- Handle 'friends' as a comma-separated string, split it into an array and then count the number of friends
        ARRAY_LENGTH(SPLIT(friends, ',')) AS friend_count  -- Number of friends the user has (split string into an array)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    WHERE elite IS NOT NULL AND elite > 0 -- Filter users who have elite years
)

-- Part 2: Calculate correlations between different user metrics
SELECT 
    -- Calculate correlation between review_count and fans (how reviews and fans relate)
    CORR(review_count, fans) AS review_fan_correlation,
    
    -- Calculate correlation between review_count and total_compliments (how reviews and compliments relate)
    CORR(review_count, total_compliments) AS review_compliment_correlation,
    
    -- Calculate correlation between friend_count and fans (how number of friends and fans relate)
    CORR(friend_count, fans) AS friend_fan_correlation,
    
    -- Calculate correlation between elite_years_count and average_stars (how elite status relates to ratings)
    CORR(elite_years_count, average_stars) AS elite_rating_correlation,
    
    -- Calculate correlation between total_votes_given and total_compliments (how voting and compliments relate)
    CORR(total_votes_given, total_compliments) AS give_receive_correlation
FROM elite_years_processed;
```

#### 4. Social Network Analysis
**Question:** Who are the most connected users and what's their impact?
**Business Value:** Identifying influencers and network effects
```sql
SELECT 
    name,
    
    -- Check if 'friends' is stored as an array (standard case)
    -- If 'friends' is already an array, use ARRAY_LENGTH directly
    -- If it's stored as a string, we'll split it by commas to treat it as an array
    ARRAY_LENGTH(SPLIT(friends, ',')) AS friend_count,  -- Count number of friends by splitting the string

    -- Extract the number of fans a user has
    fans,
    
    -- Count the total number of reviews written by the user
    review_count,
    
    -- Calculate the average star rating for all reviews by the user
    average_stars,
    
    -- Sum up the total votes given by the user: 'useful', 'funny', 'cool'
    useful + funny + cool AS total_votes_given,
    
    -- Calculate the total number of compliments received by the user.
    -- This adds up the number of compliments in different categories.
    compliment_hot + compliment_more + compliment_profile + 
    compliment_cute + compliment_list + compliment_note + 
    compliment_plain + compliment_cool + compliment_funny + 
    compliment_writer + compliment_photos AS total_compliments_received

FROM `long-loop-442611-j5.Yelp_Business_Part1.user`

-- Order the users by the number of friends in descending order to show the most socially connected users
ORDER BY friend_count DESC

-- Limit the results to the top 20 users based on the number of friends
LIMIT 20;

```

### Advanced Level Queries

#### 5. User Behavior Pattern Analysis
**Question:** What's the correlation between user activity and reception?
**Business Value:** Understanding engagement drivers
```sql
-- Step 1: Handle friends and elite years separately and calculate user metrics
WITH user_metrics AS (
    SELECT 
        review_count,    -- Number of reviews written by the user
        fans,            -- Number of fans the user has
        
        -- Calculate the total votes given by summing the 'useful', 'funny', and 'cool' votes
        useful + funny + cool AS total_votes_given,
        
        -- Calculate the total number of compliments received by summing all compliment types
        compliment_hot + compliment_more + compliment_profile + 
        compliment_cute + compliment_list + compliment_note + 
        compliment_plain + compliment_cool + compliment_funny + 
        compliment_writer + compliment_photos AS total_compliments,
        
        average_stars,   -- Average star rating given by the user
        
        -- Handle 'friends' as a comma-separated string: split it into an array and count the number of friends
        ARRAY_LENGTH(SPLIT(friends, ',')) AS friend_count,  -- Number of friends the user has
        
        -- Handle 'elite' years as a concatenated string of years
        ARRAY_LENGTH(
            ARRAY(
                SELECT SUBSTR(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', ''), pos, 4)
                FROM UNNEST(GENERATE_ARRAY(1, LENGTH(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '')) - 3, 4)) AS pos
            )
        ) AS elite_years  -- Number of years the user has been elite
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    WHERE elite IS NOT NULL AND elite > 0  -- Filter users who have elite years
)

-- Step 2: Calculate correlations between different user metrics
SELECT 
    -- Calculate correlation between review_count and fans (how reviews and fans relate)
    CORR(review_count, fans) AS review_fan_correlation,
    
    -- Calculate correlation between review_count and total_compliments (how reviews and compliments relate)
    CORR(review_count, total_compliments) AS review_compliment_correlation,
    
    -- Calculate correlation between friend_count and fans (how number of friends and fans relate)
    CORR(friend_count, fans) AS friend_fan_correlation,
    
    -- Calculate correlation between elite_years and average_stars (how elite status relates to ratings)
    CORR(elite_years, average_stars) AS elite_rating_correlation,
    
    -- Calculate correlation between total_votes_given and total_compliments (how voting and compliments relate)
    CORR(total_votes_given, total_compliments) AS give_receive_correlation
FROM user_metrics;

```

#### 6. User Growth Analysis
**Question:** How has user engagement evolved over time?
**Business Value:** Platform growth and engagement trends
```sql
WITH yearly_cohorts AS (
    SELECT 
        EXTRACT(YEAR FROM DATE(yelping_since)) as join_year,
        COUNT(*) as new_users,
        AVG(review_count) as avg_reviews_per_user,
        AVG(fans) as avg_fans_per_user,
        SUM(CASE WHEN ARRAY_LENGTH(elite) > 0 THEN 1 ELSE 0 END) as elite_users,
        AVG(average_stars) as avg_rating_given
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    GROUP BY join_year
    HAVING join_year >= 2004
)
SELECT 
    join_year,
    new_users,
    avg_reviews_per_user,
    avg_fans_per_user,
    elite_users,
    ROUND(elite_users * 100.0 / new_users, 2) as elite_percentage,
    avg_rating_given,
    LAG(new_users) OVER (ORDER BY join_year) as prev_year_users,
    ROUND((new_users - LAG(new_users) OVER (ORDER BY join_year)) * 100.0 / 
        NULLIF(LAG(new_users) OVER (ORDER BY join_year), 0), 2) as yoy_growth
FROM yearly_cohorts
ORDER BY join_year;
```

#### 7. Compliment Analysis
**Question:** What types of compliments are most common and their relationship with user engagement?
**Business Value:** Understanding user recognition patterns
```sql
WITH compliment_totals AS (
    SELECT 
        'hot' as compliment_type, SUM(compliment_hot) as total
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL SELECT 'more', SUM(compliment_more)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL SELECT 'profile', SUM(compliment_profile)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL SELECT 'cute', SUM(compliment_cute)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL SELECT 'list', SUM(compliment_list)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL SELECT 'note', SUM(compliment_note)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL SELECT 'plain', SUM(compliment_plain)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL SELECT 'cool', SUM(compliment_cool)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL SELECT 'funny', SUM(compliment_funny)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL SELECT 'writer', SUM(compliment_writer)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL SELECT 'photos', SUM(compliment_photos)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
)
SELECT 
    compliment_type,
    total,
    ROUND(total * 100.0 / SUM(total) OVER(), 2) as percentage
FROM compliment_totals
ORDER BY total DESC;
```

Based on the structure and type of analysis from the provided file, here’s a similar exercise for the **Yelp_User_Part1** dataset based on its schema:

---

## Example Analyses for `long-loop-442611-j5.Yelp_Business_Part1.user`

### Basic Level Queries

#### 1. User Review Count Distribution
**Question:** What are the top 10 users based on the number of reviews written?  
**Business Value:** Identifies the most active contributors to Yelp's platform.

```sql
SELECT 
    user_id,
    name,
    review_count
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
ORDER BY review_count DESC
LIMIT 10;
```

---

#### 2. Longest Active Users
**Question:** Which users have been active on Yelp for the longest duration?  
**Business Value:** Highlights loyal users for targeted campaigns.

```sql
SELECT 
    user_id,
    name,
    yelping_since,
    DATE_DIFF(CURRENT_DATE(), DATE(yelping_since), YEAR) AS years_active
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
ORDER BY years_active DESC
LIMIT 10;
```

---

#### 3. Users with Most Friends
**Question:** Who are the top 10 users with the highest number of friends?  
**Business Value:** Identifies socially connected users for influencer marketing.

```sql
SELECT 
    user_id,
    name,
    ARRAY_LENGTH(friends) AS friend_count
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
ORDER BY friend_count DESC
LIMIT 10;
```

---

### Intermediate Level Queries

#### 4. Elite User Analysis
**Question:** What is the distribution of elite users by the number of years they have been elite?  
**Business Value:** Provides insights into Yelp's elite user community.

```sql
SELECT 
    ARRAY_LENGTH(elite) AS elite_years,
    COUNT(*) AS user_count
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
GROUP BY elite_years
ORDER BY elite_years DESC;
```

---

#### 5. User Voting Patterns
**Question:** What are the average counts of 'useful,' 'funny,' and 'cool' votes across all users?  
**Business Value:** Understands user engagement through voting patterns.

```sql
SELECT 
    ROUND(AVG(useful), 2) AS avg_useful_votes,
    ROUND(AVG(funny), 2) AS avg_funny_votes,
    ROUND(AVG(cool), 2) AS avg_cool_votes
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`;
```

---

### Advanced Level Queries

#### 6. User Activity Versus Fan Base
**Question:** Is there a correlation between the number of reviews written and the number of fans a user has?  
**Business Value:** Explores trends between content contribution and fan following.

```sql
SELECT 
    review_count,
    fans,
    CORR(review_count, fans) AS review_fan_correlation
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`;
```

---

#### 7. Sentiment Leaders
**Question:** Which users have the highest average 'hot,' 'cool,' and 'funny' compliments?  
**Business Value:** Identifies users whose contributions are well-received.

```sql
SELECT 
    user_id,
    name,
    compliment_hot,
    compliment_cool,
    compliment_funny,
    ROUND((compliment_hot + compliment_cool + compliment_funny) / 3.0, 2) AS avg_compliments
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
ORDER BY avg_compliments DESC
LIMIT 10;
```

---

#### 8. High Impact Elite Users
**Question:** Identify elite users with a high number of reviews, fans, and compliments.  
**Business Value:** Targets influential elite users for partnership opportunities.

```sql
SELECT 
    user_id,
    name,
    review_count,
    fans,
    compliment_hot + compliment_cool + compliment_funny AS total_compliments
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
WHERE ARRAY_LENGTH(elite) > 0
ORDER BY review_count DESC, fans DESC, total_compliments DESC
LIMIT 10;
```

---

These queries can be further refined and customized to explore specific behaviors or patterns in user activity, social connections, and engagement metrics on Yelp. Let me know if you'd like a deeper dive into any particular analysis!

## Best Practices

### Query Optimization
- Use appropriate indexes on frequently queried columns
- Filter early in the query execution
- Use CTEs for better readability and maintenance
- Consider query cost and performance impacts

### Data Quality Considerations
- Handle NULL values appropriately
- Validate date ranges for temporal analysis
- Check for data consistency across related fields
- Consider the impact of outliers on aggregations

### Performance Tips
- Use appropriate data types
- Optimize JOIN operations
- Consider materialized views for frequent complex queries
- Use appropriate partitioning strategies

## Conclusion

These queries provide insights into:
1. User engagement patterns
2. Platform growth trends
3. Social network effects
4. Elite user characteristics
5. Compliment and recognition patterns
6. User retention and churn factors

The analysis can be used for:
- User engagement optimization
- Feature development prioritization
- Community management strategies
- Marketing campaign targeting
- Platform growth planning
