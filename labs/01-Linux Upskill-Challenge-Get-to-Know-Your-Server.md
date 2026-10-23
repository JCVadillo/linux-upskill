# 🐧 01 – Linux Upskill Challenge – Day 1: Get to Know Your Server
*Linux Upskill Challenge – Day 1*
*(Connect over SSH and run the basic health and inventory commands)*

---

## 🎯 Lab Summary

Day 1 of the Linux Upskill Challenge is about getting comfortable with a remote Linux server: logging in over SSH and running the everyday commands that tell you what the box is, what hardware it has, and how busy it is. Nothing fancy yet. Hardening comes on Day 3.

The idea behind it is simple. Before you can spot anything *abnormal* on a server, you need to know what *normal* looks like. This lab is my baseline snapshot of a fresh Ubuntu server.

The main tasks were to:
- Connect to the server over SSH using a key pair instead of a password
- Identify the OS, kernel, uptime and current user
- Check the CPU and disk layout
- Measure memory, CPU load, disk space and network activity

---

## ⚙️ Environment

| Component | Detail |
|---|---|
| **Server** | Google Compute Engine VM (`us-central1-a`) |
| **OS** | Ubuntu Server 24.04.5 LTS (Noble Numbat) |
| **Kernel** | `7.0.0-1014-gcp` |
| **Client** | macOS Terminal (OpenSSH client) |
| **Login method** | SSH key (`ed25519`), password login disabled |
| **User** | `jc.vadillo87` (regular user, `$` prompt) |
| **Server public IP** | Redacted (`<SERVER_PUBLIC_IP>`) |

---

## 🔑 Connecting to the Server

The server was created with my public key, so there's no password prompt. I point the client to the private key with `-i`:

```bash
ssh -i ~/.ssh/id_ed25519 jc.vadillo87@<SERVER_PUBLIC_IP>
```

The `$` at the end of the prompt means I'm a normal user. A root shell would show `#`.

