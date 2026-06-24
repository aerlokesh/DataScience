# SQL Practice Problems for DS/DA Interviews

> 50 graded problems covering every pattern tested at Meta, Google, Amazon, Netflix, Spotify, Uber.
> Each problem includes: difficulty, company tag, problem statement, schema, solution, and explanation.

---

## Easy (Problems 1-15) — Foundation Patterns

---

### Problem 1: Second Highest Salary

- **Difficulty**: Easy
- **Companies**: Amazon, Microsoft
- **Problem**: Find the second highest distinct salary from the Employees table. Return NULL if there is no second highest.

```sql
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    salary INT
);

INSERT INTO employees VALUES
(1, 'Alice', 100000),
(2, 'Bob', 80000),
(3, 'Charlie', 100000),
(4, 'Diana', 60000);
```

**Solution**:
```sql
SELECT MAX(salary) AS second_highest_salary
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

**Key Insight**: Filtering out the max before taking another max gives you the second highest without window functions.

---

### Problem 2: Duplicate Emails

- **Difficulty**: Easy
- **Companies**: Meta, Google
- **Problem**: Find all duplicate emails in a users table.

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(255)
);

INSERT INTO users VALUES
(1, 'a@example.com'),
(2, 'b@example.com'),
(3, 'a@example.com'),
(4, 'c@example.com'),
(5, 'b@example.com');
```

**Solution**:
```sql
SELECT email
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

**Key Insight**: GROUP BY + HAVING COUNT(*) > 1 is the canonical duplicate-detection pattern.

---

### Problem 3: Customers Who Never Ordered

- **Difficulty**: Easy
- **Companies**: Amazon, Walmart
- **Problem**: Find customers who have never placed an order.

```sql
CREATE TABLE customers (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT
);

INSERT INTO customers VALUES (1,'Alice'),(2,'Bob'),(3,'Charlie');
INSERT INTO orders VALUES (1,1),(2,1),(3,2);
```

**Solution**:
```sql
SELECT c.name
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.id IS NULL;
```

**Key Insight**: LEFT JOIN + WHERE right_table.key IS NULL finds non-matching rows (anti-join pattern).

---

### Problem 4: Rising Temperature

- **Difficulty**: Easy
- **Companies**: Meta, Amazon
- **Problem**: Find dates where the temperature was higher than the previous day.

```sql
CREATE TABLE weather (
    id INT PRIMARY KEY,
    record_date DATE,
    temperature INT
);

INSERT INTO weather VALUES
(1, '2023-01-01', 10),
(2, '2023-01-02', 25),
(3, '2023-01-03', 20),
(4, '2023-01-04', 30);
```

**Solution**:
```sql
SELECT w1.id
FROM weather w1
JOIN weather w2
  ON w1.record_date = w2.record_date + INTERVAL 1 DAY
WHERE w1.temperature > w2.temperature;
```

**Key Insight**: Self-join on date arithmetic compares each row with its logical predecessor.

---

### Problem 5: Rank Scores

- **Difficulty**: Easy
- **Companies**: Google, Netflix
- **Problem**: Rank scores in descending order. Ties should receive the same rank with no gaps (dense rank).

```sql
CREATE TABLE scores (
    id INT PRIMARY KEY,
    score DECIMAL(5,2)
);

INSERT INTO scores VALUES (1,3.50),(2,3.65),(3,4.00),(4,3.85),(5,4.00),(6,3.65);
```

**Solution**:
```sql
SELECT score,
       DENSE_RANK() OVER (ORDER BY score DESC) AS rank
FROM scores;
```

**Key Insight**: DENSE_RANK() assigns consecutive ranks without gaps; RANK() would leave gaps after ties.

---

### Problem 6: Department Highest Salary

- **Difficulty**: Easy
- **Companies**: Amazon, Meta
- **Problem**: Find employees who earn the highest salary in their department.

```sql
CREATE TABLE employee (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    salary INT,
    department_id INT
);

