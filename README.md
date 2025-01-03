# Credit Cards Database Analysis Using SQL

This document contains SQL queries to analyze and manipulate data in a credit cards database (`credit_cardsDB`). Below is the schema for the tables and the respective queries answering specific business questions.


## Database and Table Setup

```sql
CREATE DATABASE credit_cardsDB;
USE credit_cardsDB;

-- Creating table cc_details
CREATE TABLE cc_details (
    Client_Num INT,
    Card_Category VARCHAR(20),
    Annual_Fees INT,
    Activation_30_Days INT,
    Customer_Acq_Cost INT,
    Week_Start_Date DATE,
    Week_Num VARCHAR(20),
    Qtr VARCHAR(10), -- Quarter of the year (1 to 4)
    current_year INT,
    Credit_Limit DECIMAL(10, 2),
    Total_Revolving_Bal INT,
    Total_Trans_Amt INT,
    Total_Trans_Vol INT,
    Avg_Utilization_Ratio DECIMAL(10, 3),
    Use_Chip VARCHAR(10), -- Indicates if the card uses a chip for transactions
    Exp_Type VARCHAR(50),
    Interest_Earned DECIMAL(10, 3),
    Delinquent_Acc VARCHAR(5) -- Number of delinquent accounts associated with the client
);

-- Creating customer_details table
CREATE TABLE customer_details (
    Client_Num INT,
    Customer_Age INT,
    Gender VARCHAR(5),
    Dependent_Count INT,
    Education_Level VARCHAR(50),
    Marital_Status VARCHAR(20),
    State_cd VARCHAR(50),
    Zipcode VARCHAR(20),
    Car_Owner VARCHAR(5),
    House_Owner VARCHAR(5),
    Personal_Loan VARCHAR(5),
    Contact VARCHAR(50),
    Customer_Job VARCHAR(50),
    Income INT,
    Cust_Satisfaction_Score INT
);
```

---

---


## Questions

- **Q1.** Write a query to find the average Annual_Fees for each Card_Category.
- **Q2.** Retrieve the list of clients who have a Credit_Limit greater than 20,000 but an Avg_Utilization_Ratio less than 0.2.
- **Q3.** Identify the top 5 states with the highest average income, and within those states, list customers with a Credit_Limit higher than the 90th percentile for their respective states.
- **Q4.** Use window functions to rank customers within each State_cd based on their Total_Trans_Amt. Retrieve the top 3 customers from each state, along with their Customer_Age, Income, and Card_Category.
- **Q5.** Write a query to find the difference between the highest and lowest Credit_Limit for each Card_Category and filter only the categories with a range greater than 15,000.
- **Q6.** Identify customers (customer_details) who have the highest Avg_Utilization_Ratio in their respective states (State_cd) and retrieve their Card_Category and Credit_Limit from the cc_details table.
- **Q7.** Find the average Credit_Limit and total Interest_Earned for each Card_Category for customers who have a satisfaction score (Cust_Satisfaction_Score) above 90 and belong to a specific education level (Education_Level).
- **Q8.** Write a query to identify the top 3 customers with the highest Avg_Utilization_Ratio for each Card_Category and their respective State_cd. Retrieve the following details: client_num, State_cd, Card_Category, Avg_Utilization_Ratio. Rank them based on the utilization ratio within each Card_Category (descending order).

---

## Questions and Queries

### **Q1. Write a query to find the average Annual_Fees for each Card_Category.**
```sql
SELECT card_category, AVG(annual_fees) AS Avg_Annual_Fees
FROM cc_details
GROUP BY card_category;
```

### **Q2. Retrieve the list of clients who have a Credit_Limit greater than 20,000 but an Avg_Utilization_Ratio less than 0.2.**
```sql
SELECT client_Num, Credit_Limit, Avg_Utilization_Ratio 
FROM cc_details
WHERE Credit_Limit > 20000 AND Avg_Utilization_Ratio < 0.2;
```

