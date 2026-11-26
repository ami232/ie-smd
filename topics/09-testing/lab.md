# Lab: tests and CI for the ledger

> ℹ️ **What you need first**
>
> Your repository `smd-app` with the web API of `ledger` (see [Lab: a web API for the ledger](../08-fastapi/lab.md)) merged into `main`: the modules `models`, `storage`, `reports`, `db`, `cli` and `api`, with the filters, the `404` and the `400` of the API. The concepts of [Testing and continuous integration](notes.md): pytest, arrange-act-assert, fixtures, a database in memory, `pytest.raises`, `TestClient` and GitHub Actions.

In this lab `ledger` gets a **test suite**, in a folder `tests/` next to `src/`, with tests for the three layers: the calculations of `reports`, storage on a database in memory, and the API through HTTP requests. Then GitHub runs Ruff and the tests on every pull request, and `main` only accepts changes that pass. The application code doesn't change at all. The tests are on this page, except for a few marked `# TODO` that you write yourself.

At the end, you mark the commit you want graded with the tag `cp3`, and submit the link to that commit on Blackboard, in the assignment **Checkpoint 3**, which covers this lab and the web API.

## 1. Start a branch and add pytest

In your repository, `~/smd/smd-app`, bring `main` up to date with GitHub, check that the working tree is clean, and create a branch called `feature/tests` for this lab.

<details>
<summary>ℹ️ <b>My API is not finished</b></summary>

The tests of this lab test the API, its filters and its errors, so finish [Lab: a web API for the ledger](../08-fastapi/lab.md) first, and merge it into `main`.

</details>

Add `pytest` to the project as a **development dependency**, like Ruff: it is needed to work on the code, not to run the application.

<details>
<summary>💡 <b>Hint</b></summary>

`uv add` with the option `--dev`, then the name of the package.

</details>

<details>
<summary>✅ <b>What you should see</b></summary>

In `pyproject.toml`, `pytest` in the group `dev` of `[dependency-groups]`, next to `ruff`, and not in `dependencies`.

</details>

## 2. Test the calculations

Create a folder `tests` at the root of the repository, next to `src`, and in it a file `test_reports.py` with this code:

```python
"""Tests of the calculations: transactions in, totals out."""

from datetime import date
from decimal import Decimal

import pytest

from ledger import reports
from ledger.models import Transaction


def make(day: str, account: str, category: str, amount: str) -> Transaction:
    """A transaction with the fields the reports use."""
    return Transaction(
        date=date.fromisoformat(day),
        account=account,
        description="test",
        category=category,
        amount=Decimal(amount),
    )


@pytest.fixture
def transactions() -> list[Transaction]:
    """Two months of a small ledger, with a transfer between the two accounts."""
    return [
        make("2026-01-01", "checking", "salary", "2000.00"),
        make("2026-01-02", "checking", "groceries", "-65.15"),
        make("2026-01-15", "checking", "transfer", "-300.00"),
        make("2026-01-15", "savings", "transfer", "300.00"),
        make("2026-02-03", "checking", "groceries", "-40.00"),
        make("2026-02-10", "checking", "restaurants", "-25.50"),
    ]


def test_balances(transactions):
    result = reports.balances(transactions)

    assert result == {"checking": Decimal("1569.35"), "savings": Decimal("300.00")}


def test_balances_of_no_transactions():
    assert reports.balances([]) == {}


def test_a_transfer_is_not_spending():
    transfer = make("2026-01-15", "checking", "transfer", "-300.00")

    assert not reports.is_spending(transfer)


def test_spending_by_category(transactions):
    # TODO 1: call reports.spending_by_category with the transactions of the
    #         fixture, and check the result: the spending of each category as a
    #         positive amount, transfers left out, largest first.
    pytest.fail("not written yet")


def test_net_by_month(transactions):
    result = reports.net_by_month(transactions)

    assert result == {"2026-01": Decimal("1934.85"), "2026-02": Decimal("-65.50")}
```

