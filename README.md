# Lab: Searching Files with `find`

**Series:** linux-ops-mastery — RHCSA Essential Tools & File Operations
**Subjects covered:** `find` for live filesystem traversal, `-name`/`-iname` glob matching, `-type f/d/l`, `-size +/-`, `-mtime/-atime/-ctime ±N`, `-user`/`-group`/`-perm`, the action predicates `-print`, `-print0`, `-delete`, `-exec ... {} \;`, `-exec ... {} +`, the safe `-exec ... \;` vs efficient `+` form, logical combinators `-and` / `-or` / `-not` / `! ... `, pruning with `-prune`, performance vs `locate`, why `find` is exam-day's swiss-army knife
**Career arcs covered:** RHCSA (every "find files matching X" exam task), RHCE (Ansible `find:` module), SRE (incident triage: "what was modified in the last hour?"), DevOps (artifact cleanup, dependency audits), AI/MLOps (dataset discovery across distributed filesystems)
**Prerequisite:** Labs 05–13
**Time Estimate:** 35 to 50 minutes
**Difficulty arc:** Task 1 foundation (`-name`) · 2 type, size, time predicates · 3 user, group, perm predicates · 4 `-exec` actions · 5 logical operators and `-prune` · 6 RHCSA exam-realistic capstone

---

## Objective

Make `find` your daily reflex for "I need every file matching X." By the end of this lab you can compose multi-predicate searches with `-and`/`-or`/`-not`, run actions per match safely with both `-exec {} \;` and the more efficient `-exec {} +`, and silence the noise that `find /` produces on a real system.

The capstone is an exam-realistic prompt: *"In `/etc/`, find every file owned by `root`, ending in `.conf`, modified within the last 90 days, larger than 100 bytes, and save the full list (suppressing permission errors) to `/root/conf-list.txt` with each path null-terminated for safe scripting."*

> **Lab safety note:** All experimentation is read-only on system paths; the capstone writes only to `/root/`. The `-delete` predicate is demonstrated only inside `/tmp/find-lab`.

---

## Concept: `find` Is a Live Traversal — Not a Pre-Built Index

`find` walks the filesystem **right now**. It reads every directory it crawls, checks each entry against your predicates, and emits matches. There is no cache, no index, no "last updated" — the answer reflects the kernel's view at this moment.

```
   $ find /etc -name '*.conf' -type f -mtime -7
       │   │    │              │       │
       │   │    │              │       └─ Predicate: modified within last 7 days
       │   │    │              └─ Predicate: only regular files (not directories)
       │   │    └─ Predicate: filename matches *.conf
       │   └─ Starting directory (recurse from here)
       └─ The command

What `find` does at each entry:
   1. stat the entry → determine type, size, owner, times.
   2. Apply every predicate in order. ANY false predicate short-circuits.
   3. If all true, run the action(s) — default action is -print.
```

> **Why this matters:** `find` is the only standard tool that combines "where does this file live?" with "filter by type, size, age, owner, permission, name, content." Mastering it makes you fast on every shell.

---

## 📜 Why `find` Exists — The Story

`find` was in **Unix v1 (1971)** because the question "where is that file?" predates Unix itself. Early systems had no index — programs walked the directory tree directly. As filesystems grew, `find` collected more predicates: `-name`, `-type`, `-size`, `-mtime`, `-user`, `-perm`, then action predicates like `-print` (1980s), `-exec` (1980s), `-print0` (1990s, GNU for safe filename handling), and `-delete` (1990s GNU).

The big idea was always the same: **a predicate is a filter; an action is what to do with the matches.** This composability is why `find` has never been replaced, only complemented (by `locate`, `mlocate`, `fd-find`, etc.).

> **The point of the story:** `find` is older than the C programming language and still in active use. The reason: its predicate-and-action model is irreplaceable. Tools like `locate` are faster but stale; `find` is slow but live and complete.

---

## 👪 The find Family — Who Lives There

### Predicates by category

