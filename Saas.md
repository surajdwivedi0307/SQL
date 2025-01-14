Here are 15 SQL questions progressively moving from basic to advanced, along with their BigQuery queries. Each question will help you explore the table `long-loop-442611-j5.saas.saas_base`

---

### **Basic Questions**
1. **Retrieve all columns and rows from the table.**
   ```sql
   SELECT * 
   FROM `long-loop-442611-j5.saas.saas_base`;
   ```

2. **Count the total number of accounts.**
   ```sql
   SELECT COUNT(DISTINCT Account_ID) AS total_accounts
   FROM `long-loop-442611-j5.saas.saas_base`;
   ```

3. **Find all distinct `Pricing Model` values in the table.**
   ```sql
   SELECT DISTINCT Pricing_Model
   FROM `long-loop-442611-j5.saas.saas_base`;
   ```

---

### **1. Match Account IDs Starting with the Substring**
```sql
SELECT COUNT(Account_ID) as AccountNos, Product
FROM `long-loop-442611-j5.saas.saas_base`
WHERE Account_ID LIKE '562c88ad%'
GROUP BY Product;
```
- Matches `Account_ID` values that **start with** `562c88ad`.

---

### **2. Match Account IDs Ending with the Substring**
```sql
SELECT Account_ID, Quota_Used, MRR
FROM `long-loop-442611-j5.saas.saas_base`
WHERE Account_ID LIKE '%dc6af4';
```
- Matches `Account_ID` values that **end with** `dc6af4`.

---

### **3. Match Account IDs Containing the Substring**
```sql
SELECT DISTINCT Account_ID
FROM `long-loop-442611-j5.saas.saas_base`
WHERE Account_ID LIKE '%dc6af4%';
```
- Matches `Account_ID` values that **contain** `452b`.

---

### **4. Case-Insensitive Match for Similar IDs**
```sql
SELECT Account_ID,Month, Quota_Used, MRR
FROM `long-loop-442611-j5.saas.saas_base`
WHERE LOWER(Account_ID) LIKE LOWER('%562C88AD%');
```
- Performs a **case-insensitive match** by converting both the `Account_ID` and the substring to lowercase.

---

### **5. Match Using a Regular Expression**
```sql
SELECT Account_ID, Quota_Used, MRR
FROM `long-loop-442611-j5.saas.saas_base`
WHERE REGEXP_CONTAINS(Account_ID, r'562c88ad');
```
- Matches `Account_ID` values using a **regular expression**. This approach is more flexible for complex patterns.

---

### **6. Match Multiple Similar IDs**
```sql
SELECT DISTINCT(Account_ID)
FROM `long-loop-442611-j5.saas.saas_base`
WHERE Account_ID IN ('562c88ad-81fb-452b-b93b-dc6af4773752','dcaa0f95-1220-4261-a5ad-1562cbdae22f');
```
- Matches rows where the `Account_ID` is **one of several predefined IDs**.

---

### **7. Exclude Specific Similar IDs**
```sql
SELECT DISTINCT(Account_ID), Quota_Used, MRR
FROM `long-loop-442611-j5.saas.saas_base`
WHERE Account_ID LIKE '%562c88ad%' AND Account_ID NOT IN ('562c88ad-81fb-452b-b93b-dc6af4');
```
- Matches rows **similar to** `562c88ad` but **excludes specific IDs**.

---

### **8. Find Account IDs With a Specific Length**
```sql
SELECT Account_ID, Quota_Used, MRR
FROM `long-loop-442611-j5.saas.saas_base`
WHERE LENGTH(Account_ID) = 36 AND Account_ID LIKE '%562c88ad%';
```
- Matches rows with a specific `Account_ID` length (**e.g., 36 characters**) and similarity.

---

### **9. Use Wildcard Matching for Variations**
```sql
SELECT DISTINCT(Account_ID), MRR
FROM `long-loop-442611-j5.saas.saas_base`
WHERE Account_ID LIKE '562c%-%-%'
ORDER BY MRR DESC;
```

### **10.Group by `Country` and calculate the total MRR for each country.**
   ```sql
SELECT Country, round(SUM(MRR),0) AS total_mrr
FROM `long-loop-442611-j5.saas.saas_base`
GROUP BY Country
ORDER BY total_mrr DESC;
   ```

### **11.Use a `CASE` statement to classify accounts based on `MRR`.**
   ```sql
SELECT 
    Account_ID,Country,
    MRR, 
    CASE 
        WHEN MRR < 500 THEN 'Small'
        WHEN MRR BETWEEN 500 AND 2000 THEN 'Medium'
        WHEN MRR BETWEEN 2000 AND 5000 THEN 'Large'
        ELSE 'Very Large'
    END AS account_size
FROM `long-loop-442611-j5.saas.saas_base`;
   ```

