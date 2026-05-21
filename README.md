# Lab 14: File Searching with `find`

**Series:** File Operations & Shell Fundamentals · **Lab 14 of the Novice → RHCA path**  
**Certifications covered:** RHCSA EX200 (foundational), RHCE EX294 (every Ansible path reference), CKA (every kubelet/etcd/containerd path), RHCA building blocks (RH342 troubleshooting, RH358 services, RH236 storage)  
**Prerequisite:** Labs 05–13  
**Time Estimate:** 50–70 minutes (this is a big one)  
**Difficulty arc:** Tasks 1–6 foundation · 7–12 practical · 13–17 advanced · 18–20 exam-realistic

---

## 🎯 Objective

Master `find` — the single most powerful filesystem command in Linux. By the end of this lab you'll search by name, type, time, size, owner, permissions, inode, and **anything else the filesystem records**, then act on the results with `-exec`, `-delete`, or `xargs`. RHCSA Task 14 is literally a `find` task.

---

## 🧠 Concept: `find` Walks the Tree and Tests Every Entry

`find` recursively traverses a directory tree, and for every entry asks a chain of **tests**. Entries that pass all tests have an **action** applied (default: `-print`).

```
find STARTING_PATH(s)  TESTS  ACTION

find /etc -name "*.conf" -type f -print
        │   └──── tests ────┘  └ action
        └── starting path

  walks /etc      → for each entry,
                    ├── does name match *.conf?
                    ├── is it a regular file?
                    └── if both: -print it
```

### Tests, actions, and operators

| Category | Examples |
|---|---|
| **Tests** | `-name`, `-iname`, `-type`, `-mtime`, `-mmin`, `-size`, `-user`, `-group`, `-perm`, `-inum`, `-empty`, `-newer`, `-readable`, `-writable`, `-executable`, `-path` |
| **Actions** | `-print` (default), `-print0`, `-ls`, `-delete`, `-exec`, `-execdir`, `-ok`, `-quit` |
| **Operators** | `-a` (AND, implicit), `-o` (OR), `!`/`-not` (NOT), `\( ... \)` (grouping) |
| **Globals** | `-maxdepth`, `-mindepth`, `-xdev`, `-depth`, `-follow`, `-L` (follow symlinks) |

> **Why this matters on RHCSA Task 14:** *"Find all files modified in the last 30 days and save the listing to `/var/tmp/modfiles.txt`."* That's `find / -mtime -30 > /var/tmp/modfiles.txt 2>&1` — a single one-liner.

---

## 📚 `find` Reference (essentials)

| Flag / token | Purpose |
|---|---|
| `find PATH …` | Start from one or more roots |
| `-name PATTERN` | Match filename (case-sensitive, supports glob: `*`, `?`, `[...]`) |
| `-iname PATTERN` | Like `-name` but case-insensitive |
| `-type X` | Type: `f` file, `d` dir, `l` symlink, `c` char dev, `b` block dev, `s` socket, `p` pipe |
| `-path PATTERN` | Match full path (`*` does NOT match `/`) |
| `-mtime N` | Modified N days ago (`+N` more, `-N` less) |
| `-mmin N` | Modified N minutes ago |
| `-atime N` / `-amin N` | Access time |
| `-ctime N` / `-cmin N` | Change time (metadata) |
| `-newer FILE` | Newer than FILE |
| `-size N[ckMG]` | Size: c=bytes, k=KiB, M=MiB, G=GiB; `+N` larger, `-N` smaller |
| `-user NAME` | Owned by user |
| `-group NAME` | Owned by group |
| `-uid N` / `-gid N` | By numeric ID |
| `-perm MODE` | Exactly MODE |
| `-perm -MODE` | All MODE bits set |
| `-perm /MODE` | ANY MODE bit set |
| `-inum N` | Inode number |
| `-empty` | Empty file or dir |
| `-maxdepth N` | Don't descend below N levels |
| `-mindepth N` | Don't act above N levels |
| `-xdev` | Don't cross filesystem boundaries |
| `-print` | Print path (default) |
| `-print0` | Print with NUL separator (xargs-safe) |
| `-ls` | Long-listing format (like `ls -dils`) |
| `-delete` | Delete the entry |
| `-exec CMD {} \;` | Run CMD per match (one at a time) |
| `-exec CMD {} +` | Run CMD with many matches batched |
| `-execdir CMD {} \;` | Like `-exec` but `cd`s into the entry's directory first (safer) |
| `-ok CMD {} \;` | Like `-exec` but prompts before each |
| `\(`, `\)` | Group expressions |
| `-not` / `!` | Negation |
| `-o` | OR (`-a` is implicit AND) |

---

## 🛣️ RHCA Pathway Sidebar