CREATE TABLE department (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

INSERT INTO department VALUES (1,'Engineering'),(2,'Sales');
INSERT INTO employee VALUES
(1,'Alice',90000,1),(2,'Bob',85000,1),(3,'Charlie',70000,2),(4,'Diana',80000,2);
```

**Solution**:
```sql
SELECT d.name AS department, e.name AS employee, e.salary
FROM employee e
JOIN department d ON e.department_id = d.id
WHERE (e.department_id, e.salary) IN (
    SELECT department_id, MAX(salary)
    FROM employee
    GROUP BY department_id
);
```

**Key Insight**: Correlated subquery with tuple comparison (col1, col2) IN (subquery) handles per-group max elegantly.

---

### Problem 7: Employees Earning More Than Their Manager

- **Difficulty**: Easy
- **Companies**: Meta, Apple
- **Problem**: Find employees who earn more than their direct manager.

```sql
CREATE TABLE emp (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    salary INT,
    manager_id INT
);

INSERT INTO emp VALUES
(1,'Joe',70000,3),(2,'Henry',80000,4),(3,'Sam',60000,NULL),(4,'Max',90000,NULL);
```

**Solution**:
```sql
SELECT e.name AS employee
FROM emp e
JOIN emp m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

**Key Insight**: Self-join on manager_id links each employee to their manager for direct comparison.

---

### Problem 8: Consecutive Numbers

- **Difficulty**: Easy
- **Companies**: Google, Amazon
- **Problem**: Find all numbers that appear at least three times consecutively in the Logs table.

```sql
CREATE TABLE logs (
    id INT PRIMARY KEY,
    num INT
);

INSERT INTO logs VALUES (1,1),(2,1),(3,1),(4,2),(5,1),(6,2),(7,2);
```

**Solution**:
```sql
SELECT DISTINCT l1.num AS consecutive_num
FROM logs l1
JOIN logs l2 ON l1.id = l2.id - 1
JOIN logs l3 ON l2.id = l3.id - 1
WHERE l1.num = l2.num AND l2.num = l3.num;
```

**Key Insight**: Triple self-join on consecutive IDs checks for 3-in-a-row; DISTINCT deduplicates results.

---

### Problem 9: NULL Handling in Aggregations

- **Difficulty**: Easy
- **Companies**: Netflix, Spotify
- **Problem**: Calculate the average delivery rating, treating NULL ratings as 0 for the count denominator but not adding them to the sum.

```sql
CREATE TABLE deliveries (
    id INT PRIMARY KEY,
    rating INT  -- NULL means not yet rated
);

INSERT INTO deliveries VALUES (1,5),(2,4),(3,NULL),(4,3),(5,NULL),(6,5);
```

**Solution**:
```sql
SELECT
    COALESCE(SUM(rating), 0) AS total_rating,
    COUNT(*) AS total_deliveries,
    ROUND(COALESCE(SUM(rating), 0) * 1.0 / COUNT(*), 2) AS avg_including_unrated,
    ROUND(AVG(rating), 2) AS avg_excluding_unrated
FROM deliveries;
```

**Key Insight**: AVG() ignores NULLs; to include them in the denominator, use SUM()/COUNT(*).

---

### Problem 10: CASE WHEN for Pivoting

- **Difficulty**: Easy
- **Companies**: Uber, Meta
- **Problem**: Pivot a status table to show counts of each order status per day.

```sql
CREATE TABLE order_status (
    order_date DATE,
    status VARCHAR(20)
);

INSERT INTO order_status VALUES
('2023-01-01','delivered'),('2023-01-01','delivered'),('2023-01-01','cancelled'),
('2023-01-02','delivered'),('2023-01-02','pending'),('2023-01-02','cancelled');
```

**Solution**:
```sql
SELECT
    order_date,
    SUM(CASE WHEN status = 'delivered' THEN 1 ELSE 0 END) AS delivered,
    SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) AS cancelled,
    SUM(CASE WHEN status = 'pending' THEN 1 ELSE 0 END) AS pending
FROM order_status
GROUP BY order_date;
```

**Key Insight**: CASE WHEN inside SUM/COUNT is the manual pivot pattern — works in every SQL dialect.

---

### Problem 11: First Login Date

- **Difficulty**: Easy
- **Companies**: Meta, TikTok
- **Problem**: For each user, find their first login date.

```sql
CREATE TABLE logins (
    user_id INT,
    login_date DATE
);

INSERT INTO logins VALUES
(1,'2023-01-05'),(1,'2023-01-10'),(2,'2023-02-01'),(2,'2023-01-15'),(3,'2023-03-01');
```

**Solution**:
```sql
SELECT user_id, MIN(login_date) AS first_login
FROM logins
GROUP BY user_id;
```

**Key Insight**: MIN() on a date column per group gives the earliest event — simpler than window functions for this case.

---

### Problem 12: Delete Duplicate Rows

- **Difficulty**: Easy
- **Companies**: Amazon, Microsoft
- **Problem**: Delete duplicate emails keeping only the row with the smallest id.

```sql
CREATE TABLE person (
    id INT PRIMARY KEY,
    email VARCHAR(255)
);

INSERT INTO person VALUES (1,'john@example.com'),(2,'bob@example.com'),(3,'john@example.com');
```

**Solution**:
```sql
DELETE p1
FROM person p1
JOIN person p2
  ON p1.email = p2.email AND p1.id > p2.id;
```

**Key Insight**: Self-join delete keeps the row with the smallest id for each group by removing rows where a smaller id exists with the same value.

---

### Problem 13: Percentage of Total

- **Difficulty**: Easy
- **Companies**: Spotify, Netflix
- **Problem**: For each genre, calculate its percentage of total streams.

```sql
CREATE TABLE streams (
    id INT PRIMARY KEY,
    genre VARCHAR(50),
    stream_count INT
);

INSERT INTO streams VALUES
(1,'Pop',5000),(2,'Rock',3000),(3,'Hip-Hop',4000),(4,'Pop',2000),(5,'Rock',1000);
```

**Solution**:
```sql
SELECT
    genre,
    SUM(stream_count) AS genre_streams,
    ROUND(100.0 * SUM(stream_count) / (SELECT SUM(stream_count) FROM streams), 2) AS pct_of_total
FROM streams
GROUP BY genre
ORDER BY pct_of_total DESC;
```

**Key Insight**: Scalar subquery for the total in the denominator avoids an extra join or window function.

---

### Problem 14: Date Difference Calculation

- **Difficulty**: Easy
- **Companies**: Uber, Lyft
- **Problem**: For each ride, calculate the number of days between booking and completion.

```sql
CREATE TABLE rides (
    ride_id INT PRIMARY KEY,
    booked_date DATE,
    completed_date DATE
);

INSERT INTO rides VALUES
(1,'2023-01-01','2023-01-01'),
(2,'2023-01-05','2023-01-07'),
(3,'2023-02-10','2023-02-15');
```

**Solution**:
```sql
SELECT
    ride_id,
    booked_date,
    completed_date,
    DATEDIFF(completed_date, booked_date) AS days_to_complete
FROM rides;
```

**Key Insight**: DATEDIFF(end, start) computes day-level gaps; use TIMESTAMPDIFF for hours/minutes.

---

### Problem 15: UNION vs UNION ALL

- **Difficulty**: Easy
- **Companies**: Google, Amazon
- **Problem**: Combine active users from two platforms, preserving duplicates for a total count, and also produce a distinct user list.

```sql
CREATE TABLE ios_users (user_id INT);
CREATE TABLE android_users (user_id INT);

INSERT INTO ios_users VALUES (1),(2),(3),(4);
INSERT INTO android_users VALUES (3),(4),(5),(6);
```

**Solution**:
```sql
-- Distinct users across both platforms
SELECT user_id FROM ios_users
UNION
SELECT user_id FROM android_users;

-- All rows (duplicates preserved) for total engagement count
SELECT user_id FROM ios_users
UNION ALL
SELECT user_id FROM android_users;
```

**Key Insight**: UNION deduplicates (like DISTINCT); UNION ALL preserves every row — use ALL unless you explicitly need deduplication.

---

## Medium (Problems 16-35) — Interview-Level

---

### Problem 16: Month-over-Month Revenue Growth

- **Difficulty**: Medium
- **Companies**: Meta, Amazon
- **Problem**: Calculate MoM revenue growth percentage for each month.

```sql
CREATE TABLE revenue (
    month_date DATE,
    revenue DECIMAL(12,2)
);

INSERT INTO revenue VALUES
('2023-01-01',100000),('2023-02-01',120000),('2023-03-01',115000),
('2023-04-01',130000),('2023-05-01',150000);
```

**Solution**:
```sql
SELECT
    month_date,
    revenue,
    LAG(revenue) OVER (ORDER BY month_date) AS prev_month_revenue,
    ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month_date))
          / LAG(revenue) OVER (ORDER BY month_date), 2) AS mom_growth_pct
FROM revenue;
```

**Key Insight**: LAG() accesses the previous row's value without a self-join; growth = (current - prev) / prev * 100.

---

### Problem 17: Running Total of Sales

- **Difficulty**: Medium
- **Companies**: Amazon, Walmart
- **Problem**: Calculate a running total of daily sales within each month.

```sql
CREATE TABLE daily_sales (
    sale_date DATE,
    amount DECIMAL(10,2)
);

INSERT INTO daily_sales VALUES
('2023-01-01',500),('2023-01-02',300),('2023-01-03',700),
('2023-02-01',400),('2023-02-02',600);
```

**Solution**:
```sql
SELECT
    sale_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY DATE_FORMAT(sale_date, '%Y-%m')
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM daily_sales;
```

**Key Insight**: SUM() OVER with ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW creates a cumulative sum; PARTITION BY resets it per group.

---

### Problem 18: Top 3 Salaries Per Department

- **Difficulty**: Medium
- **Companies**: Google, Meta
- **Problem**: Find the top 3 earners in each department. Include ties.

```sql
CREATE TABLE emp_dept (
    id INT,
    name VARCHAR(100),
    salary INT,
    dept VARCHAR(50)
);

INSERT INTO emp_dept VALUES
(1,'Alice',90000,'Eng'),(2,'Bob',85000,'Eng'),(3,'Charlie',90000,'Eng'),
(4,'Diana',80000,'Eng'),(5,'Eve',70000,'Sales'),(6,'Frank',75000,'Sales');
```

**Solution**:
```sql
WITH ranked AS (
    SELECT *,
           DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC) AS rnk
    FROM emp_dept
)
SELECT dept, name, salary
FROM ranked
WHERE rnk <= 3;
```

**Key Insight**: DENSE_RANK() + filter for top-N per group handles ties correctly (unlike LIMIT).

---

### Problem 19: User Retention — Day 1 Retention Rate

- **Difficulty**: Medium
- **Companies**: Meta, Snap, TikTok
- **Problem**: Calculate the Day 1 retention rate: the percentage of users who logged in the day after their first login.

```sql
CREATE TABLE user_activity (
    user_id INT,
    activity_date DATE
);

INSERT INTO user_activity VALUES
(1,'2023-01-01'),(1,'2023-01-02'),(2,'2023-01-01'),
(3,'2023-01-02'),(3,'2023-01-03'),(4,'2023-01-03');
```

**Solution**:
```sql
WITH first_login AS (
    SELECT user_id, MIN(activity_date) AS first_date
    FROM user_activity
    GROUP BY user_id
)
SELECT
    ROUND(100.0 * COUNT(DISTINCT a.user_id) / COUNT(DISTINCT f.user_id), 2) AS day1_retention_pct
FROM first_login f
LEFT JOIN user_activity a
  ON f.user_id = a.user_id
  AND a.activity_date = f.first_date + INTERVAL 1 DAY;
```

**Key Insight**: Join first-login CTE to activity on first_date + 1 day; LEFT JOIN ensures non-retained users count in the denominator.

---

### Problem 20: Funnel Conversion Rates

- **Difficulty**: Medium
- **Companies**: Meta, Uber, Airbnb
- **Problem**: Given events (view, cart, purchase), compute step-by-step conversion rates.

```sql
CREATE TABLE funnel_events (
    user_id INT,
    event_type VARCHAR(20),
    event_date DATE
);

INSERT INTO funnel_events VALUES
(1,'view','2023-01-01'),(1,'cart','2023-01-01'),(1,'purchase','2023-01-02'),
(2,'view','2023-01-01'),(2,'cart','2023-01-01'),
(3,'view','2023-01-01'),
(4,'view','2023-01-02'),(4,'cart','2023-01-02'),(4,'purchase','2023-01-02');
```

**Solution**:
```sql
SELECT
    COUNT(DISTINCT CASE WHEN event_type = 'view' THEN user_id END) AS views,
    COUNT(DISTINCT CASE WHEN event_type = 'cart' THEN user_id END) AS carts,
    COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS purchases,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN event_type = 'cart' THEN user_id END)
          / COUNT(DISTINCT CASE WHEN event_type = 'view' THEN user_id END), 2) AS view_to_cart_pct,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END)
          / COUNT(DISTINCT CASE WHEN event_type = 'cart' THEN user_id END), 2) AS cart_to_purchase_pct
FROM funnel_events;
```

**Key Insight**: COUNT(DISTINCT CASE WHEN ... THEN user_id END) counts unique users per funnel step in a single pass.

---

### Problem 21: Year-over-Year Comparison

- **Difficulty**: Medium
- **Companies**: Amazon, Netflix
- **Problem**: Compare monthly revenue to the same month last year.

```sql
CREATE TABLE monthly_revenue (
    year_month DATE,
    revenue DECIMAL(12,2)
);

INSERT INTO monthly_revenue VALUES
('2022-01-01',100000),('2022-02-01',110000),('2022-03-01',95000),
('2023-01-01',120000),('2023-02-01',105000),('2023-03-01',130000);
```

**Solution**:
```sql
SELECT
    curr.year_month,
    curr.revenue AS current_revenue,
    prev.revenue AS prior_year_revenue,
    ROUND(100.0 * (curr.revenue - prev.revenue) / prev.revenue, 2) AS yoy_growth_pct
FROM monthly_revenue curr
LEFT JOIN monthly_revenue prev
  ON curr.year_month = prev.year_month + INTERVAL 1 YEAR;
```

**Key Insight**: Self-join on date + INTERVAL 1 YEAR aligns same-month rows across years for YoY comparison.

---

### Problem 22: Rolling 7-Day Average

- **Difficulty**: Medium
- **Companies**: Spotify, Netflix
- **Problem**: Compute a 7-day rolling average of daily active users.

```sql
CREATE TABLE dau (
    day DATE PRIMARY KEY,
    active_users INT
);

INSERT INTO dau VALUES
('2023-01-01',1000),('2023-01-02',1200),('2023-01-03',1100),('2023-01-04',1300),
('2023-01-05',1250),('2023-01-06',1400),('2023-01-07',1350),('2023-01-08',1500);
```

**Solution**:
```sql
SELECT
    day,
    active_users,
    ROUND(AVG(active_users) OVER (
        ORDER BY day
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 0) AS rolling_7day_avg
FROM dau;
```

**Key Insight**: ROWS BETWEEN 6 PRECEDING AND CURRENT ROW gives exactly 7 rows including today for a trailing average.

---

### Problem 23: Nth Transaction Per User

- **Difficulty**: Medium
- **Companies**: Stripe, Square
- **Problem**: For each user, find their 3rd transaction amount. Return NULL if they have fewer than 3.

```sql
CREATE TABLE transactions (
    id INT,
    user_id INT,
    amount DECIMAL(10,2),
    txn_date DATE
);

INSERT INTO transactions VALUES
(1,1,50,'2023-01-01'),(2,1,30,'2023-01-05'),(3,1,70,'2023-01-10'),
(4,2,100,'2023-01-02'),(5,2,200,'2023-01-08'),(6,1,20,'2023-01-15');
```

**Solution**:
```sql
WITH numbered AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY txn_date) AS rn
    FROM transactions
)
SELECT user_id, amount AS third_txn_amount
FROM numbered
WHERE rn = 3;
```

**Key Insight**: ROW_NUMBER() PARTITION BY user ORDER BY date, then filter rn = N, extracts the Nth event per user.

---

### Problem 24: Users Active on Consecutive Days

- **Difficulty**: Medium
- **Companies**: Meta, Snap
- **Problem**: Find users who were active for at least 3 consecutive days.

```sql
CREATE TABLE activity (
    user_id INT,
    active_date DATE
);

INSERT INTO activity VALUES
(1,'2023-01-01'),(1,'2023-01-02'),(1,'2023-01-03'),
(2,'2023-01-01'),(2,'2023-01-03'),
(3,'2023-01-05'),(3,'2023-01-06'),(3,'2023-01-07'),(3,'2023-01-08');
```

**Solution**:
```sql
WITH numbered AS (
    SELECT DISTINCT user_id, active_date,
           active_date - INTERVAL ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY active_date) DAY AS grp
    FROM activity
)
SELECT DISTINCT user_id
FROM numbered
GROUP BY user_id, grp
HAVING COUNT(*) >= 3;
```

**Key Insight**: Subtracting row_number from date creates a constant "group key" for consecutive sequences (island detection).

---

### Problem 25: Pivoting Rows to Columns

- **Difficulty**: Medium
- **Companies**: Amazon, Google
- **Problem**: Given student scores by subject in rows, pivot to one row per student with columns per subject.

```sql
CREATE TABLE student_scores (
    student_id INT,
    subject VARCHAR(50),
    score INT
);

INSERT INTO student_scores VALUES
(1,'Math',85),(1,'English',90),(1,'Science',78),
(2,'Math',92),(2,'English',88),(2,'Science',95);
```

**Solution**:
```sql
SELECT
    student_id,
    MAX(CASE WHEN subject = 'Math' THEN score END) AS math,
    MAX(CASE WHEN subject = 'English' THEN score END) AS english,
    MAX(CASE WHEN subject = 'Science' THEN score END) AS science
FROM student_scores
GROUP BY student_id;
```

**Key Insight**: MAX(CASE WHEN ...) inside GROUP BY is the standard row-to-column pivot technique.

---

### Problem 26: Finding Gaps in Sequential Data

- **Difficulty**: Medium
- **Companies**: Uber, Lyft
- **Problem**: Find missing seat numbers in a theater where some seats are unoccupied.

```sql
CREATE TABLE seats (
    seat_id INT PRIMARY KEY,
    is_occupied BOOLEAN
);

INSERT INTO seats VALUES (1,1),(2,0),(3,0),(4,1),(5,0),(6,1),(7,1),(8,0),(9,0),(10,1);
```

**Solution**:
```sql
SELECT seat_id
FROM seats
WHERE is_occupied = 0
  AND (seat_id - 1 IN (SELECT seat_id FROM seats WHERE is_occupied = 0)
       OR seat_id + 1 IN (SELECT seat_id FROM seats WHERE is_occupied = 0));
```

**Key Insight**: Checking neighbors (id-1 or id+1) identifies consecutive empty seats; useful for adjacency queries.

---

### Problem 27: Cohort Retention Analysis

- **Difficulty**: Medium
- **Companies**: Meta, Spotify, Netflix
- **Problem**: For users who signed up in January 2023, calculate retention at month 0, 1, 2, 3.

```sql
CREATE TABLE signups (
    user_id INT PRIMARY KEY,
    signup_date DATE
);

CREATE TABLE user_events (
    user_id INT,
    event_date DATE
);

INSERT INTO signups VALUES (1,'2023-01-05'),(2,'2023-01-12'),(3,'2023-01-20'),(4,'2023-02-01');
INSERT INTO user_events VALUES
(1,'2023-01-10'),(1,'2023-02-15'),(1,'2023-03-20'),
(2,'2023-01-15'),(2,'2023-02-10'),
(3,'2023-01-25');
```

**Solution**:
```sql
WITH jan_cohort AS (
    SELECT user_id, signup_date
    FROM signups
    WHERE signup_date >= '2023-01-01' AND signup_date < '2023-02-01'
),
months AS (
    SELECT 0 AS month_offset UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3
)
SELECT
    m.month_offset,
    COUNT(DISTINCT e.user_id) AS retained_users,
    (SELECT COUNT(*) FROM jan_cohort) AS cohort_size,
    ROUND(100.0 * COUNT(DISTINCT e.user_id) / (SELECT COUNT(*) FROM jan_cohort), 2) AS retention_pct
FROM months m
LEFT JOIN jan_cohort c ON 1=1
LEFT JOIN user_events e
  ON c.user_id = e.user_id
  AND e.event_date >= c.signup_date + INTERVAL m.month_offset MONTH
  AND e.event_date < c.signup_date + INTERVAL (m.month_offset + 1) MONTH
GROUP BY m.month_offset
ORDER BY m.month_offset;
```

**Key Insight**: Cross-join cohort with month offsets, then LEFT JOIN events within each month window to measure retention at each interval.

---

### Problem 28: Percentile / Quartile Assignment

- **Difficulty**: Medium
- **Companies**: Google, Netflix
- **Problem**: Assign each employee to a salary quartile (1-4) within their department.

```sql
CREATE TABLE salaries (
    emp_id INT,
    dept VARCHAR(50),
    salary INT
);

INSERT INTO salaries VALUES
(1,'Eng',90000),(2,'Eng',80000),(3,'Eng',110000),(4,'Eng',75000),
(5,'Sales',60000),(6,'Sales',70000),(7,'Sales',55000),(8,'Sales',65000);
```

**Solution**:
```sql
SELECT
    emp_id,
    dept,
    salary,
    NTILE(4) OVER (PARTITION BY dept ORDER BY salary) AS quartile
FROM salaries;
```

**Key Insight**: NTILE(n) divides ordered rows into n roughly equal buckets — perfect for percentile/quartile assignment.

---

### Problem 29: First and Last Event Per Session

- **Difficulty**: Medium
- **Companies**: Uber, Airbnb
- **Problem**: For each user session, find the first and last page visited.

```sql
CREATE TABLE page_views (
    user_id INT,
    session_id INT,
    page VARCHAR(50),
    view_time TIMESTAMP
);

INSERT INTO page_views VALUES
(1,101,'home','2023-01-01 10:00:00'),
(1,101,'product','2023-01-01 10:05:00'),
(1,101,'checkout','2023-01-01 10:10:00'),
(2,102,'home','2023-01-01 11:00:00'),
(2,102,'search','2023-01-01 11:03:00');
```

**Solution**:
```sql
SELECT
    user_id,
    session_id,
    FIRST_VALUE(page) OVER (PARTITION BY session_id ORDER BY view_time) AS entry_page,
    LAST_VALUE(page) OVER (
        PARTITION BY session_id ORDER BY view_time
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS exit_page
FROM page_views;
```

**Key Insight**: FIRST_VALUE/LAST_VALUE with proper frame clause (UNBOUNDED FOLLOWING for last) extracts boundary events per group.

---

### Problem 30: Self-Join for Sequential Events

- **Difficulty**: Medium
- **Companies**: Meta, Google
- **Problem**: Find users who made a purchase within 7 days after viewing a product.

```sql
CREATE TABLE events (
    user_id INT,
    event_type VARCHAR(20),
    event_date DATE,
    product_id INT
);

INSERT INTO events VALUES
(1,'view','2023-01-01',100),(1,'purchase','2023-01-05',100),
(2,'view','2023-01-10',200),(2,'purchase','2023-01-20',200),
(3,'view','2023-01-15',300);
```

**Solution**:
```sql
SELECT DISTINCT v.user_id, v.product_id
FROM events v
JOIN events p
  ON v.user_id = p.user_id
  AND v.product_id = p.product_id
  AND v.event_type = 'view'
  AND p.event_type = 'purchase'
  AND p.event_date BETWEEN v.event_date AND v.event_date + INTERVAL 7 DAY;
```

**Key Insight**: Self-join with date range constraint (BETWEEN start AND start + interval) identifies timed-sequence patterns.

---

### Problem 31: Cumulative Distribution

- **Difficulty**: Medium
- **Companies**: Netflix, Spotify
- **Problem**: For each price point, compute the cumulative percentage of products at or below that price.

```sql
CREATE TABLE products (
    product_id INT,
    price DECIMAL(8,2)
);

INSERT INTO products VALUES (1,10),(2,20),(3,20),(4,30),(5,50),(6,50),(7,100);
```

**Solution**:
```sql
SELECT
    price,
    COUNT(*) AS cnt,
    ROUND(100.0 * CUME_DIST() OVER (ORDER BY price), 2) AS cumulative_pct
FROM products
GROUP BY price;
```

**Key Insight**: CUME_DIST() returns the fraction of rows with values <= current row's value — a built-in CDF.

---

### Problem 32: Flagging Outliers with Standard Deviation

- **Difficulty**: Medium
- **Companies**: Google, Meta
- **Problem**: Flag transactions where the amount is more than 2 standard deviations from the user's mean.

```sql
CREATE TABLE user_txns (
    txn_id INT,
    user_id INT,
    amount DECIMAL(10,2)
);

INSERT INTO user_txns VALUES
(1,1,50),(2,1,55),(3,1,48),(4,1,200),(5,1,52),
(6,2,1000),(7,2,1050),(8,2,980),(9,2,5000),(10,2,1020);
```

**Solution**:
```sql
WITH stats AS (
    SELECT
        user_id,
        AVG(amount) AS avg_amount,
        STDDEV(amount) AS std_amount
    FROM user_txns
    GROUP BY user_id
)
SELECT t.*, s.avg_amount, s.std_amount,
       CASE WHEN ABS(t.amount - s.avg_amount) > 2 * s.std_amount THEN 'OUTLIER' ELSE 'NORMAL' END AS flag
FROM user_txns t
JOIN stats s ON t.user_id = s.user_id;
```

**Key Insight**: Compute per-group stats in a CTE, then join back to flag individual rows — classic anomaly detection pattern.

---

### Problem 33: Overlap Detection (Conflicting Meetings)

- **Difficulty**: Medium
- **Companies**: Google, Microsoft
- **Problem**: Find all pairs of meetings that overlap in time for the same room.

```sql
CREATE TABLE meetings (
    meeting_id INT,
    room_id INT,
    start_time TIMESTAMP,
    end_time TIMESTAMP
);

INSERT INTO meetings VALUES
(1,1,'2023-01-01 09:00','2023-01-01 10:00'),
(2,1,'2023-01-01 09:30','2023-01-01 11:00'),
(3,1,'2023-01-01 11:00','2023-01-01 12:00'),
(4,2,'2023-01-01 09:00','2023-01-01 10:30');
```

**Solution**:
```sql
SELECT m1.meeting_id AS meeting_a, m2.meeting_id AS meeting_b
FROM meetings m1
JOIN meetings m2
  ON m1.room_id = m2.room_id
  AND m1.meeting_id < m2.meeting_id
  AND m1.start_time < m2.end_time
  AND m2.start_time < m1.end_time;
```

**Key Insight**: Two intervals overlap iff start_A < end_B AND start_B < end_A; use id < id to avoid duplicate pairs.

---

### Problem 34: Weighted Average Rating

- **Difficulty**: Medium
- **Companies**: Amazon, Yelp
- **Problem**: Compute a weighted average rating for each product where recent reviews (last 30 days) count 2x.

```sql
CREATE TABLE reviews (
    review_id INT,
    product_id INT,
    rating INT,
    review_date DATE
);

INSERT INTO reviews VALUES
(1,1,5,'2023-03-01'),(2,1,4,'2023-03-15'),(3,1,3,'2023-01-10'),
(4,1,2,'2023-02-01'),(5,2,5,'2023-03-10'),(6,2,4,'2023-01-05');
```

**Solution**:
```sql
SELECT
    product_id,
    ROUND(
        SUM(rating * CASE WHEN review_date >= '2023-03-01' THEN 2 ELSE 1 END) * 1.0
        / SUM(CASE WHEN review_date >= '2023-03-01' THEN 2 ELSE 1 END),
    2) AS weighted_avg_rating
FROM reviews
GROUP BY product_id;
```

**Key Insight**: Multiply both numerator (rating*weight) and denominator (weight) by the same CASE expression for correct weighted averages.

---

### Problem 35: Active Users in the Last N Days (Sliding Window)

- **Difficulty**: Medium
- **Companies**: Meta, LinkedIn
- **Problem**: For each day, count the number of distinct users active in the past 28 days (L28 metric).

```sql
CREATE TABLE user_log (
    user_id INT,
    log_date DATE
);

INSERT INTO user_log VALUES
(1,'2023-01-01'),(2,'2023-01-05'),(1,'2023-01-10'),
(3,'2023-01-15'),(2,'2023-01-20'),(4,'2023-01-25'),(1,'2023-01-28');
```

**Solution**:
```sql
WITH date_spine AS (
    SELECT DISTINCT log_date AS report_date FROM user_log
)
SELECT
    d.report_date,
    COUNT(DISTINCT u.user_id) AS l28_users
FROM date_spine d
LEFT JOIN user_log u
  ON u.log_date BETWEEN d.report_date - INTERVAL 27 DAY AND d.report_date
GROUP BY d.report_date
ORDER BY d.report_date;
```

**Key Insight**: Cross-join a date spine with events using a BETWEEN window to compute sliding-window distinct counts (L7, L28 metrics).

---

## Hard (Problems 36-50) — Senior-Level Differentiators

---

### Problem 36: Sessionization from Raw Events

- **Difficulty**: Hard
- **Companies**: Uber, Airbnb, Spotify
- **Problem**: Given clickstream data, assign session IDs where a new session starts after 30 minutes of inactivity.

```sql
CREATE TABLE clickstream (
    user_id INT,
    event_time TIMESTAMP,
    page VARCHAR(50)
);

INSERT INTO clickstream VALUES
(1,'2023-01-01 10:00:00','home'),
(1,'2023-01-01 10:05:00','search'),
(1,'2023-01-01 10:20:00','product'),
(1,'2023-01-01 11:00:00','home'),
(1,'2023-01-01 11:10:00','cart'),
(2,'2023-01-01 09:00:00','home'),
(2,'2023-01-01 09:40:00','product');
```

**Solution**:
```sql
WITH time_gaps AS (
    SELECT *,
           LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS prev_time,
           CASE
               WHEN TIMESTAMPDIFF(MINUTE, LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time), event_time) > 30
                    OR LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) IS NULL
               THEN 1 ELSE 0
           END AS new_session_flag
    FROM clickstream
),
sessions AS (
    SELECT *,
           SUM(new_session_flag) OVER (PARTITION BY user_id ORDER BY event_time) AS session_id
    FROM time_gaps
)
SELECT user_id, session_id, event_time, page
FROM sessions
ORDER BY user_id, event_time;
```

**Key Insight**: Flag rows where gap > threshold, then cumulative SUM of flags produces monotonically increasing session IDs.

---

### Problem 37: Longest Consecutive Login Streak

- **Difficulty**: Hard
- **Companies**: Meta, Snap, TikTok
- **Problem**: Find each user's longest consecutive daily login streak.

```sql
CREATE TABLE login_history (
    user_id INT,
    login_date DATE
);

INSERT INTO login_history VALUES
(1,'2023-01-01'),(1,'2023-01-02'),(1,'2023-01-03'),(1,'2023-01-05'),
(1,'2023-01-06'),(2,'2023-01-10'),(2,'2023-01-11'),(2,'2023-01-12'),
(2,'2023-01-13'),(2,'2023-01-14');
```

**Solution**:
```sql
WITH deduped AS (
    SELECT DISTINCT user_id, login_date
    FROM login_history
),
grouped AS (
    SELECT user_id, login_date,
           login_date - INTERVAL ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) DAY AS grp
    FROM deduped
),
streaks AS (
    SELECT user_id, grp, COUNT(*) AS streak_length,
           MIN(login_date) AS streak_start, MAX(login_date) AS streak_end
    FROM grouped
    GROUP BY user_id, grp
)
SELECT user_id, MAX(streak_length) AS longest_streak
FROM streaks
GROUP BY user_id;
```

**Key Insight**: date - row_number = constant for consecutive dates (gaps-and-islands technique). Group by that constant to find streaks.

---

### Problem 38: Median Calculation Without Built-in

- **Difficulty**: Hard
- **Companies**: Google, Amazon
- **Problem**: Calculate the median salary per department without using PERCENTILE or MEDIAN functions.

```sql
CREATE TABLE dept_salaries (
    emp_id INT,
    dept VARCHAR(50),
    salary INT
);

INSERT INTO dept_salaries VALUES
(1,'Eng',50000),(2,'Eng',60000),(3,'Eng',70000),(4,'Eng',80000),(5,'Eng',90000),
(6,'Sales',40000),(7,'Sales',55000),(8,'Sales',60000),(9,'Sales',65000);
```

**Solution**:
```sql
WITH ranked AS (
    SELECT dept, salary,
           ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary) AS rn,
           COUNT(*) OVER (PARTITION BY dept) AS cnt
    FROM dept_salaries
)
SELECT dept,
       AVG(salary) AS median_salary
FROM ranked
WHERE rn IN (FLOOR((cnt + 1) / 2.0), CEIL((cnt + 1) / 2.0))
GROUP BY dept;
```

**Key Insight**: Median is at positions FLOOR and CEIL of (n+1)/2; average those two for even-count groups.

---

### Problem 39: Recursive CTE — Organization Hierarchy

- **Difficulty**: Hard
- **Companies**: Amazon, Microsoft, Google
- **Problem**: Given a manager hierarchy, find the full reporting chain (all levels) for each employee.

```sql
CREATE TABLE org (
    emp_id INT PRIMARY KEY,
    name VARCHAR(100),
    manager_id INT
);

INSERT INTO org VALUES
(1,'CEO',NULL),(2,'VP_Eng',1),(3,'VP_Sales',1),
(4,'Dir_Backend',2),(5,'Dir_Frontend',2),(6,'SDE',4),(7,'SDE2',4);
```

**Solution**:
```sql
WITH RECURSIVE hierarchy AS (
    -- Base case: top-level (no manager)
    SELECT emp_id, name, manager_id, name AS chain, 0 AS depth
    FROM org
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive case: join children to current level
    SELECT o.emp_id, o.name, o.manager_id,
           CONCAT(h.chain, ' > ', o.name),
           h.depth + 1
    FROM org o
    JOIN hierarchy h ON o.manager_id = h.emp_id
)
SELECT emp_id, name, chain, depth
FROM hierarchy
ORDER BY depth, emp_id;
```

**Key Insight**: Recursive CTEs anchor at root nodes and walk edges iteratively; CONCAT builds the path string at each level.

---

### Problem 40: Gap and Island — Finding Missing Date Ranges

- **Difficulty**: Hard
- **Companies**: Netflix, Amazon
- **Problem**: A subscription should be active every day. Find the date ranges where data is missing (gaps).

```sql
CREATE TABLE subscription_log (
    user_id INT,
    active_date DATE
);

INSERT INTO subscription_log VALUES
(1,'2023-01-01'),(1,'2023-01-02'),(1,'2023-01-03'),
(1,'2023-01-06'),(1,'2023-01-07'),
(1,'2023-01-10');
```

**Solution**:
```sql
WITH islands AS (
    SELECT user_id, active_date,
           LEAD(active_date) OVER (PARTITION BY user_id ORDER BY active_date) AS next_date
    FROM subscription_log
)
SELECT
    user_id,
    active_date + INTERVAL 1 DAY AS gap_start,
    next_date - INTERVAL 1 DAY AS gap_end,
    DATEDIFF(next_date, active_date) - 1 AS gap_days
FROM islands
WHERE DATEDIFF(next_date, active_date) > 1;
```

**Key Insight**: LEAD() finds the next date; if the difference > 1 day, there is a gap between current+1 and next-1.

---

### Problem 41: Customer Lifetime Value (LTV)

- **Difficulty**: Hard
- **Companies**: Amazon, Shopify, Uber
- **Problem**: Calculate cumulative LTV for each customer at each transaction, and identify customers whose LTV exceeds $500.

```sql
CREATE TABLE purchases (
    txn_id INT,
    customer_id INT,
    amount DECIMAL(10,2),
    purchase_date DATE
);

INSERT INTO purchases VALUES
(1,1,200,'2023-01-01'),(2,1,150,'2023-01-15'),(3,1,200,'2023-02-01'),
(4,2,100,'2023-01-05'),(5,2,50,'2023-01-20'),
(6,3,300,'2023-01-10'),(7,3,250,'2023-02-01');
```

**Solution**:
```sql
WITH ltv AS (
    SELECT *,
           SUM(amount) OVER (PARTITION BY customer_id ORDER BY purchase_date) AS cumulative_ltv,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY purchase_date) AS txn_number
    FROM purchases
)
SELECT customer_id, purchase_date, amount, cumulative_ltv, txn_number
FROM ltv
WHERE cumulative_ltv >= 500
  AND cumulative_ltv - amount < 500;  -- The transaction that crossed the threshold
```

**Key Insight**: Running SUM + filtering where cumulative crosses threshold AND previous was below it finds the exact crossing point.

---

### Problem 42: Attribution — First Touch and Last Touch

- **Difficulty**: Hard
- **Companies**: Meta, Google, Airbnb
- **Problem**: For each conversion, attribute it to both the first and last marketing touch before the conversion.

```sql
CREATE TABLE marketing_touches (
    user_id INT,
    channel VARCHAR(50),
    touch_date DATE
);

CREATE TABLE conversions (
    user_id INT,
    conversion_date DATE,
    revenue DECIMAL(10,2)
);

INSERT INTO marketing_touches VALUES
(1,'organic','2023-01-01'),(1,'email','2023-01-05'),(1,'paid_search','2023-01-08'),
(2,'social','2023-01-02'),(2,'email','2023-01-10');
INSERT INTO conversions VALUES (1,'2023-01-10',100),(2,'2023-01-12',50);
```

**Solution**:
```sql
WITH touches_before AS (
    SELECT
        c.user_id,
        c.conversion_date,
        c.revenue,
        m.channel,
        m.touch_date,
        ROW_NUMBER() OVER (PARTITION BY c.user_id, c.conversion_date ORDER BY m.touch_date ASC) AS first_rn,
        ROW_NUMBER() OVER (PARTITION BY c.user_id, c.conversion_date ORDER BY m.touch_date DESC) AS last_rn
    FROM conversions c
    JOIN marketing_touches m
      ON c.user_id = m.user_id
      AND m.touch_date <= c.conversion_date
)
SELECT
    user_id,
    conversion_date,
    revenue,
    MAX(CASE WHEN first_rn = 1 THEN channel END) AS first_touch_channel,
    MAX(CASE WHEN last_rn = 1 THEN channel END) AS last_touch_channel
FROM touches_before
GROUP BY user_id, conversion_date, revenue;
```

**Key Insight**: ROW_NUMBER ordered ASC = first touch; ordered DESC = last touch. Filter rn=1 in both to get attribution endpoints.

---

### Problem 43: Detecting Declining Revenue (3 Consecutive Months Down)

- **Difficulty**: Hard
- **Companies**: Amazon, Netflix
- **Problem**: Find products with 3 consecutive months of declining revenue.

```sql
CREATE TABLE product_revenue (
    product_id INT,
    month_date DATE,
    revenue DECIMAL(12,2)
);

INSERT INTO product_revenue VALUES
(1,'2023-01-01',10000),(1,'2023-02-01',9000),(1,'2023-03-01',8000),(1,'2023-04-01',7500),
(2,'2023-01-01',5000),(2,'2023-02-01',5500),(2,'2023-03-01',5200),(2,'2023-04-01',6000);
```

**Solution**:
```sql
WITH lagged AS (
    SELECT *,
           LAG(revenue, 1) OVER (PARTITION BY product_id ORDER BY month_date) AS rev_1m_ago,
           LAG(revenue, 2) OVER (PARTITION BY product_id ORDER BY month_date) AS rev_2m_ago
    FROM product_revenue
),
declining AS (
    SELECT *
    FROM lagged
    WHERE revenue < rev_1m_ago AND rev_1m_ago < rev_2m_ago
)
SELECT DISTINCT product_id
FROM declining;
```

**Key Insight**: LAG(col, 1) and LAG(col, 2) let you compare current with two prior rows in one pass; chain inequalities for monotonic trends.

---

### Problem 44: Market Basket Analysis (Frequent Item Pairs)

- **Difficulty**: Hard
- **Companies**: Amazon, Walmart, Instacart
- **Problem**: Find the top 5 most frequently co-purchased product pairs.

```sql
CREATE TABLE order_items (
    order_id INT,
    product_id INT
);

INSERT INTO order_items VALUES
(1,101),(1,102),(1,103),(2,101),(2,102),
(3,102),(3,103),(4,101),(4,103),(5,101),(5,102),(5,104);
```

**Solution**:
```sql
SELECT
    a.product_id AS product_a,
    b.product_id AS product_b,
    COUNT(*) AS co_purchase_count
FROM order_items a
JOIN order_items b
  ON a.order_id = b.order_id
  AND a.product_id < b.product_id
GROUP BY a.product_id, b.product_id
ORDER BY co_purchase_count DESC
LIMIT 5;
```

**Key Insight**: Self-join on order_id with product_a < product_b generates unique pairs; COUNT gives co-occurrence frequency.

---

### Problem 45: Recursive CTE — Generate Date Series

- **Difficulty**: Hard
- **Companies**: Google, Netflix
- **Problem**: Generate a complete date series and LEFT JOIN to find days with zero orders.

```sql
CREATE TABLE order_log (
    order_id INT,
    order_date DATE
);

INSERT INTO order_log VALUES
(1,'2023-01-01'),(2,'2023-01-01'),(3,'2023-01-03'),(4,'2023-01-05');
```

**Solution**:
```sql
WITH RECURSIVE date_series AS (
    SELECT DATE('2023-01-01') AS dt
    UNION ALL
    SELECT dt + INTERVAL 1 DAY
    FROM date_series
    WHERE dt < '2023-01-07'
)
SELECT
    ds.dt AS order_date,
    COALESCE(COUNT(o.order_id), 0) AS order_count
FROM date_series ds
LEFT JOIN order_log o ON ds.dt = o.order_date
GROUP BY ds.dt
ORDER BY ds.dt;
```

**Key Insight**: Recursive CTEs generate date spines on the fly; LEFT JOIN + COALESCE fills gaps with zeros for complete time series.

---

### Problem 46: Window Frame — Identifying Local Peaks

- **Difficulty**: Hard
- **Companies**: Spotify, Netflix
- **Problem**: Find days where DAU is higher than both the previous and next day (local peaks).

```sql
CREATE TABLE daily_dau (
    day DATE,
    dau INT
);

INSERT INTO daily_dau VALUES
('2023-01-01',1000),('2023-01-02',1200),('2023-01-03',1100),
('2023-01-04',1400),('2023-01-05',1350),('2023-01-06',1500),('2023-01-07',1450);
```

**Solution**:
```sql
WITH neighbors AS (
    SELECT
        day,
        dau,
        LAG(dau) OVER (ORDER BY day) AS prev_dau,
        LEAD(dau) OVER (ORDER BY day) AS next_dau
    FROM daily_dau
)
SELECT day, dau
FROM neighbors
WHERE dau > prev_dau AND dau > next_dau;
```

**Key Insight**: LAG() + LEAD() provide the neighboring values; comparing current to both detects local maxima/minima.

---

### Problem 47: Complex Business Logic — Subscription Churn

- **Difficulty**: Hard
- **Companies**: Netflix, Spotify, Amazon
- **Problem**: Identify churned subscribers: users whose subscription ended and did not renew within 30 days.

```sql
CREATE TABLE subscriptions (
    user_id INT,
    start_date DATE,
    end_date DATE
);

INSERT INTO subscriptions VALUES
(1,'2023-01-01','2023-03-01'),(1,'2023-03-15','2023-06-01'),
(2,'2023-01-01','2023-02-01'),
(3,'2023-02-01','2023-04-01'),(3,'2023-04-01','2023-06-01');
```

**Solution**:
```sql
WITH next_sub AS (
    SELECT *,
           LEAD(start_date) OVER (PARTITION BY user_id ORDER BY start_date) AS next_start
    FROM subscriptions
)
SELECT user_id, end_date AS churn_date
FROM next_sub
WHERE next_start IS NULL  -- No future subscription
   OR DATEDIFF(next_start, end_date) > 30;  -- Renewal gap > 30 days
```

**Key Insight**: LEAD(start_date) finds the next subscription start; NULL or large gap means the user churned at that end_date.

---

### Problem 48: Multi-Touch Attribution with Time Decay

- **Difficulty**: Hard
- **Companies**: Meta, Google
- **Problem**: Implement time-decay attribution: touches closer to conversion get exponentially more credit.

```sql
CREATE TABLE touches (
    user_id INT,
    channel VARCHAR(50),
    touch_ts TIMESTAMP
);

CREATE TABLE conv (
    user_id INT,
    conv_ts TIMESTAMP,
    value DECIMAL(10,2)
);

INSERT INTO touches VALUES
(1,'organic','2023-01-01 10:00'),(1,'email','2023-01-05 10:00'),
(1,'paid','2023-01-09 10:00');
INSERT INTO conv VALUES (1,'2023-01-10 10:00',100);
```

**Solution**:
```sql
WITH touch_weights AS (
    SELECT
        t.user_id,
        t.channel,
        t.touch_ts,
        c.conv_ts,
        c.value,
        TIMESTAMPDIFF(HOUR, t.touch_ts, c.conv_ts) AS hours_before,
        -- Exponential decay: weight = 2^(-hours/48) (half-life = 48 hours)
        POWER(2, -TIMESTAMPDIFF(HOUR, t.touch_ts, c.conv_ts) / 48.0) AS raw_weight
    FROM touches t
    JOIN conv c ON t.user_id = c.user_id AND t.touch_ts < c.conv_ts
),
normalized AS (
    SELECT *,
           raw_weight / SUM(raw_weight) OVER (PARTITION BY user_id, conv_ts) AS norm_weight
    FROM touch_weights
)
SELECT
    channel,
    ROUND(SUM(value * norm_weight), 2) AS attributed_value
FROM normalized
GROUP BY channel;
```

**Key Insight**: Compute decay weight per touch, normalize to sum=1 per conversion, then distribute revenue proportionally by normalized weight.

---

### Problem 49: Efficient Deduplication with Qualify

- **Difficulty**: Hard
- **Companies**: Google, Uber
- **Problem**: From a slowly-changing dimension table with duplicates, keep only the latest record per entity using a single query.

```sql
CREATE TABLE scd_records (
    record_id INT,
    entity_id INT,
    attribute VARCHAR(100),
    updated_at TIMESTAMP
);

INSERT INTO scd_records VALUES
(1,100,'v1','2023-01-01 10:00'),(2,100,'v2','2023-01-05 10:00'),
(3,100,'v3','2023-01-10 10:00'),
(4,200,'a1','2023-01-02 10:00'),(5,200,'a2','2023-01-08 10:00');
```

**Solution**:
```sql
-- Using ROW_NUMBER (works everywhere)
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY entity_id ORDER BY updated_at DESC) AS rn
    FROM scd_records
)
SELECT record_id, entity_id, attribute, updated_at
FROM ranked
WHERE rn = 1;

-- Using QUALIFY (Snowflake, BigQuery, Databricks):
-- SELECT *
-- FROM scd_records
-- QUALIFY ROW_NUMBER() OVER (PARTITION BY entity_id ORDER BY updated_at DESC) = 1;
```

**Key Insight**: ROW_NUMBER() DESC + filter rn=1 is the universal "keep latest per group" pattern; QUALIFY is the modern shorthand.

---

### Problem 50: Complex Event Processing — Fraud Detection

- **Difficulty**: Hard
- **Companies**: Stripe, PayPal, Amazon
- **Problem**: Flag users who made 3+ transactions totaling over $1000 within any 1-hour window.

```sql
CREATE TABLE fraud_txns (
    txn_id INT,
    user_id INT,
    amount DECIMAL(10,2),
    txn_time TIMESTAMP
);

INSERT INTO fraud_txns VALUES
(1,1,400,'2023-01-01 10:00:00'),
(2,1,350,'2023-01-01 10:20:00'),
(3,1,300,'2023-01-01 10:45:00'),
(4,1,100,'2023-01-01 12:00:00'),
(5,2,200,'2023-01-01 09:00:00'),
(6,2,100,'2023-01-01 09:30:00');
```

**Solution**:
```sql
WITH window_agg AS (
    SELECT
        t1.user_id,
        t1.txn_time AS window_start,
        COUNT(*) AS txn_count,
        SUM(t2.amount) AS window_total
    FROM fraud_txns t1
    JOIN fraud_txns t2
      ON t1.user_id = t2.user_id
      AND t2.txn_time BETWEEN t1.txn_time AND t1.txn_time + INTERVAL 1 HOUR
    GROUP BY t1.user_id, t1.txn_time
)
SELECT DISTINCT user_id
FROM window_agg
WHERE txn_count >= 3 AND window_total > 1000;
```

**Key Insight**: Use each transaction as a potential window start; self-join for events within 1 hour of that start; aggregate and filter on both count and sum thresholds.

---

## Quick Reference — Patterns Cheat Sheet

| Pattern | Key Technique | Problems |
|---------|--------------|----------|
| Anti-join | LEFT JOIN + WHERE IS NULL | 3, 19 |
| Top-N per group | DENSE_RANK/ROW_NUMBER + filter | 6, 18, 23, 49 |
| Running totals | SUM() OVER (ORDER BY ... ROWS ...) | 17, 41 |
| Gaps and islands | date - ROW_NUMBER() = constant | 24, 37, 40 |
| Sessionization | LAG + cumulative SUM of flags | 36 |
| Time-based comparison | LAG/LEAD or self-join on date arithmetic | 4, 16, 21, 43 |
| Pivoting | CASE WHEN inside aggregate | 10, 25 |
| Sliding window metrics | Date spine + BETWEEN join | 22, 35 |
| Funnel analysis | COUNT(DISTINCT CASE WHEN ...) | 20 |
| Cohort retention | Cross-join cohort x periods + LEFT JOIN events | 27 |
| Overlap detection | start_A < end_B AND start_B < end_A | 33 |
| Recursive traversal | WITH RECURSIVE + anchor/step | 39, 45 |
| Anomaly detection | Stats CTE + threshold comparison | 32, 50 |
| Attribution | ROW_NUMBER ASC/DESC for first/last touch | 42, 48 |
| Median without built-in | ROW_NUMBER + FLOOR/CEIL of (n+1)/2 | 38 |

---

## Study Tips

1. **Pattern recognition > memorization**: Most problems reduce to 5-6 core patterns.
2. **Always clarify edge cases**: NULLs, ties, empty groups, boundary dates.
3. **Start with the output shape**: What columns/rows does the interviewer expect?
4. **CTEs for readability**: Break complex logic into named steps; interviewers love clarity.
5. **Test with small data mentally**: Trace your query through 3-4 rows before declaring done.
6. **Know your dialect differences**: DATEDIFF syntax, QUALIFY, PERCENTILE_CONT vary across engines.
