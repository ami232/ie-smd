# Lab: a web API for the ledger

> ℹ️ **What you need first**
>
> Your repository `smd-app` with the transactions of `ledger` in a SQLite database (see [Lab: the ledger in a database](../07-databases/lab.md)) merged into `main`: the modules `models`, `storage`, `reports`, `cli` and `db`, and the commands `import`, `add`, `balance`, `spending` and `months`. The concepts of [Web APIs with FastAPI](notes.md): HTTP requests and responses, status codes, REST resources, path and query parameters, Pydantic models and `HTTPException`.

In this lab `ledger` gets a second interface: a **web API**, in a new module, `api.py`, next to `cli.py`. Other programs can then read the accounts, the transactions and the reports, and record and delete transactions, over HTTP. Storage changes a little, so that every transaction carries its `id`, the number that identifies it in the URLs; `reports.py` doesn't change at all. The code is on this page, except for a few places marked `# TODO`: the filters of the list of transactions, the `404` for a transaction that doesn't exist, and the `400` for a transaction that breaks a business rule.

You need **two terminals** in this lab: one runs the server, which occupies it until you stop it, and the other is for `curl`, Git and the command line.

## 1. Start a branch and a clean database

In your repository, `~/smd/smd-app`, bring `main` up to date with GitHub, check that the working tree is clean, and create a branch called `feature/api` for this lab.

Then start from a database with only the transactions of the CSV file, so that the `id`s on this page are the same as yours: delete `ledger.db`, import the CSV file again, and print the balances.

<details>
<summary>💡 <b>Hint</b></summary>

`rm` deletes a file. Then `uv run ledger ...`, once to import and once for the balances.

</details>

<details>
<summary>✅ <b>What you should see</b></summary>

`Imported 56 transactions from data/transactions.csv`, and the balances `3349.10` and `906.00`.

</details>

<details>
<summary>ℹ️ <b>My ledger is not in a database yet</b></summary>

