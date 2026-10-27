# Lab 1: know your machine

> ℹ️ **What you need first**
>
> Your laptop, with its password, and your IE email. The concepts are in [Environments and reproducibility](environments.md).

In this lab you meet the tool you will use more than any other in the course, the **terminal**. You use it to find out exactly which environment your laptop provides, layer by layer. You also create the two accounts we need from session 5 onwards. Before you start, download the [machine card](machine_card.md) and write your answers in it as you go: in session 2 you use it to set up the course environment, so fill it in carefully.

You don't install anything today. Every command below only *reads* information.

## 1. Open a terminal

A **terminal** is a window where you type commands as text, and a program called the **shell** runs them and prints the result. Every operating system has one:

**macOS**

Press <kbd>Cmd</kbd>+<kbd>Space</kbd>, type **Terminal** and press <kbd>Enter</kbd>. The shell on recent macOS versions is **zsh**.


**Windows**

Press the <kbd>Win</kbd> key, type **Terminal** and open **Windows Terminal** (on Windows 10, if you don't have it, open **Windows PowerShell**). The shell is **PowerShell**.


**Linux**

Press <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>T</kbd>, or look for **Terminal** in your applications. The shell is usually **bash**.


You see a **prompt**: a short line ending in `%`, `$` or `>`, followed by a blinking cursor. Type the command after it and press <kbd>Enter</kbd>. In this course, a command block shows only what you type. Don't type the prompt symbol.

Start with three commands that work in all three shells:

```bash
whoami
pwd
ls
```

`whoami` prints your user name. `pwd` (*print working directory*) prints the folder the shell is "in", which is your **home directory** when you open a terminal. `ls` lists the files in it.

<details>
<summary>✅ <b>What you should see</b></summary>

Your user name, then a path to your home folder, then its contents (`Desktop`, `Documents`, `Downloads`…). The path depends on the system:

- macOS: `/Users/<you>`
- Windows: `C:\Users\<you>`
- Linux: `/home/<you>`

Note the separators: `/` on macOS and Linux, `\` on Windows. This is the operating-system layer of the environment, and you can already see it.

</details>

## 2. Identify the operating system and the architecture

Now go up the stack from the bottom, starting with the processor architecture and the operating system. Use the tab for your system:

**macOS**

```bash
sw_vers
uname -m
```

`sw_vers` prints the macOS version. `uname -m` prints the machine architecture.


**Windows**

```powershell
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, Version
$env:PROCESSOR_ARCHITECTURE
```

The first command prints the Windows edition (Home, Pro, Enterprise…) and version. The second prints the processor architecture.


**Linux**

```bash
cat /etc/os-release
uname -m
```

`/etc/os-release` is a text file that describes the distribution. `cat` prints it.


<details>
<summary>✅ <b>What you should see</b></summary>

- **macOS**: `ProductName: macOS`, a version such as `26.6`, and then `arm64` (Apple Silicon: M1, M2…) or `x86_64` (older Intel Macs).
- **Windows**: a caption such as `Microsoft Windows 11 Pro` and a version such as `10.0.26100` (Windows 11 still reports itself as version 10.0 internally). Then `AMD64` (an Intel or AMD processor, the `x86_64` architecture) or `ARM64` (a Snapdragon processor).
- **Linux**: the name and version of your distribution (`Ubuntu 24.04.5 LTS`…), and `x86_64` or `aarch64` (the Linux name for `arm64`).

The same architecture has several names: `x86_64` = `amd64` = `AMD64`, and `arm64` = `aarch64` = `ARM64`. You will see all of them in installers and in Docker.

</details>

## 3. Memory, disk and virtualization

Containers, databases and VMs need memory and disk space. Write down how much you have:

**macOS**

```bash
echo $(( $(sysctl -n hw.memsize) / 1073741824 )) GB
df -h ~
```

`sysctl -n hw.memsize` returns the memory in bytes; dividing by 1073741824 (2³⁰) converts it to GB. `df -h ~` shows the disk that holds your home folder: look at the **Avail** column.

Virtualization is always available on Macs, so there's nothing more to check.


**Windows**

```powershell
[math]::Round((Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory / 1GB)
Get-PSDrive C
wsl --status
```

The first line prints the memory in GB, and `Get-PSDrive C` prints the used and free space of the `C:` drive, in GB. `wsl --status` tells you whether the Windows Subsystem for Linux is installed. If it isn't, it prints a message saying so, which is fine.

To check that **virtualization** is enabled, open the **Task Manager** (<kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>Esc</kbd>), go to **Performance → CPU** and look for **Virtualization: Enabled** under the graph.


**Linux**

```bash
free -h
df -h ~
```

`free -h` shows the memory: look at the **total** column of the `Mem:` row. `df -h ~` shows the free space under **Avail**.


<details>
<summary>✅ <b>What you should see</b></summary>

Something like `16 GB` of memory and a free-space figure. For the course you need **at least 8 GB of memory (16 GB is comfortable)** and **at least 20 GB of free disk**. If you have less, don't worry: write it down on your [machine card](machine_card.md). Session 2 has setup options for smaller machines.

On Windows, if **Virtualization** says **Disabled**, write that down too: it can be turned on in the computer's firmware settings (BIOS/UEFI), and we will help you with that in session 2.

</details>

## 4. Can you install software?

Many professionals bring a laptop managed by their employer, with restrictions on what they can install. These restrictions can make Docker or a VM impossible, so we need to know now.

Answer these questions for yourself:

- Is this laptop **yours**, or does it belong to your employer?
- When you install an application, does the system ask for **your** password (you are an administrator), or for an IT department's approval?
- Is there corporate security software that blocks installations or manages the device (a "device management" profile on macOS, a company account in Windows settings)?

**macOS**

This command prints `admin` if your user is an administrator, and nothing otherwise:

```bash
id -Gn | tr ' ' '\n' | grep -x admin
```


**Windows**

Open **Settings → Accounts → Your info**. Under your name it says **Administrator** or **Standard user**. Under **Settings → Accounts → Access work or school** you can see whether the laptop is connected to an organization.


**Linux**

This command prints `sudo` if you can run administrator commands:

```bash
groups | tr ' ' '\n' | grep -x sudo
```


## 5. Create your accounts

We use two online services during the course. Create the accounts now, because they can take a while to verify.

**GitHub** hosts your code from session 5 onwards and runs our automated pipelines in session 13. If you already have an account, use it.

1. Go to [github.com/signup](https://github.com/signup) and create an account. Choose a **professional username**: it appears in your repositories, and employers will see it on your CV.
2. Enable **two-factor authentication** (*Settings → Password and authentication*). GitHub requires it to contribute code.
3. Optionally, apply to the [GitHub Student Developer Pack](https://education.github.com/pack) with your IE email. It gives you free access to GitHub Copilot and other developer tools.

**Azure for Students** is the cloud account for sessions 13 and 14. It gives you **USD 100 of credit for 12 months, with no credit card**. You verify that you are a student with your IE email.

1. Go to [aka.ms/azure4students](https://aka.ms/azure4students) and select **Start free**.
2. Sign in with a Microsoft account. You can use your IE email.
3. Verify your student status with your **IE email** and complete the form.
4. When it finishes, open the [Azure portal](https://portal.azure.com), search for **Subscriptions** and check that you see one called **Azure for Students**.

> ⚠️ **Don't create any resources yet**
>
> Everything you create in Azure consumes your credit, even if you don't use it. We create resources together in session 13, and we also learn how to delete them. Until then, just check that the subscription exists.

[← Previous: Environments and reproducibility](environments.md) [Back to the session ↑](index.md)
