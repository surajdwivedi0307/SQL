# Comprehensive Yelp Business Dataset SQL Analysis

## Table of Contents
1. [Introduction](#introduction)
2. [Dataset Schema](#dataset-schema)
3. [Analysis Categories](#analysis-categories)
4. [Detailed Queries and Analysis](#detailed-queries)
5. [Best Practices](#best-practices)
6. [Implementation Guidelines](#implementation)

## Introduction

This document provides detailed SQL queries for analyzing the Yelp business dataset using BigQuery. I have categorized the queries by complexity level and included explanations, business value, and complete SQL code.

## Dataset Schema

```json
{
    "business_id": "string (22 chars)",
    "name": "string",
    "address": "string",
    "city": "string",
    "state": "string (2 chars)",
    "postal code": "string",
    "latitude": "float",
    "longitude": "float",
    "stars": "float",
    "review_count": "integer",
    "is_open": "integer (0 or 1)",
    "attributes": {
        "RestaurantsTakeOut": "boolean",
        "BusinessParking": {
            "garage": "boolean",
            "street": "boolean",
            "validated": "boolean",
            "lot": "boolean",
            "valet": "boolean"
        }
    },
    "categories": "array of strings",
    "hours": {
        "Monday": "string (HH:MM-HH:MM)",
        "Tuesday": "string (HH:MM-HH:MM)",
        "Wednesday": "string (HH:MM-HH:MM)",
        "Thursday": "string (HH:MM-HH:MM)",
        "Friday": "string (HH:MM-HH:MM)",
        "Saturday": "string (HH:MM-HH:MM)",
        "Sunday": "string (HH:MM-HH:MM)"
    }
}
```

## Detailed Queries and Analysis

### Basic Level Queries

#### 1. City-wise Restaurant Distribution
**Question:** What are the top 10 cities with the most restaurants?
**Business Value:** Market penetration analysis and expansion planning
```sql
SELECT 
    city,
    state,
    COUNT(*) as restaurant_count
FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
GROUP BY city, state
ORDER BY restaurant_count DESC
LIMIT 10;
```

#### 2. High-Rated Restaurant Analysis
**Question:** Find all Mexican restaurants in Berline with a rating of 4.0 or higher that are currently open
**Business Value:** Competitive analysis
```sql
SELECT *
FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
WHERE city = 'Berlin' 
    AND categories LIKE '%Mexican%'
    AND stars >= 4.0
    AND is_open = 1
ORDER BY stars DESC;
```

#### 3. State Performance Analysis
**Question:** What's the average rating for businesses in each state?
**Business Value:** Geographic performance benchmarking
```sql
SELECT 
    state,
    ROUND(AVG(stars), 2) as avg_rating,
    COUNT(*) as total_businesses
FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
GROUP BY state
HAVING total_businesses > 10
ORDER BY avg_rating DESC;
```

### Intermediate Level Queries

#### 4. Category Distribution in Major Cities
**Question:** Find the distribution of business categories in cities with more than 1000 businesses
**Business Value:** Market composition understanding
```sql
-- Step 1: Define a Common Table Expression (CTE) to find cities with more than 1000 businesses.
WITH city_businesses AS (
    SELECT 
        city,                      -- Select the city name.
        COUNT(*) as total          -- Count the total number of businesses in each city.
    FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
    GROUP BY city                  -- Group by city to get the count per city.
    HAVING total > 1000            -- Filter to include only cities with more than 1000 businesses.
)

-- Step 2: Use the main query to calculate the percentage distribution of categories in each city.
SELECT 
    b.city,                        -- The city name.
    category,                      -- Each individual category derived from the categories column.
    COUNT(*) as category_count,    -- Count of businesses in the city belonging to the specific category.
    ROUND(COUNT(*) * 100.0 / cb.total, 2) as percentage 
                                   -- Calculate the percentage of businesses in the city for the category.
                                   -- COUNT(*) * 100.0 / cb.total ensures a percentage format.
FROM 
    `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b
                                   -- Use the main business table with alias `b`.
CROSS JOIN 
    UNNEST(SPLIT(categories, ', ')) as category
                                   -- Split the `categories` column (comma-separated string) into an array.
                                   -- `UNNEST` converts the array into rows, one row per category per business.
JOIN 
    city_businesses cb 
    ON b.city = cb.city            -- Join the `city_businesses` CTE to include only cities with >1000 businesses.
GROUP BY 
    b.city,                        -- Group by city.
    category,                      -- Group by each category.
    cb.total                       -- Include the total business count for accurate percentage calculation.
ORDER BY 
    b.city,                        -- Sort results by city.
    category_count DESC;           -- Sort categories within each city by descending count.


```

#### 5. Operating Hours Analysis
**Question:** Which businesses are open the longest hours during the week?
**Business Value:** Operational patterns identification
```sql
WITH business_hours AS (
    SELECT 
        name,
        city,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Monday'), '-')[OFFSET(0)]) as opening_time,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Monday'), '-')[OFFSET(1)]) as closing_time,
        TIME_DIFF(
            PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Monday'), '-')[OFFSET(1)]),
            PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Monday'), '-')[OFFSET(0)]),
            HOUR
        ) as hours_open
    FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
    WHERE JSON_EXTRACT_SCALAR(hours, '$.Monday') IS NOT NULL
)
SELECT 
    name,
    city,
    FORMAT_TIME('%H:%M', opening_time) as opens_at,
    FORMAT_TIME('%H:%M', closing_time) as closes_at,
    hours_open
FROM business_hours
ORDER BY hours_open DESC
LIMIT 10;

```

### Advanced Level Queries

#### 6. Weekend Business Pattern Analysis
**Question:** Find businesses that show a pattern of being busier on weekends based on different operating hours
**Business Value:** Operational adaptation insights
```sql
WITH business_hours AS (
    SELECT 
        name,
        city,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Monday'), '-')[OFFSET(0)]) AS monday_open,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Monday'), '-')[OFFSET(1)]) AS monday_close,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Tuesday'), '-')[OFFSET(0)]) AS tuesday_open,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Tuesday'), '-')[OFFSET(1)]) AS tuesday_close,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Wednesday'), '-')[OFFSET(0)]) AS wednesday_open,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Wednesday'), '-')[OFFSET(1)]) AS wednesday_close,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Thursday'), '-')[OFFSET(0)]) AS thursday_open,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Thursday'), '-')[OFFSET(1)]) AS thursday_close,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Friday'), '-')[OFFSET(0)]) AS friday_open,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Friday'), '-')[OFFSET(1)]) AS friday_close,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Saturday'), '-')[OFFSET(0)]) AS saturday_open,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Saturday'), '-')[OFFSET(1)]) AS saturday_close,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Sunday'), '-')[OFFSET(0)]) AS sunday_open,
        PARSE_TIME('%H:%M', SPLIT(JSON_EXTRACT_SCALAR(hours, '$.Sunday'), '-')[OFFSET(1)]) AS sunday_close
    FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
    WHERE hours IS NOT NULL
),
business_analysis AS (
    SELECT 
        name,
        city,
        -- Calculate total weekday hours (Monday to Friday)
        COALESCE(TIME_DIFF(monday_close, monday_open, MINUTE), 0) +
        COALESCE(TIME_DIFF(tuesday_close, tuesday_open, MINUTE), 0) +
        COALESCE(TIME_DIFF(wednesday_close, wednesday_open, MINUTE), 0) +
        COALESCE(TIME_DIFF(thursday_close, thursday_open, MINUTE), 0) +
        COALESCE(TIME_DIFF(friday_close, friday_open, MINUTE), 0) AS weekday_minutes,
        
        -- Calculate total weekend hours (Saturday and Sunday)
        COALESCE(TIME_DIFF(saturday_close, saturday_open, MINUTE), 0) +
        COALESCE(TIME_DIFF(sunday_close, sunday_open, MINUTE), 0) AS weekend_minutes
    FROM business_hours
),
weekend_focused_businesses AS (
    SELECT 
        name,
        city,
        weekday_minutes,
        weekend_minutes,
        ROUND(weekend_minutes * 1.0 / NULLIF(weekday_minutes, 0), 2) AS weekend_to_weekday_ratio
    FROM business_analysis
    WHERE weekend_minutes > weekday_minutes -- Focus on businesses with more weekend hours
)
SELECT 
    name,
    city,
    weekday_minutes / 60 AS weekday_hours,
    weekend_minutes / 60 AS weekend_hours,
    weekend_to_weekday_ratio
