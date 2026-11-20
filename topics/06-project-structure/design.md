# Software design and project structure

> ℹ️ **What you need first**
>
> The `ledger` project in your repository `smd-app`: one file, `src/ledger/__init__.py`, that reads the CSV file, computes balances and spending, and prints them. Functions, dictionaries and `import` in Python, and [Python projects with uv](../03-python-uv/python.md), especially the `[project.scripts]` entry of `pyproject.toml`.

## Why structure matters

`ledger` is about 50 lines in one file, and it works. Now imagine the next requests. *"Keep the transactions in a database instead of a CSV file."* *"Offer the same balances through a web API, for a dashboard."* *"Add a report of spending per month."* For each one, ask: which lines of `__init__.py` change?

The honest answer is "it depends, read the whole file". `read_transactions` reads the CSV *and* turns amounts into `Decimal`. `main` reads the command line, *and* filters out transfers to find the spending, *and* draws the table. The rule "spending means a negative amount that is not a transfer" lives inside the code that prints. To serve the spending through an API, you would copy that rule, and the day it changes (refunds, say), someone updates one copy and forgets the other.

In a small script that is fine. In software that a team keeps changing for years, it is the main source of bugs and slowness: every change touches code that does several things, so every change risks breaking something unrelated. **Software design** is the set of decisions about how to split a program into parts so that each likely change touches as few parts as possible. This page covers the decisions that matter for an application like `ledger`: which parts to have, how they depend on each other, how to represent the data, and how to reorganize code that already works without breaking it.

## Separation of concerns

Look at what `ledger` does and group it by *reason to change*. There are four groups:

1. **What the data is.** A transaction has a date, an account, a description, a category and an amount. This changes only when the business changes (a transaction gets a currency, say).
2. **Where the data is stored.** Today, a CSV file. Tomorrow, a database. This changes when the infrastructure changes.
3. **What we compute.** Balances per account, spending per category, the definition of spending. This changes when the business questions change.
4. **How the user talks to the program.** Today, a command line with a `rich` table. Tomorrow, a web API. This changes when the interface changes.

> 📘 **Definition: concern, module, separation of concerns**
>
> A **concern** is one area of responsibility of a program, with its own reasons to change: the data, storage, calculations, the user interface.
>
> **Separation of concerns** is the design principle of putting each concern in its own part of the code, so that a change in one concern touches only that part. In Python, the natural part is the **module**: one `.py` file with the functions and classes of one concern.

For `ledger`, separation of concerns gives four modules:

| Module | Concern | Knows about | Doesn't know about |
|---|---|---|---|
| `models.py` | what a transaction is | nothing else in the project | files, databases, printing |
| `storage.py` | reading transactions from where they are kept | `models` | the calculations, printing |
| `reports.py` | the calculations: balances, spending, months | `models` | where the data came from, how it is shown |
| `cli.py` | the command line: arguments and output | `storage`, `reports` | how the calculations work |

### The direction of dependencies

A module *depends on* another when it imports it. The table above implies a direction:

```text
          cli.py
         /      \
        ▼        ▼
 storage.py   reports.py
         \      /
          ▼    ▼
        models.py
```

The arrows go from the outside (the parts that talk to the world: the user, the disk) towards the inside (the data and the business rules), never the other way. `reports.py` doesn't import `storage` or `cli`. That is the property that pays off: when the CSV file becomes a database, only `storage.py` changes; when the command line gets a web API next to it, the API calls the same `reports` functions, and the definition of spending exists once.

### The single responsibility principle

The same idea, applied to each piece of code, has a name.

> 📘 **Definition: single responsibility principle**
>
> *A module should have one, and only one, reason to change.* A **responsibility** is a reason to change: a part of the program that changes when one kind of requirement changes (the data, the storage, the business rules, the interface). The principle was formulated by Robert C. Martin, and it is the *S* of **SOLID**, five principles of object-oriented design.

