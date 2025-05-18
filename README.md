# Colume Feature Adoption & Retention Analysis
-----

## Table of Content
1. [Brief Details about the project](#Brief-Details-about-the-project)
2. [🔍 1. Project Overview](#1-Project-Overview)

## Brief Details about the project
**Project Duration:** February – April 2025  
**Prepared by:** Analytics Team  
**Primary Stakeholder:** Product Team 

## 1. Project Overview
In February 2025, Colume launched three new features:  

* Custom Themes
- Voice Assistant
+ Task Reminders

This analysis investigates how early adoption of these features impacted user retention and user behavior trends post-launch.

## ❓ 2. Business Question
**Did early engagement with the newly launched features improve user retention?**

### Did early engagement with the newly launched features improve user retention?

We compared users who adopted at least one feature within the first 7 days of launch against those who did not.


## 🎯 3. Objectives
1. Segment users into **early adopters** and **non-adopters**
2. Calculate **7-day adoption rate**
3. Measure and compare **weekly retention**


---

## 👥 4. Stakeholders
- **Primary:** Product Team

## 🗃️ 5. Data Sources

- `users`: The user  
- `activity_log`: The activity  
- `sessions`
- `billing`
- `features`
- `subscriptions`
- `support_tickets`
- `feedback`

## 🧹 6. Data Cleaning Summary

**Integrity Checks:**
- Checked for nulls, duplicates, outliers
- Verified foreign key relationships

**Cleaning Actions:**
- Dropped users aged <16 or >90  
- Split `full_name` → `first_name`, `last_name`  
- Standardized `plan_type`, `currency`  
- Fixed invalid `churn_date < sign_up_date`  
- Renamed and deduplicated tables  
- Handled invalid amounts and payments  

---

## 🔄 7. Analysis Pipeline

1. **Validate Launch Date** → Feb 20, 2025  
2. **Filter Eligible Users**  
   - Signed up before launch  
   - Didn’t churn on/before launch  
3. **Segment Adopters**  
   - Used feature within 7 days → `adopter`  
   - Otherwise → `non-adopter`  
4. **Calculate Weekly Retention**  
   - Used `ROW_NUMBER()` and weekly buckets from -2 to +6  
   - Retention % = active / cohort size  


-- ## STEP 1: EARLY ADOPTION ANALYSIS PIPELINE 

#### Create view to store retention analysis
```
CREATE VIEW retention_rate AS 
WITH
```
<!---
Step 1a: Get all users eligible for feature adoption
--->
### Step 1a: Get all users eligible for feature adoption
**Criteria**: signed up before launch and not churned before launch
```
eligible_users AS (
    SELECT user_id 
    FROM users
    WHERE sign_up_date < '2025-02-20' 
      AND (churn_date > '2025-02-20' OR churn_date IS NULL)
),
```

### Step 1b: Identify users who adopted any of the new features within 7 days post-launch
```
adopters AS (
    SELECT DISTINCT user_id 
    FROM activity_log
    WHERE activity_type IN ('task_reminder','voice_assistant', 'custom_theme')
      AND timestamp BETWEEN '2025-02-20' AND DATEADD(DAY, 7, '2025-02-20')
      AND user_id IN (SELECT user_id FROM eligible_users)
),
```

### Step 1c: Identify eligible users who did NOT adopt any new feature
```
non_adopters AS (
    SELECT eu.user_id AS user_id 
    FROM eligible_users eu 
    LEFT JOIN adopters a ON eu.user_id = a.user_id
    WHERE a.user_id IS NULL
),
```

### Step 1d: Combine all eligible users with their adoption status
```
all_users AS (
    SELECT 
        eu.user_id AS eligible_users, 
        a.user_id AS adopted, 
        na.user_id AS non_adopted
    FROM eligible_users eu 
    LEFT JOIN adopters a ON eu.user_id = a.user_id
    LEFT JOIN non_adopters na ON na.user_id = eu.user_id
),
```

### Step 1e: Calculate percentage of adopters vs non-adopters
```
adopter_percentage AS (
    SELECT 
        ROUND(COUNT(CASE WHEN adopted IS NOT NULL THEN 1 END) * 100.0 / COUNT(eligible_users), 2) AS adopters_count,
        ROUND(COUNT(CASE WHEN non_adopted IS NOT NULL THEN 1 END) * 100.0 / COUNT(eligible_users), 2) AS non_adopters_count
    FROM all_users
),
```

<!---
-- ============================================
-- STEP 2: WEEKLY RETENTION ANALYSIS
-- ============================================

-- Step 2a: Track weekly activity for all eligible users and assign group (adopter or non-adopter)
user_group AS (
    SELECT 
        eligible_users, 
        CASE WHEN adopted IS NOT NULL THEN 'adopter' ELSE 'non_adopter' END AS adopter_group,
        timestamp, 
        DATEDIFF(WEEK, '2025-02-20', timestamp) AS week_diff, -- Week relative to launch
        ROW_NUMBER() OVER (
            PARTITION BY eligible_users, DATEDIFF(WEEK, '2025-02-20', timestamp)
            ORDER BY timestamp
        ) AS RowN -- First appearance in the week
    FROM all_users 
    JOIN activity_log a ON a.user_id = all_users.eligible_users
    WHERE timestamp BETWEEN DATEADD(WEEK, -2, '2025-02-20') AND DATEADD(WEEK, 6, '2025-02-20')
),

-- Step 2b: Count total unique users per group
grouped AS (
    SELECT adopter_group, COUNT(DISTINCT eligible_users) AS all_users
    FROM user_group
    GROUP BY adopter_group
),

-- Step 2c: Count retained users per week (only those who appeared at least once)
Weekly_retention AS (
    SELECT 
        adopter_group,
        week_diff, 
        COUNT(DISTINCT eligible_users) AS retained_users
    FROM user_group
    WHERE RowN = 1
    GROUP BY adopter_group, week_diff
),

-- Step 2d: Combine retention counts with group size to calculate weekly retention rate
retention_rate AS (
    SELECT 
        wr.adopter_group, 
        week_diff, 
        retained_users, 
        all_users, 
        ROUND(CAST(retained_users * 100.0 AS FLOAT) / all_users, 2) AS retention_rate
    FROM Weekly_retention wr 
    JOIN grouped g ON g.adopter_group = wr.adopter_group
),

-- Step 2e: Pivot result to compare adopter vs non-adopter side-by-side per week
pivoted AS (
    SELECT 
        week_diff,
        MAX(CASE WHEN adopter_group = 'adopter' THEN retention_rate END) AS adopter,
        MAX(CASE WHEN adopter_group = 'non_adopter' THEN retention_rate END) AS non_adopter
    FROM retention_rate
    GROUP BY week_diff
)

-- Final Output: Weekly retention rates and difference between adopter and non-adopter groups
SELECT 
    week_diff, 
    adopter, 
    non_adopter, 
    (adopter - non_adopter) AS percent_diff
FROM pivoted;

--->

## ⚙️ 8. Query Highlights

- CTEs and window functions used  
- Indexed `user_id`, `timestamp`  
- Avoided correlated subqueries  
- Used `EXPLAIN` for optimization  

---

## 📊 9. Key Metrics

### Adoption Summary

- **7-Day Adoption Rate:** 45.41%  
- **Adopters:** 2,837  
- **Non-Adopters:** 3,411  

**Feature Usage Breakdown:**

- Custom Themes: 35.13%  
- Voice Assistant: 33.23%  
- Task Reminders: 31.65%

---

### Weekly Retention Rate

| Week | Adopters (%) | Non-Adopters (%) | Difference |
|:------|:--------------:|:------------------:|------------:|
| -2   | 59.32        | 31.09            | +28.23     |
| -1   | 82.23        | 57.35            | +24.88     |
| 0    | 89.81        | 56.59            | +33.22     |
| 1    | 93.48        | 53.67            | **+39.81** |
| 2    | 85.27        | 57.80            | +27.47     |
| 3    | 86.08        | 60.77            | +25.31     |
| 4    | 85.62        | 59.62            | +26.00     |
| 5    | 84.17        | 57.15            | +27.02     |
| 6    | 70.92        | 42.12            | +28.80     |

---

## 🧠 10. BAIIR Framework

| Stage         | Action Taken |
|:---------------|:--------------|
| **Baseline**  | Defined eligible users and grouped by adoption |
| **Analysis**  | Retention, churn, session time, upgrade trends |
| **Insight**   | Adoption improves retention and engagement |
| **Impact**    | Boosting adoption improves retention, revenue |
| **Recommendation** | Build nudges, reinforce value, reduce drop-off |

---

## 💡 11. Insights, Impacts & Recommendations

### **Insight 1: Early Adoption Improves Retention**

![Weekly Retention Rate](https://github.com/sylviawut/Colume_Project/blob/main/weekly%20retention%20rate.jpg)

<figure>
  <img src="https://github.com/sylviawut/Colume_Project/blob/main/weekly%20retention%20rate.jpg" width=100% height=100% alt="alt text">
  <figcaption>Figure: Weekly Retention Rate</figcaption>
</figure>
</br></br>

- Week 1 retention: 93.48% (Adopters) vs 53.67% (Non-Adopters)  
- Week 6 retention: 70.92% vs 42.12%  

**Impact:**  
10% boost in adoption → ~1,000 users → +288 retained users

**Recommendations:**  
- Prompt usage in first 3 days  
- Re-engage by Day 5  
- Celebrate first feature use  
- Push best-performing features

---

### **Insight 2: Adopters Churn More**

NOTE: THE PLAN CHANGE CHART SHOULD BE HERE

- 5.7% of adopters churned vs. 1.8% of non-adopters  

**Impact:**  
Reducing churn by 2% saves ~42 users

**Recommendations:**  
- Build 7-day post-adoption flow  
- Show benefits gained  
- Nudge inactive adopters early  
- Survey churned users

---

### **Insight 3: Low Plan Upgrades**

STILL HERE TO: PLAN CHANGE

- Over 90% stayed on same plan  
- Only 3.6% of adopters upgraded

**Impact:**  
Feature adoption didn't drive upsells

**Recommendations:**  
- Push locked features post-adoption  
- Show impact metrics (time saved, tasks done)  
- Trial premium features  
- Add upgrade suggestions in dashboard

---

## 📎 12. Supporting Files

| File | Description |
|------|-------------|
| `Data_Cleaning.sql` | SQL scripts for all cleaning steps |
| `Analysis.sql` | Logic for adoption, segmentation, retention |
| `User_Retention.pdf` | Framing the business question |
| `Retention_Analysis.pbix` | Visuals & trends |

---

## 📌 13. Final Thoughts

Early adoption drives higher retention and engagement. 
With nudges, better onboarding, and feature reinforcement, Colume can improve long-term retention and conversion.