---

### **Advanced Questions**
7. **Use a CTE (Common Table Expression) to calculate the average `MRR` per `Country` and find countries with `MRR` above 5000.**
   ```sql
   WITH country_mrr AS (
    SELECT Country, AVG(MRR) AS avg_mrr
    FROM `long-loop-442611-j5.saas.saas_base`
    GROUP BY Country
   )
   SELECT 
   Country,
   avg_mrr,
    CASE 
        WHEN avg_mrr < 500 THEN 'Small'
        WHEN avg_mrr BETWEEN 500 AND 2000 THEN 'Medium'
        ELSE 'Large'
   END AS country_mrr_bucket
   FROM country_mrr
   WHERE avg_mrr > 50;
   ```

8. **Join the table with itself to compare `MRR` for accounts with the same `Industry` but different `Geography`.**
   ```sql
    ------
   ```

9. **Use a subquery to find accounts with an `MRR` above the average `MRR` for their `Contract_Type`.**
   ```sql
    SELECT 
    main.Account_ID, 
    main.Country, 
    main.max_mrr, 
    country_avg.avg_mrr
    FROM 
     (
        SELECT 
            Account_ID, 
            Country, 
            MAX(MRR) AS max_mrr
        FROM 
            `long-loop-442611-j5.saas.saas_base`
        GROUP BY 
            Account_ID, Country
      ) main
    JOIN 
    (
        SELECT 
            Country, 
            AVG(MRR) AS avg_mrr
        FROM 
            `long-loop-442611-j5.saas.saas_base`
        GROUP BY 
            Country
    ) country_avg
    ON 
      main.Country = country_avg.Country
     WHERE 
     main.max_mrr > country_avg.avg_mrr;

   ```





Here are some **very basic** to **advanced** examples of **window functions**:

---

### **Basic Window Function Examples**

1. **Simple Window Function: Using `ROW_NUMBER()` to assign a unique row number to each row in the table.**
   ```sql
    SELECT 
      Account_ID, 
      MRR,
      ROW_NUMBER() OVER (ORDER BY MRR DESC) AS mrr_ranked
    FROM `long-loop-442611-j5.saas.saas_base`;
   ```
   - This query assigns a sequential number to each row based on the `MRR` in descending order.

2. **Window Function with Partition: Using `RANK()` to rank accounts within each `Country` based on `ARR`.**
   ```sql
     WITH avg_arr_per_country AS (
      SELECT 
        Account_ID, 
        Country, 
        AVG(ARR) AS avg_arr
      FROM `long-loop-442611-j5.saas.saas_base`
     GROUP BY Account_ID, Country
     )
     SELECT 
       Account_ID, 
       Country, 
       avg_arr, 
     RANK() OVER (PARTITION BY Country ORDER BY avg_arr DESC) AS rank_within_country
    FROM avg_arr_per_country;

   ```

2.B ** Window Function with Partition: Using RANK() to rank accounts within each `Country` based on `ARR`.**

```sql
   WITH avg_arr_per_country AS (
     SELECT 
       Account_ID, 
       Country, 
       AVG(ARR) AS avg_arr
     FROM `long-loop-442611-j5.saas.saas_base`
     GROUP BY Account_ID, Country
    )
 SELECT 
    Account_ID, 
    Country, 
    avg_arr, 
    rank_within_country
   FROM (
          SELECT 
             Account_ID, 
             Country, 
             avg_arr, 
             RANK() OVER (PARTITION BY Country ORDER BY avg_arr DESC) AS rank_within_country
          FROM avg_arr_per_country
        )
   WHERE rank_within_country < 3
   ORDER BY Country;


```
---

### **Intermediate Window Function Examples**

3. **Calculating the running total: Using `SUM()` with a window function to get a cumulative sum of `MRR` per `Geography`.**
   ```sql
   SELECT 
       Geography, 
       Account_ID, 
       MRR, 
       SUM(MRR) OVER (PARTITION BY Geography ORDER BY Account_ID) AS cumulative_mrr
   FROM `long-loop-442611-j5.saas.saas_base`;
   ```
   - This query calculates the cumulative sum of `MRR` per `Geography` as we move down the list of `Account_ID`.