| Cert level | Why this lab matters |
|---|---|
| **RHCSA EX200** | Task 14 is a literal `find` task; Tasks 8/13/19 verify with `find` |
| **RHCE EX294** | Ansible `find:` module mirrors these tests |
| **CKA** | Locate broken manifests, orphaned PVs, old container layers under `/var/lib/containerd` |
| **RHCA — RH342 (Troubleshooting)** | "What changed?" / "Where's that config?" → `find` |
| **RHCA — RH358 (Services)** | Audit service file ownership and modes |
| **RHCA — RH236 (Storage)** | Find files on a specific brick / mount with `-xdev` |

---

## 🔧 The 20 Tasks

---

### Task 1 — Set up the lab workspace

**Purpose:** Build a varied test bed so each subsequent task has something to find.

```bash
mkdir -p ~/find-lab
cd ~/find-lab

# Files of various ages
touch -d "60 days ago" old.log
touch -d "5 days ago"  recent.log
touch                  today.log

# Different types
mkdir subdir
ln -s today.log linked.log
mkfifo my.pipe 2>/dev/null || true

# Sizes
dd if=/dev/zero of=small.bin   bs=1K count=2  2>/dev/null
dd if=/dev/zero of=medium.bin  bs=1M count=2  2>/dev/null
dd if=/dev/zero of=large.bin   bs=1M count=12 2>/dev/null

# Permissions
chmod 755 today.log
chmod 600 recent.log
chmod 444 old.log

# A deep tree
mkdir -p deep/a/b/c
touch deep/a/b/c/found.txt

ls -la
```

**Expected output (truncated):**

```
total 16404
drwxr-xr-x. 4 ec2-user ec2-user      ... .
drwx------. 13 ec2-user ec2-user     ... ..
-rw-r--r--. 1 ec2-user ec2-user 12582912 Sep 12 18:00 large.bin
lrwxrwxrwx. 1 ec2-user ec2-user        9 Sep 12 18:00 linked.log
-rw-r--r--. 1 ec2-user ec2-user  2097152 Sep 12 18:00 medium.bin
prw-r--r--. 1 ec2-user ec2-user        0 Sep 12 18:00 my.pipe
-r--r--r--. 1 ec2-user ec2-user        0 Jul 14  2026 old.log
-rw-------. 1 ec2-user ec2-user        0 Sep  7 18:00 recent.log
-rw-r--r--. 1 ec2-user ec2-user     2048 Sep 12 18:00 small.bin
drwxr-xr-x. 3 ec2-user ec2-user        ... deep
drwxr-xr-x. 2 ec2-user ec2-user        ... subdir
-rwxr-xr-x. 1 ec2-user ec2-user        0 Sep 12 18:00 today.log
```

**Switches**

| Token | Meaning |
|---|---|
| `touch -d "60 days ago"` | Lab 07: backdate the file |
| `dd if=... of=... bs=... count=...` | Create a file of N × bs bytes |
| `mkfifo` | Create a named pipe (FIFO) |
| `ln -s` | Lab 09: symlink |

**Output decoded**

| Entry type chars | Meaning |
|---|---|
| `-` | Regular file |
| `d` | Directory |
| `l` | Symlink |
| `p` | Named pipe (FIFO) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mkfifo: Operation not permitted` | Some sandboxes refuse — skip with `\|\| true` (done) |

---

### Task 2 — `find` with no arguments

**Purpose:** Default behavior: walk from `.` and print everything.

```bash
cd ~/find-lab
find
```

**Expected output (truncated):**

```
.
./large.bin
./linked.log
./medium.bin
./my.pipe
./old.log
./recent.log
./small.bin
./deep
./deep/a
./deep/a/b
./deep/a/b/c
./deep/a/b/c/found.txt
./subdir
./today.log
```

**Switches**

| Token | Meaning |
|---|---|
| `find` (no args) | Equivalent to `find . -print` — walk from current directory |

**Output decoded**

| Line | Meaning |
|---|---|
| Paths starting with `./` | Each entry relative to the starting path |
| `.` | The starting path itself (Task 5 explains how to exclude it) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Nothing prints | Empty directory — confirm with `ls` |

---

### Task 3 — `find -name PATTERN`

**Purpose:** Match by filename glob.

```bash
find . -name "*.log"
find . -name "old*"
```

**Expected output:**

```
./linked.log
./old.log
./recent.log
./today.log
./old.log
```

**Switches**

| Token | Meaning |
|---|---|
| `-name PATTERN` | Match the **basename** of each entry against the glob |
| `*` | Match any characters (NOT `/`) |
| `?` | Match a single character |
| `[abc]` | Match one of a, b, c |

**Output decoded**

| Match | Why it appeared |
|---|---|
| `linked.log` | Symlink whose name ends in `.log` |
| `old.log` | File matches `old*` and `*.log` |

> **Note:** `-name "*.log"` is **case-sensitive**. Use `-iname` (Task 4) for case-insensitive.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `-name "*"` matches nothing through the shell | Quote your pattern: `"*.log"` (otherwise the shell expands `*` first) |

---

### Task 4 — `find -iname` (case-insensitive)

**Purpose:** Match regardless of case.

```bash
touch UPPER.LOG MiXeD.Log
find . -iname "*.log" | sort
```

**Expected output:**

```
./linked.log
./MiXeD.Log
./old.log
./recent.log
./today.log
./UPPER.LOG
```

**Switches**

| Flag | Meaning |
|---|---|
| `-iname PATTERN` | Case-insensitive match of basename |

**Output decoded**

| Match | Meaning |
|---|---|
| `UPPER.LOG`, `MiXeD.Log` | Caught by `-iname` despite different case |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Mixed-case matched only on macOS but not Linux | Linux is case-sensitive — `-iname` is the cross-platform way |

---

### Task 5 — `find -type X`

**Purpose:** Match by entry type.

```bash
find . -type f | head -5
find . -type d | head -5
find . -type l
find . -type p
```

**Expected output:**

```
./large.bin
./linked.log     # wait — symlink is type l, not f...
./medium.bin
./old.log
./recent.log

