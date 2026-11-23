# Web APIs with FastAPI

> ℹ️ **What you need first**
>
> `ledger` as a package whose transactions are in a SQLite database through SQLAlchemy (see [Lab: the ledger in a database](../07-databases/lab.md)). The ideas of [Software design and project structure](../06-project-structure/notes.md), especially [the direction of dependencies](../06-project-structure/notes.md#the-direction-of-dependencies): the interface calls storage and the calculations, never the other way round. What a session and a database transaction are (see [From SQL to Python: the ORM](../07-databases/notes.md#from-sql-to-python-the-orm)).

## Why an API

`ledger` has one interface, the command line. It is perfect for you at your terminal, and useless for everyone else:

- A **web page** that shows your balances runs in the browser, on another computer. It can't run `uv run ledger balance` on yours.
- A **mobile app** needs the same data, with the same rules, without a copy of the code.
- **Another program**, such as your bank sending the day's transactions, needs a way to add them without anyone typing.

All of them need the same thing: to send a request over the network to a program that is always running, and get data back in a format a program can read. That program is a **server**, and the set of requests it accepts is its **API**.

> 📘 **Definition: API, client and server**
>
> An **API** (*application programming interface*) is the set of operations a program offers to other programs, with the exact form of each request and each answer. A **web API** offers them over the network, with HTTP.
>
> The program that offers the API is the **server**, and it runs all the time, waiting for requests. A program that sends requests to it is a **client**: a web page, an app, a script, or you with a tool such as `curl`.

An API is a second **interface** to the same application, next to the command line. Both read a request (arguments in the terminal, an HTTP request on the network), call storage and the calculations, and give back the result (printed text, or data in JSON). That is why, in `ledger`, the API goes in a module of its own, `api.py`, at the same level as `cli.py`:

```text
                 cli.py ──┐
  (a person)              ├──▶ storage.py ──▶ db.py ──▶ ledger.db
                 api.py ──┤
  (a program)             └──▶ reports.py ──▶ models.py
```

Neither `storage` nor `reports` knows that an API exists. This is the [direction of dependencies](../06-project-structure/notes.md#the-direction-of-dependencies) paying off: adding a whole new interface doesn't change the calculations at all.

## HTTP: requests and responses

The web works with one protocol, **HTTP** (*Hypertext Transfer Protocol*). The browser uses it to load pages, and APIs use it to exchange data. HTTP is a conversation in turns: the client sends a **request**, and the server answers with one **response**. Both are plain text with a fixed structure.

This is a real request to the `ledger` API, asking for one transaction, and its response:

```text
GET /api/transactions/6 HTTP/1.1
Host: 127.0.0.1:8000
Accept: application/json
```

```text
HTTP/1.1 200 OK
content-type: application/json
content-length: 118

{"date":"2026-01-10","account":"checking","description":"Gym membership","category":"health","amount":"-39.90","id":6}
```

> 📘 **Definition: HTTP request and response**
>
> An **HTTP request** has:
>
> - a **method**, the kind of operation: `GET` (read), `POST` (create), `DELETE` (delete)…;
> - a **path**, which says what the request is about: `/api/transactions/6`;
> - optionally a **query string** after `?`, with extra parameters: `/api/transactions?category=health`;
> - **headers**, `Name: value` lines with information about the request, such as the format of the body (`Content-Type: application/json`);
> - optionally a **body**, the data sent, for example the new transaction to record.
>
> An **HTTP response** has a **status code**, a three-digit number that says how it went (`200 OK`, `404 Not Found`…), headers, and usually a body with the data.

The path and the query string, together with the server's address, form the **URL**: `http://127.0.0.1:8000/api/transactions?category=health`. `127.0.0.1` (also called `localhost`) always means "this same computer", and `8000` is the **port**, the number that identifies one server program among all those running on the machine.

### Methods

The method tells the server what to do with what the path names. Five methods cover nearly every API:

| Method | Meaning | Body in the request | Safe | Idempotent |
|---|---|---|---|---|
| `GET` | read | no | yes | yes |
| `POST` | create something new | yes | no | no |
| `PUT` | replace something completely | yes | no | yes |
| `PATCH` | change some fields of something | yes | no | no |
| `DELETE` | delete | no | no | yes |

A **safe** method doesn't change anything on the server: you can send a `GET` as many times as you like. An **idempotent** method has the same effect whether you send it once or several times: deleting transaction 57 twice leaves the database exactly as deleting it once. `POST` is neither: sending the same `POST` twice records two transactions, the same problem as importing the CSV file twice. These properties matter on real networks, where a client that gets no answer doesn't know whether its request arrived and may send it again.

### Status codes

The status code is the first thing a client reads. Its first digit gives the family:

| Code | Name | When |
|---|---|---|
| `200` | OK | the request worked; the data is in the body |
| `201` | Created | a `POST` created something; the body has it, with its new `id` |
| `204` | No Content | it worked and there is nothing to return, as after a `DELETE` |
| `400` | Bad Request | the request is understandable but breaks a rule: an account that doesn't exist |
| `404` | Not Found | what the path names doesn't exist: transaction 999 |
| `405` | Method Not Allowed | the path exists, but not with this method |
| `422` | Unprocessable Content | the data has the wrong shape: a date that isn't a date, a missing field |
| `500` | Internal Server Error | the server failed: a bug, not the client's fault |

The rule to remember: **`2xx` it worked, `4xx` the client must change its request, `5xx` the server has a problem**. A `4xx` comes with a message that tells the client what to fix. A `500` means your code raised an exception that nobody handled, so in a finished API every error that a client can cause gets its own `4xx`. A frequent mistake is to answer `200` with a body like `{"error": "not found"}`: the client's code checks the status code first, sees success, and carries on with wrong data.

### JSON

The body of API requests and responses is almost always **JSON** (*JavaScript Object Notation*), a text format with the same building blocks as Python's data: objects `{"key": value}` (Python dictionaries), arrays `[...]` (lists), strings, numbers, `true`/`false` and `null` (`None`).

JSON has one kind of number, and most clients, browsers included, read it as a binary floating-point number, with the errors you saw when [money is a float](../04-dependencies-tools/lab.md#3-money-is-not-a-float). That is why the `ledger` API sends amounts as **strings**: `"amount": "-39.90"`. The client that needs to calculate converts the string with an exact decimal type; the one that only shows it, shows it as it is. Many payment APIs do the same, or send integer cents.

## REST: an API made of resources

Nothing in HTTP forces a particular design of URLs. You could have `/getBalance`, `/doAddTransaction?acc=checking` and `/transactions/remove/57`, and each team would invent its own conventions. **REST** (*Representational State Transfer*) is the style most web APIs follow so that clients can guess how an API works.

> 📘 **Definition: REST resource**
>
> In a REST API, the URLs name **resources**, the things the application manages, with **nouns in the plural**: `/api/transactions` is the collection of all transactions, and `/api/transactions/57` is one of them, identified by its `id`. The **method** says what to do with the resource. The server keeps no memory of earlier requests from the client: each request carries everything needed to answer it (REST is **stateless**).

Whatever an application stores, it does four things with it: **create**, **read**, **update** and **delete**, known together as **CRUD**. CRUD is not a technology: it names the operations themselves. Each layer of the application has its own way to express them, and an HTTP method is the API's way:

| CRUD | SQL (in the database) | HTTP method (in the API) | `ledger` command line |
|---|---|---|---|
| create | `INSERT` | `POST` | `ledger add` |
| read | `SELECT` | `GET` | `ledger balance`, `spending`, `months` |
| update | `UPDATE` | `PUT` (replace) or `PATCH` (change some fields) | — |
| delete | `DELETE` | `DELETE` | — |

When a client sends `POST /api/transactions`, it asks for a *create*; the endpoint calls storage, and SQLAlchemy sends an `INSERT` to the database. The same operation travels through the three layers, with a different name in each. For transactions, the requests for each operation are (the `ledger` API has no update: a wrong transaction is deleted and recorded again):


| Operation | HTTP request | Success |
|---|---|---|
| list transactions, optionally filtered | `GET /api/transactions?category=health` | `200` and a list |
| read one transaction | `GET /api/transactions/57` | `200` and the transaction, or `404` |
| create a transaction | `POST /api/transactions` with the transaction in the body | `201` and the transaction, with its `id` |
| replace a transaction | `PUT /api/transactions/57` with the whole transaction | `200` and the transaction |
| change some fields | `PATCH /api/transactions/57` with those fields | `200` and the transaction |
| delete a transaction | `DELETE /api/transactions/57` | `204`, or `404` |

The path is for **which** resource; the query string is for **how** to filter, sort or page a list; the body is for the **data** of what you create or change. Verbs don't go in the path (`/api/deleteTransaction/57` is not REST: the method already says "delete"). Calculated results that are not stored, such as the balances or the spending by category, are resources too: `GET /api/accounts` returns each account with its balance, and `GET /api/reports/spending` the spending report.

## FastAPI

**FastAPI** is a Python framework for building web APIs. You write ordinary functions with type hints, and FastAPI uses the hints to read each request, check it, call your function and turn its result into JSON. A complete API with one endpoint:

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/api/hello")
def hello() -> dict[str, str]:
    return {"message": "Hello"}
```

`app` is the application. The **decorator** `@app.get("/api/hello")`, the line with `@` above the function, registers the function as the answer to `GET /api/hello`; FastAPI calls this pair of method and path a **path operation**, and the function, its **endpoint**. The function returns a dictionary, and FastAPI sends `{"message":"Hello"}` with the status `200`.

A server program runs the application: it listens on a port, receives each request, and passes it to FastAPI. The command `fastapi dev` starts **Uvicorn**, the usual server, in development mode: it restarts the API each time you save a file. While it runs, the API is at `http://127.0.0.1:8000`, and the terminal shows one line per request with its status code. `Ctrl+C` stops it.

### Path and query parameters

```python
@app.get("/api/transactions/{transaction_id}")
def get_transaction(transaction_id: int, session: SessionDep) -> TransactionOut:
    ...


@app.get("/api/transactions")
def list_transactions(
    session: SessionDep, account: str | None = None, category: str | None = None
) -> list[TransactionOut]:
    ...
```

- A name between braces in the path, `{transaction_id}`, is a **path parameter**: for `GET /api/transactions/6`, FastAPI calls the function with `transaction_id=6`. The type hint `int` converts the text `"6"` into the number `6`, and a request for `/api/transactions/abc` is rejected with `422` before your function runs.
- Any other parameter of a simple type (`str`, `int`, `date`…) is a **query parameter**, read from the query string: `GET /api/transactions?category=health` gives `category="health"`. With a default value (`= None`) it is optional; without one, it is required.
- `session: SessionDep` is neither: it is a **dependency**, explained below.

### The request body: Pydantic models

To create a transaction, the client sends it as JSON in the body. You describe the shape of that JSON with a **Pydantic model**, a class whose fields have types and, optionally, constraints:

```python
from pydantic import BaseModel, Field


class NewTransaction(BaseModel):
    """What the client sends to record a transaction."""

    date: datetime.date
    account: str = Field(min_length=1, max_length=50)
    description: str = Field(min_length=1, max_length=200)
    category: str = Field(min_length=1, max_length=50)
    amount: Decimal


@app.post("/api/transactions", status_code=201)
def create_transaction(new: NewTransaction, session: SessionDep) -> TransactionOut:
    ...
```

A parameter whose type is a Pydantic model is read from the **body**. Before calling the function, FastAPI checks the JSON against the model: every field present, `"2026-03-31"` converted into a `date`, `"-12.50"` into a `Decimal`, the texts not empty. If anything fails, the client gets `422` with the list of problems, and your function never runs:

```json
{"detail": [
  {"type": "date_from_datetime_parsing", "loc": ["body", "date"], "msg": "Input should be a valid date or datetime, invalid character in year", "input": "31/03/2026"},
  {"type": "decimal_parsing", "loc": ["body", "amount"], "msg": "Input should be a valid decimal", "input": "twelve"}
]}
```

The **return type** of the function works the same way in the other direction: FastAPI converts what the function returns into the shape of `TransactionOut`, and sends only its fields. The `Transaction` dataclass of `models.py` is converted automatically. If what the function returns doesn't fit (for example `None` where a transaction was expected), that is a bug in the server, and the client gets `500`.

A Pydantic model looks like a dataclass, and it is used for a different job. The `Transaction` dataclass is how the program represents a transaction inside; `NewTransaction` and `TransactionOut` are the **contract** of the API with the outside, the JSON that comes in and goes out. They are kept separate on purpose: the contract can stay stable while the inside changes, and the client never chooses fields it shouldn't, such as the `id`, which only the database assigns.

### Two kinds of invalid request

Pydantic checks the **shape** of the data: types, required fields, lengths. It can't know whether the account `credit` exists, because that depends on what is in the database. That is a **business rule**, and `ledger` checks its business rules in `storage.add_transaction`, which raises a `ValueError`, for the command line and the API alike. The endpoint turns it into a `400` with `HTTPException`:

```python
from fastapi import HTTPException


@app.post("/api/transactions", status_code=201)
def create_transaction(new: NewTransaction, session: SessionDep) -> TransactionOut:
    try:
        return storage.add_transaction(
            session, new.account, new.amount, new.category, new.description, new.date
        )
    except ValueError as error:
        raise HTTPException(400, str(error)) from None
```

Raising `HTTPException` stops the function and sends a response with that status code and `{"detail": "..."}` as its body. It is the API's version of what `cli.py` does with `SystemExit`: the same error from storage, turned into the right message for this interface. The same pattern gives the `404` when the transaction asked for doesn't exist.

| Problem | Example | Who checks it | Status |
|---|---|---|---|
| the data has the wrong shape | `"date": "31/03/2026"`, no `amount` | Pydantic, automatically | `422` |
| the data breaks a business rule | the account `credit` doesn't exist | `storage`, with a `ValueError` | `400` |
| the resource doesn't exist | `GET /api/transactions/999` | the endpoint | `404` |
| the server has a bug | an exception nobody handled | nobody | `500` |

### One database session per request

Every endpoint that uses the database needs a session, and the session must be closed when the request is answered, also if it ends in an error. FastAPI does this with a **dependency**: a function that provides something to the endpoints that ask for it.

```python
from collections.abc import Iterator
from typing import Annotated

from fastapi import Depends


def get_session() -> Iterator[Session]:
    """One database session per request, closed when the response is sent."""
    with db.get_session() as session:
        yield session


SessionDep = Annotated[Session, Depends(get_session)]
```

For each request to an endpoint with a parameter `session: SessionDep`, FastAPI calls `get_session`, which opens a session and **yields** it (`yield` hands it over and waits). The endpoint runs with that session. When the response has been sent, `get_session` continues after the `yield`, and the `with` block closes the session. Every request gets its own session, so two clients at the same time never share uncommitted changes. `SessionDep` is a name for the long type `Annotated[Session, Depends(get_session)]`, written once and used in every endpoint.

### The interactive documentation

From the paths, the parameters and the models, FastAPI writes a complete description of the API in a standard format, **OpenAPI**, at `/openapi.json`. At `/docs` it shows that description as a web page: every endpoint, its parameters, the shape of its body and its answers, with a **Try it out** button that sends real requests. It is always up to date, because it is generated from the code, and it is what a frontend developer reads to use your API.

## Where the code of an API goes

The endpoints of `ledger` are short: each one reads the request, calls one function of `storage` or `reports`, and returns the result or the right error. That is the job of an interface, and nothing more:

- **No SQL in the endpoints.** Queries are storage's job. If the API and the command line need the same data, they call the same storage function.
- **No calculations in the endpoints.** The balances come from `reports.balances`, the same function the command line uses. If the API calculated them its own way, sooner or later the two would disagree.
- **No business rules only in the API.** A rule checked only in the endpoint (the account must exist) doesn't protect the data when it arrives through the command line. Rules go in storage, or deeper, and the interface translates their errors.

With this split, the API module is easy to read and easy to replace, and everything below it can be tested without a server.

## Designing the API of your module

The tables of your module (see [Designing the tables of your module](../07-databases/notes.md#designing-the-tables-of-your-module)) give its API almost directly:

1. **One collection per entity that clients manage**, under the prefix of your module: `/api/invoices`, `/api/invoices/{invoice_id}`.
2. **The CRUD operations each client needs**, and no more. A transaction in `ledger` can't be edited, only deleted and recorded again, so there is no `PUT`. Don't add endpoints "just in case": each one is code to test and to keep working.
3. **Two models per entity**: what the client sends to create it (without `id`), and what the API returns (with `id`).
4. **The errors of each endpoint**: which business rules can fail (`400`) and which ids may not exist (`404`).
5. **Calculated results as resources of their own**, with `GET`: `/api/invoices/overdue`, `/api/reports/revenue`.

In a team product, all the modules form **one API**, so the conventions must be the same everywhere: the URL prefix of each module, the field names and date format in JSON, money as strings, and the same status codes for the same situations. Agree on them before writing endpoints, and write them in the repository's README.

## Common mistakes

- **Verbs in the URLs.** `/api/addTransaction`, `/api/transactions/delete/57`. The method is the verb: `POST /api/transactions`, `DELETE /api/transactions/57`.
- **`200` for everything.** An error with status `200` looks like a success to the client. Use the status code that matches what happened.
- **Leaving the `500`s.** An exception that the endpoint doesn't catch, such as a `ValueError` from storage, reaches the client as `500 Internal Server Error`, with no explanation. Every error the client can cause gets a `4xx` and a message.
- **Money as a JSON number.** `-12.5` is read as a float by most clients. Send amounts as strings, or as integer cents.
- **SQL or calculations in the endpoints.** The API and the command line drift apart. Endpoints call storage and reports.
- **Opening the database at import time, or sharing one session between requests.** Use a dependency that opens a session per request and closes it.
- **Forgetting that the server is running.** `fastapi dev` occupies its terminal until `Ctrl+C`. Use a second terminal for `curl` and the command line, and stop the server before starting it again: a second one fails with `Address already in use`.

> ✅ **Key takeaways**
>
> - A **web API** lets other programs use an application over the network with **HTTP**. A request has a **method**, a **path**, an optional **query string**, **headers** and an optional **body**; the response has a **status code**, headers and a body, usually **JSON**.
> - The status code says how it went: **`2xx`** it worked (`200`, `201` created, `204` nothing to return), **`4xx`** the client must change its request (`400` business rule, `404` not found, `422` wrong shape), **`5xx`** the server failed.
> - A **REST** API names **resources** with plural nouns (`/api/transactions/57`) and uses the method as the verb: `GET` reads, `POST` creates, `PUT`/`PATCH` change, `DELETE` deletes (**CRUD**).
> - In **FastAPI**, a decorated function is an endpoint. Its type hints declare the **path** and **query parameters**, the **body** as a **Pydantic model** (checked automatically, `422` if the shape is wrong) and the response. `HTTPException` returns an error with its status code, and a **dependency** gives each request its own database session.
> - The API is an **interface**, like the command line: it reads the request, calls storage and reports, and translates their results and errors. No SQL, calculations or business rules live only in the API.

## Exercises

**1.** Give the method, the path and the status code of the successful response for each request to the `ledger` API:

1. The spending report.
2. All the transactions of the account `savings`.
3. Record a coffee of 2.10 € paid from `checking`.
4. Delete transaction 12.
5. The same request as 4, sent a second time.

<details>
<summary>✅ <b>Solution</b></summary>

1. `GET /api/reports/spending`, `200`.
2. `GET /api/transactions?account=savings`, `200`: the account is a filter on the collection, so it goes in the query string.
3. `POST /api/transactions` with the transaction in the body as JSON (`"amount": "-2.10"`), `201`, and the body of the response is the transaction with its new `id`.
4. `DELETE /api/transactions/12`, `204`, with no body.
5. `DELETE /api/transactions/12` again, `404`: transaction 12 no longer exists. The database is the same after one `DELETE` or two, which is what makes `DELETE` idempotent, even though the status code of the second one is different.

</details>

**2.** A client sends `POST /api/transactions` with each of these bodies. Which status code does the `ledger` API answer, and which part of the code decides it?

1. `{"date": "2026-04-01", "account": "checking", "description": "Taxi", "category": "transport", "amount": "-18.40"}`
2. The same, without `"category"`.
3. The same as 1, with `"account": "credit"`.
4. The same as 1, with `"amount": "-18.405"`.

<details>
<summary>✅ <b>Solution</b></summary>

1. `201`: the shape is right and the rules hold. The body of the response is the transaction with its `id`.
2. `422`, decided by Pydantic: `category` is a required field of `NewTransaction`, so the function `create_transaction` never runs.
3. `400`, decided by `storage.add_transaction`, which raises a `ValueError` because there is no account `credit`. The endpoint turns it into an `HTTPException`. Pydantic can't check it: whether an account exists depends on the database.
4. `400`, also decided by storage: `-18.405` is a valid `Decimal`, so the shape is right, but it has a fraction of a cent. (The rule could also be a Pydantic constraint on the model, `Field(decimal_places=2)`, and then it would be a `422`; keeping it in storage protects the command line too.)

</details>

**3.** A colleague designs these endpoints for the invoices module of a freelancers' product. Rewrite them in REST style, with the method, the path and the status code of success.

```text
GET  /api/getInvoices
POST /api/createInvoice
GET  /api/invoice?id=7
POST /api/deleteInvoice?id=7
GET  /api/invoicesOfClient?client=3
POST /api/markInvoicePaid?id=7
```

<details>
<summary>✅ <b>Solution</b></summary>

```text
GET    /api/invoices                 200  list
POST   /api/invoices                 201  the new invoice, with its id
GET    /api/invoices/7               200  one invoice, or 404
DELETE /api/invoices/7               204, or 404
GET    /api/invoices?client_id=3     200  the list, filtered
PATCH  /api/invoices/7               200  body {"paid": true}
```

The verbs leave the paths, because the method says what to do. The `id` of one invoice goes in the path, because it names the resource; the client is a filter on the collection, so it goes in the query string. Marking an invoice as paid changes one field, so it is a `PATCH`. If paying an invoice also records a payment (an amount and a date), another option is to create that payment as a resource of its own: `POST /api/invoices/7/payments`, `201`.

</details>

**4.** This endpoint is in a colleague's API. Find three problems and fix them.

```python
@app.get("/api/accounts/{name}/balance")
def balance(name: str) -> dict:
    with sqlite3.connect("ledger.db") as con:
        rows = con.execute(f"SELECT amount FROM transactions WHERE account = '{name}'")
        total = sum(float(row[0]) for row in rows)
    if total == 0:
        return {"error": "account not found"}
    return {"balance": total}
```

<details>
<summary>✅ <b>Solution</b></summary>

- **SQL built with an f-string**: a name such as `x' OR '1'='1` changes the query (SQL injection). Besides, SQL in an endpoint doesn't belong to the interface: the query goes in storage, with parameters or the ORM.
- **Money as `float`**: the total gets binary errors, and goes to the client as a JSON number. The balance comes from `reports.balances`, as a `Decimal`, and is sent as a string.
- **Error with status `200`**: the client sees success. And a balance of zero doesn't mean that the account doesn't exist. The endpoint must answer `404` with `HTTPException` when the account doesn't exist.

One possible version, with the functions of `ledger`:

```python
@app.get("/api/accounts/{name}")
def get_account(name: str, session: SessionDep) -> AccountOut:
    if storage.find_account(session, name) is None:
        raise HTTPException(404, f"there is no account called {name!r}")
    transactions = storage.load_transactions(session, account=name)
    total = reports.balances(transactions).get(name, Decimal("0.00"))
    return AccountOut(name=name, balance=total)
```

The path names the resource, the account; its balance is one of the fields returned.

</details>

[← Back to the session](index.md) [Next: Lab →](lab.md)
