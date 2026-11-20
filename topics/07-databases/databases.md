# Databases and persistence

> ℹ️ **What you need first**
>
> The package structure of `ledger`, with its four modules (see [Lab: from one file to a package](../06-project-structure/lab.md)), and the ideas of [Software design and project structure](../06-project-structure/design.md), especially why storage is a separate concern. Why money is a `Decimal` and never a `float` (see [Money is not a float](../04-dependencies-tools/lab.md#3-money-is-not-a-float)).

## Why a database

`ledger` reads its transactions from `data/transactions.csv`. To record a new expense, you open the file in an editor and type a line. That works for one person and a few dozen lines. Now picture `ledger` as the backend of a web application:

- **Adding a transaction** means rewriting a file while other users may be reading it. Two requests that write at the same time can overwrite each other's line.
- **Nothing checks the data.** A line with the account `chekcing` or the amount `12,50` is accepted, and the balance is silently wrong.
- **Every question reads everything.** To get the balance of one account, the program reads all the lines of all the accounts. With millions of transactions, that is slow.
- **A crash in the middle of a write** can leave half a line in the file.

A **database management system** (DBMS) is software that stores data and solves exactly these problems. It accepts many readers and writers at once, checks every row against rules you declare, answers questions without reading everything, and guarantees that a change is either completely saved or not saved at all. Data that survives after the program ends is called **persistent**, and a database is how most applications persist their data.

This page covers the **relational** model, by far the most common in business software; the language to talk to relational databases, **SQL**; how to store money in them; and how a Python program uses a database through an **ORM**.

## Tables, rows and keys

A relational database stores data in **tables**. A table is like a sheet of a spreadsheet with strict rules: every row has the same columns, and every column has a type.

> 📘 **Definition: table, row, column, primary key**
>
> A **table** stores one kind of entity, such as transactions. Each **row** is one entity (one transaction), and each **column** is one attribute (the date, the amount), with a declared **type** (integer, text, date…).
>
> The **primary key** is a column whose value identifies each row: no two rows can have the same one, and it can't be empty. It is usually a number, `id`, that the database assigns automatically to each new row.

Why a separate `id`, when the description and the date could identify a transaction? Because two transactions can be identical in every field (two coffees at the same café on the same day). The `id` is the only thing that is guaranteed to be unique, and it never changes even if the transaction is corrected.

## Relationships and foreign keys

In the CSV file, the account is a text field on every line: `checking` appears 50 times. If the bank renames the account, 50 lines change. A typo, `chekcing`, creates a third account that no one notices. The relational solution is to store each account **once**, in its own table, and have each transaction refer to it by its key:

```text
accounts                         transactions
┌────┬──────────┐                ┌────┬────────────┬────────────┬────────────────────────┬───────────┬────────┐
│ id │ name     │                │ id │ date       │ account_id │ description            │ category  │ amount │
├────┼──────────┤                ├────┼────────────┼────────────┼────────────────────────┼───────────┼────────┤
│  1 │ checking │◀───────────────│  1 │ 2026-01-01 │          1 │ Rent                   │ housing   │ -95000 │
│  2 │ savings  │◀──────┐        │  2 │ 2026-01-02 │          1 │ Carrefour              │ groceries │  -6515 │
└────┴──────────┘       └────────│ 18 │ 2026-01-28 │          2 │ Transfer from checking │ transfer  │  30000 │
                                 └────┴────────────┴────────────┴────────────────────────┴───────────┴────────┘
```

> 📘 **Definition: foreign key and one-to-many relationship**
>
> A **foreign key** is a column whose values must be primary keys of another table: `transactions.account_id` must be the `id` of an existing row of `accounts`. The database rejects any row that breaks this rule.
>
> This gives a **one-to-many relationship**: one account has many transactions, and each transaction belongs to exactly one account. The foreign key is always on the "many" side.

The foreign key turns a convention into a rule that the database enforces: a transaction for an account that doesn't exist can't be saved. A rename touches one row of `accounts`. This is the main idea of relational design: **each fact is stored once**, and everything else refers to it.

(The amounts in the picture are in cents: `-95000` is −950.00 €. The section [Money in a database](#money-in-a-database) explains why.)

## SQL

**SQL** (Structured Query Language) is the language to define tables and to read and change their data, in every relational database.

Python comes with **SQLite**, a small relational database, in its standard library, so you can try SQL without installing anything. `sqlite3.connect(":memory:")` creates a database that lives only in memory while the program runs. First, the tables:

```python
import sqlite3

con = sqlite3.connect(":memory:")
con.execute("PRAGMA foreign_keys = ON")
```

```text
<sqlite3.Cursor object at 0x7adfd875edc0>
```

```python
con.execute("""
    CREATE TABLE accounts (
        id   INTEGER PRIMARY KEY,
        name TEXT NOT NULL UNIQUE
    )
""")
```

```text
<sqlite3.Cursor object at 0x7adfd875efc0>
```

```python
con.execute("""
    CREATE TABLE transactions (
        id          INTEGER PRIMARY KEY,
        date        DATE NOT NULL,
        account_id  INTEGER NOT NULL REFERENCES accounts (id),
        description TEXT NOT NULL,
        category    TEXT NOT NULL,
        amount      INTEGER NOT NULL
    )
""")
```

```text
<sqlite3.Cursor object at 0x7adfd875edc0>
```

```python
print("tables created")
```

```text
tables created
```

`NOT NULL` means the column can't be empty, `UNIQUE` that no two rows can share the value, and `REFERENCES accounts (id)` declares the foreign key. (SQLite checks foreign keys only after `PRAGMA foreign_keys = ON`; most other databases always check them.)

`INSERT` adds rows. The values are passed separately from the SQL, as **parameters** marked with `?`:

```python
con.executemany("INSERT INTO accounts (name) VALUES (?)", [("checking",), ("savings",)])
```

```text
<sqlite3.Cursor object at 0x7adfd875efc0>
```

```python
con.executemany(
    "INSERT INTO transactions (date, account_id, description, category, amount)"
    " VALUES (?, ?, ?, ?, ?)",
    [
        ("2026-01-01", 1, "Rent", "housing", -95000),
        ("2026-01-02", 1, "Carrefour", "groceries", -6515),
        ("2026-01-03", 1, "Spotify", "subscriptions", -1199),
        ("2026-01-27", 1, "Salary ACME Corp", "income", 285000),
        ("2026-01-28", 1, "Transfer to savings", "transfer", -30000),
        ("2026-01-28", 2, "Transfer from checking", "transfer", 30000),
    ],
)
```

```text
<sqlite3.Cursor object at 0x7adfd875edc0>
```

```python
print(con.execute("SELECT * FROM accounts").fetchall())
```

```text
[(1, 'checking'), (2, 'savings')]
```

`SELECT` reads. `WHERE` filters rows, `ORDER BY` sorts them:

```python
query = """
    SELECT date, description, amount
    FROM transactions
    WHERE amount < 0 AND category != 'transfer'
    ORDER BY amount
"""
for row in con.execute(query):
    print(row)
```

```text
('2026-01-01', 'Rent', -95000)
('2026-01-02', 'Carrefour', -6515)
('2026-01-03', 'Spotify', -1199)
```

To combine the two tables, `JOIN` matches each transaction with its account through the foreign key. `GROUP BY` puts the rows with the same value together, and `SUM` adds up each group. This is the balance of each account, computed by the database:

```python
query = """
    SELECT accounts.name, SUM(transactions.amount)
    FROM transactions
    JOIN accounts ON transactions.account_id = accounts.id
    GROUP BY accounts.name
    ORDER BY accounts.name
"""
for name, cents in con.execute(query):
    print(f"{name:<10} {cents / 100:>10.2f}")
```

```text
checking      1522.86
savings        300.00
```

And the foreign key at work: a transaction for account 3, which doesn't exist, is rejected:

```python
try:
    con.execute(
        "INSERT INTO transactions (date, account_id, description, category, amount)"
        " VALUES ('2026-01-30', 3, 'Coffee', 'restaurants', -250)"
    )
except sqlite3.IntegrityError as error:
    print("IntegrityError:", error)
```

```text
IntegrityError: FOREIGN KEY constraint failed
```

### SQL is declarative, Python is imperative

Compare the balance query above with the way `ledger` computes the same balances in Python, in `totals_by`:

```python
totals = defaultdict(Decimal)
for t in transactions:
    totals[t.account] += t.amount
```

The Python code says **how** to get the result, step by step: create an empty dictionary, go through the transactions one by one, add each amount to its account. The SQL query says only **what** the result is: the sum of the amounts, grouped by account name. It doesn't say in which order to read the rows, whether to use an index, or how to combine the two tables; the database decides all that.

> 📘 **Definition: imperative and declarative**
>
> An **imperative** language describes *how* to compute a result, as a sequence of steps that change the program's state (loops, assignments). Python, Java and JavaScript are mainly imperative.
>
> A **declarative** language describes *what* result you want, and leaves the steps to the system that runs it. SQL is declarative, and so are HTML and the `pyproject.toml` file.

Declarative code is usually shorter and easier to read, and it lets the database choose a fast way to answer: for the same query, it reads only the rows it needs from a table of millions, using indexes, where a Python loop would go through all of them. The price is control: when a query is slow, you can't change the steps directly, you ask the database how it plans to run the query (`EXPLAIN`) and change the query or the indexes. In practice, applications use both: SQL to select, join and aggregate the data in the database, and Python for the logic around it.

### Never build SQL with f-strings

It is tempting to write `f"SELECT * FROM accounts WHERE name = '{name}'"`. If `name` comes from a user, they can type `x' OR '1'='1`, and the query becomes `... WHERE name = 'x' OR '1'='1'`, which returns every account. With a `DELETE`, it would delete every row. This attack is called **SQL injection**, and it is still one of the most common security holes in web applications. With parameters (`?`), the database receives the value separately from the SQL, and treats it as data, never as part of the query:

```python
name = "x' OR '1'='1"
print(con.execute("SELECT * FROM accounts WHERE name = ?", (name,)).fetchall())
```

```text
[]
```

No account is called `x' OR '1'='1`, so the result is empty, as it should be.

## Money in a database

Databases have the same problem with money as Python. A column of type `REAL` or `FLOAT` stores binary floating-point numbers, and `0.1 + 0.2` is not `0.3` there either. There are two correct ways to store amounts:

1. A **decimal column**, `NUMERIC(12, 2)`: up to 12 digits, 2 of them after the decimal point, stored exactly. PostgreSQL, MySQL, SQL Server and Oracle have it.
2. An **integer number of cents**: `-950.00` is stored as `-95000`. Sums of integers are always exact, in every database. Payment systems such as Stripe store amounts this way.

SQLite has no exact decimal type: in a `NUMERIC` column, a value with decimals is stored as a float. `ledger` therefore uses the second option: amounts are **integers of cents in the database** and `Decimal` **in Python**, and a small piece of code converts between the two when rows are written and read. That piece is the `Money` type of [the lab](lab.md#2-the-database-module-dbpy). It also works unchanged on PostgreSQL, so changing the database later doesn't change how money is stored.

## SQLite and database servers

Most databases are **servers**: a program that runs all the time, often on another machine, and that applications connect to over the network with a user and a password. PostgreSQL, MySQL and SQL Server work like this. They handle many applications and users at once, and they are what production systems use.

**SQLite** is different: it is a library inside your program, and the whole database is **one file** (`ledger.db`). There is nothing to install, start or configure, and copying the file copies the database. It is the most widely deployed database in the world (every phone and every browser has several SQLite databases), and it is ideal for developing, for tests and for applications with one user. It is not designed for many machines writing at the same time.

| | SQLite | PostgreSQL |
|---|---|---|
| What it is | a library; the database is a file | a server you connect to |
| Setup | none | install or run it, create users |
| Many writers at once | limited | yes |
| Exact decimals | no | `NUMERIC` |
| Typical use | development, tests, one-user apps, phones | web applications, production |

We develop `ledger` on SQLite. Because the program talks to the database through SQLAlchemy (next section), moving it to PostgreSQL means changing one line, the **database URL**: `sqlite:///ledger.db` becomes something like `postgresql://user:password@host/ledger`.

## From SQL to Python: the ORM

You could use SQL strings everywhere in `ledger`, as in the examples above. Real applications usually don't, for three reasons. The rest of the program works with Python objects (a `Transaction` with `.amount` and `.account`), so every query needs code to turn rows into objects and back. SQL strings are not checked until they run: a misspelled column is found by the user. And each database has a slightly different SQL dialect.

> 📘 **Definition: ORM**
>
> An **ORM** (*object-relational mapper*) is a library that maps **classes to tables**, **objects to rows** and **attributes to columns**. You work with Python objects, and the ORM writes the SQL to load and save them.

**SQLAlchemy** is the standard ORM in Python. You declare each table as a class, with the columns as typed attributes:

```python
class Account(Base):
    __tablename__ = "accounts"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)

    rows: Mapped[list[TransactionRow]] = relationship(back_populates="account")


class TransactionRow(Base):
    __tablename__ = "transactions"

    id: Mapped[int] = mapped_column(primary_key=True)
    date: Mapped[datetime.date]
    account_id: Mapped[int] = mapped_column(ForeignKey("accounts.id"))
    description: Mapped[str] = mapped_column(String(200))
    category: Mapped[str] = mapped_column(String(50))
    amount: Mapped[Decimal] = mapped_column(Money)

    account: Mapped[Account] = relationship(back_populates="rows")
```

- `Mapped[int]` is the Python type of the attribute; SQLAlchemy chooses the column type from it (`INTEGER`, `VARCHAR`, `DATE`…), or takes the one you give in `mapped_column`.
- `ForeignKey("accounts.id")` declares the foreign key.
- `relationship` is not a column. It gives you the related objects as attributes: `row.account` is the `Account` of a row, and `account.rows` the list of its rows. SQLAlchemy loads them with a query when you use them.

In `ledger`, the table class is called `TransactionRow`, not `Transaction`, on purpose. The `Transaction` dataclass of `models.py` is what the rest of the program works with; `TransactionRow` is how a transaction is stored. `storage.py` converts one into the other, so the calculations in `reports.py` don't depend on SQLAlchemy and don't change when the storage does. That is the [direction of dependencies](../06-project-structure/design.md#the-direction-of-dependencies) at work.

Two more objects connect the classes to a real database:

- The **engine** knows where the database is (its URL) and how to talk to it. It is created once: `create_engine("sqlite:///ledger.db")`.
- A **session** is one conversation with the database. You `add` new objects to it, read with `select`, and call `commit` to save all the changes together.

```python
with Session(engine) as session:
    checking = Account(name="checking")
    session.add(TransactionRow(date=..., account=checking, amount=Decimal("-950.00"), ...))
    session.commit()                         # INSERT INTO accounts ...; INSERT INTO transactions ...

    query = select(TransactionRow).where(TransactionRow.category == "housing")
    for row in session.scalars(query):       # SELECT ... FROM transactions WHERE category = ?
        print(row.date, row.account.name, row.amount)
```

SQLAlchemy uses parameters in every query it writes, so it is protected against SQL injection. You can still write SQL when you need it, and an ORM doesn't replace knowing SQL: when a query is slow or returns something unexpected, you read the SQL it generated (`create_engine(..., echo=True)` prints it).

### Database transactions

The word *transaction* has a second meaning here, unrelated to money. A **database transaction** is a group of changes that the database applies **all together or not at all**. In SQLAlchemy, everything between two `commit` calls of a session is one database transaction. If the program crashes, or calls `session.rollback()`, before the commit, none of the changes is saved.

This is what makes a transfer safe: taking 300 € out of `checking` and putting it into `savings` are two rows, and if the program fails after the first, a database transaction guarantees that the first is undone too. Money doesn't disappear halfway.

### When the tables change

`Base.metadata.create_all(engine)` creates the tables that don't exist yet. It **doesn't change tables that already exist**: if you add a column to a class, the table in `ledger.db` keeps its old columns, and SQLAlchemy fails with `no such column` when it tries to use the new one. While developing with test data, the simple fix is to delete `ledger.db` and import the data again. With real data, you can't delete the database: teams use **migrations**, scripts that change the schema step by step, kept in the repository (with SQLAlchemy, the usual tool is **Alembic**).

## Designing the tables of your module

The data model you wrote for your module, its entities, fields and relationships (see [Designing your own module](../06-project-structure/design.md#designing-your-own-module)), becomes tables in four steps:

1. **One table per entity**, with an `id` primary key.
2. **One column per field**, with its type. Money is integer cents (or `NUMERIC` on a database that has it), never `REAL`.
3. **A foreign key on the "many" side** of each relationship: a payment has `invoice_id`, an invoice has `client_id`.
4. **Each fact once.** If a value is repeated on many rows and could change (a client's name on each invoice), it is an entity of its own.

## Common mistakes

- **Building SQL with f-strings.** It opens the door to SQL injection. Use parameters, or the ORM.
- **Storing money as `REAL` or `FLOAT`.** The binary error comes back in every `SUM`.
- **Forgetting `commit`.** The changes are visible inside the session, so the program seems to work, but they are lost when it ends.
- **Changing a model and keeping the old database.** `create_all` doesn't alter existing tables: `no such column`. Delete the development database, or write a migration.
- **Committing the database file.** `ledger.db` is data generated by the program, like `.venv`: it goes in `.gitignore`. A database with real users' data in a public repository is a data leak.
- **Confusing the two kinds of transaction.** A `Transaction` row is a movement of money; a database transaction is a group of changes saved together.

> ✅ **Key takeaways**
>
> - A relational database stores data in **tables** with typed columns. Each row has a **primary key**, and a **foreign key** links a row to a row of another table, which gives a one-to-many relationship. Each fact is stored once.
> - **SQL** defines tables (`CREATE TABLE`) and reads and changes their data (`SELECT ... WHERE ... GROUP BY`, `JOIN`, `INSERT`). It is **declarative**: you say what result you want, and the database decides how to get it; Python is **imperative**: you write the steps. Values always go as **parameters**, never pasted into the SQL string, to prevent SQL injection.
> - Money in a database is a `NUMERIC` column or an **integer of cents**, never a float. `ledger` stores cents and uses `Decimal` in Python.
> - **SQLite** is a database in a file, ideal for development; servers such as PostgreSQL run production systems. With an ORM, changing from one to the other means changing the database URL.
> - An **ORM** such as SQLAlchemy maps classes to tables and objects to rows. The **engine** knows the database, a **session** groups the changes, and `commit` saves them together as one **database transaction**.

## Exercises

**1.** A table `invoices` has the columns `id`, `client_name`, `client_email`, `date` and `total`. A client changes their email. What problem does this design have, and how would you change it?

<details>
<summary>✅ <b>Solution</b></summary>

The client's name and email are repeated on every invoice of that client, so the new email must be updated on all of them, and if one is missed the data contradicts itself. Each fact should be stored once: a table `clients` (`id`, `name`, `email`), and in `invoices` a foreign key `client_id` instead of the two text columns. The change of email is then one update of one row. `total` should be integer cents or `NUMERIC`.

</details>

**2.** Write the SQL query that gives the total spending of each category in January 2026, largest first, from the table `transactions` of this page (amounts in cents, transfers excluded). Dates are text like `'2026-01-15'`.

<details>
<summary>✅ <b>Solution</b></summary>

```sql
SELECT category, -SUM(amount) AS spending
FROM transactions
WHERE amount < 0
  AND category != 'transfer'
  AND date >= '2026-01-01' AND date < '2026-02-01'
GROUP BY category
ORDER BY spending DESC
```

Dates as `YYYY-MM-DD` text sort in the same order as the dates, so the comparisons work. The minus sign turns spending into positive numbers, and `ORDER BY spending DESC` puts the largest first. The result is in cents: divide by 100 to show euros.

</details>

**3.** A colleague writes this function for a search box. What can go wrong, and how do you fix it?

```python
def find(con, text):
    return con.execute(f"SELECT * FROM transactions WHERE description = '{text}'").fetchall()
```

<details>
<summary>✅ <b>Solution</b></summary>

The text is pasted into the SQL, so a user can change the query. With `text = "x' OR '1'='1"`, the query returns every transaction; other inputs can do worse. Even without bad intentions, a description with an apostrophe, `McDonald's`, breaks the query with a syntax error. The fix is a parameter:

```python
def find(con, text):
    return con.execute(
        "SELECT * FROM transactions WHERE description = ?", (text,)
    ).fetchall()
```

</details>

**4.** In `ledger`, `import_csv` adds 56 transactions with `session.add` and calls `session.commit()` once at the end. The program crashes at the 30th row because the CSV has an amount `12,50`. How many transactions are in the database afterwards? And if `commit` were called after each `add`?

<details>
<summary>✅ <b>Solution</b></summary>

With one `commit` at the end: **none**. The 29 added transactions were part of a database transaction that was never committed, so the database discards them. This is usually what you want: the import either happens completely or not at all, and you can fix the file and run it again without duplicates. With a `commit` after each row, the first 29 would be saved, and running the import again after fixing the file would add them a second time.

</details>

[← Back to the session](index.md) [Next: Lab →](lab.md)
