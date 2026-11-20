# Databases and persistence

`ledger` reads a CSV file every time it runs, and the only way to record a new expense is to edit that file by hand. Real applications keep their data in a **database**: it accepts new data from the program, checks it against rules, answers questions quickly, and keeps everything safe when something fails halfway. In this session you learn how a relational database organizes data in **tables** linked by **keys**, how to query it with **SQL**, how to store money in it correctly, and how a Python program uses it through an **ORM**, SQLAlchemy. Then you move `ledger` into a SQLite database and add a command to record transactions.

## The thread of the session

1. **[Databases and persistence](databases.md).** Tables, primary and foreign keys, SQL (`SELECT`, `JOIN`, `GROUP BY`, `INSERT`), declarative SQL versus imperative Python, SQL injection, money in a database, SQLite and database servers, the ORM, and database transactions.
2. **[Lab: the ledger in a database](lab.md).** You add a database module, link transactions to accounts with a foreign key, read the rows back as `Transaction`s, import the CSV file into SQLite, check that the reports don't change, query the database with SQL, and add the business rules for recording new transactions. At the end you tag your repository as **checkpoint 2** and submit it on Blackboard.

## Objectives

By the end of the session you should be able to:

- [ ] Explain what problems a database solves compared with a file, and what persistence means.
- [ ] Describe a table with its columns, types and primary key, and link two tables with a foreign key.
- [ ] Write SQL queries with `SELECT`, `WHERE`, `JOIN`, `GROUP BY` and `ORDER BY`, and add rows with `INSERT`.
- [ ] Explain the difference between a declarative language such as SQL and an imperative one such as Python.
- [ ] Pass values to SQL as parameters, and explain what SQL injection is.
- [ ] Store amounts of money in a database without floating-point errors.
- [ ] Explain the difference between SQLite and a database server such as PostgreSQL, and when to use each.
- [ ] Declare tables as SQLAlchemy classes, and read and write them with a session, `select`, `add` and `commit`.
- [ ] Explain what a database transaction is, and why `create_all` doesn't change existing tables.

## How to study this session

| Page | What you do |
|---|---|
| [Databases and persistence](databases.md) | read, with the exercises |
| [Lab: the ledger in a database](lab.md) | hands-on |

## Before you start

You need your repository `smd-app` with `ledger` as a package of four modules merged into `main` (see [Lab: from one file to a package](../06-project-structure/lab.md)). If it isn't finished, the lab explains how to start from the reference version.

## Class slides

📄 [Slides (PDF)](slides.pdf)
