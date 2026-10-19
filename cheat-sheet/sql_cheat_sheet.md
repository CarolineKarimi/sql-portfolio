# SQL Cheat Sheet

A personal reference of tricks, patterns and syntax that I add to as I learn. Each section has the same shape: **what it does** in one or two lines, a **code block** you can skim (one idea per line, with a comment), then **remember** notes for the traps. Examples use made-up tables (`employees`, `orders`, `customers`, `posts` and so on) and PostgreSQL syntax unless noted.

> Running SQL in SQLite (for example in a Jupyter notebook)? SQLite has no `EXTRACT`, `DATE_TRUNC`, `INTERVAL` or `::`. Use `strftime()` and `date()` instead (cheat sheet in section 9).

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
12. [Interview practice tips](#12-interview-practice-tips)

---

## 0. Tricks to remember

Skim this section before starting any new question. These are the mistakes I actually made.

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
- `WHERE COUNT(post_id) > 2` is an error: that is a `HAVING`.
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
- If the question says "there are no duplicates", you do not need `DISTINCT`.

### First / last

- "First", "earliest", "oldest" means `MIN`. "Last", "latest", "most recent" means `MAX`.
- `MIN` and `MAX` work on dates, text and numbers, not only numbers.

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

Deposits and withdrawals, credits and debits, purchases and refunds: one amount column, opposite meanings. Signals: "final balance", "net", "total after ...".

```sql
SELECT account_id,
       SUM(CASE WHEN transaction_type = 'Deposit' THEN amount
                ELSE -amount END) AS final_balance
FROM transactions
GROUP BY account_id;
```

Question to ask: "Does every row add to the total in the same way?" If not, put a `CASE` inside the `SUM`. `CASE` decides what each row contributes, `SUM` adds the contributions. Deposit 100, withdrawal 30, deposit 50 contribute +100, -30, +50, so the sum is 120.

### Top row per group ("highest X for each Y")

Ask three questions, in order:

1. Is there a "for each" something? Then I am working per group.
2. Is there a "top / highest / latest" inside it? Then I am picking a winner per group.
3. Do I need columns beyond the group and the winning value? Then I need the whole row, and `GROUP BY` alone will not work.

Yes to all three means top row per group. Only the group and the max (`genre, MAX(revenue)`) is plain `GROUP BY`. Extra columns that belong to the winning row (`artist_name`) means you need the whole row.

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

Ties for the max return both rows in options 1 and 2. `ROW_NUMBER()` returns one. Same shape: "current salary per employee", "top artist per genre", "most recent post per user".

### Self join

- SQL only compares columns within one row. To compare a row with another row of the same table, join the table to itself so the two rows sit side by side.
- Aliases make one table behave like two: `FROM employee AS emp JOIN employee AS mgr` is like photocopying the table and handing one copy to each role. Name them by role (`emp`, `mgr`), not `a` and `b`.
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

**What it does:** `SELECT` chooses which columns come back, `FROM` says which table to read. Every query starts here. `DISTINCT` removes repeated rows from the result.

```sql
SELECT user_id FROM users;                      -- chosen column
SELECT user_id, name FROM users;                -- several columns, comma separated
SELECT * FROM users;                            -- all columns
SELECT DISTINCT coin FROM sales;                -- unique values only (duplicates removed)
SELECT DISTINCT user_id, status FROM trades;    -- unique PAIRS: dropped only if both columns match
SELECT COUNT(DISTINCT user_id) FROM trades;     -- how many different users
SELECT category, COUNT(DISTINCT product)        -- unique products inside each category
FROM product_spend GROUP BY category;
```

**Remember**

- `*` is for exploring. When the question names the output columns, list exactly those.
- `DISTINCT` looks at the whole row of selected columns, not just the first column.
- `DISTINCT` keeps `NULL` as one value. To drop it: `WHERE col IS NOT NULL`.
- `DISTINCT` is for the output. `COUNT(DISTINCT col)` is for counting inside an aggregate.
- Mixing `category` with `COUNT(DISTINCT product)` needs `GROUP BY category` (section 5).

---

## 2. WHERE: filtering

**What it does:** `WHERE` keeps only the rows that pass a condition. It runs before grouping, so it works on individual rows.

```sql
SELECT event_id FROM events WHERE people < 14;              -- basic condition
SELECT * FROM people WHERE age >= 20 AND age <= 28;         -- inside a range
SELECT * FROM people WHERE age BETWEEN 20 AND 28;           -- same thing, inclusive
SELECT * FROM people WHERE age < 20 OR age > 28;            -- outside a range
SELECT * FROM people WHERE NOT status = 'retired';          -- exclude a value (same as status != 'retired')
SELECT * FROM objects WHERE colorful IS TRUE;               -- true/false column
```

| Operator | Meaning | Example |
|---|---|---|
| `=` | equals | `user_id = 2` |
| `!=`, `<>` | not equal | `user_id != 5` |
| `<`, `>`, `<=`, `>=` | comparisons | `age >= 2` |
| `BETWEEN a AND b` | within a range, inclusive | `pushups BETWEEN 20 AND 40` |
| `IN (...)` | matches any value in a list | `first_name IN ('Bob', 'Ann')` |
| `LIKE` | pattern match | `first_name LIKE 'Jo%'` |
| `IS NULL` | no value (section 3) | `age IS NULL` |

### BETWEEN

**What it does:** selects values inside a range. Works for numbers, text and dates. Both ends are included. It is always paired with `AND`.

```sql
SELECT * FROM medals WHERE type = 'GOLD' AND year BETWEEN 2000 AND 2010;   -- 2000 and 2010 both count
SELECT * FROM sales WHERE units_sold NOT BETWEEN 55000 AND 550000;          -- outside the range
```

- Smaller number first. `BETWEEN 10 AND 5` returns nothing.
- Do not mix it with a comparison (`BETWEEN < 6` is invalid). Use `< 6` on its own.

### IN

**What it does:** a shortcut for several `OR`s on the same column.

```sql
WHERE manufacturer IN ('Biogen', 'Bayer', 'Eli Lilly')   -- same as three ORs
WHERE manufacturer NOT IN ('Roche', 'Bayer')             -- exclude a list
WHERE units_sold IN (1000, 1002)                         -- numbers need no quotes
WHERE id IN (SELECT id FROM vip_customers)               -- list from a subquery (section 10)
```

### LIKE

**What it does:** matches text against a pattern. `%` means any number of characters (including none). `_` means exactly one character.

| Pattern | Finds |
|---|---|
| `'a%'` | starts with a |
| `'%a'` | ends with a |
| `'%ae%'` | contains ae |
| `'_b%'` | b is the 2nd letter |
| `'_ee%'` | 2nd and 3rd letters are e |
| `'a%o'` | starts with a, ends with o |
| `'a___'` | a followed by exactly 3 characters |

```sql
SELECT product_id, drug FROM sales WHERE drug LIKE '%Relief%';   -- contains "Relief"
SELECT * FROM customers WHERE customer_name LIKE 'F%k';          -- starts with F, ends with k
```

- For autocomplete-style matching ("starts with what was typed"), use a prefix pattern `'bid%'`, not `'%bid%'`.
- `LIKE` is case-sensitive in PostgreSQL. `ILIKE` ignores case.

### Combining conditions

**What it does:** join conditions with `AND` (both true) and `OR` (either true). Parentheses control what is evaluated together. `AND` binds tighter than `OR`, so when you mix them, always add parentheses.

```sql
WHERE (color = 'brown' AND price = 2)                -- brown and 2 ...
   OR (color = 'mixed' AND weight < 40)              -- ... or mixed and light

WHERE (age > 30 AND status = 'retired')
   OR status = 'seeking'

WHERE age BETWEEN 18 AND 22                          -- every piece must hold ...
  AND state IN ('Victoria', 'Tasmania')
  AND gender != 'n/a'                                -- 'n/a' is text, not NULL
  AND (name LIKE 'A%' OR name LIKE 'B%')             -- ... except this OR is grouped in parentheses
```

**Remember**

- Read the question for each condition: "isn't n/a" is `!= 'n/a'`, not `IS NOT NULL`.
- "Live in Victoria or Tasmania" is the `state` column, not `city`.
- Filtering an aggregate (`COUNT`, `SUM`) is `HAVING`, not `WHERE` (section 5).

---

## 3. NULL

**What it does:** `NULL` means the field has no value. It is not `0` and not an empty string. You cannot test it with `=`. Use `IS NULL`.

```sql
SELECT * FROM people WHERE age IS NULL;                    -- rows with no value
SELECT * FROM people WHERE age IS NOT NULL;                -- rows that have a value
SELECT DISTINCT name FROM people WHERE name IS NOT NULL;   -- unique names, blanks removed
SELECT part, step FROM assembly WHERE finish_date IS NULL; -- "no finish date" means unfinished

SELECT COALESCE(rating, 0) FROM books;                     -- NULL becomes 0, others unchanged
SELECT COALESCE(rating, avg_rating, 0) FROM books;         -- fallback chain: rating, then avg_rating, then 0
SELECT IFNULL(rating, 0) FROM books;                       -- same as 2-argument COALESCE (MySQL / SQLite)
```

**Remember**

- `COALESCE(a, b, c)` returns the first non-null value. It works in all databases and takes any number of arguments.
- `IFNULL(a, b)` takes exactly 2 arguments. For one fallback they behave the same.
- A `NULL` row fails every comparison (`NULL = 'x'` is never true), so `WHERE x != 'a'` also drops the `NULL` rows.
- `COUNT(col)` skips `NULL`s, `COUNT(*)` does not.
- Where `NULL`s land in a sort: see section 4.

---

## 4. ORDER BY, LIMIT, OFFSET

**What it does:** `ORDER BY` sorts the result. `LIMIT` keeps only the first N rows after sorting. `OFFSET` skips rows first, which is useful for pages. Order in the query: `ORDER BY`, then `LIMIT`, then `OFFSET`.

```sql
SELECT * FROM people ORDER BY age;                      -- ascending (A-Z, 0-9) is the default
SELECT * FROM people ORDER BY age DESC;                 -- descending (largest / last first)
SELECT * FROM people ORDER BY dept, salary DESC;        -- dept A-Z first, ties broken by salary high to low
SELECT id FROM feathers ORDER BY area DESC, weight ASC; -- each column has its own direction
SELECT name, salary FROM staff ORDER BY 2;              -- by position in SELECT (2 = salary)
SELECT * FROM staff ORDER BY CONCAT(territory, name);   -- by an expression
```

```sql
SELECT * FROM people LIMIT 10;                                      -- first 10 rows, quick look
SELECT * FROM cities ORDER BY avg_temp LIMIT 5;                     -- 5 coldest: sort ascending, then cut
SELECT id FROM employees WHERE status = 'married'
ORDER BY salary LIMIT 4;                                            -- filter, sort, cut: the 4 lowest-paid
SELECT * FROM people ORDER BY age LIMIT 10 OFFSET 20;               -- skip 20 rows, then take 10
```

```sql
-- "suspicious if ANY rule matches" = OR between the rules
SELECT id, name FROM files
WHERE size_mb < 1 OR size_mb > 100 OR created_year < 2000 OR name IS NULL
ORDER BY id DESC;
```

**Remember**

- Each column gets its own direction. `DESC` on the first column does not carry to the second.
- Writing `ASC` is allowed but redundant.
- "Top N highest" is `ORDER BY ... DESC LIMIT N`. "N lowest" is `ORDER BY ... LIMIT N` (ascending).
- `LIMIT 0` returns no rows. `LIMIT` always comes after `ORDER BY`.
- Position numbers (`ORDER BY 2`) are fragile: reorder `SELECT` and they silently point somewhere else. A position larger than the number of `SELECT` columns is an error.
- `ORDER BY` runs after `SELECT`, so it can use aliases.

### Where NULLs go in a sort

| Database | `NULL` in `ASC` | Treats `NULL` as |
|---|---|---|
| PostgreSQL, Oracle | last (first in `DESC`) | the largest value |
| MySQL, SQL Server, SQLite | first | the smallest value |

```sql
ORDER BY col NULLS FIRST       -- PostgreSQL, Oracle: choose explicitly
ORDER BY col NULLS LAST
```

---

## 5. Aggregates, GROUP BY, HAVING

**What it does:** aggregate functions collapse many rows into one summary value. `GROUP BY` does that once per group instead of once for the whole table. `HAVING` filters the groups.

| Function | Returns |
|---|---|
| `COUNT(*)` | number of rows |
| `COUNT(col)` | number of non-null values in `col` |
| `SUM(col)` | total |
| `MIN(col)` / `MAX(col)` | lowest / highest (works on dates and text too) |
| `AVG(col)` | average (ignores `NULL`s) |

```sql
SELECT COUNT(*) FROM people;                                    -- number of rows
SELECT COUNT(product_id), SUM(total_sales)
FROM sales WHERE manufacturer = 'Acme';                       -- filter first, then aggregate
SELECT MIN(age), MAX(age) FROM people;                          -- youngest and oldest
SELECT AVG(open) FROM stocks WHERE ticker = 'GOOG';             -- average of one stock
SELECT ABS(SUM(cogs - total_sales)) AS total_loss FROM sales;   -- ABS wraps the total
```

### GROUP BY

**What it does:** separates the rows into groups (one per distinct value) and runs the aggregate on each group.

```sql
SELECT category, SUM(spend) FROM product_spend GROUP BY category;       -- total per category
SELECT status, COUNT(*) FROM people GROUP BY status;                    -- how many per status
SELECT ticker, AVG(open), MAX(close) FROM stocks
GROUP BY ticker ORDER BY ticker;                                        -- several aggregates, one group column
SELECT ticker, MIN(open) AS price FROM stocks
GROUP BY ticker ORDER BY price DESC;                                    -- ORDER BY can use the alias
SELECT skill, COUNT(candidate_id) AS n FROM candidates
GROUP BY skill ORDER BY n DESC;                                         -- head-count per skill, biggest first

-- group by an expression
SELECT ticker, EXTRACT(YEAR FROM date) AS yr, ROUND(AVG(open), 2) AS avg_open
FROM stocks
GROUP BY ticker, yr;                  -- PostgreSQL allows the alias; GROUP BY 1, 2 works everywhere
```

**Remember**

- Every plain (non-aggregated) column in `SELECT` must also be in `GROUP BY`. An aggregate returns one value per group, a plain column has many values per group, and SQL does not know which one to show next to it.
- Without `GROUP BY` the whole table is one group, so `SELECT category, COUNT(DISTINCT product)` fails.
- Group by the column you want a row for. Show it in `SELECT` too, or the numbers have no labels.
- `GROUP BY` collapses rows with identical values into one row, so it can double as a way to remove duplicates for certain columns. `DISTINCT` is the direct tool.

### HAVING

**What it does:** filters groups after the aggregate has been computed. Use it whenever the condition contains `COUNT`, `SUM`, `AVG`, `MIN` or `MAX`.

```sql
SELECT ticker, AVG(open) FROM stocks
GROUP BY ticker HAVING AVG(open) > 200;                         -- groups whose average is over 200
SELECT ticker FROM stocks
GROUP BY ticker HAVING AVG(open) > 200 AND MIN(open) > 100;     -- several conditions, like WHERE
SELECT ticker, MIN(open) FROM stocks
GROUP BY ticker HAVING MIN(open) > 100;                         -- "always above 100" = the minimum is above 100
SELECT status, COUNT(*) FROM people
GROUP BY status HAVING COUNT(*) > 5;                            -- groups with more than 5 rows
SELECT candidate_id FROM candidates
GROUP BY candidate_id HAVING COUNT(skill) >= 3;                 -- "at least 3" is >= 3, not > 3
```

| | `WHERE` | `HAVING` |
|---|---|---|
| Filters | individual rows | groups |
| When | before grouping | after grouping |
| Can use an aggregate | no | yes |
| Can use a `SELECT` alias | no | no (repeat the expression) |
| Example | `WHERE followers > 1000` | `HAVING AVG(followers) > 100` |

- Clause order: `SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT`.
- One-line rule: if the condition uses an aggregate, it goes in `HAVING`. If it is a plain row comparison, it goes in `WHERE`.
- `HAVING` without `GROUP BY` is allowed and behaves like `WHERE`.
- To use a total in `WHERE`, calculate it in a subquery or CTE first (section 10).

### All of a list of values (pattern)

```sql
-- candidates who have ALL 3 required skills
SELECT candidate_id
FROM candidates
WHERE skill IN ('Python', 'Tableau', 'PostgreSQL')   -- keep only the relevant rows
GROUP BY candidate_id
HAVING COUNT(DISTINCT skill) = 3                     -- they matched all 3
ORDER BY candidate_id;
```

### Finding duplicates

```sql
-- which combinations repeat, and how often
SELECT col1, col2, COUNT(*) AS times_seen
FROM t
GROUP BY col1, col2
HAVING COUNT(*) > 1;

-- how many different ids have a duplicate in col1 + col2
SELECT COUNT(*) FROM (
  SELECT id FROM t GROUP BY id, col1, col2 HAVING COUNT(*) > 1
) AS dupes;

-- keep every row, tagged with its group size (window function)
SELECT *, COUNT(*) OVER (PARTITION BY col1, col2) AS dup_count
FROM t;
```

---

## 6. Arithmetic and math functions

**What it does:** do maths on column values inside a query. You can build a new column from existing ones, filter on it, sort by it or aggregate it.

```sql
SELECT salary + bonus AS total_comp FROM employees;      -- add
SELECT revenue - expenses AS profit FROM sales;          -- subtract
SELECT units * price AS revenue FROM orders;             -- multiply
SELECT gdp / population AS gdp_per_capita FROM stats;    -- divide
SELECT 23 % 6;                                           -- 5   (remainder, "modulus")
SELECT 10 ^ 2;                                           -- 100 (power; POWER(10, 2) works everywhere)
```

| Operator | Description | Example | Result |
|---|---|---|---|
| `+` | addition | `15 + 5` | 20 |
| `-` | subtraction | `15 - 5` | 10 |
| `*` | multiplication | `15 * 5` | 75 |
| `/` | division | `15 / 5` | 3 |
| `%` | modulus (remainder) | `14 % 5` | 4 |
| `^` | exponent (not in every database) | `15 ^ 2` | 225 |
| `-` (prefix) | negation | `-15` | -15 |

Order of operations is the same as in school maths: parentheses first, then exponents, then multiply and divide (left to right), then add and subtract (left to right).

### Using a calculated column

```sql
-- top 3 by profit: calculate, sort, cut
SELECT drug, total_sales - cogs AS profit
FROM drug_sales
ORDER BY profit DESC
LIMIT 3;

-- total loss per manufacturer: the money-losing rows only
SELECT manufacturer,
       COUNT(*)                  AS losing_drugs,
       SUM(cogs - total_sales)   AS total_loss
FROM drug_sales
WHERE cogs - total_sales > 0     -- repeat the expression: WHERE cannot see the alias total_loss
GROUP BY manufacturer
ORDER BY total_loss DESC;

-- biggest swing per card: max minus min per group
SELECT card_name, MAX(issued) - MIN(issued) AS spread
FROM monthly_cards
GROUP BY card_name
ORDER BY spread DESC;

-- count rows where the move is over 10%
SELECT ticker, COUNT(*) AS big_months
FROM stocks
WHERE ABS(close - open) / open > 0.1       -- row-level condition, so WHERE, not HAVING
GROUP BY ticker
ORDER BY big_months DESC;
```

- A column alias cannot be used in `WHERE` or `HAVING`, because those run before `SELECT`. Repeat the expression, or wrap the query in a subquery or CTE and filter on the alias outside.
- Use `COUNT(*)` when you just want the number of matching rows. `COUNT(expression)` counts non-null values only.
- `ABS(SUM(x))` and `SUM(ABS(x))` are the same only when every row has the same sign.

### Modulus (`%`)

`a % b` is the remainder after dividing `a` by `b`. The answer cycles through `0, 1, ... b-1` forever.

```sql
WHERE measurement_num % 2 = 1      -- odd numbers (remainder 1 when divided by 2)
WHERE measurement_num % 2 = 0      -- even numbers (nothing left over)
WHERE row_num % 3 = 0              -- every 3rd row: 3rd, 6th, 9th ...
GROUP BY emp_id % 3                -- split into 3 buckets (0, 1, 2), spread evenly
```

- Even numbers divide exactly by 2. Odd numbers are one more than a multiple of 2, so they always leave 1.
- The remainder can only be `0` to `b-1`, so `% 3` gives exactly 3 buckets however many rows you have.

### Math functions

```sql
ABS(close - open)              -- absolute value (drops the minus sign)
ROUND(AVG(close), 2)           -- round to 2 decimals
CEIL(high)    FLOOR(high)      -- round up / round down to a whole number
POWER(close, 2)                -- squared
MOD(close, 5)                  -- same as close % 5
```

### Division and casting

Integer divided by integer throws away the decimals: `10 / 4` gives `2`.

```sql
SELECT CAST(10 AS DECIMAL) / 4;     -- 2.5   explicit CAST
SELECT 10 / (4 * 1.0);              -- 2.5   multiply one side by 1.0
SELECT 10::DECIMAL / 4;             -- 2.5   :: shortcut (PostgreSQL)
```

- Touch **one** side only. Never both.
- It does not matter which side you cast, as long as one side is decimal.
- `AVG()` already returns a decimal.

### Percentages

```sql
SELECT (part / total) * 100 AS pct FROM stats;                   -- wrong if both are integers (0 * 100)
SELECT ROUND(part * 100.0 / total, 1) AS pct FROM stats;         -- safe: 100.0 forces decimals, 1 decimal place
```

Multiply by `100.0` **before** dividing so the decimals are not lost. Both `0.50` and `50.0` are acceptable ways to show a percentage. Choose by context.

---

## 7. CASE

**What it does:** CASE is SQL's if / else. It checks conditions in order and returns a value for each row. It creates a value in the query result. It does not change the data in the table.

```sql
SELECT actor,
       CASE
         WHEN engagement >= 8.0 THEN 'High'        -- checked first
         WHEN engagement >= 6.0 THEN 'Moderate'    -- only reached if the first was false
         ELSE 'Low'                                -- everything else
       END AS engagement_category                  -- alias names the whole CASE
FROM actors;
```

**Remember**

- Conditions run top to bottom. The first true one wins, so order matters.
- No `ELSE` means a row that matches nothing gets `NULL`.
- `END AS alias` names the whole `CASE`, not one branch. The alias is a name, so use `_` or double quotes, never single quotes.
- Use `BETWEEN low AND high` (smaller number first) or a comparison such as `< 6`. Do not combine them.
- "Categorize", "label" and "classify" all mean a new column. Select the identifying columns plus the `CASE` column. Add `ORDER BY` only if the question asks for sorting.
- Indentation does not matter. Put each `WHEN` on its own line for readability.

### CASE in WHERE

**What it does:** applies a different filter depending on a row's value.

```sql
WHERE CASE
        WHEN platform = 'Instagram' THEN followers >= 500000   -- one rule for Instagram
        WHEN platform = 'Twitter'   THEN followers >= 200000   -- another for Twitter
        ELSE followers >= 100000                               -- default rule
      END
```

### CASE inside aggregates

**What it does:** counts, adds or averages only the rows that fit a condition. Because each `CASE` is its own column, you can compute several conditional numbers side by side in one query.

| Aggregate | Use | Why |
|---|---|---|
| `COUNT` | `ELSE NULL` (or no `ELSE`) | `COUNT` counts every non-null value, so `ELSE 0` would count those rows too |
| `SUM` | `ELSE 0` | adding zero changes nothing |
| `AVG` | `ELSE NULL` (or no `ELSE`) | `NULL`s are ignored, `0`s are averaged in and drag the result down |

```sql
SELECT platform,
       COUNT(CASE WHEN followers >= 500000 THEN 1 ELSE NULL END) AS popular_count,   -- number of rows that qualify
       COUNT(CASE WHEN followers <  500000 THEN 1 ELSE NULL END) AS other_count,
       SUM(CASE WHEN engagement >= 8.0 THEN followers ELSE 0 END) AS high_followers,  -- total of qualifying rows
       AVG(CASE WHEN engagement >= 8.0 THEN followers END)        AS avg_high_followers  -- average of qualifying rows
FROM actors
GROUP BY platform;
```

```sql
-- one overall row: no GROUP BY needed
SELECT SUM(CASE WHEN device = 'laptop' THEN views ELSE 0 END)               AS laptop_views,
       SUM(CASE WHEN device IN ('tablet', 'phone') THEN views ELSE 0 END)   AS mobile_views
FROM viewership;
```

**One number or several?**

| Situation | Use |
|---|---|
| One category, one total in it | `WHERE` + `COUNT` / `SUM` |
| One category, several sub-groups side by side | `COUNT(CASE WHEN ... THEN 1 END)` / `SUM(CASE ...)` per column |
| One row per category | add `GROUP BY` |
| One overall row | no `GROUP BY` |

- `WHERE` throws rows away before counting. `CASE` inside an aggregate keeps every row and routes it to a scoreboard.
- For `SUM`, the `CASE` returns the **column** to add. For `COUNT`, it returns `1` (or any non-null value). For `AVG`, it returns the **numeric column** to average.
- Mixed `+` and `-` rows (deposits and withdrawals): see section 0.
- `CASE` with a join: see section 8.

---

## 8. Joins

**What it does:** a join combines two tables side by side by matching a shared column (usually an id). Say which columns you want, which two tables to join, and what must match.

| Join | Returns |
|---|---|
| `INNER JOIN` (or `JOIN`) | only rows with a match in both tables |
| `LEFT JOIN` | all left rows, plus matches from the right (`NULL` where there is no match) |
| `RIGHT JOIN` | all right rows, plus matches from the left. Rarely used: swap the tables and use `LEFT JOIN` |
| `FULL OUTER JOIN` | all rows from both sides, `NULL` where there is no match |

```sql
SELECT artists.name, songs.title
FROM artists
JOIN songs ON artists.artist_id = songs.artist_id;          -- INNER: only artists that have songs

SELECT o.order_id, d.delivery_id, d.delivery_status
FROM orders o
LEFT JOIN deliveries d ON o.order_id = d.order_id;          -- every order, delivery columns NULL if none

SELECT o.order_id, d.delivery_id, d.delivery_status
FROM deliveries d
RIGHT JOIN orders o ON d.order_id = o.order_id;             -- same result as the LEFT JOIN above, tables swapped

SELECT o.order_id, d.delivery_id, d.delivery_status
FROM orders o
FULL OUTER JOIN deliveries d ON o.order_id = d.order_id;    -- every order AND every delivery

SELECT * FROM sales JOIN menu USING (product_id);           -- shortcut when both tables use the same column name
```

- Think of the order as left to right: the table after `FROM` is the left table, the table after `JOIN` is the right.
- A table alias (`orders o`) is optional when the tables differ, required in a self join. Once you alias a table, use the alias everywhere.

### Join, then aggregate

**What it does:** the join builds wide rows first. After that, `WHERE`, `GROUP BY` and the aggregates work as usual, with columns from both tables.

```sql
-- top 3 cities by completed orders
SELECT u.city, COUNT(t.order_id) AS total_orders
FROM trades t
JOIN users u ON t.user_id = u.user_id        -- join first
WHERE t.status = 'Completed'                 -- filter rows
GROUP BY u.city                              -- group the joined rows
ORDER BY total_orders DESC
LIMIT 3;
```

### Anti-join: rows with no match

**What it does:** finds rows in one table that have nothing in the other. `LEFT JOIN` keeps the row and fills the right-hand columns with `NULL`, so test for `NULL`.

```sql
-- pages with zero likes (two equivalent ways)
SELECT p.page_id
FROM pages p LEFT JOIN page_likes l ON p.page_id = l.page_id
WHERE l.user_id IS NULL;                       -- no like row was found

SELECT p.page_id
FROM pages p LEFT JOIN page_likes l ON p.page_id = l.page_id
GROUP BY p.page_id
HAVING COUNT(l.user_id) = 0;                   -- COUNT of a column skips the NULLs
```

- `LEFT JOIN` and `LEFT OUTER JOIN` are the same.
- A `WHERE` on a right-table column (other than `IS NULL`) turns a `LEFT JOIN` back into an inner join.

### Conditions in ON

**What it does:** the `ON` clause can hold more than the matching columns. Extra `AND` conditions are checked for each candidate pair of rows, one row from each table.

```sql
FROM books g
INNER JOIN orders o
  ON g.book_id = o.book_id          -- the match
 AND o.quantity > 2                 -- extra conditions on the pair
 AND g.year_released > 2015
```

- Use `AND` between conditions. `OR` in an `ON` clause creates matches you did not intend.
- `ON ... AND` and `WHERE` give the same result for an `INNER JOIN`. For a `LEFT JOIN` they differ: a condition in `ON` keeps the left rows, a condition in `WHERE` can remove them.

### Join with CASE

**What it does:** after the join, `CASE` behaves exactly as normal, but can reference columns from both tables. `WHEN right_col IS NULL` catches rows that found no match.

```sql
SELECT u.name,
       CASE
         WHEN o.amount >= 100 THEN 'Big Spender'
         WHEN o.amount IS NULL THEN 'No Orders'        -- LEFT JOIN found no order
         ELSE 'Regular'
       END AS customer_type
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id;
```

### Full outer join with one clean key

**What it does:** when a row can exist in either table, a full outer join keeps both. Each table has its own copy of the key, one of which can be `NULL`, so `COALESCE` picks the one that exists.

```sql
SELECT COALESCE(a.user_id, b.user_id) AS user_id,      -- one clean key column
       CASE
         WHEN b.paid IS NULL THEN 'CHURN'              -- no payment record
         WHEN a.status IS NULL THEN 'NEW'              -- not in the advertiser table
         WHEN a.status = 'CHURN' THEN 'RESURRECT'
         ELSE 'EXISTING'
       END AS new_status
FROM advertiser a
FULL OUTER JOIN daily_pay b ON a.user_id = b.user_id
ORDER BY user_id;
```

- The `CASE` order is deliberate: checking "no payment" first means later branches never need to re-check it.
- The clue to use `FULL OUTER JOIN` is structural: ask whether a row can be in table A only **and** in table B only. If yes, use it.

### Self join

**What it does:** joins a table to itself, so two rows of the same table can be compared.

```sql
-- employees who earn more than their manager
SELECT emp.employee_id, emp.name
FROM employee AS emp
JOIN employee AS mgr ON emp.manager_id = mgr.employee_id    -- "my manager_id is your employee_id"
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

**What it does:** date functions let you read parts of a date, shift it, compare it and format it. Examples are PostgreSQL. SQLite equivalents are at the end of this section.

```sql
CURRENT_DATE                      -- today's date
CURRENT_TIME                      -- time now, no date
CURRENT_TIMESTAMP                 -- date and time now (also NOW())
```

### Comparing dates

```sql
WHERE sent_date >  '2022-08-10'                -- after the 10th (a date column needs no time part)
WHERE sent_date >= '2022-08-10 00:00:00'       -- on or after the 10th, at midnight
WHERE sent_date =  '2022-08-10 17:03:00'       -- exactly this timestamp
```

- Use `YYYY-MM-DD`. It is the safe format that works everywhere.
- You do not always need `00:00:00`. A `TIMESTAMP` compared to `'2022-08-10'` is treated as midnight, so rows later on the 10th count as greater.
- `<= '2022-08-10'` on a timestamp column misses the rest of the 10th. Use `< '2022-08-11'`.

### EXTRACT and DATE_PART

**What it does:** pulls one part (year, month, day, hour...) out of a date as a number.

```sql
EXTRACT(YEAR   FROM sent_date)      DATE_PART('year',   sent_date)
EXTRACT(MONTH  FROM sent_date)      DATE_PART('month',  sent_date)
EXTRACT(DAY    FROM sent_date)      DATE_PART('day',    sent_date)
EXTRACT(HOUR   FROM sent_date)      DATE_PART('hour',   sent_date)
EXTRACT(MINUTE FROM sent_date)      DATE_PART('minute', sent_date)
```

```sql
-- two conditions: year AND month
WHERE EXTRACT(YEAR FROM sent_date) = 2022 AND EXTRACT(MONTH FROM sent_date) = 8

-- month and year in one output column
SELECT EXTRACT(YEAR FROM d) * 100 + EXTRACT(MONTH FROM d) AS yyyymm      -- 202208
SELECT TO_CHAR(d, 'YYYY-MM') AS year_month                                -- '2022-08'
SELECT DATE_TRUNC('month', d) AS month_start                              -- 2022-08-01
```

### Filtering a whole month

```sql
WHERE DATE_TRUNC('month', sent_date) = '2022-08-01'                  -- readable
WHERE TO_CHAR(sent_date, 'YYYY-MM') = '2022-08'                      -- readable
WHERE sent_date >= '2022-08-01' AND sent_date < '2022-09-01'         -- best for speed
WHERE EXTRACT(YEAR FROM sent_date) = 2022 AND EXTRACT(MONTH FROM sent_date) = 8
```

The date-range version compares the raw column, so the database can use an index. The others run a function on every row. Extracting only the month merges the same month from different years, so also check the year if the data spans several.

### Date differences

```sql
MAX(post_date) - MIN(post_date)                      -- an interval, e.g. 206 days 10:00:00
EXTRACT(DAY FROM MAX(post_date) - MIN(post_date))    -- just 206 (the day part)
MAX(post_date::DATE) - MIN(post_date::DATE)          -- whole days as an integer
```

```sql
-- days between each user's first and last post in 2021, for users with at least 2 posts
SELECT user_id,
       EXTRACT(DAY FROM MAX(post_date) - MIN(post_date)) AS days_between
FROM posts
WHERE EXTRACT(YEAR FROM post_date) = 2021
GROUP BY user_id
HAVING COUNT(post_id) >= 2;
```

- "First post / last post" is `MIN` / `MAX` on the date column.
- `EXTRACT(DAY FROM ...)` of an interval gives the days part only. If hours matter, cast to date first.

### DATE_TRUNC

**What it does:** rounds a timestamp **down** to the start of a unit. It trims the finer detail and keeps the unit you ask for.

```sql
DATE_TRUNC('month', ts)    -- 2022-08-03 16:43 becomes 2022-08-01 00:00
DATE_TRUNC('day', ts)      -- 2022-08-03 16:43 becomes 2022-08-03 00:00
DATE_TRUNC('hour', ts)     -- 2022-08-03 16:43 becomes 2022-08-03 16:00
```

Handy for grouping by month: `GROUP BY DATE_TRUNC('month', ts)`.

### INTERVAL

**What it does:** adds or subtracts a span of time.

```sql
sent_date + INTERVAL '2 days'
sent_date - INTERVAL '10 minutes'
sent_date + INTERVAL '2 weeks'
sent_date + INTERVAL '2 months'
sent_date + INTERVAL '2 years'
sent_date + INTERVAL '2 decades'
sent_date + INTERVAL '1 century'
sent_date + INTERVAL '1 millennium'
sent_date + INTERVAL '1 year 2 months 3 days'
```

- Write the number and unit **inside the quotes**: `INTERVAL '2 weeks'`.
- Valid units: seconds, minutes, hours, days, weeks, months, years, decades, centuries, millennia.
- Days and hours are fixed lengths. Months and years follow the calendar (adding `1 month` to Jan 31 lands at the end of February).

### TO_CHAR (date to text)

**What it does:** formats a date or timestamp as a string, using format codes.

| Format | Example output |
|---|---|
| `'YYYY-MM-DD HH24:MI:SS'` | `2023-08-27 14:30:00` |
| `'YYYY-MM-DD HH:MI:SS AM'` | `2023-08-27 02:30:00 PM` |
| `'Month DDth, YYYY'` | `August 27th, 2023` |
| `'Mon DD, YYYY'` | `Aug 27, 2023` |
| `'DD Month YYYY'` | `27 August 2023` |
| `'Month'` | `August` (month name) |
| `'Day'` | `Saturday` (day of week) |

```sql
TO_CHAR(sent_date, 'Mon DD, YYYY')     -- 'Aug 27, 2023'
TO_CHAR(sent_date, 'YYYY-MM')          -- '2023-08', handy for grouping by month
```

### Converting types

**What it does:** turns text into a date or timestamp (or a timestamp into a plain date).

```sql
sent_date::DATE                                                 -- drop the time part
sent_date::TIMESTAMP
TO_DATE('2023-08-27', 'YYYY-MM-DD')                             -- text to date, with the format spelled out
TO_TIMESTAMP('2023-08-27 10:30:00', 'YYYY-MM-DD HH:MI:SS')      -- text to timestamp
```

### SQLite equivalents

SQLite has no `EXTRACT`, `DATE_TRUNC`, `INTERVAL` or `::`.

```sql
strftime('%Y', d)                  -- year as text, e.g. '2022' (wrap in CAST(... AS INTEGER) for a number)
strftime('%m', d)                  -- month '08'
strftime('%Y-%m', d)               -- '2022-08', like TO_CHAR(d, 'YYYY-MM')
date(d, '+7 days')                 -- add 7 days (like d + INTERVAL '7 days')
date(d, 'start of month')          -- like DATE_TRUNC('month', d)
date('now')                        -- like CURRENT_DATE
julianday(d2) - julianday(d1)      -- days between two dates, as a number
```

---

## 10. Subqueries and CTEs

**What it does:** a subquery and a CTE are both a query whose result is used by another query. The only difference is where you write it and whether it has a name.

### Subquery: a query in parentheses

It can sit in four places, each with a plain meaning.

```sql
-- 1) In WHERE: one value to compare against
SELECT artist_name FROM concerts
WHERE revenue > (SELECT AVG(revenue) FROM concerts);