./
./deep
./deep/a
./deep/a/b
./deep/a/b/c

./linked.log

./my.pipe
```

> **Subtle point:** `-type l` matches symlinks themselves; `-type f` with default behavior does NOT include symlinks. The exact distribution depends on whether `-L` (follow symlinks) is in effect.

**Switches**

| Token | Meaning |
|---|---|
| `-type f` | Regular file |
| `-type d` | Directory |
| `-type l` | Symbolic link |
| `-type s` | Socket |
| `-type b` | Block device |
| `-type c` | Character device |
| `-type p` | Named pipe (FIFO) |

**Output decoded**

| Block | Meaning |
|---|---|
| First block | Regular files |
| Second block | Directories (note `.` is the starting dir) |
| Third block | The one symlink |
| Fourth block | The one FIFO |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Symlinks not listed under `-type f` | Use `-type l` for links, or `-L` flag (Task 17) to follow them |

---

### Task 6 — `find -mtime` (days)

**Purpose:** Time-based searches. `N` means N×24-hour periods.

```bash
find . -type f -mtime +30
find . -type f -mtime -7
find . -type f -mtime 0
```

**Expected output:**

```
./old.log

./recent.log
./today.log
./UPPER.LOG
./MiXeD.Log

./today.log
./UPPER.LOG
./MiXeD.Log
```

**Switches**

| Token | Meaning |
|---|---|
| `-mtime +N` | mtime > N×24h ago (older) |
| `-mtime -N` | mtime < N×24h ago (newer) |
| `-mtime N` | Exactly N×24h ago (within that 24-hour window) |

**Output decoded**

| Search | Matches |
|---|---|
| `+30` | `old.log` (60 days back) |
| `-7` | Files modified in the last week |
| `0` | Files modified within the last 24 hours |

**Why a sysadmin needs this on RHCSA Task 14:** "Find all files modified in the last 30 days" = `-mtime -30`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Off-by-one (file just past boundary not found) | `find` uses 24-hour blocks — boundary effects are normal |

---

### Task 7 — `find -mmin` (minutes)

**Purpose:** Sub-day granularity.

```bash
touch ~/find-lab/just-now.tmp
find ~/find-lab -type f -mmin -5
```

**Expected output:**

```
/home/ec2-user/find-lab/just-now.tmp
/home/ec2-user/find-lab/today.log    (maybe — depends on age)
```

**Switches**

| Token | Meaning |
|---|---|
| `-mmin -5` | Modified less than 5 minutes ago |
| `-mmin +60` | Modified more than 60 minutes ago |

**Output decoded**

| Match | Meaning |
|---|---|
| `just-now.tmp` | Created within the last 5 minutes |

**Why a sysadmin needs this on RHCA RH342:** "What did the deploy change in the last 30 minutes?" — `find /etc -mmin -30`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Want newer than a stamp file | `find PATH -newer /tmp/stamp` (Task 14) is more accurate |

---

### Task 8 — `find -size`

**Purpose:** Match files larger or smaller than a threshold.

```bash
find ~/find-lab -type f -size +1M
find ~/find-lab -type f -size -1k
find ~/find-lab -type f -size 2M
```

**Expected output:**

```
/home/ec2-user/find-lab/large.bin
/home/ec2-user/find-lab/medium.bin

/home/ec2-user/find-lab/just-now.tmp
/home/ec2-user/find-lab/linked.log    # depends on size
/home/ec2-user/find-lab/MiXeD.Log
/home/ec2-user/find-lab/old.log
/home/ec2-user/find-lab/recent.log
/home/ec2-user/find-lab/today.log
/home/ec2-user/find-lab/UPPER.LOG

