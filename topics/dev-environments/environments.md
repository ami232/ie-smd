# Environments and reproducibility

> ℹ️ **What you need first**
>
> Nothing. This is the first page of the course.

## "It works on my machine"

Picture a small fintech team. Ana writes a Python script that reads the day's card transactions and flags suspicious ones. It works perfectly on her laptop. She sends it to Ben, who runs the overnight jobs, and on his machine it fails in four different ways in one afternoon:

1. It crashes immediately: `ModuleNotFoundError: No module named 'pandas'`. Ana installed pandas months ago and forgot about it.
2. Ben installs pandas, and the script crashes again with a syntax error on a line that uses `match`. Ana has Python 3.12; the server has Python 3.9, which doesn't know that keyword.
3. Ben upgrades Python, and the script runs but cannot find `C:\Users\ana\Desktop\cards.csv`. That path exists only on Ana's Windows laptop; the server runs Linux.
4. Ben fixes the path and the script finishes, but it flags almost nothing. Ana's laptop had an environment variable, `RISK_THRESHOLD=0.8`, set in her shell profile. On the server the script fell back to its default of `0.99`.

None of these failures is a bug in the logic Ana wrote. They are all differences between **her machine and Ben's**. Each one took time to diagnose, and the last one is the most dangerous: the script *seemed* to work. In a real company, the fourth failure could let fraud through for weeks.

This is why software teams care so much about environments. The code is only one part of what runs. The rest is everything around it, and it has to be the same, or at least known, everywhere the code runs: on each developer's laptop, on the test server and in production.

## What an environment is made of

We can describe everything around the code as a stack of layers. Each layer is built on the one below it, and each one can cause a failure like Ana's.

> 📘 **Definition: development environment**
>
> The **environment** of a program is everything, apart from its own source code, that determines how it runs. From the bottom up:
>
> 1. **Hardware**: above all, the processor **architecture** (`x86_64`, also called `amd64`, on most Windows PCs and servers; `arm64`, also called `aarch64`, on Apple Silicon Macs and many cloud servers).
> 2. **Operating system**: its **kernel** (the core that manages memory, processes and devices: Linux, Windows NT, Darwin on macOS) and its **userland** (the basic programs and libraries around the kernel).
> 3. **System packages**: programs and libraries installed on the OS (`git`, `curl`, a PostgreSQL client, the SSL libraries…).
> 4. **Language runtime**: the interpreter or compiler, with its version (Python 3.14, Node 22…).
> 5. **Project dependencies**: the third-party libraries the code imports, with their versions (`pandas 2.3`, `fastapi 0.120`…).
> 6. **Configuration**: environment variables, configuration files, secrets (passwords and API keys).
> 7. **External services**: everything the program talks to over the network: databases, other APIs, file storage.

In Ana's story, the failures came from layers 5 (pandas missing), 4 (Python version), 2 (Windows paths on Linux) and 6 (the threshold variable). The table below gives an example of a bug for every layer. Use it as a checklist when something "works on my machine":