-- 2) In WHERE with IN: a list to compare against
SELECT artist_name FROM concerts
WHERE artist_id IN (SELECT artist_id FROM big_acts);

-- 3) In SELECT: an extra computed column (same value on every row)
SELECT artist_name, revenue,
       (SELECT AVG(revenue) FROM concerts) AS avg_revenue
FROM concerts;

-- 4) In FROM: a temporary table (alias required)
SELECT COUNT(company_id)
FROM (SELECT company_id, title, description, COUNT(job_id) AS job_count
      FROM job_listings
      GROUP BY company_id, title, description
      HAVING COUNT(job_id) > 1) AS count_dups;
```

- The inner query runs first and hands its result to the outer query.
- `WHERE revenue > AVG(revenue)` is illegal, because `WHERE` cannot use aggregates. A subquery works around that.
- `IN` needs a list or a subquery on its right side. `NOT IN` and `EXISTS` work similarly. `manager_id IN (...)` needs a list, so you write the list as a subquery.
- A query in parentheses used like a table needs a name (`count_dups` above). It works exactly like a table alias, applied to a query instead of a real table.
- To compare two values from the same table (employee vs manager), use a self join (section 8) or a correlated subquery.

### Correlated subquery

**What it does:** the inner query uses a value from the outer row, so it runs once per outer row. It is a "for each row, look up..." tool.

```sql
SELECT artist_name, genre, revenue
FROM concerts AS c1
WHERE revenue = (SELECT MAX(revenue)
                 FROM concerts AS c2
                 WHERE c2.genre = c1.genre);    -- link to the outer row: max of THIS row's genre