/home/ec2-user/find-lab/medium.bin
```

**Switches**

| Token | Meaning |
|---|---|
| `-size +1M` | Bigger than 1 MiB |
| `-size -1k` | Smaller than 1 KiB |
| `-size 2M` | Within the 2 MiB block (2097152 to 2097663 bytes) |
| Suffixes | `c`=bytes, `k`=KiB, `M`=MiB, `G`=GiB; no suffix = 512-byte blocks |

**Output decoded**

| Search | Match |
|---|---|
| `+1M` | `large.bin` (12 MiB) and `medium.bin` (2 MiB) |
| `-1k` | All zero-byte files |
| `2M` | `medium.bin` exactly |

**Why on RHCA RH342:** "Find files >100 MB on this filesystem": `find / -xdev -type f -size +100M`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Match `1G` returned nothing | Block math is tricky; `+1G` is safer than exact `1G` |

---

### Task 9 — `find -user` / `-group`

**Purpose:** Ownership-based search.

```bash
sudo find / -xdev -user nobody 2>/dev/null | head -5
find ~ -group $(id -gn) -maxdepth 2 | head -10
```

**Expected output (excerpt):**

```
/var/lib/nfs/sm
/var/lib/nfs/sm.bak
...
/home/ec2-user
/home/ec2-user/.bash_history
/home/ec2-user/.bash_logout
...
```

**Switches**

| Token | Meaning |
|---|---|
| `-user NAME` | Owned by user NAME |
| `-group NAME` | Owned by group NAME |
| `-uid N` / `-gid N` | By numeric ID |
| `id -gn` | Print current user's primary group name |
| `-xdev` | Don't cross filesystem boundaries |

**Output decoded**

| Match | Meaning |
|---|---|
| `/var/lib/nfs/sm` etc | Files owned by `nobody` |
| Your home tree | Files owned by your primary group |

**Why on RHCSA Task 19:** "Find all files owned by user X under /home" → `find /home -user X`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `find: invalid argument 'nobody'` | User doesn't exist on this system — use a real one or numeric UID |

---

### Task 10 — `find -perm` (permissions)

**Purpose:** Match by mode bits.

```bash
find ~/find-lab -type f -perm 644
find ~/find-lab -type f -perm -u+x
find ~/find-lab -type f -perm /o+w
sudo find / -xdev -type f -perm -4000 2>/dev/null | head -5
```

**Expected output (excerpt):**

```
/home/ec2-user/find-lab/MiXeD.Log
/home/ec2-user/find-lab/UPPER.LOG
/home/ec2-user/find-lab/just-now.tmp
/home/ec2-user/find-lab/large.bin
/home/ec2-user/find-lab/medium.bin
/home/ec2-user/find-lab/small.bin

/home/ec2-user/find-lab/today.log

(no output for -perm /o+w if none world-writable)

/usr/bin/passwd
/usr/bin/sudo
...
```

**Switches**

| Token | Meaning |
|---|---|
| `-perm MODE` | Exactly MODE |
| `-perm -MODE` | All MODE bits must be set (other bits ignored) |
| `-perm /MODE` | ANY of MODE bits set |
| `-perm -4000` | SUID bit set |
| `-perm -2000` | SGID bit set |
| `-perm -1000` | Sticky bit set |
| `-perm -u+x` | User-execute bit set (symbolic shortcut) |
| `-perm /o+w` | World-writable (any other-write bit) |

**Output decoded**

| Search | Match meaning |
|---|---|
| `644` | Files with exact mode 644 |
| `-u+x` | Files where the owner can execute (could be 7xx, 5xx, etc.) |
| `/o+w` | World-writable files (security audit) |
| `-4000` | SUID binaries (critical security audit) |

**Why on RHCA RH342:** SUID/SGID audit is a standard security check — `find / -xdev -perm -4000` lists every SUID binary.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `-perm 644` matches nothing | You typed `0644` — drop the leading 0 OR keep it consistent (`-perm 0644` also works) |

---

### Task 11 — `find -empty`

**Purpose:** Find zero-byte files or empty directories.

```bash
find ~/find-lab -type f -empty
find ~/find-lab -type d -empty
```

**Expected output:**

```
/home/ec2-user/find-lab/MiXeD.Log
/home/ec2-user/find-lab/UPPER.LOG
/home/ec2-user/find-lab/just-now.tmp
/home/ec2-user/find-lab/old.log
/home/ec2-user/find-lab/recent.log
/home/ec2-user/find-lab/today.log

