# Users, packages and the environment

> ℹ️ **What you need first**
>
> Moving around the filesystem and running commands, from [The shell and the filesystem](shell.md).

## Three questions every system answers

When you run a program on a Unix system, three things decide what happens before your code does anything at all:

1. **Who is running it?** The user determines which files the program can read, write or execute.
2. **What is installed?** The program can only use the tools and libraries that are on the system, in the versions that are there.
3. **What is its environment?** A set of named values, the environment variables, tells it where to find things and how to behave.

These are layers 2, 3 and 6 of the environment stack from [session 1](../dev-environments/environments.md#what-an-environment-is-made-of). This page covers each one.

## Users, groups and permissions

### Users and groups

Unix was designed for many people sharing one machine, so every file and every running program belongs to a **user**. Users are collected into **groups**, which let several users share access. The `id` command shows who you are:

```bash
id
```

```text
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),27(sudo),...
```

You are the user `ubuntu`, with number (*uid*) 1000, and you belong to several groups, including `sudo` (more on that below). There is also one special user, **root** (uid 0), the administrator, who can read, change and delete anything.

Servers also have users that are not people. A database server runs as user `postgres`, a web server as `www-data`. If one of them is compromised, the attacker gets only that user's permissions, not the whole machine. You will see the same idea in session 11: well-built containers don't run as root.

### Reading permissions

`ls -l` shows who owns each file and what everyone may do with it:

```bash
ls -l balance.sh /etc/passwd /etc/shadow
```

```text
-rwxr-xr-x 1 ubuntu ubuntu  252 Oct 27 12:12 balance.sh
-rw-r--r-- 1 root   root    888 Oct 24 14:12 /etc/passwd
-rw-r----- 1 root   shadow  502 Oct 24 14:12 /etc/shadow
```

The first column has ten characters. The first says what the entry is (`-` a file, `d` a directory, `l` a link). The other nine are three groups of three, for three kinds of people:

```text
 -   rwx   r-x   r-x
     │     │     └── others: everyone else
     │     └──────── group: members of the file's group (the second name)
     └────────────── owner: the file's user (the first name)
```

In each group, `r` is permission to **read**, `w` to **write** and `x` to **execute** (run the file as a program; for a directory, to enter it). A `-` means the permission is missing.

On a folder the same letters mean slightly different things: `r` lets you list what is inside, `w` lets you create, rename and delete files in it, and `x` lets you enter it with `cd`. A folder with `drwx------` is private: nobody but its owner can even look inside.

So `balance.sh` can be read, changed and run by `ubuntu`, and read and run by everyone else. `/etc/passwd` (the list of users) is readable by everybody but writable only by root. `/etc/shadow`, which contains the password hashes, cannot even be read by normal users:

```bash
cat /etc/shadow
```

```text
cat: /etc/shadow: Permission denied
```

> 📘 **Definition: file permissions**
>
> Each file has an **owner** and a **group**, and three sets of permissions: for the owner, for the group and for **others**. Each set says whether that class of user may **read** (`r`), **write** (`w`) or **execute** (`x`) the file.

### Changing permissions

`chmod` (*change mode*) changes permissions. The readable form says *who*, `+` or `-`, and *what*:

```bash
chmod +x balance.sh      # everyone may execute it
chmod go-r notes.txt     # group and others may no longer read it
chmod u+w report.txt     # the owner (user) may write it
```

You will also see a numeric form, where each set is a digit: `r` = 4, `w` = 2, `x` = 1, added up. `chmod 755 file` means `rwxr-xr-x` (7 = 4+2+1 for the owner, 5 = 4+1 for the others), and `chmod 600 key.pem` means `rw-------`: only the owner can read and write. That last one is the standard for files with secrets, such as SSH keys, and SSH refuses to use a private key that others can read.

The most frequent reason to use `chmod` in this course is a script you have just written. A new file is not executable, so `./balance.sh` gives `Permission denied` until you run `chmod +x balance.sh`.

With `-R` (*recursive*), `chmod` changes a folder and everything inside it: `chmod -R g+w data` lets the group write in `data` and in all its files.

### Changing the owner

`chown` (*change owner*) gives a file to another user, and optionally another group, written `user:group`:

```bash
sudo chown ana report.txt          # ana becomes the owner
sudo chown ana:staff report.txt    # owner ana, group staff
sudo chown -R ana:ana data         # the folder and everything inside it
```

Giving a file away needs `sudo`, explained next: otherwise any user could hand files to someone else. You'll use `chown` mostly to repair files that were created as root by mistake, for example by running a command with `sudo` that didn't need it, and that your own user now can't change.

### sudo

Installing software or changing system configuration requires root. Instead of logging in as root, you run single commands with **`sudo`** (*superuser do*):

```bash
sudo apt install tree
```

`sudo` asks for **your** password (not root's), checks that your user is allowed (on Ubuntu, membership of the `sudo` group; on macOS, of `admin`) and runs that one command as root. Every use is logged.

This applies the **principle of least privilege**: work as a normal user, and become root only for the command that needs it. A typo in a normal command can damage your files; the same typo with `sudo` can damage the whole system. Don't put `sudo` in front of a command just because it failed. Find out *why* it failed first. The answer is often a wrong path, not a missing permission.

## Installing software: package managers

On Windows, people traditionally install software by downloading an installer from a website. Unix systems use a **package manager** instead: a program that downloads software from trusted **repositories**, installs it in the standard places, installs whatever else it depends on, and can update or remove all of it later with one command.

| System | Package manager | Install | Update everything |
|---|---|---|---|
| Ubuntu, Debian (and WSL) | `apt` | `sudo apt install tree` | `sudo apt update && sudo apt upgrade` |
| macOS | Homebrew (`brew`) | `brew install tree` | `brew update && brew upgrade` |
| Windows | `winget` | `winget install Git.Git` | `winget upgrade --all` |

With `apt`, `sudo apt update` refreshes the *list* of available packages and their versions from the repositories. It doesn't install anything. `sudo apt upgrade` installs the new versions. Run `update` before installing anything on a machine you haven't used for a while, or `apt` may try to download versions that no longer exist.

Homebrew installs into its own folder (`/opt/homebrew` on Apple Silicon) and doesn't need `sudo`.

Package managers solve the same problem at the system level that the lockfiles of session 3 solve for a project: known versions, installed in a repeatable way, from a trusted source. You can write down a list of packages and anyone can install exactly that list.

> ⚠️ **Install from trusted sources**
>
> A package manager verifies that what it downloads comes from the repository it claims to come from. A random installer or script from the web has no such check. When documentation tells you to run `curl … | sh` (download a script and run it immediately), it is executing code you haven't read with your user's permissions. Do it only for well-known projects, from their official site.

## Environment variables

### What they are

Every running program receives a set of named text values from whoever started it: its **environment variables**. Programs read them to find out where things are and how to behave. Some are always there:

```bash
echo $HOME      # your home directory
echo $USER      # your user name
echo $SHELL     # your shell
env             # all of them
```

The `$` in front of a name asks the shell to replace it with the variable's value. By convention, environment variable names are in capitals.

> 📘 **Definition: environment variable**
>
> An **environment variable** is a named text value (`NAME=value`) that a process receives when it starts. A child process inherits a copy of its parent's variables. Changes in the child never affect the parent.

### Setting them

There are three ways, with different scopes:

```bash
LEDGER_FILE=data/checking.csv ./balance.sh   # only for this one command
export LEDGER_FILE=data/checking.csv         # for this shell and everything it starts afterwards
unset LEDGER_FILE                            # remove it
```

A variable set with `export` disappears when you close the terminal. To make it permanent, add the `export` line to your shell's startup file: `~/.zshrc` for zsh, `~/.bashrc` for bash. The shell reads that file every time it starts.

This is how Ana's script in [session 1](../dev-environments/environments.md#it-works-on-my-machine) got its `RISK_THRESHOLD`: an `export` line in her `~/.zshrc` that nobody else had. Environment variables are the standard way to configure software, especially in containers and in the cloud. Session 12 shows how to manage them so that this can't happen to you.

### PATH: how the shell finds commands

When you type `python3`, how does the shell know which program to run? It looks in the folders listed in the **`PATH`** variable, in order, and runs the first `python3` it finds:

```bash
echo $PATH | tr ":" "\n"
```

```text
/usr/local/sbin
/usr/local/bin
/usr/sbin
/usr/bin
/sbin
/bin
```

(`tr ":" "\n"` replaces the colons with line breaks so the list is readable.) `which` tells you which file a command name resolves to:

```bash
which python3
```

Two everyday errors come from `PATH`:

- **`command not found`** right after installing something. The program was installed in a folder that isn't in `PATH`. The installer usually says which folder and how to add it. Often you just need to open a new terminal, so that the shell reads its updated startup file.
- **The wrong version runs.** You have two Pythons, and the one that comes first in `PATH` isn't the one you expected. `which python3` and `python3 --version` tell you which one you're getting.

That is also why we ran `./balance.sh` and not `balance.sh`. The current folder isn't in `PATH` (for security), so a program in it must be called with an explicit path.

## Processes

Every running program is a **process**, with a number (*PID*), an owner and its own environment. A few commands are enough to start with:

| Command | Does |
|---|---|
| `ps aux` | lists all processes |
| `top` or `htop` | live view of processes and their CPU and memory use (<kbd>Q</kbd> to quit) |
| `kill PID` | asks a process to stop (`kill -9 PID` forces it) |
| `cmd &` | runs `cmd` in the background, giving you the prompt back |

You will need these when a server you started in one terminal keeps a port busy, or a program hangs.

## Common mistakes

- **`sudo` as a reflex.** It hides the real problem and can create files owned by root in your home folder, which then cause *more* `Permission denied` errors. Use it only for installing software and changing the system.
- **`chmod 777`** "to make it work". It gives every user write permission. It's never the right fix, especially for files with secrets.
- **Spaces around `=`.** `NAME = value` is an error in the shell; it must be `NAME=value`.
- **Forgetting `export`.** `NAME=value` alone (on its own line) sets a *shell* variable that the programs you start don't see. Use `export NAME=value`, or put the assignment in front of the command.
- **Editing `PATH` and losing the old value.** Write `export PATH="$HOME/bin:$PATH"`, which adds a folder, never `export PATH="$HOME/bin"`, which replaces the whole list.

> ✅ **Key takeaways**
>
> - Every file has an **owner**, a **group** and `rwx` permissions for owner, group and others. `chmod` changes them, and `chmod +x` makes a script runnable.
> - **root** can do anything. Work as a normal user and use **`sudo`** only for the single commands that need it (least privilege).
> - **Package managers** (`apt`, `brew`, `winget`) install software from trusted repositories, with its dependencies, in a repeatable way.
> - **Environment variables** configure programs and are inherited by child processes. Set them for one command (`NAME=x cmd`), for a session (`export`) or permanently (`~/.zshrc`, `~/.bashrc`).
> - **`PATH`** lists where the shell looks for commands. `which` shows what a name resolves to, and most `command not found` errors are `PATH` problems.

## Exercises

**1.** Decode these permissions and say who can do what: *(a)* `-rw-r--r--` owned by `ana`; *(b)* `-rwx------`; *(c)* `drwxr-x---` owned by `ana`, group `finance`.

<details>
<summary>✅ <b>Solution</b></summary>

- *(a)* A file that `ana` can read and write, and that everyone else can only read. This is the usual default for a new file.
- *(b)* A file that only its owner can read, write and execute. Nobody else can do anything with it: typical for a private script.
- *(c)* A directory (the `d`). `ana` can list it, create files in it and enter it. Members of `finance` can list it and enter it, but not create or delete files. Others can't even enter it.

</details>

**2.** Write the numeric `chmod` for: *(a)* `rwxr-xr-x`; *(b)* `rw-------`; *(c)* `rw-r-----`.

<details>
<summary>✅ <b>Solution</b></summary>

Add up `r` = 4, `w` = 2, `x` = 1 in each group of three:

- *(a)* 7, 5, 5 → `chmod 755`
- *(b)* 6, 0, 0 → `chmod 600`
- *(c)* 6, 4, 0 → `chmod 640`

</details>

**3.** You install a tool following its instructions, but typing its name gives `command not found`. The installer printed "installed to `~/.local/bin`". What is happening, and how do you fix it?

<details>
<summary>✅ <b>Solution</b></summary>

The shell looks for commands only in the folders listed in `PATH`, and `~/.local/bin` isn't one of them. First, open a new terminal: many installers add the folder to your startup file, and only new shells read it. If it still fails, check with `echo $PATH | tr ":" "\n"`. If the folder is missing, add `export PATH="$HOME/.local/bin:$PATH"` at the end of `~/.zshrc` (or `~/.bashrc`) and open a new terminal.

</details>

**4.** In one terminal you run `export API_KEY=abc123`. In a second terminal window, `echo $API_KEY` prints an empty line. Why?

<details>
<summary>✅ <b>Solution</b></summary>

An environment variable belongs to a process and is inherited only by the processes **it** starts. Each terminal window runs a separate shell, and neither is a child of the other, so the second one never received the variable. To have it in every shell you would put the `export` in your startup file. For a secret like an API key, though, that's a bad idea: session 12 shows better ways.

</details>

**5.** In the lab, `balance.sh` ends with `awk … "$file" | sort`. When `LEDGER_FILE` points to a file that doesn't exist, awk prints an error, but `echo $?` right after shows `0`. Why? Why does it matter?

<details>
<summary>✅ <b>Solution</b></summary>

The exit code of a pipeline is the exit code of its **last** command. `awk` failed, but `sort` successfully sorted its empty input and returned `0`. That matters because automated tools (scripts, CI pipelines, schedulers) decide whether a step worked by its exit code. This script would report success while producing nothing, which is Ana's dangerous kind of failure again. In bash, `set -o pipefail` at the top of a script makes a pipeline fail if any of its commands fails. It is a good habit, together with `set -e` (stop at the first failing command) and `set -u` (treat an undefined variable as an error).

</details>

[← Previous: The shell and the filesystem](shell.md) [Next: Setting up the course environment →](setup.md)