```

Spot one: the inner query mentions the outer alias (`c1.genre`). Without that link it would return one global max instead of one per genre. It is the same idea as a self join: the table appears twice, once as the outer row and once as the lookup.

### CTE: named and written first

**What it does:** the same as a subquery in `FROM`, but written first with a name, so the main query reads top to bottom.

```sql
WITH genre_totals AS (                         -- name the intermediate result
  SELECT genre, SUM(revenue) AS total_revenue
  FROM concerts
  GROUP BY genre
)                                              -- the CTE ends at this closing bracket
SELECT g.genre, g.total_revenue, AVG(c.revenue) AS avg_revenue
FROM genre_totals AS g                         -- use it like a table
JOIN concerts AS c ON g.genre = c.genre
WHERE c.revenue > g.total_revenue * 0.5
GROUP BY g.genre, g.total_revenue;
```

- The CTE is everything inside `WITH name AS ( ... )`. The query after it is the main query.
- Two names are involved: the CTE's name, and an optional alias for it in the main query's `FROM` (`genre_totals AS g`). Where did `g` come from? Only from that `FROM` line.
- A table alias is optional when the tables are different. It is required in a self join. Once you alias a table, use the alias everywhere.
- Even without an alias, a column that exists in both tables needs a prefix, or SQL reports "ambiguous".
- Several CTEs: `WITH a AS (...), b AS (...) SELECT ...`.

### Same query both ways

```sql
-- Subquery version
SELECT * FROM (
  SELECT genre, SUM(revenue) AS total_revenue FROM concerts GROUP BY genre
) g;

