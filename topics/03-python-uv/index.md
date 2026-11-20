# Python projects with uv

Most people learn Python in a notebook: open it, `pip install` whatever is missing, run the cells. That is fine for exploring data, and it is exactly how you get Ana's ["works on my machine"](../01-dev-environments/environments.md#it-works-on-my-machine) problem. Software that a team builds, tests and deploys needs more. It needs a known Python version, its own environment and a standard way to run it. This session sets that up with **uv**: a single, fast tool that manages Python versions, virtual environments and projects.

By the end, `ledger` is a Python project with its own Python and environment, ready for its code.

## The thread of the session

1. **[Python projects and environments](python.md).** What the interpreter is and what Python versions mean for compatibility. Where installed packages go, why every project needs its own **virtual environment**, and what `pyproject.toml` declares.
2. **[Lab: the ledger becomes a Python project](lab.md).** You install uv and a specific Python, create the `ledger` project, run it in its own environment, and see `requires-python` refuse an older Python.

## Objectives

By the end of the session you should be able to:

- [ ] Explain what the Python interpreter does, and what a project's `requires-python` protects against.
- [ ] Explain what a virtual environment is, why each project needs its own, and how `uv run` uses it.
- [ ] Read the `[project]` and `[project.scripts]` sections of a `pyproject.toml`.
- [ ] Install a Python version with uv, create a project with `uv init` and run it with `uv run`.

## Before you start

You need the [course environment](../02-linux-shell/setup.md) and the `~/smd/ledger` folder with the transactions file (see [A ledger in the shell](../02-linux-shell/lab.md)). If you don't have the folder, the start of the [lab](lab.md) has the two commands that create it and download the data. Basic Python is assumed: variables, lists, dictionaries, loops and functions. If you need a refresher, the official [Python tutorial](https://docs.python.org/3/tutorial/) (sections 3 to 5) covers it.

## Class slides

📄 [Slides (PDF)](slides.pdf)