To confirm that password logins really are off (and that I'm not just assuming it), I grepped the SSH daemon config:

```bash
sudo grep -i "PasswordAuthentication" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*
```

| File | Line found | What it means |
|---|---|---|
| `/etc/ssh/sshd_config` | `#PasswordAuthentication yes` | Commented out, so it's just the default placeholder and does nothing |
| `/etc/ssh/sshd_config.d/60-cloudimg-settings.conf` | `PasswordAuthentication no` | The cloud image sets it to **no**. This is the line that counts |

> 💡 The grep shows what's *written* in the files, not what the daemon actually loaded. `sudo sshd -T | grep passwordauthentication` would show the effective value. I didn't run it this time, but it's the check I'd use on a real system.

---

## 🖥️ General Information

### `cat /etc/os-release`

```bash
cat /etc/os-release
```

| Field | Value | Meaning |
|---|---|---|
| `PRETTY_NAME` | Ubuntu 24.04.5 LTS | Human-readable version |
| `VERSION_CODENAME` | noble | Release codename (Noble Numbat) |
| `ID` / `ID_LIKE` | ubuntu / debian | Ubuntu is Debian-based, so `apt` and `.deb` packages apply |
| `SUPPORT_URL`, `BUG_REPORT_URL`... | ubuntu.com links | Where the vendor documents support and bugs |

I picked this over `lsb_release -a` because `/etc/os-release` exists on practically every modern distro, while `lsb_release` may not be installed.

### `uname -a`, `uptime` and `whoami`

```bash
uname -a
uptime
whoami
```

| Command | Output (trimmed) | Meaning |
|---|---|---|
| `uname -a` | `Linux ubuntu-linux-server 7.0.0-1014-gcp ... SMP PREEMPT ... x86_64 GNU/Linux` | Hostname, kernel version, and 64-bit architecture. The `-gcp` suffix is the kernel flavor tuned for Google Cloud |
| `uptime` | `up 44 min, 1 user, load average: 0.00, 0.00, 0.00` | Time since boot, logged-in users, and load over the last 1, 5 and 15 minutes |
| `whoami` | `jc.vadillo87` | The account I'm logged in as. Linux is case-sensitive here |

A load average of `0.00` on all three values means the CPU had basically nothing waiting to run.

---

## 🔧 Hardware Information

### `lscpu`

```bash
lscpu
```

| Field | Value | Meaning |
|---|---|---|
| **Architecture** | x86_64 | 64-bit Intel/AMD instruction set |
| **Model name** | AMD EPYC 7B12 | The physical CPU model behind the VM |
| **CPU(s)** | 2 | Logical CPUs the OS can use |
| **Thread(s) per core / Core(s) per socket** | 2 / 1 | One core with two threads, which is why 2 CPUs show up |
| **Hypervisor vendor / Virtualization type** | KVM / full | Confirms this is a VM, not bare metal |
| **Flags** | long list | CPU feature list (AES, AVX2, SHA...). Useful for compatibility checks, not for a daily read |

So the "2 CPUs" are really two hardware threads of a single core. That's good to know when reading CPU graphs later.

### `lsblk`

```bash
lsblk
```

| Device | Size | Mountpoint | Meaning |
|---|---|---|---|
| `sda` | 10G | — | The whole virtual disk |
| `sda1` | 9G | `/` | Root filesystem, where the OS lives |
| `sda14` | 4M | — | Small partition with no mountpoint (typically BIOS boot) |
| `sda15` | 106M | `/boot/efi` | EFI system partition |
| `sda16` | 913M | `/boot` | Kernel and boot files |
| `loop0-2` | 66.8M / 353.3M / 44.7M | `/snap/...` | Snap packages (`core24`, `google-cloud-cli`, `snapd`) mounted read-only |

The `loop` devices look like disks but they're just snap packages mounted as images. They're not real storage, so ignore them when counting space.

---

## 📈 Memory and CPU Usage

### `free -h`

```bash
free -h
```

| | total | used | free | shared | buff/cache | available |
|---|---|---|---|---|---|---|
| **Mem** | 953Mi | 325Mi | 505Mi | 964Ki | 231Mi | 627Mi |
| **Swap** | 0B | 0B | 0B | | | |

- **free** is memory nobody is touching at all.
- **available** is the number that matters. It's what new programs can actually get, because the kernel hands back `buff/cache` when needed.
- **Swap is 0B.** There's no swap space configured, so if the 953Mi ever runs out, the kernel's out-of-memory killer will start killing processes. Fine for a lab, worth remembering for anything real.

### `top`

```bash
top
```

Key lines from the header:

| Line | Value | Meaning |
|---|---|---|
| `up` / `load average` | 1:25 / 0.00, 0.00, 0.00 | Uptime and CPU load, same idea as `uptime` |
| `Tasks` | 110 total, 1 running, 109 sleeping, 0 zombie | Processes by state. No zombies is what you want |
| `%Cpu(s)` | 99.8 id | 99.8% idle. The server is doing almost nothing |
| `MiB Mem` | 953.6 total, 319.8 used, 231.9 buff/cache | Same picture as `free` |
| `MiB Swap` | 0.0 total | Confirms there's no swap |

In the process table, the interesting columns are `PID`, `USER`, `%CPU`, `%MEM` and `COMMAND`. Everything shown is owned by `root` and sleeping (`S`) or idle kernel workers (`I`), headed by `systemd` as PID 1.

---

## 💽 Disk Usage

### `df -h`

```bash
df -h
```

| Filesystem | Size | Used | Avail | Use% | Mounted on | Notes |
|---|---|---|---|---|---|---|
| `/dev/root` | 8.7G | 3.1G | 5.7G | 35% | `/` | Main disk, the one to watch |
| `/dev/sda16` | 881M | 86M | 734M | 11% | `/boot` | Plenty of room for kernel updates |
| `/dev/sda15` | 105M | 6.2M | 99M | 6% | `/boot/efi` | EFI partition |
| `tmpfs` (several) | 5M to 477M | ~0 | — | 0 to 1% | `/dev/shm`, `/run`... | Lives in RAM, not on disk |

`/dev/root` is the same partition `lsblk` calls `sda1`. The 9G vs 8.7G difference is filesystem overhead plus unit rounding.

---

## 🌐 Network

### `ip address`

```bash
ip address
```

| Interface | Address | Meaning |
|---|---|---|
| `lo` | `127.0.0.1/8` | Loopback. The server talking to itself |
| `ens4` | `10.128.0.2/32` | Private IP given by Google's DHCP (`dynamic`). State `UP`, MTU 1460 |
| `ens4` (IPv6) | `fe80::4001:aff:fe80:2/64` | Link-local only, so no public IPv6 here |

The public IP I use for SSH doesn't appear anywhere in this output. Google NATs it in front of the VM, and the OS only knows its private `10.128.0.2`. The `/32` mask and the 1460 MTU are both GCP defaults.

### `sudo iftop -i ens4`

```bash
sudo iftop -i ens4
```

| Row | Cumulative | Peak | Rate (2s / 10s / 40s avg) |
|---|---|---|---|
| **TX** (sent) | 32.1KB | 3.21Kb | 1.47Kb / 934b / 0.99Kb |
| **RX** (received) | 36.3KB | 16.0Kb | 208b / 208b / 0.99Kb |
| **TOTAL** | 68.4KB | 19.2Kb | 1.67Kb / 1.12Kb / 1.98Kb |

The two connections listed:

| Remote host | What it is |
|---|---|
| `<home-ISP-hostname>` (a German ISP residential address) | My own SSH session. The server is talking to my Mac |
| `metadata.google.internal` | Google's internal metadata service, which the guest agent on the VM polls regularly. Tiny, constant, normal |

Total traffic is a few kilobits per second, and everything on the wire is either me or the cloud provider's own plumbing. That's the baseline.

---

## 📊 Results Summary

| Area | Finding |
|---|---|
| **OS** | Ubuntu 24.04.5 LTS (noble), Debian-based |
| **Kernel** | 7.0.0-1014-gcp, x86_64 |
| **CPU** | AMD EPYC 7B12, 2 vCPUs (1 core, 2 threads), KVM |
| **Memory** | 953Mi total, 627Mi available, **no swap** |
| **Disk** | 10G disk, root at 35% used |
| **Load** | 0.00 / 0.00 / 0.00, ~99.8% idle |
| **Network** | `ens4` at `10.128.0.2`, only my SSH session and GCP metadata traffic |
| **Access** | SSH key only, password auth set to `no` |

---

## 🗂️ Commands Covered but Not Documented

The Day 1 guide lists a lot of commands, and several of them show the same information in a different shape. I ran these too, but left their output out of the main write-up:

| Command | Why it's not documented |
|---|---|
| `lsb_release -a` | Same info as `/etc/os-release`, plus a "No LSB modules" notice that adds nothing |
| `sudo lshw` | Huge output, and mostly VM identity fields (serials, UUID) that I don't want in a public repo. `lscpu` and `lsblk` give the useful parts more cleanly |
| `lspci` | Every device is a Red Hat Virtio virtual device. On bare metal it's valuable (NICs, GPUs, controllers), but on a cloud VM it tells you very little |
| `vmstat` | Single run only shows averages since boot, and the memory data duplicates `free`. It earns its place with an interval, like `vmstat 1 5` |
| `htop` | Same data as `top` with a friendlier layout, but it needs installing. `top` is on every server, so that's the one I documented |
| `du -h` | I only ran it in my home folder, which is 44K. Nothing to learn, though it's the right tool for hunting down what's eating a disk |
| `netstat -i` | Legacy `net-tools` command. It showed `ens4` with ~397k packets in, ~327k out and 0 errors or drops, which is healthy, but `ip` and `iftop` cover the same ground |

Other commands from the guide (`who`, `w`, `lsusb`, `ifstat`, `cat /proc/uptime`) are not part of this write-up.

---

## 🧠 Reflections / Notes

- **A baseline is the real deliverable.** A server at 99.8% idle with two network flows is easy to describe. If something later eats CPU or opens a connection to an odd IP, I'll notice because I know what quiet looks like.
- **`available`, not `free`.** `free` made it look like only half the RAM was untouched, but `available` is the honest number. Reading the wrong column is an easy mistake.
- **No swap on a 1 GB machine** is the one thing here I'd flag in a review. Nothing is wrong today, but there's no safety net.
- **Cloud VMs hide things.** The public IP isn't on the interface, `lspci` is all virtual hardware, and `lscpu` reports one core with two threads. You have to remember you're looking at a slice of someone else's hardware.
- **I checked the SSH config instead of trusting it.** Seeing `PasswordAuthentication no` in the cloud-init drop-in file, while the main file still has a commented `yes`, was a nice reminder that config on Linux often lives in more than one place.
- **Redaction matters even in a lab.** Screenshots had the server's public IP, a home ISP hostname, and hardware serials, so I masked or skipped those before publishing.

---

## 📚 Key Skills Demonstrated

- Key-based SSH authentication from the macOS command line
- Verifying SSH daemon settings with `grep` on config files
- System identification with `/etc/os-release`, `uname`, `uptime` and `whoami`
- Reading CPU and disk layout with `lscpu` and `lsblk`
- Interpreting memory (`free -h`) and process/CPU load (`top`)
- Checking disk usage with `df -h`
- Reviewing network configuration and live traffic with `ip address` and `iftop`
- Choosing relevant tools and sanitizing sensitive details before documenting

---

## 🔗 References

- [Linux Upskill Challenge – Day 1: Get to know your server](https://linuxupskillchallenge.org/01)

---

*Nothing on this server was doing anything interesting, and that was the point. Day 1 is about learning to read the dashboard so that on a later day, when something does look odd, I'll recognize it.*

