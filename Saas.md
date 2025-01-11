Here are 15 SQL questions progressively moving from basic to advanced concepts, along with their BigQuery queries. Each question will help you explore the table `long-loop-442611-j5.saas.saas_base` while teaching SQL concepts.

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

### **Intermediate Questions**
4. **Filter all accounts with `Quota_Used` greater than 80.**
   ```sql
  Here are several variants of your query, depending on the pattern or similarity you are looking for:

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

5. **Group by `Country` and calculate the total MRR for each country.**
   ```sql
   SELECT Country, SUM(MRR) AS total_mrr
   FROM `long-loop-442611-j5.saas.saas_base`
   GROUP BY Country
   ORDER BY total_mrr DESC;
   ```

6. **Use a `CASE` statement to classify accounts based on `MRR`.**
   ```sql
   SELECT 
       Account_ID, 
       MRR, 
       CASE 
           WHEN MRR < 500 THEN 'Small'
           WHEN MRR BETWEEN 500 AND 2000 THEN 'Medium'
           ELSE 'Large'
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
   SELECT Country, avg_mrr
   FROM country_mrr
   WHERE avg_mrr > 5000;
   ```

8. **Join the table with itself to compare `MRR` for accounts with the same `Industry` but different `Geography`.**
   ```sql
   SELECT 
       a.Account_ID AS account_1, 
       b.Account_ID AS account_2, 
       a.Industry, 
       a.Geography AS geography_1, 
       b.Geography AS geography_2, 
       a.MRR AS mrr_1, 
       b.MRR AS mrr_2
   FROM `long-loop-442611-j5.saas.saas_base` a
   JOIN `long-loop-442611-j5.saas.saas_base` b
   ON a.Industry = b.Industry AND a.Geography != b.Geography;
   ```

9. **Use a subquery to find accounts with an `MRR` above the average `MRR` for their `Contract_Type`.**
   ```sql
   SELECT Account_ID, Contract_Type, MRR
   FROM `long-loop-442611-j5.saas.saas_base`
   WHERE MRR > (
       SELECT AVG(MRR)
       FROM `long-loop-442611-j5.saas.saas_base` sub
       WHERE sub.Contract_Type = `long-loop-442611-j5.saas.saas_base`.Contract_Type
   );
   ```

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
   SELECT 
       Account_ID, 
       Country, 
       ARR, 
       RANK() OVER (PARTITION BY Country ORDER BY ARR DESC) AS rank_within_country
   FROM `long-loop-442611-j5.saas.saas_base`;
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

---

### **Expert-Level Questions**
14. **Calculate the retention rate for each `Plan_Offering` using a self-join.**
   ```sql
   SELECT 
       current.Plan_Offering, 
       COUNT(DISTINCT current.Account_ID) AS total_accounts,
       COUNT(DISTINCT future.Account_ID) AS retained_accounts,
       COUNT(DISTINCT future.Account_ID) / COUNT(DISTINCT current.Account_ID) AS retention_rate
   FROM `long-loop-442611-j5.saas.saas_base` current
   LEFT JOIN `long-loop-442611-j5.saas.saas_base` future
   ON current.Account_ID = future.Account_ID 
      AND current.Month < future.Month
   GROUP BY current.Plan_Offering;
   ```

15. **Calculate the churn rate for `Contract_Type` using a window function.**
   ```sql
   SELECT 
       Contract_Type, 
       COUNT(Account_ID) AS total_accounts, 
       SUM(CASE WHEN Contract_Status = 'Churned' THEN 1 ELSE 0 END) AS churned_accounts,
       SUM(CASE WHEN Contract_Status = 'Churned' THEN 1 ELSE 0 END) / COUNT(Account_ID) AS churn_rate
   FROM `long-loop-442611-j5.saas.saas_base`
   GROUP BY Contract_Type;
   ```

---

These queries progressively build SQL skills, starting from basics like filtering and aggregations to advanced concepts like CTEs, window functions, and joins. Let me know if you want any query elaborated or modified further!
