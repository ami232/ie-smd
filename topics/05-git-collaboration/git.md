# Version control with Git

> ℹ️ **What you need first**
>
> The shell from [Linux and the shell](../02-linux-shell/shell.md): moving between folders, hidden files (`ls -a`) and running commands. The `ledger` project from [Lab 4](../04-dependencies-tools/lab.md), and which of its files describe the project and which are generated, from [Dependencies and the lockfile](../04-dependencies-tools/dependencies.md).

## Why version control

Your `ledger` folder has changed every session: a shell script, then a Python project, then real code, a dependency and a lockfile. If tomorrow `uv run ledger spending` stopped working, could you say what changed since it last worked? Could you go back to Friday's version? Most people's first answer is to copy the folder: `ledger_old`, `ledger_v2`, `ledger_final`, `ledger_final_REALLY`. That works for a day. After a month nobody knows which copy is which, what changed between two of them, or why.

Now add a team. Four people edit the same project. Ana changes `read_transactions` while Ben adds a command to `main`. They email files to each other, and one of them overwrites the other's work without noticing. On a real team, with real money in the data, this is how bugs reach production and nobody can say when they arrived.

A **version control system** solves both problems. It records every saved state of the project with who saved it, when and why, it lets several people work on the same files at the same time, and it combines their work, warning when two changes collide. **Git** is the version control system almost everyone uses today, and **GitHub** is the most popular website for hosting Git projects and working on them as a team.

## Commits: snapshots of the project

Git records the history of a project as a sequence of **commits**. Think of a commit as a photograph of every file of the project at one moment, with a caption: the author, the date and a message saying what changed and why. You decide when to take the photograph, typically after each small piece of work that makes sense on its own ("add the spending command", "store amounts as Decimal").

> 📘 **Definition: repository and commit**
>
> A **repository** (*repo*) is a project folder whose history Git tracks. The history is stored in a hidden folder, `.git`, at the top of the project.
>
> A **commit** is a snapshot of all the tracked files of the repository, together with its author, date, message and a reference to the previous commit (its *parent*). Each commit is identified by a **hash**, a 40-character code computed from its contents, such as `7259904e1c…`, usually shortened to its first 7 characters.

Because each commit points to its parent, the commits form a chain: the **history**. `git log` shows it, newest first. Because the hash is computed from the contents, including the parent's hash, nobody can change an old commit without changing every hash after it. Git notices, which is what makes the history trustworthy.

Commits are cheap. Git stores only the files that changed between snapshots and compresses them, so committing often costs almost nothing.

### The three areas

Taking a snapshot is a two-step process, and this is the part of Git that confuses people most at first. Your files live in three places:

1. The **working tree**: the files in the folder, as you edit them.
2. The **staging area** (also called the *index*): the changes you have selected for the next commit.
3. The **repository**: the commits already recorded in `.git`.

`git add` copies a change from the working tree to the staging area. `git commit` records everything in the staging area as a new commit. `git status` tells you, at any moment, what is changed but not staged, what is staged, and what Git doesn't track at all.

```text
 working tree  ──git add──▶  staging area  ──git commit──▶  repository
  (your files)               (next commit)                 (history in .git)
```

Why two steps? Because you often change several things at once and want to record them as separate commits. If you fixed a bug in `totals_by` and also updated the README, `git add src/ledger/__init__.py` and a commit "fix totals for empty files", then `git add README.md` and a commit "document the spending command", gives a history where each commit has one purpose. When something breaks later, a history like that tells you which change caused it.

### Worked example

You have just edited the README of a repository. These are the commands and what each one shows:

```bash
git status                   # README.md is listed as "modified", in red: changed, not staged
git diff                     # the changed lines: - for removed, + for added
git add README.md
git status                   # README.md is now under "Changes to be committed", in green
git commit -m "Describe what the app does"
git log --oneline            # the new commit is at the top, with its short hash
```

The message after `-m` is for people: write what the commit does and, if it isn't obvious, why. "Describe what the app does" helps someone reading the history in three months. "changes" or "asdf" doesn't.

## What goes in the repository

A repository should contain everything needed to rebuild the project, and nothing that can be rebuilt from it or that must stay private. For `ledger`:

| Commit | Don't commit |
|---|---|
| `pyproject.toml`, `uv.lock`, `.python-version` | `.venv/`: generated by `uv`, specific to your machine |
| `src/`, `README.md` | `__pycache__/`, `.ruff_cache/`: generated by Python and Ruff |
| sample data the code needs (`data/transactions.csv`) | `.env` files with passwords or keys (session 12) |