| Category | Predicates |
|---|---|
| **Name** | `-name`, `-iname`, `-path`, `-ipath`, `-regex`, `-iregex` |
| **Type** | `-type f` (file) / `d` (dir) / `l` (symlink) / `b/c/p/s` |
| **Size** | `-size +100c` (bytes) / `+1k` / `+1M` / `+1G` |
| **Time** | `-mtime ±N`, `-atime ±N`, `-ctime ±N`, `-newer FILE`, `-mmin ±N` (minutes) |
| **Owner** | `-user NAME/UID`, `-group NAME/GID`, `-nouser`, `-nogroup` |
| **Permission** | `-perm MODE` (exact), `-perm -MODE` (all bits set), `-perm /MODE` (any bit set) |
| **Inode / link** | `-inum N`, `-samefile FILE`, `-links N` |
| **Depth / pruning** | `-mindepth N`, `-maxdepth N`, `-prune` |

### Actions

| Action | Notes |
|---|---|
| `-print` | Default; print path + newline |
| `-print0` | Null-terminated path — pair with `xargs -0` |
| `-delete` | Unlink (use `-depth` to remove dirs after their contents) |
| `-exec CMD {} \;` | Run CMD once per match, replacing `{}` with the path |
| `-exec CMD {} +` | Batch matches into one CMD call (much faster) |
| `-ls` | Long-listing-style output |
| `-quit` | Stop after first match |

### Logical combinators

| Operator | Meaning |
|---|---|
| `-and` (or implicit) | Both predicates true |
| `-or` | Either predicate true |
| `-not` / `!` | Negate next predicate |
| `( ... )` | Group (escape parens in shell: `\(` `\)`) |

> **The point of the family tree:** Predicates filter; actions act; combinators glue them together. Once you can name each part of the syntax, complex finds stop looking complex.

---

## 🔬 The Anatomy of `find` — In One Diagram

```
$ find /etc \( -name '*.conf' -o -name '*.cfg' \) -type f -mtime -90 -size +100c -exec ls -lh {} +
       │      │  └─────────┘     └─────────┘   │    │       │         │           │           │
       │      │       │                │       │    │       │         │           │           └─ Action: run `ls -lh` ONCE with up to ARG_MAX matches
       │      │       │                │       │    │       │         │           └─ `{}` is replaced by matched paths
       │      │       │                │       │    │       │         └─ Modified within last 90 days
       │      │       │                │       │    │       └─ File size > 100 bytes
       │      │       │                │       │    └─ Regular files only
       │      │       │                │       └─ -or
       │      │       │                └─ second name pattern
       │      │       └─ first name pattern
       │      └─ grouped (-name '*.conf' OR -name '*.cfg')
       └─ starting directory (or list of starting directories)
```

> **Reading rule:** Always read left-to-right. Each predicate filters the set. Each action runs on whatever survives. `-exec {} +` is the modern default; `-exec {} \;` is the one-at-a-time form (slow on large matches).

---

## 📚 find Reference Table

| Task | Command | Notes |
|---|---|---|
| Filename glob | `find / -name '*.conf' 2>/dev/null` | Case-sensitive |
| Filename glob, ignore case | `find / -iname '*.CONF' 2>/dev/null` | |
| Regex on whole path | `find / -regex '.*/sshd_config' 2>/dev/null` | Default emacs regex; GNU has `-regextype posix-extended` |
| Only files | `find PATH -type f` | |
| Only directories | `find PATH -type d` | |
| Only symlinks | `find PATH -type l` | |
| Size > 100 MiB | `find PATH -size +100M` | M = MiB |
| Size between 1k and 1M | `find PATH -size +1k -size -1M` | Combine predicates |
| Modified within last 7 days | `find PATH -mtime -7` | `+N` = older than N; `-N` = newer than N |
| Modified more than 30 days ago | `find PATH -mtime +30` | |
| Modified within last 60 minutes | `find PATH -mmin -60` | |
| Owner | `find PATH -user alice` | |
| Group | `find PATH -group wheel` | |
| Exact permission | `find PATH -perm 0644` | |
| All bits set | `find PATH -perm -u+w` | |
| Any bit set | `find PATH -perm /a+x` | |
| Files with no owner / orphans | `find PATH -nouser -o -nogroup` | |
| Pipe-safe output | `find PATH ... -print0 \| xargs -0 cmd` | Handles spaces / newlines in names |
| Run command per match | `find PATH ... -exec cmd {} \;` | One invocation per match (slow) |
| Run command in batches | `find PATH ... -exec cmd {} +` | Batches up to ARG_MAX matches per invocation |
| Delete matches | `find PATH ... -delete` | Use `-depth` to delete dirs after contents |
| Stop at filesystem boundary | `find PATH -xdev ...` | Do not cross mounts |
| Limit recursion depth | `find PATH -maxdepth 3 ...` | |
| Skip a subtree | `find PATH -type d -name node_modules -prune -o -print` | |
| Silence permission errors | `find / ... 2>/dev/null` | |

