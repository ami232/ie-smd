# Python projects and environments

> ℹ️ **What you need first**
>
> The layers of an environment and the idea of reproducibility, from [Environments and reproducibility](../01-dev-environments/environments.md). Environment variables and `PATH`, from [Users, packages and the environment](../02-linux-shell/system.md#environment-variables).

## From a script to a project

Ana's fraud script from [session 1](../01-dev-environments/environments.md#it-works-on-my-machine) was a **script**: one file, run by hand, depending on whatever happened to be installed on her laptop. Scripts are great for exploring. Once other people need to run the code, though, on other machines and for months, it becomes a **project**, and it needs four things the script didn't have:

1. **A declared Python version**, so nobody runs it on an interpreter that can't understand it.
2. **Its own set of installed libraries**, separate from those of every other project on the machine.
3. **A precise list of those libraries and their versions**, so that every installation is identical.
4. **A standard way to run it**, documented in the project itself.

This page explains the version, the environment and the way to run it, and the [lab](lab.md) builds them with uv. The precise list of libraries comes with the project's first dependency, in [session 4](../04-dependencies-tools/dependencies.md).

## The interpreter

When you run `python3 report.py`, the program `python3` is the **interpreter**. It reads your source code, translates it into an intermediate form called *bytecode* and executes it, instruction by instruction. The standard interpreter, the one you download from python.org or that uv installs, is called **CPython** because it is written in C.

This is different from languages like C, Go or Rust, which are **compiled**: a compiler turns the source code into machine code for one processor and one operating system *before* it runs, and you distribute the result. Python code, by contrast, is distributed as source, and it runs wherever there is a suitable interpreter. That makes the interpreter itself part of the environment, with two consequences:

- **The interpreter must be installed** on every machine that runs the code. A container image for a Python application (session 11) always starts from an image that already has Python.
- **Its version matters**, because each version understands a slightly different language.

### Python versions

Python versions have the form `3.14.2`: **major**, **minor** and **patch** numbers. Python has been at major version 3 since 2008. A new **minor** version comes out every October, adding features to the language and to the standard library. Each one receives bug fixes and then security fixes for **five years**, after which it reaches its *end of life*. **Patch** versions (`3.14.1`, `3.14.2`…) only fix bugs and never change the language.

Code is **forward compatible** within major version 3, with few exceptions: code written for 3.10 almost always runs on 3.14. The reverse is not true. Code that uses something new fails on older versions:

| Feature | Since | What happens on an older Python |
|---|---|---|
| `match` statement | 3.10 | `SyntaxError` before anything runs |
| `tomllib` module (read TOML files) | 3.11 | `ModuleNotFoundError` |
| `type` aliases (`type Amount = Decimal`) | 3.12 | `SyntaxError` |

That is why a project declares the **oldest Python it supports**, with `requires-python = ">=3.14"` in its configuration. Tools refuse to install the project on anything older, so the error happens at installation with a clear message, not in production with a confusing one.

Which version should a new project use? Usually **the newest one that all your libraries support**. A brand-new Python version takes a few months to be supported by every library that includes compiled code, so professional teams often stay one version behind the newest during its first months. In this course we use **Python 3.14**.

## Where packages go

### Imports and site-packages

When Python runs `import pandas`, it searches a list of folders, `sys.path`, for a module or package called `pandas`. One of those folders is the interpreter's **`site-packages`**, and that is where installing a package puts it. Installing is just downloading a package (usually a **wheel**, a ready-to-install `.whl` archive) from the **Python Package Index** (PyPI, [pypi.org](https://pypi.org)) and unpacking it into `site-packages`.

That works well for one project. Now imagine two projects on the same laptop. Project A was written two years ago with `pandas 1.5`. Project B is new and needs `pandas 2.3`. There is only one `site-packages` per interpreter, so there can be only one pandas. Install 2.3 for project B, and project A breaks. This is the classic dependency conflict, and the more projects you have, the worse it gets.

### Virtual environments

The solution is to give each project its own `site-packages`.

> 📘 **Definition: virtual environment**
>
> A **virtual environment** (venv) is a folder, conventionally called `.venv` inside the project, that contains a link to a Python interpreter and its own empty `site-packages`. Packages installed in it are visible only to programs that run with that environment's Python, `.venv/bin/python`.

Inside the venv there is a `bin` folder with its own `python` and the command-line programs of the installed packages. Running code "in the environment" just means using that `python`. There are two ways to do so:

- **Activate** the environment: `source .venv/bin/activate` puts `.venv/bin` at the front of your `PATH` (that is all it does), so `python` now means the venv's Python. `deactivate` undoes it.
- **Let a tool choose it**: `uv run report.py` finds the project's `.venv`, creates or updates it if needed, and runs the command with its Python. You never activate anything, so you can't forget to.

A virtual environment is disposable. It is built from the project's files, it is not committed to Git, and if it gets into a strange state you delete it and rebuild it. It is also **specific to one machine**: it contains absolute paths and, for compiled libraries, code for one operating system and architecture. You never copy a `.venv` to another computer; you rebuild it there.

## pyproject.toml

Every modern Python project is described by one file, `pyproject.toml`, written in the TOML format (sections in `[brackets]`, then `key = value` lines). This is the one `uv init` creates in the lab:

```toml
[project]
name = "ledger"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.14"
dependencies = []

[project.scripts]
ledger = "ledger:main"

[build-system]
requires = ["uv_build>=0.13.6,<0.14.0"]
build-backend = "uv_build"
```

- `[project]` holds the standard metadata: the name, the version of *your* project, the supported Python and the list of direct dependencies. It is empty for now: `ledger` gets its first dependency in [session 4](../04-dependencies-tools/dependencies.md).
- `[project.scripts]` creates a command: `ledger` will run the function `main` of the package `ledger`. This is the "standard way to run it" from the start of the page.
- `[build-system]` says which tool turns the project into an installable package. You won't need to touch it.

The structure is a standard (it is the same for every tool), so the project doesn't depend on uv: pip and others can read it too.

## uv

For years, Python developers needed several tools for all this: `pyenv` to install Python versions, `venv` or `virtualenv` for environments, `pip` to install packages, `pip-tools` or Poetry for lockfiles, `pipx` for command-line tools. **uv** does all of it in one program, and much faster (it is written in Rust). These are the commands for creating and running a project:

| Command | Does |
|---|---|
| `uv python install 3.14` | downloads and installs a Python version, independently of the system's |
| `uv python list` | shows the Python versions uv can find, its own and the system's |
| `uv init --package` | creates a project: `pyproject.toml`, `.python-version`, `src/<name>/` |
| `uv run <cmd>` | runs a command in the project's environment, creating the environment first if needed |
| `uv python pin 3.14` | changes the project's Python version in `.python-version` |

The file `.python-version` (just `3.14`) tells uv which Python to use for the project's environment, and uv downloads it if it's missing. The commands for adding libraries to a project come in [session 4](../04-dependencies-tools/dependencies.md#uv-commands-for-dependencies).

## Common mistakes

- **Using `pip install` on the system Python.** It mixes the packages of every project, and on recent Linux and macOS versions it is refused with an `externally-managed-environment` error, precisely to prevent this. Install libraries in a project's environment instead.
- **Committing `.venv`.** It is large, specific to one machine and rebuilt in seconds. Commit the files that describe the project (`pyproject.toml`, `.python-version`, `src/`) and ignore `.venv`.
- **"It works in my notebook."** A notebook's kernel may use a completely different Python and packages than your project. If the code is part of the project, run it with `uv run`.

> ✅ **Key takeaways**
>
> - Python code runs on an **interpreter**, which is part of the environment. New language features fail on older versions, so a project declares `requires-python`.
> - Installed packages go into the interpreter's `site-packages`. A **virtual environment** gives each project its own, avoiding conflicts. It is disposable and is rebuilt, never copied.
> - `pyproject.toml` describes the project: its name, the Python it needs (`requires-python`) and the commands it provides (`[project.scripts]`).
> - **uv** installs Python versions, creates projects and runs commands in the project's environment with `uv run`.

## Exercises

**1.** A colleague's code starts with `import tomllib` and fails with `ModuleNotFoundError` on the server, though it works on their laptop. What is the likely cause, and what line in `pyproject.toml` would have caught it earlier?

<details>
<summary>✅ <b>Solution</b></summary>

`tomllib` was added to the standard library in Python 3.11, so the server is running an older Python. If the project declared `requires-python = ">=3.11"` (or higher), installing it on the server would have failed at once with a clear message about the Python version, instead of failing when the code ran.

</details>

**2.** Explain what `source .venv/bin/activate` changes, and why `uv run ledger` works without it.

<details>
<summary>✅ <b>Solution</b></summary>

Activating only puts `.venv/bin` at the front of the `PATH` variable of the current shell, so commands like `python` or `ledger` are found in the virtual environment first. `uv run` doesn't need that, because it finds the project's environment itself (from the `pyproject.toml` of the current folder or its parents), creates or updates it if needed, and runs the command with it directly.

</details>

[← Back to the session](index.md) [Next: Lab 3 →](lab.md)