A file called **`.gitignore`**, at the top of the repository, lists the patterns of files Git must ignore. A line `.venv/` makes Git ignore the whole folder: `git status` doesn't show it, and `git add .` doesn't add it. GitHub offers ready-made `.gitignore` templates for each language when you create a repository.

> ⚠️ **Secrets in Git are forever**
>
> If you commit a password or an API key, deleting the file in the next commit doesn't remove it: it is still in the history, and anyone with a copy of the repository can find it. Public repositories on GitHub are scanned by bots looking for keys within minutes. If it happens, the only safe fix is to **revoke the key** and create a new one. Session 12 explains how to keep secrets out of the code.

## Local and remote repositories

So far, everything happens on your laptop: Git doesn't need a server. To share the project, and to keep a copy that survives a broken laptop, you connect your repository to a **remote**: a copy of the same repository on a server, usually GitHub.

> 📘 **Definition: remote, clone, push and pull**
>
> A **remote** is another copy of the repository that yours knows about, by a name and a URL. The default name is **`origin`**.
>
> - `git clone <url>` downloads a repository with its whole history and sets it up as `origin`.
> - `git push` sends your new commits to the remote.
> - `git pull` downloads the new commits from the remote and merges them into your current branch. It is two operations in one: `git fetch` (download) followed by `git merge` (combine).

Your local repository and the remote are independent copies: a commit you make on your laptop is not on GitHub until you push it, and a change someone makes on GitHub is not on your laptop until you pull it. `git status` tells you when your branch is "ahead of 'origin/main' by 2 commits" (you have commits to push) or "behind" (there are commits to pull).

GitHub needs to know who you are before it accepts a push. With the **GitHub CLI** (`gh`), `gh auth login` signs you in once through the browser and configures Git to use that login, so you don't type passwords or manage keys.

## Branches

On a team, several pieces of work are in progress at the same time: Ana's new command, Ben's bug fix, a teammate's experiment that may never be finished. If everyone committed to the same line of history, each person's half-finished work would break the project for the others. **Branches** let each piece of work have its own line of history until it is ready.

> 📘 **Definition: branch**
>
> A **branch** is a movable name that points to a commit. When you commit on a branch, the branch moves forward to the new commit. **`main`** is the branch that holds the project's official history. **`HEAD`** is the branch you are on now.

A branch costs almost nothing: it is a name and a 40-character hash. The usual workflow is to create a branch for each piece of work, commit on it, and bring it back into `main` when it's done and reviewed:

```bash
git switch -c feature/monthly-report   # create the branch and move to it
# ...edit, add, commit, as many times as needed...
git push -u origin feature/monthly-report   # publish the branch on GitHub
git switch main                          # go back to main
git branch                               # list the local branches; * marks the current one
```

`-u` (short for `--set-upstream`) links the local branch to the one on GitHub, so that from then on a plain `git push` or `git pull` on this branch knows where to go. Branch names are free text; prefixes like `feature/` and `fix/` are a common convention that makes them easier to read.

## Merging and conflicts

When the work on a branch is finished, you **merge** it: you bring its commits into another branch, usually `main`. There are two cases.

If `main` has not moved since the branch was created, Git just moves `main` forward to the branch's last commit. This is a **fast-forward**: no new commit is needed. It is also what usually happens when you `git pull` and only the remote has new commits.

If both branches have new commits, Git creates a **merge commit**, a commit with two parents that combines both lines of work. Git combines the changes on its own, line by line, as long as they touch different parts of the files. Two people editing different functions of the same file is fine.

```text
        A───B───C        main
             \
              D───E      feature/monthly-report

git switch main
git merge feature/monthly-report

        A───B───C───M    main      M has two parents: C and E
             \     /
              D───E
```

A **conflict** happens when both branches changed **the same lines** of the same file. Git can't know which version is right, so it stops the merge and asks you. It writes both versions into the file, between markers:

```text
<<<<<<< HEAD
Personal finance ledger: balances, spending and monthly reports.
=======
Personal finance ledger for the command line.
>>>>>>> main
```

The part between `<<<<<<< HEAD` and `=======` is your version (the branch you are on). The part between `=======` and `>>>>>>>` is the incoming version. To resolve the conflict, you edit the file so it contains what it should contain (one version, the other, or a combination), delete the three marker lines, then `git add` the file and `git commit` to finish the merge. VS Code highlights the conflicts and offers buttons (*Accept Current*, *Accept Incoming*, *Accept Both*), but the decision is always yours: Git can't know what the text means.

