# Lab: from one file to a package

> ℹ️ **What you need first**
>
> Your repository `smd-app` on GitHub (see [Lab: your app's repository](../05-git-collaboration/lab.md)), cloned in `~/smd/smd-app`, where `uv run ledger balance` works. The ideas of [Software design and project structure](design.md): concerns, modules, the direction of dependencies, dataclasses and refactoring.

In this lab you reorganize `ledger` from one file into a package of four modules, each with one responsibility: `models` (what a transaction is), `storage` (reading the CSV file), `reports` (the calculations) and `cli` (the command line). You don't write the modules from scratch: this page gives them to you, complete except for a few places marked `# TODO`, where you write the code yourself. This is a **refactoring**, so when you finish every command prints exactly what it printed before, and you prove it with `diff`. Then you add a command, configure Ruff and merge the work through a pull request.

## 1. Start a branch and record the behaviour

Go to your repository, make sure `main` is up to date and clean, and create the branch for the refactoring:

```bash
cd ~/smd/smd-app
git switch main
git pull
git status
git switch -c refactor/package
```

<details>
<summary>💡 <b>My repository is not ready</b></summary>

If you don't have the repository yet, create it as in [Create the repository on GitHub and clone it](../05-git-collaboration/lab.md#3-create-the-repository-on-github-and-clone-it), then copy into it the files of the reference version at the tag **`ledger-script`** of [github.com/ami232/ie-smd-ledger](https://github.com/ami232/ie-smd-ledger) (switch to the tag, **Code → Download ZIP**): `pyproject.toml`, `uv.lock`, `.python-version`, `src/` and `data/`. Commit them on `main`, push, and create the branch.

</details>

Before changing any code, **record what the program does now**, in files outside the repository:

```bash
mkdir -p ~/smd/before
uv run ledger balance > ~/smd/before/balance.txt
uv run ledger spending > ~/smd/before/spending.txt
cat ~/smd/before/balance.txt
```

<details>
<summary>✅ <b>What you should see</b></summary>

`cat` prints the two balances, `3349.10` and `906.00`. `>` sends a command's output to a file instead of the terminal, so `ledger spending` printed nothing: its table is in `spending.txt`.

</details>

## 2. Create the modules

In VS Code (`code .`), create four new files in `src/ledger/`: in the Explorer, right-click the folder `src/ledger`, choose **New File…** and type the name. Copy into each one the code below. Two of them are complete; the other two have places marked `# TODO` that you write in the next steps.

| Module | Contents | Imports from `ledger` | Your work |
|---|---|---|---|
| `models.py` | the `Transaction` dataclass: five typed fields, `frozen=True` | nothing | copy it |
| `storage.py` | `read_csv(path)`: one `Transaction` per row, with the date as a `date` and the amount as a `Decimal` | `models` | copy it |
| `reports.py` | `totals_by`, `is_spending`, `balances`, `spending_by_category`, `net_by_month` | `models` | copy it, then TODO 1, 2 and 3 |
| `cli.py` | `main`: reads the arguments, calls `storage` and `reports`, and prints | `storage`, `reports` | copy it, then TODO 4 |

`src/ledger/models.py`:

```python
"""The data the application works with."""

from dataclasses import dataclass
from datetime import date
from decimal import Decimal


@dataclass(frozen=True)
class Transaction:
    """One movement of money in one account. Negative amounts are money going out."""

    date: date
    account: str
    description: str
    category: str
    amount: Decimal
```

`src/ledger/storage.py`:

```python
"""Reading transactions from where they are stored."""

import csv
from datetime import date
from decimal import Decimal
from pathlib import Path

from ledger.models import Transaction


def read_csv(path: Path) -> list[Transaction]:
    """Reads a CSV file with the columns of a `Transaction`, one row per transaction."""
    with open(path, newline="") as f:
        return [
            Transaction(
                date=date.fromisoformat(row["date"]),
                account=row["account"],
                description=row["description"],
                category=row["category"],
                amount=Decimal(row["amount"]),
            )
            for row in csv.DictReader(f)
        ]
```

`src/ledger/reports.py`:

```python
"""Calculations on transactions. No files, no printing: data in, data out."""

from collections import defaultdict
from collections.abc import Callable, Iterable
from decimal import Decimal

from ledger.models import Transaction

TRANSFER = "transfer"


def totals_by(
    transactions: Iterable[Transaction], key: Callable[[Transaction], str]
) -> dict[str, Decimal]:
    """Adds up the amounts of the transactions for each value of `key`."""
    totals: dict[str, Decimal] = defaultdict(Decimal)
    for t in transactions:
        totals[key(t)] += t.amount
    return dict(totals)


def is_spending(t: Transaction) -> bool:
    """Money that left the accounts: negative, and not a transfer between them."""
    # TODO 1: return True when the amount is negative and the category is not
    #         TRANSFER, and False otherwise.
    raise NotImplementedError


def balances(transactions: Iterable[Transaction]) -> dict[str, Decimal]:
    """The balance of each account, by account name, sorted by name."""
    # TODO 2: add up the amounts of each account with totals_by, and return the
    #         result sorted by account name. spending_by_category, below, shows
    #         how to call totals_by with a key.
    raise NotImplementedError


def spending_by_category(transactions: Iterable[Transaction]) -> dict[str, Decimal]:
    """The spending of each category, as positive amounts, largest first."""
    spending = [t for t in transactions if is_spending(t)]
    totals = totals_by(spending, key=lambda t: t.category)
    return {c: -total for c, total in sorted(totals.items(), key=lambda item: item[1])}


def net_by_month(transactions: Iterable[Transaction]) -> dict[str, Decimal]:
    """Income minus spending of each month (`2026-01`...), sorted by month."""
    # TODO 3: add up all the amounts of each month, transfers included.
    #         t.date.strftime("%Y-%m") gives the month of a transaction as text.
    raise NotImplementedError
```

`src/ledger/cli.py`:

```python
"""The command line: reads the arguments, calls storage and reports, prints."""

import argparse
from decimal import Decimal
from pathlib import Path

from rich.console import Console
from rich.table import Table

from ledger import reports, storage

DEFAULT_FILE = Path("data/transactions.csv")


def print_totals(totals: dict[str, Decimal]) -> None:
    for name, total in totals.items():
        print(f"{name:<10} {total:>10}")


def print_spending(spending: dict[str, Decimal]) -> None:
    table = Table(title="Spending by category")
    table.add_column("Category")
    table.add_column("EUR", justify="right")
    for category, total in spending.items():
        table.add_row(category, f"{total:,.2f}")
    Console().print(table)


def main() -> None:
    parser = argparse.ArgumentParser(
        prog="ledger", description="Balances and spending of your accounts."
    )
    parser.add_argument("command", choices=["balance", "spending"])
    parser.add_argument("file", nargs="?", type=Path, default=DEFAULT_FILE)
    args = parser.parse_args()

    transactions = storage.read_csv(args.file)
    if args.command == "balance":
        print_totals(reports.balances(transactions))
    else:
        print_spending(reports.spending_by_category(transactions))
    # TODO 4: add a command "months" that prints reports.net_by_month with
    #         print_totals. Two changes: the choices above, and a branch here.
```

All the code of the old `__init__.py` now has a new home, so replace its contents with just a docstring:

```python
"""ledger: balances and spending from a CSV file of transactions."""
```

The `ledger` command must now call the `main` that is in `cli.py`. In `pyproject.toml`, change the entry point:

```toml
[project.scripts]
ledger = "ledger.cli:main"
```

Run the program:

```bash
uv run ledger balance
```

<details>
<summary>✅ <b>What you should see</b></summary>

A few lines such as `Built ledger` (uv noticed the change in `pyproject.toml` and reinstalled the project), then a traceback that ends in:

```text
  File ".../src/ledger/reports.py", line 34, in balances
    raise NotImplementedError
NotImplementedError
```

The command now starts in `cli.main`, which calls `reports.balances`, which is one of your TODOs. If you get `AttributeError: module 'ledger' has no attribute 'main'` instead, `pyproject.toml` still says `ledger:main`.

</details>

## 3. Write the calculations

Open `src/ledger/reports.py`. `totals_by` and `spending_by_category` are written: read them first. `totals_by` adds up the amounts grouped by a **key**, a function that receives a transaction and returns the group it belongs to, such as `lambda t: t.category`. A `lambda` is a short function without a name: `lambda t: t.category` is the same as `def f(t): return t.category`.

Now complete the two functions marked `TODO 1` and `TODO 2`, and delete the `raise NotImplementedError` line of each:

- **TODO 1, `is_spending(t)`:** return `True` when the transaction is spending, that is, its amount is negative and its category is not `TRANSFER`; `False` otherwise.
- **TODO 2, `balances(transactions)`:** return the total amount of each account, as a dictionary sorted by account name.

<details>
<summary>💡 <b>Hints</b></summary>

- A comparison such as `t.amount < 0` is already `True` or `False`, so `is_spending` can return two comparisons combined with `and`. Fields are attributes: `t.amount`, `t.category`.
- `spending_by_category` shows how to call `totals_by` with a key. For the balances, the key is the account.
- `sorted(d.items())` gives the pairs `(key, value)` of a dictionary sorted by key, and `dict(...)` turns the pairs back into a dictionary.

</details>

Then run both commands and **compare** them with the output you recorded in step 1:

```bash
uv run ledger balance > ~/smd/before/balance-new.txt
uv run ledger spending > ~/smd/before/spending-new.txt
diff ~/smd/before/balance.txt ~/smd/before/balance-new.txt
diff ~/smd/before/spending.txt ~/smd/before/spending-new.txt
```

<details>
<summary>✅ <b>What you should see</b></summary>

The two `diff` commands print **nothing**. No output from `diff` means the files are identical: the refactoring kept the behaviour.

If `diff` prints lines, those are the differences: lines starting with `<` are the old output, `>` the new one. Missing or extra categories usually mean that `is_spending` is wrong; balances in a different order mean they are not sorted.

</details>

Commit the refactoring:

```bash
git status
git add src pyproject.toml
git commit -m "Split the ledger into models, storage, reports and cli"
```

<details>
<summary>✅ <b>What you should see</b></summary>

`git status` shows `__init__.py` and `pyproject.toml` as modified, and the four new modules as untracked. The commit reports `6 files changed`.

</details>

## 4. Add the `months` command

With the new structure, a new report is one function in `reports.py` and a few lines in `cli.py`. Add a command `ledger months` that prints the net change of each month (income minus spending, transfers included):

- **TODO 3, `net_by_month(transactions)` in `reports.py`:** the total amount of each month, as a dictionary sorted by month. `t.date.strftime("%Y-%m")` gives the month of a transaction as text, such as `2026-01`.
- **TODO 4, in `cli.py`:** add `"months"` to the choices of the command, and a branch in `main` that prints `reports.net_by_month(transactions)` with `print_totals`.

<details>
<summary>💡 <b>Hints</b></summary>

- `net_by_month` is `balances` with a different key.
- In `main`, the `else` catches everything that is not `balance`. Turn it into `elif args.command == "spending":` and add the new case after it.

</details>

```bash
uv run ledger months
git diff --stat
git commit -am "Add the months command"
```

<details>
<summary>✅ <b>What you should see</b></summary>

```text
2026-01       1375.22
2026-02       1440.07
2026-03       1439.81
```

The three months add up to `4255.10`, the sum of the two balances. `git diff --stat` shows that the feature touched only `reports.py` (the calculation) and `cli.py` (the command), and `print_totals`, written for the balances, is reused as it is. `git commit -a` stages every modified tracked file before committing.

</details>

## 5. Configure Ruff's rules

Choose the project's linting rules: add this at the end of `pyproject.toml`:

```toml
[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM"]
```

[Code smells](design.md#code-smells) explains each group. Run the linter and the formatter:

```bash
uv run ruff check
uv run ruff format --diff
```

<details>
<summary>✅ <b>What you should see</b></summary>

The code given on this page passes, so anything Ruff reports is in the code you wrote. For example:

- `SIM103 Return the condition directly`: an `if ...: return True` / `return False` that can be a single `return`.
- `E501 Line too long (92 > 88)`: a line longer than 88 characters.

</details>

Fix them: `uv run ruff check --fix` fixes the ones marked `[*]`, `uv run ruff format` fixes the layout, and the rest you fix by hand. Check that `uv run ledger balance` still works, and commit:

```bash
uv run ruff check
git commit -am "Configure Ruff's rules"
```

## 6. Pull request

Publish the branch and merge it through a pull request (see [Open and merge a pull request](../05-git-collaboration/lab.md#10-open-and-merge-a-pull-request)):

```bash
git push -u origin refactor/package
```

On GitHub, open a pull request from `refactor/package` to `main`. In the description, say what the change does (the four modules and what each one is for), how you checked that the behaviour didn't change, and the new command. Merge it with **Create a merge commit**. Locally:

```bash
git switch main
git pull
git branch -d refactor/package
git log --oneline --graph
```

<details>
<summary>✅ <b>What you should see</b></summary>

`main` has the merge of the pull request, with your three commits on the branch, and `uv run ledger months` works on `main`.

</details>

## Checkpoint

You have finished the lab when your repository on GitHub shows:

- [ ] `src/ledger/` with `models.py`, `storage.py`, `reports.py` and `cli.py`, and an `__init__.py` with only a docstring;
- [ ] no `TODO` and no `NotImplementedError` left, and `reports.py` with no `print`, no `open` and no import of `cli` or `storage`;
- [ ] `[project.scripts]` pointing to `ledger.cli:main`, and the `[tool.ruff.lint]` rules, with `uv run ruff check` passing;
- [ ] `ledger balance`, `ledger spending` and `ledger months` working;
- [ ] a merged pull request from `refactor/package` with a description, merged with a merge commit.

[← Previous: Software design and project structure](design.md) [Back to the session ↑](index.md)
