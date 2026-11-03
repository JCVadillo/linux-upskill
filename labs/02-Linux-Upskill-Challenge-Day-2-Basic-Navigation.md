# 🐧 02 – Linux Upskill Challenge – Day 2: Basic Navigation
*Linux Upskill Challenge – Day 2*
*(Find command documentation, work with files and directories, and practise directory-stack navigation)*

---

## 🎯 Lab Summary

Day 2 was about getting more comfortable in the terminal: looking up commands, reading directory listings, and creating and removing files and folders. I also tried `pushd` and `popd` to move between directories without retyping every path.

What I did not expect was a problem with `tldr`. A quick command lookup turned into a small troubleshooting exercise when its cache update failed. That gave me a practical reason to use `file` and `unzip`, rather than just reading about them.

The main tasks were to:
- Read manual pages and compare them with short `tldr` examples
- Explore the Linux filesystem hierarchy
- Compare `ls` options and reveal hidden files
- Create and remove a test directory and file
- Practise directory-stack navigation with `pushd` and `popd`
- Investigate the missing `tldr` cache and work through a manual-cache workaround

---

## ⚙️ Environment

| Component | Detail |
|---|---|
| **Server** | `ubuntu-linux-server` |
| **OS** | Ubuntu Server |
| **Client** | macOS Terminal, connected over SSH |
| **User** | `jc.vadillo87` |
| **Main working directory** | `/home/jc.vadillo87` (`~`) |
| **Other directories visited** | `/var/log`, `/dev`, `/etc` |
| **tldr client** | Tealdeer, identified by its configuration and cache paths |

---

## 📖 Finding Help with `man` and `tldr`

### Reading manual pages

I started by looking at the manuals for `pwd`, `cp` and `mv`:

```bash
man pwd
man cp
man mv
```

The pages follow a consistent structure:

| Section | What I use it for |
|---|---|
| **NAME** | A short explanation of the command |
| **SYNOPSIS** | The accepted command syntax |
| **DESCRIPTION** | Behaviour and available options |

The `pwd` manual explains how to print the current working directory. It also introduces logical and physical paths: a physical path resolves symbolic links instead of preserving them in the displayed path.

The other two manuals helped clarify a basic distinction: `cp` copies files or directories, while `mv` moves or renames them. In this activity, I focused on reviewing their documentation rather than practising copy and move operations.

### Comparing a manual with a quick reference

Once command pages were available, I checked:

```bash
tldr pwd
```

The difference was easy to see. The manual gives a detailed reference; `tldr` gets straight to a few examples. For a quick reminder, the shorter page is easier to scan. For the details of an option, I would go back to the manual.

| Tool | Useful when |
|---|---|
| `man` | I need a fuller description of behaviour and options |
| `tldr` | I want a short explanation and common examples |

---

## 🔧 Troubleshooting — `tldr` Could Not Update Its Cache

### Initial error

My first attempt was:

```bash
tldr pwd
```

It returned:

```text
Error: Page cache not found. Please run `tldr --update` to download the cache.
```

The cache is the local collection of command pages. Without those files, this client could not display the requested documentation.

I followed the suggestion:

```bash
tldr --update
```

This failed too, but with a more specific error:

```text
Error: Could not update cache

Caused by:
    0: Could not decompress downloaded ZIP archive
    1: invalid Zip archive: Could not find central directory end
```

The updater could not read the ZIP archive it received. An incomplete download or a response that was not really a ZIP could explain that error, but neither cause was established in this session.

### Checking the manual download

I downloaded `tldr.zip` separately with `wget`, then checked what kind of file I had:

```bash
file tldr.zip
```

Output:

```text
tldr.zip: Zip archive data, at least v1.0 to extract, compression method=store
```

That identified it as a ZIP archive. It did **not**, on its own, prove that the whole archive was intact. To test that, I ran:

```bash
unzip -t tldr.zip | tail -3
```

Output:

```text
    testing: LICENSE.md               OK
    testing: index.json               OK
No errors detected in compressed data of tldr.zip.
```

| Command | What it checked |
|---|---|
| `file tldr.zip` | Recognised the downloaded file type |
| `unzip -t tldr.zip` | Tested the archive's entries without extracting them |
| `tail -3` | Displayed only the last three lines of the test output |

This confirmed that the manually downloaded archive passed its integrity test. It did not tell me why the automatic download had failed.

### Finding the application's directories

Next, I checked where tealdeer expected its files:

```bash
tldr --show-paths
```

The output was:

```text
Config dir:       /home/jc.vadillo87/.config/tealdeer/ (OS convention)
Config path:      /home/jc.vadillo87/.config/tealdeer/config.toml
Cache dir:        /home/jc.vadillo87/.cache/tealdeer (OS convention)
Pages dir:        /home/jc.vadillo87/.cache/tealdeer/tldr-pages/
Custom pages dir: /home/jc.vadillo87/.local/share/tealdeer/pages/ (OS convention)
```

| Path | Purpose |
|---|---|
| `~/.config/tealdeer/config.toml` | Client configuration |
| `~/.cache/tealdeer` | Cache directory |
| `~/.cache/tealdeer/tldr-pages/` | Downloaded command pages |
| `~/.local/share/tealdeer/pages/` | Custom command pages |

### Manual-cache workaround

With a tested archive and the pages location identified, the workaround provided during troubleshooting was:

```bash
mkdir -p ~/.cache/tealdeer/tldr-pages
unzip tldr.zip -d ~/.cache/tealdeer/tldr-pages
```

Here, `~` refers to my home directory. The `-p` option creates missing parent directories and avoids an error if the directory already exists. The `-d` option tells `unzip` where to extract the archive.

> 💡 Later, `tldr pwd` and `tldr pushd` displayed their command pages successfully. The manual-cache procedure above was the proposed workaround; its extraction output was not retained. The working lookups confirmed that those pages were available afterward, not that the automatic updater had been repaired.

### Outcome and remaining uncertainty

| Item | Result |
|---|---|
| Initial page lookup | Failed because the page cache was missing |
| Automatic cache update | Failed while reading the downloaded ZIP |
| Manual archive identification | Recognised as ZIP data |
| Manual archive integrity test | No errors detected |
| Later page lookups | `tldr pwd` and `tldr pushd` displayed documentation |
| Root cause of automatic-update failure | Not established |
| Subsequent successful `tldr --update` | Not verified |

The useful lesson was to separate the questions: what did I download, is it intact, where does the application expect its data, and does the lookup work afterward? Getting the pages working was useful, but it was not the same as fixing or explaining the automatic updater.

---

## 📂 Reading the Filesystem Hierarchy

I opened the filesystem hierarchy manual:

```bash
man hier
```

I reviewed the start of the directory reference, including `/`, `/bin`, `/boot`, `/dev` and `/etc`.

| Directory | Purpose described in the manual |
|---|---|
| `/` | Root of the filesystem tree |
| `/bin` | Essential executable programs |
| `/boot` | Files needed during the boot process |
| `/dev` | Device files |
| `/etc` | System configuration files |

The main idea is that Linux has one directory tree starting at `/`, rather than a separate tree for each drive letter. This also gave some context to the directories I visited later: `/etc` was no longer just another path to type.

---

## 📋 Listing Files and Hidden Entries

I compared several directory listings in my home directory:

```bash
ls
ls -l
ls -l -t
ls -l -t -r
ls -l -t -r -a
```

| Command / option | Effect |
|---|---|
| `ls` | Lists non-hidden entries |
| `-l` | Shows the long listing format |
| `-t` | Sorts by modification time, newest first |
| `-r` | Reverses the sorting order |
| `-a` | Includes hidden entries, plus `.` and `..` |

The options can also be combined:

```bash
ls -ltra
```

The plain listing showed only `tldr.zip`. With just one visible file, reversing the time order did not have much to demonstrate. Adding `-a` made the difference much clearer: entries such as `.profile`, `.bashrc`, `.ssh`, `.config` and `.cache` appeared.

The long listing also gives more than names:

| Field | What it tells me |
|---|---|
| First character | File type: `-` for a regular file, `d` for a directory |
| Permission characters | Read, write and execute permissions |
| Owner and group | Ownership of the entry |
| Size | File size; not a recursive directory-content total |
| Timestamp | Modification time |
| Name | File or directory name |

I can use the first character to distinguish files from directories without depending on terminal colours.

---

## 🗂️ Creating and Removing a Directory

I created a test directory and checked the listing:

```bash
mkdir test
ls
```

The output included `test` alongside `tldr.zip`.

Then I removed the empty directory and checked again:

```bash
rmdir test
ls
```

Only `tldr.zip` remained.

`rmdir` was enough here because the directory was empty. Its refusal to remove a non-empty directory is useful protection when that is the behaviour I want.

---

## 📄 Creating and Removing a File

Next, I created a test file:

```bash
touch test-file
ls
```

Because `test-file` did not already exist, `touch` created an empty file. The listing confirmed it was there.

I also opened it in the editor:

```bash
nano test-file
```

This step was simply to open the test file in an editor; the exercise here focuses on the file lifecycle rather than its contents.

