+++
title = "SQL Basics (Easy) - Walkthrough"
author = "nvru"
date = "2026-07-25"
description = "Walkthrough for the Hack Smarter Challenge Lab: SQL Basics (Easy)."
tags = [
  "hack-smarter",
  "lab",
  "sql",
  "sql-basics",
  "sqlite",
  "database",
  "walkthrough"
]
draft = false
+++

# Hack Smarter - SQL Fundamentals Lab Walkthrough

In this lab, the objective was to complete four SQL missions by interacting directly with a live SQLite database. Rather than exploiting a vulnerable application, this exercise focused on understanding how SQL works from the database administrator's perspective.

Before completing the missions, I first explored the database to understand its structure.

---

## Step 1 - Identify the Database Type

The first thing I wanted to know was which database engine the application was using.

```sql
SELECT sqlite_version();
```

The `sqlite_version()` function returns the version of the SQLite engine currently in use. Identifying the database type is always a good first step because different database systems have different syntax, features, and built-in functions.

The query confirmed that the backend database was **SQLite**.

![SQLite Version](/images/hsm/sql_basic/version.png)

---

## Step 2 - Enumerate the Database Tables

Next, I listed all available tables.

```sql
SELECT name
FROM sqlite_master
WHERE type = 'table';
```

SQLite stores metadata about every database object inside a special table called `sqlite_master`. Filtering on `type = 'table'` returns every user-created table in the database.

![Database Tables](/images/hsm/sql_basic/tables.png)

The database contained three tables.

One of them was `sqlite_sequence`, which is an internal SQLite table used to keep track of the current value of columns that use `AUTOINCREMENT`.

![sqlite_sequence](/images/hsm/sql_basic/sql_seq.png)

After identifying the tables, I was ready to begin the lab missions.

![Mission Overview](/images/hsm/sql_basic/missions.png)

---

## Mission 1 - Find the Product with Stock 1337

To inspect the products table, I displayed every record.

```sql
SELECT *
FROM products;
```

`SELECT *` returns every column from every row in the table, making it useful for quickly inspecting the available data.

From the results, I located the product whose stock value was **1337**, completing the first challenge.

![Mission 1](/images/hsm/sql_basic/mis1.png)

✅ **Mission Complete**

---

## Mission 2 - Move "Alice Smith" to "Cybersecurity"

The second task required updating an employee's department.

```sql
UPDATE employees
SET department = 'Cybersecurity'
WHERE id = 1;
```

The `UPDATE` statement modifies existing records.

- `SET` specifies the new value.
- `WHERE id = 1` ensures only Alice Smith's record is updated.

Without the `WHERE` clause, every employee would have been assigned to the Cybersecurity department.

![Mission 2](/images/hsm/sql_basic/mis2.png)

✅ **Mission Complete**

---

## Mission 3 - Add a New Employee

The third mission required inserting a new employee into the database.

```sql
INSERT INTO employees (name, department, salary)
VALUES ('Hacker', 'Cybersecurity', '9999999');
```

`INSERT INTO` creates a new row in the specified table.

The values provided populate the `name`, `department`, and `salary` columns for the new employee.

![Mission 3](/images/hsm/sql_basic/mis3.png)

✅ **Mission Complete**

---

## Mission 4 - Delete an Employee

The final challenge required removing an employee from the database.

```sql
DELETE FROM employees
WHERE id = 2;
```

The `DELETE` statement removes records from a table.

Using a `WHERE` clause ensures that only the employee with ID `2` is deleted. Without it, every row in the table would have been removed.

![Mission 4](/images/hsm/sql_basic/mis4.png)

✅ **Mission Complete**

---

## Final Flag

```text
HSM{SQL_ISNT_THAT_HARD}
```

---

## Conclusion

This lab was a great introduction to the fundamentals of SQL. Instead of immediately focusing on SQL Injection, it emphasized understanding how databases work by performing common operations such as:

- Identifying the database engine.
- Enumerating database tables.
- Retrieving data with `SELECT`.
- Updating records with `UPDATE`.
- Inserting new rows with `INSERT`.
- Deleting records with `DELETE`.

Building a strong foundation in SQL makes it much easier to understand how SQL Injection vulnerabilities occur and, more importantly, how to identify and prevent them during security assessments.
