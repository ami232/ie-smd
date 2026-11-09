# Setting up the course environment

> ℹ️ **What you need first**
>
> Your "machine card" from [Lab 1](../01-dev-environments/lab.md), administrator rights on your laptop and a good internet connection. The downloads add up to a few gigabytes, most of it Docker. The ideas behind the options are in [Environments and reproducibility](../01-dev-environments/environments.md#three-places-to-run-software-host-virtual-machine-container).

For the rest of the course, everybody needs the same three things: a **Unix shell** (bash or zsh), a **package manager** to install tools, and **Docker** to run containers. You install all three today: first the shell and the package manager, which are all that [Lab 2](lab.md) needs, and then [Docker](#docker), which we use from session 7 onwards. Installing it now leaves time to fix any problem before then. There are several ways to get them, and the right one depends on your laptop's operating system: keep the [machine card](../01-dev-environments/machine_card.md) from Lab 1 at hand.

This page describes the three possible setups, which we call *tracks*. Follow the one for your operating system, unless your instructor tells you individually to use another one.

| Track | What you get | Who it suits |
|---|---|---|
| **A. Native** | your own OS's terminal and package manager, plus Docker | macOS and Linux laptops |
| **B. Linux VM** | an Ubuntu or Debian virtual machine on your laptop, the one you set up in the introductory course, with Docker inside | **everybody on Windows**; on macOS or Linux, anyone who prefers to keep the course separate from their own system |
| **C. Cloud** | a Linux machine in GitHub Codespaces, used from the browser or VS Code | laptops where neither A nor B works: managed laptops without administrator rights, low-spec machines, tablets |

Whichever track you follow, the commands in the rest of the course are the same: you always work in a Linux-style shell.

> 📝 **And WSL?**
>
> On Windows there is another way to get Linux, **WSL** (the Windows Subsystem for Linux). It is well integrated with Windows, but when something goes wrong, fixing it needs a good understanding of how Windows, Linux and the virtual machine underneath fit together. **Use WSL only if you already know what you're doing**; otherwise use the VM (track B). If you do use it, the instructions are in [WSL (advanced)](#wsl-advanced).

## Track A: native

For macOS and Linux. On Windows, use [track B](#track-b-a-linux-vm).

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


**Linux**

You already have the shell and the package manager: only [Docker](#docker) is left.


## Track B: a Linux VM

Use the **Ubuntu or Debian virtual machine** you set up in the introductory course. Start it, open its terminal and do **everything in the course inside the VM**: the shell, the package manager (`apt`), Docker, VS Code and your projects. Your laptop's own operating system only runs the VM.

- **Keep your projects in your home folder inside the VM** (`~`, which is `/home/<you>`), not in a folder shared with your laptop. Shared folders are slower and handle permissions differently, which breaks some tools.
- **Give the VM enough resources:** at least 2 CPUs, 4 GB of memory and 30 GB of disk. Docker images take space. You change this in the VM's settings while it is shut down.
- **Shut the VM down** when you aren't working, to free your laptop's memory.

Ubuntu and Debian are both Debian-based: the same `apt` commands work on both, and wherever the course says "Ubuntu", it applies to Debian too. Desktop installations don't always include `curl` and `git`, which the course uses from the start, so install them now:

```bash
sudo apt update && sudo apt install -y curl git
```

If you don't have a working VM, tell your instructor.

## Track C: GitHub Codespaces

A **codespace** is a Linux machine in GitHub's cloud, with VS Code in the browser and Docker ready to use. Nothing is installed on your laptop. A personal GitHub account includes a free monthly quota (enough for about 60 hours on the smallest machine), and the GitHub Student Developer Pack raises it.

1. Sign in to GitHub. Your instructor gives you the link to the course's starter repository.
2. In the repository, click **Code → Codespaces → Create codespace on main**.
3. After a minute, VS Code opens in the browser. The **Terminal** panel at the bottom is a bash shell on Ubuntu.

> ⚠️ **Stop your codespace when you finish**
>
> A codespace consumes your quota while it runs. It stops on its own after 30 minutes of inactivity, but it's better to stop it yourself: on [github.com/codespaces](https://github.com/codespaces), **… → Stop codespace**.

## Docker

Install Docker once your shell works. Docker needs about **20 GB of free disk** for the program and the images, and on a Mac a recent macOS: Docker Desktop supports the current version and the two previous ones.

> ⚠️ **Not enough space, or an older macOS?**
>
> Skip Docker today and tell your instructor. Free space or update macOS before session 7, the first session that uses Docker. Everything else today, Lab 2 included, works without it.

**macOS**

**Install Docker Desktop.** Download it from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/). Choose **Apple Silicon** or **Intel** according to the architecture on your machine card. Open the `.dmg`, drag Docker to Applications and start it. Accept the terms. You can skip signing in.


**Linux and the VM (track B)**

Install Docker Engine with Docker's installation script, which works on Ubuntu and Debian, and let your user run Docker without `sudo`:

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

**Log out and in again** (or restart the VM) for the group change to take effect. On other distributions, follow the official instructions at [docs.docker.com/engine/install](https://docs.docker.com/engine/install/).


**Codespace (track C)**

Docker is already installed.


## Check that it works

Whatever your track, open the course shell (the macOS Terminal, the terminal on Linux or inside your VM, or the codespace terminal) and run:

```bash
echo $SHELL
uname -sm
curl --version | head -n 1
docker run --rm hello-world
```

<details>
<summary>✅ <b>What you should see</b></summary>

- `/bin/zsh` or `/bin/bash`.
- `Darwin arm64` (or `Darwin x86_64`) on a Mac. `Linux x86_64` or `Linux aarch64` everywhere else: in the VM, in WSL and in a codespace you are on Linux.
- A line starting with `curl 8.` (any version is fine).
- Docker downloads a tiny image and prints a message that starts with `Hello from Docker!` and explains what just happened.

If `docker` says `permission denied while trying to connect to the Docker daemon socket`, your user isn't in the `docker` group yet: log out and in again (or restart the VM). If it says `Cannot connect to the Docker daemon`, Docker isn't running: start Docker Desktop on macOS, or run `sudo systemctl start docker` on Linux.

</details>

Once the first three commands work, you're ready for [Lab 2](lab.md). The Docker check can wait until after the lab if Docker is still downloading.

## WSL (advanced)

The course supports Windows through the VM ([track B](#track-b-a-linux-vm)). Use WSL only if you are already comfortable with it and can solve its problems on your own: in class, we help with the VM first.

<details>
<summary>⚠️ <b>WSL instructions: only if you know what you're doing</b></summary>

**WSL** (the Windows Subsystem for Linux) gives you a real Ubuntu, with bash and `apt`, integrated with Windows. Docker Desktop uses it too.

> 📘 **What WSL is**
>
> WSL lets Linux programs run on Windows **without dual booting and without managing a VM yourself**. Its current version, WSL 2, runs a real Linux kernel, built by Microsoft, inside a small, lightweight [virtual machine](../01-dev-environments/environments.md#virtual-machines) that Windows starts and stops for you in a second or two. On top of that kernel you install a *distribution* (we use Ubuntu), which brings the usual Linux userland: bash, `apt` and the standard tools.
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

If `wsl --install` fails, the most common cause is virtualization being disabled in the firmware ([Lab 1, step 3](../01-dev-environments/lab.md#3-memory-disk-and-virtualization)). If you can't fix it, use the VM.

**3. Install Docker Desktop** on Windows. Download it from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/) (choose **ARM64** if your machine card says `ARM64`) and install it with the default options, which use **WSL 2**. Start it, then in **Settings → Resources → WSL integration** turn on the integration with **Ubuntu**.

With WSL, "the course shell" is the **Ubuntu** terminal, and the rest of the course works as on Linux.

</details>

[← Previous: Users, packages and the environment](system.md) [Next: Lab 2 →](lab.md)
