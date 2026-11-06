# Lab 4: writing and checking the ledger

> ℹ️ **What you need first**
>
> The `ledger` project from [Lab 3](../03-python-uv/lab.md): the folder `~/smd/ledger` with `pyproject.toml`, `src/ledger/__init__.py` and `.venv`, where `uv run ledger` prints `Hello from ledger!`. The concepts of dependencies are in [Dependencies and the lockfile](dependencies.md).

In Lab 3 you created the `ledger` project. In this lab you write its code in a real editor and make it a complete project. You set up VS Code and write the ledger in Python (steps 1 to 3). You add a third-party library, which gives the project a lockfile, and prove that the project rebuilds from its files (steps 4 to 6). Then you use two tools on the code: the debugger, to watch the program run line by line, and Ruff, to find a bug without running the code (steps 7 and 8).

## 1. Set up VS Code

From here on you write Python. A plain text editor saves the file, but it doesn't help you write it. Set up VS Code, the editor we use for the rest of the course, and connect it to the project.

### Install VS Code and the `code` command

You open projects from the terminal with the `code` command, so VS Code must be installed **and** `code` must be in your `PATH`.

**macOS**

```bash
brew install --cask visual-studio-code
```

Homebrew installs the app in Applications and puts `code` in its own folder of the `PATH`. If Homebrew says the app already exists, you installed VS Code earlier from a download: keep it, and skip to the check below.


**Windows (WSL)**

VS Code runs **on Windows**, not inside Ubuntu. In PowerShell:

```powershell
winget install -e --id Microsoft.VisualStudioCode
```

