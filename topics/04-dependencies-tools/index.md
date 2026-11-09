# Dependencies and developer tools

In [session 3](../03-python-uv/index.md) you created the `ledger` project: its own Python, its own environment and a `ledger` command that only says hello. This session gives it real code, and the tools to write and check that code. You set up **VS Code**, an editor that understands Python, and write the ledger in it. You add the project's first third-party library, which brings the last piece of a reproducible project: the **lockfile**. Then you use a **debugger** to watch the program run line by line, and **Ruff** to find mistakes without running the code.

## The thread of the session

1. **[Dependencies and the lockfile](dependencies.md).** Direct and transitive dependencies, version constraints, and why a lockfile is needed on top of them.
2. **[Lab 4: writing and checking the ledger](lab.md).** You set up VS Code and write the ledger in Python, add `rich` and rebuild the project from its files, step through the code with the debugger, and run Ruff.

The concepts of editors, language servers, debuggers, linters and formatters are summarized in the slides below. A full notes page will follow.

## Objectives

By the end of the session you should be able to:

- [ ] Open a project in VS Code from the terminal with `code .`, on your laptop or inside your Linux VM.
- [ ] Select the project's virtual environment as the Python interpreter, and explain what the editor uses it for.
- [ ] Explain why amounts of money use `Decimal` and not `float`.
- [ ] Tell direct dependencies from transitive ones, and version constraints from locked versions.
- [ ] Add and remove dependencies with uv, and rebuild the environment from the lockfile.
- [ ] Run a command-line program under the debugger with a `launch.json`, stop it at a breakpoint, step through it and inspect its variables.
- [ ] Explain the difference between a linter and a formatter, and run Ruff on a project as a development dependency.

## Before you start

You need the `ledger` project from [Lab 3](../03-python-uv/lab.md): `uv run ledger` prints `Hello from ledger!`. The concepts of [Python projects and environments](../03-python-uv/python.md) are assumed, especially virtual environments.

## Class slides

📄 [Slides (PDF)](slides.pdf)