4. **Window function with `LEAD()` and `LAG()`: Compare each account’s `MRR` with the next and previous account in the `Month` using `LEAD()` and `LAG()`.**
   ```sql
    SELECT 
        Account_ID, 
        Industry,
        Month, 
        MRR, 
        LEAD(MRR) OVER (PARTITION BY Account_ID ORDER BY Month) AS next_mrr,
        LAG(MRR) OVER (PARTITION BY Account_ID ORDER BY Month) AS previous_mrr
     FROM `long-loop-442611-j5.saas.saas_base`
     Where Account_ID='01732900-b003-4cfb-9be4-40fde466ee5c'
     ORDER BY Month;
   ```


   ```sql
    SELECT 
       Account_ID, 
       Industry,
       Month, 
       MRR, 
       LEAD(MRR) OVER (PARTITION BY Account_ID ORDER BY Month) AS next_mrr,
       LAG(MRR) OVER (PARTITION BY Account_ID ORDER BY Month) AS previous_mrr,
       FIRST_VALUE(Month) OVER (PARTITION BY Account_ID ORDER BY Month ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS 
         first_month,
       LAST_VALUE(Month) OVER (PARTITION BY Account_ID ORDER BY Month ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS 
         last_month
    FROM `long-loop-442611-j5.saas.saas_base`
    WHERE Account_ID='01732900-b003-4cfb-9be4-40fde466ee5c'
    ORDER BY Month;

   ```

---

### **Advanced Window Function Examples**

5. **Window function with `NTILE()`: Split accounts into quartiles based on `ARR`.**
   ```sql
   SELECT 
       Account_ID, 
       ARR, 
       NTILE(4) OVER (ORDER BY ARR DESC) AS quartile
   FROM `long-loop-442611-j5.saas.saas_base`;
   ```
   - This query divides all accounts into 4 quartiles based on their `ARR`, with the highest `ARR` in the first quartile.

6. **Calculating a moving average: Using `AVG()` with a window function to calculate a 3-month moving average of `MRR`.**
   ```sql
     SELECT 
       Account_ID, 
       Month, 
       MRR,
       AVG(MRR) OVER (PARTITION BY Account_ID ORDER BY Month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg,
       LEAD(MRR) OVER (PARTITION BY Account_ID ORDER BY Month) AS next_mrr,
       LAG(MRR) OVER (PARTITION BY Account_ID ORDER BY Month) AS previous_mrr,
       FIRST_VALUE(Month) OVER (PARTITION BY Account_ID ORDER BY Month ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS 
         first_month,
       LAST_VALUE(Month) OVER (PARTITION BY Account_ID ORDER BY Month ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS 
         last_month
     FROM `long-loop-442611-j5.saas.saas_base`
     WHERE Account_ID='01732900-b003-4cfb-9be4-40fde466ee5c'
     ORDER BY Month;
   ```
   - This query calculates the moving average of `MRR` over the past 3 months for each `Account_ID`.

7. **Finding the first and last `MRR` for each `Account ID` using `FIRST_VALUE()` and `LAST_VALUE()`.**
   ```sql
     SELECT 
         Account_ID, 
         Country, 
         MRR, 
         FIRST_VALUE(MRR) OVER (PARTITION BY Account_ID ORDER BY Month) AS first_mrr,
         LAST_VALUE(MRR) OVER (PARTITION BY Account_ID ORDER BY Month ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING) AS last_mrr
     FROM `long-loop-442611-j5.saas.saas_base`
     WHERE Account_ID='2754d14a-662c-4ec1-a4d2-c1cb449f133d'
    ORDER BY Month;
   ```
   - This query retrieves the first and last `MRR` for each `Country`, based on the `Month`.

8. **Calculating the percentage difference between `MRR` and the average `MRR` for each `Country` using window functions.**
   ```sql
      SELECT 
       Account_ID,
       Month,
       Country,
       Industry,
       MRR,
       AVG(MRR) OVER (PARTITION BY Country, Industry) AS country_ind_avg_mrr,
       CASE 
          WHEN MRR = 0 THEN NULL
          ELSE ((MRR - AVG(MRR) OVER (PARTITION BY Country, Industry)) / MRR) * 100
      END AS pct_diff_from_avg
     FROM `long-loop-442611-j5.saas.saas_base`
     ORDER BY Account_ID;


   ```
   - This query calculates the percentage difference of each account's `MRR` from the average `MRR` for its `Country`.

---

These examples gradually move from simple row-based window functions to more complex aggregate window functions with partitions, moving averages, and first/last value calculations. Let me know if you'd like further details or modifications!