A conflict is not an error. It is Git refusing to guess. The usual cause is two people working on the same part of the code at the same time, and the best prevention is small branches that are merged often.

## Pull requests and code review

On GitHub, a branch doesn't usually go into `main` with a plain `git merge`. Instead, you open a **pull request** (PR): a proposal to merge a branch, with a title and a description of what it changes and why. GitHub shows the differences, and teammates **review** them: they read the code, comment on specific lines, ask for changes or approve. When the PR is approved, someone clicks **Merge**, and GitHub makes the merge on the server.

Code review is how teams keep quality up and spread knowledge: at least two people have read every change before it reaches `main`, and both of them understand it. In the group project, every change to `main` goes through a reviewed pull request.

GitHub offers three ways to merge a PR. **Create a merge commit** keeps every commit of the branch and adds a merge commit, so the history shows exactly what happened. *Squash and merge* combines all the commits of the branch into one. *Rebase and merge* copies the commits onto `main` without a merge commit. In this course we use **merge commits**, because the history is part of what we assess.

## Common mistakes

- **Committing without checking `git status` first.** You commit files you didn't mean to (`.venv`, a `.env` with a password) or forget the ones you did. Run `git status` before every commit, and `git diff --staged` to see exactly what you're about to record.
- **Working on `main` for everything.** It works alone, but on a team every half-finished change breaks `main` for everybody. Create a branch per piece of work.
- **Forgetting to pull.** If you start working without pulling, you build on an old version and run into conflicts that didn't need to happen. Pull before you start.
- **Leaving conflict markers in a file.** After a conflict, search the file for `<<<<<<<` before committing. A file with markers is not valid Python, Markdown or anything else.
- **Rewriting shared history.** Commands such as `git push --force` replace the history on the remote. On a branch other people use, they delete their work. Don't use them until you know exactly what they do.

> ✅ **Key takeaways**
>
> - A **commit** is a snapshot of the project with an author, a date, a message and a parent. The chain of commits is the history, and each commit is identified by its hash.
> - Changes go from the **working tree** to the **staging area** (`git add`) and from there to the **repository** (`git commit`). `git status` shows where each change is.
> - `.gitignore` keeps generated files and secrets out of the repository. A secret that was committed must be revoked.
> - `clone`, `push` and `pull` copy commits between your repository and a **remote** such as GitHub. `pull` is `fetch` plus `merge`.
> - A **branch** is a movable name for a line of work. Merging combines branches; a **conflict** happens when both changed the same lines, and you resolve it by editing the file. A **pull request** proposes a merge and is where code review happens.

## Exercises

**1.** You edited `src/ledger/__init__.py` and `README.md`, then ran `git add README.md`. What does `git status` show for each file? If you now run `git commit -m "..."`, which changes go into the commit?

<details>
<summary>✅ <b>Solution</b></summary>

`README.md` appears under **Changes to be committed** (staged), and `src/ledger/__init__.py` under **Changes not staged for commit**. The commit records only the README change. The change to `__init__.py` stays in the working tree, still modified, until you `git add` it and commit again.

</details>

**2.** After a teammate's commit, `git status` says "Your branch is behind 'origin/main' by 1 commit, and can be fast-forwarded". What does that mean, and what happens when you run `git pull`?

<details>
<summary>✅ <b>Solution</b></summary>

The remote `main` has one commit that your local `main` doesn't, and you have no commits of your own that the remote lacks. `git pull` downloads the commit and moves your `main` forward to it (a fast-forward). No merge commit is created and there can be no conflict.

</details>

**3.** Your branch changed line 3 of the README, and `main` changed line 10 of the same file. Do you get a conflict when you merge `main` into your branch? And if both changed line 3?

<details>
<summary>✅ <b>Solution</b></summary>

Different lines: no conflict. Git combines both changes on its own and creates a merge commit. Same line: conflict. Git stops, writes both versions of line 3 between `<<<<<<<`, `=======` and `>>>>>>>` markers, and waits for you to edit the file, `git add` it and `git commit`.

</details>

**4.** A teammate committed a file `.env` with the password of the team's database, then deleted it in the next commit and pushed both. Is the password safe now? What should the team do?

<details>
<summary>✅ <b>Solution</b></summary>

No. The first commit is still in the history, on GitHub and in every clone, and anyone can read the file from it (`git show <hash>:.env`). The team must **change the password** of the database at once (revoke the old one), then add `.env` to `.gitignore` so it isn't committed again. Rewriting the history to remove the file is possible, but it doesn't help with copies that already exist, so changing the password is the step that matters.

</details>

[← Back to the session](index.md) [Next: Lab 5 →](lab.md)
