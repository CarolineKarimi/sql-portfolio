# SQL Cheat Sheet

A personal reference of tricks, patterns and syntax that I add to as I learn. Examples use made-up tables (`employees`, `orders`, `customers`, `posts` and so on) and PostgreSQL syntax unless noted.

> Running SQL in SQLite (for example in a Jupyter notebook)? SQLite has no `EXTRACT`, `DATE_TRUNC`, `INTERVAL` or `::`. Use `strftime()` and `date()` for dates instead.

## Contents

0. [Tricks to remember](#0-tricks-to-remember)
1. [SELECT and DISTINCT](#1-select-and-distinct)
2. [WHERE: filtering](#2-where-filtering)
3. [NULL](#3-null)
4. [ORDER BY, LIMIT, OFFSET](#4-order-by-limit-offset)
5. [Aggregates, GROUP BY, HAVING](#5-aggregates-group-by-having)
6. [Arithmetic and math functions](#6-arithmetic-and-math-functions)
7. [CASE](#7-case)
8. [Joins](#8-joins)
9. [Dates](#9-dates)
10. [Subqueries and CTEs](#10-subqueries-and-ctes)
11. [Window function preview](#11-window-function-preview)

---

## 0. Tricks to remember

### Quoting: names vs values

| | No spaces | Has spaces |
|---|---|---|
| **Identifier** (table, column, alias) | `total_orders` (no quotes) | `"total orders"` (double quotes) |
| **String value** (actual data) | `'sushi'` (single quotes) | `'big spender'` (single quotes) |

- Values always get single quotes, even one-word values. Without quotes SQL thinks it is a column name.
- Single quotes around an alias are wrong. Double quotes are for names.
- Easiest habit: use underscores in names and you never need quotes.
- A quoted name must be quoted the same way everywhere it is used.

### Read the wording carefully

- **"At least N"** means `>= N`, not `> N`.
- **"Between a and b"** is inclusive, and `BETWEEN` needs the smaller number first.
- **`'n/a'` is not `NULL`.** If a question says a value "isn't n/a", compare against the text: `col != 'n/a'`. `IS NOT NULL` will not catch it.
- **"Sort by X" with no direction** means the default, ascending. Add `DESC` only when the question says highest first, descending, and so on.
- **"First day" is 0 days after sign-up**, "second day" is 1 day after. Dates count from zero.
- **Pick the column by the kind of value.** "Victoria, Tasmania, Queensland" are states, not cities. Check which column holds that kind of thing.
- **The reason for the question tells you the level of detail.** "Investigating bottlenecks" means you need where a part is stuck, so keep the step column, not just the part name.

### Which columns to select

- If the question lists the output columns, output exactly those. Do not add `SELECT *`.
- If it says "categorize", include the new column plus something that identifies the row.
- If it says "all", include all.
- After grouping, the output is usually the group column plus the aggregate. An aggregate alone is a number with no label.
- Ask: would leaving this column out hide something relevant to the question?

### Execution order

```
FROM -> JOIN -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY -> LIMIT
```

What you type first is not what runs first. This order explains most errors:

- `WHERE` and `HAVING` run before `SELECT`, so they **cannot use a `SELECT` alias**. Repeat the expression, or wrap the query in a subquery.
- `ORDER BY` runs after `SELECT`, so it **can** use aliases.
- PostgreSQL also lets `GROUP BY` use an alias, as a convenience. SQL Server and Oracle do not. `GROUP BY 1, 2` (column positions) or repeating the expression works everywhere.
- `COUNT(...)` is evaluated near the end, after joining, filtering and grouping.

### WHERE vs HAVING

- `WHERE` filters rows before grouping. `HAVING` filters groups after aggregation.
- If the condition uses an aggregate (`COUNT`, `SUM`, `AVG`...), it goes in `HAVING`. A plain row condition goes in `WHERE`.
- `HAVING` can use an aggregate that is not in `SELECT`. A column can do work without being displayed.

### GROUP BY rules

- When you mix aggregates with plain columns, every plain column in `SELECT` must also be in `GROUP BY`.
- Row-by-row math (`a - b`, concatenation, `CASE`) is not an aggregate. It does not force `GROUP BY`.
- `GROUP BY` collapses rows. It can find the max, but it cannot hand back the row that owns the max (see "Top row per group").
- Without `GROUP BY`, the whole table is one group.

### NULL

- Use `IS NULL` / `IS NOT NULL`. `= NULL` never works.
- `COUNT(*)` counts rows. `COUNT(col)` skips `NULL`s.
- In PostgreSQL and Oracle, `NULL` sorts as the highest value: last in `ASC`, first in `DESC`. MySQL and SQL Server sort it first in `ASC`.
- `NULL = 'x'` is never true, so a `NULL` row fails any comparison.
- `COALESCE(a, b, c)` returns the first non-null value.

### Division

- Integer divided by integer drops the decimals (`10 / 4` gives `2`).
- Make **one** side decimal: `CAST(10 AS DECIMAL) / 4`, `10 * 1.0 / 4`, or `10::DECIMAL / 4`. You never need both sides.
- `AVG()` already returns a decimal, so no casting is needed there.

### Ties

- If two rows can have equal values in your `ORDER BY` column, their order is not guaranteed. Add a second sort column.
- `LIMIT 2` can cut off a tie for second place. Check whether the question promises no ties.

### Duplicates

- SQL has no "drop duplicates" step. Use `SELECT DISTINCT` for the output, or `COUNT(DISTINCT col)` inside an aggregate.
- `DISTINCT` only removes rows that are identical in every selected column. If the rows differ, it is a "pick one row per group" problem, not a duplicates problem.
- `COUNT(skill)` and `COUNT(DISTINCT skill)` only differ when duplicates exist.

### First / last

- "First", "earliest", "oldest" means `MIN`. "Last", "latest", "most recent" means `MAX`.
- `MIN` and `MAX` work on dates, text and numbers.

### CASE tricks

- `CASE` stops at the first true condition, so order matters.
- `CASE` creates a value per row. It does not change the table.
- With no `ELSE`, a row that matches nothing gets `NULL`.
- Inside aggregates:
  - `COUNT(CASE WHEN ... THEN 1 ELSE NULL END)`: use `NULL`, because `COUNT` skips it.
  - `SUM(CASE WHEN ... THEN col ELSE 0 END)`: `0` is fine.
  - `AVG(CASE WHEN ... THEN col ELSE NULL END)`: **never `ELSE 0`**, because the zeros drag the average down.
- **One number or several?** One filter and one number: `WHERE` plus the aggregate. Several conditional numbers side by side: one `aggregate(CASE ...)` per column.
- `WHERE` throws rows out before counting. `CASE` inside an aggregate keeps every row and routes it to a scoreboard.
- `GROUP BY` decides how many rows you get. `CASE` inside an aggregate decides how many columns of numbers you get.

### Mixed + and - rows in one column

Deposits and withdrawals, credits and debits, purchases and refunds: one amount column, opposite meanings.

```sql
SELECT account_id,
       SUM(CASE WHEN transaction_type = 'Deposit' THEN amount
                ELSE -amount END) AS final_balance
FROM transactions
GROUP BY account_id;
```

Question to ask: "Does every row add to the total in the same way?" If not, put a `CASE` inside the `SUM`.

### Top row per group ("highest X for each Y")

Ask three questions:

1. Is there a "for each" something? Then I am working per group.
2. Is there a "top / highest / latest" inside it? Then I am picking a winner per group.
3. Do I need columns beyond the group and the winning value? Then I need the whole row, and `GROUP BY` alone will not work.

Three ways to do it (all give the same rows):

```sql
-- 1) Correlated subquery: keep the row equal to its own group's max
SELECT *
FROM concerts AS c1
WHERE revenue = (SELECT MAX(revenue) FROM concerts AS c2 WHERE c2.genre = c1.genre);

-- 2) Join back to the grouped max
SELECT c.*
FROM concerts AS c
JOIN (SELECT genre, MAX(revenue) AS max_rev FROM concerts GROUP BY genre) AS m
  ON c.genre = m.genre AND c.revenue = m.max_rev;

-- 3) ROW_NUMBER (section 11): shortest, and returns exactly one row per group
```

Ties for the max return both rows in options 1 and 2. `ROW_NUMBER()` returns one.

### Self join

- SQL only compares columns within one row. To compare a row with another row of the same table, join the table to itself so the two rows sit side by side.
- Aliases make one table behave like two. Name them by role (`emp`, `mgr`), not `a` and `b`.
- `manager_id` is a pointer: it holds another row's `employee_id`. Read the `ON` line aloud: "my manager_id equals your employee_id, so you are my manager." The alias holding the pointer column is the employee.
- Alias every column in a self join, or SQL reports "ambiguous".
- Spot a self join from phrases like "than their manager", "than the previous day", "pairs in the same department", "who referred whom".

### Join tricks

- `JOIN` alone means `INNER JOIN`. `LEFT JOIN` and `LEFT OUTER JOIN` are the same.
- To find rows with no match: `LEFT JOIN` then `WHERE right_table.key IS NULL`.
- A `WHERE` condition on the right table of a `LEFT JOIN` turns it back into an inner join. Put the condition in `ON` if you need to keep the unmatched rows.
- For an `INNER JOIN`, a condition in `ON ... AND` or in `WHERE` gives identical results.
- `FULL OUTER JOIN` plus a shared key column: use `COALESCE(a.key, b.key)` to get one clean key column.
- Choose a join type by asking: can rows exist in only one table, and from which direction? One direction means `LEFT`. Both directions means `FULL OUTER`.
- Joins do not change aggregation. After the join, `GROUP BY` and aggregates work as usual on the wide rows.

### Join row counts

Duplicates multiply. If a value appears `a` times on the left and `b` times on the right, it produces `a x b` rows.

- `INNER` = sum of `a x b` over the values both sides share.
- `LEFT` = `INNER` + left rows with no match (each counted once).
- `RIGHT` = `INNER` + right rows with no match.
- `FULL OUTER` = `INNER` + both sets of unmatched rows.

Example: left values `1,1,1,2,2,3`, right values `1,1,2,2,2,4`. Shared: value 1 gives `3 x 2 = 6`, value 2 gives `2 x 3 = 6`, so inner = 12. Left adds the `3` (13), right adds the `4` (13), full outer adds both (14). Counting unique values gives the wrong method even when the numbers happen to match.

### Dates

- A date minus a date gives an interval (`206 days 10:00:00`). To get just the number: `EXTRACT(DAY FROM d2 - d1)`, or cast to date first: `d2::DATE - d1::DATE`.
- `EXTRACT` works on the result of a subtraction, not only on columns.
- `YYYY-MM-DD` is the safe, unambiguous date format.
- `< '2022-08-10'` excludes the whole of the 10th. To include it, use `< '2022-08-11'`.
- Filtering a whole month in one line: see section 9.

### Output tricks

- `SELECT *` is wrong when the question names output columns.
- A subquery in `FROM` needs an alias, even if you never use it.
- An `ORDER BY` inside a subquery does nothing. Only the outer `ORDER BY` counts.
- Give aggregates an alias (`AS total_orders`) instead of relying on the default name (`count`).
- Indentation and line breaks do not matter to SQL. They are for people.

---

## 1. SELECT and DISTINCT

```sql
SELECT user_id FROM users;            -- chosen columns
SELECT * FROM users;                  -- all columns
SELECT DISTINCT coin FROM sales;      -- unique values only
SELECT DISTINCT user_id, status FROM trades;   -- unique pairs
SELECT COUNT(DISTINCT user_id) FROM trades;    -- number of unique users
```

---

## 2. WHERE: filtering

```sql
SELECT * FROM people WHERE age >= 20 AND age <= 28;
SELECT * FROM people WHERE age BETWEEN 20 AND 28;       -- same, inclusive
SELECT * FROM people WHERE age < 20 OR age > 28;
SELECT * FROM people WHERE NOT status = 'retired';      -- same as status != 'retired'
SELECT * FROM objects WHERE colorful IS TRUE;
```

| Operator | Meaning |
|---|---|
| `=` | equals |
| `!=`, `<>` | not equal |
| `<`, `>`, `<=`, `>=` | comparisons |
| `BETWEEN a AND b` | within a range, inclusive |
| `IN (...)` | matches any value in a list (shorthand for several `OR`s) |
| `LIKE` | pattern match |

### IN

```sql
WHERE manufacturer IN ('Biogen', 'Bayer', 'Eli Lilly')
WHERE units_sold IN (1000, 1002)       -- numbers need no quotes
```

### LIKE

`%` matches any number of characters, `_` matches exactly one.

| Pattern | Finds |
|---|---|
| `'a%'` | starts with a |
| `'%a'` | ends with a |
| `'%ae%'` | contains ae |
| `'_b%'` | b is the 2nd letter |
| `'a%o'` | starts with a, ends with o |
| `'a___'` | a followed by exactly 3 characters |

For autocomplete-style matching ("starts with what was typed"), use a prefix pattern `'bid%'`, not `'%bid%'`.

### Combining conditions

Parentheses control what is evaluated together.

```sql
WHERE (color = 'brown' AND price = 2)
   OR (color = 'mixed' AND weight < 40)

WHERE age BETWEEN 18 AND 22
  AND state IN ('Victoria', 'Tasmania')
  AND (name LIKE 'A%' OR name LIKE 'B%')
```

---

## 3. NULL

```sql
SELECT * FROM people WHERE age IS NULL;
SELECT * FROM people WHERE age IS NOT NULL;

SELECT COALESCE(rating, 0) FROM books;               -- replace NULL with 0
SELECT COALESCE(rating, avg_rating, 0) FROM books;   -- chain of fallbacks
SELECT IFNULL(rating, 0) FROM books;                 -- MySQL/SQLite, 2 arguments only
```

`COALESCE` and `IFNULL` behave the same with two arguments. `COALESCE` just has more slots.

---

## 4. ORDER BY, LIMIT, OFFSET

```sql
SELECT * FROM people ORDER BY age;                    -- ascending is the default
SELECT * FROM people ORDER BY age DESC;
SELECT * FROM people ORDER BY dept, salary DESC;      -- dept A-Z, then salary high to low
SELECT name, salary FROM staff ORDER BY 2;            -- by column position in SELECT
SELECT * FROM staff ORDER BY CONCAT(territory, name); -- by an expression
```

- Each column gets its own direction. `DESC` on one column does not carry to the next.
- Writing `ASC` is allowed but redundant.
- Position numbers are fragile: reorder `SELECT` and they silently point somewhere else.

```sql
SELECT * FROM people ORDER BY age LIMIT 10;               -- first 10 rows
SELECT * FROM people ORDER BY age LIMIT 10 OFFSET 20;     -- skip 20, then take 10
```

- `LIMIT 0` returns no rows.
- "Top N" means `ORDER BY ... DESC LIMIT N`. `LIMIT` always comes after `ORDER BY`.

Control where NULLs go (PostgreSQL, Oracle): `ORDER BY col NULLS FIRST` or `NULLS LAST`.

---

## 5. Aggregates, GROUP BY, HAVING

```sql
SELECT COUNT(*), SUM(total), MIN(price), MAX(price), AVG(price) FROM orders;

SELECT status, COUNT(*) FROM people GROUP BY status;
SELECT ticker, AVG(open), MAX(close) FROM stocks GROUP BY ticker ORDER BY ticker;

-- group by an expression
SELECT EXTRACT(YEAR FROM date) AS yr, ticker, ROUND(AVG(open), 2) AS avg_open
FROM stocks
GROUP BY yr, ticker;             -- or GROUP BY 1, 2

-- HAVING filters groups
SELECT ticker, AVG(open) FROM stocks GROUP BY ticker HAVING AVG(open) > 200;
SELECT ticker FROM stocks GROUP BY ticker HAVING AVG(open) > 200 AND MIN(open) > 100;
```

Clause order: `SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT`.

`HAVING` without `GROUP BY` works like `WHERE`.

### Counting with a condition

```sql
-- one number: WHERE + COUNT
SELECT COUNT(*) FROM actors WHERE followers >= 500000;

-- several numbers side by side: CASE inside the aggregate
SELECT platform,
       COUNT(CASE WHEN followers >= 500000 THEN 1 END) AS popular,
       COUNT(CASE WHEN followers <  500000 THEN 1 END) AS less_popular
FROM actors
GROUP BY platform;
```

No `GROUP BY` is needed if you only want one overall row.

### Finding duplicates

```sql
-- which combinations repeat, and how often
SELECT col1, col2, COUNT(*) AS times_seen
FROM t
GROUP BY col1, col2
HAVING COUNT(*) > 1;

-- keep every row, tagged with its group size (window function)
SELECT *, COUNT(*) OVER (PARTITION BY col1, col2) AS dup_count
FROM t;
```

Group by the grouped column(s) *and* show them in `SELECT`, or the counts have no labels.

---

## 6. Arithmetic and math functions

```sql
SELECT salary + bonus AS total_comp FROM employees;
SELECT revenue - expenses AS profit FROM sales;
SELECT units * price AS revenue FROM orders;
SELECT gdp / population AS gdp_per_capita FROM stats;
SELECT 23 % 6;       -- 5   (remainder)
SELECT 10 ^ 2;       -- 100 (power; not standard everywhere, POWER(10, 2) is safer)
```

Order of operations follows PEMDAS: parentheses, exponents, multiply/divide, add/subtract.

### Modulus (`%`)

`a % b` is the remainder after dividing `a` by `b`. It repeats in a cycle of length `b`.

- Even or odd: `n % 2 = 0` is even, `n % 2 = 1` is odd.
- Every 3rd row: `row_num % 3 = 0`.
- Buckets: `emp_id % 3` gives 0, 1 or 2, so you can split rows into 3 groups and `GROUP BY emp_id % 3`.

### Math functions

```sql
ABS(close - open)               -- absolute value
ROUND(AVG(close), 2)            -- round to 2 decimals
CEIL(high)    FLOOR(high)       -- round up / down
POWER(close, 2)                 -- squared
MOD(close, 5)                   -- same as close % 5
```

`ABS(SUM(x))` and `SUM(ABS(x))` are the same only when every row has the same sign.

### Percentages

```sql
SELECT ROUND(part * 100.0 / total, 1) AS pct FROM stats;
```

Both `0.50` and `50.0` are acceptable ways to show a percentage. Choose by context.

---

## 7. CASE

```sql
SELECT actor,
       CASE
         WHEN engagement >= 8.0 THEN 'High'
         WHEN engagement >= 6.0 THEN 'Moderate'
         ELSE 'Low'
       END AS engagement_category
FROM actors;
```

- Conditions run top to bottom. The first true one wins.
- `END AS alias` names the whole `CASE`, not one branch.
- Use `BETWEEN low AND high` (smaller number first) or a comparison such as `< 6`. Do not combine them (`BETWEEN < 6` is invalid).

### CASE in WHERE

```sql
WHERE CASE
        WHEN platform = 'Instagram' THEN followers >= 500000
        WHEN platform = 'Twitter'   THEN followers >= 200000
        ELSE followers >= 100000
      END
```

### CASE inside SUM and AVG

```sql
SELECT platform,
       SUM(CASE WHEN rate >= 8.0 THEN followers ELSE 0 END) AS high_followers,
       AVG(CASE WHEN rate >= 8.0 THEN followers END)        AS avg_high_followers
FROM actors
GROUP BY platform;
```

### CASE with a join

The join builds the wide rows first. `CASE` then reacts to columns from either table. `WHEN right_col IS NULL` catches rows that found no match.

```sql
SELECT u.name,
       CASE
         WHEN o.amount >= 100 THEN 'Big Spender'
         WHEN o.amount IS NULL THEN 'No Orders'
         ELSE 'Regular'
       END AS customer_type
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id;
```

---

## 8. Joins

| Join | Returns |
|---|---|
| `INNER JOIN` (or `JOIN`) | only rows with a match on both sides |
| `LEFT JOIN` | all left rows, plus matches from the right (`NULL` where none) |
| `RIGHT JOIN` | all right rows, plus matches from the left. Rarely used: swap the tables and use `LEFT JOIN` |
| `FULL OUTER JOIN` | all rows from both sides, `NULL` where no match |

```sql
SELECT artists.name, songs.title
FROM artists
JOIN songs ON artists.artist_id = songs.artist_id;

-- count per group after a join
SELECT u.city, COUNT(t.order_id) AS total_orders
FROM trades t
JOIN users u ON t.user_id = u.user_id
WHERE t.status = 'Completed'
GROUP BY u.city
ORDER BY total_orders DESC
LIMIT 3;
```

### Anti-join: rows with no match

```sql
-- pages with zero likes (two equivalent ways)
SELECT p.page_id
FROM pages p LEFT JOIN page_likes l ON p.page_id = l.page_id
WHERE l.user_id IS NULL;

SELECT p.page_id
FROM pages p LEFT JOIN page_likes l ON p.page_id = l.page_id
GROUP BY p.page_id
HAVING COUNT(l.user_id) = 0;
```

When a page has no likes, `LEFT JOIN` keeps the page and fills every `likes` column with `NULL`. That is why `IS NULL` works.

### Full outer join with one clean key

```sql
SELECT COALESCE(a.user_id, b.user_id) AS user_id,
       CASE
         WHEN b.paid IS NULL THEN 'CHURN'        -- no payment record
         WHEN a.status IS NULL THEN 'NEW'        -- not in the advertiser table
         WHEN a.status = 'CHURN' THEN 'RESURRECT'
         ELSE 'EXISTING'
       END AS new_status
FROM advertiser a
FULL OUTER JOIN daily_pay b ON a.user_id = b.user_id
ORDER BY user_id;
```

The `CASE` order is deliberate: checking the "no payment" case first means later branches never need to re-check it.

### Conditions in ON

```sql
FROM goodreads g
INNER JOIN orders o
  ON g.book_id = o.book_id
 AND o.quantity > 2
```

`ON` is checked per candidate pair of rows (one from each side), not per table. `OR` in an `ON` clause creates matches you did not intend. Use `AND`.

### Self join

```sql
-- employees who earn more than their manager
SELECT emp.employee_id, emp.name
FROM employee AS emp
JOIN employee AS mgr ON emp.manager_id = mgr.employee_id
WHERE emp.salary > mgr.salary;
```

- The top person has `manager_id = NULL`, so an inner join drops them. That is correct here.
- The same result with a correlated subquery:

```sql
SELECT emp.employee_id, emp.name
FROM employee AS emp
WHERE emp.salary > (SELECT mgr.salary FROM employee mgr WHERE mgr.employee_id = emp.manager_id);
```

Use a self join when you need columns from both sides. Use a correlated subquery when you only need one looked-up value to compare against.

---

## 9. Dates

```sql
CURRENT_DATE            -- today
CURRENT_TIME            -- time now
CURRENT_TIMESTAMP       -- date and time now (also NOW())
```

### Comparing

```sql
WHERE sent_date >  '2022-08-10'
WHERE sent_date >= '2022-08-10 00:00:00'
```

A plain `DATE` column needs no time. A `TIMESTAMP` compared to `'2022-08-10'` is treated as midnight.

### EXTRACT and DATE_PART

```sql
EXTRACT(YEAR  FROM sent_date)      DATE_PART('year',  sent_date)
EXTRACT(MONTH FROM sent_date)      DATE_PART('month', sent_date)
EXTRACT(DAY   FROM sent_date)      EXTRACT(HOUR FROM sent_date)
```

### Filtering a whole month

```sql
WHERE DATE_TRUNC('month', sent_date) = '2022-08-01'                  -- readable
WHERE TO_CHAR(sent_date, 'YYYY-MM') = '2022-08'                      -- readable
WHERE sent_date >= '2022-08-01' AND sent_date < '2022-09-01'         -- best for speed
WHERE EXTRACT(YEAR FROM sent_date) = 2022 AND EXTRACT(MONTH FROM sent_date) = 8
```

The date-range version compares the raw column, so the database can use an index. The others run a function on every row. Extracting only the month merges the same month from different years, so also check the year if the data spans several.

### DATE_TRUNC

Rounds a timestamp down to the start of a unit.

```sql
DATE_TRUNC('month', ts)    -- 2022-08-03 16:43 becomes 2022-08-01 00:00
DATE_TRUNC('day', ts)
DATE_TRUNC('hour', ts)
```

### Date differences

```sql
MAX(post_date) - MIN(post_date)                      -- interval, e.g. 206 days 10:00:00
EXTRACT(DAY FROM MAX(post_date) - MIN(post_date))    -- just 206
MAX(post_date::DATE) - MIN(post_date::DATE)          -- whole days as an integer
```

### INTERVAL

```sql
sent_date + INTERVAL '2 days'
sent_date - INTERVAL '10 minutes'
sent_date + INTERVAL '2 weeks'
sent_date + INTERVAL '1 year 2 months 3 days'
```

Valid units include seconds, minutes, hours, days, weeks, months, years, decades, centuries and millennia. Days and hours are fixed lengths. Months and years follow the calendar (adding `1 month` to Jan 31 lands on the end of February).

### TO_CHAR (date to text)

| Format | Example output |
|---|---|
| `'YYYY-MM-DD HH24:MI:SS'` | `2023-08-27 14:30:00` |
| `'YYYY-MM-DD HH:MI:SS AM'` | `2023-08-27 02:30:00 PM` |
| `'Month DDth, YYYY'` | `August 27th, 2023` |
| `'Mon DD, YYYY'` | `Aug 27, 2023` |
| `'DD Month YYYY'` | `27 August 2023` |
| `'Month'` | `August` |
| `'Day'` | `Saturday` |

### Converting types

```sql
sent_date::DATE                       sent_date::TIMESTAMP
TO_DATE('2023-08-27', 'YYYY-MM-DD')   TO_TIMESTAMP('2023-08-27 10:30:00', 'YYYY-MM-DD HH:MI:SS')
```

---

## 10. Subqueries and CTEs

A subquery and a CTE are both a query whose result is used by another query. The difference is where you write it and whether it has a name.

### Subquery: in parentheses

```sql
-- compare to one value
SELECT name FROM concerts
WHERE revenue > (SELECT AVG(revenue) FROM concerts);

-- compare to a list
SELECT name FROM concerts
WHERE artist_id IN (SELECT artist_id FROM big_acts);

-- extra computed column
SELECT name, revenue, (SELECT AVG(revenue) FROM concerts) AS avg_revenue
FROM concerts;

-- temporary table (alias required)
SELECT COUNT(company_id)
FROM (SELECT company_id FROM jobs GROUP BY company_id, title HAVING COUNT(*) > 1) AS dupes;
```

- `WHERE` cannot use aggregates directly, so a subquery is the way to compare against one.
- `IN` needs a list or a subquery on its right side. `NOT IN` and `EXISTS` work similarly.
- The alias after a `FROM` subquery is required.
- The inner query runs first and hands its result to the outer query.

### Correlated subquery

The inner query uses a value from the outer row, so it runs once per outer row.

```sql
SELECT name, genre, revenue
FROM concerts AS c1
WHERE revenue = (SELECT MAX(revenue) FROM concerts AS c2 WHERE c2.genre = c1.genre);
```

Spot one: the inner query mentions the outer alias (`c1.genre`). Without that link it would return one global max instead of one per genre.

### CTE: named and written first

```sql
WITH genre_totals AS (
  SELECT genre, SUM(revenue) AS total_revenue
  FROM concerts
  GROUP BY genre
)
SELECT g.genre, g.total_revenue, AVG(c.revenue) AS avg_revenue
FROM genre_totals AS g
JOIN concerts AS c ON g.genre = c.genre
GROUP BY g.genre, g.total_revenue;
```

- The CTE is everything inside `WITH name AS ( ... )`. It ends at the closing `)`. The query after it is the main query.
- Two names are involved: the CTE's name, and an optional alias for it in the main query's `FROM`.
- A table alias is optional when the tables are different. It is required in a self join. Once you alias a table, use the alias everywhere.
- Even without an alias, a column that exists in both tables needs a prefix, or SQL reports "ambiguous".

### Which to use

| Situation | Use |
|---|---|
| Compare to one value or a list in `WHERE` | subquery |
| Add a computed column | subquery |
| "For each row, find..." | correlated subquery |
| Long query in several steps | CTE |
| Same intermediate result used more than once | CTE |
| Walk a hierarchy of unknown depth | recursive CTE (later) |

### Recursive CTE (later topic)

A CTE that refers to itself so it can repeat, used to walk hierarchies such as "everyone under this manager, at any depth". It has two parts joined by `UNION ALL`: an anchor query (the starting rows) and a recursive step that joins to the CTE itself, repeating until no new rows appear. PostgreSQL needs `WITH RECURSIVE`.

---

## 11. Window function preview

A window function works like `GROUP BY` that does not collapse the rows. `PARTITION BY` splits rows into buckets, and every row stays visible.

```sql
-- one row per id: the row with the highest salary
SELECT id, first_name, last_name, department_id, salary AS current_salary
FROM (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY id ORDER BY salary DESC) AS rn
  FROM employee_salaries
) AS ranked
WHERE rn = 1
ORDER BY id;
```

- `PARTITION BY id`: a separate bucket per id.
- `ORDER BY salary DESC`: sort each bucket, highest first.
- `ROW_NUMBER()`: number the rows 1, 2, 3... in each bucket, so the max-salary row is `1`.
- It is wrapped in a subquery because `WHERE` cannot see `rn` at the same level (execution order).
- `ROW_NUMBER` always gives exactly one row `1`, even on ties. That fits "if tied, return any one".
- PostgreSQL shortcut: `SELECT DISTINCT ON (id) ... ORDER BY id, salary DESC`.
- Related: `RANK()`, `DENSE_RANK()`, `LAG()`, `LEAD()` (to be added).
