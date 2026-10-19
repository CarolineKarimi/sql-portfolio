# Jupyter SQL Basics

My first SQL tutorial. It uses a Jupyter notebook as the SQL environment, connected to a database through SQLAlchemy.

After importing `create_engine` from `sqlalchemy`, the notebook builds a SQL database, loads data into it from the Excel files in this folder, and then manipulates that data with SQL commands.

## Files

| File | What it is |
|---|---|
| `sql tutorial.ipynb` | The tutorial notebook |
| `categories.xlsx`, `customer_data.xlsx`, `employees.xlsx`, `order_details.xlsx`, `orders.xlsx`, `products.xlsx`, `shippers.xlsx`, `suppliers.xlsx` | The tables loaded into the database |

## How to run it

```bash
pip install notebook pandas sqlalchemy openpyxl
jupyter notebook
```

Open `sql tutorial.ipynb` and run the cells from the top.

## Note

This was my starting point. My more recent work is in [`8-week-sql-challenge/`](../../8-week-sql-challenge/) and the [cheat sheet](../../cheat-sheet/sql_cheat_sheet.md).