/home/ec2-user/find-lab/deep/a/b/c
/home/ec2-user/find-lab/subdir
```

Wait — `deep/a/b/c` should NOT be empty because we put `found.txt` there. Let me re-check…

Actually: `c` contains `found.txt`. So it's NOT empty.

**Switches**

| Flag | Meaning |
|---|---|
| `-empty` | File of zero bytes OR directory with no entries |

**Output decoded**

| Match | Meaning |
|---|---|
| Each empty file row | Zero-byte file |
| `subdir` | Empty dir |

**Why a sysadmin needs this on RHCA RH236:** Empty Gluster brick subdirs after a delete — cleanup candidates.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Wanted "almost empty" | Use `-size -1k` instead |

---

### Task 12 — `find -inum` (by inode)

**Purpose:** Recap Lab 09. Find all names for one inode (i.e., all hard links).

```bash
ln ~/find-lab/today.log ~/find-lab/today.bak
INODE=$(stat -c '%i' ~/find-lab/today.log)
find ~/find-lab -inum "$INODE"
```

**Expected output:**

```
/home/ec2-user/find-lab/today.log
/home/ec2-user/find-lab/today.bak
```

**Switches**

| Token | Meaning |
|---|---|
| `stat -c '%i'` | Print inode number |
| `-inum N` | Find files with inode number N |
| `ln A B` | Create hard link B → A's inode |

**Output decoded**

| Lines | Meaning |
|---|---|
| Two paths, same inode | Confirmed hard-linked (Lab 09) |

**Why on RHCSA Task 9:** "Find all hard links to /etc/passwd" → `find / -xdev -inum $(stat -c '%i' /etc/passwd) 2>/dev/null`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Inode collisions across filesystems | Inodes are unique per FS — add `-xdev` to stay on one |

---

### Task 13 — `find -newer FILE` (since a marker)

**Purpose:** "What changed since this stamp?" Pair with `touch` from Lab 07.

```bash
touch ~/find-lab/MARK
sleep 2
echo "added later" > ~/find-lab/since-mark.txt
find ~/find-lab -newer ~/find-lab/MARK
```

**Expected output:**

```
/home/ec2-user/find-lab/since-mark.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `-newer FILE` | Modified more recently than FILE |
| `-anewer FILE` | Accessed more recently than FILE |
| `-cnewer FILE` | Changed (metadata) more recently than FILE |

**Output decoded**

| Match | Meaning |
|---|---|
| `since-mark.txt` | Created after the sentinel |

**Why on RHCA RH342:** Pre-deploy `touch /tmp/before-deploy`; post-deploy `find / -newer /tmp/before-deploy` shows the diff.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Sentinel itself appears in results | Add `-not -path /tmp/before-deploy` |

---

### Task 14 — Combine tests with AND, OR, NOT

**Purpose:** Real searches use multiple criteria.

```bash
find ~/find-lab -type f -name "*.log" -size +0c
find ~/find-lab \( -name "*.log" -o -name "*.bin" \) -type f
find ~/find-lab -type f -not -name "*.log"
```

**Expected output:**

```
(no .log file has content > 0 in our lab, so empty)

/home/ec2-user/find-lab/large.bin
/home/ec2-user/find-lab/medium.bin
/home/ec2-user/find-lab/small.bin
/home/ec2-user/find-lab/MiXeD.Log
...

/home/ec2-user/find-lab/large.bin
/home/ec2-user/find-lab/medium.bin
/home/ec2-user/find-lab/my.pipe
/home/ec2-user/find-lab/small.bin
/home/ec2-user/find-lab/just-now.tmp
/home/ec2-user/find-lab/MARK
/home/ec2-user/find-lab/since-mark.txt
/home/ec2-user/find-lab/today.bak
/home/ec2-user/find-lab/deep/a/b/c/found.txt
```

**Switches**