- `make` builds a `Transaction` in one line, so that the data of each test is easy to read.
- The fixture `transactions` is a small ledger of two months, with a transfer, whose results can be calculated by hand: `checking` is 2000.00 − 65.15 − 300.00 − 40.00 − 25.50 = 1569.35.
- Each test has the three parts, separated by empty lines: arrange (the fixture, or `make`), act (one call to `reports`), assert.
- `pytest.fail` makes a test fail on purpose: it marks the test you write in the TODO, so that it can't pass by accident while it is empty.

Run the tests:

```bash
uv run pytest
```

<details>
<summary>✅ <b>What you should see</b></summary>

```text
tests/test_reports.py ...F.                                              [100%]
```

Then the details of the failure, `Failed: not written yet`, and at the end `1 failed, 4 passed`. pytest found the file, ran its five tests and printed one character per test: four passed (`.`) and one failed (`F`), the TODO.

</details>

**TODO 1, in `tests/test_reports.py`:** write `test_spending_by_category`, as the comment describes, and delete the line `pytest.fail(...)`.

<details>
<summary>💡 <b>Hints</b></summary>

- Follow `test_balances`: one line to act, `result = reports.spending_by_category(transactions)`, an empty line, and the asserts.
- Calculate the expected spending by hand from the fixture: which transactions are spending, and what is the total of each category? The amounts are positive.
- Comparing two dictionaries with `==` ignores the order. To check that the largest comes first, compare the keys too: `list(result)` is the list of categories in order.

</details>

Run the tests again.

<details>
<summary>✅ <b>What you should see</b></summary>

`5 passed`. If your test fails, read the comparison that pytest prints: it shows what `spending_by_category` returned and what your test expected. Check your calculation by hand before changing anything else.

</details>

## 3. A test catches a bug

Tests are worth what they catch. In `src/ledger/reports.py`, change the last line of `is_spending` so that it forgets the transfers:

```python
    return t.amount < 0
```

Run the tests.

<details>
<summary>✅ <b>What you should see</b></summary>

```text
tests/test_reports.py ..FF.                                              [100%]
```

Two failures. `test_a_transfer_is_not_spending` names the broken rule. `test_spending_by_category` shows the consequence, if your TODO checks the categories: the result has one category more than expected, `{'transfer': Decimal('300.00')}`. Without the tests, the spending report would now be 300.00 too high, and nothing would warn you.

</details>

Undo the change, and check that the tests pass again.

<details>
<summary>💡 <b>Hint</b></summary>

The original line is `return t.amount < 0 and t.category != TRANSFER`. Git can also undo the changes to a file that you haven't committed.

</details>

## 4. Test storage on a database in memory

The tests of storage need a database, a new and empty one for each test. In `tests`, create a file `conftest.py`, where pytest finds the fixtures shared by all the test files:

