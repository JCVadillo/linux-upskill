# ⚡ 03 – Linux Upskill Challenge – Day 3: Power Trip
*Linux Upskill Challenge – Day 3*
*(Use sudo instead of root, then change the hostname and the timezone)*

---

## 🎯 Lab Summary

Day 3 is about **power**: root can do anything on a Linux box, including breaking it with one typo. The modern habit is to stop logging in as root and let a trusted regular user borrow root's powers one command at a time with `sudo`.

In this lab I tested exactly where that line sits. I tried privileged actions as a normal user, watched them get refused, repeated them with `sudo`, and then used those powers for two real administrative changes: the hostname and the timezone.

The main tasks were to:
- Change my own password with `passwd`
- Compare commands with and without `sudo` (`adduser`, `cat /etc/shadow`, `reboot`)
- Confirm the reboot and review login history (`uptime`, `last`, `lastb`)
- Compare `sudo -i` and `sudo -s`
- Rename the server with `hostnamectl` and make the name survive reboots
- Change the timezone with `timedatectl`

---

## ⚙️ Environment

| Component | Detail |
|---|---|
| **Server** | Google Compute Engine VM |
| **OS** | Ubuntu Server 24.04 LTS (same VM as Day 1) |
| **Client** | macOS Terminal (OpenSSH client) |
| **Login method** | SSH key (`ed25519`) |
| **User** | `jc.vadillo87` (regular user with sudo rights) |
| **Server public IP** | Redacted (`<SERVER_PUBLIC_IP>`) |
| **Hostname** | `ubuntu-linux-server` → `cloudbox` |
| **Timezone** | `Etc/UTC` → `Europe/Berlin` |

---

## 👥 Who Am I and What Can I Do?

The guide splits Linux accounts into three types. This is the mental model for the whole day:

| Type | Can do | Example in this lab |
|---|---|---|
| **root** | Everything, at every level, for every user | The `root@...#` shell I get from `sudo -i` |
| **sudoer** | Some or all global changes, by prefixing commands with `sudo` | `jc.vadillo87` |
| **Regular user** | Only local changes (own files and environment) | `oots2`, the test user I created |

A **local** change affects one user. A **global** change (a new user, a hostname, a timezone) affects everyone on the system, which is why those need `sudo`.

---

## 🔐 Changing My Password

```bash
passwd
```

The prompts, in order: current password → new password → confirm. The result was `passwd: password updated successfully`. As usual, nothing is echoed while typing.

> 💡 I log in with an SSH key, so this password isn't my front door. It still matters as a second line of defense, and the guide's advice applies: long, strong and unique.

---

## 🧪 The Power of `sudo`: Same Command, Two Results

Each test was run first as a regular user, then with `sudo`.

| # | Without `sudo` | With `sudo` |
|---|---|---|
| 1 | `adduser oots2` → `fatal: Only root may add a user or group to the system.` | `sudo adduser oots2` → user and group created (UID/GID 1004), home directory created, password set |
| 2 | `cat /etc/shadow` → `Permission denied` | `sudo cat /etc/shadow` → file contents displayed |
| 3 | `reboot` → `Call to Reboot failed: Interactive authentication required.` | `sudo reboot` → `The system will reboot now!` |

### What each one shows

- **`adduser`:** creating accounts is a global change, so it's reserved for root. With `sudo`, it copied the default files from `/etc/skel`, created `/home/oots2`, and put the user in the `users` group.
- **`/etc/shadow`:** this file holds the hashed passwords, a prime target for attackers running offline crackers. A regular user can't even read it. The `sudo` output is left out of this write-up because it contains real hashes.
- **`reboot`:** the failure message is about authorization. A normal user isn't allowed to restart the machine. Running it with `sudo` was enough.

---

## 🔄 Confirming the Reboot

After reconnecting over SSH:

```bash
uptime
```

```text
14:40:38 up 3 min,  1 user,  load average: 0.02, 0.01, 0.00
```

The `sudo reboot` broadcast message was timestamped `14:36:33 UTC`, and `uptime` read `14:40:38` with **3 minutes** up. The two timestamps agree, so the machine really did restart and not just my session.

---

## 🕵️ Login History

```bash
last oots2
last root
lastb root          # without sudo
sudo lastb root
```

| Command | Result | What it means |
|---|---|---|
| `last oots2` | Empty, only `wtmp begins Fri Oct 16 12:41:59 2026` | The test user was created but has never logged in |
| `last root` | Empty, same `wtmp begins` line | No successful root logins since the log starts on Oct 16 |
| `lastb root` | `cannot open /var/log/btmp: Permission denied` | The failed-login log is itself privileged |
| `sudo lastb root` | Empty, only `btmp begins Sun Nov 1 00:03:29 2026` | No failed root login attempts recorded since Nov 1 |

Two things worth noting:
- An **empty result is still a result**. It's the expected outcome for an account nobody has used.
- `lastb` is restricted for a good reason: it records the usernames people *tried* to break in with, and mistyped passwords can end up there.

---

## 🧑‍💻 `sudo -i` vs `sudo -s`

