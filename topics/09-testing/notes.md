# Testing and continuous integration

> ℹ️ **What you need first**
>
> `ledger` as a package with a database and a web API (see [Lab: a web API for the ledger](../08-fastapi/lab.md)): the pure calculations of `reports`, the functions of `storage` with a SQLAlchemy session, and the endpoints of `api.py`. Pull requests and a protected `main` (see [The workflow for a small team](../05-git-collaboration/notes.md#the-workflow-for-a-small-team)).

## Why automated tests

Each time `ledger` changed, you checked it by hand. After splitting it into a package, you compared the reports with `diff`. After moving it to a database, you compared them again and looked inside the tables. After adding the API, you sent requests with `curl` and read the answers. Every check worked, and every one had the same weaknesses:

- **It takes time**, so you run it once, after the change, and not again.
- **You only check what you remember.** Nobody re-checks the 400 for an unknown account after changing how the balances are calculated, yet a change in one place can break another. A feature that worked and stops working is a **regression**.
- **It lives in your head.** A teammate who changes your module doesn't know what you checked, or what the right answers were.

An **automated test** writes the check down as code: it runs a piece of the program with known data and compares the result with the expected one. A computer runs hundreds of them in a second, every time, exactly the same way.

> 📘 **Definition: automated test**
>
> An **automated test** is a small program that runs a piece of the application with fixed inputs and checks that the result is the expected one. It **passes** if every check holds and **fails** otherwise. A set of tests is a **test suite**.

Tests give a team three things. **Confidence to change code**: if the suite passes after a change, the behaviours it covers still work, which is what makes refactoring safe. **Early errors**: a failing test points to the function and the line, minutes after the mistake, not weeks later in a user's complaint. And **documentation that can't go out of date**: a test such as `test_a_transfer_is_not_spending` states a business rule, and it fails the day the code stops following it.

## Tests with pytest

**pytest** is the standard tool for testing in Python. A test is a function whose name starts with `test_`, in a file whose name starts with `test_`, usually in a folder `tests/`. Inside it, `assert` checks a condition:

```python
from datetime import date
from decimal import Decimal

from ledger import reports
from ledger.models import Transaction


def test_balances():
    transactions = [
        Transaction(date(2026, 1, 1), "checking", "Salary", "salary", Decimal("2000.00")),
        Transaction(date(2026, 1, 2), "checking", "Carrefour", "groceries", Decimal("-65.15")),
    ]

    result = reports.balances(transactions)

    assert result == {"checking": Decimal("1934.85")}
```

`uv run pytest` finds every test in the project, runs it, and prints one character per test: `.` if it passed, `F` if it failed. At the end, a summary:

```text
tests/test_reports.py .....                                              [100%]

============================== 5 passed in 0.01s ===============================
```

`assert` is plain Python: `assert condition` does nothing if the condition is true, and raises `AssertionError` if it is false. pytest catches that error, marks the test as failed, and shows the values on both sides of the comparison, so you see what the code returned and what the test expected.

### Arrange, act, assert

Almost every test has three parts, in this order, separated by an empty line:

1. **Arrange**: prepare the data, here the list of transactions.
2. **Act**: call the code being tested, once.
3. **Assert**: check the result.

The structure makes a test easy to read, and it keeps each test about **one behaviour**. A test that calls three functions and checks ten things fails for many reasons, and its name can't say which.

### What to test

The easiest code to test is a **pure function**: the same inputs always give the same output, with no files, database or screen involved. That is why `ledger` keeps its calculations in `reports.py`, without `print` or storage (see [Software design and project structure](../06-project-structure/notes.md)): its tests need nothing but some `Transaction`s.

For each function, test three kinds of case:

- **The usual case**: a small, realistic input whose result you have calculated by hand.
- **The edge cases**: an empty list, a single transaction, a transfer, a month with no movements. Bugs gather at the edges.
- **The errors**: what must be refused (an unknown account, a fraction of a cent), and how.

Calculate the expected results **by hand**, never by running the code and copying what it prints: a test that copies the output of a bug confirms the bug. With small inputs, the calculation by hand is quick: salary 2000.00, groceries −65.15, so the balance is 1934.85.

Name each test after the behaviour it checks, as a sentence: `test_a_transfer_is_not_spending`, `test_an_unknown_account_is_refused`. When it fails, the name alone tells you which rule is broken.

## Fixtures

Several tests often need the same data. Copying the list of transactions into each test makes them long, and a change to the data has to be made everywhere. A **fixture** prepares it once:

```python
import pytest


@pytest.fixture
def transactions() -> list[Transaction]:
    """Two months of a small ledger, with a transfer between the two accounts."""
    return [
        make("2026-01-01", "checking", "salary", "2000.00"),
        make("2026-01-02", "checking", "groceries", "-65.15"),
        make("2026-01-15", "checking", "transfer", "-300.00"),
        make("2026-01-15", "savings", "transfer", "300.00"),
        ...
    ]


def test_balances(transactions):
    result = reports.balances(transactions)
    ...
```

> 📘 **Definition: fixture**
>
> A **fixture** is a function decorated with `@pytest.fixture` that prepares something a test needs. A test asks for it by having a **parameter with the same name**: pytest calls the fixture before the test and passes the result in. Each test gets a **new** result, so no test can change the data of another.

pytest also has **built-in fixtures**. `tmp_path` gives each test a new, empty temporary folder, which is perfect to test `import_csv` with a small CSV file written by the test itself. Fixtures used by several test files go in a file called `conftest.py` in the `tests/` folder: pytest loads it on its own, without any import.

A fixture can also **clean up** after the test. Instead of `return`, it uses `yield`: the code before `yield` runs before the test, and the code after it runs after the test, even if the test failed. It is the same idea as the dependency `get_session` of the API.

## Testing code that uses a database

Storage reads and writes a database. Tests can't use `ledger.db`: they would depend on what is in it, and they would change it. Each test needs a database of its own, empty at the start, that disappears at the end.

SQLite can keep a whole database **in memory**, with the URL `sqlite://` (no file name). Creating one takes a millisecond, so every test can have a new one. This fixture, in `tests/conftest.py`, creates the tables of `db.py` in an empty database in memory and gives the test a session on it:

```python
@pytest.fixture
def session():
    """An empty database in memory, new for each test."""
    engine = create_engine(
        "sqlite://",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    Base.metadata.create_all(engine)
    with Session(engine) as session:
        yield session
```

A database in memory exists only inside one connection, so `StaticPool` makes the engine use always the same connection, and `check_same_thread=False` lets the API use it from the threads where FastAPI runs the endpoints. The tables are the ones declared in `db.py`, the same that the application uses.

A test of storage then writes and reads through the real functions:

```python
def test_add_and_load_a_transaction(db):
    storage.add_transaction(db, "checking", Decimal("-12.50"), "food", "Lunch", DAY)

    [loaded] = storage.load_transactions(db)

    assert loaded.account == "checking"
    assert loaded.amount == Decimal("-12.50")
```

Here `db` is another fixture that takes `session` and adds an account `checking` to it: fixtures can use other fixtures. The test checks that a transaction survives the trip to the database and back: the `Money` type stores it as cents, and storage returns it as a `Decimal` with the same value.

## Testing errors

A business rule says that a transaction for an account that doesn't exist must be refused with a `ValueError`. Checking it means checking that the exception **is raised**. `pytest.raises` does it:

```python
def test_an_unknown_account_is_refused(db):
    with pytest.raises(ValueError, match="no account called 'credit'"):
        storage.add_transaction(db, "credit", Decimal("-5"), "food", "Coffee", DAY)

    assert storage.load_transactions(db) == []
```

The test passes only if the code inside the `with` block raises a `ValueError` whose message contains the text of `match`. If it raises nothing, or another exception, the test fails. The last line checks the second half of the rule: nothing was saved.

## Testing an API

An API is tested through HTTP requests, as a client would use it, but without starting a server. FastAPI's **`TestClient`** sends the requests directly to the application, and returns the responses:

```python
from fastapi.testclient import TestClient

from ledger import api


def test_a_missing_transaction_is_404(client):
    response = client.get("/api/transactions/999")

    assert response.status_code == 404
```

`response.status_code` is the status code, and `response.json()` the body converted into Python: a list or a dictionary, with the amounts as strings, as the client receives them. `client.post(url, json={...})` sends a body.

The API must use the database in memory of the test, not `ledger.db`. FastAPI lets a test replace a dependency with another one: `api.app.dependency_overrides[api.get_session] = lambda: session` makes every endpoint receive the session of the test. The fixture `client` does this, adds two accounts and two transactions, and removes the replacement at the end:

```python
@pytest.fixture
def client(session):
    session.add_all([Account(name="checking"), Account(name="savings")])
    session.commit()
    ...
    api.app.dependency_overrides[api.get_session] = lambda: session
    yield TestClient(api.app)
    api.app.dependency_overrides.clear()
```

An API test checks the **contract** with the client: the status code for each situation (201, 400, 404, 422) and the exact JSON. These are the things a frontend depends on, and that a change inside the API must not break.

## Reading a failing test

A failing test is information, not a disaster. Suppose someone simplifies `is_spending` and forgets the transfers:

```python
def is_spending(t: Transaction) -> bool:
    return t.amount < 0
```

Two tests fail, and pytest explains each one:

```text
>       assert result == {"groceries": Decimal("105.15"), "restaurants": Decimal("25.50")}
E       AssertionError: assert {'transfer': ...imal('25.50')} == {'groceries':...imal('25.50')}
E
E         Omitting 2 identical items, use -vv to show
E         Left contains 1 more item:
E         {'transfer': Decimal('300.00')}
```

Read it from the bottom: the result (the left side) has one item more than expected, `transfer`, with 300.00. The transfer was counted as spending. The line marked `>` is the assertion that failed, and the file and line number follow. The other failing test, `test_a_transfer_is_not_spending`, names the broken rule directly.

When a test fails, there are two possibilities: the code is wrong, or the test is. Decide which by going back to the rule, not by changing the expected value until the test passes.

## Kinds of tests

| Kind | What it checks | In `ledger` | Speed |
|---|---|---|---|
| **Unit test** | one function, alone | `reports.balances` with a list of transactions | milliseconds |
| **Integration test** | several parts together | storage with a real (in memory) database; the API with storage and reports behind it | milliseconds to seconds |
| **End-to-end test** | the whole system, as a user | a browser that opens the web page, against a running server and database | seconds to minutes |

A healthy suite has many unit tests, fewer integration tests and a few end-to-end tests: the fast ones run constantly and point to the exact function, and the slow ones prove that the parts fit together. In `ledger`, the API tests are integration tests: each one goes through the endpoint, storage and SQLAlchemy down to the database.

Tests show that the cases you thought of work. They can't show that there are no bugs: an untested case can still fail. That is why each bug found in use should become a new test first, then be fixed: the test proves the fix, and the bug can't come back unnoticed.

## Continuous integration

Tests only protect `main` if they run. On a team, someone always forgets to run them before pushing, or runs them on a computer where an extra package hides a missing dependency. The answer is to have them run **automatically**, on a clean machine, for every change.

> 📘 **Definition: continuous integration (CI)**
>
> **Continuous integration** is the practice of merging small changes into `main` often, with a server that checks every change automatically: it installs the project from scratch on a clean machine, runs the linters and the test suite, and reports the result on the pull request. A change whose checks fail is not merged.

GitHub's CI service is **GitHub Actions**. A **workflow** is a YAML file in `.github/workflows/` of the repository, which says when to run and what to do:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: astral-sh/setup-uv@v10.3.3
      - name: Install the project and its dependencies
        run: uv sync --locked
      - name: Ruff rules
        run: uv run ruff check
      - name: Ruff format
        run: uv run ruff format --check
      - name: Tests
        run: uv run pytest
```

- **`on`** lists the events that start the workflow: every pull request, and every push to `main` (which is what a merge produces).
- **`jobs`** are independent sets of steps, each on a new virtual machine. This workflow has one job, `checks`, on the latest Ubuntu.
- **`steps`** run in order, and the job stops at the first one that fails. A step either **`uses`** an action, a ready-made step published by someone else (`actions/checkout` copies the repository onto the machine, `astral-sh/setup-uv` installs uv), or **`run`**s a command in the shell.
- `uv sync --locked` installs the exact versions of `uv.lock`, and fails if `uv.lock` doesn't match `pyproject.toml`. Python itself is installed by uv, in the version of `.python-version`.

Each step is a command you already run on your computer. The difference is that CI runs them on a machine where nothing is installed beforehand: if a dependency is missing from `pyproject.toml`, or a test passes only because of a file you have locally, CI finds it.

On the pull request, GitHub shows the result of the job as a **check**: a yellow dot while it runs, a green tick if every step passed, a red cross if one failed, with a link to the log of each step. A commit pushed to the branch starts the workflow again, so the check always shows the state of the latest commit.

## Requiring the checks on `main`

A red check is only a warning: GitHub still lets anyone merge the pull request. To make it a rule, protect `main` with a **ruleset** (*Settings → Rulesets*) that requires a pull request before merging and requires the status check `checks` (the name of the job) to pass. With it, GitHub blocks the merge button until the check is green, for everyone, including the owner of the repository.

On a team, the same ruleset also requires an approval from a teammate (see [The workflow for a small team](../05-git-collaboration/notes.md#the-workflow-for-a-small-team)). Together, the two rules mean that every change in `main` has been read by a second person and has passed the checks on a clean machine.

## Designing the tests of your module

For the module you own in the group project:

1. **Unit tests of every calculation** of the module, with small inputs calculated by hand, including the edge cases.
2. **A test for every business rule**, the accepted case and the refused one with `pytest.raises`.
3. **API tests of every endpoint**: the success status code and JSON, and each `4xx` the endpoint can return.
4. **The shared fixtures in `tests/conftest.py`**, agreed by the team, so that all the modules test against the same database in memory.

A new endpoint or rule comes with its tests in the same pull request, and CI runs them all, the module's and everybody else's.

## Common mistakes

- **Expected values copied from the output.** Then the test checks that the code does what it does, bugs included. Calculate them by hand.
- **Tests that depend on each other**, or on the order in which they run. Each test must prepare everything it needs; fixtures give each one fresh data.
- **Tests on the real database.** They fail or pass depending on what `ledger.db` contains, and they change it. Use a database in memory.
- **Testing only the usual case.** The edges and the errors are where bugs live.
- **Changing the test to make it pass.** First decide whether the code or the test is wrong.
- **A red check ignored "because it works on my computer".** CI runs on a clean machine: what fails there will fail for your teammates and in production.
- **A workflow that never runs.** It must be in `.github/workflows/`, with the extension `.yml`, and on the branch being pushed. Check the **Actions** tab after pushing.

> ✅ **Key takeaways**
>
> - An **automated test** runs a piece of code with fixed inputs and checks the result. A suite of them catches **regressions** in seconds, makes changes safe, and documents the rules.
> - With **pytest**, a test is a function `test_...` with `assert`, organised as **arrange, act, assert**, about one behaviour. Test the usual case, the edge cases and the errors, with expected values calculated by hand.
> - **Fixtures** prepare what tests need, fresh for each test; shared ones go in `conftest.py`. Code that uses a database is tested on a **database in memory** (`sqlite://`), and errors with **`pytest.raises`**.
> - An API is tested through HTTP with FastAPI's **`TestClient`**, replacing the session dependency with the test's one, and checking status codes and JSON.
> - **Continuous integration** runs the linters and the tests on a clean machine for every pull request. With **GitHub Actions**, it is a workflow in `.github/workflows/`; a **ruleset** on `main` that requires its check turns a red result into a blocked merge.

## Exercises

**1.** Write the expected result of `reports.net_by_month` for these transactions, and the test that checks it.

| date | account | category | amount |
|---|---|---|---|
| 2026-01-01 | checking | salary | 2000.00 |
| 2026-01-02 | checking | groceries | −65.15 |
| 2026-01-15 | checking | transfer | −300.00 |
| 2026-01-15 | savings | transfer | 300.00 |
| 2026-02-03 | checking | groceries | −40.00 |
| 2026-02-10 | checking | restaurants | −25.50 |

<details>
<summary>✅ <b>Solution</b></summary>

`net_by_month` adds every amount of each month, transfers included: in January, 2000.00 − 65.15 − 300.00 + 300.00 = 1934.85; in February, −40.00 − 25.50 = −65.50. The transfer adds nothing, because its two rows cancel out.

```python
def test_net_by_month(transactions):
    result = reports.net_by_month(transactions)

    assert result == {"2026-01": Decimal("1934.85"), "2026-02": Decimal("-65.50")}
```

</details>

**2.** A colleague writes this test. Find three problems.

```python
def test_storage():
    session = db.get_session()
    storage.import_csv(session, Path("data/transactions.csv"))
    balances = reports.balances(storage.load_transactions(session))
    assert balances["checking"] == storage.load_transactions(session)[0].amount
    storage.add_transaction(session, "credit", Decimal("5"), "x", "y", date.today())
```

<details>
<summary>✅ <b>Solution</b></summary>

- **It uses the real database**, `ledger.db`, through `db.get_session()`. The test adds 56 transactions to it each time it runs, and its result depends on what was there before. It should use the `session` fixture, a database in memory.
- **The expected value is not an expected value**: it compares the balance with the amount of the first transaction, which has nothing to do with it, and is calculated by the code itself. The expected balance should be a number calculated by hand, from a small CSV written by the test (with `tmp_path`).
- **It tests several things, and the last line is not a check**: `add_transaction` with the unknown account `credit` raises a `ValueError`, so the test fails with an error instead of checking the rule. That rule needs its own test, with `pytest.raises`. Splitting it into one test per behaviour also gives each one a name that says what it checks.

</details>

**3.** Your pull request has a red cross. The log of the job shows that the step **Install the project and its dependencies** failed, and the tests never ran. On your computer, `uv run pytest` passes. What is the most likely cause, and how do you fix it?

<details>
<summary>✅ <b>Solution</b></summary>

`uv sync --locked` fails when `uv.lock` doesn't match `pyproject.toml`: most likely you added or changed a dependency in `pyproject.toml` (by hand, or with `uv add`) and didn't commit the updated `uv.lock`. On your computer the package is installed, so the tests pass. Commit `uv.lock` (run `uv lock` first if it is out of date) and push: the workflow runs again on the new commit.

</details>

**4.** A team protects `main` with a ruleset that requires a pull request with one approval, but not the status check. What can still happen? And if it requires the check but not the pull request?

<details>
<summary>✅ <b>Solution</b></summary>

Without the check: a teammate can approve a pull request whose tests fail, and it can be merged with a red cross, so `main` can be broken. Without the pull request: anyone can push commits directly to `main`; CI runs after the push, so a red result only reports the problem once it is already in `main`, and no one has reviewed the change. The two rules protect against different things, and a team needs both.

</details>

[← Back to the session](index.md) [Next: Lab →](lab.md)