> **Rule one of find:** Predicates are AND'd implicitly. Use `-o` for OR; use `\( ... \)` (with escaped parens) to group.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | "Find every file under `/etc` larger than 1 MiB" — exam-canonical. |
| **RHCE candidate** | Ansible's `find:` module — same predicates expressed declaratively. |
| **SRE / Platform** | "What was modified in the last 10 minutes during the incident?" → `find / -mmin -10 -type f 2>/dev/null`. |
| **DevOps** | Cleaning up build caches: `find $CACHE -atime +30 -delete`. |
| **AI / MLOps** | "Which checkpoints are still useful?" → `find /scratch -name '*.pt' -size +10M -mtime -30`. |

---

## 🔧 The 6 Tasks

> Six exam-realistic phases that build the **predicate → action → safe output** habit.

---

### Task 1 — Sandbox setup and basic `-name` searches

**Purpose:** Build a controlled tree and find files by name pattern.

```bash
mkdir -p /tmp/find-lab && cd /tmp/find-lab
mkdir -p logs/{2026,2025}/{01..03} docs/{old,new}
touch logs/2026/01/{a,b,c}.log logs/2025/01/legacy.log
touch docs/old/manual.txt docs/new/readme.md docs/new/INDEX.TXT

find /tmp/find-lab -name '*.log'
find /tmp/find-lab -name 'README*'
find /tmp/find-lab -iname 'index.txt'
find /tmp/find-lab -name '*.md' -o -name '*.txt'
```

**Human-Readable Breakdown:** Build a small tree. Search by exact glob, case-insensitive glob, and OR of two globs.

**Reading it left to right:** `-name 'PATTERN'` uses shell glob (`*`, `?`, `[abc]`). `-iname` ignores case. `-name X -o -name Y` ORs two name predicates. Quote globs so the shell does not expand them.

**The story:** `-name` is the predicate you reach for 80% of the time. Quote your pattern, always — otherwise the shell expands `*` before `find` ever sees it.

**Expected output:**

```text
/tmp/find-lab/logs/2026/01/a.log
/tmp/find-lab/logs/2026/01/b.log
/tmp/find-lab/logs/2026/01/c.log
/tmp/find-lab/logs/2025/01/legacy.log
/tmp/find-lab/docs/new/readme.md
/tmp/find-lab/docs/new/INDEX.TXT
/tmp/find-lab/docs/old/manual.txt
/tmp/find-lab/docs/new/readme.md
/tmp/find-lab/docs/new/INDEX.TXT
```

**Switches**

| Token | Meaning |
|---|---|
| `-name PAT` | Glob match (case-sensitive) |
| `-iname PAT` | Glob match (case-insensitive) |
| `-o` | Logical OR |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Pattern expanded by shell | Quote it: `'*.log'` |
| No matches but should be some | Check `pwd`; check spelling; try `-iname` |
| `find` printed errors | Add `2>/dev/null` to silence permission noise |

---

### Task 2 — Type, size, and time predicates

**Purpose:** Narrow searches by file type (`-type`), size (`-size`), and mtime (`-mtime`/`-mmin`).

```bash
cd /tmp/find-lab

dd if=/dev/zero of=big.bin bs=1M count=2 2>/dev/null
dd if=/dev/zero of=small.bin bs=1k count=1 2>/dev/null
touch -d "40 days ago" old.log

find /tmp/find-lab -type d
find /tmp/find-lab -type f -size +1M
find /tmp/find-lab -type f -size -500c
find /tmp/find-lab -type f -mtime +30
find /tmp/find-lab -type f -mmin -5
find /tmp/find-lab -type f -newer big.bin
```

**Human-Readable Breakdown:** Create a 2 MiB file, a 1 KiB file, and an old log. Then run six searches: directories only, large files, tiny files, old files, files modified in the last 5 minutes, files newer than the big binary.

