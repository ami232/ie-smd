# The shell and the filesystem

> ℹ️ **What you need first**
>
> A terminal open on a Unix shell: macOS Terminal, Linux, WSL on Windows or an Ubuntu container. [Lab 1](../dev-environments/lab.md) shows how to open one. Try every command on this page as you read: the shell is learned with the fingers.

## Why a text interface

Suppose your manager asks: *how much did we spend on restaurants last quarter, and with which merchant the most?* The transactions are in a CSV file. With a spreadsheet you open it, filter a column, select cells, read the sum, sort another column. It works, but there is no record of what you did, and next quarter you do it all again by hand.

In the shell, the same question is one line:

```bash
grep ",restaurants," transactions.csv | cut -d, -f3 | sort | uniq -c | sort -rn
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

You will meet these folders again. In session 11, a container image is exactly a tree like this one, packed into a file.

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

**PowerShell**, on Windows, is a different design. Its commands pass *objects* instead of text, and many names differ (`Get-ChildItem` is its `ls`, although it accepts `ls` as an alias). It is a good shell, but it is not what your servers and containers run. That is why, on Windows, we work inside **WSL**, a real Linux with bash ([Setting up the course environment](setup.md)).

## Common mistakes

- **Typing the path relative to the wrong place.** If a "file not found" makes no sense, run `pwd` and `ls`: you are probably in another folder.
- **Spaces in names.** `cd My Documents` tries to enter `My`. Quote it (`cd "My Documents"`) or use <kbd>Tab</kbd>, which adds the quotes for you.
- **`>` instead of `>>`.** `>` replaces the file. The previous contents are gone.
- **`uniq` without `sort`.** `uniq` only merges *adjacent* duplicates. Unsorted input gives wrong counts without any error.
- **Copying commands from the web without reading them.** Especially anything with `sudo`, `rm` or `curl … | sh`. Read it first.

> ✅ **Key takeaways**
>
> - The shell runs **commands** (`command -options arguments`). Each returns an **exit code**, where `0` means success.
> - Unix has **one filesystem tree** starting at `/`. Paths are **absolute** (from `/`) or **relative** (from the current directory), and `~`, `.` and `..` are shortcuts.
> - Programs read **stdin** and write **stdout** and **stderr**. Redirection (`>`, `>>`, `<`, `2>`) connects them to files, and **pipes** (`|`) connect programs.
> - Small tools (`grep`, `cut`, `sort`, `uniq`, `wc`, `awk`) combined in pipelines answer real questions about text files, and the pipeline itself documents the analysis.
> - <kbd>Tab</kbd>, the history and `man` make you fast. `rm` makes you careful.

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

[← Back to the session](index.md) [Next: Users, packages and the environment →](system.md)
