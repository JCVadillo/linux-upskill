# 🐧 Linux Upskill Challenge Labs

**Practical Linux server administration labs and write-ups** documenting hands-on exercises completed through the [Linux Upskill Challenge](https://linuxupskillchallenge.org/) platform, focusing on remote server management, command-line operations, service administration, and system security on cloud infrastructure.

---

## 👤 About This Repository

This repository serves as a structured record of my daily learning journey through the Linux Upskill Challenge. Each lab entry documents core sysadmin concepts, essential CLI tools, system performance monitoring, user management, and security hardening techniques — performed on a remote Linux server hosted on Google Cloud Platform (GCP).

Labs cover fundamental to advanced operational topics including remote SSH access, file system management, process control, networking telemetry, log analysis, package management, and basic shell scripting.

---

## 🗂️ Lab Index

| # | Lab Name | Category | Tools Used | Status |
|---|---|---|---|---|
| 01 | [Day 1 – Get to Know Your Server](./labs/01-get-to-know-your-server.md) | Server Provisioning & Recon | GCP, SSH, `lsblk`, `free`, `vmstat`, `iftop` | ✅ Complete |
| 02 | [Swap File Configuration](./labs/02-swap-file-configuration.md) | Memory Management | `fallocate`, `chmod`, `mkswap`, `swapon`, `/etc/fstab` | ✅ Complete |

---

## 🛠️ Environment

- **Primary OS / Target:** Ubuntu 24.04 LTS (Google Cloud Platform – Compute Engine `e2-micro`)
- **Local Environment:** macOS Terminal
- **Connection Method:** SSH Key Pair (Ed25519)

---

## 📁 Repository Structure

```
linux-upskill/
├── README.md
└── labs/
    ├── 01-get-to-know-your-server.md
    └── 02-swap-file-configuration.md
```

---

## 🔗 Connect

- 💼 LinkedIn: [Juan Carlos Vadillo](https://www.linkedin.com/in/juan-carlos-vadillo/)
- 🏆 TryHackMe Profile: [jcvc87](https://tryhackme.com/p/jcvc87)
- 📁 Other Portfolio: [tryhackme-labs](https://github.com/JCVadillo/tryhackme-labs)
- 📁 Other Portfolio: [google-cybersecurity-cert-portfolio](https://github.com/JCVadillo/google-cybersecurity-cert-portfolio)
