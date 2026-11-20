# Software design and project structure

`ledger` works, but all of it is in one file: reading the CSV file, the calculations and the printing are mixed in the same functions. That is fine for 50 lines and a problem for an application that will get a database, a web API and a frontend. In this session you learn how to split a program into parts with one responsibility each, how Python organizes those parts into **modules** and **packages**, and how to represent the data with **dataclasses** instead of dictionaries. Then you **refactor** `ledger` into a package, proving that it still does exactly the same, and add a new command to see what the structure buys you.

## The thread of the session

1. **[Software design and project structure](notes.md).** Separation of concerns and the single responsibility principle, the direction of dependencies, modules and packages, dataclasses and type hints, pure functions, refactoring and code smells, and how to design your own module.
2. **[Lab: from one file to a package](lab.md).** You split `ledger` into `models`, `storage`, `reports` and `cli` on a branch, writing the calculations yourself in a starter package, check with `diff` that the output didn't change, add the `months` command, configure Ruff's rules and merge through a pull request.

## Objectives

By the end of the session you should be able to:

- [ ] Identify the concerns of a program (data, storage, calculations, interface) and decide which module each piece of code belongs to.
- [ ] State the single responsibility principle, and use it to tell whether a function or a module should be split.
- [ ] Explain why dependencies should point from the interface and storage towards the data and the calculations, and recognize a circular import.
- [ ] Organize a Python project as a package of modules with absolute imports, and point its command to the right function in `pyproject.toml`.
- [ ] Define a dataclass with typed fields, and explain what `frozen=True` and type hints do and don't do.
- [ ] Tell a pure function from one with side effects, and explain why the calculations should be pure.
- [ ] Refactor code without changing its behaviour, and check it by comparing the output before and after.
- [ ] Choose Ruff's rules for a project and fix what they report.

## How to study this session

| Page | What you do |
|---|---|
| [Software design and project structure](notes.md) | read, with the exercises |
| [Lab: from one file to a package](lab.md) | hands-on |

## Before you start

You need your repository `smd-app` on GitHub, with the ledger in it (see [Lab: your app's repository](../05-git-collaboration/lab.md)). If you don't have it, the lab explains how to start from the reference version.

## Class slides

📄 [Slides (PDF)](slides.pdf)
