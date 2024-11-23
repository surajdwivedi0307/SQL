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
WITH user_tenure AS (
    SELECT 
        EXTRACT(YEAR FROM CURRENT_DATE()) - EXTRACT(YEAR FROM DATE(yelping_since)) as years_on_platform,
        COUNT(*) as user_count
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    GROUP BY years_on_platform
)
SELECT 
    years_on_platform,
    user_count,
    ROUND(user_count * 100.0 / SUM(user_count) OVER(), 2) as percentage
FROM user_tenure
ORDER BY years_on_platform;
```

### Intermediate Level Queries

#### 3. Elite User Analysis
**Question:** What characteristics define elite users compared to regular users?
**Business Value:** Understanding valuable user segments
```sql
WITH user_segments AS (
    SELECT 
        CASE WHEN ARRAY_LENGTH(elite) > 0 THEN 'Elite' ELSE 'Regular' END as user_type,
        COUNT(*) as user_count,
        AVG(review_count) as avg_reviews,
        AVG(fans) as avg_fans,
        AVG(average_stars) as avg_rating,
        AVG(useful + funny + cool) as avg_vote_count,
        AVG(compliment_hot + compliment_more + compliment_profile + 
            compliment_cute + compliment_list + compliment_note + 
            compliment_plain + compliment_cool + compliment_funny + 
            compliment_writer + compliment_photos) as avg_compliments
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    GROUP BY user_type
)
SELECT 
    *,
    ROUND(user_count * 100.0 / SUM(user_count) OVER(), 2) as percentage_of_users
FROM user_segments;
```

#### 4. Social Network Analysis
**Question:** Who are the most connected users and what's their impact?
**Business Value:** Identifying influencers and network effects
```sql
SELECT 
    name,
    ARRAY_LENGTH(friends) as friend_count,
    fans,
    review_count,
    average_stars,
    useful + funny + cool as total_votes_given,
    compliment_hot + compliment_more + compliment_profile + 
    compliment_cute + compliment_list + compliment_note + 
    compliment_plain + compliment_cool + compliment_funny + 
    compliment_writer + compliment_photos as total_compliments_received
FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
ORDER BY friend_count DESC
LIMIT 20;
```

### Advanced Level Queries

#### 5. User Behavior Pattern Analysis
**Question:** What's the correlation between user activity and reception?
**Business Value:** Understanding engagement drivers
```sql
WITH user_metrics AS (
    SELECT 
        review_count,
        fans,
        useful + funny + cool as total_votes_given,
        compliment_hot + compliment_more + compliment_profile + 
        compliment_cute + compliment_list + compliment_note + 
        compliment_plain + compliment_cool + compliment_funny + 
        compliment_writer + compliment_photos as total_compliments,
        average_stars,
        ARRAY_LENGTH(friends) as friend_count,
        ARRAY_LENGTH(elite) as elite_years
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
)
SELECT 
    CORR(review_count, fans) as review_fan_correlation,
    CORR(review_count, total_compliments) as review_compliment_correlation,
    CORR(friend_count, fans) as friend_fan_correlation,
    CORR(elite_years, average_stars) as elite_rating_correlation,
    CORR(total_votes_given, total_compliments) as give_receive_correlation
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
