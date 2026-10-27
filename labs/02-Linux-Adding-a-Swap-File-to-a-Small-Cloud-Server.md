# 🧠 02 – Linux – Adding a Swap File to a Small Cloud Server
*Self-directed lab, follow-up to Linux Upskill*
*(Create, enable and persist a 2 GB swap file on Ubuntu Server)*

---

## 🎯 Lab Summary

On the previous activity, `free -h` showed my server had about 1 GB of RAM and **no swap at all**. This lab explains what swap is, why it matters, and walks through adding a swap file to fix that.

The main tasks were to:
- Check the disk and filesystem to see where a swap area could live
- Create a 2 GB swap file and lock down its permissions
- Format it as swap, turn it on, and make it survive a reboot
- Confirm the system actually sees the new swap

---

## 📖 What Is Swap?

Swap is disk space the kernel can use as overflow when RAM fills up. Inactive memory pages get moved to disk to free RAM for things that need it right now. It's much slower than RAM, so it's a safety net, not a performance upgrade.

Without swap, a full-RAM server has one option left: the **OOM killer** starts terminating processes. On a 1 GB machine that could easily be `sshd` or whatever service I care about.

### Swap partition vs. swap file

| | Swap partition | Swap file |
|---|---|---|
| **Where it lives** | A dedicated partition on the disk | A regular file inside an existing filesystem |
| **Flexibility** | Low. Resizing means repartitioning | High. Delete it and make a new one with a different size |
| **Setup** | Needs free or unpartitioned disk space | Needs only free space on the filesystem |
| **Speed** | Direct block access | Practically the same on modern kernels |
| **Typical use** | Older setups, hibernation-focused machines | Cloud VMs, Ubuntu's default for new installs |

About speed: I went in assuming the file was slower because the system has to go through the filesystem. That's true in theory, and it was true on older kernels. On current ones, the kernel works out where the file's blocks sit on disk once, when swap is enabled, and after that it reads and writes them directly. In practice the difference isn't noticeable.

### Why I picked a swap file

My server is a Google Compute Engine `e2-micro` with a 10 GB disk. The `lsblk` output on Day 1 shows the disk is already fully split between `sda1` (root) and the small boot/EFI partitions, so there's no free space for a new partition. A swap file on the root filesystem was the practical choice, and easier to resize later if I change my mind.

---

## ⚙️ Environment

| Component | Detail |
|---|---|
| **Server** | Google Compute Engine `e2-micro` (`us-central1-a`) |
| **OS** | Ubuntu Server 24.04 LTS |
| **RAM** | 953Mi |
| **Disk** | 10G, root filesystem `ext4` on `sda1` |
| **Access** | SSH key, commands run from a root shell |
| **Swap before** | 0B |

---

## 📏 Rule of Thumb: How Much Swap?

| Physical RAM | Recommended swap |
|---|---|
| Less than 2 GB | 2x the RAM |
| 2 GB to 8 GB | Equal to the RAM (1x) |
| 8 GB to 64 GB | 4 GB to 8 GB (fixed) |
| More than 64 GB | Usually not needed (4 GB at most as a precaution) |

This is a guideline, not a law. Workload matters, and so does whether you plan to use hibernation, which needs swap at least as big as RAM. Some distro docs also word the upper tiers as "at least 4 GB" instead of a cap, so I treat the numbers as a starting point.

My case: 953Mi is under 2 GB, so the rule says about 2x RAM, which lands at roughly **2 GB**. That's what I created.

---

## 🧪 Step by Step

### Step 1 – Check the filesystem and free space

Before creating a 2 GB file I wanted to know what filesystem I'm on and whether the disk can afford it:

```bash
lsblk -f
```

| Field | Value | Why it matters |
|---|---|---|
| `sda1` FSTYPE | `ext4` | Fully supports swap files |
| `sda1` FSAVAIL / FSUSE% | 5.6G free / 35% used | Plenty of room for 2G |
| `sda1` MOUNTPOINT | `/` | The swap file will live on the root filesystem |
| `loop0` to `loop3` | `squash` at 100% | Snap packages. Always 100% because they're read-only images, not a problem |

