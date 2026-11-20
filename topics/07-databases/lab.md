# Lab: the ledger in a database

> ℹ️ **What you need first**
>
> Your repository `smd-app` with `ledger` as a package (see [Lab: from one file to a package](../06-project-structure/lab.md)) merged into `main`: the modules `models`, `storage`, `reports` and `cli`, and the commands `balance`, `spending` and `months`. The concepts of [Databases and persistence](databases.md): tables, primary and foreign keys, SQL, money in a database, and the ORM.

In this lab `ledger` stops reading a CSV file on every run and keeps its transactions in a **SQLite database**, through **SQLAlchemy**. The CSV file becomes something you import once, and a new command records transactions that are still there the next time the program runs. The tables and the connection go in a new module, `db.py`, and `storage.py` and `cli.py` grow new functions. `models.py` and `reports.py` don't change at all: the reports keep working on `Transaction`s, wherever they come from. The code is on this page, except for a few places marked `# TODO`: the link between transactions and accounts, reading the rows back, two business rules, and a SQL query.

At the end, you mark the commit you want graded with the tag `cp2`, and submit the link to that commit on Blackboard, in the assignment **Checkpoint 2**.

## 1. Start a branch and record the behaviour

```bash
cd ~/smd/smd-app
git switch main
git pull
git status
git switch -c feature/database
```

Record the output of the three reports. At the end you check that the database gives the same results as the CSV file:

```bash
mkdir -p ~/smd/before
uv run ledger balance > ~/smd/before/balance.txt
uv run ledger spending > ~/smd/before/spending.txt
uv run ledger months > ~/smd/before/months.txt
```

<details>
<summary>💡 <b>My ledger is not a package yet</b></summary>