or download the installer from [code.visualstudio.com](https://code.visualstudio.com/) and keep the option **Add to PATH** checked. Then open VS Code, go to the Extensions view (<kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>X</kbd>), search for **WSL** and install the extension by Microsoft. Close VS Code.

With the WSL extension, the VS Code window runs on Windows, and a small server runs inside Ubuntu. The files, Python and uv stay in Linux, where the rest of the course runs.


Now **open a new terminal** (the Ubuntu terminal on Windows) and check:

```bash
code --version
```

<details>
<summary>✅ <b>What you should see</b></summary>

Three lines: a version number such as `1.106.0`, a long commit identifier and the architecture (`arm64` or `x64`).

If you get `command not found`, the `code` command is not in your `PATH`:

- **macOS, VS Code installed from a download:** move Visual Studio Code to the Applications folder if it isn't there, and open it. Open the Command Palette (<kbd>Cmd</kbd>+<kbd>Shift</kbd>+<kbd>P</kbd>), type `shell command` and choose **Shell Command: Install 'code' command in PATH**. Open a new terminal and try again.
- **Windows:** close every Ubuntu terminal and open a new one. WSL adds the folders of the Windows `PATH` to Linux's `PATH` when the terminal starts, so a terminal opened before installing VS Code doesn't see `code`.

</details>

### Open the project

```bash
cd ~/smd/ledger
code .
```

`.` is the current folder: you are telling VS Code to open the folder you are in. VS Code calls an open folder a **workspace**.

<details>
<summary>✅ <b>What you should see</b></summary>

A VS Code window with the project's files in the Explorer on the left: `.venv`, `data`, `src`, `.python-version`, `pyproject.toml`, `README.md` and `uv.lock`.

- **macOS:** VS Code asks *Do you trust the authors of the files in this folder?* Choose **Yes, I trust the authors**: they are your own files.
- **Windows:** the first time, VS Code installs its server in Ubuntu, which takes a few seconds. The bottom-left corner shows **WSL: Ubuntu**. If it doesn't, the window is working on Windows files: close it and run `code .` again from the Ubuntu terminal. Don't open the folder from the Windows File Explorer.

</details>

### Install the Python extension

Open the Extensions view (<kbd>Cmd</kbd>+<kbd>Shift</kbd>+<kbd>X</kbd> on macOS, <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>X</kbd> on Windows), search for **Python** and install the extension **Python** by Microsoft. On Windows, the button says **Install in WSL: Ubuntu**: extensions that work with your code run on the Linux side, next to it.

The Python extension also installs two others: **Pylance**, the language server that understands your code, and **Python Debugger**, which you use in step 7.

### Select the project's Python

Pylance needs to know which Python runs the project, to know which libraries are installed and where. Open the Command Palette (<kbd>Cmd</kbd>+<kbd>Shift</kbd>+<kbd>P</kbd> on macOS, <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>P</kbd> on Windows), type `select interpreter`, choose **Python: Select Interpreter**, and pick the one whose path is `./.venv/bin/python`. VS Code usually marks it as *Recommended*.

Then open a terminal inside VS Code with **Terminal → New Terminal** (<kbd>Ctrl</kbd>+<kbd>Grave</kbd>):

```bash
pwd
uv run ledger
```

<details>
<summary>✅ <b>What you should see</b></summary>

The status bar at the bottom right shows the Python version (3.14) and `.venv`. The terminal opens at the bottom of the window, already in the project folder: `pwd` prints `.../smd/ledger`, and `uv run ledger` prints `Hello from ledger!`.

On Windows this terminal is bash in Ubuntu, the same shell as the Ubuntu terminal.

</details>

## 2. Write the ledger

In VS Code, open `src/ledger/__init__.py` from the Explorer on the left, and run the commands of this lab in VS Code's terminal. If you can't use VS Code, any text editor works: `nano src/ledger/__init__.py` in the terminal (<kbd>Ctrl</kbd>+<kbd>O</kbd> saves, <kbd>Ctrl</kbd>+<kbd>X</kbd> exits) or TextEdit on macOS in plain-text mode.

Replace the contents of the file with the following, and save it (<kbd>Cmd</kbd>+<kbd>S</kbd> on macOS, <kbd>Ctrl</kbd>+<kbd>S</kbd> on Windows). A dot on the file's tab means it has unsaved changes, and `uv run` only sees what is saved:

```python
"""ledger: balances and spending from a CSV file of transactions."""

import argparse
import csv
from collections import defaultdict
from decimal import Decimal


def read_transactions(path):
    """Reads the CSV file and returns one dict per row, with the amount as a Decimal."""
    with open(path, newline="") as f:
        transactions = list(csv.DictReader(f))
    for t in transactions:
        t["amount"] = Decimal(t["amount"])
    return transactions


def totals_by(transactions, column):
    """Adds up the amounts for each value of `column` (account, category...)."""
    totals = defaultdict(Decimal)
    for t in transactions:
        totals[t[column]] += t["amount"]
    return dict(totals)


def main() -> None:
    parser = argparse.ArgumentParser(prog="ledger", description=__doc__)
    parser.add_argument("command", choices=["balance", "spending"])
    parser.add_argument("file", nargs="?", default="data/transactions.csv")
    args = parser.parse_args()

    transactions = read_transactions(args.file)
    if args.command == "balance":
        for account, total in sorted(totals_by(transactions, "account").items()):
            print(f"{account:<10} {total:>10}")
    else:
        spending = [t for t in transactions if t["amount"] < 0 and t["category"] != "transfer"]
        for category, total in sorted(totals_by(spending, "category").items(), key=lambda item: item[1]):
            print(f"{category:<14} {total:>10}")
```

Type it rather than pasting it, and notice what the editor does: it completes names (`Decim` + <kbd>Tab</kbd> gives `Decimal`), shows the documentation of a function when you hover over it, and underlines in red anything it doesn't understand.

What it does:

- `read_transactions` uses the standard `csv` module. `DictReader` turns each row into a dictionary keyed by the header (`t["account"]`, `t["amount"]`…). Each amount becomes a `Decimal` from its **string**. [Step 3](#3-money-is-not-a-float) explains why.
- `totals_by` adds up the amounts grouped by any column. This is the same idea as `s[$2] += $5` in `balance.sh`.
- `main` uses the standard `argparse` module to read the command line: a command (`balance` or `spending`) and an optional file, which defaults to `data/transactions.csv`.

Run it:

```bash
uv run ledger balance
uv run ledger spending
uv run ledger --help
uv run ledger total
```

<details>
<summary>✅ <b>What you should see</b></summary>

```text
checking      3349.10
savings        906.00
```

The same balances as `balance.sh` in Lab 2. Then the spending by category, most negative first:

```text
housing          -2850.00
groceries         -797.60
restaurants       -320.09
transport         -135.57
health            -119.70
subscriptions      -77.94
```

`--help` prints a usage message that `argparse` generated for free. `ledger total` fails with `invalid choice: 'total' (choose from 'balance', 'spending')`, and `echo $?` would print `2`: a proper error and a proper exit code, unlike the script from Lab 2.

</details>

## 3. Money is not a float

`ledger` handles money, and the code above stores every amount as a `Decimal`, not a `float`. Here is why. Python's `float` stores numbers in **binary**, and most decimal fractions, like 0.1, have no exact binary representation, just as 1/3 has no exact decimal one. The error is tiny, but it shows up:

```python
print(0.1 + 0.2)
```

```text
0.30000000000000004
```

```python
print(0.1 + 0.2 == 0.3)
```

```text
False
```

Adding up many amounts accumulates these errors. These are the payments to a gym in three months, as floats:

```python
print(-39.90 + -39.90 + -39.90)
```

```text
-119.69999999999999
```

The standard library's **`Decimal`** type stores numbers in decimal, exactly as written, so sums of amounts with two decimals are exact. Create a `Decimal` from a **string**, not from a float, or you copy the float's error into it:

```python
from decimal import Decimal

print(Decimal("-39.90") + Decimal("-39.90") + Decimal("-39.90"))
```

```text
-119.70
```

```python
print(Decimal("0.1") + Decimal("0.2") == Decimal("0.3"))
```

```text
True
```

```python
print(Decimal(0.1))  # created from a float: the binary error is now visible
```

```text
0.1000000000000000055511151231257827021181583404541015625
```

In a report, a balance of `3349.0999999999995` looks careless. In a payment system, an amount that is one cent wrong is a reconciliation problem, and comparisons such as `balance == 0` can fail when they shouldn't. The rule in financial software is simple: **money is never a `float`**. In Python you use `Decimal`. In databases you use a `NUMERIC` column (session 7). In some systems amounts are stored as integers in cents.

Now see it in `ledger`: break it on purpose. In `read_transactions`, change `Decimal(t["amount"])` to `float(t["amount"])`. In `totals_by`, change `defaultdict(Decimal)` to `defaultdict(float)`. Run both commands again:

```bash
uv run ledger balance
uv run ledger spending
```

<details>
<summary>✅ <b>What you should see</b></summary>

```text
checking   3349.0999999999995
savings         906.0
```

and, among the categories, `health -119.69999999999999`. Same data, same logic, and wrong-looking money. **Undo both changes** before you continue, and check that `uv run ledger balance` prints `3349.10` again.

</details>

**Question.** A teammate stores amounts as floats and writes `if balance == 0: close_account()`. After a deposit of 0.10, a deposit of 0.20 and a withdrawal of 0.30, the account is not closed. Explain why, and show the fix.

<details>
<summary>✅ <b>Solution</b></summary>

With floats, `0.1 + 0.2 - 0.3` is `5.551115123125783e-17`, not 0, because 0.1, 0.2 and 0.3 are not exactly representable in binary. The comparison is false and the account stays open with a "balance" of a few hundred-quadrillionths of a euro. With decimals created from strings, `Decimal("0.10") + Decimal("0.20") - Decimal("0.30")` is exactly `Decimal("0.00")`, which equals 0, so the comparison works.

</details>

## 4. Add a dependency

Plain columns of text are fine, but a table is easier to read. The `rich` library draws tables in the terminal. Add it to the project:

```bash
uv add rich
```

Now look at what changed:

```bash
grep -A2 "^dependencies" pyproject.toml
uv tree
grep -c "^\[\[package\]\]" uv.lock
```

<details>
<summary>✅ <b>What you should see</b></summary>

`pyproject.toml` now lists one direct dependency with a constraint, such as `"rich>=15.0.0"`. `uv tree` shows `rich` and its three transitive dependencies (`markdown-it-py`, `mdurl`, `pygments`). `uv.lock` has **5** packages: those four plus `ledger` itself, each with an exact version and file hashes. Your versions may be newer than the ones shown in [Dependencies and the lockfile](dependencies.md#direct-and-transitive).

</details>

Use `rich` in the `spending` command. At the top of `__init__.py`, after `from decimal import Decimal`, add:

```python
from rich.console import Console
from rich.table import Table
```

In VS Code, these two lines must not be underlined: if they are, the selected interpreter is not the project's ([step 1](#1-set-up-vs-code)). Then replace the last two lines of `main` (the `for` loop of `spending`) with:

```python
        table = Table(title="Spending by category")
        table.add_column("Category")
        table.add_column("EUR", justify="right")
        for category, total in sorted(totals_by(spending, "category").items(), key=lambda item: item[1]):
            table.add_row(category, f"{-total:,.2f}")
        Console().print(table)
```

`-total` shows spending as positive numbers, and the format `,.2f` adds thousands separators and two decimals.

```bash
uv run ledger spending
```

<details>
<summary>✅ <b>What you should see</b></summary>

```text
    Spending by category
┏━━━━━━━━━━━━━━━┳━━━━━━━━━━┓
┃ Category      ┃      EUR ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━━━┩
│ housing       │ 2,850.00 │
│ groceries     │   797.60 │
│ restaurants   │   320.09 │
│ transport     │   135.57 │
│ health        │   119.70 │
│ subscriptions │    77.94 │
└───────────────┴──────────┘
```

</details>

## 5. Isolation

`rich` is installed in the project's environment, and only there. Check it from outside the project's environment:

```bash
python3 -c "import rich"
uv run python -c "import rich; print('rich is here')"
```

<details>
<summary>✅ <b>What you should see</b></summary>

The system Python fails with `ModuleNotFoundError: No module named 'rich'` (or `python3: command not found` if your system has none). Through `uv run`, the import works. Your other projects, and your system, are unaffected by what you install in `ledger`.

</details>

## 6. Rebuild from the files

The environment is disposable. Delete it and rebuild it:

```bash
rm -rf .venv
uv sync
uv run ledger balance
```

Now simulate a teammate who receives the project. Copy **only the files that define it** to a new folder, and run it there:

```bash
mkdir -p /tmp/ledger-copy
cp -r pyproject.toml uv.lock .python-version README.md src data /tmp/ledger-copy/
cd /tmp/ledger-copy
uv run ledger spending
cd ~/smd/ledger
```

<details>
<summary>✅ <b>What you should see</b></summary>

After `rm -rf .venv`, `uv sync` recreates it and installs the same five packages in well under a second. The packages are in uv's cache, so nothing is downloaded again. In the copy, `uv run` creates a new environment from the lockfile and prints the same table.

These files are what you will commit to Git in session 5. `.venv` is not among them.

</details>

## 7. Debug the ledger

When a program gives a wrong result, the first reflex is to add `print` calls, run it again, read the output and remove the `print` calls. A **debugger** does the same job without changing the code: it stops the program at the lines you choose, shows every variable at that moment, and lets you run the rest one line at a time.

### Tell VS Code how to run `ledger`

VS Code's default debug configuration runs the file open in the editor, without arguments. `ledger` is a command that needs an argument (`spending`), so you describe how to run it in a file called `.vscode/launch.json`.

In the Explorer, click the **New File** icon next to the project's name, type `.vscode/launch.json` and press <kbd>Enter</kbd>. VS Code creates the folder and the file. Paste this and save:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "ledger spending",
            "type": "debugpy",
            "request": "launch",
            "program": "${workspaceFolder}/.venv/bin/ledger",
            "args": ["spending"],
            "console": "integratedTerminal"
        }
    ]
}
```

- `"name"` is the name you choose from a list when you start debugging.
- `"type": "debugpy"` uses the Python Debugger extension.
- `"program"` is the `ledger` command that uv created in `.venv/bin` in [step 4 of Lab 3](../03-python-uv/lab.md#4-first-run). `${workspaceFolder}` is the project folder. This is what `uv run ledger` runs: the command imports the `ledger` package and calls `main()`.
- `"args"` are the command-line arguments, the same words you type after `uv run ledger`.
- `"console": "integratedTerminal"` shows the program's output in VS Code's terminal.

### Stop at a breakpoint

Open `src/ledger/__init__.py` and find the line inside `totals_by` that adds up the amounts:

```python
        totals[t[column]] += t["amount"]