-- CTE version
WITH g AS (
  SELECT genre, SUM(revenue) AS total_revenue FROM concerts GROUP BY genre
)
SELECT * FROM g;
```

Both give identical results.

### Which to use

| Situation | Use |
|---|---|
| Compare to one value or a list in `WHERE` | subquery |
| Add a computed column in `SELECT` | subquery |
| "For each row, find..." | correlated subquery |
| Long query in several steps | CTE, since you can read it top to bottom |
| Same intermediate result used more than once | CTE, since you write it once and name it |
| Walk a hierarchy of unknown depth | recursive CTE (below) |

Rule of thumb: if the inner query is short and used once, a subquery is fine. If you are building in steps or would repeat yourself, use a CTE.

### Recursive CTE (later topic)

A CTE that refers to itself so it can repeat. Only needed for hierarchies such as "everyone under this manager, at any depth". It has two parts joined by `UNION ALL`: an anchor query (the starting rows) and a recursive step that joins to the CTE itself, repeating until no new rows appear. PostgreSQL needs `WITH RECURSIVE`. Nothing above needs it, so come back to it last.

### Quick version

```
CTE / Subquery = a query whose result is used by another query.

Subquery: in parentheses, inline.
  WHERE col > (SELECT one_value ...)     -> compare to a single value
  WHERE col IN (SELECT list ...)         -> compare to a list
  SELECT ..., (SELECT value ...) AS x    -> extra column
  FROM (SELECT ...) alias                -> temp table (alias required)

