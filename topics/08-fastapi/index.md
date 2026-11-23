# Backend APIs with FastAPI

`ledger` keeps its data in a database, but only a person sitting at a terminal can use it. A web page, a mobile app or another company's system can't type `uv run ledger balance`. They need an **API**: a way for programs to ask `ledger` for data and send it new data over the network. In this session you learn how programs talk on the web with **HTTP**, how to design an API around **resources** following the **REST** style, and how to build one in Python with **FastAPI**, which checks every request for you and writes the API's documentation from your code. Then you add a web API to `ledger`, next to the command line, on the same database.

## The thread of the session

1. **[Web APIs with FastAPI](notes.md).** HTTP requests and responses, methods and status codes, JSON, REST resources and CRUD, FastAPI path operations, path and query parameters, request bodies with Pydantic, errors with `HTTPException`, one database session per request, and the interactive documentation.
2. **[Lab: a web API for the ledger](lab.md).** You add a module `api.py` with endpoints for accounts, transactions and reports, run the development server, try the API from the browser and with `curl`, and write the error handling (404 and 400) and the filters yourself. The command line and the API share the database.

## Objectives

By the end of the session you should be able to:

- [ ] Describe an HTTP request (method, path, query, headers, body) and an HTTP response (status code, headers, body).
- [ ] Choose the right method (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`) and status code (`200`, `201`, `204`, `400`, `404`, `422`, `500`) for an operation.
- [ ] Design the URLs of an API around resources, and map create, read, update and delete (CRUD) to HTTP methods.
- [ ] Write FastAPI endpoints with path parameters, query parameters and a request body declared as a Pydantic model.
- [ ] Tell a request that has the wrong shape (422, checked by Pydantic) from one that breaks a business rule (400, checked by your code), and return each with a clear message.
- [ ] Open one database session per request with a dependency, and explain why the API module contains no SQL and no calculations.
- [ ] Run an API with a development server and try it from its interactive documentation and with `curl`.

## How to study this session

| Page | What you do |
|---|---|
| [Web APIs with FastAPI](notes.md) | read, with the exercises |
| [Lab: a web API for the ledger](lab.md) | hands-on |

## Before you start

You need your repository `smd-app` with `ledger` in a SQLite database through SQLAlchemy, merged into `main` (see [Lab: the ledger in a database](../07-databases/lab.md)). If it isn't finished, the lab explains how to start from the reference version.

## Class slides

📄 [Slides (PDF)](slides.pdf)