10. **Use a window function to calculate the cumulative MRR for each `Geography`.**
   ```sql
   SELECT 
       Geography, 
       Account_ID, 
       MRR, 
       SUM(MRR) OVER (PARTITION BY Geography ORDER BY Account_ID) AS cumulative_mrr
   FROM `long-loop-442611-j5.saas.saas_base`;
   ```

11. **Rank accounts within each `Country` by `ARR` using a window function.**
   ```sql
             WITH arr_account AS (
         SELECT 
             Account_ID, 
              Country, 
            SUM(ARR) AS ARR_sum
     FROM `long-loop-442611-j5.saas.saas_base`
    GROUP BY Account_ID, Country
    --HAVING Account_ID = '1a5d5493-8017-4c24-8561-32443db61afb'
)

SELECT 
    Account_ID, 
    Country, 
    ARR_sum, 
    RANK() OVER (PARTITION BY Country ORDER BY ARR_sum DESC) AS rank_within_country
FROM arr_account
ORDER BY Country;

   ```

12. **Find the `Lead` and `Lag` MRR values for each account within the same `Industry`.**
   ```sql
   SELECT 
       Account_ID, 
       Industry, 
       MRR, 
       LEAD(MRR) OVER (PARTITION BY Industry ORDER BY Account_ID) AS next_mrr,
       LAG(MRR) OVER (PARTITION BY Industry ORDER BY Account_ID) AS previous_mrr
   FROM `long-loop-442611-j5.saas.saas_base`;
   ```

13. **Generate row numbers for accounts within each `Contract_Type`.**
   ```sql
   SELECT 
       Account_ID, 
       Contract_Type, 
       ROW_NUMBER() OVER (PARTITION BY Contract_Type ORDER BY MRR DESC) AS row_no
   FROM `long-loop-442611-j5.saas.saas_base`;
   ```

14. **One Final query`.**
   ```sql