Correlated subquery: inner query uses the outer alias; runs per row.
CTE: WITH name AS (SELECT ...) SELECT ... FROM name;   ends at the ")".
Use a CTE for multi-step or reused logic, a subquery for quick one-offs.
```

Practice idea: write "best concert per genre" as a CTE instead of a correlated subquery. It uses the window function from the next section.

---

## 11. Window function preview

**What it does:** a window function works like `GROUP BY` that does not collapse the rows. `PARTITION BY` splits the rows into buckets, and every row stays visible, so you can rank rows inside each bucket or add a group figure next to each row.

```sql
-- one row per id: the row with the highest salary
SELECT id, first_name, last_name, department_id, salary AS current_salary
FROM (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY id ORDER BY salary DESC) AS rn   -- number rows 1, 2, 3 per id
  FROM employee_salaries
) AS ranked
WHERE rn = 1                                                              -- keep each id's top row
ORDER BY id;
```

- `PARTITION BY id`: a separate bucket per id.
- `ORDER BY salary DESC`: sort each bucket, highest first.
- `ROW_NUMBER()`: number the rows 1, 2, 3... in each bucket, so the max-salary row is `1`.
- It is wrapped in a subquery because `WHERE` cannot see `rn` at the same level (execution order).
- `ROW_NUMBER` always gives exactly one row `1`, even on ties. That fits "if tied, return any one".
- PostgreSQL shortcut: `SELECT DISTINCT ON (id) ... ORDER BY id, salary DESC`.
- Related, to be added: `RANK()`, `DENSE_RANK()`, `LAG()`, `LEAD()`.
- Same top-row-per-group shape as section 0: self join, subquery in `WHERE`, or `ROW_NUMBER`.

---

## 12. Interview practice tips

- Practice on sites like DataLemur and StrataScratch for analyst roles. Mode's SQL tutorial is good for the concepts behind the practice questions.
- For mid-level analyst roles, window functions come up a lot: `LAG` / `LEAD`, `ROW_NUMBER`, `DENSE_RANK`. Be comfortable with them.
- Also practise from scratch on a blank dataset, not only on predefined problems. Some interviews hand you raw data and ask you to explore it yourself.
- When asked how you use SQL against a data warehouse, answer with real things you did: tables you queried, joins you built, aggregations you ran, how you checked the results.