**Reading it left to right:** `-type` filters by file type. `-size` uses suffix `c` (bytes), `k` (KiB), `M` (MiB), `G` (GiB); `+N` means greater, `-N` means less. `-mtime ±N` is days; `-mmin ±N` is minutes. `-newer FILE` compares mtime against another file.

**The story:** These three predicates compose to answer 90% of "find every X" prompts. Combine them — they AND together.

**Expected output:**

```text
/tmp/find-lab
/tmp/find-lab/logs
/tmp/find-lab/logs/2026
/tmp/find-lab/logs/2026/01
/tmp/find-lab/logs/2026/02
...
/tmp/find-lab/big.bin
/tmp/find-lab/small.bin
... (every empty file)
/tmp/find-lab/old.log
... (recent files including big.bin, small.bin)
... (files newer than big.bin — likely none if you just created it)
```

**Switches**

| Token | Meaning |
|---|---|
| `-type f/d/l/b/c/p/s` | File-type filter |
| `-size ±N[cbkMG]` | Size filter |
| `-mtime ±N` | Days since modification |
| `-mmin ±N` | Minutes since modification |
| `-newer REF` | Newer than REF's mtime |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `-size +1G` returns nothing | None that big — try smaller |
| `+30` and `-30` both empty | The boundary is exclusive; check `stat` directly |
| `-mmin` returns wrong items | Mounts may use `relatime`/`noatime` — affects atime, not mtime |

---

### Task 3 — Owner, group, and permission predicates

**Purpose:** Filter by ownership and permission bits — useful for security audits.

```bash
sudo -i
cd /tmp/find-lab
touch root-owned.txt
chown root:root root-owned.txt

find /tmp/find-lab -user root
find /tmp/find-lab -group root
find /tmp/find-lab ! -user root
find /tmp/find-lab -perm 0644
find /tmp/find-lab -perm -u+w           # all of u+w set
find /tmp/find-lab -perm /a+x           # any of u/g/o execute
find / -nouser -o -nogroup 2>/dev/null | head -n 5
```

**Human-Readable Breakdown:** Create a root-owned file in the sandbox. Search for root-owned, root-group, NOT-root-owned, exact mode, "all of these bits set," "any of these bits set," and orphan files (no user or no group — common after deleting a user).

**Reading it left to right:** `-user`/`-group` accept names or numeric IDs. `-perm 0644` matches exact mode. `-perm -MODE` requires all bits in MODE to be set. `-perm /MODE` requires any bits in MODE to be set. `-nouser` finds files whose UID is not in `/etc/passwd`.

**The story:** Security audits live on `-perm` and `-user`. "Find every world-writable file" is `-perm -o+w`. "Find every SUID binary" is `-perm -u+s`.

**Expected output:**

```text
/tmp/find-lab/root-owned.txt
/tmp/find-lab/root-owned.txt
/tmp/find-lab/big.bin
/tmp/find-lab/small.bin
/tmp/find-lab/old.log
...
/tmp/find-lab
/tmp/find-lab/big.bin
/tmp/find-lab/small.bin
/tmp/find-lab/root-owned.txt
...
```

**Switches**

| Token | Meaning |
|---|---|
| `-user NAME` | Match owner |
| `-group NAME` | Match group |
| `! PRED` | Negate |
| `-perm MODE` | Exact mode match |
| `-perm -MODE` | All MODE bits set |
| `-perm /MODE` | Any MODE bit set |
| `-nouser` | UID not in passwd |
| `-nogroup` | GID not in group file |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `-user unknown` errors | Username does not exist; use numeric UID |
| `-perm 644` did not match | Mode is octal — include leading 0 (`0644`) |
| `-nouser` is slow on large filesystems | Normal — it consults passwd for every match |

---

### Task 4 — Actions: `-print0`, `-exec ... \;`, `-exec ... +`, `-delete`

**Purpose:** Run actions on matches safely: `-print0` for pipelines, `-exec` (two forms), and `-delete`.