Copy the files of the tag **`ledger-package`** of [github.com/ami232/ie-smd-ledger](https://github.com/ami232/ie-smd-ledger) into your repository (switch to the tag, **Code → Download ZIP**): `pyproject.toml`, `uv.lock` and the folder `src/`. Commit them on `main` with a message that says where they come from, push, and then start this step. The checkpoint also checks the package work (its pull request and the Ruff rules), so finish it if you can.

</details>

## 2. The database module: `db.py`

Add SQLAlchemy to the project:

```bash
uv add sqlalchemy
```

Create `src/ledger/db.py` with this code. It declares the two tables and the connection to the database:

```python
"""The database: its tables and the connection to it."""

import datetime
from decimal import Decimal

from sqlalchemy import ForeignKey, Integer, String, create_engine, event
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    Session,
    mapped_column,
    relationship,
)
from sqlalchemy.types import TypeDecorator

DATABASE_URL = "sqlite:///ledger.db"


class Money(TypeDecorator):
    """An amount: a Decimal in Python, a whole number of cents in the database."""

    impl = Integer
    cache_ok = True

    def process_bind_param(self, value, dialect):
        if value is None:
            return None
        cents = value * 100
        if cents != cents.to_integral_value():
            raise ValueError(f"{value} has fractions of a cent")
        return int(cents)

    def process_result_value(self, value, dialect):
        if value is None:
            return None
        return Decimal(value).scaleb(-2)


class Base(DeclarativeBase):
    pass


class Account(Base):
    __tablename__ = "accounts"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)

    rows: Mapped[list[TransactionRow]] = relationship(back_populates="account")


class TransactionRow(Base):
    """One row of the table `transactions`."""

    __tablename__ = "transactions"

    id: Mapped[int] = mapped_column(primary_key=True)
    date: Mapped[datetime.date]
    description: Mapped[str] = mapped_column(String(200))
    category: Mapped[str] = mapped_column(String(50))
    amount: Mapped[Decimal] = mapped_column(Money)

    # TODO 1: the account of each row. Add
    #         - a column account_id: an int, foreign key to accounts.id
    #           (ForeignKey("accounts.id") inside mapped_column), and
    #         - an attribute account: the Account, through a relationship whose
    #           back_populates is the name of the other side, Account.rows.


engine = create_engine(DATABASE_URL)


@event.listens_for(engine, "connect")
def enable_foreign_keys(dbapi_connection, connection_record):
    """SQLite checks foreign keys only when asked to, once per connection."""
    dbapi_connection.execute("PRAGMA foreign_keys = ON")


def create_tables() -> None:
    """Creates the tables that don't exist yet. Existing tables are not changed."""
    Base.metadata.create_all(engine)


def get_session() -> Session:
    return Session(engine)
```

Read it from the top:

- **`DATABASE_URL`** says which database to use: `sqlite:///ledger.db` is a SQLite file called `ledger.db` in the folder where you run the program. To use PostgreSQL instead, only this line would change.
- **`Money`** is a column type for amounts. It stores each amount as a whole number of **cents** (`-950.00` → `-95000`) and gives it back as a `Decimal`, so the rest of the program never sees the cents. SQLite has no exact decimal type, and money is never a float (see [Money in a database](databases.md#money-in-a-database)).
- **`Base`** is the class every table inherits from. SQLAlchemy collects the tables of its subclasses, which is how `create_tables` knows what to create.
- **`Account`** is the table `accounts`: each account is stored once, with a unique name. `account.rows` is the list of its transactions, through a `relationship`.
- **`TransactionRow`** is the table `transactions`, one row per transaction. It is not your `Transaction` dataclass: `Transaction` is what the program works with, `TransactionRow` is how it is stored, and `storage.py` converts one into the other.
- The **engine** is created once and shared by the whole program. `enable_foreign_keys` runs on each new connection, because SQLite doesn't check foreign keys unless asked.

## 3. Storage: the database functions

`storage.py` keeps `read_csv`, and gets the functions that read and write the database. Add these imports at the top of `src/ledger/storage.py`, next to the others:

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

from ledger.db import Account, TransactionRow
```

Change its docstring (the first line) to `"""Reading and writing transactions: the CSV file and the database."""`, and add these functions at the end of the file:

```python
def find_account(session: Session, name: str) -> Account | None:
    """The account with this name, or None if there is none."""
    return session.scalars(select(Account).where(Account.name == name)).first()


def get_or_create_account(session: Session, name: str) -> Account:
    account = find_account(session, name)
    if account is None:
        account = Account(name=name)
        session.add(account)
    return account


def import_csv(session: Session, path: Path) -> int:
    """Adds the transactions of a CSV file to the database. Returns how many."""
    transactions = read_csv(path)
    for t in transactions:
        session.add(
            TransactionRow(
                date=t.date,
                account=get_or_create_account(session, t.account),
                description=t.description,
                category=t.category,
                amount=t.amount,
            )
        )
    session.commit()
    return len(transactions)


def load_transactions(session: Session) -> list[Transaction]:
    """All the transactions in the database, oldest first."""
    query = select(TransactionRow).order_by(TransactionRow.date, TransactionRow.id)
    rows = session.scalars(query)
    # TODO 2: return a list with one Transaction (from models.py) per row. Each
    #         field is the attribute of the row with the same name, except the
    #         account: a Transaction has the account's name, row.account.name.
    raise NotImplementedError


def add_transaction(
    session: Session,
    account_name: str,
    amount: Decimal,
    category: str,
    description: str,
    day: date,
) -> None:
    """Records one transaction in an existing account."""
    account = find_account(session, account_name)
    # TODO 3: check two rules before saving anything, and raise a ValueError with
    #         a clear message when one fails:
    #         - the account exists: find_account returned an Account, not None;
    #         - the amount is in whole cents: it equals round(amount, 2).
    session.add(
        TransactionRow(
            date=day,
            account=account,
            description=description,
            category=category,
            amount=amount,
        )
    )
    session.commit()
```

- `select(Account).where(Account.name == name)` builds the query `SELECT ... FROM accounts WHERE name = ?`, with the name as a parameter. `session.scalars(...)` runs it and returns `Account` objects; `.first()` the first one, or `None`.
- `import_csv` reuses `read_csv`, and creates each account the first time it appears. It calls `commit` **once**, at the end: the import is one database transaction, so if a row fails, nothing is saved.
- `load_transactions` (TODO 2) and `add_transaction` (TODO 3) are yours, in the next steps.

## 4. The command line

The commands now need different arguments (`import` takes a file, `add` an account and an amount…), so `cli.py` uses **subcommands**, as `git` does. In `src/ledger/cli.py`, keep `DEFAULT_FILE`, `print_totals` and `print_spending`. Replace the imports at the top with:

```python
import argparse
import datetime
from decimal import Decimal, InvalidOperation
from pathlib import Path

from rich.console import Console
from rich.table import Table

from ledger import db, reports, storage
```

and replace the function `main` with these three functions:

```python
def parse_amount(text: str) -> Decimal:
    """Converts an amount typed by the user, such as -12.50, into a Decimal."""
    try:
        amount = Decimal(text)
    except InvalidOperation:
        raise argparse.ArgumentTypeError(f"not an amount: {text!r}") from None
    if not amount.is_finite():
        raise argparse.ArgumentTypeError(f"not an amount: {text!r}")
    return amount


def make_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        prog="ledger", description="Balances and spending of your accounts."
    )
    commands = parser.add_subparsers(dest="command", required=True)

    commands.add_parser("balance", help="balance of each account")
    commands.add_parser("spending", help="spending by category")
    commands.add_parser("months", help="net change of each month")

    import_ = commands.add_parser("import", help="add the transactions of a CSV file")
    import_.add_argument("file", nargs="?", type=Path, default=DEFAULT_FILE)

    add = commands.add_parser("add", help="record one transaction")
    add.add_argument("account")
    add.add_argument("amount", type=parse_amount, help="negative for money going out")
    add.add_argument("category")
    add.add_argument("description")
    add.add_argument(
        "--date", type=datetime.date.fromisoformat, default=datetime.date.today()
    )
    return parser


def main() -> None:
    args = make_parser().parse_args()
    db.create_tables()

    with db.get_session() as session:
        if args.command == "import":
            count = storage.import_csv(session, args.file)
            print(f"Imported {count} transactions from {args.file}")
        elif args.command == "add":
            try:
                storage.add_transaction(
                    session,
                    args.account,
                    args.amount,
                    args.category,
                    args.description,
                    args.date,
                )
            except ValueError as error:
                raise SystemExit(f"ledger: {error}") from None
            print(f"Added {args.amount} to {args.account}")
        else:
            transactions = storage.load_transactions(session)
            if args.command == "balance":
                print_totals(reports.balances(transactions))
            elif args.command == "spending":
                print_spending(reports.spending_by_category(transactions))
            else:
                print_totals(reports.net_by_month(transactions))
```

- `add_subparsers` creates one parser per command, each with its own arguments.
- `parse_amount` converts the amount from text. `Decimal("twelve")` raises `decimal.InvalidOperation`, which `argparse` doesn't recognize; `ArgumentTypeError` is the error it expects, and it turns it into a one-line usage message.
- `main` creates the tables if needed, opens **one session** for the whole command (`with` closes it at the end), and turns a `ValueError` from storage into a clean message and exit code 1, with `SystemExit`. `import_` has a trailing underscore because `import` is a Python keyword.

## 5. Link transactions to accounts

Try to import the data:

```bash
uv run ledger import
```

<details>
<summary>✅ <b>What you should see</b></summary>

A long traceback that ends in:

```text
sqlalchemy.exc.NoForeignKeysError: Could not determine join condition between parent/child tables on relationship Account.rows - there are no foreign keys linking these tables. ...
```

`Account` declares a relationship to its rows, but nothing in `transactions` says which account each row belongs to.

</details>

**TODO 1, in `db.py`:** in `TransactionRow`, add the foreign key column `account_id` and the relationship `account`, as the comment describes. `Account` shows the syntax of both.

<details>
<summary>💡 <b>Hints</b></summary>

- A column: `name: Mapped[type] = mapped_column(...)`. A foreign key goes inside `mapped_column`, as `ForeignKey("accounts.id")`; it is already imported.
- A relationship: `name: Mapped[OtherClass] = relationship(back_populates="...")`, where `back_populates` names the attribute on the other side. `Account` has `rows`, with `back_populates="account"`.
- Until you write the TODO, `uv run ruff check` reports `F401 ForeignKey imported but unused`. Don't run `ruff check --fix` before TODO 1: it would delete the import you need.

</details>

```bash
uv run ledger import
ls
```

<details>
<summary>✅ <b>What you should see</b></summary>

`Imported 56 transactions from data/transactions.csv`, and a new file, `ledger.db`, in `ls`: the whole database.

</details>

## 6. Read the transactions back

```bash
uv run ledger balance
```

<details>
<summary>✅ <b>What you should see</b></summary>

A traceback that ends in `NotImplementedError`, raised in `load_transactions`: the reports ask storage for the transactions, and that is your next TODO.

</details>

**TODO 2, in `storage.py`:** complete `load_transactions`: return a list with one `Transaction` per row, as the comment describes, and delete the `raise NotImplementedError` line.

<details>
<summary>💡 <b>Hints</b></summary>

- `read_csv` builds `Transaction`s from the rows of a CSV file. Here you build them from `TransactionRow`s, which already have the right types: `row.date` is a `date` and `row.amount` a `Decimal`.
- A list comprehension, `[... for row in rows]`, builds the list in one expression.

</details>

Then compare the three reports with the ones you recorded from the CSV version:

```bash
uv run ledger balance | diff ~/smd/before/balance.txt -
uv run ledger spending | diff ~/smd/before/spending.txt -
uv run ledger months | diff ~/smd/before/months.txt -
```

`|` sends the output of `ledger` to `diff`, and `-` tells `diff` to read it from there instead of a file.

<details>
<summary>✅ <b>What you should see</b></summary>

The three `diff` commands print nothing: the database gives exactly the same results as the CSV file, to the cent. And `reports.py` didn't change at all: storage moved from a file to a database without the calculations knowing.

</details>

Now run the import **a second time**, look at the balances, and start again from a clean database:

```bash
uv run ledger import
uv run ledger balance
rm ledger.db
uv run ledger import
```

<details>
<summary>✅ <b>What you should see</b></summary>

After the second import, `checking 6698.20` and `savings 1812.00`: exactly double. The database kept the first 56 transactions and added the same 56 again. A database is **state**: it remembers what every run did. Deleting the file and importing once gives the right balances again.

</details>

The database is a file that the program generates, like `.venv`, and it must not be committed:

```bash
echo "*.db" >> .gitignore
git status
```

<details>
<summary>✅ <b>What you should see</b></summary>

`ledger.db` is not in the list of untracked files, and `.gitignore` appears as modified.

</details>

## 7. Look inside the database

The database is a file in a standard format. Python includes a SQL prompt for it, `python -m sqlite3`. Open it, and type each query at the `sqlite>` prompt, ending it with `;`:

```bash
uv run python -m sqlite3 ledger.db
```

```sql
SELECT * FROM accounts;
SELECT * FROM transactions WHERE category = 'health';
```

<details>
<summary>✅ <b>What you should see</b></summary>

```text
(1, 'checking')
(2, 'savings')
```

```text
(6, '2026-01-10', 1, 'Gym membership', 'health', -3990)
(25, '2026-02-10', 1, 'Gym membership', 'health', -3990)
(44, '2026-03-10', 1, 'Gym membership', 'health', -3990)
```

The amounts are in cents (`-3990` is −39.90 €), and the account of each transaction is a number: `1`, the `id` of `checking`.

</details>

**Your query:** write the SQL query that gives the balance of each account by **name**, in cents. Then leave with `.quit`.

<details>
<summary>💡 <b>Hints</b></summary>

The name is in `accounts` and the amounts are in `transactions`: you need a `JOIN` on the foreign key, a `SUM` and a `GROUP BY` (see [SQL](databases.md#sql)).

</details>

<details>
<summary>✅ <b>What you should see</b></summary>

`('checking', 334910)` and `('savings', 90600)`: the same balances, in cents.

</details>

## 8. Record new transactions

This is what the CSV version couldn't do:

```bash
uv run ledger add checking -12.50 restaurants "Lunch with a client" --date 2026-03-31
uv run ledger add savings 15.00 interest "Bonus interest" --date 2026-03-31
uv run ledger balance
```

<details>
<summary>✅ <b>What you should see</b></summary>

`Added -12.50 to checking` and `Added 15.00 to savings`, and then the balances `3336.60` and `921.00`. Run `uv run ledger balance` again, or open a new terminal: the transactions are still there. They are **persistent**: they live in `ledger.db`, not in the program's memory.

</details>

Now try two transactions that break the rules:

```bash
uv run ledger add credit -5 restaurants "Coffee"
uv run ledger add checking -1.005 restaurants "Coffee"
```

<details>
<summary>✅ <b>What you should see</b></summary>

Two tracebacks. The first ends in `IntegrityError: NOT NULL constraint failed: transactions.account_id`: there is no account called `credit`, so the transaction has no account, and the database refuses it. The second ends in `ValueError: -1.005 has fractions of a cent`, raised by `Money`. The data is safe, but the user gets a wall of text.

</details>

**TODO 3, in `storage.py`:** in `add_transaction`, check the two rules before saving anything, and raise a `ValueError` with a clear message when one fails: the account must exist, and the amount must be in whole cents. `cli.py` already turns that `ValueError` into a one-line message.

<details>
<summary>💡 <b>Hints</b></summary>

- `find_account` returns `None` when there is no account with that name.
- `round(amount, 2)` rounds to cents, so an amount in whole cents is equal to its rounded value.
- `raise ValueError(f"there is no account called {account_name!r}")`: `!r` puts the name between quotes.

</details>

```bash
uv run ledger add credit -5 restaurants "Coffee"
echo $?
uv run ledger add checking -1.005 restaurants "Coffee"
uv run ledger add checking twelve restaurants "Coffee"
uv run ledger balance
```

<details>
<summary>✅ <b>What you should see</b></summary>

One line per error, such as `ledger: there is no account called 'credit'` (and `echo $?` prints `1`) and `ledger: -1.005 has more than two decimals`. `twelve` is refused by `parse_amount` in `cli.py`: `argument amount: not an amount: 'twelve'`. None of them changed the database: the balances are still `3336.60` and `921.00`.

</details>

## 9. Commit, merge and tag

Check that no `TODO` is left and that Ruff passes, then commit:

```bash
grep -n "TODO" src/ledger/*.py
uv run ruff check
git status
git add .gitignore pyproject.toml uv.lock src
git commit -m "Store the transactions in a SQLite database with SQLAlchemy"
git push -u origin feature/database
```

<details>
<summary>✅ <b>What you should see</b></summary>

`grep` prints nothing. `git status` lists `.gitignore`, `pyproject.toml`, `uv.lock`, `storage.py` and `cli.py` as modified, and `src/ledger/db.py` as untracked. **`ledger.db` is not in the list.** If it is, fix `.gitignore` before you commit.

</details>

On GitHub, open a pull request from `feature/database` to `main`. In the description, explain the two tables and their relationship, how money is stored, and how you checked that the results didn't change. Merge it with **Create a merge commit**. Then, locally, mark this version with a **tag**, a name for one commit that doesn't move:

```bash
git switch main
git pull
git branch -d feature/database
git tag cp2
git push origin cp2
git log --oneline --decorate -3
```

<details>
<summary>✅ <b>What you should see</b></summary>

The last commit of `main`, the merge of the pull request, shows `(HEAD -> main, tag: cp2, origin/main)`. On GitHub, the tag appears under **Tags**. If you change something after tagging, move the tag to the new commit (`git tag -f cp2` and `git push -f origin cp2`), and submit the link to the new commit.

</details>

## Checkpoint 2

The tag `cp2` marks the commit you want graded. On Blackboard, in the assignment **Checkpoint 2**, submit the **link to that commit**. To get it, open `https://github.com/<your-username>/smd-app/tags`, click the commit id under `cp2` (seven characters, such as `a1b2c3d`), and copy the address from the browser. It looks like `https://github.com/<your-username>/smd-app/commit/a1b2c3d...`. You can also print it in the terminal, with your username:

```bash
echo "https://github.com/<your-username>/smd-app/commit/$(git rev-parse cp2)"
```

What is checked is that commit:

- [ ] **Package**: the modules `models.py`, `storage.py`, `reports.py` and `cli.py`; `reports.py` with no `print`, no file or database access, and no import of `cli`, `storage` or `db`;
- [ ] `[tool.ruff.lint]` rules in `pyproject.toml`, and `uv run ruff check` passing;
- [ ] **Database**: `db.py` with the tables `accounts` and `transactions`, and the foreign key `account_id` and the relationship `account` written by you;
- [ ] `load_transactions` returning `Transaction`s, and `models.py` and `reports.py` unchanged by this lab;
- [ ] `add_transaction` refusing unknown accounts and fractions of a cent with a `ValueError`;
- [ ] no `TODO` and no `NotImplementedError` left in the code;
- [ ] the commands `import`, `add`, `balance`, `spending` and `months` working from a clean clone (`uv run ledger import`, then the rest);
- [ ] `sqlalchemy` in the dependencies, and `*.db` in `.gitignore`, with no database file committed;
- [ ] two merged pull requests, from `refactor/package` (see [Lab: from one file to a package](../06-project-structure/lab.md)) and from `feature/database`, each with a description and merged with a merge commit;
- [ ] the tag `cp2` pushed to GitHub.

[← Previous: Databases and persistence](databases.md) [Back to the session ↑](index.md)