FROM weekend_focused_businesses
ORDER BY weekend_to_weekday_ratio DESC
LIMIT 10;

```

#### 7. Geographic Cluster Analysis
**Question:** Identify geographic clusters of highly-rated businesses
**Business Value:** Location strategy optimization
```sql
-- Create a temporary table (CTE) to calculate nearby businesses
WITH nearby_businesses AS (
    SELECT 
        b1.business_id,        -- Unique identifier of the business being evaluated
        b1.name,               -- Name of the business being evaluated
        b1.latitude,           -- Latitude of the business
        b1.longitude,          -- Longitude of the business
        COUNT(*) as nearby_count,   -- Count of businesses within a 1km radius
        AVG(b2.stars) as cluster_rating -- Average rating of the businesses in the cluster
    FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b1
    -- Join the same table to compare each business with every other business
    JOIN `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b2
    ON 
        -- Use ST_DISTANCE to calculate the distance between two geographic points
        ST_DISTANCE(
            ST_GEOGPOINT(b1.longitude, b1.latitude), -- Geographical point of the first business
            ST_GEOGPOINT(b2.longitude, b2.latitude) -- Geographical point of the second business
        ) <= 1000  -- Filter for businesses within a 1km radius
    GROUP BY 
        -- Group by the unique attributes of the base business
        b1.business_id, 
        b1.name, 
        b1.latitude, 
        b1.longitude
    HAVING 
        -- Filter to include only businesses with at least 10 nearby businesses
        nearby_count >= 10
)

-- Final query to filter and order the results
SELECT *
FROM nearby_businesses
WHERE 
    -- Include only clusters with an average rating of 4.0 or higher
    cluster_rating >= 4.0
ORDER BY 
    -- Order by cluster rating in descending order first
    cluster_rating DESC,
    -- If ratings are tied, order by the number of nearby businesses in descending order
    nearby_count DESC;

```

#### 9. Category Performance Variation
**Question:** Find businesses with significant rating variations compared to their category average
**Business Value:** Performance benchmarking
```sql
-- Calculate average rating and standard deviation for each category
WITH category_stats AS (
    SELECT 
        category,                           -- Individual category extracted from the categories array
        AVG(stars) as category_avg,         -- Average rating for businesses in this category
        STDDEV(stars) as category_stddev    -- Standard deviation of ratings in this category
    FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
    -- Extract each category from the comma-separated categories column into separate rows
    CROSS JOIN UNNEST(SPLIT(categories, ", ")) as category
    GROUP BY category                       -- Group by each category to calculate statistics
    HAVING COUNT(*) >= 30                   -- Filter out categories with fewer than 30 businesses
)

-- Main query to identify businesses with unusual ratings in their category
SELECT 
    b.name,                                 -- Name of the business
    b.city,                                 -- City where the business is located
    b.stars as business_rating,             -- Rating of the business
    c.category,                             -- Category under consideration
    ROUND(c.category_avg, 2) as category_avg, -- Average rating for this category (rounded to 2 decimal places)
    -- Calculate how far the business rating is from the category mean in terms of standard deviations
    ROUND((b.stars - c.category_avg) / c.category_stddev, 2) as standard_deviations_from_mean
FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b
-- Extract individual categories for each business
CROSS JOIN UNNEST(SPLIT(b.categories, ", ")) as business_category
-- Join with category_stats to get average and standard deviation for each category
JOIN category_stats c ON business_category = c.category
-- Filter businesses with ratings far from the category mean (≥ 2 standard deviations)
WHERE ABS(b.stars - c.category_avg) >= 2 * c.category_stddev
-- Sort businesses by the number of standard deviations they deviate from their category mean, descending
ORDER BY ABS(b.stars - c.category_avg) / c.category_stddev DESC;

```

#### 10. Business Success Pattern Analysis
**Question:** Analyze business success patterns based on operating hours and location
**Business Value:** Success factor identification
```sql
WITH business_operating_hours AS (
    SELECT 
        business_id,
        TIME_DIFF(
            PARSE_TIME('%H:%M', SPLIT(hours.Monday, '-')[OFFSET(1)]),
            PARSE_TIME('%H:%M', SPLIT(hours.Monday, '-')[OFFSET(0)]),
            HOUR
        ) as monday_hours,
        city,
        state,
        stars,
        review_count
    FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
    WHERE hours.Monday IS NOT NULL
),
city_stats AS (
    SELECT 
        city,
        state,
        AVG(stars) as city_avg_rating,
        COUNT(*) as total_businesses
    FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
    GROUP BY city, state
    HAVING COUNT(*) >= 50
)
SELECT 
    b.city,
    b.state,
    ROUND(AVG(monday_hours), 1) as avg_operating_hours,
    ROUND(AVG(CASE WHEN stars > cs.city_avg_rating THEN 1.0 ELSE 0.0 END) * 100, 2) as pct_above_city_avg,
    ROUND(AVG(stars), 2) as avg_rating,
    ROUND(AVG(review_count), 0) as avg_reviews
FROM business_operating_hours b
JOIN city_stats cs ON b.city = cs.city AND b.state = cs.state
GROUP BY b.city, b.state
HAVING avg_operating_hours IS NOT NULL
ORDER BY pct_above_city_avg DESC
LIMIT 20;
```

## Best Practices

### Query Optimization
- Use appropriate indexes
- Filter early in the query
- Avoid SELECT *
- Use CTEs for better readability
- Consider query cost and performance

### Data Quality Considerations
- Handle NULL values appropriately
- Validate date ranges
- Consider timezone implications
- Check for data consistency

### Performance Tips
- Use partitioning where applicable
- Consider materialized views
- Optimize JOIN operations
- Use appropriate data types

## Implementation Guidelines

### Query Execution
1. Test queries with smaller datasets first
2. Monitor query performance
3. Use appropriate timeouts
4. Consider concurrent users

### Customization Options
1. Modify geographic distances
2. Adjust rating thresholds
3. Customize time windows
4. Adapt category filters

### Maintenance
1. Regular performance monitoring
2. Query optimization reviews
3. Data quality checks
4. Documentation updates

## Conclusion

This comprehensive analysis framework provides a robust foundation for analyzing the Yelp business dataset. The queries can be customized and combined to generate specific insights based on business requirements.

For additional support or custom query development, please contact the data team.