```bash
cd /tmp/find-lab

# Default action — print
find /tmp/find-lab -type f

# -print0 with xargs -0 for safe handling
find /tmp/find-lab -type f -print0 | xargs -0 ls -l | head -n 5

# -exec one-per-match
find /tmp/find-lab -type f -name '*.log' -exec wc -l {} \;

# -exec batched (much faster)
find /tmp/find-lab -type f -name '*.log' -exec wc -l {} +

# -delete (sandbox only)
mkdir -p /tmp/find-lab/scratch
touch /tmp/find-lab/scratch/{a,b,c}.tmp
find /tmp/find-lab/scratch -type f -name '*.tmp' -print -delete
ls /tmp/find-lab/scratch/
```

**Human-Readable Breakdown:** Try every common action: default `-print`, `-print0` paired with `xargs -0`, the slow per-match `-exec ... \;`, the fast batched `-exec ... +`, and `-delete` for in-place cleanup.

**Reading it left to right:** `-print0` emits null bytes between paths, which `xargs -0` reads safely (handles spaces, newlines, quotes). `-exec ... \;` runs the command once per match — slow. `-exec ... +` batches matches into one command invocation — fast. `-delete` unlinks each match (combine with `-depth` for inside-out deletion of directory trees).

**The story:** Beginners use `-exec ... \;` until they discover `+`. The speed difference on large match sets is dramatic (one process startup vs thousands).

**Expected output:**

```text
/tmp/find-lab/big.bin
/tmp/find-lab/small.bin
/tmp/find-lab/old.log
...
-rw-r--r--. 1 user user 2097152 ... big.bin
-rw-r--r--. 1 user user    1024 ... small.bin
-rw-r--r--. 1 user user       0 ... old.log
0 /tmp/find-lab/logs/2026/01/a.log
0 /tmp/find-lab/logs/2026/01/b.log
...
0 /tmp/find-lab/logs/2026/01/a.log
0 /tmp/find-lab/logs/2026/01/b.log
0 /tmp/find-lab/logs/2025/01/legacy.log
0 total
/tmp/find-lab/scratch/a.tmp
/tmp/find-lab/scratch/b.tmp
/tmp/find-lab/scratch/c.tmp
```

**Switches**

| Token | Meaning |
|---|---|
| `-print0` | Null-terminated output |
| `xargs -0` | Read null-terminated input |
| `-exec CMD {} \;` | Per-match invocation |
| `-exec CMD {} +` | Batched invocation (preferred) |
| `-delete` | Unlink matches |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `\;` printed literally | Quote it: `'\;'` or escape in some shells |
| `+` syntax error | The `+` must come at the end; nothing after `{}` |
| `-delete` removed parent dirs unexpectedly | Add `-depth` to delete contents first |
| Filenames with spaces broke | Use `-print0` + `xargs -0` |

---

### Task 5 — Logical operators, parens, `-prune`

**Purpose:** Combine predicates with `-and` (implicit), `-or`, `-not`/`!`, group with escaped `\( \)`, and skip subtrees with `-prune`.

```bash
cd /tmp/find-lab

# Implicit AND
find /tmp/find-lab -type f -name '*.log'

# OR
find /tmp/find-lab -name '*.log' -o -name '*.txt'

# Combined: ( *.log OR *.txt ) AND -type f
find /tmp/find-lab \( -name '*.log' -o -name '*.txt' \) -type f

# NOT
find /tmp/find-lab -type f ! -name '*.log'

# Prune — skip the docs/old subtree
find /tmp/find-lab -type d -name old -prune -o -type f -print
```

**Human-Readable Breakdown:** Learn AND/OR/NOT, group with escaped parens, and use `-prune` to skip subtrees you do not want to descend into.

**Reading it left to right:** Predicates separated by space are AND'd. `-o` means OR — but it binds looser than AND, so use `\( ... \)` to group. `!` or `-not` negates the next predicate. `-prune` tells `find` "do not descend into this directory" — useful for skipping `.git`, `node_modules`, `/proc`, `/sys`.

**The story:** `-prune` is the underrated predicate. It is how you do `find /` without descending into `/proc` (which would return millions of irrelevant entries).

**Expected output:**