```

Click just to the left of its line number. A red dot appears: that is a **breakpoint**, a line where the debugger stops *before* running it.

Open the Run and Debug view (<kbd>Cmd</kbd>+<kbd>Shift</kbd>+<kbd>D</kbd> on macOS, <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>D</kbd> on Windows), check that **ledger spending** is selected at the top, and click the green triangle (or press <kbd>F5</kbd>).

<details>
<summary>✅ <b>What you should see</b></summary>

The program starts and stops at the breakpoint: the line is highlighted in yellow and a toolbar with the debugging buttons appears at the top. On the left:

- **Variables → Locals** shows the variables of `totals_by` at this moment: `column` is `'category'`, `totals` is an empty `defaultdict`, `transactions` is a list of 44 transactions (the 56 transactions minus income, interest and transfers), and `t` is the first of them, the rent:

    ```text
    {'date': '2026-01-01', 'account': 'checking', 'description': 'Rent', 'category': 'housing', 'amount': Decimal('-950.00')}
    ```

- **Call Stack** shows how the program got here: `main` called `totals_by`. Click `main` to see *its* variables (`args`, `transactions`, `spending`), then click `totals_by` again.

</details>

### Step through the code

The debugging toolbar has six buttons. From left to right, with their keys:

| Button | Key | What it does |
|---|---|---|
| Continue | <kbd>F5</kbd> | run until the next breakpoint |
| Step Over | <kbd>F10</kbd> | run the current line and stop at the next one |
| Step Into | <kbd>F11</kbd> | go inside the function called on the current line |
| Step Out | <kbd>Shift</kbd>+<kbd>F11</kbd> | finish the current function and stop where it was called |
| Restart | <kbd>Shift</kbd>+<kbd>Cmd</kbd>+<kbd>F5</kbd> / <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>F5</kbd> | start again |
| Stop | <kbd>Shift</kbd>+<kbd>F5</kbd> | end the program |

On a Mac laptop, the function keys may need <kbd>Fn</kbd>, and <kbd>F11</kbd> may show the desktop: use the buttons instead.

Press **Step Over** twice and watch the Variables panel. The loop runs the highlighted line, goes back to `for t in transactions`, and stops on the highlighted line again with the next transaction. Values that just changed are highlighted.

Now use the **Debug Console**, the tab next to the terminal at the bottom. It evaluates any Python expression in the program's current state. Type, one at a time:

```python
t["description"]
dict(totals)
len(transactions)
sum(t["amount"] for t in transactions)
```

<details>
<summary>✅ <b>What you should see</b></summary>

After two steps, `t` is the second transaction (Carrefour, `-65.15`) and `totals` contains `{'housing': Decimal('-950.00')}`. In the Debug Console: `'Carrefour'`, the same dictionary, `44`, and `Decimal('-4300.90')`, the total spending of the three months.

</details>

### A conditional breakpoint

Stepping through 44 transactions one by one is slow. A **conditional breakpoint** stops only when an expression is true. Right-click the red dot, choose **Edit Breakpoint…**, type this expression and press <kbd>Enter</kbd>:

```python
t["category"] == "health"
```

Press **Continue**. The program runs until the first health transaction. Use **Step Over** and **Continue** to follow `totals["health"]` through the three gym payments.

<details>
<summary>✅ <b>What you should see</b></summary>

The program stops at the gym membership of 10 January (`-39.90`). `totals` has no `health` key yet. After **Step Over** it has `Decimal('-39.90')`. At the next two stops (10 February and 10 March), the total goes to `-79.80` and then, after the last step, `-119.70`.

This is the line where the `float` version of [step 3](#3-money-is-not-a-float) produced `-119.69999999999999`. With `Decimal`, the sum is exact.

</details>

Remove the breakpoint (click the red dot) and press **Continue**. The program runs to the end, and the table appears in the terminal.

**Exercise.** Add a second configuration to `launch.json` that runs `ledger balance`, and use it to stop in `totals_by` while it adds up the accounts. What is the value of `column` this time?

<details>
<summary>✅ <b>Solution</b></summary>

`configurations` is a list: add a second object, separated from the first by a comma. Then choose **ledger balance** from the list at the top of the Run and Debug view.

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "ledger spending",
            "type": "debugpy",
            "request": "launch",
            "program": "${workspaceFolder}/.venv/bin/ledger",
            "args": ["spending"],
            "console": "integratedTerminal"
        },
        {
            "name": "ledger balance",
            "type": "debugpy",
            "request": "launch",
            "program": "${workspaceFolder}/.venv/bin/ledger",
            "args": ["balance"],
            "console": "integratedTerminal"
        }
    ]
}
```

