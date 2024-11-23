# Comprehensive Yelp User Dataset SQL Analysis Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Dataset Schema](#dataset-schema)
3. [Basic Level Queries](#basic-level-queries)
4. [Intermediate Level Queries](#intermediate-level-queries)
5. [Advanced Level Queries](#advanced-level-queries)
6. [Best Practices](#best-practices)

## Introduction
This guide provides a complete set of SQL queries for analyzing the Yelp user dataset using BigQuery. The queries range from basic to advanced complexity and include detailed explanations of both the SQL concepts and business value.

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

## Basic Level Queries

### 1. User Engagement Analysis
**Purpose**: Identify power users and potential brand ambassadors
**SQL Concepts**: Basic SELECT, ORDER BY, DATE functions

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

### 2. User Retention Analysis
**Purpose**: Understanding user retention patterns
**SQL Concepts**: CTEs, Window functions, Percentage calculations

```sql
WITH user_tenure AS (
    SELECT 
        EXTRACT(YEAR FROM CURRENT_DATE()) - EXTRACT(YEAR FROM DATE(yelping_since)) AS years_on_platform,
        COUNT(*) AS user_count
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    GROUP BY years_on_platform
)
SELECT 
    years_on_platform,
    user_count,
    ROUND(user_count * 100.0 / SUM(user_count) OVER(), 2) AS percentage
FROM user_tenure
ORDER BY years_on_platform;
```

## Intermediate Level Queries

### 3. Elite User Handling
**Purpose**: Processing elite user data
**SQL Concepts**: Array handling, String manipulation, Regular expressions

```sql
SELECT 
    user_id,
    name,
    ARRAY(
        SELECT SUBSTR(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', ''), pos, 4)
        FROM UNNEST(GENERATE_ARRAY(1, LENGTH(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '')) - 3, 4)) AS pos
    ) AS elite_years_split,
    ARRAY_LENGTH(
        ARRAY(
            SELECT SUBSTR(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', ''), pos, 4)
            FROM UNNEST(GENERATE_ARRAY(1, LENGTH(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '')) - 3, 4)) AS pos
        )
    ) AS elite_years_count
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
WHERE elite IS NOT NULL AND elite > 0;
```

### 4. Elite User Analysis
**Purpose**: Understanding valuable user segments
**SQL Concepts**: Complex CTEs, Conditional aggregation, Multiple metrics

```sql
WITH elite_years AS (
    SELECT 
        REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '') AS elite_years_raw,
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
        ARRAY(SELECT SUBSTR(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', ''), pos, 4)
              FROM UNNEST(GENERATE_ARRAY(1, LENGTH(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '')) - 3, 4)) AS pos) AS elite_years_split
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    WHERE elite IS NOT NULL AND elite > 0
),
user_segments AS (
    SELECT 
        CASE 
            WHEN ARRAY_LENGTH(elite_years_split) > 0 THEN 'Elite' 
            ELSE 'Regular' 
        END AS user_type,
        COUNT(*) AS user_count,
        AVG(review_count) AS avg_reviews,
        AVG(fans) AS avg_fans,
        AVG(average_stars) AS avg_rating,
        AVG(useful + funny + cool) AS avg_vote_count,
        AVG(
            compliment_hot + compliment_more + compliment_profile + 
            compliment_cute + compliment_list + compliment_note + 
            compliment_plain + compliment_cool + compliment_funny + 
            compliment_writer + compliment_photos
        ) AS avg_compliments
    FROM elite_years
    GROUP BY user_type
)
SELECT 
    *,
    ROUND(user_count * 100.0 / SUM(user_count) OVER(), 2) AS percentage_of_users
FROM user_segments;
```

## Advanced Level Queries

### 5. User Behavior Pattern Analysis
**Purpose**: Understanding engagement drivers
**SQL Concepts**: Correlation analysis, Complex metrics, CTEs

```sql
WITH elite_years_processed AS (
    SELECT 
        user_id,
        name,
        ARRAY(
            SELECT SUBSTR(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', ''), pos, 4)
            FROM UNNEST(GENERATE_ARRAY(1, LENGTH(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '')) - 3, 4)) AS pos
        ) AS elite_years_split,
        ARRAY_LENGTH(
            ARRAY(
                SELECT SUBSTR(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', ''), pos, 4)
                FROM UNNEST(GENERATE_ARRAY(1, LENGTH(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '')) - 3, 4)) AS pos
            )
        ) AS elite_years_count,
        review_count,
        fans,
        useful + funny + cool AS total_votes_given,
        compliment_hot + compliment_more + compliment_profile + 
        compliment_cute + compliment_list + compliment_note + 
        compliment_plain + compliment_cool + compliment_funny + 
        compliment_writer + compliment_photos AS total_compliments,
        average_stars,
        ARRAY_LENGTH(SPLIT(friends, ',')) AS friend_count
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    WHERE elite IS NOT NULL AND elite > 0
)
SELECT 
    CORR(review_count, fans) AS review_fan_correlation,
    CORR(review_count, total_compliments) AS review_compliment_correlation,
    CORR(friend_count, fans) AS friend_fan_correlation,
    CORR(elite_years_count, average_stars) AS elite_rating_correlation,
    CORR(total_votes_given, total_compliments) AS give_receive_correlation
FROM elite_years_processed;
```

### 6. Social Network Analysis
**Purpose**: Identifying influencers and network effects
**SQL Concepts**: String splitting, Multiple metrics, Complex aggregation

```sql
SELECT 
    name,
    ARRAY_LENGTH(SPLIT(friends, ',')) AS friend_count,
    fans,
    review_count,
    average_stars,
    useful + funny + cool AS total_votes_given,
    compliment_hot + compliment_more + compliment_profile + 
    compliment_cute + compliment_list + compliment_note + 
    compliment_plain + compliment_cool + compliment_funny + 
    compliment_writer + compliment_photos AS total_compliments_received
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
ORDER BY friend_count DESC
LIMIT 20;
```

### 7. User Growth Analysis
**Purpose**: Platform growth and engagement trends
**SQL Concepts**: Time-based analysis, Window functions, YoY calculations

```sql
WITH yearly_cohorts AS (
    SELECT 
        EXTRACT(YEAR FROM DATE(yelping_since)) AS join_year,
        COUNT(*) AS new_users,
        AVG(review_count) AS avg_reviews_per_user,
        AVG(fans) AS avg_fans_per_user,
        SUM(
            CASE 
                WHEN ARRAY_LENGTH(
                    ARRAY(
                        SELECT SUBSTR(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', ''), pos, 4)
                        FROM UNNEST(GENERATE_ARRAY(1, LENGTH(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '')) - 3, 4)) AS pos
                    )
                ) > 0 
                THEN 1 ELSE 0 
            END
        ) AS elite_users,
        AVG(average_stars) AS avg_rating_given
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
    ROUND(elite_users * 100.0 / new_users, 2) AS elite_percentage,
    avg_rating_given,
    LAG(new_users) OVER (ORDER BY join_year) AS prev_year_users,
    ROUND(
        (new_users - LAG(new_users) OVER (ORDER BY join_year)) * 100.0 / 
        NULLIF(LAG(new_users) OVER (ORDER BY join_year), 0), 2
    ) AS yoy_growth
FROM yearly_cohorts
ORDER BY join_year;
```

### 8. Compliment Analysis
**Purpose**: Understanding user recognition patterns
**SQL Concepts**: UNION ALL, Percentage calculations, Multiple aggregations

```sql
WITH compliment_totals AS (
    SELECT 
        'hot' AS compliment_type,
        SUM(compliment_hot) AS total
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL 
    SELECT 'more', SUM(compliment_more)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL 
    SELECT 'profile', SUM(compliment_profile)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL 
    SELECT 'cute', SUM(compliment_cute)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL 
    SELECT 'list', SUM(compliment_list)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL 
    SELECT 'note', SUM(compliment_note)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL 
    SELECT 'plain', SUM(compliment_plain)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL 
    SELECT 'cool', SUM(compliment_cool)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL 
    SELECT 'funny', SUM(compliment_funny)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL 
    SELECT 'writer', SUM(compliment_writer)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    UNION ALL 
    SELECT 'photos', SUM(compliment_photos)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
)
SELECT 
    compliment_type,
    total,
    ROUND(total * 100.0 / SUM(total) OVER(), 2) AS percentage
FROM compliment_totals
ORDER BY total DESC;
```

## Additional Analytical Queries

### User Review Count Distribution
**Purpose**: Identifies top contributors
```sql
SELECT 
    user_id,
    name,
    review_count
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
ORDER BY review_count DESC
LIMIT 10;
```

### Longest Active Users
**Purpose**: Identifies platform loyalty
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

### Users with Most Friends
**Purpose**: Identifies social influencers
```sql
SELECT 
    user_id,
    name,
    ARRAY_LENGTH(friends) AS friend_count
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
ORDER BY friend_count DESC
LIMIT 10;
```

### User Voting Patterns
**Purpose**: Understanding engagement through votes
```sql
SELECT 
    ROUND(AVG(useful), 2) AS avg_useful_votes,
    ROUND(AVG(funny), 2) AS avg_funny_votes,
    ROUND(AVG(cool), 2) AS avg_cool_votes
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`;
```

### Sentiment Leaders
**Purpose**: Identifies well-received contributors
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