### **Q3. Identify the top 5 states with the highest average income, and within those states, list customers with a Credit_Limit higher than the 90th percentile for their respective states.**
```sql
WITH percentile90 AS (
    SELECT 
        cc.Client_Num,
        cd.state_cd, 
        cc.credit_limit,
        PERCENT_RANK() OVER(PARTITION BY cd.state_cd ORDER BY cc.Credit_Limit DESC) AS percentile
    FROM cc_details cc
    JOIN customer_details cd ON cd.Client_Num = cc.Client_Num
),
topstates AS (
    SELECT 
        state_cd, 
        AVG(income) AS Avg_Income
    FROM customer_details
    GROUP BY state_cd
    ORDER BY Avg_Income DESC
    LIMIT 5
)
SELECT 
    p9.Client_Num, 
    p9.state_cd, 
    ts.Avg_Income,
    p9.Credit_Limit
FROM percentile90 p9
JOIN topstates ts ON p9.state_cd = ts.state_cd
WHERE p9.percentile > 0.9;
```

### **Q4. Use window functions to rank customers within each State_cd based on their Total_Trans_Amt. Retrieve the top 3 customers from each state, along with their Customer_Age, Income, and Card_Category.**
```sql
SELECT 
    ClientNum,
    Age,
    Income,
    Card_Category
FROM (
    SELECT 
        cc.client_num AS ClientNum,
        cd.Customer_Age AS Age,
        cd.income AS Income,
        cc.Card_Category AS Card_Category,
        DENSE_RANK() OVER(PARTITION BY cd.state_cd ORDER BY cc.Total_Trans_Amt DESC) AS rnk
    FROM cc_details cc
    JOIN customer_details cd ON cd.client_num = cc.client_num
) T
WHERE rnk <= 3;
```

### **Q5. Write a query to find the difference between the highest and lowest Credit_Limit for each Card_Category and filter only the categories with a range greater than 15,000.**
```sql
SELECT card_category,
    MAX(credit_limit) AS Max_credit,
    MIN(credit_limit) AS Min_credit,
    MAX(credit_limit) - MIN(credit_limit) AS creditDiff
FROM cc_details
GROUP BY card_category
HAVING (MAX(credit_limit) - MIN(credit_limit)) > 15000;
```

### **Q6. Identify customers (customer_details) who have the highest Avg_Utilization_Ratio in their respective states (State_cd) and retrieve their Card_Category and Credit_Limit from the cc_details table.**
```sql
WITH max_utilization AS (
    SELECT 
        cd.client_num,
        cd.state_cd,
        MAX(cc.Avg_Utilization_Ratio) AS high_utilization
    FROM customer_details cd
    JOIN cc_details cc ON cc.client_num = cd.client_num
    GROUP BY cd.state_cd, cd.client_num
)
SELECT 
    cd.client_num,
    cc.card_category,
    cc.credit_limit,
    cd.state_cd,
    mu.high_utilization
FROM customer_details cd
JOIN cc_details cc ON cc.client_num = cd.client_num
JOIN max_utilization mu ON mu.client_num = cd.client_num AND mu.state_cd = cd.state_cd
WHERE cc.Avg_Utilization_Ratio = mu.high_utilization;
```

### **Q7. Find the average Credit_Limit and total Interest_Earned for each Card_Category for customers who have a satisfaction score (Cust_Satisfaction_Score) above 90 and belong to a specific education level (Education_Level).**
```sql
SELECT 
    AVG(cc.credit_limit) AS Avg_credit_limit,
    SUM(Interest_Earned) AS Total_interest_earned,
    Card_Category
FROM cc_details cc
JOIN customer_details cd ON cd.client_num = cc.client_num
WHERE 
    Cust_Satisfaction_Score > 90 AND 
    Education_level = 'Graduate'
GROUP BY Card_Category;
```

### **Q8. Write a query to identify the top 3 customers with the highest Avg_Utilization_Ratio for each Card_Category and their respective State_cd. Retrieve the following details: client_num, State_cd, Card_Category, Avg_Utilization_Ratio, and rank them based on the utilization ratio within each Card_Category (descending order).**
```sql
SELECT 
    client_num, 
    State_cd, 
    Card_Category, 
    Avg_Utilization_Ratio,
    DENSE_RANK() OVER(PARTITION BY Card_Category ORDER BY highest_ur DESC) AS Rank_ur
FROM (
    SELECT
        cd.client_num, 
        cd.State_cd, 
        cc.Card_Category, 
        cc.Avg_Utilization_Ratio,
        DENSE_RDENSE_RANK() OVER(PARTITION BY cc.card_category ORDER BY cc.Avg_Utilization_Ratio DESC) AS highest_ur
	 FROM cc_details cc
	JOIN customer_details cd 
		ON cd.client_num = cc.client_num
 ) T
 WHERE highest_ur <= 3;

