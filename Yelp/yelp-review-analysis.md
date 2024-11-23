# Yelp Review Data Analysis: SQL Query Guide

## Table Structure
The analysis is performed on the Yelp review table with the following schema:
```sql
-- Schema:
-- review_id: string (22 char unique identifier)
-- user_id: string (maps to user.json)
-- business_id: string (maps to business.json)
-- stars: integer (rating)
-- date: string (YYYY-MM-DD)
-- text: string (review content)
-- useful: integer (useful votes)
-- funny: integer (funny votes)
-- cool: integer (cool votes)
```

## 1. Basic Queries

### 1.1 Recent Reviews Query
```sql
-- Purpose: Find the most recent reviews to analyze latest customer feedback
-- Key concepts: Basic SELECT, ORDER BY, LIMIT
SELECT *
FROM `long-loop-442611-j5.Yelp_Business_Part1.review`
ORDER BY date DESC  -- Sort by date in descending order (newest first)
LIMIT 10;          -- Only get top 10 results for quick analysis
```

### 1.2 Average Rating Analysis
```sql
-- Purpose: Get overall platform rating health and total review volume
-- Key concepts: Aggregate functions (AVG, COUNT)
SELECT 
    AVG(stars) as avg_rating,        -- Calculate mean rating across all reviews
    COUNT(*) as total_reviews        -- Get total number of reviews in system
FROM `long-loop-442611-j5.Yelp_Business_Part1.review`;
```

### 1.3 Yearly Review Distribution
```sql
-- Purpose: Understand review volume trends over years
-- Key concepts: Date functions, GROUP BY
SELECT 
    EXTRACT(YEAR FROM date) as year,  -- Extract year from date field
    COUNT(*) as review_count          -- Count reviews per year
FROM `long-loop-442611-j5.Yelp_Business_Part1.review`
GROUP BY year
ORDER BY year;
```

## 2. Intermediate Queries

### 2.1 Top Reviewers Analysis
```sql
-- Purpose: Identify most active users and their rating patterns
-- Key concepts: Multiple aggregations, GROUP BY
SELECT 
    user_id,
    COUNT(*) as review_count,         -- Total reviews by user
    AVG(stars) as avg_rating          -- User's average rating
FROM `long-loop-442611-j5.Yelp_Business_Part1.review`
GROUP BY user_id
ORDER BY review_count DESC            -- Sort by most reviews first
LIMIT 5;
```

### 2.2 High-Quality Business Analysis
```sql
-- Purpose: Find businesses with consistently good ratings and engagement
-- Key concepts: HAVING clause, multiple conditions
SELECT 
    business_id,
    AVG(stars) as avg_stars,          -- Average rating
    SUM(useful) as total_useful,      -- Total useful votes
    COUNT(*) as review_count          -- Number of reviews
FROM `long-loop-442611-j5.Yelp_Business_Part1.review`
GROUP BY business_id
HAVING avg_stars >= 4                 -- Only businesses with high ratings
    AND total_useful >= 10            -- And significant engagement
ORDER BY total_useful DESC;
```

### 2.3 Review Length Analysis
```sql
-- Purpose: Understand correlation between review length and ratings
-- Key concepts: String functions, arrays
SELECT 
    stars,
    -- Split text into words and count them
    AVG(ARRAY_LENGTH(SPLIT(text, ' '))) as avg_word_count,
    COUNT(*) as review_count
FROM `long-loop-442611-j5.Yelp_Business_Part1.review`
GROUP BY stars
ORDER BY stars;
```

## 3. Advanced Queries

### 3.1 Rating Variation Analysis
```sql
-- Purpose: Identify businesses with inconsistent ratings
-- Key concepts: Standard deviation, statistical analysis
SELECT 
    business_id,
    COUNT(*) as review_count,
    AVG(stars) as avg_stars,
    STDDEV(stars) as stars_stddev     -- Calculate rating standard deviation
FROM `long-loop-442611-j5.Yelp_Business_Part1.review`
GROUP BY business_id
HAVING review_count >= 10             -- Only consider businesses with enough reviews
    AND stars_stddev > 1              -- Show those with high rating variation
ORDER BY stars_stddev DESC;
```

