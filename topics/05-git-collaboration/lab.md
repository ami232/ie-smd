# Lab 5: your app's repository

> ℹ️ **What you need first**
>
> The `ledger` project from [Lab 4](../04-dependencies-tools/lab.md) in `~/smd/ledger`, your GitHub account from [Lab 1](../01-dev-environments/lab.md#5-create-your-github-account), and the concepts of [Version control with Git](git.md). If your ledger isn't finished, you can start from the reference version, as explained in [step 4](#4-add-the-ledger-and-commit).

> ⛔ **Do this lab by hand, without AI**
>
> In this lab, **AI assistants are not allowed**: type every command yourself and read what Git answers. Don't ask a chatbot for the commands, and turn off inline suggestions in VS Code for today (if you use Copilot: click the Copilot icon in the status bar and disable completions).
>
> Git is the safety net for everything else you do in this course, including the code you will write with AI: when an assistant breaks your project, Git is how you get it back. You can only trust a safety net you understand. Once you know what each command does, you can use AI for Git as for the rest of the course, under the rules in [Using AI](../../README.md#using-ai).

In this lab you create the repository where you will build your application for the rest of the course, and you go through the whole Git workflow once, on purpose: commits, pushing and pulling, a change made on GitHub, a branch, a conflict and a pull request. The repository's history is the proof that you did each step, so do them in order and don't rewrite it afterwards.

**Submission.** At the end, you submit the URL of your repository on Blackboard, **by Friday 13 November, 23:59**. The [checkpoint](#checkpoint) lists what is checked.

## 1. Check Git and sign in to GitHub

Work in the course shell (the macOS Terminal, the terminal on Linux or inside your VM; the Ubuntu terminal if you use WSL). Check that Git is installed:

```bash
git --version
```

If you get `command not found`: on macOS, run `xcode-select --install`; on Linux and in the VM, `sudo apt install -y git`.

Install the **GitHub CLI**, `gh`, and sign in. It configures Git to use your GitHub login, so you never type a password or manage keys:

**macOS**

```bash
brew install gh
```


**Linux, VM, WSL**

```bash
sudo apt update
sudo apt install -y gh
```


```bash
gh auth login
```

Answer the questions with the arrow keys and <kbd>Enter</kbd>: **GitHub.com**, **HTTPS**, **Yes** (authenticate Git with your GitHub credentials), **Login with a web browser**. `gh` prints a one-time code; press <kbd>Enter</kbd>, paste the code in the browser page that opens, and authorize. Then check:

```bash
gh auth status
```

<details>
<summary>✅ <b>What you should see</b></summary>

`Logged in to github.com account <your-username>`, with `Git operations protocol: https`. If the browser doesn't open (some VMs), open [github.com/login/device](https://github.com/login/device) yourself and type the code.

</details>

## 2. Configure Git

Git records an author in every commit. Tell it your name and email, once per computer:

```bash
git config --global user.name "Your Name"
git config --global user.email "ID+username@users.noreply.github.com"
git config --global init.defaultBranch main
git config --global pull.rebase false
git config --global core.editor nano
```

- For the email, use your GitHub **no-reply address**: on GitHub, *Settings → Emails*, under "Keep my email addresses private". It looks like `12345678+username@users.noreply.github.com`. Your commits will be public, and this keeps your real address out of them. It also links the commits to your GitHub account.
- `pull.rebase false` tells `git pull` to combine histories with a merge. Without it, recent versions of Git refuse a pull when both sides have new commits, which you will do in step 9.
- `core.editor nano` makes Git open `nano` if it ever needs you to write a message. In `nano`, <kbd>Ctrl</kbd>+<kbd>O</kbd> and <kbd>Enter</kbd> save, <kbd>Ctrl</kbd>+<kbd>X</kbd> exits.

```bash
git config --global --list
```

<details>
<summary>✅ <b>What you should see</b></summary>

Your five settings, among others: `user.name=...`, `user.email=...@users.noreply.github.com`, `init.defaultbranch=main`, `pull.rebase=false` and `core.editor=nano`.

</details>

## 3. Create the repository on GitHub and clone it

On [github.com](https://github.com), click **+ → New repository** (top right) and fill in:

- **Repository name:** `smd-app`.
- **Public.** Your repository is public: it is part of your portfolio, and it is how your work is checked.
- **Add README:** on.
- **Add .gitignore:** choose the **Python** template.
- No license for now.

Click **Create repository**. GitHub makes the first commit for you, with the two files. Now copy the repository to your laptop. On the repository's page, click **Code**, choose **HTTPS** and copy the URL. Then:

```bash
cd ~/smd
git clone https://github.com/<your-username>/smd-app.git
cd smd-app
ls -a
git log
git status
```

<details>
<summary>✅ <b>What you should see</b></summary>

`git clone` prints `Cloning into 'smd-app'...` and creates the folder. `ls -a` shows `.git` (the history), `.gitignore` and `README.md`. `git log` shows one commit, "Initial commit", by you, with its full hash. `git status` says `On branch main`, `Your branch is up to date with 'origin/main'` and `nothing to commit, working tree clean`.

</details>

## 4. Add the ledger and commit

Copy the files that describe and build the ledger project into the repository. These are the files a teammate needs ([Lab 4, step 6](../04-dependencies-tools/lab.md#6-rebuild-from-the-files)); `.venv` is not one of them:

```bash
cp -r ~/smd/ledger/{pyproject.toml,uv.lock,.python-version,src,data} .
uv run ledger balance
```

<details>
<summary>💡 <b>My ledger from Lab 4 is not finished</b></summary>

Use the reference version. On [github.com/ami232/ie-smd-ledger](https://github.com/ami232/ie-smd-ledger), switch to the tag **`s05-start`** (button **main → Tags**), click **Code → Download ZIP**, unzip it, and copy the same five items from the unzipped folder instead of from `~/smd/ledger`. Don't copy its `README.md` or `.gitignore`: your repository has its own.

</details>

`uv run` creates `.venv` inside the repository. Look at what Git sees:

```bash
git status
```

<details>
<summary>✅ <b>What you should see</b></summary>

Under **Untracked files**: `.python-version`, `data/`, `pyproject.toml`, `src/` and `uv.lock`. There is no `.venv`: the Python `.gitignore` from GitHub lists it, so Git ignores it. `ledger balance` printed `3349.10` and `906.00`, as in Lab 4.

</details>

Stage the files, check exactly what will be recorded, and commit:

```bash
git add .
git status
git diff --staged --stat
git commit -m "Add the ledger project from Lab 4"
git log --oneline
```

<details>
<summary>✅ <b>What you should see</b></summary>

After `git add .`, the five items are under **Changes to be committed**. `git diff --staged --stat` lists the files and how many lines each adds. The commit prints its short hash and `5 files changed` or similar. `git log --oneline` shows two commits, newest first.

</details>

Now a second commit, for the README. Open the repository in VS Code (`code .`) and replace the contents of `README.md` with this, keeping the exact second line, which you will need in step 9:

```markdown
# smd-app

Personal finance ledger: balances and spending from a CSV file.

## Run it

    uv run ledger balance
    uv run ledger spending
```

Save it and commit it from the terminal:

```bash
git status
git diff
git add README.md
git commit -m "Describe the app and how to run it"
```

<details>
<summary>✅ <b>What you should see</b></summary>

`git diff` shows the old README lines with `-` and the new ones with `+`. After the commit, `git status` says `Your branch is ahead of 'origin/main' by 2 commits`: you have two commits that GitHub doesn't have yet.

</details>

## 5. Push

```bash
git push
git status
```

<details>
<summary>✅ <b>What you should see</b></summary>

`git push` prints a few lines ending in `main -> main`. `git status` says `Your branch is up to date with 'origin/main'`. Refresh the repository's page on GitHub: the files are there, the README shows its new text, and the commit count says 3 commits.

</details>

## 6. Change something on GitHub

Changes can also come from GitHub: a teammate's work, or an edit in the browser. Make one. On the repository's page, open `README.md`, click the **pencil** icon (*Edit this file*), and add at the end:

```markdown
## Author

<your-github-username>
```

Click **Commit changes…**, keep "Commit directly to the main branch", write a message such as `Add author` and confirm. Your laptop doesn't have this commit yet.

## 7. Pull

```bash
git status
git pull
git log --oneline
cat README.md
```

<details>
<summary>✅ <b>What you should see</b></summary>

The first `git status` still says `up to date with 'origin/main'`: Git only knows about the remote what it saw at the last push or pull. `git pull` downloads the new commit and reports `Fast-forward`, since only GitHub had new work. `git log` shows four commits, the newest being yours from the browser, and the README now ends with the Author section.

</details>

## 8. Create a branch, commit and push it

Work on a new feature in its own branch. Create it and move to it:

```bash
git switch -c feature/roadmap
git branch
```

In `README.md`, make two changes:

1. Change the second line (the description) to:

    ```markdown
    Personal finance ledger: balances, spending and monthly reports from a CSV file.
    ```

2. Add this section before **Author**:

    ```markdown
    ## Roadmap

    - Monthly report of income and spending
    - A database instead of a CSV file
    - A web API and a dashboard
    ```

Save, commit and publish the branch:

```bash
git add README.md
git commit -m "Add a roadmap"
git push -u origin feature/roadmap
```

<details>
<summary>✅ <b>What you should see</b></summary>

`git branch` lists `main` and `* feature/roadmap` (the `*` is where you are). The push creates the branch on GitHub: the output includes `* [new branch] feature/roadmap -> feature/roadmap` and a link to create a pull request (not yet). On GitHub, the branch selector (the **main** button) now lists `feature/roadmap`.

</details>

## 9. Create a conflict and resolve it

While you were working on the branch, someone changed `main`. Do it yourself: on GitHub, make sure the branch selector says **main**, open `README.md`, click the pencil, and change the **same second line** to:

```markdown
Personal finance ledger for the command line.
```

Commit it directly to `main`. Now `main` and your branch have both changed the same line. Bring `main`'s new work into your branch:

```bash
git pull origin main
```

<details>
<summary>✅ <b>What you should see</b></summary>

```text
Auto-merging README.md
CONFLICT (content): Merge conflict in README.md
Automatic merge failed; fix conflicts and then commit the result.
```

Git combined everything it could and stopped at the line both sides changed. `git status` says `You have unmerged paths` and lists `README.md` under **Unmerged paths**.

</details>

Open `README.md`. The second line is now:

```text
<<<<<<< HEAD
Personal finance ledger: balances, spending and monthly reports from a CSV file.
=======
Personal finance ledger for the command line.
>>>>>>> 1a2b3c4d5e6f...
```

`HEAD` is your branch's version; the other one comes from `main` on GitHub (the long code is its commit hash). Resolve it **by editing the text yourself**: replace the five lines, markers included, with one description that keeps both ideas:

```markdown
Personal finance ledger for the command line: balances, spending and monthly reports from a CSV file.
```

Check that no marker is left, then finish the merge:

```bash
grep -n "<<<<<<<\|=======\|>>>>>>>" README.md
git add README.md
git commit --no-edit
git log --oneline --graph
git push
```

<details>
<summary>✅ <b>What you should see</b></summary>

`grep` prints nothing: no markers left. `git commit --no-edit` creates the **merge commit** with Git's default message, `Merge branch 'main' of https://github.com/...`. In `git log --oneline --graph`, two lines of history join at that commit: your roadmap commit on one side, the browser edit on the other. `git push` sends the merge to your branch on GitHub.

</details>

## 10. Open and merge a pull request

On GitHub, open the **Pull requests** tab and click **New pull request**. Choose **base: main** and **compare: feature/roadmap**. GitHub shows the differences and `Able to merge`, because you already resolved the conflict on the branch. Click **Create pull request** and fill in:

- **Title:** `Add a roadmap`
- **Description:** two or three sentences: what the branch changes, and how you resolved the conflict in the description line.

Click **Create pull request**. Read the **Files changed** tab, as a reviewer would. Then, in the **Conversation** tab, merge it: check that the green button says **Merge pull request** (if it says *Squash and merge* or *Rebase and merge*, use the arrow next to it to choose **Create a merge commit**), click it and confirm. GitHub offers to delete the branch: you can.

Finally, bring the result to your laptop and tidy up:

```bash
git switch main
git pull
git branch -d feature/roadmap
git log --oneline --graph --all
```

<details>
<summary>✅ <b>What you should see</b></summary>

`git pull` fast-forwards `main` to the merge commit GitHub created (`Merge pull request #1 from <your-username>/feature/roadmap`). `git branch -d` deletes the local branch, which is safe because its commits are now in `main`. The graph shows the whole lab: the initial commit, your two commits, the browser edit, the branch with its roadmap commit, the merge that resolved the conflict, and the merge of the pull request. The README on GitHub has the combined description, the roadmap and the author.

</details>

## Checkpoint

Submit the URL of your repository (`https://github.com/<your-username>/smd-app`) on Blackboard by **Friday 13 November, 23:59**. Don't change the history after submitting: no force pushes, and don't delete the repository. Your repository should show:

- [ ] a public repository with `pyproject.toml`, `uv.lock`, `.python-version`, `src/` and `data/`, and **no** `.venv`;
- [ ] commits made from your laptop, with your name and your no-reply email;
- [ ] a commit made on GitHub's website;
- [ ] a branch other than `main`, pushed to GitHub, with its own commit;
- [ ] on that branch, a merge of `main` that resolved a conflict, with no markers left;
- [ ] a pull request from that branch to `main`, with a description, merged with a merge commit.

This repository is where you build your application from now on. In the next labs you add to it, and from time to time you submit a version of it as a checkpoint.

[← Previous: Version control with Git](git.md) [Back to the session ↑](index.md)