```text
/tmp/find-lab/old.log
/tmp/find-lab/logs/2026/01/a.log
...
/tmp/find-lab/old.log
/tmp/find-lab/docs/old/manual.txt
...
/tmp/find-lab/old.log
/tmp/find-lab/docs/old/manual.txt
...
/tmp/find-lab/big.bin
/tmp/find-lab/small.bin
/tmp/find-lab/docs/new/readme.md
/tmp/find-lab/docs/new/INDEX.TXT
...
... (docs/old/ skipped)
```

**Switches**

| Token | Meaning |
|---|---|
| `-and` (or space) | Logical AND |
| `-o` | Logical OR |
| `!` / `-not` | Logical NOT |
| `\( \)` | Group (escape parens in shell) |
| `-prune` | Skip subtree |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `find: paths must precede expression` | The shell expanded an unquoted `*` — quote it |
| `-o` results unexpected | Operator precedence — group with `\( ... \)` |
| `-prune` did not stop descent | `-prune` only works with `-o -print` pattern |

---

### Task 6 — Capstone: RHCSA-realistic multi-predicate search

**Task statement:** *"In `/etc/`, find every file owned by `root`, ending in `.conf`, modified within the last 90 days, larger than 100 bytes. Suppress permission errors. Save the resulting paths to `/root/conf-list.txt`, one per line, and also save a null-terminated copy to `/root/conf-list.txt0` for safe scripting."*

**Purpose:** Execute a five-predicate `find` end-to-end and produce both human-readable and null-terminated output.

```bash
sudo -i

find /etc \
  -type f \
  -name '*.conf' \
  -user root \
  -mtime -90 \
  -size +100c \
  -print 2>/dev/null \
  | tee /root/conf-list.txt \
  | wc -l

find /etc \
  -type f \
  -name '*.conf' \
  -user root \
  -mtime -90 \
  -size +100c \
  -print0 2>/dev/null \
  > /root/conf-list.txt0

# Verify
wc -l /root/conf-list.txt
ls -l /root/conf-list.txt0
test -s /root/conf-list.txt && echo "VERIFY: text list exists and is non-empty"
test -s /root/conf-list.txt0 && echo "VERIFY: null-terminated list exists"
head -n 5 /root/conf-list.txt
echo "--- counted from null-list ---"
tr '\0' '\n' < /root/conf-list.txt0 | grep -c .
```

**Human-Readable Breakdown:** Become root. Run the multi-predicate `find` twice: once with `-print` piped through `tee` and `wc -l` to make `conf-list.txt` and a count, once with `-print0` redirected to `conf-list.txt0` for the null-terminated copy. Verify both files exist and the line counts agree.

**Layer stack you built:**

```text
/etc
   │  find -type f -name '*.conf' -user root -mtime -90 -size +100c
   │      (all predicates ANDed; permission errors silenced)
   ▼
matches
   ├── /root/conf-list.txt   (newline-separated, human readable, tee'd from -print)
   └── /root/conf-list.txt0  (null-terminated, scripting-safe, from -print0)
```

**The story:** This is the **canonical 90-second exam answer** for multi-predicate finds. Memorize the spine: `find /path -type f -name PAT -user USER -mtime -N -size +Nc 2>/dev/null | tee /out`. Adjust predicates per question.

**Expected verification output:**

```text
185
185 /root/conf-list.txt
-rw-r--r--. 1 root root 7402 May 26 ... /root/conf-list.txt0
VERIFY: text list exists and is non-empty
VERIFY: null-terminated list exists
/etc/dnf/dnf.conf
/etc/ssh/sshd_config.d/50-redhat.conf
/etc/yum.conf
/etc/sysctl.conf
/etc/sysctl.d/99-sysctl.conf
--- counted from null-list ---
185
```

**Cleanup**