```bash
sudo -i
exit
sudo -s
```

| | `sudo -i` | `sudo -s` |
|---|---|---|
| **Prompt after running** | `root@ubuntu-linux-server:~#` | `root@ubuntu-linux-server:/home/jc.vadillo87#` |
| **Directory** | Moved to root's home | Stayed in my own home directory |
| **Environment** | Root's login environment | Mostly my current environment |
| **Best for** | A series of admin tasks that need a clean root environment | A quick root shell without leaving my working folder |

Both prompts end with `#` instead of `$`, which is the visual cue that I'm root. I used `exit` to drop back to my normal user after `sudo -i`.

---

## 🏷️ Changing the Hostname

```bash
sudo hostnamectl set-hostname cloudbox
hostnamectl
```

`hostnamectl` now reports `Static hostname: cloudbox`. No reboot is needed. The prompt still showed `ubuntu-linux-server` in the same session, because the prompt only reads the hostname when a new shell starts.

### Making it survive reboots

On cloud servers, cloud-init can reset the hostname at boot. To prevent that, the guide says to edit `/etc/cloud/cloud.cfg`:

```yaml
# This will cause the set+update hostname module to not operate (if true)
preserve_hostname: true
```

| Setting | Effect |
|---|---|
| `preserve_hostname: false` | cloud-init may set the hostname again on boot |
| `preserve_hostname: true` | cloud-init's hostname module is skipped, so my `cloudbox` name stays |

---

## 🕐 Changing the Timezone

```bash
timedatectl
timedatectl list-timezones
sudo timedatectl set-timezone Europe/Berlin
timedatectl
```

| Field | Before | After |
|---|---|---|
| **Local time** | Thu 2026-11-05 15:31:43 UTC | Thu 2026-11-05 16:33:55 CET |
| **Universal time** | Thu 2026-11-05 15:31:43 UTC | Thu 2026-11-05 15:33:55 UTC |
| **Time zone** | `Etc/UTC (UTC, +0000)` | `Europe/Berlin (CET, +0100)` |
| **System clock synchronized** | yes | yes |
| **NTP service** | active | active |
| **RTC in local TZ** | no | no |

The universal time is the same clock in both, only two minutes apart because the "after" capture was taken later. Only the *local* representation changed, and the hardware clock stays in UTC (`RTC in local TZ: no`), which is the recommended setup for servers.

### Why it matters
- **Scheduled tasks** will run on the new local time.
- **Log timestamps** under `/var/log` will use the new zone, so there is a visible "jump" in the records at the moment of the change.

---

## 📊 Results Summary

| Area | Finding |
|---|---|
| **Password** | Changed with `passwd` |
| **Privilege boundary** | `adduser`, `cat /etc/shadow` and `reboot` all failed without sudo and worked with it |
| **Reboot** | Confirmed by `uptime` (3 min), consistent with the reboot timestamp |
| **Login history** | No logins for `oots2` or `root`, and no failed root attempts since Nov 1 |
| **Root shells** | `sudo -i` lands in root's home, `sudo -s` keeps my current directory |
| **Hostname** | `ubuntu-linux-server` → `cloudbox` (persistence setting applied, not tested) |
| **Timezone** | `Etc/UTC` → `Europe/Berlin (CET, +0100)` |

---

## 🧠 Reflections / Notes

- **Permission errors are the feature, not the bug.** Seeing the same command fail and then succeed made the sudo model concrete in a way reading about it didn't.
- **`/etc/shadow` is the clearest example.** One file, zero access for normal users. If an attacker lands as a regular user, they can't just grab the hashes.
- **`sudo` is a gate, not an audit trail by itself.** The guide points to `journalctl` for sudo history, and I didn't run it. Without it, I only know that sudo worked, not how to review what was done with it.
- **Empty `last` output isn't a failure.** It means nothing to report, which is what I want to see for `root`.
- **Config changes need a test.** I set `preserve_hostname: true`, but until I reboot and see `cloudbox` again, it's an assumption rather than a result.
- **Timezone is a decision, not a cosmetic change.** Fleets usually stay on UTC. I switched to local time for this lab, knowing logs and cron schedules will now follow it.

---

## 📚 Key Skills Demonstrated

- Distinguishing root, sudoers and regular users
- Using `sudo` for global changes (`adduser`, `reboot`, `hostnamectl`, `timedatectl`)
- Reading permission errors and understanding what they protect
- Changing a password with `passwd`
- Verifying a reboot with `uptime`
- Reviewing login history with `last` and `lastb`
- Comparing `sudo -i` and `sudo -s` and recognizing root by the `#` prompt
- Renaming a server with `hostnamectl` and understanding `preserve_hostname` in cloud-init
- Checking and changing the timezone with `timedatectl`
- Redacting sensitive details (public IP, password hashes) before documenting

---

## 🔗 References

- [Linux Upskill Challenge – Day 3: Power trip](https://linuxupskillchallenge.org/03)

---

*Day 3 is the first day where I could actually break the server, and the lesson is that Linux makes you ask for that power one command at a time.*

