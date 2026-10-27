# Software Management & Development

> The web version of this course, with search, is at <https://alberpro.com/ie-smd/>.
> This repository is updated automatically from it.

**IE University** · *Master in Financial Technology (MFT) · Term 1, 2026–27*

In this course you build real software, from an empty laptop to an application running in the cloud. In fifteen sessions you will set up a professional development environment, write a Python backend with a database and a REST API, put a web frontend on top of it, test it, package it in containers, and deploy it to Microsoft Azure. Along the way you will learn the habits that make software teams work: version control, code review, reproducible environments, and small, frequent deliveries.

Every session follows the same pattern. First, a short explanation of the ideas (why the tool exists and what problem it solves). Then a **lab** where you apply them, step by step, to one application that grows during the course.

## The application we build: a personal ledger

The running example is **ledger**, a small fintech application that records accounts and transactions and computes balances and spending by category. It starts in session 3 as a 30-line Python script that reads a CSV file. It ends in session 14 as a three-part system running on Azure: a web dashboard, a REST API and a PostgreSQL database.

```text
 S3–S6   laptop:  python script ──▶ structured package
 S7–S10  laptop:  frontend ──▶ FastAPI ──▶ PostgreSQL (in Docker)
 S11–S12 laptop:  the same three parts, in Docker Compose
 S13–S14 Azure:   Static Web Apps ──▶ Container Apps ──▶ PostgreSQL
```

Your **group project** follows the same path with an application of your own choosing (a budgeting tool, a portfolio tracker, an invoicing system…). Whatever you learn in a lab on the ledger, you then apply to your project.

## Sessions

| # | Session | Material |
|---|---------|----------|
| 1 | [Development environments](topics/dev-environments/index.md) | notes · lab · slides |
| 2 | Linux and the shell | *coming soon* |
| 3 | Professional Python with uv | *coming soon* |
| 4 | Code editors and developer productivity | *coming soon* |
| 5 | Version control and collaboration with Git | *coming soon* |
| 6 | Software design and project structure | *coming soon* |
| 7 | Databases and persistence | *coming soon* |
| 8 | Backend APIs with FastAPI | *coming soon* |
| 9 | Testing, errors and reliability | *coming soon* |
| 10 | Frontend fundamentals and API consumption | *coming soon* |
| 11 | Containers and Docker | *coming soon* |
| 12 | Configuration, environments and secrets | *coming soon* |
| 13 | Cloud fundamentals and container registries (Azure) | *coming soon* |
| 14 | Cloud deployment (Azure) | *coming soon* |
| 15 | Final presentations and final exam | — |

## Local first, then the cloud

We build everything **on your own machine first**. You run the database and the services yourself, with Docker, so you understand every part and can break and repair it at no cost. Only then do we move to the cloud. There, managed services do the same jobs for us, and we pay for them.

We use **Microsoft Azure**. The syllabus describes the cloud part with Google Cloud names. The concepts are the same, and this is the equivalent we use:

| Concept | Google Cloud (used in the syllabus) | Azure (used in this course) |
|---|---|---|
| Private registry for container images | Artifact Registry | Azure Container Registry |
| Serverless containers for the API | Cloud Run | Azure Container Apps |
| Hosting for the web frontend | App Engine | Azure Static Web Apps |
| Managed relational database | Cloud SQL | Azure Database for PostgreSQL |
| Secrets | Secret Manager | Container Apps secrets / Key Vault |

You will use an **Azure for Students** account: it is free with your IE email, gives you credit for a year and needs no credit card. We activate it in session 1.

## Evaluation

| Component | Weight |
|---|---|
| Final exam (multiple choice and written answers) | 30 % |
| Group project: final presentation and live demo | 40 % |
| Individual work: labs and homework | 20 % |
| Class participation | 10 % |

Grades and submissions are always on Blackboard.

## Using AI

Generative AI is **encouraged** in this course. Professional developers use AI assistants every day, and you should learn to use them well. Three rules:

1. **You are responsible for every line you submit.** Assume AI output is wrong until you have run it and understood it. In the exam and in the live demo, you will be asked to explain your code.
2. **Acknowledge it.** State which tool you used, for what, and with which prompts, as the syllabus describes. Acknowledging AI use does not lower your grade.
3. **Learn the fundamentals first.** AI makes a developer who understands the basics much faster. A developer who doesn't understand them, it makes much slower at debugging. The labs are designed for you to type the commands and read the errors yourself.

## Tools

You will install all of these during the course, one session at a time:

- **Terminal** and a Unix shell (bash or zsh)
- **Python** and **uv** (Python versions, dependencies and projects)
- **VS Code** with the Python extension, **Ruff** and the debugger
- **Git** and **GitHub** (repositories, pull requests, Actions)
- **PostgreSQL**, **SQLAlchemy**, **FastAPI** and **pytest**
- **HTML, CSS and JavaScript** in the browser
- **Docker** and **Docker Compose**
- **Azure CLI** (`az`) and the Azure portal