Copy the files of the tag **`ledger-database`** of [github.com/ami232/ie-smd-ledger](https://github.com/ami232/ie-smd-ledger) into your repository (switch to the tag, **Code → Download ZIP**): `pyproject.toml`, `uv.lock` and the folder `src/`. Add `*.db` to `.gitignore`, commit them on `main` with a message that says where they come from, push, and then start this step. Checkpoint 2 checks your own database work, so finish it too if you can.

</details>

## 2. Add FastAPI

Add the package `fastapi[standard]` to the dependencies of your project. It is FastAPI with its usual extras: the server Uvicorn and the command `fastapi`.

<details>
<summary>💡 <b>Hint</b></summary>

`uv add ...`, with the package name between quotes, `"fastapi[standard]"`: without them, the shell reads the brackets as a file name pattern.

</details>

<details>
<summary>✅ <b>What you should see</b></summary>

`fastapi[standard]` in the `dependencies` of `pyproject.toml`, next to `rich` and `sqlalchemy`, and a long list of installed packages: `fastapi`, `pydantic`, `uvicorn`, `starlette`…

</details>

## 3. Storage: transactions with their id

To read or delete one transaction, the API names it by its `id` in the URL (`/api/transactions/57`), so a `Transaction` needs its `id`. In `src/ledger/models.py`, add a last field to `Transaction`, and complete its docstring:

```python
@dataclass(frozen=True)
class Transaction:
    """One movement of money in one account. Negative amounts are money going out.

    `id` is the number the database gave it, or None if it is not stored yet.
    """

    date: date
    account: str
    description: str
    category: str
    amount: Decimal
    id: int | None = None
```

The field has a default value, `None`, so `read_csv` keeps working without changes: a transaction read from the CSV file isn't stored yet.

In `src/ledger/storage.py`, **replace** the functions `load_transactions` and `add_transaction`, which are the last ones in the file, with these:

```python
def to_transaction(row: TransactionRow) -> Transaction:
    """The Transaction stored in one row of the table `transactions`."""
    return Transaction(
        date=row.date,
        account=row.account.name,
        description=row.description,
        category=row.category,
        amount=row.amount,
        id=row.id,
    )


def load_transactions(
    session: Session, account: str | None = None, category: str | None = None
) -> list[Transaction]:
    """The transactions in the database, oldest first, of one account or category
    if they are given."""
    query = select(TransactionRow).order_by(TransactionRow.date, TransactionRow.id)
    if account is not None:
        query = query.join(Account).where(Account.name == account)
    if category is not None:
        query = query.where(TransactionRow.category == category)
    return [to_transaction(row) for row in session.scalars(query)]


def get_transaction(session: Session, transaction_id: int) -> Transaction | None:
    """The transaction with this id, or None if there is none."""
    row = session.get(TransactionRow, transaction_id)
    if row is None:
        return None
    return to_transaction(row)


def add_transaction(
    session: Session,
    account_name: str,
    amount: Decimal,
    category: str,
    description: str,
    day: date,
) -> Transaction:
    """Records one transaction in an existing account, and returns it with its id."""
    account = find_account(session, account_name)
    if account is None:
        raise ValueError(f"there is no account called {account_name!r}")
    if amount != round(amount, 2):
        raise ValueError(f"{amount} has more than two decimals")
    row = TransactionRow(
        date=day,
        account=account,
        description=description,
        category=category,
        amount=amount,
    )
    session.add(row)
    session.commit()
    return to_transaction(row)


def delete_transaction(session: Session, transaction_id: int) -> bool:
    """Deletes the transaction with this id. Returns False if there is none."""
    row = session.get(TransactionRow, transaction_id)
    if row is None:
        return False
    session.delete(row)
    session.commit()
    return True
```

- `to_transaction` converts one row into a `Transaction`, with its `id`. It is what your `load_transactions` did inside its list comprehension, now in a function of its own, because three functions need it.
- `load_transactions` accepts two optional filters. Each one adds a condition to the query, so the database does the filtering, in SQL. Filtering by account name needs the table `accounts`, hence the `join`.
- `session.get(TransactionRow, 6)` reads the row with the primary key `6`, or returns `None`.
- `add_transaction` now returns the transaction it recorded, with the `id` that the database gave it on `commit`. Its two business rules are the ones you wrote.
- `delete_transaction` returns `False` when there is nothing to delete, so that the API can answer `404`.

The command line calls `load_transactions` and `add_transaction` as before, so it doesn't need any change. Check it: compare the output of `balance`, `spending` and `months` with the outputs you recorded in `~/smd/before`.

<details>
<summary>💡 <b>Hint</b></summary>

`uv run ledger balance | diff ~/smd/before/balance.txt -`, and the same for the other two.

</details>

<details>
<summary>✅ <b>What you should see</b></summary>

The three `diff` commands print nothing. (If you don't have the folder `~/smd/before`, print the balances and check that they are `3349.10` and `906.00`.)

</details>

## 4. The API module: `api.py`

Create `src/ledger/api.py` with this code:

```python
"""The web API: reads HTTP requests, calls storage and reports, returns JSON."""

import datetime
from collections.abc import Iterator
from contextlib import asynccontextmanager
from decimal import Decimal
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, status
from pydantic import BaseModel, Field
from sqlalchemy.orm import Session

from ledger import db, reports, storage


@asynccontextmanager
async def lifespan(app: FastAPI):
    db.create_tables()
    yield


app = FastAPI(title="ledger", lifespan=lifespan)


def get_session() -> Iterator[Session]:
    """One database session per request, closed when the response is sent."""
    with db.get_session() as session:
        yield session


SessionDep = Annotated[Session, Depends(get_session)]


class NewTransaction(BaseModel):
    """What the client sends to record a transaction."""

    date: datetime.date
    account: str = Field(min_length=1, max_length=50)
    description: str = Field(min_length=1, max_length=200)
    category: str = Field(min_length=1, max_length=50)
    amount: Decimal


class TransactionOut(NewTransaction):
    """What the API returns: a stored transaction, with its id."""

    id: int


class AccountOut(BaseModel):
    name: str
    balance: Decimal


@app.get("/api/accounts")
def list_accounts(session: SessionDep) -> list[AccountOut]:
    balances = reports.balances(storage.load_transactions(session))
    return [AccountOut(name=name, balance=total) for name, total in balances.items()]


@app.get("/api/transactions")
def list_transactions(session: SessionDep) -> list[TransactionOut]:
    # TODO 1: add two optional query parameters, account and category, and pass
    #         them to storage.load_transactions, which accepts them as keyword
    #         arguments.
    return storage.load_transactions(session)


@app.get("/api/transactions/{transaction_id}")
def get_transaction(transaction_id: int, session: SessionDep) -> TransactionOut:
    transaction = storage.get_transaction(session, transaction_id)
    # TODO 2: if there is no transaction with this id (transaction is None),
    #         raise an HTTPException with the status 404 and a clear message,
    #         as delete_transaction does.
    return transaction


@app.post("/api/transactions", status_code=status.HTTP_201_CREATED)
def create_transaction(new: NewTransaction, session: SessionDep) -> TransactionOut:
    # TODO 3: storage.add_transaction raises a ValueError when the transaction
    #         breaks a business rule. Catch it, and raise an HTTPException with
    #         the status 400 and the message of the error, str(error).
    return storage.add_transaction(
        session, new.account, new.amount, new.category, new.description, new.date
    )


@app.delete(
    "/api/transactions/{transaction_id}", status_code=status.HTTP_204_NO_CONTENT
)
def delete_transaction(transaction_id: int, session: SessionDep) -> None:
    if not storage.delete_transaction(session, transaction_id):
        raise HTTPException(
            status.HTTP_404_NOT_FOUND, f"there is no transaction {transaction_id}"
        )


@app.get("/api/reports/spending")
def spending(session: SessionDep) -> dict[str, Decimal]:
    return reports.spending_by_category(storage.load_transactions(session))


@app.get("/api/reports/months")
def months(session: SessionDep) -> dict[str, Decimal]:
    return reports.net_by_month(storage.load_transactions(session))
```

Read it from the top:

- **`lifespan`** runs once, when the server starts: it creates the tables that don't exist yet, as `main` does in `cli.py`. (`async` and `asynccontextmanager` are how FastAPI asks for it; the code before `yield` runs at start-up.)
- **`app`** is the application, the object the server runs.
- **`get_session`** and **`SessionDep`**: every endpoint with a parameter `session: SessionDep` gets its own database session, closed after the response (see [One database session per request](notes.md#one-database-session-per-request)).
- **`NewTransaction`** is what a client sends to record a transaction, and **`TransactionOut`** what the API returns: the same fields plus the `id`, which it inherits. **`AccountOut`** is one account with its balance. They are the API's contract; `Transaction` stays the program's own representation.
- **Seven endpoints**, all under `/api`: the accounts with their balances, the transactions (list, read one, create, delete), and two reports. Each one is a few lines: it calls `storage` or `reports`, and returns the result. The return type says what the JSON looks like: FastAPI converts the `Transaction`s that storage returns into `TransactionOut`s, and the `Decimal`s into strings.
- `status.HTTP_201_CREATED` is the number `201` with a name that says what it means.

## 5. Start the server

In your first terminal, start the API:

```bash
uv run fastapi dev src/ledger/api.py
```

<details>
<summary>✅ <b>What you should see</b></summary>

A few lines that end in `Server started at http://127.0.0.1:8000` and `Documentation at http://127.0.0.1:8000/docs`, then `Application startup complete.` The command doesn't end: the server is running, waiting for requests, and this terminal shows one line for each request it receives. Leave it running. Each time you save a file in `src/`, it restarts by itself.

</details>

Open [http://127.0.0.1:8000/api/accounts](http://127.0.0.1:8000/api/accounts) in your browser (in the browser of your virtual machine, if you work in one). A browser that loads a page sends a `GET` request.

<details>
<summary>✅ <b>What you should see</b></summary>

```json
[{"name":"checking","balance":"3349.10"},{"name":"savings","balance":"906.00"}]
```

The same balances as `uv run ledger balance`, calculated by the same function, `reports.balances`, now in JSON. The amounts are strings, so that no client reads them as floats. In the server's terminal, a line `"GET /api/accounts HTTP/1.1" 200 OK`.

</details>

Now open [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs): the interactive documentation that FastAPI writes from your code. Click **GET /api/reports/spending**, then **Try it out** and **Execute**.

<details>
<summary>✅ <b>What you should see</b></summary>

Under **Responses**, the code `200` and the spending by category, `"housing": "2850.00"` first. The page also shows the `curl` command it sent and the URL. At the bottom, under **Schemas**, your three Pydantic models (what the API accepts and returns) and the two that FastAPI adds for its `422` errors.

</details>

## 6. See the HTTP

The browser hides the HTTP conversation. `curl` shows it. Open a **second terminal**, go to your repository, and ask the API for transaction 6:

```bash
curl -i http://127.0.0.1:8000/api/transactions/6
```

`-i` includes the status line and the headers of the response in the output.

<details>
<summary>✅ <b>What you should see</b></summary>

```text
HTTP/1.1 200 OK
date: ...
server: uvicorn
content-length: 118
content-type: application/json

{"date":"2026-01-10","account":"checking","description":"Gym membership","category":"health","amount":"-39.90","id":6}
```

The status line, the headers, an empty line and the body: an HTTP response, exactly as it travels. Transaction `6` is the one you saw in the database as `(6, '2026-01-10', 1, 'Gym membership', 'health', -3990)`: here the account is a name and the amount a decimal string, because the API returns `Transaction`s, not rows.

</details>

## 7. Filter the transactions

Open [http://127.0.0.1:8000/api/transactions?category=health](http://127.0.0.1:8000/api/transactions?category=health) in the browser.

<details>
<summary>✅ <b>What you should see</b></summary>

All the 56 transactions: the endpoint ignores the query string, because the function has no parameter called `category`.

</details>

**TODO 1, in `api.py`:** in `list_transactions`, add two optional query parameters, `account` and `category`, and pass them to `storage.load_transactions`, as the comment describes.

<details>
<summary>💡 <b>Hints</b></summary>

- A parameter of the function that isn't in the path and has a simple type is a query parameter. With a default value, it is optional: `category: str | None = None` (see [Path and query parameters](notes.md#path-and-query-parameters)).
- `load_transactions(session, account=account, category=category)`.
- If Ruff complains that the line is too long, write the parameters on their own line, as `load_transactions` does in `storage.py`.

</details>

With `curl`, ask for the transactions of the category `health`. Then for those of the account `savings` and the category `interest`, together.

<details>
<summary>💡 <b>Hint</b></summary>

Put the whole URL between quotes, `curl "http://...?category=health"`: without them, the shell reads `?` and `&` itself. Two query parameters are joined with `&`.

</details>

<details>
<summary>✅ <b>What you should see</b></summary>

The three gym payments, with the ids `6`, `25` and `44`. Then the three interest payments of `savings`, `19`, `37` and `56`: two parameters, joined with `&`. In **/docs**, **GET /api/transactions** now shows the two parameters as fields to fill in.

</details>

## 8. A transaction that doesn't exist

With `curl -i`, ask for transaction `999`, which doesn't exist.

<details>
<summary>✅ <b>What you should see</b></summary>

`HTTP/1.1 500 Internal Server Error`, and in the server's terminal, a traceback that ends in `fastapi.exceptions.ResponseValidationError` with `'input': None`. `get_transaction` returned `None`, which is not a `TransactionOut`. To the client it looks as if the server is broken, when the truth is simply that transaction 999 doesn't exist.

</details>

**TODO 2, in `api.py`:** in `get_transaction`, raise an `HTTPException` with the status `404` and a clear message when there is no transaction with that id, as the comment describes.

<details>
<summary>💡 <b>Hints</b></summary>

- `delete_transaction`, at the end of the file, already does the same: copy its pattern.
- `raise` stops the function, so the `return` after the `if` only runs when the transaction exists.

</details>

Save the file (the server restarts) and ask for transaction `999` again. Then ask for transaction `abc`.

<details>
<summary>✅ <b>What you should see</b></summary>

`HTTP/1.1 404 Not Found`, with the body `{"detail":"there is no transaction 999"}` (or your message). And for `abc`: `422 Unprocessable Content`, with a body that says `Input should be a valid integer`. Your function didn't run: the type hint `transaction_id: int` made FastAPI reject the request.

</details>

## 9. Record a transaction

In the documentation page (reload it), open **POST /api/transactions**, click **Try it out**, and replace the example body with:

```json
{
  "date": "2026-03-31",
  "account": "checking",
  "description": "Lunch with a client",
  "category": "restaurants",
  "amount": "-12.50"
}
```

Click **Execute**.

<details>
<summary>✅ <b>What you should see</b></summary>

The code `201` and the transaction with its new `id`, `57`. Open [http://127.0.0.1:8000/api/accounts](http://127.0.0.1:8000/api/accounts) again: `checking` is now `3336.60`.

</details>

<details>
<summary>💡 <b>The same with curl</b></summary>

```bash
curl -i -X POST http://127.0.0.1:8000/api/transactions \
  -H "Content-Type: application/json" \
  -d '{"date": "2026-03-31", "account": "checking", "description": "Lunch with a client", "category": "restaurants", "amount": "-12.50"}'
```

`-X POST` sets the method, `-H` adds a header that says the body is JSON, and `-d` is the body. The `\` at the end of a line continues the command on the next one. Running it records a second lunch: `POST` is not idempotent.

</details>

Now send, with **Execute** each time, two bodies with the wrong shape. First, the same body with `"date": "31/03/2026"`. Then the same body without the line `"category"` (remove the comma before it too).

<details>
<summary>✅ <b>What you should see</b></summary>

Both times `422`, with a `detail` that names the field: `["body", "date"]` and `Input should be a valid date or datetime`, then `["body", "category"]` and `Field required`. Pydantic checked the shape of the body, and your function never ran.

</details>

Then a body with the right shape that breaks a business rule: the original body with `"account": "credit"`.

<details>
<summary>✅ <b>What you should see</b></summary>

`500 Internal Server Error`, and in the server's terminal a traceback that ends in `ValueError: there is no account called 'credit'`. Storage refused the transaction, as it should, but nothing turned its error into an HTTP answer.

</details>

**TODO 3, in `api.py`:** in `create_transaction`, catch the `ValueError` and raise an `HTTPException` with the status `400` and the message of the error, as the comment describes.

<details>
<summary>💡 <b>Hints</b></summary>

- `cli.py` does the same for the command line: `try:`, the call to storage, `except ValueError as error:`, and then a `raise`.
- The `return` can stay inside the `try`.
- End the `raise` with `from None`, as in `cli.py`. Without it, `uv run ruff check` reports `B904`: an exception raised inside an `except` should say what it comes from.

</details>

Send the body with `"account": "credit"` again, and then the original body with `"amount": "-1.005"`.

<details>
<summary>✅ <b>What you should see</b></summary>

`400` and `{"detail": "there is no account called 'credit'"}`, then `400` and `{"detail": "-1.005 has more than two decimals"}`: the same rules, and the same messages, as `uv run ledger add`, because both interfaces call the same function.

</details>

## 10. One database, two interfaces

The server and the command line are two programs that use the same file, `ledger.db`. In the second terminal, record with the command line a transaction of `15.00` in `savings`, category `interest`, description `Bonus interest`, dated `2026-03-31`. Then ask the API for the accounts.

<details>
<summary>💡 <b>Hint</b></summary>

`uv run ledger add ...`, with the arguments in the order of `uv run ledger add --help`, and the description between quotes. Then `curl` with the URL of the accounts.

</details>

<details>
<summary>✅ <b>What you should see</b></summary>

`Added 15.00 to savings`, and then `checking` `3336.60` and `savings` `921.00`: the API sees at once what the command line recorded, because each request opens a new session on the database.

</details>

Now delete the lunch, transaction `57`, through the API, twice. Then print the balances with the command line.

<details>
<summary>💡 <b>Hint</b></summary>

`-X` sets the method of the request: `curl -i -X ... http://...`.

</details>

<details>
<summary>✅ <b>What you should see</b></summary>

`HTTP/1.1 204 No Content` the first time, with no body. `404 Not Found` the second: the transaction no longer exists, and the database is the same after one `DELETE` or two. Then the balances `3349.10` and `921.00`: the lunch is gone and the bonus is still there.

</details>

Stop the server with `Ctrl+C` in its terminal.

## 11. Commit and merge

Check that no `TODO` is left in `src/ledger`, that Ruff's rules pass, and that the code is formatted.

<details>
<summary>💡 <b>Hint</b></summary>

`grep -n "TODO" ...` on the Python files of `src/ledger`. Then `uv run ruff ...`: `check` for the rules and `format --check` for the format.

</details>

<details>
<summary>✅ <b>What you should see</b></summary>

`grep` prints nothing, and Ruff passes. If `format --check` lists a file, run `uv run ruff format` and check again.

</details>

Commit your work on `feature/api` and push the branch. The commit has `pyproject.toml`, `uv.lock`, `models.py`, `storage.py` and the new `api.py`; check that `ledger.db` is **not** in it.

On GitHub, open a pull request from `feature/api` to `main`. In the description, list the endpoints of the API with their method, path and status codes, and explain which errors return `400`, `404` and `422`. Merge it with **Create a merge commit**. Then bring your local `main` up to date and delete the branch.

## What checkpoint 3 checks from this lab

This API is the first half of **checkpoint 3** (API and tests). There is no tag yet: checkpoint 3 also includes tests of this API. When they pass, you mark that commit with the tag `cp3` and submit the link to that commit on Blackboard, as for checkpoint 2. From this lab, it checks:

- [ ] `api.py` with the seven endpoints of this page, under `/api`, and the Pydantic models `NewTransaction`, `TransactionOut` and `AccountOut`;
- [ ] the filters `account` and `category` of `GET /api/transactions` (TODO 1);
- [ ] `404` for a transaction that doesn't exist, in `get_transaction` (TODO 2) and `delete_transaction`;
- [ ] `400` with the message of storage for a transaction that breaks a business rule (TODO 3), and `201` for one that is recorded;
- [ ] no SQL and no calculations in `api.py`: it calls `storage` and `reports`, and `reports.py` unchanged by this lab;
- [ ] `fastapi[standard]` in the dependencies, `uv run ruff check` passing, and no `TODO` left;
- [ ] the API working from a clean clone: `uv run ledger import`, then `uv run fastapi dev src/ledger/api.py`;
- [ ] a merged pull request from `feature/api`, with a description of the endpoints, merged with a merge commit.

[← Previous: Web APIs with FastAPI](notes.md) [Back to the session ↑](index.md)
