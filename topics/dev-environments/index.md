# Development environments

Software never runs in a vacuum. Your Python script runs on an interpreter, which runs on an operating system, which runs on a particular kind of processor. The script also reads files, connects to a database and expects some settings to be there. When any of these pieces changes, the same code can behave differently or not run at all. This first session is about that whole stack, the **development environment**: what it is made of, why teams work so hard to make it identical for everyone, and which tools they use to do so (virtual machines, containers and the cloud).

It is also the session where we get to know your machine. The rest of the course depends on it.

## The thread of the session

1. **[Environments and reproducibility](environments.md).** What an environment is, layer by layer, and why "it works on my machine" is one of the most expensive sentences in software. Then the three tools for making environments reproducible (host installation, virtual machines and containers) and where the cloud fits in. It ends with a map of what we will build during the course.
2. **[Lab 1: know your machine](lab.md).** You open a terminal, find out exactly what your laptop is (operating system, processor architecture, memory, disk), create the accounts we need (GitHub and Azure for Students), and write it all down on a [machine card](machine_card.md). You need that card in session 2 to set up the course environment.

## Objectives

By the end of the session you should be able to:

- [ ] List the layers of a development environment and give an example of a bug caused by each one.
- [ ] Explain the difference between running software on the host, in a virtual machine and in a container, and when each one is the right choice.
- [ ] Explain why the processor architecture (`arm64` or `x86_64`) matters when you run and ship software.
- [ ] Describe what changes when an application moves from your laptop to the cloud, and who is responsible for each layer in each case.
- [ ] Open a terminal on your computer and find its operating system, architecture, memory and free disk space.

## Before you start

You need no prior knowledge. Bring your laptop charged and make sure you know your password: you will need it to install software in the coming sessions.

## How to study this session

| Page | What you do | Time |
|---|---|---|
| [Environments and reproducibility](environments.md) | read, exercises | 50 min |
| [Lab 1: know your machine](lab.md) | hands-on in your terminal | 45 min |
| [Machine card](machine_card.md) | download and fill in during the lab | — |

## Class slides

The slides are what we use in the classroom. They summarize the session, but these pages are the complete material.

📄 [Slides (PDF)](slides.pdf)