```bash
rm -rf /tmp/find-lab
rm -f /root/conf-list.txt /root/conf-list.txt0
exit
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Counts disagree | A path contains a newline — that is why `-print0` exists |
| Empty result | One predicate was too strict — drop predicates one at a time |
| Permission errors on screen | Add `2>/dev/null` |
| Wrote to wrong path | Use absolute `/root/...` |

---

## 🔍 find Decision Guide

```
Got a "find every file matching X" need?
  │
  ├── "By name"
  │       └── ✅ find PATH -name 'GLOB'   (or -iname)
  │
  ├── "By type (file vs dir vs symlink)"
  │       └── ✅ find PATH -type f / d / l
  │
  ├── "By size"
  │       └── ✅ find PATH -size +100M    (etc.)
  │
  ├── "By age"
  │       └── ✅ find PATH -mtime -7      (within last 7 days)
  │       └── ✅ find PATH -mtime +30     (older than 30 days)
  │
  ├── "By owner / group"
  │       └── ✅ find PATH -user NAME -group NAME
  │
  ├── "By permission"
  │       └── ✅ find PATH -perm 0644          (exact)
  │       └── ✅ find PATH -perm -u+w          (all of these set)
  │       └── ✅ find PATH -perm /a+x          (any of these set)
  │
  ├── "Multiple predicates"
  │       └── ✅ Just chain — they AND implicitly.
  │       └── ✅ Use \( -name '*.log' -o -name '*.txt' \) for OR groups.
  │
  ├── "Skip a subtree"
  │       └── ✅ -type d -name SKIPDIR -prune -o -print
  │
  ├── "Do something with each match"
  │       └── ✅ -exec cmd {} +                (batched — preferred)
  │       └── ✅ -exec cmd {} \;               (per-match — when batching breaks)
  │       └── ✅ -delete                       (unlink each)
  │
  ├── "Safe pipeline output"
  │       └── ✅ -print0 | xargs -0 cmd
  │
  └── "Silence permission noise"
          └── ✅ ... 2>/dev/null
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Set up `/tmp/find-lab` and run `-name` / `-iname` / `-o` name searches
- [ ] 02 Filter by `-type`, `-size ±[ckMG]`, and `-mtime ±N`
- [ ] 03 Filter by `-user`, `-group`, and `-perm` flags
- [ ] 04 Use `-print0`, `-exec ... \;`, `-exec ... +`, and `-delete`
- [ ] 05 Combine with `-and` / `-o` / `!` / `\( \)` / `-prune`
- [ ] 06 Execute the RHCSA capstone — multi-predicate `find` saving to `/root/conf-list.txt` + null-terminated copy

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Unquoted glob expanded by shell | `find: paths must precede expression` | Quote it `'*.log'` |
| Mixed `-and`/`-o` precedence | Wrong results | Group with `\( ... \)` |
| `-exec ... \;` is slow | One process per match | Use `-exec ... +` |
| Filenames with newlines/spaces | Garbled pipeline | Use `-print0 \| xargs -0` |
| Forgot `2>/dev/null` | Noisy permission errors | Append `2>/dev/null` |
| `-delete` removed too much | No preview | First run with `-print` only |
| `-prune` had no effect | Wrong predicate order | `... -prune -o -print` pattern |
| `-size 100c` matched nothing | No `+/-` means exactly | Use `+/-` for ranges |
| `-mtime 0` confusing | `0` = within last day; `+0` = older than today | Use explicit `-N`/`+N` |
| Crossed mount points unexpectedly | Hits NFS, /proc, etc. | Add `-xdev` |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize the four most useful predicate trios: `-type f -name '*.X' -user USER`, `-type f -size +N`, `-type f -mtime -N`, `-type f -perm /a+x`.

**RHCE candidate**
- Ansible `find:` module accepts `paths`, `patterns`, `age`, `size`, `file_type`, `recurse` — same predicates declarative.

**SRE / Platform interview**
- "Walk me through finding everything modified during a 14:00–14:30 window." → `touch -d "...14:00" /tmp/s; touch -d "...14:30" /tmp/e; find / -newer /tmp/s ! -newer /tmp/e 2>/dev/null`.

**DevOps**
- `find $CACHE -atime +30 -delete` for cache eviction.

**AI / MLOps**
- `find /scratch -name '*.pt' -size +10M -mtime -30 -printf '%T@ %p\n' | sort -nr | head` for "biggest recent checkpoints."

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 07 — Timestamps with `touch` | The fixtures `find -mtime` consumes |
| Lab 11 — Safe Deletion | `find -delete` is the bulk-delete primitive |
| Lab 15 — Instant File Searching with `locate` | The cache-backed sibling of `find` |
| Lab 22 — Filtering with `grep` and Regex | Pair: `find ... -exec grep ... {} +` |
| Lab 25 — Extracting Columns with `awk` | After `find` lists paths, `awk` extracts fields |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