`column` is `'account'`, and `transactions` has all 56 transactions, because `balance` doesn't filter them.

</details>

## 8. Lint and format with Ruff

**Ruff** does two jobs. As a **linter**, it reads the code without running it and reports problems: names that are not defined, imports that are not used, and many patterns that usually hide a bug. As a **formatter**, it rewrites the layout of the code (spaces, line breaks, quotes) in one standard style, so that a whole team writes code that looks the same and a code review shows only real changes.

Ruff is a tool for whoever develops the project, not something `ledger` needs to run. Add it as a **development dependency**:

```bash
uv add --dev ruff
grep -A3 "dependency-groups" pyproject.toml
```

<details>
<summary>✅ <b>What you should see</b></summary>

```toml
[dependency-groups]
dev = [
    "ruff>=0.17.4",
]
```

Your version may be newer. Development dependencies go in their own group, separate from `dependencies`. `uv.lock` pins Ruff too, so the whole team uses the same version and gets the same results.

</details>

### The linter

Run it on the project:

```bash
uv run ruff check
```

It prints `All checks passed!`. Now introduce a typo. In `totals_by`, change `for t in transactions:` to `for t in transaction:` and save. Before running anything, look at the editor: Pylance underlines `transaction` and, when you hover over it, says `"transaction" is not defined`. Run Ruff, then the program:

