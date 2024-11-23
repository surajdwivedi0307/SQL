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
**Question:** Find all Mexican restaurants in San Francisco with a rating of 4.0 or higher that are currently open
**Business Value:** Competitive analysis
```sql
SELECT 
    name,
    stars,
    review_count,
    address
FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
WHERE city = 'San Francisco' 
    AND 'Mexican' IN UNNEST(categories)
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
WITH city_businesses AS (
    SELECT city, COUNT(*) as total
    FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
    GROUP BY city
    HAVING total > 1000
)
SELECT 
    b.city,
    category,
    COUNT(*) as category_count,
    ROUND(COUNT(*) * 100.0 / cb.total, 2) as percentage
FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b
CROSS JOIN UNNEST(categories) as category
JOIN city_businesses cb ON b.city = cb.city
GROUP BY b.city, category, cb.total
ORDER BY b.city, category_count DESC;
```

#### 5. Operating Hours Analysis
**Question:** Which businesses are open the longest hours during the week?
**Business Value:** Operational patterns identification
```sql
WITH business_hours AS (
    SELECT 
        name,
        city,
        PARSE_TIME('%H:%M', SPLIT(hours.Monday, '-')[OFFSET(1)]) as closing_time,
        PARSE_TIME('%H:%M', SPLIT(hours.Monday, '-')[OFFSET(0)]) as opening_time,
        TIME_DIFF(
            PARSE_TIME('%H:%M', SPLIT(hours.Monday, '-')[OFFSET(1)]),
            PARSE_TIME('%H:%M', SPLIT(hours.Monday, '-')[OFFSET(0)]),
            HOUR
        ) as hours_open
    FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
    WHERE hours.Monday IS NOT NULL
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
WITH weekday_hours AS (
    SELECT 
        business_id,
        name,
        TIME_DIFF(
            PARSE_TIME('%H:%M', SPLIT(hours.Monday, '-')[OFFSET(1)]),
            PARSE_TIME('%H:%M', SPLIT(hours.Monday, '-')[OFFSET(0)]),
            MINUTE
        ) as monday_minutes,
        TIME_DIFF(
            PARSE_TIME('%H:%M', SPLIT(hours.Saturday, '-')[OFFSET(1)]),
            PARSE_TIME('%H:%M', SPLIT(hours.Saturday, '-')[OFFSET(0)]),
            MINUTE
        ) as saturday_minutes
    FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
    WHERE hours.Monday IS NOT NULL AND hours.Saturday IS NOT NULL
)
SELECT 
    name,
    monday_minutes/60 as weekday_hours,
    saturday_minutes/60 as weekend_hours,
    ROUND((saturday_minutes - monday_minutes)*100.0/monday_minutes, 2) as percent_difference
FROM weekday_hours
WHERE saturday_minutes > monday_minutes
ORDER BY percent_difference DESC
LIMIT 15;
```

#### 7. Geographic Cluster Analysis
**Question:** Identify geographic clusters of highly-rated businesses
**Business Value:** Location strategy optimization
```sql
WITH nearby_businesses AS (
    SELECT 
        b1.business_id,
        b1.name,
        b1.latitude,
        b1.longitude,
        COUNT(*) as nearby_count,
        AVG(b2.stars) as cluster_rating
    FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b1
    JOIN `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b2
    ON ST_DISTANCE(
        ST_GEOGPOINT(b1.longitude, b1.latitude),
        ST_GEOGPOINT(b2.longitude, b2.latitude)
    ) <= 1000  -- 1km radius
    GROUP BY b1.business_id, b1.name, b1.latitude, b1.longitude
    HAVING nearby_count >= 10
)
SELECT *
FROM nearby_businesses
WHERE cluster_rating >= 4.0
ORDER BY cluster_rating DESC, nearby_count DESC;
```

#### 8. Parking Impact Analysis
**Question:** Analyze the correlation between parking options and ratings
**Business Value:** Facility planning insights
```sql
WITH parking_options AS (
    SELECT 
        business_id,
        JSON_EXTRACT_SCALAR(attributes.BusinessParking, '$.garage') = 'true' as has_garage,
        JSON_EXTRACT_SCALAR(attributes.BusinessParking, '$.street') = 'true' as has_street,
        JSON_EXTRACT_SCALAR(attributes.BusinessParking, '$.lot') = 'true' as has_lot,
        JSON_EXTRACT_SCALAR(attributes.BusinessParking, '$.valet') = 'true' as has_valet
    FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
    WHERE attributes.BusinessParking IS NOT NULL
)
SELECT 
    CASE 
        WHEN has_garage THEN 'Garage'
        WHEN has_street THEN 'Street'
        WHEN has_lot THEN 'Lot'
        WHEN has_valet THEN 'Valet'
        ELSE 'No Parking'
    END as parking_type,
    COUNT(*) as business_count,
    ROUND(AVG(b.stars), 2) as avg_rating,
    ROUND(AVG(b.review_count), 0) as avg_reviews
FROM parking_options p
JOIN `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b ON p.business_id = b.business_id
GROUP BY parking_type
ORDER BY avg_rating DESC;
```

#### 9. Category Performance Variation
**Question:** Find businesses with significant rating variations compared to their category average
**Business Value:** Performance benchmarking
```sql
WITH category_stats AS (
    SELECT 
        category,
        AVG(stars) as category_avg,
        STDDEV(stars) as category_stddev
    FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp`
    CROSS JOIN UNNEST(categories) as category
    GROUP BY category
    HAVING COUNT(*) >= 30
)
SELECT 
    b.name,
    b.city,
    b.stars as business_rating,
    c.category,
    ROUND(c.category_avg, 2) as category_avg,
    ROUND((b.stars - c.category_avg)/c.category_stddev, 2) as standard_deviations_from_mean
FROM `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b
CROSS JOIN UNNEST(categories) as business_category
JOIN category_stats c ON business_category = c.category
WHERE ABS(b.stars - c.category_avg) >= 2*c.category_stddev
ORDER BY ABS(b.stars - c.category_avg)/c.category_stddev DESC;
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