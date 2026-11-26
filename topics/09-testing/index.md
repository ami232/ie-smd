# Testing and continuous integration

Every time `ledger` changed, you checked by hand that it still worked: you compared the reports with `diff`, sent requests with `curl`, and read the answers. Each check took a few minutes, and you only ran the ones you remembered. A professional team can't work like that: with dozens of changes a day, something always breaks without anyone noticing. In this session you learn to write **automated tests** with **pytest**, small programs that check your code in a second, for the calculations, for storage on a database in memory, and for the web API through HTTP requests. Then you set up **continuous integration** with GitHub Actions, so that GitHub runs Ruff and the tests on every pull request, and you protect `main` so that nothing reaches it with a failing check.

## The thread of the session

1. **[Testing and continuous integration](notes.md).** Why automated tests, pytest, arrange-act-assert, what to test, fixtures, a database in memory, testing errors, testing an API with `TestClient`, reading a failing test, kinds of tests, continuous integration with GitHub Actions, and requiring the checks on `main`.
2. **[Lab: tests and CI for the ledger](lab.md).** You add pytest and a `tests/` folder, write tests for the reports, for storage and for the API, see a test catch a bug, add a GitHub Actions workflow, protect `main` so that the checks must pass, watch a pull request go red and green, and tag your repository for checkpoint 3.

## Objectives

By the end of the session you should be able to:

- [ ] Explain what an automated test is, and what it gives a team that manual checks don't.
- [ ] Write tests with pytest following arrange, act, assert, and run them with `uv run pytest`.
- [ ] Choose what to test in a function: the usual case, the edge cases and the errors.
- [ ] Share test data with fixtures, and test code that uses a database on a database in memory.
- [ ] Check that code raises the right exception with `pytest.raises`.
- [ ] Test the endpoints of a FastAPI application with `TestClient`, checking status codes and JSON.
- [ ] Read the output of a failing test and find the cause.
- [ ] Explain what continuous integration is, and write a GitHub Actions workflow that runs Ruff and the tests on every pull request.
- [ ] Protect `main` with a rule that requires a pull request and a passing check.

## How to study this session

| Page | What you do |
|---|---|
| [Testing and continuous integration](notes.md) | read, with the exercises |
| [Lab: tests and CI for the ledger](lab.md) | hands-on |

## Before you start

You need your repository `smd-app` with the web API of `ledger` merged into `main` (see [Lab: a web API for the ledger](../08-fastapi/lab.md)). If it isn't finished, finish it first: this lab tests that API.

## Class slides

📄 [Slides (PDF)](slides.pdf)