```bash
uv run ruff check
uv run ledger spending
```

<details>
<summary>✅ <b>What you should see</b></summary>

Ruff finds the problem and shows where it is (your line numbers may be different):

```text
F821 Undefined name `transaction`
  --> src/ledger/__init__.py:24:14
   |
22 |     """Adds up the amounts for each value of `column` (account, category...)."""
23 |     totals = defaultdict(Decimal)
24 |     for t in transaction:
   |              ^^^^^^^^^^^
25 |         totals[t[column]] += t["amount"]
26 |     return dict(totals)
   |

Found 1 error.
```

The program fails with `NameError: name 'transaction' is not defined. Did you mean: 'transactions'?`. Python only finds the error when it reaches that line. Ruff found it without running anything, and in a big program that line might only run in a case nobody tested.

</details>

Fix the typo and check that `uv run ruff check` prints `All checks passed!` again.

### The formatter

Ask Ruff what it would change, without changing anything yet:

```bash
uv run ruff format --diff
```

<details>
<summary>✅ <b>What you should see</b></summary>

A *diff*: lines starting with `-` would be removed, and lines starting with `+` would be added. Ruff wants to split the two lines of `main` that are longer than 88 characters:

```diff
-        spending = [t for t in transactions if t["amount"] < 0 and t["category"] != "transfer"]
+        spending = [
+            t for t in transactions if t["amount"] < 0 and t["category"] != "transfer"
+        ]
```

