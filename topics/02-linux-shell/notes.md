# The shell and the system


> ℹ️ **What you need first**
>
> A terminal open on a Unix shell: macOS Terminal, Linux, the terminal of your Linux VM, or an Ubuntu container. [Know your machine](../01-dev-environments/lab.md) shows how to open one. Try every command on this page as you read: the shell is learned with the fingers.

## Why a text interface

Suppose your manager asks: *how much did we spend on restaurants last quarter, and with which merchant the most?* The transactions are in a CSV file. With a spreadsheet you open it, filter a column, select cells, read the sum, sort another column. It works, but there is no record of what you did, and next quarter you do it all again by hand.

In the shell, the same question is one line:

```bash
grep ",restaurants," transactions.csv | cut -d, -f3 | sort | uniq -c | sort -rn
```

```text
   5 Rodilla
   4 La Tagliatella
   2 Five Guys
```

You can read the line from left to right. It keeps the restaurant rows, takes the third column (the merchant), sorts it, counts repeated values and sorts by the count. The line *is* the record of what you did. You can save it, share it, run it again next quarter, or run it on a server that has no screen at all. That is the reason the shell has survived fifty years of graphical interfaces: **commands are text, and text can be saved, repeated, combined and automated**.

## Terminal, shell and other words

A few words appear constantly in this course, and they are often used loosely. The two most often confused are *terminal* and *shell*, and they are different programs. The **terminal** is the window: it shows text and sends what you type to the program running inside it. The **shell** is the program inside the window that reads your command, runs it and prints the result. A useful comparison is a phone call: the terminal is the phone, and the shell is the person on the other end who carries out your requests. You can change one without the other. macOS Terminal, Windows Terminal and the terminal panel in VS Code are all terminals, and any of them can run zsh, bash or PowerShell.

| Word | What it means | Examples |
|---|---|---|
| **Terminal** (terminal emulator) | the window where you type text and read the output | macOS Terminal, Windows Terminal, the VS Code terminal |
| **Shell** | the program that reads your commands and runs them | zsh, bash, PowerShell |
| **Command line**, **CLI** | working by typing commands, as opposed to a graphical interface (GUI). A *CLI tool* is a program meant to be used this way | `git`, `docker`, `uv` |
| **Prompt** | the text the shell prints when it is waiting for a command | `ana@laptop ~ %`, `ubuntu@smd:~$` |
| **Command** | one instruction: a program name plus its options and arguments | `ls -la data` |
| **Console** | historically, the physical screen and keyboard of a computer. Today mostly a synonym of terminal | |
| **Script** | a text file of commands that the shell runs one after another | `balance.sh` |
| **Process** | a program while it is running. Every command you type starts at least one | |
| **Working directory** | the folder the shell is "in" right now. Relative paths start from it | `/home/ana/smd` |
| **Home folder** | your personal folder, written `~` | `/home/ana`, `/Users/ana` |
| **Kernel** | the core of the operating system: it manages memory, processes, files and devices | Linux, Darwin (macOS) |
| **Unix** | the operating system from the 1970s whose design Linux and macOS follow, which is why their shells work the same way | |
| **Linux**, **distribution** | Linux is strictly a kernel. A distribution packages it with a shell, tools and a package manager into a complete system | Ubuntu, Debian, Fedora |

When this course says "open a terminal", it means: open the terminal window, where a Unix shell (zsh or bash) shows you its prompt.

## The Unix philosophy

Unix, the operating system Linux and macOS descend from, was designed around a few ideas that still explain how the shell works:

- **Each program does one thing well.** `sort` sorts lines. `wc` counts them. `grep` keeps the lines that match a pattern. None of them tries to do everything.
- **Programs work together.** The output of one is the input of the next. A program doesn't care whether its input comes from a file, the keyboard or another program.
- **Text is the universal interface.** Programs read and write lines of plain text, so any program can be combined with any other.

The `grep … | cut … | sort | uniq -c | sort -rn` line above is this philosophy in action: five tiny programs, each trivial, combined into an analysis.

## Anatomy of a command

Every command has the same shape:

```bash
ls -l -h /etc
```

- `ls` is the **command**: the program to run.
- `-l` and `-h` are **options** (or *flags*). They change how the command behaves: here, *long* format with *human-readable* sizes. Short options can be combined (`ls -lh`). Long options start with two dashes (`ls --all`).
- `/etc` is an **argument**: what the command acts on.