### Step 2 – Allocate the file and lock down permissions

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
```

| Command | What it does |
|---|---|
| `fallocate -l 2G /swapfile` | Reserves a 2 GB file instantly, without writing 2 GB of zeros |
| `chmod 600 /swapfile` | Only root can read or write it |

The `chmod` isn't cosmetic. Swap holds chunks of whatever was in memory, which can include passwords, session tokens and keys. If any user could read the file, they could read that too. If you skip it, `swapon` will warn you about insecure permissions.

### Step 3 – Format the file as swap

```bash
mkswap /swapfile
```

Output: `Setting up swapspace version 1, size = 2 GiB (2147479552 bytes)`, no label, plus a generated UUID.

`mkswap` writes the swap header into the file. The size is a few KB under 2 GiB because the header takes the first page.

### Step 4 – Enable it and make it permanent

```bash
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

| Command | What it does |
|---|---|
| `swapon /swapfile` | Activates the swap file right now |
| `echo '...' >> /etc/fstab` | Adds a line so it's activated again at every boot |

The `fstab` line, field by field:

| Field | Value | Meaning |
|---|---|---|
| Device | `/swapfile` | What to mount |
| Mount point | `none` | Swap has no mount point |
| Type | `swap` | Filesystem type |
| Options | `sw` | Standard swap options |
| Dump / Pass | `0 0` | Not backed up, not checked at boot |

Be careful with `>>` here. A single `>` would overwrite `/etc/fstab` and could leave the machine unable to boot cleanly. `>>` appends.

### Step 5 – Confirm

```bash
free -h
```

| | total | used | free | shared | buff/cache | available |
|---|---|---|---|---|---|---|
| **Mem** | 953Mi | 342Mi | 459Mi | 980Ki | 268Mi | 611Mi |
| **Swap** | 2.0Gi | 0B | 2.0Gi | | | |

Swap went from `0B` to `2.0Gi`, and the 0B in "used" is expected. Nothing needs it while RAM is mostly idle.

---

## 📊 Results Summary

| Item | Before | After |
|---|---|---|
| **Swap total** | 0B | 2.0Gi |
| **Swap type** | None | File (`/swapfile`, ext4) |
| **Permissions** | n/a | `600` (root only) |
| **Persistent across reboot** | n/a | Configured in `/etc/fstab` |
| **Disk used by swap** | 0 | 2G of the 5.6G that was free |

---

## 🧠 Reflections / Notes

- **Swap is insurance.** It won't make the server faster. It stops the OOM killer from picking off processes the moment RAM spikes.
- **On a cloud VM, swap is slow.** The disk behind this VM is network-attached, so if the server really starts swapping heavily, it'll crawl. 2 GB gives breathing room, but if I see constant swap use, the real fix is more RAM or a lighter workload.
- **`chmod 600` is a security step.** It's easy to treat it as part of the recipe. It's there because swap can leak memory contents to anyone who can read the file.
- **`>>` vs `>`** is the kind of detail that bites you exactly once. Before editing `/etc/fstab` on anything that matters, I'd back it up first with `cp /etc/fstab /etc/fstab.bak`.
- **What I didn't verify:** I confirmed with `free -h` only. I did not run `swapon --show` or `cat /etc/fstab` to read the entry back, and I did not reboot to prove persistence works. The `fstab` line is the standard one, but until I reboot and check, "permanent" is what I configured, not what I tested.
- **Not tuned yet:** I left `vm.swappiness` at its default. That setting controls how eagerly the kernel uses swap, and it's a natural next thing to read about.

---

## 📚 Key Skills Demonstrated

- Explaining swap and the trade-offs between a swap partition and a swap file
- Sizing swap from physical RAM
- Inspecting filesystems and free space with `lsblk -f`
- Creating and securing a swap file (`fallocate`, `chmod 600`, `mkswap`)
- Enabling swap and persisting it with `swapon` and `/etc/fstab`
- Verifying memory and swap with `free -h`
- Identifying my own gaps (no persistence test) instead of declaring the job done

---

## 🔗 References

- [Linux Upskill Challenge – Day 1: Get to know your server](https://linuxupskillchallenge.org/01) (where swap came up as an extension topic)
- Source article used for the commands: *(https://wiki.archlinux.org/title/Swap)* **Please note** this is a guide, commands may be different according to your version

---

*A server with no swap works fine right up until the day it doesn't. Two gigabytes of disk is a cheap way to make that day less dramatic.*