and the same for the `for category, total in sorted(...)` line.

</details>

Now apply it, and check that the program still does the same:

```bash
uv run ruff format
uv run ledger spending
```

<details>
<summary>✅ <b>What you should see</b></summary>

Ruff reports `1 file reformatted`. VS Code shows the new layout of `__init__.py`, and `ledger spending` prints the same table as before: a formatter changes how the code looks, never what it does. Run `uv run ruff format` again: there is nothing left to change.

</details>

Ruff has hundreds of rules, and a project chooses which ones to apply in `pyproject.toml`. We configure them later in the course, when we reorganize `ledger` into modules. If you want to see Ruff's messages as you type, install the **Ruff** extension (by Astral) from the Extensions view.

## Checkpoint

You have finished the lab when:

- [ ] `code .` opens `~/smd/ledger` in VS Code (on Windows, with **WSL: Ubuntu** in the bottom-left corner), with `.venv` as the interpreter;
- [ ] `uv run ledger balance` prints `3349.10` and `906.00`, and `uv run ledger spending` prints the table;
- [ ] `~/smd/ledger` contains `uv.lock`, and you can explain which files a teammate needs, and why `.venv` is not one of them;
- [ ] `.vscode/launch.json` runs `ledger spending` under the debugger, and you can stop it in `totals_by` and read `t` and `totals`;
- [ ] `ruff` is a development dependency in `pyproject.toml`, and `uv run ruff check` and `uv run ruff format --diff` report nothing to fix.

**Challenge (optional).** Add a third command, `ledger months`, that prints the net change (income minus spending, transfers included) for each month: `2026-01`, `2026-02`, `2026-03`. Hint: `t["date"][:7]` gives the month of a transaction. You could add it as a key to each transaction and reuse `totals_by`.

<details>
<summary>✅ <b>Result to check</b></summary>

The three months sum to the net total of both accounts, `4255.10`:

```text
2026-01    1375.22
2026-02    1440.07
2026-03    1439.81
```

</details>

[← Previous: Dependencies and the lockfile](dependencies.md) [Back to the session ↑](index.md)