Spaces separate the parts. A file name with spaces must be quoted, `"my file.csv"`, or the shell sees two arguments. This is one reason why developers avoid spaces in file names.

When a command finishes, it returns an **exit code**: `0` means success, anything else means an error. You rarely see it, but scripts and automated pipelines depend on it. `echo $?` prints the exit code of the last command.

## The filesystem

### One tree

On Windows, each disk has its own letter (`C:\`, `D:\`). On Unix systems there is a **single tree** that starts at the **root**, written `/`. Every file, every disk, even every device hangs somewhere below it. The main folders follow a convention, the *Filesystem Hierarchy Standard*, which is why any Linux server feels familiar:

| Folder | What it contains |
|---|---|
| `/` | the root of the tree |
| `/home/<user>` | each user's personal files (on macOS, `/Users/<user>`) |
| `/root` | the home of the administrator user, `root` |
| `/etc` | system-wide configuration files (text files, by tradition) |
| `/usr/bin`, `/bin` | programs available to all users |
| `/usr/local/bin` | programs installed by hand, outside the package manager |
| `/var/log` | logs: what services report while they run |
| `/tmp` | temporary files, deleted on reboot |
| `/proc`, `/sys` | not real files: a view into the running kernel (Linux only) |

You will meet these folders again: a container image is exactly a tree like this one, packed into a file.

### Paths

A **path** says where a file is in the tree:

- An **absolute path** starts at the root: `/home/ana/smd/ledger/data/transactions.csv`. It means the same thing wherever you are.
- A **relative path** starts from the **current directory**, the folder the shell is in now: if you are in `/home/ana/smd`, then `ledger/data/transactions.csv` refers to the same file.

Three shortcuts appear in paths all the time:

| Symbol | Means |
|---|---|
| `~` | your home directory (`/home/ana`) |
| `.` | the current directory |
| `..` | the parent directory, one level up |

So `../data` means "the `data` folder next to the one I'm in", and `./balance.sh` means "the `balance.sh` in this folder".

> 📘 **Definition: current directory and paths**
>
> Every shell has a **current (working) directory**, printed by `pwd` and changed with `cd`. An **absolute path** starts with `/` (or `~`) and is resolved from the root (or home). A **relative path** is resolved from the current directory.

### Moving around

| Command | Does |
|---|---|
| `pwd` | prints the current directory |
| `ls` | lists the files here (`ls -la`: all files, including hidden ones, with details) |
| `cd folder` | goes into `folder` |
| `cd ..` | goes up one level |
| `cd` or `cd ~` | goes home |
| `cd -` | goes back to the previous directory |

Files whose names start with a dot (`.env`, `.git`, `.bashrc`) are **hidden**: `ls` doesn't show them unless you add `-a`. Many configuration files are hidden this way, so remember `-a` when "the file isn't there".

### Working with files

| Command | Does |
|---|---|
| `mkdir -p a/b/c` | creates folders (`-p`: including missing parents, no error if they exist) |
| `touch file.txt` | creates an empty file (or updates its date) |
| `cp src dst` | copies (`cp -r` for folders) |
| `mv src dst` | moves or renames |
| `rm file` | deletes a file (`rm -r folder` for a folder) |
| `cat file` | prints a whole file |
| `less file` | shows a long file page by page (<kbd>Q</kbd> to quit, `/` to search) |
| `head -n 5 file`, `tail -n 5 file` | the first or last lines |

> ⚠️ **There is no recycle bin**
>
> `rm` deletes immediately and for good. `rm -r` deletes a folder and everything inside it. Read an `rm` command twice before pressing Enter, and never run `rm -rf` with a path you have not checked.

## Pipes and redirection

### The three standard streams

Every program starts with three channels already open:

- **standard input** (*stdin*): where it reads from, by default the keyboard;
- **standard output** (*stdout*): where it writes its results, by default the terminal;
- **standard error** (*stderr*): where it writes error messages, also the terminal by default.

Together they are called **standard I/O**, or *stdio* (you'll meet the name in documentation and in C's `stdio.h`). Each stream also has a number: 0 for stdin, 1 for stdout and 2 for stderr. You'll see the 2 again in the redirection `2>`.

Why keep errors apart from results? Because results are often data that another program or a file will consume. If `ls data nofile > list.txt` wrote its complaint about `nofile` into `list.txt`, the error would end up mixed with the file names, and you wouldn't see it. With a separate stderr, the file gets only the listing and the error still appears on your screen.

The shell can reconnect these channels before the program starts. This is what makes the combination of small tools possible.

### Redirection

| Syntax | Does |
|---|---|
| `cmd > file` | sends stdout to `file`, **replacing** its contents |
| `cmd >> file` | **appends** stdout to the end of `file` |
| `cmd < file` | reads stdin from `file` |
| `cmd 2> file` | sends stderr to `file` |
| `cmd > file 2>&1` | sends both stdout and stderr to `file` |

For example, `ls -l /etc > listing.txt` saves the listing in a file instead of printing it.

### Pipes

A **pipe**, `|`, connects the stdout of one program to the stdin of the next:

```bash
ls /usr/bin | wc -l
```

`ls` writes the list of programs, one per line, and `wc -l` counts the lines it receives. Nothing is written to disk in between, and both programs run at the same time.

> 📘 **Definition: pipeline**
>
> A **pipeline** `a | b | c` runs the commands together, connecting the standard output of each one to the standard input of the next. Only the output of the last one reaches the terminal (or a file, if redirected).

### The text toolbox

These are the tools you combine in pipelines most often. Each reads lines from a file or from stdin:

| Command | Does | Example |
|---|---|---|
| `grep pattern` | keeps lines containing `pattern` (`-i`: ignore case, `-v`: keep lines that *don't* match, `-c`: count) | `grep -i error app.log` |
| `cut -d, -f3` | keeps a column: here the 3rd, splitting on `,` | `cut -d, -f1,5 data.csv` |
| `sort` | sorts lines (`-n` numerically, `-r` reversed) | `sort -rn` |
| `uniq -c` | collapses *adjacent* repeated lines and counts them (so `sort` first) | `uniq -c sorted.txt` |
| `wc -l` | counts lines | `wc -l data.csv` |
| `head`, `tail` | first or last lines (`tail -n +2`: everything from line 2, which skips a header) | `tail -f app.log` follows a log live |
| `awk` | a small language for columns: filters and sums | `awk -F, '{ s += $5 } END { print s }'` |

Here is the analysis from the start of the page again. Read each step as a transformation of lines:

```bash
grep ",restaurants," transactions.csv |   # only restaurant rows
  cut -d, -f3 |                           # only the merchant column
  sort |                                  # equal merchants become adjacent
  uniq -c |                               # count each merchant
  sort -rn                                # biggest count first
```

A line that ends in `|` continues on the next one, so long pipelines can be split and commented like this in a script. When you type in the terminal, write it on one line without the comments. (The interactive zsh of macOS doesn't accept `#` comments by default.)

## Help, history and shortcuts

Nobody remembers every option. These save a lot of time:

- `man grep` opens the manual of `grep` (<kbd>Q</kbd> to quit). `grep --help` prints a shorter summary.
- <kbd>Tab</kbd> **completes** file and command names. Type `cd Doc` and press <kbd>Tab</kbd>. Pressing it twice lists the options. Use it constantly: it is faster and it avoids typos.
- <kbd>↑</kbd> and <kbd>↓</kbd> browse previous commands. <kbd>Ctrl</kbd>+<kbd>R</kbd> searches the history. `history` lists it.
- <kbd>Ctrl</kbd>+<kbd>C</kbd> stops the running command. <kbd>Ctrl</kbd>+<kbd>L</kbd> (or `clear`) clears the screen.

## bash, zsh and PowerShell

**bash** is the standard shell on Linux and the one you will find on servers and in containers. **zsh** is the default on macOS. For everything in this course it behaves like bash. Both follow the same Unix conventions, and so does everything on this page.

**PowerShell**, on Windows, is a different design. Its commands pass *objects* instead of text, and many names differ (`Get-ChildItem` is its `ls`, although it accepts `ls` as an alias). It is a good shell, but it is not what your servers and containers run. That is why, on Windows, we work inside a **Linux virtual machine** (Ubuntu or Debian) with bash ([Setting up the course environment](setup.md)).

## Three questions every system answers

The shell is how you talk to the system. The rest of this page is about the system itself: who can do what, how software gets installed, and what configures every program you run.

When you run a program on a Unix system, three things decide what happens before your code does anything at all:

1. **Who is running it?** The user determines which files the program can read, write or execute.
2. **What is installed?** The program can only use the tools and libraries that are on the system, in the versions that are there.
3. **What is its environment?** A set of named values, the environment variables, tells it where to find things and how to behave.

These are layers 2, 3 and 6 of the [environment stack](../01-dev-environments/notes.md#what-an-environment-is-made-of). The rest of this page covers each one.

## Users, groups and permissions

### Users and groups

Unix was designed for many people sharing one machine, so every file and every running program belongs to a **user**. Users are collected into **groups**, which let several users share access. The `id` command shows who you are:

```bash
id
```

```text
uid=1000(ana) gid=1000(ana) groups=1000(ana),4(adm),27(sudo),...
```

You are the user `ana`, with number (*uid*) 1000, and you belong to several groups, including `sudo` (more on that below). There is also one special user, **root** (uid 0), the administrator, who can read, change and delete anything.

Servers also have users that are not people. A database server runs as user `postgres`, a web server as `www-data`. If one of them is compromised, the attacker gets only that user's permissions, not the whole machine. The same idea applies to containers: well-built containers don't run as root.

### Reading permissions

`ls -l` shows who owns each file and what everyone may do with it. Here is a ledger folder like the one from the [lab](lab.md), with an extra `private` folder, and two system files:

```text
$ ls -l ~/smd/ledger
drwxr-xr-x  2 ana  ana   4096 Oct 29 10:02 data
-rwxr-xr-x  1 ana  ana    252 Oct 29 10:05 balance.sh
-rw-r--r--  1 ana  ana    180 Oct 29 10:07 report.txt
drwx------  2 ana  ana   4096 Oct 29 10:08 private
$ ls -l /etc/passwd /etc/shadow
-rw-r--r--  1 root root    888 Oct 24 14:12 /etc/passwd
-rw-r-----  1 root shadow  502 Oct 24 14:12 /etc/shadow
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

So `balance.sh` can be read, changed and run by `ana`, and read and run by everyone else. `report.txt` can be changed only by `ana`, and `private` is a folder nobody else can enter. `/etc/passwd` (the list of users) is readable by everybody but writable only by root. `/etc/shadow`, which contains the password hashes, cannot even be read by normal users:

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

Installing software or changing system configuration requires root. Instead of logging in as root, you run single commands with **`sudo`** (*superuser do*). Without it, installing fails because a normal user can't write to the system folders:

```text
$ apt install tree
E: Could not open lock file /var/lib/dpkg/lock-frontend - open (13: Permission denied)
E: Unable to acquire the dpkg frontend lock (/var/lib/dpkg/lock-frontend), are you root?
$ sudo apt install tree
[sudo] password for ana:
```

`sudo` asks for **your** password (not root's), checks that your user is allowed (on Ubuntu, membership of the `sudo` group; on macOS, of `admin`) and runs that one command as root. Every use is logged.

This applies the **principle of least privilege**: work as a normal user, and become root only for the command that needs it. A typo in a normal command can damage your files; the same typo with `sudo` can damage the whole system. Don't put `sudo` in front of a command just because it failed. Find out *why* it failed first. The answer is often a wrong path, not a missing permission.

## Installing software: package managers

On Windows, people traditionally install software by downloading an installer from a website. Unix systems use a **package manager** instead: a program that downloads software from trusted **repositories**, installs it in the standard places, installs whatever else it depends on, and can update or remove all of it later with one command.

| System | Package manager | Install | Update everything |
|---|---|---|---|
| Ubuntu, Debian (your VM, WSL) | `apt` | `sudo apt install tree` | `sudo apt update && sudo apt upgrade` |
| macOS | Homebrew (`brew`) | `brew install tree` | `brew update && brew upgrade` |
| Windows | `winget` | `winget install Git.Git` | `winget upgrade --all` |

With `apt`, `sudo apt update` refreshes the *list* of available packages and their versions from the repositories. It doesn't install anything. `sudo apt upgrade` installs the new versions. Run `update` before installing anything on a machine you haven't used for a while, or `apt` may try to download versions that no longer exist.

Homebrew installs into its own folder (`/opt/homebrew` on Apple Silicon) and doesn't need `sudo`.

Package managers solve the same problem at the system level that [lockfiles](../04-dependencies-tools/notes.md) solve for a project: known versions, installed in a repeatable way, from a trusted source. You can write down a list of packages and anyone can install exactly that list.

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

This is how Ana's script in [It works on my machine](../01-dev-environments/notes.md#it-works-on-my-machine) got its `RISK_THRESHOLD`: an `export` line in her `~/.zshrc` that nobody else had. Environment variables are the standard way to configure software, especially in containers and in the cloud, and a project must declare the ones it needs so that this can't happen.

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

- **Typing the path relative to the wrong place.** If a "file not found" makes no sense, run `pwd` and `ls`: you are probably in another folder.
- **Spaces in names.** `cd My Documents` tries to enter `My`. Quote it (`cd "My Documents"`) or use <kbd>Tab</kbd>, which adds the quotes for you.
- **`>` instead of `>>`.** `>` replaces the file. The previous contents are gone.
- **`uniq` without `sort`.** `uniq` only merges *adjacent* duplicates. Unsorted input gives wrong counts without any error.
- **Copying commands from the web without reading them.** Especially anything with `sudo`, `rm` or `curl … | sh`. Read it first.
- **`sudo` as a reflex.** It hides the real problem and can create files owned by root in your home folder, which then cause *more* `Permission denied` errors. Use it only for installing software and changing the system.
- **`chmod 777`** "to make it work". It gives every user write permission. It's never the right fix, especially for files with secrets.
- **Spaces around `=`.** `NAME = value` is an error in the shell; it must be `NAME=value`.
- **Forgetting `export`.** `NAME=value` alone (on its own line) sets a *shell* variable that the programs you start don't see. Use `export NAME=value`, or put the assignment in front of the command.
- **Editing `PATH` and losing the old value.** Write `export PATH="$HOME/bin:$PATH"`, which adds a folder, never `export PATH="$HOME/bin"`, which replaces the whole list.

> ✅ **Key takeaways**
>
> - The shell runs **commands** (`command -options arguments`). Each returns an **exit code**, where `0` means success.
> - Unix has **one filesystem tree** starting at `/`. Paths are **absolute** (from `/`) or **relative** (from the current directory), and `~`, `.` and `..` are shortcuts.
> - Programs read **stdin** and write **stdout** and **stderr**. Redirection (`>`, `>>`, `<`, `2>`) connects them to files, and **pipes** (`|`) connect programs.
> - Small tools (`grep`, `cut`, `sort`, `uniq`, `wc`, `awk`) combined in pipelines answer real questions about text files, and the pipeline itself documents the analysis.
> - <kbd>Tab</kbd>, the history and `man` make you fast. `rm` makes you careful.
> - Every file has an **owner**, a **group** and `rwx` permissions for owner, group and others. `chmod` changes them, and `chmod +x` makes a script runnable.
> - **root** can do anything. Work as a normal user and use **`sudo`** only for the single commands that need it (least privilege).
> - **Package managers** (`apt`, `brew`, `winget`) install software from trusted repositories, with its dependencies, in a repeatable way.
> - **Environment variables** configure programs and are inherited by child processes. Set them for one command (`NAME=x cmd`), for a session (`export`) or permanently (`~/.zshrc`, `~/.bashrc`).
> - **`PATH`** lists where the shell looks for commands. `which` shows what a name resolves to, and most `command not found` errors are `PATH` problems.

## Exercises

**1.** You are in `/home/ana/smd/ledger/data`. Write the absolute path and the shortest relative path to *(a)* `/home/ana/smd/ledger/balance.sh`; *(b)* `/home/ana/smd/notes.txt`; *(c)* `/etc/hosts`.

<details>
<summary>✅ <b>Solution</b></summary>

- *(a)* Absolute: `/home/ana/smd/ledger/balance.sh`. Relative: `../balance.sh` (one level up).
- *(b)* Absolute: `/home/ana/smd/notes.txt`. Relative: `../../notes.txt` (two levels up). Also `~/smd/notes.txt`, which works from anywhere.
- *(c)* Absolute: `/etc/hosts`. A relative path would be `../../../../../etc/hosts` (up to the root and down again). It works, but the absolute path is obviously better: use relative paths for things *near* you.

</details>

**2.** What does each command do? *(a)* `sort data.csv > data.csv`; *(b)* `grep -v "^#" config.txt | wc -l`; *(c)* `tail -n +2 data.csv | cut -d, -f1 | sort | uniq | wc -l`.

<details>
<summary>✅ <b>Solution</b></summary>

- *(a)* A classic trap: it **empties** `data.csv`. The shell opens the file for writing (and truncates it) *before* `sort` starts, so `sort` reads an empty file. To sort a file in place, write to another file and rename it, or use `sort -o data.csv data.csv`.
- *(b)* Counts the lines of `config.txt` that do **not** start with `#` (`^` means "start of line"), that is, the lines that aren't comments.
- *(c)* Skips the header line, takes the first column, and counts how many **distinct** values it has. For a transactions file whose first column is the date, that is the number of days with at least one transaction.

</details>

**3.** Write a pipeline that prints the five most frequent words in a file `report.txt`, assuming there is one word per line.

<details>
<summary>✅ <b>Solution</b></summary>

```bash
sort report.txt | uniq -c | sort -rn | head -n 5
```
`sort` puts equal words together, `uniq -c` counts them, `sort -rn` puts the biggest counts first and `head -n 5` keeps five. This pattern, `sort | uniq -c | sort -rn | head`, is worth memorizing: it answers "what is most frequent" for any column of any file.

</details>

**4.** A colleague runs `uniq -c categories.txt` and gets `groceries` counted several times on different lines, each time with a small number. What went wrong?

<details>
<summary>✅ <b>Solution</b></summary>

`uniq` only merges **adjacent** identical lines. The file wasn't sorted, so the `groceries` lines were scattered and each run of consecutive lines was counted separately. The fix is `sort categories.txt | uniq -c`.

</details>

**5.** Explain the difference between `ls > out.txt`, `ls >> out.txt` and `ls | wc -l > out.txt`.

<details>
<summary>✅ <b>Solution</b></summary>

- `ls > out.txt` writes the listing to `out.txt`, replacing whatever was there.
- `ls >> out.txt` adds the listing at the end of `out.txt`, keeping the previous contents.
- `ls | wc -l > out.txt` sends the listing through `wc -l`, and only its result, the number of files, goes into `out.txt` (replacing it). The redirection applies to the last command of the pipeline.

</details>

**6.** Decode these permissions and say who can do what: *(a)* `-rw-r--r--` owned by `ana`; *(b)* `-rwx------`; *(c)* `drwxr-x---` owned by `ana`, group `finance`.

<details>
<summary>✅ <b>Solution</b></summary>

- *(a)* A file that `ana` can read and write, and that everyone else can only read. This is the usual default for a new file.
- *(b)* A file that only its owner can read, write and execute. Nobody else can do anything with it: typical for a private script.
- *(c)* A directory (the `d`). `ana` can list it, create files in it and enter it. Members of `finance` can list it and enter it, but not create or delete files. Others can't even enter it.

</details>

**7.** Write the numeric `chmod` for: *(a)* `rwxr-xr-x`; *(b)* `rw-------`; *(c)* `rw-r-----`.

<details>
<summary>✅ <b>Solution</b></summary>

Add up `r` = 4, `w` = 2, `x` = 1 in each group of three:

- *(a)* 7, 5, 5 → `chmod 755`
- *(b)* 6, 0, 0 → `chmod 600`
- *(c)* 6, 4, 0 → `chmod 640`

</details>

**8.** You install a tool following its instructions, but typing its name gives `command not found`. The installer printed "installed to `~/.local/bin`". What is happening, and how do you fix it?

<details>
<summary>✅ <b>Solution</b></summary>

The shell looks for commands only in the folders listed in `PATH`, and `~/.local/bin` isn't one of them. First, open a new terminal: many installers add the folder to your startup file, and only new shells read it. If it still fails, check with `echo $PATH | tr ":" "\n"`. If the folder is missing, add `export PATH="$HOME/.local/bin:$PATH"` at the end of `~/.zshrc` (or `~/.bashrc`) and open a new terminal.

</details>

**9.** In one terminal you run `export API_KEY=abc123`. In a second terminal window, `echo $API_KEY` prints an empty line. Why?

<details>
<summary>✅ <b>Solution</b></summary>

An environment variable belongs to a process and is inherited only by the processes **it** starts. Each terminal window runs a separate shell, and neither is a child of the other, so the second one never received the variable. To have it in every shell you would put the `export` in your startup file. For a secret like an API key, though, that's a bad idea: session 12 shows better ways.

</details>

**10.** In the lab, `balance.sh` ends with `awk … "$file" | sort`. When `LEDGER_FILE` points to a file that doesn't exist, awk prints an error, but `echo $?` right after shows `0`. Why? Why does it matter?

<details>
<summary>✅ <b>Solution</b></summary>

The exit code of a pipeline is the exit code of its **last** command. `awk` failed, but `sort` successfully sorted its empty input and returned `0`. That matters because automated tools (scripts, CI pipelines, schedulers) decide whether a step worked by its exit code. This script would report success while producing nothing, which is Ana's dangerous kind of failure again. In bash, `set -o pipefail` at the top of a script makes a pipeline fail if any of its commands fails. It is a good habit, together with `set -e` (stop at the first failing command) and `set -u` (treat an undefined variable as an error).

</details>

[← Back to the session](index.md) [Next: Setting up the course environment →](setup.md)
