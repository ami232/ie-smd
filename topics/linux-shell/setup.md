# Setting up the course environment

> ℹ️ **What you need first**
>
> Your "machine card" from [Lab 1](../dev-environments/lab.md), administrator rights on your laptop and a good internet connection. The downloads add up to a few gigabytes, most of it Docker. The ideas behind the options are in [Environments and reproducibility](../dev-environments/environments.md#three-places-to-run-software-host-virtual-machine-container).

For the rest of the course, everybody needs the same three things: a **Unix shell** (bash or zsh), a **package manager** to install tools, and **Docker** to run containers. You install all three today: first the shell and the package manager, which are all that [Lab 2](lab.md) needs, and then [Docker](#docker), which we use from session 7 onwards. Installing it now leaves time to fix any problem before then. There are several ways to get them, and the right one depends on your laptop: keep the [machine card](../dev-environments/machine_card.md) from Lab 1 at hand. Your instructor announces the setup for the class.

This page describes the three possible setups, which we call *tracks*. Follow the one announced in class, unless your instructor tells you individually to use another one.

| Track | What you get | Who it suits |
|---|---|---|
| **A. Native** | your own OS's terminal + Docker Desktop (on Windows, a real Ubuntu through WSL) | most laptops: 16 GB of memory or more, administrator rights |
| **B. Ubuntu VM** | an Ubuntu server running in a VM on your laptop, with Docker inside | when you want everything isolated from your own system, or native Docker doesn't work |
| **C. Cloud** | a Linux machine in GitHub Codespaces, used from the browser or VS Code | managed laptops without administrator rights, low-spec machines, tablets |

Whichever track you follow, the commands in the rest of the course are the same: you always work in a Linux-style shell.

## Track A: native

**macOS**

macOS is already a Unix system, and its Terminal runs zsh, so you only need a package manager and [Docker](#docker).

**Install Homebrew**, the package manager for macOS. Paste the command from [brew.sh](https://brew.sh) into the Terminal:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

It asks for your password and installs Apple's command-line developer tools if you don't have them (that takes a few minutes). **At the end it prints "Next steps": two commands that add Homebrew to your `PATH`. Run them.** Then check:

```bash
brew --version
```


**Windows**

On Windows, we work inside **WSL** (the Windows Subsystem for Linux): a real Ubuntu, with bash and `apt`, integrated with Windows. Docker Desktop uses it too.

> 📘 **What WSL is**
>
> WSL lets Linux programs run on Windows **without dual booting and without managing a VM yourself**. Its current version, WSL 2, runs a real Linux kernel, built by Microsoft, inside a small, lightweight [virtual machine](../dev-environments/environments.md#virtual-machines) that Windows starts and stops for you in a second or two. On top of that kernel you install a *distribution* (we use Ubuntu), which brings the usual Linux userland: bash, `apt` and the standard tools.
>
> Unlike a VM you create yourself, WSL is integrated with Windows. Ubuntu appears as a tab in Windows Terminal; the Windows disk is visible from Linux at `/mnt/c`; your Linux files are visible from the File Explorer under **Linux**; and Windows programs (such as VS Code) can work on Linux folders. Docker Desktop uses the same lightweight VM to run Linux containers, which is why it needs WSL.
>
> The older WSL 1 had no Linux kernel: it translated Linux system calls into Windows ones. It was slower for files and couldn't run Docker. `wsl --install` installs WSL 2.

**1. Install WSL with Ubuntu.** Right-click **Terminal** (or **PowerShell**) and choose **Run as administrator**. Then run:

```powershell
wsl --install
```

This enables the virtualization components, installs the Linux kernel and the Ubuntu distribution. **Restart Windows** when it finishes.

If WSL is already installed with another distribution (or only with `docker-desktop`), `wsl --install` doesn't add Ubuntu. Run `wsl --install -d Ubuntu` instead. `wsl --list --verbose` shows what you have.

**2. Create your Linux user.** After the restart, open **Ubuntu** from the Start menu (it also appears as a tab in Windows Terminal). The first time, it asks for a **user name and password** for Linux. They are independent of your Windows account; choose something you'll remember, because `sudo` will ask for this password. Then update the system:

```bash
sudo apt update && sudo apt upgrade -y
```

> ⚠️ **Work inside Ubuntu, not in PowerShell**
>
> From now on, open the **Ubuntu** terminal for the course and keep your projects in your Linux home folder (`~`, which is `/home/<you>`), **not** in `/mnt/c/...`. Files on the Windows disk are much slower to access from Linux, and their permissions behave differently.

If `wsl --install` fails, the most common cause is virtualization being disabled in the firmware ([Lab 1, step 3](../dev-environments/lab.md#3-memory-disk-and-virtualization)). Ask your instructor.


**Linux**

You already have the shell and the package manager: only [Docker](#docker) is left.


## Track B: an Ubuntu VM

We use **Multipass**, a tool from Canonical (the company behind Ubuntu) that creates Ubuntu VMs with one command, on macOS, Windows and Linux. You work in the VM through its shell.

**1. Install Multipass.** On macOS: `brew install --cask multipass` (install Homebrew first, as in track A). On Windows or Linux, download it from [canonical.com/multipass](https://canonical.com/multipass/install). On Windows Home, Multipass needs VirtualBox, and the installer explains how to set it up.

**2. Create the course VM** with 2 CPUs, 4 GB of memory and 30 GB of disk, and open a shell in it:

```bash
multipass launch 24.04 --name smd --cpus 2 --memory 4G --disk 30G
multipass shell smd
```

The prompt changes to `ubuntu@smd:~$`: you are now inside the VM. Everything else in the course happens here.

Useful commands, run on your laptop (not inside the VM): `multipass list` (VMs and their state), `multipass stop smd` (frees the memory when you're not working) and `multipass start smd`.

## Track C: GitHub Codespaces

A **codespace** is a Linux machine in GitHub's cloud, with VS Code in the browser and Docker ready to use. Nothing is installed on your laptop. A personal GitHub account includes a free monthly quota (enough for about 60 hours on the smallest machine), and the GitHub Student Developer Pack raises it.

1. Sign in to GitHub. Your instructor gives you the link to the course's starter repository.
2. In the repository, click **Code → Codespaces → Create codespace on main**.
3. After a minute, VS Code opens in the browser. The **Terminal** panel at the bottom is a bash shell on Ubuntu.

> ⚠️ **Stop your codespace when you finish**
>
> A codespace consumes your quota while it runs. It stops on its own after 30 minutes of inactivity, but it's better to stop it yourself: on [github.com/codespaces](https://github.com/codespaces), **… → Stop codespace**.

## Docker

Install Docker once your shell works: on Windows, Docker Desktop needs WSL with Ubuntu, so finish the steps above first. Docker needs about **20 GB of free disk** for the program and the images, and on a Mac a recent macOS: Docker Desktop supports the current version and the two previous ones.

> ⚠️ **Not enough space, or an older macOS?**
>
> Skip Docker today and tell your instructor. Free space or update macOS before session 7, the first session that uses Docker. Everything else today, Lab 2 included, works without it.

**macOS**

**Install Docker Desktop.** Download it from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/). Choose **Apple Silicon** or **Intel** according to the architecture on your machine card. Open the `.dmg`, drag Docker to Applications and start it. Accept the terms. You can skip signing in.


**Windows**

**Install Docker Desktop.** Download it from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/) (choose **ARM64** if your machine card says `ARM64`) and install it with the default options, which use **WSL 2**. Start it, then in **Settings → Resources → WSL integration** turn on the integration with **Ubuntu**.


**Linux**

Install Docker Engine following the official instructions for your distribution ([docs.docker.com/engine/install](https://docs.docker.com/engine/install/)). Then let your user run Docker without `sudo`:

```bash
sudo usermod -aG docker $USER
```

Log out and in again for the group change to take effect.


**Ubuntu VM (track B)**

Inside the VM (`multipass shell smd`), install Docker from Ubuntu's own packages and let your user use it:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo usermod -aG docker $USER
exit
```

`exit` leaves the VM. Run `multipass shell smd` again so that the group change takes effect.


**Codespace (track C)**

Docker is already installed.


## Check that it works

Whatever your track, open the course shell (macOS Terminal, the Ubuntu terminal on Windows, `multipass shell smd`, or the codespace terminal) and run:

```bash
echo $SHELL
uname -sm
curl --version | head -n 1
docker run --rm hello-world
```

<details>
<summary>✅ <b>What you should see</b></summary>

- `/bin/zsh` or `/bin/bash`.
- `Darwin arm64` (or `Darwin x86_64`) on a Mac. `Linux x86_64` or `Linux aarch64` everywhere else: in WSL, in the VM and in a codespace you are on Linux.
- A line starting with `curl 8.` (any version is fine).
- Docker downloads a tiny image and prints a message that starts with `Hello from Docker!` and explains what just happened.

If `docker` says `permission denied while trying to connect to the Docker daemon socket`, your user isn't in the `docker` group yet: log out and in again (or exit and reopen the VM shell). If it says `Cannot connect to the Docker daemon`, Docker Desktop isn't running: start it.

</details>

Once the first three commands work, you're ready for [Lab 2](lab.md). The Docker check can wait until after the lab if Docker is still downloading.

[← Previous: Users, packages and the environment](system.md) [Next: Lab 2 →](lab.md)
