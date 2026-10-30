# Lab 2: a ledger in the shell

> ℹ️ **What you need first**
>
> The course environment installed and checked ([Setting up the course environment](setup.md#check-that-it-works)). The commands are explained in [The shell and the filesystem](shell.md) and [Users, packages and the environment](system.md).

In this lab you create the folder for **ledger**, the application we build during the course. You download its first data, a CSV file with three months of transactions from a personal checking account and a savings account, and answer questions about the money using only the shell. Then you turn your best pipeline into a script, the first piece of code of the project.

Type the commands rather than copying them: the point is to get them into your fingers. Use <kbd>Tab</kbd> to complete names.

## 1. Create the project folder

All the course work goes into one folder, `smd`, in your home directory. Inside it, create the folder for the application and a `data` folder for its data:

```bash
cd ~
mkdir -p smd/ledger/data
cd smd/ledger
pwd
```

<details>
<summary>✅ <b>What you should see</b></summary>

`pwd` prints the absolute path of the new folder: `/Users/<you>/smd/ledger` on macOS, `/home/<you>/smd/ledger` on Linux, WSL and the VM, `/workspaces/...` in a codespace (where the course repository already gives you a working folder; create `ledger` inside it).

`mkdir -p` created the three nested folders in one go. Run the same command again: `-p` doesn't complain when they already exist.

</details>

## 2. Download the data

`curl` downloads a URL. `-o` saves the result to a file instead of printing it, and `-fsSL` makes curl fail clearly on errors (`-f`), stay quiet (`-s`) but still show errors (`-S`), and follow redirects (`-L`):

```bash
curl -fsSL -o data/transactions.csv https://raw.githubusercontent.com/ami232/ie-smd/main/data/transactions.csv
ls -l data
```

Look at the first lines and count them:

```bash
head -n 5 data/transactions.csv
wc -l data/transactions.csv
```

<details>
<summary>✅ <b>What you should see</b></summary>

```text
date,account,description,category,amount
2026-01-01,checking,Rent,housing,-950.00
2026-01-02,checking,Carrefour,groceries,-65.15
2026-01-03,checking,Spotify,subscriptions,-11.99
2026-01-05,checking,Netflix,subscriptions,-13.99
```

and `57 data/transactions.csv`: a header line plus **56 transactions**. Each row has a date, the account (`checking` or `savings`), a description, a category and a signed amount in euros. Spending is negative, income positive.

</details>

Use `less data/transactions.csv` to scroll through the whole file (<kbd>Q</kbd> to quit). What happens at the end of each month?

## 3. Questions with grep

How many grocery purchases are there? `grep` keeps the matching lines, and `-c` counts them instead of printing them:

```bash
grep -c groceries data/transactions.csv
```

Now, how many times did the account holder shop at Mercadona? Try it with and without `-i` (ignore case):

```bash
grep mercadona data/transactions.csv
grep -i mercadona data/transactions.csv
```

Finally, show only the movements of the **savings** account. Searching for `savings` alone would also match `Transfer to savings` in the checking account. Include the commas to match the whole column:

```bash
grep ",savings," data/transactions.csv
```

<details>
<summary>✅ <b>What you should see</b></summary>

- `12` grocery purchases.
- `grep mercadona` prints nothing, because the file says `Mercadona` with a capital M. With `-i`, three purchases appear (16 January, 10 February and 16 March).
- Six movements in savings: each month a 300.00 transfer from checking and a small interest payment.

</details>

## 4. Spending by category

Which categories have the most transactions? This is the `sort | uniq -c | sort -rn` pattern from [the concepts page](shell.md#the-text-toolbox), applied to the fourth column. `tail -n +2` skips the header line:

```bash
cut -d, -f4 data/transactions.csv | tail -n +2 | sort | uniq -c | sort -rn
```

Build the pipeline **one step at a time**. Run `cut -d, -f4 data/transactions.csv` alone, then add `| tail -n +2` and run it again, then `| sort`, and so on. Look at what each step does to the lines. That is how you write and debug pipelines.

<details>
<summary>✅ <b>What you should see</b></summary>

```text
  12 groceries
  11 restaurants
   9 transport
   6 transfer
   6 subscriptions
   3 interest
   3 income
   3 housing
   3 health
```

The number of spaces at the start of each line may vary between systems.

</details>

## 5. Sums with awk

Counting is not enough for a ledger: we need **amounts**. `awk` splits each line into fields (`-F,` uses the comma as separator, `$5` is the fifth field) and can add them up. `NR > 1` skips the first line (*NR* is the line number) and the `END` block runs after the last line:

```bash
awk -F, 'NR > 1 { total += $5 } END { printf "%.2f\n", total }' data/transactions.csv
```

The total of every row combined is the net change across both accounts. Now compute the total spent on restaurants by filtering first with `grep`:

```bash
grep ",restaurants," data/transactions.csv | awk -F, '{ total += $5 } END { printf "%.2f\n", total }'
```

(No `NR > 1` here: `grep` already removed the header.)

<details>
<summary>✅ <b>What you should see</b></summary>

`4255.10` for the net total, and `-320.09` for restaurants: 320.09 euros spent in three months.

`printf "%.2f\n"` prints the number with two decimals. Without it, `awk` would print `4255.1`. In session 3 you will see that adding money with ordinary floating-point numbers can give results like `4255.099999999999`, and how Python avoids it.

</details>

## 6. Save results to a file

Use redirection to build a small report:

```bash
echo "Transactions per category" > report.txt
cut -d, -f4 data/transactions.csv | tail -n +2 | sort | uniq -c | sort -rn >> report.txt
cat report.txt
```

The first line creates the file (`>`) and the second **appends** (`>>`). Change the second `>>` to `>` and run both again: what happens to the title?

## 7. Your first script

Commands you need more than once belong in a **script**: a text file of commands. Create `balance.sh`, which prints the balance of each account. The `cat > file <<'EOF' … EOF` construction writes everything up to the `EOF` line into the file. Type or paste the whole block at once:

```bash
cat > balance.sh <<'EOF'
#!/usr/bin/env bash
# Prints the balance of each account in a transactions CSV.
file="${LEDGER_FILE:-data/transactions.csv}"
echo "Balances in $file"
awk -F, 'NR > 1 { s[$2] += $5 } END { for (a in s) printf "%-10s %10.2f\n", a, s[a] }' "$file" | sort
EOF
```

Line by line:

- `#!/usr/bin/env bash`, the **shebang**, tells the system to run the file with bash.
- `file="${LEDGER_FILE:-data/transactions.csv}"` reads the environment variable `LEDGER_FILE` and, if it isn't set, uses the default path. This is how programs get their configuration, as described in [the concepts page](system.md#environment-variables).
- The `awk` program adds the amounts *per account*. `s[$2]` is a table indexed by the second column.

Now run it:

```bash
./balance.sh
```

It fails. Look at the permissions, fix them and run it again:

```bash
ls -l balance.sh
chmod +x balance.sh
ls -l balance.sh
./balance.sh
```

<details>
<summary>✅ <b>What you should see</b></summary>

The first attempt gives `permission denied: ./balance.sh` (zsh) or `bash: ./balance.sh: Permission denied` (bash): a new file is not executable. `ls -l` shows `-rw-r--r--`. After `chmod +x` it shows `-rwxr-xr-x`, and the script prints:

```text
Balances in data/transactions.csv
checking      3349.10
savings        906.00
```

</details>

## 8. Configure it with an environment variable

Make a file with only the checking account's movements, and point the script at it **for one command**:

```bash
grep -v ",savings," data/transactions.csv > data/checking.csv
LEDGER_FILE=data/checking.csv ./balance.sh
./balance.sh
```

Then **export** a file that doesn't exist, run the script and check its exit code:

```bash
export LEDGER_FILE=data/missing.csv
./balance.sh
echo $?
unset LEDGER_FILE
```

<details>
<summary>✅ <b>What you should see</b></summary>

With `LEDGER_FILE=data/checking.csv`, only the `checking` line appears: `grep -v` kept all the lines *except* the savings ones (the header included). The next `./balance.sh` uses the default file again, because the variable existed only for that one command.

With the missing file, `awk` prints an error saying it can't open `data/missing.csv`, yet `echo $?` prints `0`: the script says it succeeded. [Exercise 5 of the concepts page](system.md#exercises) explains why, and how `set -o pipefail` fixes it. Add `set -euo pipefail` as the second line of `balance.sh` and try again. Now the exit code is not 0.

</details>

## 9. Install a tool with the package manager

Install `tree`, which draws a folder as a tree:

**macOS**

```bash
brew install tree
```


**Ubuntu (WSL, VM, codespace)**

```bash
sudo apt update
sudo apt install -y tree
```


Then look at your project:

```bash
cd ~/smd
tree
which tree
```

<details>
<summary>✅ <b>What you should see</b></summary>

```text
.
└── ledger
    ├── balance.sh
    ├── data
    │   ├── checking.csv
    │   └── transactions.csv
    └── report.txt
```

`which tree` prints `/opt/homebrew/bin/tree` on an Apple Silicon Mac and `/usr/bin/tree` on Ubuntu. Those are two folders in your `PATH`, where each package manager puts its programs.

</details>

## Checkpoint

You have finished the lab when:

- [ ] `~/smd/ledger` contains `data/transactions.csv`, `report.txt` and an executable `balance.sh`;
- [ ] `./balance.sh` prints the two balances (3349.10 and 906.00);
- [ ] you can explain, without looking, what each command of the category pipeline in step 4 does.

**Challenge (optional).** Write a pipeline that prints the total spent per category, from the most to the least. Count only negative amounts, and leave out transfers, which move money between your own accounts but don't spend it. Hint: in `awk`, the condition can combine tests with `&&` (`$5 < 0 && $4 != "transfer"`). Use a table indexed by `$4`, as in `balance.sh`, then sort the output by its second column, numerically, with `sort -t' ' -k2 -n`.

<details>
<summary>✅ <b>Result to check</b></summary>

With spending as negative numbers, sorting numerically puts the biggest spending first:

```text
housing -2850.00
groceries -797.60
restaurants -320.09
transport -135.57
health -119.70
subscriptions -77.94
```

</details>

[← Previous: Setting up the course environment](setup.md) [Back to the session ↑](index.md)