It applies to functions and classes as much as to modules. The old `main` of `ledger` has four reasons to change: the command-line arguments, the rule of what counts as spending, how totals are computed, and how the table is drawn. The new `is_spending` has one: the rule of what counts as spending. When that rule changes (say, refunds must reduce spending), there is exactly one place to edit, and nothing else can break.

A rule of thumb to check a design: for each module or function, finish the sentence "this changes when…". If the sentence needs an "and" ("when the file format changes *and* when the definition of spending changes"), it mixes two responsibilities, and it should be split.

### How much structure

Separation of concerns can be overdone. A program split into thirty files of five lines each, with layers that only pass calls through, is as hard to change as one big file, because every change now visits many places. The goal is one module per concern that really changes on its own, not as many modules as possible. Four modules for `ledger` is enough; a large application has the same four layers, each one a package with several modules.

## Modules and packages in Python

> 📘 **Definition: module and package**
>
> A **module** is a `.py` file. Its name, without `.py`, is what you import: the file `reports.py` is the module `reports`.
>
> A **package** is a folder of modules with an `__init__.py` file. The folder `src/ledger/` is the package `ledger`, and `src/ledger/reports.py` is the module `ledger.reports`.

Inside the package, modules import each other by their full name, starting at the package. These are **absolute imports**, and they work the same from any module and from any folder you run the program in:

```python
from ledger.models import Transaction    # one name from a module
from ledger import reports, storage      # whole modules; then reports.balances(...)
```

`__init__.py` runs the first time anything from the package is imported. It can stay almost empty: a docstring that says what the package is. Before, all of `ledger` lived in it; now each concern has its own module.

### The `src` layout and the entry point

`ledger` uses the **src layout**: the package is in `src/ledger/`, not in the project's top folder. uv installs the package into `.venv` in *editable* mode (the environment points to your `src` folder, so your edits are used at once), and the code always runs as the installed package `ledger`. Without the `src` folder, Python could also import `ledger` straight from the current folder, and code that works on your laptop could fail once installed somewhere else.

The command `ledger` is declared in `pyproject.toml`:

```toml
[project.scripts]
ledger = "ledger.cli:main"
```

This means: the command `ledger` imports the module `ledger.cli` and calls its function `main`. When `main` moves from `__init__.py` to `cli.py`, this line must change from `"ledger:main"` to `"ledger.cli:main"`. `uv run` notices that `pyproject.toml` changed and reinstalls the project before running it.

### Circular imports

If `reports.py` imports `cli.py` and `cli.py` imports `reports.py`, Python starts loading one, has to load the other halfway through, and fails with `ImportError: cannot import name ... (most likely due to a circular import)`. With dependencies in one direction, from the outside to the inside, this can't happen. A circular import is almost always a sign that two modules share a concern, or that a module is reaching outwards.

## Data as classes: dataclasses

In the one-file `ledger`, a transaction is a dictionary: `{"date": "2026-01-01", "account": "checking", ..., "amount": Decimal("-950.00")}`. Dictionaries are flexible, and that is their problem as data models. Nothing says which keys a transaction has. A typo, `t["amout"]`, is only found when that line runs. The editor can't complete `t["...` for you, and the date is a string that every piece of code has to remember to parse.

A **dataclass** declares the shape of the data once. You write the fields and their types, and Python generates the rest: a constructor, a readable `repr`, and comparison with `==`.

```python
from dataclasses import dataclass
from datetime import date
from decimal import Decimal


@dataclass(frozen=True)
class Transaction:
    date: date
    account: str
    description: str
    category: str
    amount: Decimal


rent = Transaction(date(2026, 1, 1), "checking", "Rent", "housing", Decimal("-950.00"))
print(rent)
```

```text
Transaction(date=datetime.date(2026, 1, 1), account='checking', description='Rent', category='housing', amount=Decimal('-950.00'))
```

```python
print(rent.amount, rent.date.month)
```

```text
-950.00 1
```

