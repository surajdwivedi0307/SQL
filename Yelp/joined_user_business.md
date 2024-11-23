# Yelp Data Analysis Queries Documentation

A collection of SQL queries for analyzing Yelp business and user data, focusing on user behavior, business performance, and social network impacts.

## Table of Contents
- [Top Reviewers Analysis](#top-reviewers-analysis)
- [Elite Users' Category Preferences](#elite-users-category-preferences)
- [Social Network Impact Analysis](#social-network-impact-analysis)
- [Geographic Elite User Distribution](#geographic-elite-user-distribution)
- [User Tenure Analysis](#user-tenure-analysis)
- [Category Performance by User Segments](#category-performance-by-user-segments)
- [Friend Network Category Analysis](#friend-network-category-analysis)
- [Geographic Clustering Analysis](#geographic-clustering-analysis)
- [Compliment Impact Analysis](#compliment-impact-analysis)
- [Business Hours Impact Analysis](#business-hours-impact-analysis)

## Top Reviewers Analysis

### Purpose
Identifies the most active reviewers on the platform and analyzes their rating patterns.

### Business Value
- Helps identify influential users who shape platform opinions
- Reveals potential biases in user rating behaviors
- Assists in understanding the correlation between review frequency and rating patterns

### Query
```sql
SELECT 
    u.name AS user_name,
    u.review_count,
    ROUND(u.average_stars, 2) AS user_avg_rating,
    ROUND(AVG(b.stars), 2) AS business_avg_rating,
    COUNT(DISTINCT b.business_id) AS businesses_reviewed
FROM `long-loop-442611-j5.Yelp_Business_Part1.user` u
JOIN `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b 
    ON b.stars = u.average_stars
WHERE u.review_count > 50
GROUP BY u.name, u.review_count, u.average_stars
ORDER BY u.review_count DESC
LIMIT 10;
```

## Elite Users' Category Preferences

### Purpose
Analyzes which business categories are most frequently reviewed by elite users.

### Business Value
- Understanding preferences of high-value users
- Identifying popular business categories among influential reviewers
- Guiding business category growth strategies

### Query
```sql
-- Step 1: Create a Common Table Expression (CTE) to process elite users and calculate the number of elite years
WITH elite_users AS (
    SELECT 
        user_id,  -- Select the user_id of each user
        name,     -- Select the name of each user
        
        -- Calculate the number of years a user has been elite (count of 4-digit elite years)
        ARRAY_LENGTH(
            ARRAY(
                -- Convert the elite years (concatenated string) into an array of 4-digit years
                -- REGEXP_REPLACE removes any decimal (e.g., '.0')
                SELECT SUBSTR(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', ''), pos, 4)  -- Extract the 4-digit years
                FROM UNNEST(GENERATE_ARRAY(1, LENGTH(REGEXP_REPLACE(CAST(elite AS STRING), r'\.0$', '')) - 3, 4)) AS pos
                -- UNNEST(GENERATE_ARRAY(...)) generates positions from which to extract each 4-digit substring
            )
        ) AS elite_years_count  -- Calculate the total number of elite years (length of the array)
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    -- Filter users who have at least one elite year (elite is not null and greater than 0)
    WHERE elite IS NOT NULL AND elite > 0
)

-- Step 2: Main query to calculate the average rating and review count per category
SELECT 
    category,  -- Select each category (from the business categories)
    COUNT(*) AS review_count,  -- Count the number of reviews for each category
    ROUND(AVG(b.stars), 2) AS avg_rating  -- Calculate the average rating for businesses in this category, rounded to 2 decimal places
FROM elite_users e  -- Use the elite_users CTE to bring in users' elite years count
JOIN `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b 
    -- Join the 'business_yelp' table on the condition that the number of elite years matches the business stars rating
    ON b.stars = e.elite_years_count  -- This implies that the business rating corresponds to the number of elite years of the user
CROSS JOIN UNNEST(SPLIT(b.categories, ', ')) AS category  -- Split the categories string into separate categories and cross join them
GROUP BY category  -- Group the results by category
HAVING review_count > 10  -- Filter categories that have more than 10 reviews
ORDER BY review_count DESC  -- Order the results by the number of reviews in descending order
LIMIT 15;  -- Limit the result to the top 15 categories

```

## Social Network Impact Analysis

### Purpose
Examines how users' social connections correlate with their rating behaviors.

### Business Value
- Understanding social influence on rating patterns
- Identifying network effects in user behavior
- Supporting community engagement strategies

### Query
```sql
WITH user_friends AS (
    SELECT 
        user_id,
        ARRAY_LENGTH(SPLIT(friends, ', ')) as friend_count,
        average_stars
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    WHERE friends IS NOT NULL
)
SELECT 
    CASE 
        WHEN friend_count < 10 THEN '0-9 friends'
        WHEN friend_count < 50 THEN '10-49 friends'
        WHEN friend_count < 100 THEN '50-99 friends'
        ELSE '100+ friends'
    END as friend_group,
    COUNT(DISTINCT uf.user_id) as user_count,
    ROUND(AVG(uf.average_stars), 2) as avg_user_rating,
    ROUND(AVG(b.stars), 2) as avg_business_rating
FROM user_friends uf
JOIN `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b 
    ON b.stars = uf.average_stars
GROUP BY friend_group
ORDER BY user_count DESC;
```

## Geographic Elite User Distribution

### Purpose
Maps the distribution of elite users across different cities and analyzes local business performance.

### Business Value
- Understanding geographic concentration of influential users
- Identifying high-performing markets
- Supporting market expansion decisions

### Query
```sql
WITH elite_user_cities AS (
    SELECT 
        b.city,
        b.state,
        COUNT(DISTINCT u.user_id) as elite_user_count,
        COUNT(DISTINCT b.business_id) as business_count,
        ROUND(AVG(b.stars), 2) as avg_business_rating
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user` u
    JOIN `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b 
        ON b.stars = u.average_stars
    WHERE u.elite IS NOT NULL AND u.elite > 0
    GROUP BY b.city, b.state
    HAVING business_count >= 10
)
SELECT 
    city,
    state,
    elite_user_count,
    business_count,
    avg_business_rating,
    ROUND(elite_user_count * 100.0 / business_count, 2) as elite_per_business_ratio
FROM elite_user_cities
ORDER BY elite_per_business_ratio DESC
LIMIT 20;
```

## User Tenure Analysis

### Purpose
Analyzes how user behavior and ratings change over time.

### Business Value
- Understanding user maturity impact on ratings
- Identifying long-term user trends
- Supporting user retention strategies

### Query
```sql
WITH user_tenure AS (
    SELECT 
        user_id,
        DATE_DIFF(CURRENT_DATE(), DATE(yelping_since), YEAR) as years_active,
        average_stars,
        review_count
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
)
SELECT 
    CASE 
        WHEN years_active < 2 THEN 'New Users (<2 years)'
        WHEN years_active < 5 THEN 'Established Users (2-5 years)'
        ELSE 'Veteran Users (5+ years)'
    END as user_segment,
    COUNT(DISTINCT ut.user_id) as user_count,
    ROUND(AVG(ut.average_stars), 2) as avg_user_rating,
    ROUND(AVG(b.stars), 2) as avg_business_rating,
    ROUND(AVG(ut.review_count), 0) as avg_reviews_per_user
FROM user_tenure ut
JOIN `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b 
    ON b.stars = ut.average_stars
GROUP BY user_segment
ORDER BY avg_reviews_per_user DESC;
```

## Category Performance by User Segments

### Purpose
Analyzes how different user segments interact with various business categories.

### Business Value
- Understanding category-specific user engagement
- Identifying high-performing user-category combinations
- Supporting targeted marketing strategies

### Query
```sql
WITH user_segments AS (
    SELECT 
        user_id,
        CASE 
            WHEN elite IS NOT NULL AND elite > 0 THEN 'Elite'
            WHEN review_count > 50 THEN 'Power User'
            WHEN fans > 10 THEN 'Popular'
            ELSE 'Regular'
        END as user_type,
        average_stars
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
)
SELECT 
    category,
    user_type,
    COUNT(DISTINCT us.user_id) as user_count,
    COUNT(DISTINCT b.business_id) as business_count,
    ROUND(AVG(b.stars), 2) as avg_business_rating,
    ROUND(AVG(us.average_stars), 2) as avg_user_rating
FROM user_segments us
JOIN `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b 
    ON b.stars = us.average_stars
CROSS JOIN UNNEST(SPLIT(b.categories, ', ')) as category
GROUP BY category, user_type
HAVING user_count >= 5
ORDER BY user_count DESC, category;
```

## Friend Network Category Analysis

### Purpose
Examines how social networks influence business category success.

### Business Value
- Understanding social influence on category performance
- Identifying viral business categories
- Supporting social marketing strategies

### Query
```sql
WITH user_friend_metrics AS (
    SELECT 
        user_id,
        ARRAY_LENGTH(SPLIT(friends, ', ')) as friend_count,
        average_stars,
        review_count
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
    WHERE friends IS NOT NULL
)
SELECT 
    category,
    ROUND(AVG(ufm.friend_count), 0) as avg_friends_per_user,
    ROUND(AVG(b.stars), 2) as avg_business_rating,
    ROUND(AVG(ufm.average_stars), 2) as avg_user_rating,
    COUNT(DISTINCT b.business_id) as business_count,
    ROUND(AVG(b.review_count), 0) as avg_reviews_per_business
FROM user_friend_metrics ufm
JOIN `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b 
    ON b.stars = ufm.average_stars
CROSS JOIN UNNEST(SPLIT(b.categories, ', ')) as category
GROUP BY category
HAVING business_count >= 10
ORDER BY avg_friends_per_user DESC;
```

## Geographic Clustering Analysis

### Purpose
Analyzes the spatial distribution of elite users and highly-rated businesses.

### Business Value
- Understanding geographic clustering effects
- Identifying high-potential locations
- Supporting expansion strategies

### Query
```sql
WITH elite_user_locations AS (
    SELECT 
        u.user_id,
        b.latitude,
        b.longitude,
        b.city,
        b.state,
        u.average_stars,
        b.stars as business_stars
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user` u
    JOIN `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b 
        ON b.stars = u.average_stars
    WHERE u.elite IS NOT NULL AND u.elite > 0
)
SELECT 
    city,
    state,
    COUNT(DISTINCT user_id) as elite_users,
    ROUND(AVG(average_stars), 2) as avg_user_rating,
    ROUND(AVG(business_stars), 2) as avg_business_rating,
    ST_DISTANCE(
        ST_GEOGPOINT(MIN(longitude), MIN(latitude)),
        ST_GEOGPOINT(MAX(longitude), MAX(latitude))
    ) as cluster_radius_meters
FROM elite_user_locations
GROUP BY city, state
HAVING elite_users >= 5
ORDER BY elite_users DESC;
```

## Compliment Impact Analysis

### Purpose
Examines how user compliments correlate with business ratings.

### Business Value
- Understanding engagement metrics impact
- Identifying valuable user behaviors
- Supporting user recognition strategies

### Query
```sql
WITH user_compliments AS (
    SELECT 
        user_id,
        average_stars,
        (compliment_hot + compliment_more + compliment_profile + 
         compliment_cute + compliment_list + compliment_note + 
         compliment_plain + compliment_cool + compliment_funny + 
         compliment_writer + compliment_photos) as total_compliments
    FROM `long-loop-442611-j5.Yelp_Business_Part1.user`
)
SELECT 
    CASE 
        WHEN total_compliments = 0 THEN 'No compliments'
        WHEN total_compliments < 10 THEN '1-9 compliments'
        WHEN total_compliments < 50 THEN '10-49 compliments'
        WHEN total_compliments < 100 THEN '50-99 compliments'
        ELSE '100+ compliments'
    END as compliment_range,
    COUNT(DISTINCT uc.user_id) as user_count,
    ROUND(AVG(uc.average_stars), 2) as avg_user_rating,
    ROUND(AVG(b.stars), 2) as avg_business_rating,
    COUNT(DISTINCT b.business_id) as businesses_reviewed
FROM user_compliments uc
JOIN `long-loop-442611-j5.Yelp_Business_Part1.business_yelp` b 
    ON b.stars = uc.average_stars
GROUP BY compliment_range
ORDER BY user_count DESC;
```

## Contributing
Feel free to submit issues and enhancement requests.

## License
This project is licensed under the MIT License.
