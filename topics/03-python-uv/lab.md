# Lab 3: the ledger becomes a Python project

> ℹ️ **What you need first**
>
> The course environment from [session 2](../02-linux-shell/setup.md) and the folder `~/smd/ledger` with `data/transactions.csv` from [Lab 2](../02-linux-shell/lab.md). The concepts are in [Python projects and environments](python.md).

> ⚠️ **If you didn't finish Lab 2**
>
> This lab only needs the data file from Lab 2. These commands create the folder and download the file:
>
> ```bash
> mkdir -p ~/smd/ledger/data
> cd ~/smd/ledger
> curl -fsSL -o data/transactions.csv https://raw.githubusercontent.com/ami232/ie-smd/main/data/transactions.csv
> ```
>
> `ls data` should show `transactions.csv`. Finish the rest of Lab 2 at home.

In Lab 2, `balance.sh` answered one question with `awk`. In this lab you turn `~/smd/ledger` into a Python project: a pinned Python version, its own virtual environment and a command called `ledger`. In [Lab 4](../04-dependencies-tools/lab.md) you write the code of that command.

Work in the course shell (the macOS Terminal, the Ubuntu terminal in WSL, the VM or the codespace).

## 1. Install uv

**macOS**

```bash
brew install uv
```


**Linux, WSL, VM, codespace**

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

The installer puts `uv` in `~/.local/bin` and adds that folder to your `PATH` in your shell's startup file. **Open a new terminal** (or run `source $HOME/.local/bin/env`) so the shell sees it.


Then check:

```bash
uv --version
```

<details>
<summary>✅ <b>What you should see</b></summary>

A line such as `uv 0.13.6`. Any recent version works. If you get `command not found`, it is the `PATH` problem from [session 2](../02-linux-shell/system.md#path-how-the-shell-finds-commands): open a new terminal.

</details>

> ⚠️ **If your prompt starts with `(base)`: conda**
>
> If you installed Anaconda or Miniconda, your prompt may start with `(base)`. This means conda's base environment is active in every terminal, so `python3` and `pip` are conda's. In the next steps you compare the system Python with the project's Python, and with conda active the results are confusing. VS Code may also suggest conda's Python instead of the project's. Turn it off for this course:
>
> ```bash
> conda deactivate
> conda config --set auto_activate_base false
> ```
>
> The first command turns it off in this terminal. The second stops it from turning on in new terminals (if conda says the setting is deprecated, use `conda config --set auto_activate false`). Conda stays installed, and `conda activate base` turns it on when you need it. Open a new terminal: the `(base)` should be gone.

## 2. Install a Python version

First, see what your system already has:

```bash
python3 --version
which python3
```

This is the **system Python**: the one your operating system uses for its own tools. It may be missing, or a different version on each machine (Ubuntu 24.04 comes with 3.12; macOS may have none, or the 3.9 from Apple's developer tools). Don't rely on it for projects. Ask uv for the exact version we use instead:

```bash
uv python install 3.14
uv python list --only-installed
```

<details>
<summary>✅ <b>What you should see</b></summary>

uv downloads a standalone build of Python 3.14 (about 30 MB) and reports something like `Installed Python 3.14.10`. The list shows it next to any Python your system already had. uv keeps its own Pythons in its own folder and never touches the system one.

</details>

## 3. Create the project

Go into the ledger folder and turn it into a project:

```bash
cd ~/smd/ledger
uv init --package --vcs none --python 3.14
```

- `--package` creates an installable package with a command-line entry point (not just a script).
- `--vcs none` skips creating a Git repository. We do that properly in session 5.
- `--python 3.14` pins the project to Python 3.14.

Look at what it created, including hidden files:

```bash
ls -a
cat pyproject.toml
cat .python-version
cat src/ledger/__init__.py
```

<details>
<summary>✅ <b>What you should see</b></summary>

Your files from Lab 2 (`data/` and, if you made them, `balance.sh` and `report.txt`) are untouched. Next to them there are four new ones: `.python-version` (contains `3.14`), `README.md`, `pyproject.toml` and `src/ledger/__init__.py`.

`pyproject.toml` is like the one explained in [the concepts page](python.md#pyprojecttoml): `requires-python = ">=3.14"`, no dependencies yet, and a script `ledger = "ledger:main"`. If your Git configuration has your name, uv adds an `authors` line too. The package contains a single function:

```python
def main() -> None:
    print("Hello from ledger!")
```

</details>

## 4. First run

```bash
uv run ledger
```

The first time, uv creates the virtual environment and installs the project in it. Then it runs the `ledger` command, which calls `main()`. Now look at the environment:

```bash
ls -a
ls .venv/bin
uv run python -c "import sys; print(sys.executable)"
```

<details>
<summary>✅ <b>What you should see</b></summary>

```text
Using CPython 3.14.10
Creating virtual environment at: .venv
...
Hello from ledger!
```

There are now a `.venv` folder and a `uv.lock` file. `uv.lock` lists the exact packages installed in `.venv`; for now only `ledger` itself, as the project has no dependencies yet. It becomes important when the project gets its first dependency, in [Lab 4](../04-dependencies-tools/lab.md). `.venv/bin` contains `python` and `ledger`, the command defined in `[project.scripts]`. The last command prints a path ending in `ledger/.venv/bin/python3`: `uv run` uses the project's own Python, not the system one.

</details>

## 5. requires-python at work

Try to pin the project to an older Python:

```bash
uv python pin 3.13
```

<details>
<summary>✅ <b>What you should see</b></summary>

```text
error: The requested Python version `3.13` is incompatible with the project `requires-python` value of `>=3.14`.
```

(uv may first download Python 3.13.) The project declares that it needs 3.14 or newer, and uv enforces it. This error, at setup time, is what would have saved Ana from the `SyntaxError` on the server in [session 1](../01-dev-environments/environments.md#it-works-on-my-machine).

</details>

## Checkpoint

You have finished the lab when:

- [ ] `~/smd/ledger` contains `pyproject.toml`, `.python-version`, `src/ledger/__init__.py` and a `.venv` folder;
- [ ] `uv run ledger` prints `Hello from ledger!`, and `uv run python -c "import sys; print(sys.executable)"` prints a path inside `.venv`;
- [ ] `uv python pin 3.13` is refused, and you can explain why;
- [ ] you can explain what is inside `.venv`, and why each project has its own.

[← Previous: Python projects and environments](python.md) [Back to the session ↑](index.md)