Each field is an attribute, `rent.amount`, so the editor completes it and Ruff and Pylance flag a misspelled one before the code runs. The date is a real `date`, so `rent.date.month` works without parsing a string.

The annotations (`amount: Decimal`) are **type hints**. Python doesn't check them when the program runs: `Transaction(..., amount="lots")` is accepted. Tools do check them, and the editor uses them to tell you what each value is. They also document the code: a reader knows what `balances` returns from its signature, `def balances(transactions: Iterable[Transaction]) -> dict[str, Decimal]`, without reading its body.

`frozen=True` makes the objects **immutable**: once created, their fields can't change. A transaction that happened doesn't change, and immutable objects can be passed to any function without the risk that it modifies them behind your back:

```python
from dataclasses import FrozenInstanceError

try:
    rent.amount = Decimal("0")
except FrozenInstanceError as error:
    print("FrozenInstanceError:", error)
```

```text
FrozenInstanceError: cannot assign to field 'amount'
```

## Pure functions at the centre

Look at the functions of `reports.py` in [the lab](lab.md#3-write-the-calculations): they receive transactions and return dictionaries. They don't open files, print, read the clock or ask the user anything.

> 📘 **Definition: pure function and side effect**
>
> A **side effect** is anything a function does besides returning a value: reading or writing a file or a database, printing, reading the clock, changing a global variable.
>
> A **pure function** has no side effects, and its result depends only on its arguments: called twice with the same arguments, it returns the same result.

Pure functions are the easiest code to understand and to check. To know whether `spending_by_category` is right, build a few transactions by hand and look at the result. No file, no database, no terminal:

```python
from collections import defaultdict


def totals_by(transactions, key):
    totals = defaultdict(Decimal)
    for t in transactions:
        totals[key(t)] += t.amount
    return dict(totals)


def is_spending(t):
    return t.amount < 0 and t.category != "transfer"


def spending_by_category(transactions):
    spending = [t for t in transactions if is_spending(t)]
    totals = totals_by(spending, key=lambda t: t.category)
    return {c: -total for c, total in sorted(totals.items(), key=lambda item: item[1])}


sample = [
    Transaction(date(2026, 1, 1), "checking", "Rent", "housing", Decimal("-950.00")),
    Transaction(date(2026, 1, 2), "checking", "Carrefour", "groceries", Decimal("-65.15")),
    Transaction(date(2026, 1, 3), "checking", "To savings", "transfer", Decimal("-300.00")),
    Transaction(date(2026, 1, 4), "checking", "Salary", "income", Decimal("2600.00")),
]
print(spending_by_category(sample))
```

```text
{'housing': Decimal('950.00'), 'groceries': Decimal('65.15')}
```

The transfer and the salary are not spending, and the result says so. This is the shape of a design that is easy to test: **side effects at the edges** (`storage` reads, `cli` prints) and **pure logic at the centre** (`reports`). Program bugs are mostly in the logic, and the logic is the part you can check with four lines of data.

`totals_by` takes `key`, a *function* that says how to group, instead of a column name: `key=lambda t: t.category`. A `lambda` is a short function without a name; `lambda t: t.category` is the same as `def f(t): return t.category`. Passing a function makes `totals_by` work for any grouping, including ones that are not a field, like the month: `key=lambda t: t.date.strftime("%Y-%m")`.

## Refactoring

Moving the code of `ledger` into four modules doesn't add a feature. The program must do exactly what it did before.

> 📘 **Definition: refactoring**
>
> **Refactoring** is changing the structure of code without changing its behaviour: the same inputs give the same outputs, before and after.

Refactoring is how real code gets a good design: almost nobody gets the structure right before the program exists. The risk is breaking something while moving it, so a refactoring is done with a safety net:

1. **Record the behaviour before you start.** Save the output of each command to a file. (With automated tests, the tests are the record; that is the next step for any serious project.)
2. **Work on a branch,** so `main` keeps working while you refactor.
3. **Move code, don't rewrite it.** Change the structure and the names, keep the logic. If you find a bug on the way, note it and fix it in a separate commit, after the refactoring, so that each commit does one thing.
4. **Compare the behaviour after.** Run the same commands and compare the output with `diff`. No differences means the refactoring kept the behaviour, at least for these inputs.
5. **Review and merge** with a pull request.

## Code smells

A **code smell** is a pattern in the code that is not a bug but usually makes the next change harder. Naming them helps a team agree on what to improve. These are the ones the one-file `ledger` had:

| Smell | In the old `ledger` | The fix |
|---|---|---|
| **A function that does several things** | `main` reads arguments, filters, computes and draws | one function per job: parse, compute, print |
| **Magic value** | the string `"transfer"` inside a filter | a named constant, `TRANSFER = "transfer"`, defined once |
| **Duplicated knowledge** | the rule "spending is negative and not a transfer" would be copied by any new interface | one function, `is_spending`, used everywhere |
| **Primitive data** | a transaction is a dict of strings | a `Transaction` dataclass with real types |
| **Vague names** | `totals`, `t`, `data` used for different things | names that say what a value is (`balances`, `spending`) |

Some smells can be detected automatically. A linter like Ruff does not judge the design, but it catches the patterns that often hide bugs, and a team chooses which rules to apply in `pyproject.toml`:

```toml
[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM"]
```

| Rules | Name | Example of what they find |
|---|---|---|
| `E`, `F` | pycodestyle, Pyflakes | lines too long, undefined names, unused imports (the default) |
| `I` | isort | imports not grouped and sorted (standard library, third party, the project) |
| `B` | flake8-bugbear | likely bugs, such as a mutable default argument `def f(items=[])` |
| `UP` | pyupgrade | old syntax that the project's Python version can write more simply |
| `SIM` | flake8-simplify | code that can be written more simply, such as `if x: return True else: return False` |

Most of these problems Ruff can fix itself with `uv run ruff check --fix`. Read what it changed before committing, as you would read a teammate's change.

## Designing your own module

Your application, and your module of the group product, will go through the same steps as `ledger`, so it pays to design its data before writing code. Write down, in the README of your repository:

1. **The entities**: the kinds of things your module records (`Subscription`, `Invoice`, `Loan`…). One or two is enough.
2. **Their fields, with types**: `name: str`, `monthly_price: Decimal`, `start: date`. Money is always `Decimal`.
3. **The relationships**: which entity belongs to which (a payment belongs to an invoice; an invoice belongs to a client).
4. **The calculations**: what your module computes from the data, as pure functions (`monthly_cost(subscriptions) -> Decimal`, `overdue(invoices, today) -> list[Invoice]`).
5. **The commands**: what the user asks for, as `ledger balance` or `ledger spending` do.

The four modules of `ledger` are a template: your `models.py` holds the entities, your `reports.py` the calculations, and the other two adapt with few changes.

## Common mistakes

- **Moving code and changing it at the same time.** If the output changes, you can't tell whether the move or the edit broke it. Move first, compare, commit; then change.
- **Forgetting to update the entry point.** After moving `main` to `cli.py`, `uv run ledger` fails with `AttributeError: module 'ledger' has no attribute 'main'` until `pyproject.toml` says `ledger.cli:main`.
- **Imports from the current folder.** `import models` (instead of `from ledger import models`) may work when you run a file directly from `src/ledger/`, and fails everywhere else.
- **Printing inside `reports.py`.** A calculation that prints can't be reused by an API or checked without reading the terminal. Return the value; let `cli.py` print it.
- **One module per function.** Splitting by function rather than by concern gives many tiny files that always change together.

> ✅ **Key takeaways**
>
> - Design means splitting a program so that each likely change touches as few parts as possible. The **single responsibility principle**: each module, class or function has one reason to change. For an application: data, storage, calculations, interface.
> - In Python each concern is a **module** (`.py` file), and a folder of modules with `__init__.py` is a **package**. Import with absolute names: `from ledger.models import Transaction`.
> - Dependencies point **inwards**: the interface and storage use the models and the calculations, never the other way round. This avoids circular imports and lets you replace the CSV file or the command line without touching the calculations.
> - A **dataclass** declares the shape of the data with typed fields; with `frozen=True`, the objects can't be changed after they are created.
> - Keep side effects at the edges and **pure functions** at the centre: they are easy to read, reuse and check.
> - **Refactoring** changes the structure, not the behaviour: record the output before, move code without rewriting it, compare after, and merge through a pull request.

## Exercises

**1.** For each change, say which module or modules of the new `ledger` (`models`, `storage`, `reports`, `cli`) must change: (a) transactions now come from a JSON file; (b) refunds, positive amounts in a spending category, must reduce that category's spending; (c) the balance must be shown in a `rich` table too; (d) each transaction gets a `currency` field.

<details>
<summary>✅ <b>Solution</b></summary>

(a) `storage` only: a new function that reads JSON and returns the same `Transaction` objects. (b) `reports` only: the rule for spending changes in `is_spending` (and `spending_by_category`), and every interface picks up the change. (c) `cli` only: printing is its job. (d) `models` (the new field), `storage` (read it from the file) and probably `reports` (adding amounts in different currencies needs a decision); `cli` only if the currency is shown. A change to the data model is the one that travels furthest, which is why it is worth designing the data first.

</details>

**2.** A teammate adds this to `reports.py`. What is wrong with it from a design point of view, and how would you change it?

```python
def print_balances(path):
    transactions = storage.read_csv(path)
    for account, total in balances(transactions).items():
        print(f"{account:<10} {total:>10}")
```

<details>
<summary>✅ <b>Solution</b></summary>

It mixes three concerns in the module of calculations: reading (storage), computing and printing (the interface). `reports.py` now depends on `storage`, and the function can't be reused by an API or checked without a file and a terminal. The calculation already exists (`balances`); reading belongs in `cli.main`, which calls `storage.read_csv`, and printing in `cli.print_totals`. The function should be deleted.

</details>

**3.** `cli.py` has `import reports` and `reports.py` has `from cli import print_totals`. What happens when you run `uv run ledger balance`, and why?

<details>
<summary>✅ <b>Solution</b></summary>

Two problems. First, `import reports` and `from cli import ...` are imports from the current folder, not from the package: with the src layout they fail with `ModuleNotFoundError: No module named 'reports'`; they must be `from ledger import reports` and `from ledger.cli import print_totals`. Second, with those fixed, the two modules import each other: a circular import, which fails with `ImportError: cannot import name 'print_totals' from partially initialized module 'ledger.cli'`. The real fix is the design: `reports` must not depend on `cli`. It returns values and `cli` prints them.

</details>

**4.** Design the data for a *Subscriptions* module: the user records their subscriptions (Netflix, the gym…), each with a monthly price and a start date, and can cancel one. Write the dataclass, and the signature and docstring of two pure functions: the total monthly cost of the active subscriptions, and the subscriptions that cost more than a given amount.

<details>
<summary>✅ <b>Solution</b></summary>

```python
@dataclass(frozen=True)
class Subscription:
    name: str
    monthly_price: Decimal
    start: date
    cancelled: date | None = None   # None while it is active


def monthly_cost(subscriptions: Iterable[Subscription]) -> Decimal:
    """Total monthly price of the subscriptions that are not cancelled."""


def more_expensive_than(
    subscriptions: Iterable[Subscription], limit: Decimal
) -> list[Subscription]:
    """Active subscriptions whose monthly price is above `limit`, most expensive first."""
```

The price is a `Decimal`, not a `float`. Cancelling is a date, not a boolean, so the data says *when* it was cancelled; `date | None` means "a date or nothing". Neither function reads files or prints, so both can be checked with a list of three subscriptions written by hand.

</details>

[← Back to the session](index.md) [Next: Lab →](lab.md)