WITH country_month_arr AS (
    SELECT 
        Account_ID,
        Country,
        MRR,
        PARSE_DATE('%Y-%m', Month) AS parsed_month,
        EXTRACT(YEAR FROM PARSE_DATE('%Y-%m', Month)) AS year,
        LEAD(MRR) OVER (PARTITION BY Account_ID ORDER BY PARSE_DATE('%Y-%m', Month)) AS next_month_revenue,
        FIRST_VALUE(MRR) OVER (PARTITION BY Account_ID ORDER BY PARSE_DATE('%Y-%m', Month)) AS first_mrr,
        LAST_VALUE(MRR) OVER (PARTITION BY Account_ID ORDER BY PARSE_DATE('%Y-%m', Month) ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_mrr,
        FIRST_VALUE(PARSE_DATE('%Y-%m', Month)) OVER (PARTITION BY Account_ID ORDER BY PARSE_DATE('%Y-%m', Month)) AS first_month,
        LAST_VALUE(PARSE_DATE('%Y-%m', Month)) OVER (PARTITION BY Account_ID ORDER BY PARSE_DATE('%Y-%m', Month) ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_month,
        RANK() OVER (PARTITION BY Account_ID ORDER BY MRR DESC) AS mrr_rank,
        CASE 
           WHEN RANK() OVER (PARTITION BY Account_ID ORDER BY MRR DESC) = 1 THEN PARSE_DATE('%Y-%m', Month)
           ELSE NULL 
        END AS highest_mrr_month 
    FROM 
        `long-loop-442611-j5.saas.saas_base` 
    GROUP BY 
        Account_ID, Country, Month, MRR
    --Having Account_ID='03371a94-3e25-407b-a56e-fdc50abb6c11'
),
country_year_rev AS (
    SELECT 
        Country,
        year,
        SUM(MRR) AS country_rev,  -- Yearly revenue for each country
        MAX(SUM(MRR)) OVER () AS max_rev,
        MIN(SUM(MRR)) OVER () AS min_rev,
        CASE
            WHEN SUM(MRR) >= 0.75 * MAX(SUM(MRR)) OVER () THEN 'High Revenue'
            WHEN SUM(MRR) >= 0.50 * MAX(SUM(MRR)) OVER () THEN 'Medium Revenue'
            ELSE 'Low Revenue'
        END AS revenue_bucket
    FROM 
        country_month_arr
    GROUP BY
        Country, year
),

final_table AS (
SELECT 
    cma.Account_ID,
    cma.MRR,
    cma.Country,
    cma.parsed_month,
    cyr.year,
    cyr.country_rev,
    cyr.revenue_bucket,
    cma.next_month_revenue,
    cma.first_month,
    cma.last_month,
    cma.first_mrr,
    cma.last_mrr,
    cma.highest_mrr_month,
    DATE_DIFF(cma.last_month, cma.first_month, MONTH) AS account_tenure_months,  -- Account tenure in months
    cma.mrr_rank  -- Rank based on MRR per Account_ID for each month
FROM 
    country_month_arr cma
JOIN 
    country_year_rev cyr
ON 
    cma.Country = cyr.Country
    AND EXTRACT(YEAR FROM cma.parsed_month) = cyr.year
)
select* from final_table
where mrr_rank=1 AND Country='Belgium'
   ```

15. **Above Final query using Subquery`.**
   ```sql

SELECT 
    cma.Account_ID,
    cma.MRR,
    cma.Country,
    cma.parsed_month,
    cyr.year,
    cyr.country_rev,
    cyr.revenue_bucket,
    cma.next_month_revenue,
    cma.first_month,
    cma.last_month,
    cma.first_mrr,
    cma.last_mrr,
    cma.highest_mrr_month,
    DATE_DIFF(cma.last_month, cma.first_month, MONTH) AS account_tenure_months,
    cma.mrr_rank
FROM 
    (
        SELECT 
            Account_ID,
            Country,
            MRR,
            PARSE_DATE('%Y-%m', Month) AS parsed_month,
            EXTRACT(YEAR FROM PARSE_DATE('%Y-%m', Month)) AS year,
            LEAD(MRR) OVER (PARTITION BY Account_ID ORDER BY PARSE_DATE('%Y-%m', Month)) AS next_month_revenue,
            FIRST_VALUE(MRR) OVER (PARTITION BY Account_ID ORDER BY PARSE_DATE('%Y-%m', Month)) AS first_mrr,
            LAST_VALUE(MRR) OVER (PARTITION BY Account_ID ORDER BY PARSE_DATE('%Y-%m', Month) ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_mrr,
            FIRST_VALUE(PARSE_DATE('%Y-%m', Month)) OVER (PARTITION BY Account_ID ORDER BY PARSE_DATE('%Y-%m', Month)) AS first_month,
            LAST_VALUE(PARSE_DATE('%Y-%m', Month)) OVER (PARTITION BY Account_ID ORDER BY PARSE_DATE('%Y-%m', Month) ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_month,
            RANK() OVER (PARTITION BY Account_ID ORDER BY MRR DESC) AS mrr_rank,
            CASE 
                WHEN RANK() OVER (PARTITION BY Account_ID ORDER BY MRR DESC) = 1 THEN PARSE_DATE('%Y-%m', Month)
                ELSE NULL 
            END AS highest_mrr_month
        FROM 
            `long-loop-442611-j5.saas.saas_base`
        GROUP BY 
            Account_ID, Country, Month, MRR
    ) AS cma
JOIN 
    (
        SELECT 
            Country,
            year,
            SUM(MRR) AS country_rev,
            MAX(SUM(MRR)) OVER () AS max_rev,
            MIN(SUM(MRR)) OVER () AS min_rev,
            CASE
                WHEN SUM(MRR) >= 0.75 * MAX(SUM(MRR)) OVER () THEN 'High Revenue'
                WHEN SUM(MRR) >= 0.50 * MAX(SUM(MRR)) OVER () THEN 'Medium Revenue'
                ELSE 'Low Revenue'
            END AS revenue_bucket
        FROM 
            (
                SELECT 
                    Account_ID,
                    Country,
                    MRR,
                    PARSE_DATE('%Y-%m', Month) AS parsed_month,
                    EXTRACT(YEAR FROM PARSE_DATE('%Y-%m', Month)) AS year
                FROM 
                    `long-loop-442611-j5.saas.saas_base`
                GROUP BY 
                    Account_ID, Country, Month, MRR
            ) AS country_month_arr
        GROUP BY 
            Country, year
    ) AS cyr
ON 
    cma.Country = cyr.Country
    AND EXTRACT(YEAR FROM cma.parsed_month) = cyr.year
WHERE 
    cma.mrr_rank = 1 AND cma.Country = 'Belgium'

   ```
### **. Other types of joins (LEFT JOIN, RIGHT JOIN, FULL OUTER JOIN)**
### **. Aggregation without grouping (e.g., averages)**
### **. UNION / UNION ALL**
### **. Window functions without partitioning (e.g., ROW_NUMBER() without PARTITION BY)**
### **. Data modification (INSERT, UPDATE, DELETE)g**
### **. Transactions, locking, and commit/rollback**
### **. Foreign keys, constraints, and indexes**
### **. UNION / UNION ALL**
### **. Full-text search**
### **. Triggers**
### **. TStored procedures or functionsk**
### **. Recursive queries**
### **. More complex CASE logic**