Finally, I removed the file and checked the listing:

```bash
rm test-file
ls
```

The final output contained only `tldr.zip`.

| Command | Role in this exercise |
|---|---|
| `touch test-file` | Created the file |
| `nano test-file` | Opened the editor |
| `rm test-file` | Removed the file |
| `ls` | Checked the visible directory contents after changes |

One detail worth remembering: `touch` is not only a file-creation command. When a file already exists, it normally updates timestamps instead of clearing its contents.

---

## 🧭 Directory Navigation with `pushd` and `popd`

For the extension exercise, I first checked the short reference:

```bash
tldr pushd
```

Then I worked through this sequence:

```bash
pushd /var/log
pushd /dev
pushd /etc
pushd
popd
popd
```

The output made the directory stack much easier to understand:

| Command | Stack afterward | Current directory |
|---|---|---|
| `pushd /var/log` | `/var/log ~` | `/var/log` |
| `pushd /dev` | `/dev /var/log ~` | `/dev` |
| `pushd /etc` | `/etc /dev /var/log ~` | `/etc` |
| `pushd` | `/dev /etc /var/log ~` | `/dev` |
| `popd` | `/etc /var/log ~` | `/etc` |
| `popd` | `/var/log ~` | `/var/log` |

The current directory is the leftmost item. Each `pushd` with a path changed directory and put it at the front of the stack. With no argument, `pushd` swapped the top two entries. Each `popd` then removed the top entry and returned me to the next one.

As a related navigation shortcut, a bare `cd` returns to the home directory. The verified stack sequence above ends in `/var/log`.

I found this more useful than treating the stack as a list of places I had visited. Its order tells me where the next `popd` will take me.

---

## 📊 Results Summary

| Area | Result |
|---|---|
| Command documentation | Manual pages for `pwd`, `cp`, `mv` and `hier` |
| Quick references | Successful `tldr pwd` and `tldr pushd` lookups |
| Directory listings | `ls` with long-format, time-sort, reverse and hidden-entry options |
| Directory operations | Created and removed the empty `test` directory |
| File operations | Created, opened in an editor, and removed `test-file` |
| Navigation extension | Built, swapped and popped directory-stack entries |
| Troubleshooting | Identified and tested the manual ZIP; located tealdeer's cache paths |
| Remaining issue | Automatic-update failure was not explained or retested successfully |

---

## 🗂️ Scope of This Activity

The Day 2 guide introduces more commands than I covered in this session. This write-up focuses on the work described above, with the following limits:

| Topic | Status in this write-up |
|---|---|
| Copying and moving files | Documentation reviewed; practical copy and move exercises are outside this write-up |
| Relative-path navigation | Not covered in this session |
| `cd -` | Mentioned in the guide's extension work; not demonstrated here |
| Recursive deletion | Not demonstrated |
| Additional help tools such as `apropos` and `info` | Referenced by the guide, not demonstrated here |
| Editing file contents | No specific saved changes documented |
| Automatic cache-update repair | Not verified |

---

## 🧠 Reflections / Notes

- **Looking things up is part of the work.** I do not need to memorise every switch, but I do need to know where to find an explanation and how to read it.
- **An almost-empty listing can be misleading.** Plain `ls` showed one file. Adding `-a` revealed the configuration and history files that had been there all along.
- **Recognising a ZIP is not the same as testing it.** `file` identified the archive; `unzip -t` checked its contents. Those were separate checks, and both helped narrow down the problem.
- **The directory stack has an order, not just a history.** Watching it change after each command made `pushd` and `popd` much less abstract.
- **A workaround is not a root-cause fix.** The later `tldr` lookups worked, but I still cannot say why its automatic download failed.

---

## 📚 Key Skills Demonstrated

- Reading command manual pages and short command references
- Understanding the basic Linux directory hierarchy
- Interpreting long directory listings and hidden entries
- Creating and removing test files and empty directories
- Navigating with `pushd` and `popd`
- Identifying downloaded file types and testing ZIP integrity
- Locating application configuration and cache directories
- Separating verified results from suggested steps and unresolved causes

---

## 🔗 References

- Linux Upskill Challenge — https://linuxupskillchallenge.org/
- Supplied activity guide: `2 - Basic navigation.pdf`
- Local manual pages: `man pwd`, `man cp`, `man mv`, `man hier`
- Tealdeer diagnostic output and archive-test results recorded during troubleshooting

---

*I came into Day 2 to practise moving around the filesystem. The unexpected part was learning to slow down when a command failed: read the error, check the file, and keep track of what I had actually proved.*