### 3.2 Rolling Average Analysis
```sql
-- Purpose: Track rating trends over time using moving averages
-- Key concepts: Window functions, date ranges
SELECT 
    business_id,
    date,
    stars,
    AVG(stars) OVER (
        PARTITION BY business_id      -- Calculate separately for each business
        ORDER BY date                 -- Order by date for time-based analysis
        RANGE BETWEEN 30 PRECEDING    -- Look back 30 days
        AND CURRENT ROW              -- Up to current date
    ) as rolling_30_day_avg
FROM `long-loop-442611-j5.Yelp_Business_Part1.review`
ORDER BY business_id, date;
```

### 3.3 User Rating Deviation Analysis
```sql
-- Purpose: Find users whose ratings differ significantly from global average
-- Key concepts: CTEs, subqueries, global aggregates
WITH user_stats AS (
    SELECT 
        user_id,
        COUNT(*) as review_count,
        AVG(stars) as user_avg_rating,
        -- Subquery to get global average
        (SELECT AVG(stars) FROM `long-loop-442611-j5.Yelp_Business_Part1.review`) as global_avg
    FROM `long-loop-442611-j5.Yelp_Business_Part1.review`
    GROUP BY user_id
    HAVING review_count >= 5          -- Only consider active users
)
SELECT *,
       -- Calculate how much user differs from global average
       ABS(user_avg_rating - global_avg) as rating_difference
FROM user_stats
WHERE ABS(user_avg_rating - global_avg) >= 1
ORDER BY rating_difference DESC;
```

### 3.4 Seasonal Pattern Analysis
```sql
-- Purpose: Identify seasonal trends in ratings
-- Key concepts: CTEs, multiple time periods, complex aggregations
WITH seasonal_stats AS (
    SELECT 
        business_id,
        EXTRACT(MONTH FROM date) as month,
        EXTRACT(YEAR FROM date) as year,
        AVG(stars) as avg_stars,
        COUNT(*) as review_count
    FROM `long-loop-442611-j5.Yelp_Business_Part1.review`
    GROUP BY business_id, month, year
)
SELECT 
    business_id,
    month,
    AVG(avg_stars) as avg_monthly_rating,
    AVG(review_count) as avg_monthly_reviews,
    COUNT(DISTINCT year) as years_of_data
FROM seasonal_stats
GROUP BY business_id, month
HAVING years_of_data >= 2            -- Only show businesses with multi-year data
ORDER BY business_id, month;
```

### 3.5 Review Length Impact Analysis
```sql
-- Purpose: Analyze how review length affects engagement and ratings
-- Key concepts: CASE statements, string functions
SELECT 
    CASE 
        WHEN LENGTH(text) < 100 THEN 'Short'
        WHEN LENGTH(text) < 500 THEN 'Medium'
        ELSE 'Long'
    END as review_length_category,    -- Categorize reviews by length
    AVG(stars) as avg_rating,         -- Average rating per category
    AVG(useful) as avg_useful_votes,  -- Average engagement metrics
    AVG(funny) as avg_funny_votes,
    AVG(cool) as avg_cool_votes,
    COUNT(*) as review_count
FROM `long-loop-442611-j5.Yelp_Business_Part1.review`
GROUP BY review_length_category
ORDER BY review_count DESC;
```

## Key SQL Concepts Demonstrated

1. **Basic SQL Operations**
   - SELECT, FROM, WHERE
   - ORDER BY, GROUP BY, HAVING
   - Aggregate functions (COUNT, AVG, SUM)

2. **Intermediate Concepts**
   - String functions (LENGTH, SPLIT)
   - Date/time functions (EXTRACT)
   - CASE statements
   - Multiple aggregations

3. **Advanced Concepts**
   - Window functions
   - Common Table Expressions (CTEs)
   - Statistical functions (STDDEV)
   - Complex subqueries
   - Date range analysis

4. **Performance Considerations**
   - Using appropriate indexes
   - Limiting result sets
   - Efficient grouping strategies
   - Proper use of window functions vs. self-joins

This guide provides a comprehensive set of SQL queries for analyzing Yelp review data, from basic metrics to advanced pattern analysis. Each query is designed to answer specific business questions while demonstrating various SQL concepts and best practices.