| Layer | Typical "works on my machine" failure |
|---|---|
| Hardware | A library ships compiled code only for `x86_64`, and installation fails on an `arm64` Mac |
| Operating system | Paths with `\` instead of `/`; file names that differ only in capitals (`Data.csv` and `data.csv` are the same file on macOS and Windows, but different files on Linux) |
| System packages | The code calls `git` or `psql`, and they aren't installed on the server |
| Runtime | New syntax (`match`, `type` aliases) or new standard modules (`tomllib`) on an old Python |
| Dependencies | A different version of a library changes a default or removes a function |
| Configuration | A variable set on one machine but not on another, as in Ana's story |
| External services | The test database has different data, or a different version, than production |

## Reproducibility

The remedy is to make environments **reproducible**.

> 📘 **Definition: reproducible environment**
>
> An environment is **reproducible** if anyone, on any supported machine, can rebuild it from written instructions or files and get the same result: the same versions of everything, the same configuration and therefore the same behaviour of the code.

The key words are *from files*. If the environment exists only as "what I happened to install on my laptop over two years", nobody can rebuild it, not even its owner after a hard-disk failure. The practical rule is that **the environment should be described in files that live with the code**, in the same repository.

There are three levels of ambition, and professional teams usually combine all three:

1. **Document it.** A README that lists what to install and how. It is better than nothing, but people skip steps and instructions go out of date.
2. **Automate it.** Files that tools read to install exactly what is needed. In Python that is `pyproject.toml` with a **lockfile** (`uv.lock`), which pins the exact version of every dependency (session 3). The configuration goes into an `.env` file, which is not committed, and a template of it, which is (session 12).
3. **Package it.** Ship the environment itself together with the code: a **virtual machine image** or a **container image** that already contains the operating system, the runtime and the dependencies (sessions 11–14).

Each level covers more layers of the stack. A README can't stop someone from installing the wrong Python. A lockfile fixes layer 5, and with a tool like uv also layer 4. A container image fixes layers 2 to 5 at once.

## Three places to run software: host, virtual machine, container

### On the host

The simplest option is to install everything directly on your computer, the **host**. Python, PostgreSQL and the libraries go on the operating system you use every day. It is fast and nothing stands between the program and the machine. The price is that every project shares the same system. Project A needs PostgreSQL 15 and project B needs 17; one upgrade breaks the other. After a year, nobody knows exactly what is installed or why.

### Virtual machines

A **virtual machine** (VM) is a complete computer simulated in software. A program called a **hypervisor** (VirtualBox, UTM, VMware, Hyper-V, Parallels) divides the real hardware into virtual CPUs, memory and disks. On those, it boots a whole second operating system, the **guest OS**. You can run Ubuntu inside your Mac or Windows laptop, with its own kernel, its own users and its own installed software. If you break it, you delete it and create it again.

VMs give strong **isolation**: each guest is a separate computer, and the guest can be a different OS from the host. The cost is weight. Each VM runs a full operating system, takes gigabytes of disk and a fixed slice of RAM, and takes tens of seconds to boot.

### Containers

A **container** isolates a program *without* simulating a computer. Every container on a machine shares the host's **kernel**. The kernel itself keeps containers apart. Each one gets its own view of the filesystem, its own processes and its own network (Linux *namespaces*), and limits on how much CPU and memory it can use (*cgroups*). Inside, the container only needs the program and its libraries, not a whole OS.

A container starts from an **image**: a read-only package with the filesystem the program needs (for example, a minimal Ubuntu userland plus Python plus your code). From one image you can start as many containers as you want, and they will all be identical. The most common tool for building and running them is **Docker** (session 11).

> 📘 **Definition: virtual machine and container**
>
> - A **virtual machine** emulates a complete computer. It runs its own operating system, kernel included, on virtual hardware provided by a hypervisor.
> - A **container** is an isolated group of processes that shares the host's kernel. It sees only the files of its **image**, its own processes and its own network.

The difference has practical consequences:

| | Host | Virtual machine | Container |
|---|---|---|---|
| What is isolated | nothing | the whole computer | processes, files, network |
| Own kernel | — | yes | no, shares the host's |
| Start-up time | — | tens of seconds | under a second |
| Size | — | gigabytes | megabytes to a few hundred MB |
| Runs a different OS | — | yes (Windows on a Mac, Linux on Windows…) | only the same *kind* of kernel: Linux containers need a Linux kernel |
| Typical use | your daily tools | a full, separate workstation or server | packaging and shipping one application |

That "Linux containers need a Linux kernel" line explains something that surprises many people. **Docker Desktop on macOS and Windows runs a small Linux VM in the background**, and your containers run inside it. On Windows, that VM is usually the one from **WSL2** (the Windows Subsystem for Linux). So on a laptop the two technologies are stacked: a VM that provides the Linux kernel, and containers on top. On a Linux server there is no VM in between, and containers run directly on the host kernel.

### Processor architectures

There is one layer that neither VMs nor containers make disappear: the processor. A program compiled for `x86_64` contains machine instructions that an `arm64` processor does not understand, and the other way round. Python code is portable, but many Python libraries (NumPy, the PostgreSQL driver, cryptography) include compiled parts, and they are published separately for each architecture.

This matters in this course in two places:

- **Your laptop.** Apple Silicon Macs (M1 to M5) are `arm64`. Most Windows laptops are `x86_64`, but Windows on ARM (Snapdragon processors) is becoming common. You will see these names in installers: choose the right one.
- **Shipping to the cloud.** A container image built on an `arm64` Mac is, by default, an `arm64` image. Many cloud services expect `x86_64` (`linux/amd64`). If you push the wrong one, the deployment fails with an error like `exec format error`. In session 13 we will build images for the right **platform** explicitly.

Emulation can run code for the other architecture (Rosetta 2 on Macs, QEMU in Docker), but it is slower and sometimes fragile. It is useful in an emergency, but not something to rely on.

## From your laptop to the cloud

Everything so far runs on a machine you own: your laptop, or a server in your company's data centre. This is called **on-premises** ("on-prem"): you own the hardware and you are responsible for every layer of the stack.

**Cloud computing** means renting those layers from a provider (Microsoft Azure, Amazon Web Services, Google Cloud) and paying for what you use. Different services take more or less of the stack off your hands:

| Model | You manage | The provider manages | Example in Azure |
|---|---|---|---|
| On-premises | everything | nothing | — |
| **IaaS** (infrastructure as a service) | OS, runtime, dependencies, app, data | hardware, network, virtualization | Azure Virtual Machines |
| **PaaS / serverless containers** | your container image (app + dependencies) and its configuration | everything below, plus scaling | Azure Container Apps |
| **Managed services** | your data and settings | the whole service | Azure Database for PostgreSQL, Azure Static Web Apps |
| **SaaS** (software as a service) | only how you use it | everything | Microsoft 365, GitHub |

As you move down the table you control less, but you also have less to patch, monitor and keep running at 3 a.m. This is the main trade-off in the cloud. What is still yours is your code, its dependencies and its configuration. That is exactly the part a container packages. **The container is the unit that travels from your laptop to the cloud unchanged**, and that is why the course spends three sessions on it.

The cloud has costs of its own. It is billed by usage, and a forgotten resource keeps billing. Your data lives in someone else's data centre, which matters for regulated industries like finance (where the data is stored, who can access it). And every service is a little different from its equivalent at another provider. We will meet all three issues in sessions 13 and 14.

## The course map

The whole course follows one application, **ledger**, through this stack:

| Sessions | Where it runs | What you add |
|---|---|---|
| 1–2 | your laptop | a known, reproducible working environment |
| 3–6 | your laptop, on the host | a Python project with locked dependencies, under version control, well structured |
| 7–10 | your laptop, with PostgreSQL in a container | a database, a REST API, tests and a web frontend |
| 11–12 | your laptop, all in containers | the three services in Docker Compose, configured from the environment |
| 13–14 | Azure | the same images, running on managed cloud services |

The movement is from left to right in the table of the previous section, from on-premises toward managed services. At every step you will know exactly what is underneath, because you first ran it yourself.

## Common mistakes

- **"A container is a lightweight VM."** It looks like one from the inside, but it has no kernel of its own. This is why it starts in milliseconds, and also why it cannot run a different kind of operating system than the host.
- **"It runs in Docker, so it runs everywhere."** Only on the same architecture, or under emulation. Check the platform of your images.
- **"I'll remember how I set it up."** You won't, and your teammates can't read your memory. If it isn't in a file in the repository, it isn't reproducible.
- **Confusing *environment* and *configuration*.** The configuration (layer 6) is one part of the environment. It is the part that *should* differ between your laptop and production (a different database, different keys). Everything else should be the same.

> ✅ **Key takeaways**
>
> - A program's **environment** is everything except its own code: hardware, operating system, system packages, runtime, dependencies, configuration and external services. A difference in any layer can break it.
> - An environment is **reproducible** when it can be rebuilt from files that live with the code: documentation, then automation (lockfiles), then packaging (images).
> - A **VM** emulates a whole computer with its own kernel. A **container** isolates processes but shares the host kernel, so it is far lighter. On Mac and Windows, Docker runs containers inside a small Linux VM.
> - The processor **architecture** (`arm64` or `x86_64`) is part of the environment. Images must be built for the platform where they will run.
> - The cloud models (IaaS, PaaS, managed services, SaaS) differ in **which layers you manage**. The container image is the part that is yours in every model, and the unit that travels from laptop to cloud.

## Exercises

**1.** For each failure, name the layer of the environment that causes it: *(a)* `pip install` fails with "no matching distribution" on a new Mac but works on a colleague's Windows laptop; *(b)* the app connects to the test database instead of the production one; *(c)* `import tomllib` fails with `ModuleNotFoundError`; *(d)* the script works on the developer's Mac but cannot find `Config.yaml` on the Linux server, where the file is called `config.yaml`.

<details>
<summary>✅ <b>Solution</b></summary>

- *(a)* **Hardware** (architecture): the library has a compiled package for `x86_64` but not for `arm64`. If no version for your platform exists, pip has nothing to install.
- *(b)* **Configuration**: the database address comes from a setting or environment variable that points to the wrong place. The services are fine; the configuration chooses the wrong one.
- *(c)* **Runtime**: `tomllib` is part of the standard library only from Python 3.11. The code runs on an older Python.
- *(d)* **Operating system**: macOS file systems are case-insensitive by default, so `Config.yaml` finds `config.yaml`. Linux is case-sensitive, so it doesn't.

</details>

**2.** A colleague says: "Our app runs in Docker, so we don't need to pin library versions in `pyproject.toml`; the image already contains them." Is that right?

<details>
<summary>✅ <b>Solution</b></summary>

Only half right. The image freezes the versions *that were installed when it was built*. But the image is rebuilt from the `Dockerfile` every time someone changes the code. If the dependencies aren't pinned, each rebuild can install newer versions, and two images built a week apart can behave differently. The lockfile makes the **build** reproducible; the image makes the **result** portable. You need both.

</details>

**3.** You need to: *(a)* try a Linux command-line tool for an afternoon without installing anything on your Windows laptop; *(b)* run the same API in ten identical copies to handle more traffic; *(c)* test how your company's software installs on a fresh Windows 11 machine. For each case, would you choose a VM or a container? Why?

<details>
<summary>✅ <b>Solution</b></summary>

- *(a)* A **container**: `docker run -it ubuntu` gives you a disposable Linux shell in a second, and nothing remains when you exit. A VM would also work, but it is much heavier for an afternoon.
- *(b)* **Containers**: starting ten copies from the same image is fast and cheap, and they are identical by construction. This is exactly what services like Azure Container Apps do.
- *(c)* A **VM**: you need a complete Windows operating system, kernel included, in a clean state. A container shares the host's kernel, so it can't give you a fresh Windows.

</details>

**4.** Ana builds a container image on her M3 MacBook and deploys it to a cloud service that runs `x86_64` servers. The deployment fails with `exec format error`. Explain what happened and how she can avoid it.

<details>
<summary>✅ <b>Solution</b></summary>

Her Mac is `arm64`, so by default Docker built an `arm64` image: the Python interpreter and every compiled library inside it contain ARM instructions. The cloud server's processor is `x86_64`, cannot execute them, and the kernel refuses with `exec format error`. She must build the image for the target platform, `linux/amd64` (with Docker, `docker build --platform linux/amd64 …`), or let the image be built by a CI service that runs on `x86_64`. We do both in session 13.

</details>

**5.** A bank wants to run its new customer-facing API. Option A: buy servers for its own data centre. Option B: Azure Virtual Machines. Option C: Azure Container Apps. For each option, list two things the bank's team must still manage, and give one reason a bank might still choose option A.

<details>
<summary>✅ <b>Solution</b></summary>

- **A, on-premises:** everything, for example the physical hardware and its replacement, and the operating system updates (plus networking, power, physical security…).
- **B, IaaS:** the operating system (patches, security updates) and the runtime and dependencies installed on it, plus scaling: deciding how many VMs to run.
- **C, serverless containers:** the container image (the app, its dependencies, the base image's security updates) and its configuration and secrets. Scaling is configured, not managed by hand.
- **Why choose A:** regulation or data-residency rules that require full control of where customer data lives and who can touch the hardware. Others are an existing investment in its own data centre, or predictable, constant loads, which can be cheaper on hardware it owns than on pay-per-use.

</details>

[← Back to the session](index.md) [Next: Lab 1 →](lab.md)