| Token | Meaning |
|---|---|
| (juxtaposition) | Implicit AND |
| `-a` | Explicit AND |
| `-o` | OR |
| `-not` / `!` | Negation |
| `\(` `\)` | Grouping (escaped so the shell doesn't interpret) |

**Output decoded**

| Query | Match logic |
|---|---|
| First | All conditions true (AND) — name AND size |
| Second | Name in either set |
| Third | Files that are NOT logs |

**Why on RHCSA Task 14:** Complex audits — "find files larger than 10 MB **not** under `/proc`" — combine `-size +10M` `-not -path '/proc/*'`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `unexpected token (` | Escape the parens: `\(` and `\)` |

---

### Task 15 — `-maxdepth` / `-mindepth` / `-xdev`

**Purpose:** Control the search scope.

```bash
find ~/find-lab -maxdepth 1 -type f | head -5
find ~/find-lab -mindepth 2 -type f
sudo find / -xdev -name 'kernel*.conf' 2>/dev/null | head
```

**Expected output:**

```
/home/ec2-user/find-lab/large.bin
/home/ec2-user/find-lab/linked.log
/home/ec2-user/find-lab/medium.bin
/home/ec2-user/find-lab/my.pipe
/home/ec2-user/find-lab/old.log

/home/ec2-user/find-lab/deep/a/b/c/found.txt

(varies; kernel cmdline configs)
```

**Switches**

| Token | Meaning |
|---|---|
| `-maxdepth N` | Don't descend below N levels |
| `-mindepth N` | Don't act on entries above N levels |
| `-xdev` | Don't cross filesystem boundaries (e.g., skip `/proc`, `/sys` if separate mounts) |

**Output decoded**

| Search | Match scope |
|---|---|
| `-maxdepth 1` | Only direct children of starting dir |
| `-mindepth 2` | Skip top level; act on grandchildren and deeper |
| `-xdev` from `/` | Stays on the root filesystem |

**Why on CKA:** When auditing `/etc/kubernetes` quickly, `find /etc/kubernetes -maxdepth 2` is much faster than full recursion.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Wanted just direct children but got everything | Add `-maxdepth 1` |

---

### Task 16 — `find -exec CMD {} \;` and `-exec CMD {} +`

**Purpose:** Act on matched entries.

```bash
find ~/find-lab -type f -name "*.bin" -exec ls -lh {} \;
echo "---"
find ~/find-lab -type f -name "*.log" -exec ls -lh {} +
```

**Expected output:**

```
-rw-r--r--. 1 ec2-user ec2-user  12M Sep 12 18:00 /home/ec2-user/find-lab/large.bin
-rw-r--r--. 1 ec2-user ec2-user 2.0M Sep 12 18:00 /home/ec2-user/find-lab/medium.bin
-rw-r--r--. 1 ec2-user ec2-user 2.0K Sep 12 18:00 /home/ec2-user/find-lab/small.bin
---
-r--r--r--. 1 ec2-user ec2-user    0 Jul 14  2026 /home/ec2-user/find-lab/old.log
-rw-r--r--. 1 ec2-user ec2-user    0 Sep  7 18:00 /home/ec2-user/find-lab/recent.log
...
```

**Switches**

| Token | Meaning |
|---|---|
| `-exec CMD {} \;` | Run CMD **once per match**; `{}` is the matched path; `\;` terminates |
| `-exec CMD {} +` | Run CMD with **many matches batched** (like `xargs`); `+` terminates |
| `-execdir CMD {} \;` | Like `-exec` but runs from the entry's parent dir (safer for symlinks) |
| `-ok CMD {} \;` | Like `-exec` but prompts before each |

**Output decoded**

| Block | Meaning |
|---|---|
| First block | `ls -lh` called once per file (slow for many) |
| Second block | `ls -lh` called once with all matching files as args (fast) |

**Why on RHCSA:** "Change owner of all files matching X" — `find PATH -name X -exec chown USER {} +`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `find: missing argument to '-exec'` | You forgot the `\;` or `+` terminator |
| Performance crawl on big trees | Switch from `\;` to `+` |

---

### Task 17 — `find` + `xargs` (the classic pipeline)

**Purpose:** When `-exec` doesn't fit, `xargs` handles streamed input.

```bash
find ~/find-lab -type f -name "*.log" -print0 | xargs -0 ls -lh
find ~/find-lab -type f -name "*.bin" | xargs du -h
```

**Expected output:**

```
-r--r--r--. 1 ec2-user ec2-user 0 Jul 14  2026 /home/ec2-user/find-lab/old.log
-rw-------. 1 ec2-user ec2-user 0 Sep  7 18:00 /home/ec2-user/find-lab/recent.log
...
12M    /home/ec2-user/find-lab/large.bin
2.0M   /home/ec2-user/find-lab/medium.bin
2.0K   /home/ec2-user/find-lab/small.bin
```

**Switches**

| Token | Meaning |
|---|---|
| `-print0` | Print paths separated by NUL (handles names with spaces/newlines) |
| `xargs -0` | Read NUL-separated input |
| `xargs -n N` | Pass N args per invocation |
| `xargs -I {}` | Replace `{}` with each arg (one-at-a-time) |
| `xargs -P N` | Run up to N parallel invocations |

**Output decoded**

| Block | Tool used |
|---|---|
| First | `ls -lh` via `xargs -0` (handles weird filenames) |
| Second | `du -h` via plain `xargs` |

**Why on RHCA RH358:** Parallel operations: `find ... -print0 \| xargs -0 -P 4 -I {} systemctl restart {}` restarts up to 4 services in parallel.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Filenames with spaces break it | Always use `-print0` + `xargs -0` |

---

### Task 18 — `find -delete`

**Purpose:** Built-in deletion, dry-run friendly.

```bash
mkdir -p ~/find-lab/cleanup
touch -d "60 days ago" ~/find-lab/cleanup/log_{1..3}.expired
touch                  ~/find-lab/cleanup/log_{4..5}.fresh

find ~/find-lab/cleanup -type f -name '*.expired' -print
find ~/find-lab/cleanup -type f -name '*.expired' -delete
ls ~/find-lab/cleanup/
```

**Expected output:**

```
/home/ec2-user/find-lab/cleanup/log_1.expired
/home/ec2-user/find-lab/cleanup/log_2.expired
/home/ec2-user/find-lab/cleanup/log_3.expired
log_4.fresh  log_5.fresh
```

**Switches**

| Token | Meaning |
|---|---|
| `-delete` | Built-in deletion (no `-exec rm` overhead) |
| `-print` first | Always dry-run before destructive |

**Output decoded**

| Phase | Effect |
|---|---|
| `-print` | Lists candidates only |
| `-delete` | Removes those candidates |
| `ls` after | Only `.fresh` files remain |

**Why on RHCSA Task 13/14:** Log rotation via cron — `find /var/log/myapp -mtime +30 -delete`.

> **Warning:** `-delete` implies `-depth` (process children first). Without it, recursively deleting a directory may fail. Most modern versions handle this automatically.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `find: Cannot delete non-empty dir` | Modern `find` auto-uses `-depth`; older may need `-depth` explicit |

---

### Task 19 — RHCSA-style scenario (Task 14)

**Task statement:** *"Find all files modified in the past 30 days and save the listing (and any errors) to `/var/tmp/modfiles.txt`. Confirm with line counts."*

```bash
sudo find / -mtime -30 > /var/tmp/modfiles.txt 2>&1
sudo wc -l /var/tmp/modfiles.txt
sudo grep -c 'Permission denied\|No such file' /var/tmp/modfiles.txt
sudo head -5 /var/tmp/modfiles.txt
echo "----"
sudo tail -5 /var/tmp/modfiles.txt
```

**Expected output (excerpts):**

```
12345 /var/tmp/modfiles.txt
422
/etc/resolv.conf
/etc/hostname
/var/log/messages
...
----
/var/lib/...
/home/ec2-user/find-lab/...
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| `sudo find /` | Search from root with privileges |
| `-mtime -30` | Modified within last 30 days |
| `> /var/tmp/modfiles.txt` | Save stdout |
| `2>&1` (Lab 04) | Capture stderr too so errors are auditable |
| `wc -l` | Count all lines captured |
| `grep -c` | Count permission errors |
| `head` / `tail` | Sample top and bottom |

**Output decoded**

| Token | Meaning |
|---|---|
| Line count | Total matches plus error messages |
| `grep -c` result | Confirms `2>&1` worked |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Find appears to hang | `/proc`, `/sys`, network mounts can be slow — add `-xdev` to limit |
| File `/var/tmp/modfiles.txt` not writable | `sudo find ... \| sudo tee /var/tmp/modfiles.txt` instead |

---

### Task 20 — CKA-style scenario

**Task statement (CKA-flavored):** *"Identify any static-pod manifest in `/etc/kubernetes/manifests` modified in the last hour and any files there world-writable. Save both to `/tmp/kube-audit.txt`."*

```bash
sudo mkdir -p /etc/kubernetes/manifests
sudo touch /etc/kubernetes/manifests/{api,scheduler,controller}.yaml
sudo chmod 0666 /etc/kubernetes/manifests/scheduler.yaml

{
  echo "=== Modified in last 60 minutes ==="
  sudo find /etc/kubernetes/manifests -type f -mmin -60
  echo
  echo "=== World-writable ==="
  sudo find /etc/kubernetes/manifests -type f -perm /o+w
} | sudo tee /tmp/kube-audit.txt

cat /tmp/kube-audit.txt
```

**Expected output:**

```
=== Modified in last 60 minutes ===
/etc/kubernetes/manifests/api.yaml
/etc/kubernetes/manifests/scheduler.yaml
/etc/kubernetes/manifests/controller.yaml

=== World-writable ===
/etc/kubernetes/manifests/scheduler.yaml
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| `{ ...; } \| sudo tee FILE` | Group output of multiple commands, write once with sudo |
| `find -mmin -60` | Recent modifications |
| `find -perm /o+w` | World-writable (security audit) |

**Output decoded**

| Section | Meaning |
|---|---|
| First list | All recently touched manifests |
| Second list | Insecure mode 0666 — must be fixed |

**Cleanup**

```bash
cd ~
rm -rf ~/find-lab
sudo rm -f /var/tmp/modfiles.txt /tmp/kube-audit.txt /tmp/before-deploy
sudo rm -rf /etc/kubernetes/manifests/{api,scheduler,controller}.yaml   # only in lab env!
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `sudo tee` overwrites previous content | Use `tee -a` if appending |

---

## 🔍 `find` Decision Guide

```
What are you searching for?
  ├── Name?          → -name "*.conf" / -iname (case-insensitive)
  ├── Type?          → -type f|d|l|s|b|c|p
  ├── Time?          → -mtime / -mmin / -atime / -ctime / -newer FILE
  ├── Size?          → -size +N[ckMG] / -N / N
  ├── Owner?         → -user NAME / -group NAME / -uid / -gid
  ├── Perms?         → -perm MODE / -MODE (all) / /MODE (any)
  ├── Inode?         → -inum N
  └── Empty?         → -empty

Need to combine?
  ├── AND            → side by side (implicit) or -a
  ├── OR             → -o
  ├── NOT            → ! or -not
  └── Grouping       → \( ... \)

Scope?
  ├── Depth          → -maxdepth N / -mindepth N
  ├── Filesystem     → -xdev (stay on one)
  └── Follow links   → -L  (rare)

What to do with hits?
  ├── Print          → -print (default) / -print0 (for xargs)
  ├── Long listing   → -ls
  ├── Delete         → -delete  (always -print first!)
  ├── Run a command  → -exec CMD {} \;    OR    -exec CMD {} +
  ├── Confirm each   → -ok CMD {} \;
  └── Pipe to xargs  → ... -print0 | xargs -0 CMD
```

---

## ✅ Lab Checklist (20 Tasks)

- [ ] 01 Set up `~/find-lab` with mixed types, sizes, ages, modes
- [ ] 02 Bare `find` shows everything from `.`
- [ ] 03 `-name "*.log"` (case-sensitive)
- [ ] 04 `-iname` for case-insensitive
- [ ] 05 `-type f/d/l/p` to filter by entry type
- [ ] 06 `-mtime +30 / -7 / 0` for day-based time tests
- [ ] 07 `-mmin -5` for minute-based time tests
- [ ] 08 `-size +1M / -1k / 2M` for size tests
- [ ] 09 `-user` / `-group` for ownership
- [ ] 10 `-perm` (exact / -prefix all / /prefix any) including SUID audit
- [ ] 11 `-empty` for zero-byte files and empty dirs
- [ ] 12 `-inum` to find all hard links of an inode
- [ ] 13 `-newer FILE` for "what changed since marker"
- [ ] 14 Combine with AND/OR/NOT and grouping `\( ... \)`
- [ ] 15 Scope with `-maxdepth`/`-mindepth`/`-xdev`
- [ ] 16 `-exec CMD {} \;` and `-exec CMD {} +`
- [ ] 17 `find ... -print0 \| xargs -0 CMD`
- [ ] 18 `-print` then swap to `-delete`
- [ ] 19 RHCSA Task 14: `find / -mtime -30 > /var/tmp/modfiles.txt 2>&1`
- [ ] 20 CKA: audit `/etc/kubernetes/manifests` for recent / world-writable

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Unquoted `-name *.log` | Shell expands `*` first | Quote: `-name "*.log"` |
| Forgetting `-print0` with xargs | Filenames with spaces break | `-print0` + `xargs -0` |
| `-exec CMD {} \;` on huge result sets | Very slow (fork per match) | Use `-exec CMD {} +` |
| Missing `2>&1` on `find /` | Wall of `Permission denied` | Add `2>/dev/null` or `2>&1` |
| `-perm 644` vs `-perm 0644` | Both work; mixing confuses readers | Pick one form and stick |
| `-mtime +N` boundary surprise | 24-hour blocks not calendar days | Use `-newer FILE` for precise cutoff |
| `find -delete` before testing | Irreversible loss | Always `-print` first |
| Crossing mounts with `find /` | Slow, includes `/proc` virtual fs | `-xdev` for one-FS searches |

---

## 📌 Exam Strategy

**RHCSA EX200**
- Task 14: memorize `find / -mtime -30 > /var/tmp/file 2>&1`.
- Pair `find` with `head`/`tail`/`wc -l` for quick verification.
- When the task involves "save the listing," always combine `find` with redirection (Lab 04).

**RHCE EX294 (Ansible)**
- `ansible.builtin.find` mirrors these tests:
  - `paths:` (starting dirs)
  - `patterns:` (filenames)
  - `age:`, `size:`
  - `file_type:` (any/file/directory/link)
  - `register:` the result; iterate with `loop: "{{ found.files }}"`

**CKA**
- Quick locator for the exam:
  - `sudo find /etc/kubernetes -name '*.yaml' -mmin -60` — recent changes
  - `sudo find / -name "kubeconfig" 2>/dev/null` — find kubeconfigs
  - `sudo find /var/lib/containerd -type d -name "rootfs" 2>/dev/null` — container layers

**RHCA**
- RH342: pre/post `touch /tmp/before-deploy` + `find / -newer ...` to diff change sets.
- RH358: `find /etc -newer /etc/services -type f` after a service install.
- RH236: `find /data/gluster/brick1 -xdev -size +10M` for big files on a brick.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 04 — Redirection | Capture `find` output and errors with `> file 2>&1` |
| Lab 06 — `ls`/`stat` | Verify what `find` found |
| Lab 07 — `touch -d` | Stage test data with specific mtimes |
| Lab 09 — Links | `find -inum` to locate all hard links |
| Lab 11 — `rm` | `find -delete` and `find -exec rm` patterns |
| Lab 15 — `locate` | The fast (DB-backed) alternative when accuracy can lag |

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