```python
"""Fixtures shared by all the tests."""

import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import Session
from sqlalchemy.pool import StaticPool

from ledger.db import Base


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

- `sqlite://`, with no file name, is a SQLite database in memory: it disappears at the end of the test. `ledger.db` is never touched.
- `create_all` creates in it the tables of `db.py`.
- `yield` gives the session to the test, and closes it when the test ends. `StaticPool` and `check_same_thread` keep the same connection for the whole test, also when the API uses it (see [Testing code that uses a database](notes.md#testing-code-that-uses-a-database)).

Then create `tests/test_storage.py`:

```python
"""Tests of storage, on a database in memory."""

from datetime import date
from decimal import Decimal

import pytest

from ledger import storage
from ledger.db import Account

DAY = date(2026, 3, 31)


@pytest.fixture
def db(session):
    """The empty database of the fixture session, plus one account, checking."""
    session.add(Account(name="checking"))
    session.commit()
    return session


def test_add_and_load_a_transaction(db):
    storage.add_transaction(db, "checking", Decimal("-12.50"), "food", "Lunch", DAY)

    [loaded] = storage.load_transactions(db)

    assert loaded.date == DAY
    assert loaded.account == "checking"
    assert loaded.category == "food"
    assert loaded.amount == Decimal("-12.50")


def test_import_csv(session, tmp_path):
    csv_file = tmp_path / "transactions.csv"
    csv_file.write_text(
        "date,account,description,category,amount\n"
        "2026-01-01,checking,Salary,salary,2000.00\n"
        "2026-01-02,checking,Carrefour,groceries,-65.15\n"
        "2026-01-15,savings,Transfer from checking,transfer,300.00\n"
    )

    count = storage.import_csv(session, csv_file)

    assert count == 3
    amounts = [t.amount for t in storage.load_transactions(session)]
    assert amounts == [Decimal("2000.00"), Decimal("-65.15"), Decimal("300.00")]


def test_an_unknown_account_is_refused(db):
    # TODO 2: check that add_transaction raises a ValueError for an account that
    #         doesn't exist, and that nothing was saved.
    pytest.fail("not written yet")


def test_a_fraction_of_a_cent_is_refused(db):
    # TODO 2: the same for an amount with a fraction of a cent, such as -1.005.
    pytest.fail("not written yet")
```

- The fixture `db` uses the fixture `session` and adds one account to it: fixtures can build on each other.
- `test_add_and_load_a_transaction` sends a transaction to the database and reads it back: the amount goes in as cents and comes back as the same `Decimal`.
- `[loaded] = ...` unpacks a list that must have exactly one element; with zero or two, the test fails.
- `tmp_path` is a fixture of pytest: a new, empty folder for each test, where `test_import_csv` writes a small CSV file.

Run the tests.

<details>
<summary>✅ <b>What you should see</b></summary>

```text
tests/test_reports.py .....                                              [ 55%]
tests/test_storage.py ..FF                                               [100%]
```

And `2 failed, 7 passed`: the two tests of the TODO.

</details>

**TODO 2, in `tests/test_storage.py`:** write the two tests, as the comments describe: `add_transaction` must raise a `ValueError` for an account that doesn't exist and for a fraction of a cent, and save nothing.

<details>
<summary>💡 <b>Hints</b></summary>

- `with pytest.raises(ValueError):` followed by the call to `storage.add_transaction`, indented inside the `with` (see [Testing errors](notes.md#testing-errors)).
- `match="..."` checks part of the message too: use a few words of the message that `add_transaction` raises.
- After the `with` block, check that nothing was saved: `storage.load_transactions(db)` must be an empty list.
- The account `checking` exists in `db`: use another name for the unknown one.

</details>

<details>
<summary>✅ <b>What you should see</b></summary>

`9 passed`.

</details>

## 5. Test the API

The API is tested through HTTP requests, with FastAPI's `TestClient`, on the database in memory. Create `tests/test_api.py`:

```python
"""Tests of the web API, through HTTP requests, on a database in memory."""

from datetime import date
from decimal import Decimal

import pytest
from fastapi.testclient import TestClient

from ledger import api, storage
from ledger.db import Account

LUNCH = {
    "date": "2026-03-31",
    "account": "checking",
    "description": "Lunch with a client",
    "category": "restaurants",
    "amount": "-12.50",
}


@pytest.fixture
def client(session):
    """A client of the API, on the database of the fixture session, with two
    accounts and one transaction in each."""
    session.add_all([Account(name="checking"), Account(name="savings")])
    session.commit()
    day = date(2026, 1, 10)
    storage.add_transaction(session, "checking", Decimal("-39.90"), "sport", "Gym", day)
    storage.add_transaction(session, "savings", Decimal("1.60"), "interest", "Pay", day)

    api.app.dependency_overrides[api.get_session] = lambda: session
    yield TestClient(api.app)
    api.app.dependency_overrides.clear()


def test_list_accounts(client):
    response = client.get("/api/accounts")

    assert response.status_code == 200
    assert response.json() == [
        {"name": "checking", "balance": "-39.90"},
        {"name": "savings", "balance": "1.60"},
    ]


def test_get_a_transaction(client):
    response = client.get("/api/transactions/1")

    assert response.status_code == 200
    assert response.json()["description"] == "Gym"
    assert response.json()["amount"] == "-39.90"


def test_a_missing_transaction_is_404(client):
    response = client.get("/api/transactions/999")

    assert response.status_code == 404


def test_filter_by_category(client):
    response = client.get("/api/transactions", params={"category": "interest"})

    assert response.status_code == 200
    assert [t["id"] for t in response.json()] == [2]


def test_create_a_transaction(client):
    response = client.post("/api/transactions", json=LUNCH)

    assert response.status_code == 201
    assert response.json() == LUNCH | {"id": 3}


def test_an_unknown_account_is_400(client):
    response = client.post("/api/transactions", json=LUNCH | {"account": "credit"})

    assert response.status_code == 400
    assert response.json() == {"detail": "there is no account called 'credit'"}


def test_a_date_that_is_not_a_date_is_422(client):
    response = client.post("/api/transactions", json=LUNCH | {"date": "31/03/2026"})

    assert response.status_code == 422


def test_delete_a_transaction(client):
    # TODO 3: delete transaction 1 and check the status code; then ask for it
    #         and check that it no longer exists.
    pytest.fail("not written yet")
```

- The fixture `client` fills the database of the test with two accounts and two transactions, whose `id`s are `1` and `2`. Then it replaces the dependency `get_session` of the API with the session of the test, so every endpoint uses the database in memory, and gives the test a `TestClient`. After the test, `clear()` removes the replacement.
- `client.get(...)` and `client.post(..., json=...)` send requests without starting a server. `response.json()` is the body as Python data: the amounts are strings, as a real client receives them.
- `LUNCH | {"account": "credit"}` is a copy of the dictionary `LUNCH` with one key changed.
- The tests check the API's contract: the status code of each situation and the JSON.

Run the tests.

<details>
<summary>✅ <b>What you should see</b></summary>

```text
tests/test_api.py .......F                                               [ 47%]
tests/test_reports.py .....                                              [ 76%]
tests/test_storage.py ....                                               [100%]
```

And `1 failed, 16 passed`. If the `404`, `400` or filter tests fail, the API is missing the code of its TODOs: the tests found it.

</details>

**TODO 3, in `tests/test_api.py`:** write `test_delete_a_transaction`, as the comment describes.

<details>
<summary>💡 <b>Hints</b></summary>

- `client.delete(url)` sends a `DELETE` request.
- Which status code does the API answer to a `DELETE` that worked, and which one to a `GET` of a transaction that doesn't exist?
- One test can send two requests: the act is the `DELETE`, and the `GET` is part of the check.

</details>

Then run the tests, and check Ruff too.

<details>
<summary>💡 <b>Hint</b></summary>

`uv run ruff check` and `uv run ruff format --check`. If `format --check` lists a file, `uv run ruff format` fixes it.

</details>

<details>
<summary>✅ <b>What you should see</b></summary>

`17 passed`, and Ruff passes. No `ledger.db` was created or changed by the tests: they only used databases in memory.

</details>

## 6. Continuous integration

Create the folders `.github/workflows` at the root of the repository (the first one starts with a dot), and in it a file `ci.yml`:

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

Indentation matters in YAML: use spaces, exactly as above. The workflow runs on every pull request and on every push to `main`: on a clean machine, it installs uv and the exact dependencies of `uv.lock`, and runs Ruff and the tests, the same commands you run (see [Continuous integration](notes.md#continuous-integration)).

Commit your work on `feature/tests` and push the branch. The commit has `pyproject.toml`, `uv.lock`, the folder `tests` and `.github/workflows/ci.yml`; nothing in `src` has changed. Then, on GitHub, open a pull request from `feature/tests` to `main`, and don't merge it yet.

<details>
<summary>✅ <b>What you should see</b></summary>

At the bottom of the pull request, the check of the job `checks` of the workflow `CI`: first a yellow dot while it runs, then a green tick after a few seconds. Click **Details** to see the log: one section per step, and in **Tests**, the same `17 passed` as on your computer. The **Actions** tab of the repository lists every run of the workflow.

</details>

## 7. Protect `main`

A red check is only a warning until `main` requires it. On GitHub, in your repository, go to **Settings → Rulesets** (in the section *Code, planning, and automation*), click **New ruleset → New branch ruleset**, and create a ruleset with:

- **Ruleset name**: `Protect main`;
- **Enforcement status**: *Active*;
- **Target branches**: **Add target → Include default branch**;
- **Require a pull request before merging**, with **Required approvals** at `0` (in your own repository, there is nobody else to approve);
- **Require status checks to pass**, and in **Add checks**, the check `checks`, the name of the job of the workflow.

Save it with **Create**. From now on, nobody can push to `main` directly, you included: every change goes through a pull request whose check is green.

## 8. Red and green

Check that the protection works. On `feature/tests`, change one expected value of a test so that it is wrong, for example the balance of `checking` in `test_balances` to `1569.36`, commit the change and push it.

<details>
<summary>✅ <b>What you should see</b></summary>

The check of the pull request runs again on the new commit, and ends with a red cross. In **Details**, the step **Tests** failed, with `FAILED tests/test_reports.py::test_balances` and `1 failed, 16 passed`. The merge button of the pull request is blocked: the ruleset requires the check to pass.

</details>

Undo the change in another commit, and push it.

<details>
<summary>✅ <b>What you should see</b></summary>

The check runs again and turns green, and the pull request can be merged.

</details>

In the description of the pull request, say what the tests cover (which functions and which endpoints), and how the CI workflow works. Merge it with **Create a merge commit**. Then bring your local `main` up to date and delete the branch.

<details>
<summary>✅ <b>What you should see</b></summary>

The merge is a push to `main`, so the workflow runs once more, now on `main`: in the **Actions** tab, a run for the pull request and one for `main`, both green.

</details>

## 9. Tag checkpoint 3

Mark the last commit of `main`, the merge of this pull request, with the tag `cp3`, and push the tag. Then submit the link to that commit on Blackboard, in the assignment **Checkpoint 3**.

<details>
<summary>💡 <b>Hint</b></summary>

The same steps as for the tag `cp2` (see [Tags: a name for one commit](../05-git-collaboration/notes.md#tags-a-name-for-one-commit)): create the tag on the commit, then push it to GitHub by its name. The link to the commit is on the page `https://github.com/<your-username>/smd-app/tags`: click the commit id under `cp3` and copy the address. If you change something afterwards, move the tag to the new commit and submit the new link.

</details>

## Checkpoint 3

Checkpoint 3 covers the web API and its tests. What is checked is the commit tagged `cp3`:

- [ ] **API**: `api.py` with the seven endpoints under `/api` and the Pydantic models `NewTransaction`, `TransactionOut` and `AccountOut`; the filters `account` and `category`, the `404` for a transaction that doesn't exist and the `400` for a transaction that breaks a business rule (see [Lab: a web API for the ledger](../08-fastapi/lab.md)); no SQL and no calculations in `api.py`;
- [ ] **Tests**: `tests/test_reports.py`, `tests/test_storage.py` and `tests/test_api.py`, with your tests of the TODOs, and `tests/conftest.py` with the database in memory;
- [ ] `uv run pytest` and `uv run ruff check` passing from a clean clone, with `pytest` as a development dependency and no `TODO` and no `pytest.fail` left;
- [ ] **CI**: `.github/workflows/ci.yml` running Ruff and the tests, with a green run on the commit tagged `cp3`;
- [ ] **`main` protected** by a ruleset that requires a pull request and the check `checks`;
- [ ] the merged pull requests from `feature/api` and `feature/tests`, each with a description and merged with a merge commit; the second one shows a red run and a green one;
- [ ] the tag `cp3` pushed to GitHub, and the link to its commit submitted on Blackboard.

[← Previous: Testing and continuous integration](notes.md) [Back to the session ↑](index.md)
