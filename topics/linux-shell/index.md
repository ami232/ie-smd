# Linux and the shell

Almost all the software you will deploy in your career runs on Linux. That includes the servers behind banking apps, the containers in this course and the machines that run Azure Container Apps. Almost all of it is set up and operated from a **shell**: a text interface where you type commands. Graphical interfaces come and go, but the shell is the same on your laptop, in a container and on a server on the other side of the world, and it is the only interface many of those machines have.

This session gives you a working knowledge of both: how a Unix system is organized, and how to get things done in its shell. It is also the session where everybody sets up the course environment, using the [machine card](../dev-environments/machine_card.md) you filled in during session 1.

## The thread of the session

1. **[The shell and the filesystem](shell.md).** The Unix philosophy (small tools that do one thing, combined with pipes), how the filesystem is organized, and the twenty or so commands that cover most daily work.
2. **[Users, packages and the environment](system.md).** Who can do what (users, groups, permissions, `sudo`), how software is installed (package managers), and the environment variables that configure every program, starting with `PATH`.
3. **[Setting up the course environment](setup.md).** The setup we use for the rest of the course, with installation steps for each operating system.
4. **[Lab 2: a ledger in the shell](lab.md).** You create the project folder for the course application, download its first dataset, answer questions about the transactions with pipelines, and write your first shell script.

## Objectives

By the end of the session you should be able to:

- [ ] Navigate the filesystem with absolute and relative paths, and create, copy, move and delete files and folders from the shell.
- [ ] Answer simple questions about a text file by combining `grep`, `cut`, `sort`, `uniq`, `wc` and `awk` with pipes and redirections.
- [ ] Read and change file permissions, and explain when and why to use `sudo`.
- [ ] Install software with your system's package manager.
- [ ] Explain what an environment variable is, set one for a command or for a session, and diagnose a "command not found" error with `PATH`.
- [ ] Have the course environment installed and working.

## Before you start

You need the terminal and the "machine card" from [Lab 1](../dev-environments/lab.md). The concepts of [environments and reproducibility](../dev-environments/environments.md) are useful, especially the difference between a VM and a container.

## Class slides

📄 [Slides (PDF)](slides.pdf)
