# Linux Administration — From Zero to Senior

A single reference that **starts from first principles** and ends at senior interview depth.

**How to read this:**

- **New to a topic?** Read [Part 0](#part-0-start-here--the-mental-model) first, then the
  "🔰 **In plain English**" boxes. They explain every hard idea with an analogy, no jargon.
- **Revising for an interview?** Skip to the tables, the ⚠️ **Gotcha** boxes, and
  [Part 14](#part-14-interview-preparation).

Being able to explain these ideas *simply* is exactly what a senior interview tests. If you
can only recite commands, you get found out. If you can explain the mental model, you pass.

Commands are given for **RHEL/Rocky/Alma (dnf)** and **Debian/Ubuntu (apt)**. Where they
differ, both are shown. Assumes a modern kernel (5.x/6.x) with **systemd** and **cgroups v2**.

> Convention in this doc:
> `#` = run as root, `$` = unprivileged. Placeholders in `<angle brackets>`.

---

## Table of Contents

| Part | Topic |
|---|---|
| [0](#part-0-start-here--the-mental-model) | **Start Here — The Mental Model** |
| [1](#part-1-boot--init) | Boot & Init |
| [2](#part-2-processes--scheduling) | Processes & Scheduling |
| [3](#part-3-memory) | Memory |
| [4](#part-4-storage--filesystems) | Storage & Filesystems |
| [5](#part-5-networking) | Networking |
| [6](#part-6-users-auth--security) | Users, Auth & Security |
| [7](#part-7-packages--software) | Packages & Software |
| [8](#part-8-logging) | Logging |
| [9](#part-9-performance-analysis) | Performance Analysis |
| [10](#part-10-automation) | Automation |
| [11](#part-11-backup--ha) | Backup & HA |
| [12](#part-12-containers-from-first-principles) | Containers from First Principles |
| [13](#part-13-end-to-end-runbooks) | End-to-End Runbooks |
| [14](#part-14-interview-preparation) | Interview Preparation |

---

# PART 0: START HERE — THE MENTAL MODEL

Everything later in this document rests on six ideas. Get these and the rest is detail.

## 0.1 What "Linux" actually is

People say "Linux" to mean three different things. Interviewers notice when you blur them.

```text
┌──────────────────────────────────────────────────────┐
│  YOU, typing commands                                │
├──────────────────────────────────────────────────────┤
│  SHELL          bash / zsh                           │  ← reads what you type
├──────────────────────────────────────────────────────┤
│  USERSPACE      ls, systemd, nginx, python           │  ← ordinary programs
├──────────────────────────────────────────────────────┤
│  KERNEL         "Linux" proper                       │  ← the only thing that
│                 memory, CPU time, disks, network     │     touches hardware
├──────────────────────────────────────────────────────┤
│  HARDWARE       CPU, RAM, disks, NICs                │
└──────────────────────────────────────────────────────┘
```

🔰 **In plain English**
Think of a restaurant. The **kernel** is the kitchen — it is the only place with access to the
ingredients (hardware). You are the customer. The **shell** is the waiter who takes your
order. **Programs** are dishes on the menu. You never walk into the kitchen yourself; you ask
the waiter, who passes the order through a hatch.

That hatch is called a **system call** (syscall). When a program wants to read a file, it
cannot touch the disk. It asks the kernel: `read()`. The kernel does it and hands back the
result. This is why `strace` (which shows syscalls) is such a powerful debugging tool — it
shows you every request a program makes to the kitchen.

- **The kernel** is Linux. One project, one codebase.
- **A distribution** (RHEL, Ubuntu, Debian) = the kernel + thousands of userspace programs +
  a package manager, bundled and tested together.
- **The shell** is just another program. It is not Linux.

**Why you care:** when something breaks, your first question is always *"which layer?"*
A permissions error is kernel. A "command not found" is shell. A crash loop is userspace.

## 0.2 Everything is a file

This is the single most important design idea in Unix.

Your keyboard is a file. Your network card is reachable through file-like handles. A running
process's memory is exposed as files. A disk is a file. Even randomness is a file.

```bash
$ cat /proc/cpuinfo        # CPU details — not a real file on disk
$ cat /proc/meminfo        # memory stats
$ ls /proc/1234/           # everything about process 1234
$ cat /dev/urandom         # infinite random bytes
$ echo hi > /dev/pts/3     # write text to someone else's terminal
```

🔰 **In plain English**
Instead of inventing a different way to talk to every device, Unix said: *pretend it is all
files*. Then you only need to learn four verbs — `open`, `read`, `write`, `close` — and they
work on everything.

It is like a building where every room, lift, and cupboard uses the exact same key. You learn
one key instead of two hundred.

**Why you care:** this is why `/proc` and `/sys` exist. They are not real files on a disk —
they are the kernel *pretending* to be files so you can inspect and change it with normal
tools:

```bash
$ cat /proc/sys/vm/swappiness     # read a kernel setting
60
# echo 10 > /proc/sys/vm/swappiness   # change it, live
```

That is the whole trick behind `sysctl`.

## 0.3 A process is a running program

A **program** is a file on disk (dead). A **process** is that program loaded into memory and
running (alive). One program can become many processes — open three terminals, you have three
`bash` processes from one `/bin/bash` file.

Every process has:

| Property | What it means |
|---|---|
| **PID** | Its ID number |
| **PPID** | Its *parent's* ID — who started it |
| **UID** | Which user it runs as — this decides what it may touch |
| **State** | Running, sleeping, waiting for disk, dead |
| **fds** | Its open files (file descriptors) |

### Where processes come from

There is only one way to make a new process: an existing one **copies itself**.

```text
     bash (PID 2000)
        │
        │  fork()  ── makes a near-identical copy
        ▼
     bash (PID 2001)          ← the copy, "the child"
        │
        │  exec("/bin/ls")  ── child REPLACES itself with a different program
        ▼
     ls   (PID 2001)          ← same PID, totally different program
```

🔰 **In plain English**
`fork()` is **cloning yourself**. `exec()` is that clone **putting on a costume** and becoming
someone else. The PID never changes — the body is the same, the identity is replaced.

So when you type `ls`, bash clones itself and the clone turns into `ls`. When `ls` finishes,
bash notices and gives you your prompt back.

This explains a lot:

- **Why every process has a parent.** It was cloned by something.
- **Why PID 1 is special.** Nothing cloned it — the kernel made it directly at boot. On
  modern systems PID 1 is `systemd`. It is the ancestor of everything.
- **Why a "zombie" exists.** When a child dies, its exit code is kept until the parent asks
  for it. Parent forgets to ask → the corpse stays in the table. That is a zombie. (§4)

```bash
$ ps -ef --forest | head -20     # see the family tree
```

## 0.4 Users, root, and permissions

Linux decides "may you do this?" by comparing **your UID** with **the file's owner and mode**.

- Every user has a numeric **UID**. Your name is just a label in `/etc/passwd`.
- **UID 0 is root.** Root bypasses almost every permission check. That is the *only* thing
  special about it — the name `root` is irrelevant, the number `0` is what matters.

```text
  -rw-r-----   1  alice  devs   4096  Jan 10 09:14  secrets.txt
  │ │  │  │       │      │
  │ │  │  │       │      └── group that owns it
  │ │  │  │       └───────── user that owns it
  │ │  │  └──────────────── what EVERYONE ELSE may do   (---  = nothing)
  │ │  └─────────────────── what the GROUP may do       (r--  = read)
  │ └────────────────────── what the OWNER may do       (rw-  = read + write)
  └──────────────────────── type: - file, d directory, l link
```

🔰 **In plain English**
Three sets of rules, checked in order: *Are you the owner?* Use the owner's rules and stop.
*No — are you in the group?* Use the group rules and stop. *Neither?* You get the "other"
rules.

Note it **stops at the first match**. If you own a file and the owner bits say `---`, you
cannot read it, even if group says `rwx`. Being the owner is not automatically better.

Numbers are just the three bits added up: `r`=4, `w`=2, `x`=1.

```text
  7 = rwx      6 = rw-      5 = r-x      4 = r--      0 = ---

  chmod 640 f   →   rw- r-- ---   →   owner reads+writes, group reads, others nothing
  chmod 755 d   →   rwx r-x r-x   →   owner full, everyone else read+enter
```

⚠️ **On a directory, the bits mean something different:**

| Bit | On a file | On a directory |
|---|---|---|
| `r` | read contents | **list the names** inside |
| `w` | change contents | **create/delete files** inside |
| `x` | run it | **enter it** (`cd`, access things inside) |

This is why a directory with `r` but no `x` is useless — you can see the names but cannot
open anything. And why `w` on a directory lets you **delete a file you do not own**: deleting
is modifying the *directory*, not the file. (That is exactly what the sticky bit fixes — §25.)

## 0.5 The filesystem is one tree

Windows has `C:\`, `D:\`. Linux has **one** tree starting at `/`. Extra disks get *grafted on*
at a directory, called **mounting**.

```text
/                     ← root of everything (not the same as /root)
├── bin  sbin  lib    → programs and libraries (usually links into /usr)
├── boot              → kernel + bootloader. Often a SEPARATE small disk partition
├── dev               → devices as files (/dev/sda, /dev/null)
├── etc               → ALL system configuration. Text files. Back this up.
├── home              → users' personal dirs (/home/alice)
├── proc              → fake dir: live kernel + process info
├── sys               → fake dir: devices and kernel tunables
├── run               → runtime state since boot (PID files, sockets). Cleared on reboot.
├── tmp               → scratch space, world-writable, wiped periodically
├── usr               → installed software (read-mostly)
├── var               → data that CHANGES: logs, databases, caches, mail, spool
└── root              → the root user's home directory
```

🔰 **In plain English**
Mounting is like plugging a USB stick into a *folder* instead of getting a new drive letter.
You say "this disk now appears at `/data`", and from then on everything under `/data` is
physically on that disk. The folder it covers is called a **mount point**.

```bash
$ lsblk -f      # which disk is mounted where
$ findmnt       # the mount tree
$ df -hT        # how full each mounted filesystem is
```

**The two directories that cause most outages are `/var` and `/boot`** — `/var` because logs
grow forever and fill the disk (§17), `/boot` because it is small and old kernels pile up.

## 0.6 How the pieces fit at runtime

```text
   You type a command
        │
        ▼
   SHELL forks itself, the child execs your program
        │
        ▼
   PROCESS runs. Needs RAM → asks kernel. Needs a file → syscall.
        │
        ├── kernel checks PERMISSIONS (your UID vs the file)
        ├── kernel schedules it onto a CPU (it competes with everything else)
        ├── kernel gives it MEMORY (and may reclaim some later)
        └── kernel does the DISK/NETWORK I/O on its behalf
        │
        ▼
   Process exits → parent collects the exit code → PID freed
```

Almost every problem in this document is one of five things:

| Symptom | Which resource |
|---|---|
| Slow, high load | **CPU** — too many processes competing (§39) |
| Killed unexpectedly | **Memory** — ran out, OOM killer chose a victim (§10) |
| Slow, waiting | **Disk** — I/O is the bottleneck (§41) |
| Cannot connect | **Network** — link, route, DNS, firewall or port (§23) |
| "Permission denied" | **Permissions** — UID, mode, or SELinux (§25, §29) |

When you get a vague ticket that says "the server is slow", your job is to work out *which of
these five* it is, and then prove it. That is the whole game. Part 9 gives you the commands.

## 0.7 Ten commands to survive your first day

```bash
pwd                   # where am I
ls -lh                # what's here (long, human-readable sizes)
cd /var/log           # go somewhere
cat file              # print a small file
less file             # page through a big file  (q to quit, / to search)
tail -f app.log       # watch a log live  ← you will use this constantly
grep -r "error" /etc  # search inside files
ps aux                # what's running
df -h                 # is the disk full
systemctl status nginx   # is this service ok
```

> **When you are stuck:** `man <command>` for the manual, `<command> --help` for a quick
> reminder, and `apropos <keyword>` to find a command whose name you forgot.

---

# PART 1: BOOT & INIT

## 1. The Boot Sequence, End to End

🔰 **In plain English**
Booting is a **chain of handovers**, where each step's only job is to find and start the next,
slightly bigger step. Like waking up: alarm → you sit up → you stand → you walk → you leave
the house. Each stage can only do a little, so it loads something more capable and hands over.

The computer starts knowing *nothing* — it cannot even read your disk properly yet. So it
bootstraps itself in stages, each one teaching it a bit more about the hardware.

Know this cold. It is the single most common senior-level interview question because every
stage maps to a class of real outage.

```text
 1. Firmware (BIOS or UEFI)
      │  POST, initialise hardware
      │  BIOS  → reads MBR (first 512 bytes of disk)
      │  UEFI  → reads EFI System Partition (FAT32, /boot/efi), runs .efi binary
      ▼
 2. Bootloader (GRUB2)
      │  stage1 → stage1.5 → stage2
      │  reads /boot/grub2/grub.cfg (RHEL) or /boot/grub/grub.cfg (Debian)
      │  presents menu, loads kernel + initramfs into memory
      ▼
 3. Kernel
      │  decompresses itself, initialises memory management, scheduler
      │  mounts initramfs as a temporary rootfs
      ▼
 4. initramfs / initrd
      │  loads drivers needed to see the REAL root (LVM, RAID, iSCSI, LUKS, NVMe)
      │  unlocks LUKS, assembles md/LVM, fsck, mounts real root
      │  switch_root → pivots to real /
      ▼
 5. systemd (PID 1)
      │  reads default.target (usually multi-user.target or graphical.target)
      │  resolves the dependency graph, starts units in parallel
      ▼
 6. Login (getty / sshd / display manager)
```

**The one-line version for an interview:**
"Firmware → bootloader → kernel → initramfs → switch_root → systemd → target."

### Where each stage fails

| Stage | Typical symptom | First thing to check |
|---|---|---|
| Firmware | No disk found | Boot order, disk seated, RAID controller |
| GRUB | `grub>` or `grub rescue>` prompt | Missing/corrupt `grub.cfg`, wiped MBR |
| Kernel | `Kernel panic - not syncing` | Bad kernel pkg, wrong `root=` |
| initramfs | `dracut-initqueue timeout`, drops to emergency shell | Missing driver, bad UUID in fstab/cmdline |
| systemd | Boots to `emergency.target` | Failed mount in `/etc/fstab` |
| Login | Boots but no SSH | `sshd` failed, firewall, network |

### Inspecting the boot

```bash
# What the kernel was told at boot
$ cat /proc/cmdline
BOOT_IMAGE=/vmlinuz-5.14.0-427.el9.x86_64 root=/dev/mapper/rhel-root ro crashkernel=1G-4G:192M

# Is this system UEFI or BIOS?
$ [ -d /sys/firmware/efi ] && echo UEFI || echo BIOS

# Time spent in each boot stage
$ systemd-analyze
Startup finished in 3.281s (firmware) + 6.017s (loader) + 1.183s (kernel) + 4.921s (userspace) = 15.404s

# Slowest units
$ systemd-analyze blame | head -10

# The critical path (this is what actually delayed the boot)
$ systemd-analyze critical-chain

# Previous boot's logs (-1 = last boot, -2 = the one before)
$ journalctl -b -1 -p err
```

> **Gotcha:** `systemd-analyze blame` lists units by *duration*, but a slow unit that nothing
> waits on does not delay boot. Always use `critical-chain` to find the real culprit.

---

## 2. GRUB2

GRUB config is **generated**, not hand-edited. Editing `grub.cfg` directly gets overwritten on
the next kernel update.

```bash
# Edit this instead:
/etc/default/grub                 # global settings
/etc/grub.d/                      # per-section scripts

# Then regenerate:
# RHEL 8/9 (BIOS)
grub2-mkconfig -o /boot/grub2/grub.cfg
# RHEL 8/9 (UEFI)
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
# Debian/Ubuntu (handles both)
update-grub
```

**Persistent kernel arguments (RHEL 8+, preferred):**

```bash
# Add an argument to ALL kernels, survives kernel updates
grubby --update-kernel=ALL --args="audit=1 transparent_hugepage=never"

# Remove one
grubby --update-kernel=ALL --remove-args="quiet"

# Show current default kernel + its args
grubby --info=DEFAULT
```

### E2E: Reset a lost root password

This is a classic practical exam task.

```text
1. Reboot. At the GRUB menu press `e` on the default entry.
2. Find the line starting with `linux` (or `linux16`/`linuxefi`).
3. Append:   rd.break enforcing=0
      (rd.break stops in initramfs BEFORE switch_root)
4. Ctrl-X to boot.
5. You land in the initramfs shell. Real root is mounted READ-ONLY at /sysroot.

   switch_root:/# mount -o remount,rw /sysroot
   switch_root:/# chroot /sysroot
   sh-5.1# passwd root
   sh-5.1# touch /.autorelabel        # SELinux: relabel on next boot
   sh-5.1# exit
   switch_root:/# exit

6. System reboots, relabels SELinux (can take minutes), comes up with new password.
```

> **Why `/.autorelabel`?** You just wrote `/etc/shadow` from a context where SELinux was
> disabled, so the file gets the wrong label and `sshd`/`login` cannot read it. Skipping this
> step is the #1 reason this procedure "doesn't work."

---

## 3. systemd

🔰 **In plain English**
`systemd` is the **manager** of the machine. It is PID 1 — the first process the kernel
starts, and the ancestor of everything else.

Think of it as a **restaurant manager opening up for the day**. Lots of jobs need doing:
unlock the doors, turn on the ovens, start the coffee machine, log the staff in. Some depend
on others (you cannot brew coffee before the power is on), but many can happen at the same
time. A good manager works out the dependencies and then starts everything possible in
parallel.

The old system (SysV init) was a manager who did jobs strictly one at a time in numbered
order — slow, and if job 12 hung, jobs 13+ never happened. systemd builds a dependency graph
and runs everything it can simultaneously. That is why modern Linux boots in seconds.

Vocabulary you need:

| Term | Plain meaning |
|---|---|
| **unit** | One thing systemd manages (a service, a mount, a timer) |
| **service** | A program systemd keeps running |
| **target** | A *milestone*, like "network is up" or "ready for logins" |
| **enabled** | Start automatically at boot |
| **started/active** | Running *right now* |

⚠️ **enabled ≠ started.** They are completely independent. A service can be running now but
not come back after a reboot (started, not enabled) — a classic cause of "it broke after the
maintenance window". Or enabled but currently stopped. `systemctl enable --now <svc>` does
both.

### Unit types

| Suffix | Purpose |
|---|---|
| `.service` | A daemon or one-shot process |
| `.socket` | Socket activation — starts the service on first connection |
| `.target` | A grouping/synchronisation point (replaces runlevels) |
| `.mount` / `.automount` | Mount points (auto-generated from `/etc/fstab`) |
| `.timer` | Cron replacement |
| `.path` | Activate on filesystem change |
| `.slice` | cgroup resource grouping |

### Runlevel → target mapping

| SysV runlevel | systemd target |
|---|---|
| 0 | `poweroff.target` |
| 1 / S | `rescue.target` |
| 3 | `multi-user.target` |
| 5 | `graphical.target` |
| 6 | `reboot.target` |

```bash
systemctl get-default
systemctl set-default multi-user.target
systemctl isolate rescue.target        # switch NOW, without reboot
```

### Anatomy of a unit file

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
Documentation=https://internal.wiki/myapp
# Ordering only — does NOT pull the unit in:
After=network-online.target postgresql.service
# Dependency — if this fails, we fail:
Requires=postgresql.service
# Weaker: start it if present, but don't fail if it isn't:
Wants=network-online.target

[Service]
Type=notify              # simple|forking|oneshot|notify|exec|dbus|idle
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
Environment=NODE_ENV=production
EnvironmentFile=-/etc/sysconfig/myapp     # '-' => ignore if missing
ExecStartPre=/opt/myapp/bin/preflight.sh
ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s
# Give up if it restarts 5 times within 60s (prevents crash loops):
StartLimitBurst=5
StartLimitIntervalSec=60
TimeoutStopSec=30

# Resource limits (cgroups v2)
MemoryMax=2G
MemoryHigh=1800M         # soft: throttle + reclaim before hitting Max
CPUQuota=200%            # 2 full cores
TasksMax=4096
LimitNOFILE=65535

# Hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict     # / read-only except...
ReadWritePaths=/var/lib/myapp /var/log/myapp
ProtectHome=true
ProtectKernelTunables=true
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

[Install]
WantedBy=multi-user.target
```

**`Type=` — the field people get wrong:**

| Type | systemd considers it "started" when… | Use for |
|---|---|---|
| `simple` | `ExecStart` is *forked* (immediately) | Default; app stays in foreground |
| `exec` | the binary is *executed* (catches exec failures) | Better default than `simple` |
| `forking` | the parent *exits* | Classic daemons that background themselves |
| `oneshot` | the process *exits* | Scripts; pair with `RemainAfterExit=yes` |
| `notify` | the app calls `sd_notify(READY=1)` | Most correct; no race on startup |

> **Interview answer:** "`simple` tells systemd the service is up the instant it forks, which
> is a lie — the port isn't bound yet. Anything ordered `After=` it can race. `notify` (or a
> readiness check) is the correct fix."

### Overrides — never edit vendor unit files

```bash
# Creates /etc/systemd/system/nginx.service.d/override.conf
systemctl edit nginx

# See the fully merged, effective unit:
systemctl cat nginx

# Full unit file copy to override wholesale:
systemctl edit --full nginx

# Show every property with its resolved value:
systemctl show nginx -p MemoryMax -p Restart

# After ANY manual file change:
systemctl daemon-reload
```

**Precedence:** `/etc/systemd/system/` > `/run/systemd/system/` > `/usr/lib/systemd/system/`

### Daily operations

```bash
systemctl status nginx              # state + recent logs + cgroup tree
systemctl list-units --failed       # the first command to run on a sick box
systemctl list-unit-files --state=enabled
systemctl list-dependencies nginx
systemctl mask nginx                # symlink to /dev/null — cannot be started at all
systemctl unmask nginx

# Why did it die?
systemctl show nginx -p ExecMainStatus -p Result
journalctl -u nginx -b --no-pager
```

> `disable` only removes the `[Install]` symlinks — a masked-vs-disabled distinction that
> comes up constantly. A **disabled** unit can still be started manually or pulled in as a
> dependency. A **masked** unit cannot be started by anything.

### systemd timers (cron replacement)

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Nightly backup

[Timer]
OnCalendar=*-*-* 02:30:00
RandomizedDelaySec=900     # jitter, so 500 servers don't stampede at 02:30:00
Persistent=true            # if the box was off at 02:30, run at next boot
Unit=backup.service

[Install]
WantedBy=timers.target
```

```bash
systemctl enable --now backup.timer
systemctl list-timers --all
systemd-analyze calendar "*-*-* 02:30:00"     # validate & preview next elapse
```

**Why timers over cron:** logging via journald, dependency ordering, resource control,
`Persistent=` catch-up, and per-run isolation. **Why cron anyway:** it's simpler and
universally understood.

---

# PART 2: PROCESSES & SCHEDULING

## 4. Process Lifecycle & States

```text
        fork()/clone()
             │
             ▼
     ┌───────────────┐   scheduled    ┌──────────────┐
     │ R  runnable   │◄──────────────►│ R  running   │
     └───────┬───────┘                └──────┬───────┘
             │ waits on resource              │ exit()
             ▼                                ▼
     ┌───────────────┐                ┌──────────────┐
     │ S interruptible│               │ Z  zombie    │
     │ D uninterrupt. │               │ (until       │
     │ T stopped      │               │  parent      │
     └───────────────┘                │  wait()s)    │
                                      └──────────────┘
```

| State | Meaning | Notes |
|---|---|---|
| `R` | Running or runnable | On a CPU or in the run queue |
| `S` | Interruptible sleep | Waiting on an event; can be signalled. Normal. |
| `D` | **Uninterruptible sleep** | Usually blocked on I/O. **Counts toward load average.** Cannot be killed. |
| `Z` | Zombie | Exited, parent hasn't reaped. Holds only a PID slot. |
| `T` | Stopped | `SIGSTOP`/`SIGTSTP`, or under a debugger |
| `I` | Idle kernel thread | Does *not* count toward load |

**Two facts that separate seniors from juniors:**

1. **Load average on Linux includes `D` state**, not just CPU. A load of 50 on a 8-core box
   with 2% CPU usage means **I/O or NFS is hung**, not that the CPU is busy.
2. **You cannot `kill -9` a `D`-state process.** The signal is only delivered when it returns
   to userspace, and it never does until the I/O completes. Fix the storage/NFS, or reboot.

```bash
# Find D-state processes
ps -eo pid,stat,wchan:30,comm | awk '$2 ~ /D/'

# What kernel function is it stuck in?
cat /proc/<pid>/stack
cat /proc/<pid>/wchan
```

### Zombies

A zombie is **not** a leak of memory — it's a leak of a *PID slot* plus its exit status.

```bash
ps -eo pid,ppid,stat,comm | awk '$3 ~ /Z/'
```

You cannot kill a zombie (it's already dead). You must make the **parent** reap it:
`kill -CHLD <ppid>`, or kill/restart the parent. If the parent dies, the zombie is re-parented
to PID 1, which reaps it immediately. **A growing zombie count is a bug in the parent** — it
isn't calling `wait()`.

## 5. Signals

```bash
kill -l          # list all
```

| Signal | № | Default action | Catchable? |
|---|---|---|---|
| `SIGHUP` | 1 | Terminate | Yes — by convention, "reload config" |
| `SIGINT` | 2 | Terminate | Yes (Ctrl-C) |
| `SIGQUIT` | 3 | Core dump | Yes (Ctrl-\\) |
| `SIGKILL` | 9 | Terminate | **No** |
| `SIGTERM` | 15 | Terminate | Yes — **the polite default** |
| `SIGSTOP` | 19 | Stop | **No** |
| `SIGCONT` | 18 | Continue | Yes |
| `SIGUSR1/2` | 10/12 | Terminate | Yes — app-defined |

```bash
kill -TERM <pid>            # ask nicely (default)
kill -9 <pid>               # last resort: no cleanup, no flush, temp files orphaned
pkill -u alice -TERM        # by user
pkill -f 'java.*myapp'      # by full command line
killall -HUP nginx          # by exact name
```

> **Always `SIGTERM` first.** `SIGKILL` skips the handler, so buffers aren't flushed, locks
> aren't released, and children are orphaned. It's the correct answer *only* after `TERM`
> has been given time.

## 6. Priority: nice vs real-time

```bash
# nice: -20 (highest priority) .. 19 (lowest). Only root can go negative.
nice -n 10 ./batch_job.sh
renice -n 5 -p 1234
renice -n 5 -u alice

# I/O priority (CFQ/BFQ schedulers)
ionice -c2 -n7 -p 1234      # class 2 = best-effort, 0-7
ionice -c3 -p 1234          # class 3 = idle, only runs when disk is free
```

`nice` is a **weight, not a reservation**. A `nice 19` process still gets CPU if nothing else
wants it. For hard limits, use cgroups (`CPUQuota=`).

## 7. cgroups v2

🔰 **In plain English**
A **cgroup** ("control group") is a **box you put processes in, with a meter and a limit on
it**. It answers two questions: *how much is this group using?* and *what is the most it is
allowed?*

Without cgroups, every process competes freely and one runaway job can starve the whole
machine. With cgroups you can say: "this group of processes may use at most 2 GB of RAM and
1.5 CPUs." If they exceed the memory cap, they get killed — and **nothing else on the box is
affected**.

This is the foundation of two things you use daily:
- **systemd** puts every service in its own cgroup automatically (that is how
  `MemoryMax=`/`CPUQuota=` work).
- **Containers** are essentially "a process in a cgroup, in namespaces" (§12).

⚠️ This is why a container can be killed for running out of memory while the host has plenty
free — it hit *its own box's* limit, not the machine's. Confusing these two is one of the most
common production misdiagnoses (§10).

cgroups are the mechanism behind both systemd resource control and containers.

```bash
# Confirm v2 (unified hierarchy)
mount | grep cgroup2
stat -fc %T /sys/fs/cgroup     # "cgroup2fs" = v2

# systemd puts every service in its own cgroup:
systemd-cgls                    # tree of cgroups + processes
systemd-cgtop                   # top(1) per cgroup — very useful

# Live resource control without editing files:
systemctl set-property nginx.service MemoryMax=1G CPUQuota=50%
# add --runtime to make it non-persistent
```

Reading a cgroup directly:

```bash
cd /sys/fs/cgroup/system.slice/nginx.service
cat memory.current        # bytes in use
cat memory.max            # limit
cat memory.events         # 'oom' and 'oom_kill' counters
cat cpu.stat              # usage_usec, throttled_usec  <-- throttling evidence
cat pids.current
```

> **Diagnostic gold:** `cpu.stat`'s `nr_throttled` / `throttled_usec` rising means the app is
> hitting its CPU quota. Latency spikes with low reported CPU usage are almost always
> throttling.

---

# PART 3: MEMORY

## 8. The Memory Model

🔰 **In plain English**
Linux deliberately uses **almost all your RAM, almost all the time** — and this alarms people
who then "fix" a problem that does not exist.

Here is the idea. Reading from disk is roughly 1,000× slower than reading from RAM. So when
Linux reads a file, it keeps a copy in spare RAM in case you want it again. That copy is the
**page cache**. If you do read the file again, it comes from memory — instantly.

Think of it as a **kitchen worktop**. The cupboard is your disk; the worktop is RAM. A good
cook leaves ingredients they are using out on the worktop rather than returning them to the
cupboard after every step. A worktop that is bare is not "tidy" — it is wasted space.

Now the key part: **if you suddenly need worktop space, you just sweep the ingredients back
into the cupboard. Instantly.** That is what "reclaimable" means. Cached memory is not really
"used" — it is *borrowed*, and the kernel takes it back the moment a program needs it.

So:

| Column in `free -h` | What it really means |
|---|---|
| `used` | Genuinely taken by programs |
| `buff/cache` | Borrowed for speed — **available on demand** |
| `free` | Completely untouched — **doing nothing useful** |
| `available` | **used + cache the kernel can hand back** ← the only number that matters |

⚠️ **"Free memory is low" is not a problem.** Low `free` with healthy `available` is a system
working exactly as designed. Only `available` running low means real memory pressure.

```text
 Total RAM
 ├── Kernel (slab, page tables, stacks)
 ├── Anonymous memory  (heap/stack — backed by SWAP)
 ├── Page cache        (file data — backed by DISK, reclaimable)
 ├── Buffers           (block device metadata)
 └── Free              (wasted, arguably)
```

```bash
$ free -h
               total   used   free   shared  buff/cache   available
Mem:            31Gi   12Gi  1.2Gi    600Mi        18Gi        18Gi
Swap:          8.0Gi   256Mi  7.7Gi
```

**The most misunderstood output in Linux.**

- `free` being low is **normal and good** — unused RAM is wasted RAM.
- `buff/cache` is **reclaimable**; the kernel drops it instantly under pressure.
- **`available` is the only number that matters.** It estimates what a new process can get
  without swapping.

> Interview question: *"The server has 1 GB free out of 32 GB. Is it out of memory?"*
> Answer: *"Almost certainly not — look at `available`. `free` is low because the page cache
> is doing its job."*

### Detail

```bash
cat /proc/meminfo                # authoritative source
vmstat 1 5                       # si/so columns = swap in/out. Non-zero sustained = trouble
smem -rs uss                     # USS = memory truly unique to a process (best per-proc metric)
ps -eo pid,comm,rss,vsz --sort=-rss | head
```

- **VSZ** — virtual size. Includes mapped-but-never-touched pages. **Mostly meaningless.**
- **RSS** — resident set. Real pages, but **shared libs are double-counted** across processes.
- **PSS** — proportional set. Shared pages divided by sharers. Sums correctly across a system.
- **USS** — unique set. What you'd actually free by killing it.

## 9. Swap & Swappiness

🔰 **In plain English**
**Swap is disk space used as pretend RAM.** When memory gets tight, the kernel takes pages
that have not been touched in a while and parks them on disk, freeing real RAM for things
being used right now.

It is an **overflow car park**. The main car park (RAM) is close and fast. When it fills, you
start parking in the overflow half a mile away (disk). Cars still fit — but anyone who needs
one of those has a long walk.

That walk is the catch: disk is around 1,000× slower than RAM. A little swapping is healthy
housekeeping. **Heavy, continuous swapping ("thrashing") makes a machine feel frozen** — it
spends all its time shuffling pages instead of doing work. A box that is swapping hard is
often *less* responsive than one that OOM-kills a process and moves on.

⚠️ **`vm.swappiness` is the most misunderstood tunable in Linux.** It is **not** "swap when
RAM is N% full."

It is a *preference dial* between two ways of freeing memory: throw away cached file data, or
swap out application memory. Low value = "leave my apps alone, drop the cache first."
High value = "keep the cache, page the apps out."

And `swappiness=0` does **not** disable swap. Under real pressure the kernel will still swap
rather than start killing processes. If you want no swap, remove the swap device.

```bash
swapon --show
cat /proc/sys/vm/swappiness      # default 60

# 0   = only swap to avoid OOM
# 10  = typical for databases
# 60  = desktop/general default
# 100 = swap aggressively
sysctl -w vm.swappiness=10
echo 'vm.swappiness=10' > /etc/sysctl.d/99-swap.conf
```

`swappiness` is **not** "swap only at N% full". It's the relative preference for reclaiming
*anonymous* pages versus *page cache*. Even at 0, the kernel will still swap to avoid OOM.

**Adding swap:**

```bash
fallocate -l 4G /swapfile        # or: dd if=/dev/zero of=/swapfile bs=1M count=4096
chmod 600 /swapfile              # mandatory, or swapon refuses
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

## 10. The OOM Killer

🔰 **In plain English**
When Linux genuinely runs out of memory and cannot free any more, it has a horrible choice:
freeze the whole machine, or **kill one program to save the rest**. It picks the second.

The "OOM killer" is the kernel doing triage. It scores every process — roughly *"how much
memory would I get back, and how important is this?"* — and kills the highest scorer.

The cruel part: it usually kills **your biggest, most important process**, because that is the
one holding the most memory. Your database is a much more attractive target than a small
shell script. This is why the OOM killer feels like it always picks the worst option — it is
optimising for reclaiming memory, not for what you care about.

You can bias the decision with `oom_score_adj` (−1000 = never kill me, +1000 = kill me first).

⚠️ **The distinction that matters in an interview:** there are *two* kinds of OOM.

| | What happened | Where to look |
|---|---|---|
| **System OOM** | The whole machine ran out of RAM | `dmesg`, `free -h` |
| **cgroup OOM** | *One service/container* hit **its own limit** while the host was fine | `memory.events` in its cgroup |

If a container dies with "OOMKilled" but the host shows 20 GB free, it is the second kind.
Adding RAM to the server will not help — you need to raise that service's `MemoryMax`, or fix
the app.

When the kernel cannot reclaim enough, it picks a victim.

```bash
# Evidence
dmesg -T | grep -i -E 'killed process|out of memory'
journalctl -k | grep -i oom
grep -i oom /var/log/messages
```

Typical line:

```text
Out of memory: Killed process 12345 (java) total-vm:8388608kB, anon-rss:6291456kB,
  file-rss:0kB, shmem-rss:0kB, UID:1000 pgtables:12800kB oom_score_adj:0
```

**Victim selection** is driven by `oom_score`, roughly proportional to memory used, adjusted
by `oom_score_adj` (`-1000`..`1000`):

```bash
cat /proc/<pid>/oom_score
cat /proc/<pid>/oom_score_adj

# Protect a critical process (-1000 = never kill)
echo -1000 > /proc/<pid>/oom_score_adj

# In systemd:
#   OOMScoreAdjust=-900
```

> **cgroup OOM vs system OOM:** if a *container or service* is killed but the host has free
> memory, it hit its **cgroup** `memory.max`, not system-wide exhaustion. Check
> `memory.events` in the unit's cgroup. This distinction is a very common interview probe.

## 11. Key VM Tunables

```bash
vm.swappiness=10                 # anon vs cache reclaim preference
vm.min_free_kbytes=131072        # reclaim headroom; too low => allocation stalls
vm.overcommit_memory=0           # 0=heuristic, 1=always allow, 2=strict accounting
vm.dirty_ratio=20                # % of RAM dirty before writers BLOCK
vm.dirty_background_ratio=10     # % before kernel starts flushing in background
vm.vfs_cache_pressure=100        # >100 reclaims dentries/inodes faster
```

**Dirty page tuning matters for write-heavy workloads.** If `dirty_ratio` is large on a box
with lots of RAM, you accumulate gigabytes of dirty pages, then a flush stalls every writer at
once. Lower it for latency-sensitive systems.

```bash
grep -E 'Dirty|Writeback' /proc/meminfo
```

### Dropping caches (diagnostics only — never routine)

```bash
sync; echo 3 > /proc/sys/vm/drop_caches    # 1=page cache, 2=dentries+inodes, 3=both
```

This is a benchmarking aid. Doing it in production just forces expensive re-reads.

---

# PART 4: STORAGE & FILESYSTEMS

## 12. The Storage Stack

```text
   Application
       │  read()/write()
       ▼
   VFS  (virtual filesystem layer)
       │
       ▼
   Filesystem   (ext4 / XFS / Btrfs / ZFS)
       │
       ▼
   Block layer  (I/O scheduler: mq-deadline, bfq, none)
       │
       ▼
   Device Mapper  (LVM, LUKS, multipath)
       │
       ▼
   MD RAID  (optional software RAID)
       │
       ▼
   SCSI / NVMe driver  →  Physical disk
```

```bash
lsblk -f                         # tree: device → partition → LVM → fs → mountpoint
blkid                            # UUIDs and fs types
df -hT                           # usage by mount, with fs type
df -i                            # INODE usage — check this when df says space is free
findmnt                          # mount tree with options
lsof +D /var                     # what's using files under /var
```

## 13. Partitioning

```bash
# MBR: max 2 TiB, 4 primary partitions. GPT: 128 partitions, huge disks. Use GPT.
parted /dev/sdb mklabel gpt
parted -a optimal /dev/sdb mkpart primary 0% 100%
parted /dev/sdb set 1 lvm on

# Interactive alternatives
fdisk /dev/sdb         # now GPT-aware
gdisk /dev/sdb         # GPT-native

partprobe /dev/sdb     # re-read partition table without reboot
```

## 14. LVM — End to End

🔰 **In plain English**
Normally a partition is a fixed slice of one disk. If you guess the size wrong, you are stuck
— you cannot grow it past the end of the disk, and you cannot make it span two disks.

**LVM fixes this by adding a middle layer.** Instead of carving filesystems directly out of
disks, you throw the disks into a shared **pool**, then hand out slices from the pool.

The water analogy:

```text
  Physical Volumes (PV)  =  bottles of water you pour in   (/dev/sdb1, /dev/sdc1)
            ↓
  Volume Group    (VG)  =  the TANK they all pour into     (vg_data)
            ↓
  Logical Volumes (LV)  =  glasses you pour out of the tank (lv_app, lv_logs)
            ↓
  Filesystem            =  what you actually drink from     (XFS / ext4)
```

Why this is worth the extra concept:

- **Grow a filesystem while it is in use.** Add a disk to the tank, top up the glass. No
  downtime, no reboot.
- **A filesystem can span multiple physical disks.** The tank does not care which bottle the
  water came from.
- **Snapshots** — freeze a moment in time for a consistent backup.

⚠️ **Growing is always two steps**, and forgetting the second is the classic beginner error:
1. Make the **glass** bigger — `lvextend`
2. Tell the **filesystem** it now has more room — `xfs_growfs` / `resize2fs`

The `-r` flag on `lvextend` does both for you. Use it.

LVM is the single most important storage skill for an enterprise Linux admin.

```text
  Physical Volume (PV)   /dev/sdb1  /dev/sdc1
            │
            ▼
  Volume Group (VG)      vg_data          <- the pool
            │
            ▼
  Logical Volume (LV)    lv_app  lv_logs  <- carved from the pool
            │
            ▼
  Filesystem             XFS / ext4
```

### Create

```bash
pvcreate /dev/sdb1 /dev/sdc1
vgcreate vg_data /dev/sdb1 /dev/sdc1
lvcreate -L 50G -n lv_app vg_data
# or use a percentage of free space:
lvcreate -l 100%FREE -n lv_logs vg_data

mkfs.xfs /dev/vg_data/lv_app
mkdir -p /app
mount /dev/vg_data/lv_app /app

# Persist — ALWAYS use UUID, never /dev/sdX (device names are not stable)
echo "UUID=$(blkid -s UUID -o value /dev/vg_data/lv_app) /app xfs defaults 0 0" >> /etc/fstab
mount -a            # verify BEFORE rebooting
systemctl daemon-reload
```

### Inspect

```bash
pvs ; vgs ; lvs                   # summary
pvdisplay ; vgdisplay ; lvdisplay # detail
lvs -o +devices                   # which PV each LV lives on
vgs -o +vg_free
```

### Grow (online, no downtime)

```bash
# 1. Add capacity to the pool
pvcreate /dev/sdd1
vgextend vg_data /dev/sdd1

# 2. Grow the LV (+ resize the fs in one step with -r)
lvextend -L +20G -r /dev/vg_data/lv_app
# or consume everything free:
lvextend -l +100%FREE -r /dev/vg_data/lv_app
```

Without `-r`, resize the filesystem manually:

```bash
xfs_growfs /app                       # XFS: mounted, grow-only
resize2fs /dev/vg_data/lv_app         # ext4: online grow
```

> **XFS cannot be shrunk. Ever.** If asked to shrink XFS: back up, `mkfs` smaller, restore.
> ext4 *can* shrink, but only **offline** — unmount, `e2fsck -f`, `resize2fs`, then `lvreduce`.
> Shrinking in the wrong order destroys data. This is a favourite interview trap.

### Snapshots

```bash
# COW snapshot for a consistent backup
lvcreate -L 5G -s -n lv_app_snap /dev/vg_data/lv_app
mount -o ro,nouuid /dev/vg_data/lv_app_snap /mnt/snap     # nouuid needed for XFS
# ... back up from /mnt/snap ...
umount /mnt/snap
lvremove -y /dev/vg_data/lv_app_snap
```

> A snapshot that fills up is **dropped by the kernel** and becomes invalid. Size it for the
> write churn during the backup window, and monitor with `lvs` (the `Data%` column).

## 15. Filesystems

| | ext4 | XFS |
|---|---|---|
| RHEL 7+ default | no | **yes** |
| Grow online | yes | yes |
| **Shrink** | offline only | **never** |
| Max file/fs | 16 TiB / 1 EiB | 8 EiB / 8 EiB |
| Parallel I/O | good | **excellent** (allocation groups) |
| Repair tool | `e2fsck` | `xfs_repair` |
| Best for | general, small files | large files, high concurrency |

```bash
# Create with a label
mkfs.xfs -L appdata /dev/vg_data/lv_app
mkfs.ext4 -L appdata /dev/vg_data/lv_app

# Tune
tune2fs -l /dev/sda1              # ext4: dump superblock
tune2fs -m 1 /dev/sda1            # reduce root reserve 5% -> 1% (big win on large disks)
xfs_info /app                     # XFS geometry

# Repair (UNMOUNT FIRST)
e2fsck -f /dev/sda1
xfs_repair /dev/sda1
xfs_repair -L /dev/sda1           # LAST RESORT: zeroes the log, loses in-flight data
```

### Mount options worth knowing

```text
defaults        rw,suid,dev,exec,auto,nouser,async
noatime         don't update access times — real performance win
relatime        update atime only if older than mtime (modern default)
nodiratime      as noatime, directories only
ro              read-only
noexec          no binary execution — use on /tmp, /var/tmp, /dev/shm
nosuid          ignore setuid bits — use on /home, /tmp
nodev           ignore device files — use on everything except /dev
discard         inline TRIM on SSD (prefer a weekly fstrim.timer instead)
nofail          don't block boot if the device is missing  <- critical for NFS/iSCSI
_netdev         wait for network before mounting
```

**Hardened `/etc/fstab` line:**

```text
UUID=abc-123  /tmp  xfs  defaults,nosuid,nodev,noexec  0 0
```

> **`nofail` is the fix for "server won't boot after adding a disk".** Without it, a missing
> device drops you into emergency mode. Every external/network mount should have it.

## 16. RAID

| Level | Min disks | Capacity | Redundancy | Notes |
|---|---|---|---|---|
| 0 | 2 | 100% | **none** | Stripe. One disk dies, all data gone. |
| 1 | 2 | 50% | 1 disk | Mirror. Best read latency. |
| 5 | 3 | (n-1)/n | 1 disk | Parity. Slow writes; risky rebuilds on big disks. |
| 6 | 4 | (n-2)/n | 2 disks | Survives a second failure during rebuild. |
| 10 | 4 | 50% | ≥1 disk | Stripe of mirrors. **Best for databases.** |

```bash
mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sd[bcde]1
mdadm --detail /dev/md0
cat /proc/mdstat                          # live rebuild progress
mdadm --detail --scan >> /etc/mdadm.conf  # persist the array

# Replace a failed disk
mdadm /dev/md0 --fail /dev/sdb1 --remove /dev/sdb1
mdadm /dev/md0 --add /dev/sdf1
```

> **RAID is not a backup.** It protects against *disk* failure, not `rm -rf`, corruption,
> ransomware, or fire. Say this in an interview.

## 17. Disk Full: Space vs Inodes

🔰 **In plain English**
A filesystem can run out of space in **two independent ways**, and the error message is the
same for both — which is why this confuses people.

Picture a **library**:

- **Blocks** = the *shelves*. Where the actual book contents go.
- **Inodes** = the *index cards*. One card per book, holding its title, author, size,
  permissions, and which shelves hold it.

You can run out of shelf space (huge books). But you can *also* run out of index cards while
shelves sit empty — if you store a million pamphlets, each one still needs its own card.

That is inode exhaustion: **millions of tiny files**. `df -h` says you have 40 GB free;
writing a file still fails with "No space left on device". Always check both:

```bash
df -h     # shelves
df -i     # index cards
```

⚠️ **The third case that fools everyone:** `df` says the disk is 100% full, but `du` adds up
to far less. Nothing is lying. A program still has a **deleted file open**.

When you delete a file, Linux removes its *name*, but keeps the data until every program that
has it open closes it. Log rotation deletes `app.log`, but the app is still writing to the
now-nameless file. `du` walks names, so it cannot see it. `df` asks the filesystem, which
still counts the blocks.

Find it with `lsof +L1` and restart the process — or truncate the file descriptor in place.

```bash
df -h          # blocks
df -i          # INODES  <-- "No space left on device" with free space = inode exhaustion
```

Find the consumers:

```bash
du -xh --max-depth=1 / | sort -rh | head -20      # -x = stay on one filesystem
# inode hogs (millions of tiny files)
for d in /var/*; do echo "$(find $d -xdev 2>/dev/null | wc -l) $d"; done | sort -rn | head
```

### The deleted-but-open-file trap

`df` says 100% full, `du` says the space is free. A process still holds a deleted file open,
so the kernel cannot release the blocks.

```bash
lsof +L1                    # files with link count < 1 == deleted but open
lsof | grep deleted
```

**Fix:** restart the holding process, or truncate the fd in place without restarting:

```bash
> /proc/<pid>/fd/<fd_number>
```

> This is *the* classic "disk full but du shows nothing" scenario, usually caused by an app
> logging to a file that logrotate deleted without signalling the app to reopen it.

---

# PART 5: NETWORKING

## 18. Interfaces & Addressing

`ifconfig`, `route`, and `netstat` are **deprecated**. Use `ip` and `ss`.

| Old | New |
|---|---|
| `ifconfig` | `ip addr` / `ip link` |
| `route -n` | `ip route` |
| `arp -a` | `ip neigh` |
| `netstat -tulpn` | `ss -tulpn` |
| `netstat -i` | `ip -s link` |

```bash
ip addr show                      # or: ip a
ip -br -c addr                    # brief + colour — great for a quick look
ip link set eth0 up
ip addr add 10.0.0.5/24 dev eth0  # runtime only, lost on reboot
ip -s link show eth0              # RX/TX counters, errors, drops
```

### Persistent config

**RHEL 8/9 — NetworkManager (`nmcli`):**

```bash
nmcli con show
nmcli con add type ethernet con-name prod ifname eth0 \
      ip4 10.0.0.5/24 gw4 10.0.0.1
nmcli con mod prod ipv4.dns "10.0.0.53 8.8.8.8"
nmcli con mod prod ipv4.method manual
nmcli con up prod
nmcli dev status
```

**Ubuntu — netplan (`/etc/netplan/01-net.yaml`):**

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses: [10.0.0.5/24]
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses: [10.0.0.53, 8.8.8.8]
```

```bash
netplan try      # applies with auto-rollback after 120s — use this over 'apply' remotely
netplan apply
```

## 19. Routing

```bash
ip route                          # main table
ip route get 8.8.8.8              # WHICH route/source IP would actually be used
ip route add 192.168.5.0/24 via 10.0.0.254 dev eth0
ip route add default via 10.0.0.1
ip rule show                      # policy routing rules
```

`ip route get` is the fastest way to answer "why is traffic leaving the wrong NIC?"

## 20. Sockets & Ports

🔰 **In plain English**
A **port** is a numbered door on a machine. The IP address gets you to the building; the port
number gets you to the right door. Web traffic knocks on 80/443, SSH on 22, PostgreSQL on 5432.

A program that wants to receive connections **listens** on a port — it sits behind that door
waiting for knocks. Only one program can hold a given door at a time, which is why starting a
second service on a taken port fails with "address already in use".

`ss -tulpn` is the command that shows you every door currently being watched, and by whom.

A **connection** is identified by four things together: *source IP, source port, destination
IP, destination port*. That is why thousands of people can all connect to your port 443 at
once — each has a different source port, so every connection is unique.

⚠️ **The one that matters in interviews** — connections do not vanish the instant they close,
and two leftover states mean very different things:

| State | Plain meaning | Is it a problem? |
|---|---|---|
| `TIME_WAIT` | *You* hung up first. The OS waits ~60s in case stray packets are still arriving. | **No.** Normal, self-clearing. |
| `CLOSE_WAIT` | *They* hung up, and your application **never picked up the receiver** | **Yes — application bug.** |

`CLOSE_WAIT` piling up means your code is not calling `close()` on its sockets. No kernel
tuning fixes that; the file descriptors leak until the process hits its limit and starts
refusing connections.

```bash
ss -tulpn                  # TCP+UDP listening, with PID  <-- the workhorse
ss -tan state established
ss -tan state time-wait | wc -l
ss -s                      # summary totals
ss -tin                    # TCP internals: rtt, cwnd, retrans
ss -tp dst 10.0.0.9        # connections to a specific host
```

**TCP states you must be able to explain:**

```text
Client                              Server
  │  ── SYN ──────────────────────►  │   LISTEN
  │  ◄──────────── SYN-ACK ────────  │   SYN_RECV
  │  ── ACK ──────────────────────►  │   ESTABLISHED
  │            ... data ...          │
  │  ── FIN ──────────────────────►  │   CLOSE_WAIT
  │  ◄──────────── ACK ────────────  │
  │  ◄──────────── FIN ────────────  │   LAST_ACK
  │  ── ACK ──────────────────────►  │   CLOSED
  │  TIME_WAIT (2*MSL, ~60s)         │
```

| Symptom | Meaning |
|---|---|
| Many `TIME_WAIT` on client | Normal. Client closed first. Rarely a problem. |
| Many `CLOSE_WAIT` on server | **App bug** — it isn't calling `close()` on the socket. |
| Many `SYN_RECV` | Possible SYN flood, or backlog too small. |

> `CLOSE_WAIT` is the one to flag: the remote side sent FIN, the kernel ACKed, and the
> application never closed its file descriptor. No amount of sysctl tuning fixes it.

## 21. DNS

Resolution order is set by `/etc/nsswitch.conf`:

```text
hosts: files dns myhostname
        │     │
        │     └── /etc/resolv.conf
        └──────── /etc/hosts
```

```bash
cat /etc/resolv.conf
dig example.com                    # full answer with sections
dig +short example.com
dig @10.0.0.53 example.com         # query a SPECIFIC server (bypass local config)
dig -x 10.0.0.5                    # reverse lookup
dig example.com +trace             # walk the delegation from the root
host example.com
resolvectl status                  # systemd-resolved: per-link DNS + DNSSEC state
```

> **`ping` uses NSS (so `/etc/hosts` counts); `dig` talks straight to the DNS server.**
> If `ping myhost` works but `dig myhost` fails, you have an `/etc/hosts` entry masking a
> broken DNS record. This trips people up constantly.

## 22. Firewalls

### firewalld (RHEL default)

```bash
firewall-cmd --state
firewall-cmd --get-active-zones
firewall-cmd --list-all

# Add a service/port — --permanent writes config, but does NOT affect the running firewall
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload                       # NOW it's live

# Rich rule: allow one source to one port
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" \
  source address="10.0.0.0/24" port port="5432" protocol="tcp" accept'
```

> **The classic mistake:** `--permanent` without `--reload` changes nothing right now.
> Conversely, without `--permanent` your rule vanishes on reload/reboot. Do both.

### nftables (modern) / iptables (legacy)

```bash
nft list ruleset
iptables -L -n -v --line-numbers
iptables -t nat -L -n -v

# Rule order matters — first match wins
iptables -I INPUT 1 -s 10.0.0.9 -j DROP        # insert at top
iptables -A INPUT -p tcp --dport 22 -j ACCEPT  # append at bottom
```

## 23. Network Troubleshooting Toolkit

```bash
ping -c4 10.0.0.1                 # L3 reachability (ICMP may be blocked — absence ≠ down)
traceroute -n 8.8.8.8
mtr -n 8.8.8.8                    # continuous traceroute + loss %, best of both
tracepath 8.8.8.8                 # no root needed, discovers MTU

# Is the PORT open? (far more useful than ping)
nc -zv 10.0.0.9 5432
timeout 2 bash -c '</dev/tcp/10.0.0.9/5432' && echo open || echo closed   # no tools needed

curl -sSv https://api.internal/health
curl -w '@-' -o /dev/null -s https://api.internal <<'EOF'
   dns: %{time_namelookup}s  connect: %{time_connect}s
   tls: %{time_appconnect}s  ttfb: %{time_starttransfer}s
 total: %{time_total}s
EOF

# Packet capture
tcpdump -i eth0 -nn 'tcp port 443 and host 10.0.0.9'
tcpdump -i any -nn -s0 -w /tmp/cap.pcap 'port 5432'    # -s0 = full packet, analyse in Wireshark
tcpdump -i eth0 -nn 'tcp[tcpflags] & (tcp-syn|tcp-rst) != 0'   # SYNs and RSTs only
```

**MTU / fragmentation** — the cause of "SSH connects then hangs", or "works for small
requests, times out for large ones", especially over VPN/tunnels:

```bash
ip link show eth0 | grep mtu
# Find the real path MTU: -M do = don't fragment, -s = payload size (add 28 for headers)
ping -M do -s 1472 -c2 10.0.0.9      # 1472 + 28 = 1500
ping -M do -s 1372 -c2 10.0.0.9      # try 1400 if the above fails
```

## 24. Network Namespaces

The primitive behind container networking.

```bash
ip netns add blue
ip netns exec blue ip link set lo up

# veth pair: one end in the namespace, one in the host
ip link add veth0 type veth peer name veth1
ip link set veth1 netns blue

ip addr add 10.10.0.1/24 dev veth0 && ip link set veth0 up
ip netns exec blue ip addr add 10.10.0.2/24 dev veth1
ip netns exec blue ip link set veth1 up
ip netns exec blue ip route add default via 10.10.0.1

ip netns exec blue ping -c2 10.10.0.1
ip netns list
```

---

# PART 6: USERS, AUTH & SECURITY

## 25. Users, Groups, Permissions

```bash
useradd -m -s /bin/bash -G wheel,docker -c "Alice Smith" alice
usermod -aG docker alice          # -a is MANDATORY; without it you REPLACE all groups
userdel -r alice                  # -r also removes home + mail spool
passwd -l alice                   # lock (prefixes hash with '!')
chage -l alice                    # password ageing
chage -M 90 -W 7 alice            # expire in 90 days, warn 7 days before
id alice ; groups alice
```

> **`usermod -G` without `-a` wipes a user's supplementary groups.** Locking yourself out of
> `wheel` this way is a genuine production incident.

Key files:

| File | Contents |
|---|---|
| `/etc/passwd` | `name:x:uid:gid:gecos:home:shell` |
| `/etc/shadow` | `name:$hash:lastchg:min:max:warn:inactive:expire` |
| `/etc/group` | `name:x:gid:members` |
| `/etc/login.defs` | UID ranges, ageing defaults |
| `/etc/skel/` | Template copied into new home dirs |

### Permission bits

```text
   -   rwx   r-x   r--
   │    │     │     │
   │    │     │     └── other
   │    │     └──────── group
   │    └────────────── owner
   └─────────────────── type: - file, d dir, l link, b block, c char, s socket, p pipe
```

```bash
chmod 750 /app ; chmod u+s /usr/bin/tool ; chmod -R g+w /shared
chown -R alice:devs /app
umask 022                    # default: new files 644, dirs 755
```

### Special bits — a guaranteed interview question

| Bit | Octal | On a file | On a directory |
|---|---|---|---|
| **setuid** | 4000 | Run as the file's **owner** | (ignored) |
| **setgid** | 2000 | Run as the file's **group** | **New files inherit the dir's group** |
| **sticky** | 1000 | (ignored) | **Only the owner can delete their files** |

```bash
chmod 2775 /shared      # setgid dir: team collaboration, group is inherited
chmod 1777 /tmp         # sticky: everyone writes, nobody deletes others' files
ls -ld /tmp             # drwxrwxrwt  <- the 't'
find / -perm -4000 -type f 2>/dev/null     # AUDIT: all setuid binaries
```

### ACLs — when POSIX bits aren't enough

```bash
setfacl -m u:bob:rx /app
setfacl -m d:u:bob:rx /app      # 'd:' = default, inherited by new files
getfacl /app
setfacl -x u:bob /app
# ls shows a '+' when ACLs are present:  drwxr-x---+
```

### Immutable attribute

```bash
chattr +i /etc/resolv.conf     # cannot be modified, deleted, or renamed — even by root
lsattr /etc/resolv.conf
chattr -i /etc/resolv.conf
```

> "I can't delete this file and I'm root" → check `lsattr`. Also check whether it's a
> mounted-read-only filesystem.

## 26. sudo

```bash
visudo                          # ALWAYS use visudo — it syntax-checks before saving
visudo -c                       # validate
visudo -f /etc/sudoers.d/ops    # preferred: drop-in files
```

```text
# /etc/sudoers.d/ops
# user  host = (runas_user:runas_group)  NOPASSWD: commands
alice   ALL = (ALL:ALL) ALL
%dba    ALL = (postgres)     NOPASSWD: /usr/bin/psql, /usr/bin/pg_dump
%ops    ALL = (root)         /usr/bin/systemctl restart nginx

Defaults  logfile=/var/log/sudo.log
Defaults  timestamp_timeout=5
Defaults  requiretty
```

```bash
sudo -l              # what am I allowed to run?
sudo -u postgres psql
sudo -i              # full root login shell (loads root's env)
```

> **Security trap:** granting `sudo vi`, `sudo less`, `sudo find`, or `sudo awk` is equivalent
> to granting full root — each can spawn a shell (`:!sh` in vi). Never whitelist an
> interpreter or pager. Also avoid wildcards: `systemctl restart *` lets a user restart
> anything, including services that run arbitrary `ExecStart` you control.

## 27. SSH Hardening

```text
# /etc/ssh/sshd_config
Port 22
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
PermitEmptyPasswords no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
AllowGroups sshusers
X11Forwarding no
LoginGraceTime 30
```

```bash
sshd -t                         # VALIDATE CONFIG BEFORE RESTARTING
systemctl reload sshd
```

> **Never restart sshd over SSH without testing first.** Run `sshd -t`, and keep a second
> session open until you've confirmed you can still log in. A syntax error plus a dropped
> session equals a console/IPMI trip.

```bash
ssh-keygen -t ed25519 -C "alice@corp"       # ed25519 > rsa
ssh-copy-id -i ~/.ssh/id_ed25519.pub alice@server
```

`~/.ssh/config` for jump hosts:

```text
Host bastion
    HostName bastion.corp.com
    User alice
    IdentityFile ~/.ssh/id_ed25519

Host prod-*
    User deploy
    ProxyJump bastion
    ServerAliveInterval 60
```

Permissions matter — sshd silently refuses keys otherwise:

```bash
chmod 700 ~/.ssh ; chmod 600 ~/.ssh/authorized_keys ~/.ssh/id_ed25519
```

## 28. PAM

Pluggable Authentication Modules — the framework every login path goes through.

```text
/etc/pam.d/sshd, /etc/pam.d/su, /etc/pam.d/system-auth

Type       Control      Module
auth       required     pam_unix.so        # who are you
account    required     pam_access.so      # are you allowed (time, origin, expiry)
password   requisite    pam_pwquality.so   # password change rules
session    optional     pam_limits.so      # set up/tear down the session
```

| Control | Behaviour |
|---|---|
| `required` | Must pass; failure recorded but **remaining modules still run** |
| `requisite` | Must pass; fails **immediately** |
| `sufficient` | If it passes (and nothing required failed), stop — success |
| `optional` | Result ignored unless it's the only module |

Lock accounts after failed attempts:

```text
auth  required  pam_faillock.so preauth  silent deny=5 unlock_time=900
auth  required  pam_faillock.so authfail deny=5 unlock_time=900
```

```bash
faillock --user alice            # view
faillock --user alice --reset    # clear
```

Resource limits via `pam_limits` (`/etc/security/limits.conf`):

```text
# domain   type    item     value
*          soft    nofile   65535
*          hard    nofile   65535
@dba       soft    nproc    8192
```

> **`limits.conf` only applies to PAM sessions** (login/SSH). It does **not** apply to systemd
> services — those need `LimitNOFILE=` in the unit. Misunderstanding this is why "I raised
> nofile but the daemon still hits EMFILE."

## 29. SELinux

🔰 **In plain English**
Normal Linux permissions ask: **"who are you?"** If you own the file, you may write it.
SELinux adds a second, stricter question: **"regardless of who you are, is this program
*supposed* to do that?"**

An analogy: file permissions are the **key to a room**. SELinux is a **job description**. A
cleaner may have a master key to every room in the building — but their job description says
"cleaning only". If the cleaner starts reading files in the finance office, they are stopped,
even though the key worked.

So SELinux confines a program to what that *type* of program normally does. If your web server
is compromised, the attacker gets a shell running as the web server — and SELinux still will
not let it read `/etc/shadow` or open random network connections, because web servers do not
do that.

Everything gets a **label** (a "type"): `httpd_t` for the web server process,
`httpd_sys_content_t` for files it is allowed to serve. The rules are between *labels*, not
users.

**That is why SELinux problems almost always look like this:** you copied a file into
`/var/www`, permissions are `644` and owned by the right user, and the web server *still*
gets "Permission denied". The file kept the label from where it came from, so as far as
SELinux is concerned it is not web content.

The fix is to correct the **label**, not to turn SELinux off:

```bash
restorecon -Rv /var/www          # reset labels to what policy says they should be
```

⚠️ Setting `SELINUX=disabled` "to make it work" is the reflex to unlearn. It is the answer
that ends interviews. Read the denial (`ausearch -m AVC -ts recent`), then fix the label, the
port, or the boolean.

Do not disable it. Knowing SELinux is a differentiator.

```bash
getenforce                       # Enforcing | Permissive | Disabled
sestatus
setenforce 0                     # -> Permissive, runtime only (for testing)
# Persist in /etc/selinux/config : SELINUX=enforcing
```

Every process and file has a context: `user:role:type:level`

```bash
ls -Z /var/www/html/index.html   # system_u:object_r:httpd_sys_content_t:s0
ps -eZ | grep nginx
id -Z
```

**Troubleshooting flow:**

```bash
# 1. Is SELinux actually the cause? Look for AVC denials.
ausearch -m AVC,USER_AVC -ts recent
grep -i denied /var/log/audit/audit.log | tail

# 2. Human-readable explanation + suggested fix
sealert -a /var/log/audit/audit.log

# 3. Usually it's a wrong file label. Fix the label, don't disable SELinux:
semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"
restorecon -Rv /srv/web

# 4. Non-standard port
semanage port -a -t http_port_t -p tcp 8443
semanage port -l | grep http

# 5. Booleans toggle common behaviours
getsebool -a | grep httpd
setsebool -P httpd_can_network_connect on      # -P = persistent
```

> **`restorecon` vs `chcon`:** `chcon` changes a label now but is lost on relabel.
> `semanage fcontext` + `restorecon` changes the *policy* so it survives. Always use the
> latter for anything permanent.

## 30. auditd

```bash
# Watch a file for any write/attribute change
auditctl -w /etc/passwd -p wa -k passwd_changes
# Watch a syscall
auditctl -a always,exit -F arch=b64 -S execve -F auid>=1000 -k exec_log

auditctl -l                              # list active rules
ausearch -k passwd_changes -ts today
aureport --summary
```

Persist rules in `/etc/audit/rules.d/audit.rules`.

---

# PART 7: PACKAGES & SOFTWARE

## 31. RHEL — rpm / dnf

```bash
dnf install -y nginx
dnf remove nginx
dnf update                          # everything
dnf update --security               # security errata only
dnf info nginx
dnf search keyword
dnf provides /usr/bin/dig           # WHICH PACKAGE provides this file?
dnf repoquery -l nginx              # list files in a package (not installed)
dnf history                         # transaction log
dnf history undo 42                 # ROLL BACK a transaction
dnf downgrade nginx
dnf module list nginx               # RHEL 8+ modularity (app streams)
dnf module enable nginx:1.22
```

```bash
rpm -qa                    # all installed
rpm -qi nginx              # info
rpm -ql nginx              # files owned
rpm -qf /etc/nginx/nginx.conf   # which package owns this file
rpm -qc nginx              # config files only
rpm -V nginx               # VERIFY against the DB — detects tampering
rpm -qa --last | head      # recently installed
```

`rpm -V` output flags: `S`=size, `M`=mode, `5`=MD5 digest, `T`=mtime, `U`=user, `G`=group.
`S.5....T.  c /etc/nginx/nginx.conf` means the config changed (expected for `c` files).

**Version locking:**

```bash
dnf install python3-dnf-plugin-versionlock
dnf versionlock add nginx-1.22.1
dnf versionlock list
```

## 32. Debian — dpkg / apt

```bash
apt update && apt upgrade
apt full-upgrade                   # allows removing packages to resolve deps
apt install nginx
apt purge nginx                    # remove + config files
apt autoremove
apt-cache policy nginx             # installed vs candidate versions
apt-file search /usr/bin/dig       # needs apt-file update first
apt-mark hold nginx                # pin
```

```bash
dpkg -l | grep nginx
dpkg -L nginx                      # files owned
dpkg -S /etc/nginx/nginx.conf      # which package owns it
dpkg -i pkg.deb ; apt -f install   # fix deps after a manual .deb
dpkg --configure -a                # repair interrupted installs
```

> **Cross-family cheat:** `dnf provides` == `dpkg -S` (installed) / `apt-file search` (any).

## 33. Repositories

```text
# /etc/yum.repos.d/internal.repo
[internal]
name=Internal Repo
baseurl=https://repo.corp.com/rhel/9/x86_64/
enabled=1
gpgcheck=1
gpgkey=https://repo.corp.com/RPM-GPG-KEY-corp
priority=1
```

```bash
dnf repolist ; dnf repolist --all
dnf config-manager --set-disabled epel
createrepo /srv/repo               # build a local repo
reposync --repoid=baseos -p /srv/mirror
```

Never set `gpgcheck=0` in production — that's how you ship an unsigned/hostile package.

---

# PART 8: LOGGING

## 34. journald

```bash
journalctl -u nginx                # one unit
journalctl -u nginx -f             # follow
journalctl -u nginx --since "10 min ago"
journalctl --since "2024-01-15 09:00" --until "2024-01-15 10:00"
journalctl -p err -b               # priority err+ this boot
journalctl -b -1                   # previous boot
journalctl -k                      # kernel (dmesg equivalent)
journalctl _PID=1234
journalctl _UID=1000
journalctl -n 100 --no-pager
journalctl -o json-pretty          # full structured fields
journalctl -u nginx --grep 'timeout'
```

Priorities: `0 emerg, 1 alert, 2 crit, 3 err, 4 warning, 5 notice, 6 info, 7 debug`

**Persistence** — by default many distros keep the journal in `/run` (RAM), so it's lost on
reboot:

```bash
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald
```

```text
# /etc/systemd/journald.conf
Storage=persistent
SystemMaxUse=2G
MaxRetentionSec=1month
```

```bash
journalctl --disk-usage
journalctl --vacuum-size=500M
journalctl --vacuum-time=7d
journalctl --verify
```

## 35. rsyslog

```text
# /etc/rsyslog.conf   (facility.severity   destination)
*.info;mail.none;authpriv.none   /var/log/messages
authpriv.*                       /var/log/secure
mail.*                           -/var/log/maillog      # '-' = async write
cron.*                           /var/log/cron
*.emerg                          :omusrmsg:*

# Ship to a central server  (@@ = TCP, @ = UDP)
*.*   @@logserver.corp.com:514
```

Facilities: `auth, authpriv, cron, daemon, kern, local0-7, mail, syslog, user`

```bash
logger -p local0.info "deploy started"      # generate a test message
rsyslogd -N1                                # validate config
```

## 36. logrotate

```text
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress        # keep the most recent rotation uncompressed
    missingok
    notifempty
    create 0640 myapp myapp
    sharedscripts
    postrotate
        systemctl reload myapp > /dev/null 2>&1 || true
    endscript
}
```

```bash
logrotate -d /etc/logrotate.conf      # DRY RUN — always test first
logrotate -f /etc/logrotate.d/myapp   # force now
cat /var/lib/logrotate/logrotate.status
```

> **`copytruncate` vs `postrotate` reload.** If the app holds the log fd open and you just
> rename the file, it keeps writing to the *deleted* inode — disk fills, `du` shows nothing
> (see §17). Either signal the app to reopen (`postrotate`), or use `copytruncate` (small race
> window, loses lines written during the copy). Prefer `postrotate`.

---

# PART 9: PERFORMANCE ANALYSIS

## 37. The USE Method

For every resource, check **U**tilisation, **S**aturation, **E**rrors.

| Resource | Utilisation | Saturation | Errors |
|---|---|---|---|
| CPU | `mpstat -P ALL 1` | run queue: `vmstat` `r` column | `mcelog` |
| Memory | `free`, `/proc/meminfo` | `si/so`, OOM kills | `dmesg` EDAC |
| Disk | `iostat -x 1` (`%util`) | `aqu-sz`, `await` | `smartctl -a` |
| Network | `sar -n DEV 1` | `ip -s link` drops | `ifconfig` errors |

## 38. The 60-Second Triage

Run this on any "server is slow" ticket:

```bash
uptime                      # load trend: 1/5/15 min
dmesg -T | tail -20         # OOM, hardware, filesystem errors
vmstat 1 5                  # r, b, si/so, us/sy/id/wa
mpstat -P ALL 1 3           # per-CPU: is ONE core pinned?
pidstat 1 3                 # per-process CPU
iostat -xz 1 3              # per-disk: await, %util
free -h                     # available memory
sar -n DEV 1 3              # NIC throughput
ss -s                       # socket summary
top -b -n1 | head -20
```

**Reading `vmstat`:**

```text
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 4  1      0 210344  15400 918232    0    0    12   340  980 2200 25  8 62  5  0
 │  │                                │    │                        │  │  │  │  │
 │  │                                │    │                        │  │  │  │  └ steal (hypervisor)
 │  │                                │    │                        │  │  │  └── iowait
 │  │                                │    │                        │  │  └───── idle
 │  │                                │    │                        │  └──────── system
 │  │                                │    │                        └─────────── user
 │  │                                └────┴──── swap in/out: NON-ZERO = memory pressure
 │  └──── processes BLOCKED on I/O
 └─────── processes RUNNABLE (compare to core count)
```

Rules of thumb:
- `r` consistently > number of cores → **CPU saturated**.
- `b` high with high `wa` → **I/O bound**.
- `si`/`so` sustained non-zero → **memory pressure** (not just "swap is allocated").
- `st` > 0 → **hypervisor stealing CPU**; you're a noisy-neighbour victim.

## 39. Load Average

🔰 **In plain English**
Load average is **how long the queue is**, not how busy the CPU is. Those are different
things, and mixing them up is the most common misreading in Linux.

Think of **checkouts at a supermarket**. Load average counts everyone *being served* plus
everyone *waiting in line*. So:

- Load **4** on a **4-core** machine = 4 checkouts, 4 customers. Perfectly busy, no queue.
- Load **16** on a **4-core** machine = 4 checkouts, 16 customers. Three-quarters are waiting.

**So the number is meaningless until you divide by `nproc`.** "Load is 8" is not a fact you
can act on. "Load is 8 on 4 cores" is.

The three numbers are the last **1, 5 and 15 minutes**, which tell you the *direction*:

```text
load average: 8.42, 6.15, 4.03     ← rising fast. Getting worse.
load average: 4.03, 6.15, 8.42     ← falling. Already recovering.
```

⚠️ **The Linux-specific twist that interviewers probe:** on Linux, load counts processes
waiting for **disk/network I/O** as well as CPU (state `D`). On most other Unixes it does not.

So you can see **load 50 with a 98% idle CPU**. That is not a contradiction — it means 50
processes are stuck waiting on storage, usually a hung NFS mount or a dead SAN path. Adding
CPUs would do nothing. The queue is at the *disk*, not the till.

That single insight is worth memorising: **high load + idle CPU = I/O problem, not a CPU
problem.**

```bash
$ uptime
 14:32:01 up 42 days,  3:14,  2 users,  load average: 8.42, 6.15, 4.03
```

Load = runnable (`R`) **+ uninterruptible (`D`)** processes, exponentially averaged over
1/5/15 minutes.

- Compare against `nproc`. Load 8 on 16 cores = ~50% busy. Load 8 on 2 cores = badly saturated.
- Trend matters: `8.42, 6.15, 4.03` is **rising**. The reverse is recovering.
- **High load + low CPU = I/O or NFS hang**, because of `D` state.

```bash
nproc
lscpu
```

## 40. CPU Analysis

```bash
mpstat -P ALL 1                   # per-core. One core at 100% = single-threaded bottleneck
pidstat -t 1                      # per-THREAD
top -H -p <pid>                   # threads of one process
perf top                          # live hot functions
perf record -F 99 -g -p <pid> -- sleep 30 ; perf report
```

`%steal` on a VM means the hypervisor isn't giving you scheduled time — the fix is capacity or
placement, not tuning the guest.

## 41. Disk I/O Analysis

```bash
iostat -xz 1
```

```text
Device  r/s    w/s   rkB/s   wkB/s  r_await w_await aqu-sz  %util
nvme0n1 120.0  340.0 4800.0 13600.0    0.45    1.20   0.62   12.4
                                       │       │      │      │
                                       │       │      │      └ % of time with I/O in flight
                                       │       │      └─────── avg queue depth  <- SATURATION
                                       └───────┴────────────── avg latency (ms) <- what users feel
```

- **`await` is the number that matters.** NVMe should be <1 ms; SAS 10k ~5–10 ms; anything
  >20 ms sustained is a problem.
- **`%util` is misleading on SSD/NVMe.** These devices handle many parallel requests, so 100%
  `%util` only means "≥1 request in flight", not "saturated". Use `aqu-sz` and `await`.

```bash
iotop -oPa                        # which PROCESS is doing the I/O
pidstat -d 1
smartctl -a /dev/sda              # disk health: reallocated sectors, wear
biolatency                        # bcc/eBPF: latency histogram
```

## 42. Tracing

```bash
# System calls
strace -p <pid>                       # attach
strace -f -e trace=openat,read ./app  # follow forks, filter syscalls
strace -c -p <pid>                    # SUMMARY: count + time per syscall  <- start here
strace -tt -T -p <pid>                # timestamps + duration per call

# Library calls
ltrace -p <pid>

# Files/sockets held open
lsof -p <pid>
lsof -i :8080
lsof -u alice
lsof /var/log/app.log

# Where is the time going? (eBPF/bcc)
execsnoop     # every exec()
opensnoop     # every open()
biosnoop      # every block I/O
tcpconnect    # every outbound TCP connect
funclatency   # latency histogram of a kernel function
```

> `strace` pauses the target on **every syscall** — it can slow a process by 10–100×. Never
> attach it to a busy production process without expecting impact. Use `-c` for a quick
> summary, or eBPF tools which are far cheaper.

## 43. sysctl Tuning

```bash
sysctl -a                        # everything
sysctl net.ipv4.tcp_syncookies
sysctl -w net.ipv4.ip_forward=1  # runtime only
sysctl -p /etc/sysctl.d/99-tuning.conf
```

```text
# /etc/sysctl.d/99-tuning.conf

# --- Network ---
net.core.somaxconn = 65535               # accept queue depth (match app's listen backlog)
net.ipv4.tcp_max_syn_backlog = 65535
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1                # safe: reuse TIME_WAIT for OUTBOUND connections
net.ipv4.ip_local_port_range = 10240 65535
net.ipv4.tcp_syncookies = 1

# --- Files ---
fs.file-max = 2097152
fs.inotify.max_user_watches = 524288     # fixes "inotify watch limit reached"

# --- Memory ---
vm.swappiness = 10
vm.max_map_count = 262144                # Elasticsearch, JVM-heavy apps
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5

# --- Security ---
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
kernel.randomize_va_space = 2
```

> `net.ipv4.tcp_tw_recycle` **was removed in kernel 4.12** and must never be used — it broke
> NAT'd clients badly. If you see it in a runbook, that runbook is stale. `tcp_tw_reuse` is
> the safe one.

## 44. Limits

```bash
ulimit -a                    # current shell
ulimit -n 65535              # soft limit, this shell only
cat /proc/<pid>/limits       # ACTUAL limits of a running process  <- the truth
```

Three different places set limits — know which applies:

| Mechanism | Applies to |
|---|---|
| `/etc/security/limits.conf` | PAM sessions (login, SSH) |
| `LimitNOFILE=` in a unit | systemd services |
| `ulimit` in a shell/script | that shell and its children |

---

# PART 10: AUTOMATION

## 45. Bash for Admins

**Every production script starts with this:**

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'
```

| Flag | Effect |
|---|---|
| `-e` | Exit on any command failure |
| `-u` | Error on undefined variable (catches typos) |
| `-o pipefail` | A pipeline fails if **any** stage fails, not just the last |
| `-E` | ERR traps are inherited by functions/subshells |

Without `pipefail`, `false | true` succeeds — which is how silent data-loss bugs happen.

### A real-world template

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

readonly SCRIPT_NAME="${0##*/}"
readonly LOCK="/var/run/${SCRIPT_NAME}.lock"
readonly LOG="/var/log/${SCRIPT_NAME}.log"

log()  { printf '%s [%s] %s\n' "$(date -Is)" "$1" "${*:2}" | tee -a "$LOG"; }
die()  { log ERROR "$*"; exit 1; }
cleanup() { rm -f "$LOCK"; }
trap cleanup EXIT
trap 'die "failed at line $LINENO"' ERR

# Prevent concurrent runs (flock is atomic; [ -f ] is not)
exec 9>"$LOCK"
flock -n 9 || die "already running"

[[ $EUID -eq 0 ]] || die "must run as root"
command -v rsync >/dev/null || die "rsync not installed"

main() {
    local src="${1:?usage: $SCRIPT_NAME <src> <dst>}"
    local dst="${2:?usage: $SCRIPT_NAME <src> <dst>}"
    log INFO "syncing $src -> $dst"
    rsync -aAX --delete "$src/" "$dst/" || die "rsync failed"
    log INFO "done"
}

main "$@"
```

### Idioms that come up in interviews

```bash
# Safe iteration over filenames (handles spaces, newlines)
find /var/log -name '*.log' -print0 | while IFS= read -r -d '' f; do
    echo "$f"
done

# Default / required variables
: "${ENVIRONMENT:=staging}"        # default if unset
: "${API_KEY:?must be set}"        # abort if unset

# String manipulation without sed
f="/var/log/app.log"
echo "${f##*/}"      # app.log        (basename)
echo "${f%/*}"       # /var/log       (dirname)
echo "${f%.log}"     # /var/log/app   (strip suffix)
echo "${f//\//_}"    # _var_log_app.log (replace all)

# Arrays
declare -a hosts=(web1 web2 web3)
for h in "${hosts[@]}"; do ssh "$h" uptime; done

# Always quote. "$@" preserves arguments; $* does not.
```

### Text processing

```bash
awk '{sum+=$3} END {print sum}' file
awk -F: '$3 >= 1000 {print $1}' /etc/passwd        # real users
awk '$9 == 500 {print $7}' access.log | sort | uniq -c | sort -rn | head

sed -i.bak 's/old/new/g' file                       # in-place with backup
sed -n '10,20p' file                                # print a line range
sed '/^#/d;/^$/d' config                            # strip comments + blanks

grep -rn 'pattern' /etc --include='*.conf'
grep -c ERROR app.log
grep -A3 -B3 'Exception' app.log

cut -d: -f1,7 /etc/passwd
sort -k3 -n -r file
comm -13 <(sort a) <(sort b)          # lines only in b
join -t: -1 1 -2 1 file1 file2
tr -s ' ' | column -t
xargs -P8 -n1 -I{} sh -c 'echo {}'    # parallel execution
```

**Top-talkers one-liner** (a genuine interview favourite):

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
```

## 46. cron

```bash
crontab -e ; crontab -l ; crontab -r
crontab -u alice -l
```

```text
# m  h  dom mon dow   command
  */5 *   *   *   *   /usr/local/bin/check.sh
  0   2   *   *   *   /usr/local/bin/backup.sh
  0   0   1   *   *   /usr/local/bin/monthly.sh
  @reboot             /usr/local/bin/startup.sh

  ┌── minute (0-59)
  │ ┌── hour (0-23)
  │ │ ┌── day of month (1-31)
  │ │ │ ┌── month (1-12)
  │ │ │ │ ┌── day of week (0-7, 0 and 7 = Sunday)
  *  *  *  *  *
```

**Cron gotchas that cause real incidents:**

1. **PATH is minimal** (`/usr/bin:/bin`). Always use absolute paths, or set `PATH=` at the top.
2. **No login shell** — `.bashrc`/`.bash_profile` are not sourced. Export what you need.
3. **`%` is a literal newline** in crontabs — escape it: `date +\%F`.
4. **Output is mailed**, and if mail isn't configured it's lost. Redirect explicitly:
   `>> /var/log/job.log 2>&1`.
5. **Overlapping runs** — a 6-minute job on a 5-minute schedule stacks up. Use `flock`:
   `*/5 * * * * /usr/bin/flock -n /tmp/job.lock /usr/local/bin/job.sh`

## 47. Ansible

```text
inventory.ini
---
[web]
web1.corp.com
web2.corp.com

[db]
db1.corp.com

[prod:children]
web
db

[all:vars]
ansible_user=deploy
```

```yaml
# site.yml
- name: Configure web servers
  hosts: web
  become: true
  serial: 1                      # rolling: one host at a time
  max_fail_percentage: 0

  vars:
    nginx_port: 8080

  handlers:
    - name: reload nginx
      ansible.builtin.systemd:
        name: nginx
        state: reloaded

  tasks:
    - name: Install nginx
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Deploy config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        mode: '0644'
        validate: 'nginx -t -c %s'     # VALIDATE before replacing — prevents outages
      notify: reload nginx

    - name: Ensure running
      ansible.builtin.systemd:
        name: nginx
        state: started
        enabled: true

    - name: Wait for port
      ansible.builtin.wait_for:
        port: "{{ nginx_port }}"
        timeout: 30
```

```bash
ansible -i inventory.ini all -m ping
ansible -i inventory.ini web -a 'uptime'
ansible-playbook -i inventory.ini site.yml --check --diff    # DRY RUN
ansible-playbook -i inventory.ini site.yml --limit web1 --tags nginx
ansible-vault encrypt group_vars/prod/secrets.yml
```

**Idempotency** is the core concept: running the playbook twice must produce
`changed=0` the second time. `command`/`shell` modules are *not* idempotent unless you add
`creates=`, `removes=`, or `changed_when:`.

---

# PART 11: BACKUP & HA

## 48. Backup Strategy

**3-2-1 rule:** 3 copies, on 2 different media, 1 off-site.

```bash
# rsync — incremental file sync
rsync -aAXv --delete --exclude={'/proc/*','/sys/*','/dev/*','/tmp/*','/run/*'} \
      / /mnt/backup/
#  -a archive (rlptgoD)  -A ACLs  -X xattrs  --delete mirror deletions

# Hard-link snapshots: full-looking backups at incremental cost
rsync -aAX --link-dest=/backup/prev / /backup/$(date +%F)/

# tar
tar -czvf /backup/etc-$(date +%F).tar.gz /etc
tar -tzf archive.tar.gz            # list without extracting
tar -xzf archive.tar.gz -C /restore

# Block-level clone
dd if=/dev/sda of=/backup/sda.img bs=64M status=progress
```

**Test your restores.** An untested backup is a hypothesis, not a backup.

## 49. High Availability Concepts

| Term | Meaning |
|---|---|
| **Quorum** | Majority needed to act; prevents split-brain. `floor(N/2)+1`. Use **odd** node counts. |
| **Split-brain** | Both nodes believe they're primary → data divergence. |
| **Fencing / STONITH** | Forcibly power off a suspect node so it *cannot* write. |
| **VIP** | Floating IP that moves to the active node. |
| **Failover** | Promoting standby to active. |

```bash
# Pacemaker/Corosync (RHEL HA add-on)
pcs status
pcs cluster start --all
pcs resource create vip IPaddr2 ip=10.0.0.100 cidr_netmask=24
pcs constraint colocation add vip with nginx
```

> **Fencing is not optional.** Without STONITH, a hung-but-alive node can wake up and corrupt
> shared storage. Interviewers ask this to see if you've run real clusters.

## 50. keepalived (simple VIP failover)

```text
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100          # BACKUP uses a lower number
    advert_int 1
    authentication { auth_type PASS; auth_pass secret }
    virtual_ipaddress { 10.0.0.100/24 }
    track_script { chk_nginx }
}
vrrp_script chk_nginx {
    script "/usr/bin/killall -0 nginx"
    interval 2
    weight -20
}
```

---

# PART 12: CONTAINERS FROM FIRST PRINCIPLES

A container is **not** a VM. It is a normal Linux process with restricted visibility.

```text
Container = namespaces (isolation)
          + cgroups    (resource limits)
          + capabilities / seccomp / LSM (privilege reduction)
          + union filesystem (image layers)
```

### Namespaces

| Namespace | Isolates |
|---|---|
| `pid` | Process IDs (container PID 1) |
| `net` | Interfaces, routes, firewall rules |
| `mnt` | Mount points |
| `uts` | Hostname, domain |
| `ipc` | Shared memory, semaphores |
| `user` | UID/GID mapping (root inside ≠ root outside) |
| `cgroup` | cgroup root |
| `time` | Boot/monotonic clocks |

```bash
lsns                              # list all namespaces
ls -l /proc/<pid>/ns/             # namespaces of a process
nsenter -t <pid> -n ip addr       # run a command in that process's NET namespace
unshare --pid --fork --mount-proc bash    # make your own PID namespace
```

```bash
# Inspect a running container from the host
CPID=$(podman inspect -f '{{.State.Pid}}' mycontainer)
nsenter -t "$CPID" -n ss -tulpn
cat /proc/$CPID/cgroup
```

### podman / docker

```bash
podman run -d --name web -p 8080:80 --memory=512m --cpus=1.5 nginx:alpine
podman ps -a ; podman logs -f web ; podman exec -it web sh
podman stats
podman inspect web
podman system prune -a
```

> `podman` is daemonless and supports rootless containers — increasingly the RHEL default.
> The CLI is deliberately Docker-compatible (`alias docker=podman` usually just works).

---

# PART 13: END-TO-END RUNBOOKS

## R1. Server Won't Boot

```text
STEP 1  Identify where it stops
  - No POST / no disk        -> firmware, hardware, boot order
  - grub> or grub rescue>    -> bootloader
  - Kernel panic             -> kernel / initramfs
  - "emergency mode" prompt  -> systemd (usually /etc/fstab)

STEP 2  GRUB rescue
  grub> ls                             # list disks: (hd0,gpt2)
  grub> set root=(hd0,gpt2)
  grub> linux /vmlinuz-... root=/dev/mapper/rhel-root
  grub> initrd /initramfs-....img
  grub> boot
  # Once booted:
  grub2-mkconfig -o /boot/grub2/grub.cfg && grub2-install /dev/sda

STEP 3  Bad kernel after an update
  At the GRUB menu, choose the PREVIOUS kernel.
  grubby --set-default /boot/vmlinuz-<older>

STEP 4  Bad /etc/fstab (most common cause of emergency mode)
  Log in as root at the emergency prompt.
  mount -o remount,rw /
  vi /etc/fstab            # comment out the offending line, or add 'nofail'
  mount -a                 # MUST be clean before you reboot
  systemctl reboot

STEP 5  Corrupt root filesystem
  Boot rescue/live media.
  xfs_repair /dev/mapper/rhel-root      # or: e2fsck -f
  # if the XFS log is corrupt and nothing else works:
  xfs_repair -L /dev/mapper/rhel-root   # DESTRUCTIVE: discards the log

STEP 6  Still broken -> boot install media in rescue mode, chroot /mnt/sysroot,
        rebuild initramfs:  dracut -f --regenerate-all
```

## R2. High Load Average

```text
STEP 1  Quantify
  uptime; nproc
  Load 8 on 16 cores = fine. Load 80 on 8 cores = severe.
  Rising or falling? (compare 1/5/15 min)

STEP 2  CPU-bound or I/O-bound?
  vmstat 1 5
    high 'r', high us/sy      -> CPU bound      -> STEP 3
    high 'b', high 'wa'       -> I/O bound      -> STEP 4
    high 'st'                 -> hypervisor steal -> capacity/placement issue
    load high, CPU idle       -> D-state / NFS  -> STEP 5

STEP 3  CPU
  mpstat -P ALL 1        # one core pinned => single-threaded hot spot
  pidstat 1              # which process
  top -H -p <pid>        # which thread
  perf top -p <pid>      # which function

STEP 4  Disk
  iostat -xz 1           # look at await and aqu-sz, NOT %util
  iotop -oPa             # which process
  Check for: RAID rebuild (cat /proc/mdstat), backup job, log flood, failing disk
  smartctl -a /dev/sda

STEP 5  D-state
  ps -eo pid,stat,wchan:30,comm | awk '$2 ~ /D/'
  cat /proc/<pid>/stack
  Usually NFS or a SAN path. Check: mount | grep nfs ; dmesg | grep -i 'nfs\|i/o error'
  multipath -ll
  These CANNOT be killed. Restore storage, or reboot.

STEP 6  Mitigate, then fix
  renice / ionice the offender, or stop the batch job.
  Then address the root cause.
```

## R3. Disk Full

```text
STEP 1  Space or inodes?
  df -h        # blocks
  df -i        # inodes -- "No space left on device" WITH free space = inodes

STEP 2  Find it
  du -xh --max-depth=1 / | sort -rh | head -20      # descend the biggest
  find / -xdev -type f -size +500M -exec ls -lh {} + 2>/dev/null

STEP 3  Deleted-but-open files (df full, du clean)
  lsof +L1
  # reclaim WITHOUT restarting:
  > /proc/<pid>/fd/<n>
  # or restart the holding service

STEP 4  Usual suspects
  journalctl --disk-usage ; journalctl --vacuum-size=500M
  du -sh /var/log/* ; check logrotate is working: logrotate -d /etc/logrotate.conf
  dnf clean all / apt clean
  /tmp, /var/tmp, core dumps (coredumpctl list), old kernels:
  dnf remove --oldinstallonly --setopt installonly_limit=2

STEP 5  Inode exhaustion
  for d in /var/*; do echo "$(find $d -xdev 2>/dev/null|wc -l) $d"; done | sort -rn | head
  Typically a session/cache dir with millions of tiny files.

STEP 6  Extend (if LVM)
  vgs                                  # free space in the VG?
  lvextend -l +100%FREE -r /dev/vg/lv  # grows LV + filesystem
```

## R4. Out of Memory

```text
STEP 1  Confirm it was the OOM killer
  dmesg -T | grep -i -E 'killed process|out of memory'
  journalctl -k | grep -i oom

STEP 2  System-wide or cgroup?
  If a service/container died while the host had free RAM,
  it hit its OWN limit:
    cat /sys/fs/cgroup/system.slice/<unit>/memory.events    # oom_kill count
    systemctl show <unit> -p MemoryMax
  -> raise MemoryMax, or fix the app.

STEP 3  System-wide
  free -h                 # look at 'available', not 'free'
  ps -eo pid,comm,rss --sort=-rss | head
  smem -rs uss            # USS = what you'd actually reclaim

STEP 4  Leak or legitimate growth?
  Watch RSS over time. A steadily climbing RSS with flat workload = leak.
  pmap -x <pid> | tail -1
  cat /proc/<pid>/status | grep -E 'VmRSS|VmSwap'

STEP 5  Mitigate
  Add swap (buys time, doesn't fix a leak)
  Protect critical procs: OOMScoreAdjust=-900
  Set MemoryMax per service so ONE app can't take the box down
  Right-size / fix the leak
```

## R5. Service Won't Start

```text
STEP 1  systemctl status <svc> -l
        journalctl -u <svc> -n 100 --no-pager
STEP 2  Config syntax
        nginx -t ; sshd -t ; httpd -t ; named-checkconf
STEP 3  Port already bound?
        ss -tulpn | grep :<port>
STEP 4  Permissions / SELinux
        ls -lZ on the data + config paths
        ausearch -m AVC -ts recent
        # non-standard port:
        semanage port -a -t http_port_t -p tcp 8443
STEP 5  Dependencies
        systemctl list-dependencies <svc>
        systemctl --failed
STEP 6  Resource limits
        cat /proc/<pid>/limits
        # EMFILE => LimitNOFILE= in the unit (NOT limits.conf)
STEP 7  Run it in the foreground, as the service user, to see the real error
        sudo -u nginx /usr/sbin/nginx -g 'daemon off;'
```

## R6. Cannot SSH In

```text
STEP 1  From your machine
  ping <host>                       # ICMP may be filtered; absence != down
  nc -zv <host> 22                  # is the PORT reachable?
  ssh -vvv user@host                # verbose: shows exactly where it stops

STEP 2  Read the -vvv output
  "Connection refused"  -> sshd not running, or wrong port
  "Connection timed out"-> firewall / security group / routing
  "Permission denied (publickey)" -> key problem, reached sshd fine
  "Too many authentication failures" -> agent offering too many keys; use -o IdentitiesOnly=yes

STEP 3  On the host (console / IPMI / cloud serial console)
  systemctl status sshd
  ss -tulpn | grep :22
  sshd -t
  journalctl -u sshd -n 50

STEP 4  Key problems (most common)
  ls -ld ~/.ssh                     # must be 700
  ls -l ~/.ssh/authorized_keys      # must be 600, owned by the user
  # SELinux label:
  restorecon -Rv ~/.ssh
  # home dir must NOT be group/world writable

STEP 5  Account state
  passwd -S <user>                  # locked?
  chage -l <user>                   # expired?
  faillock --user <user>            # locked out by failed attempts?
  grep -E 'AllowUsers|AllowGroups|DenyUsers' /etc/ssh/sshd_config

STEP 6  Firewall
  firewall-cmd --list-all
  iptables -L -n | grep 22
```

## R7. Network Unreachable

```text
STEP 1  Layer 1/2 -- is the link up?
  ip -br link                       # state UP?
  ethtool eth0 | grep -i 'link detected'
  ip -s link show eth0              # errors/drops climbing?

STEP 2  Layer 3 -- addressing and routes
  ip -br addr
  ip route
  ip route get <dest>               # which route/source would be used
  ping <default-gw>

STEP 3  Layer 3 beyond the gateway
  ping 8.8.8.8                      # raw IP -- isolates DNS from connectivity
  mtr -n 8.8.8.8                    # where does it start losing packets?

STEP 4  DNS (if IP works but names don't)
  cat /etc/resolv.conf
  dig @<server> example.com
  grep hosts /etc/nsswitch.conf

STEP 5  Layer 4 -- the port
  nc -zv <host> <port>
  ss -tulpn | grep <port>           # is anything LISTENING on this side?
  # listening on 127.0.0.1 only will refuse remote connections -- check the bind address

STEP 6  Firewall (both ends + anything between)
  firewall-cmd --list-all ; iptables -L -n -v
  Cloud security groups / NACLs / on-prem ACLs

STEP 7  Prove it with packets
  tcpdump -i any -nn "host <peer> and port <port>"
  SYN out, no SYN-ACK back  -> blocked/dropped upstream
  SYN out, RST back         -> reached the host, nothing listening
  No SYN leaving at all     -> local routing or local firewall
```

## R8. Filesystem Mounted Read-Only

```text
STEP 1  Confirm and find out why
  mount | grep ' / '
  dmesg -T | grep -i -E 'error|remount|ext4|xfs|i/o'
  The kernel remounts ro on I/O errors to protect data. This is a SYMPTOM.

STEP 2  Check the hardware
  smartctl -a /dev/sda
  multipath -ll                     # SAN path failure?

STEP 3  Repair (unmount first; from rescue media for /)
  xfs_repair /dev/mapper/vg-lv
  e2fsck -fy /dev/mapper/vg-lv

STEP 4  Remount
  mount -o remount,rw /
  Do NOT just remount rw and carry on -- if the disk is failing you will corrupt data.
  Replace the disk.
```

---

# PART 14: INTERVIEW PREPARATION

## 51. Core Questions with Model Answers

**Q: What happens when you type a command and press Enter?**
Shell parses/expands → `fork()` → child `execve()`s the binary → kernel loads the ELF, maps
segments, invokes the dynamic linker (`ld.so`) to resolve shared libs → jumps to `_start` →
`main()`. Parent `wait()`s. On exit, the child becomes a zombie until reaped; the exit status
lands in `$?`.

**Q: Difference between a process and a thread?**
Both are scheduled entities (`task_struct`). A thread shares the address space, file
descriptors, and signal handlers with its peers; a process does not. `fork()` copies (COW),
`clone()` with `CLONE_VM|CLONE_FILES` shares.

**Q: Hard link vs soft link?**
A hard link is another directory entry pointing at the *same inode* — same filesystem only,
cannot link directories, file data survives until link count hits 0. A symlink is a file whose
content is a *path* — can cross filesystems, can point at directories, breaks if the target
moves. `ls -i` shows the inode; `stat` shows the link count.

**Q: What is a zombie? How do you kill one?**
A process that exited but whose parent hasn't `wait()`ed, so the exit status remains in the
table. You can't kill it — it's already dead. Signal the **parent** (`kill -CHLD <ppid>`) or
restart it. If the parent dies, PID 1 adopts and reaps it.

**Q: Load average is 40 but CPU is 95% idle. Explain.**
Linux load includes `D`-state (uninterruptible) tasks. 40 processes are blocked on I/O —
typically a hung NFS mount or failed SAN path — not CPU. Find them with
`ps -eo pid,stat,wchan`, check `dmesg` and `multipath -ll`.

**Q: `df` says 100% full but `du` shows far less. Why?**
A deleted file is still held open by a process, so the kernel can't free the blocks. Find with
`lsof +L1`; reclaim by restarting the process or `> /proc/<pid>/fd/<n>`.

**Q: Difference between `SIGTERM` and `SIGKILL`?**
`TERM` is catchable — the app can flush buffers, release locks, and exit cleanly. `KILL`
cannot be caught or ignored; the kernel destroys the process immediately, leaving temp files,
locks, and children orphaned. Always `TERM` first.

**Q: How do you make a config change survive reboot?**
Depends on the subsystem: `sysctl` → `/etc/sysctl.d/*.conf`; mounts → `/etc/fstab` (by UUID);
network → `nmcli`/netplan; services → `systemctl enable`; kernel args → `grubby`; firewall →
`firewall-cmd --permanent` + `--reload`.

**Q: `setuid` on a directory?**
Ignored. On a *directory*, `setgid` makes new entries inherit the directory's group, and the
sticky bit restricts deletion to the owner. `setuid` only affects executables.

**Q: When would you use XFS over ext4 — and what's the catch?**
XFS for large files and high parallel I/O (RHEL default). Catch: **XFS cannot be shrunk**.
Plan LV sizes accordingly, or use ext4 where shrinking is plausible.

**Q: What's the difference between `TIME_WAIT` and `CLOSE_WAIT`?**
`TIME_WAIT` is on the side that closed first, lasting 2×MSL to absorb stray packets — normal.
`CLOSE_WAIT` means the peer sent FIN and your application never called `close()` — an
application bug, not a tuning problem.

**Q: How does `sudo` differ from `su`?**
`su` switches user by authenticating as the *target* (needs root's password). `sudo`
authenticates as *yourself* and consults `/etc/sudoers` for authorisation — per-command
granularity, full audit trail, no shared password.

## 52. Scenario Questions

1. Users report intermittent 502s from an app behind nginx. Walk me through your diagnosis.
2. A server was rebooted and never came back. You have IPMI. What now?
3. `/var` filled up at 03:00 and paged you. Find the cause and prevent a recurrence.
4. A batch job that ran in 20 minutes now takes 4 hours. Nothing changed. Investigate.
5. You must patch 200 servers with zero downtime. Design the process.
6. A developer needs to restart one service in production, nothing else. Implement it.
7. SSH to one host hangs after the banner; other hosts are fine. Why?
8. Database latency spikes every night at 02:00. Diagnose.
9. Someone ran `chmod -R 777 /`. What breaks first, and how do you recover?
10. Explain what you'd check before and after a kernel upgrade.

## 53. Command Cheat Sheet

```bash
# --- Triage ---
uptime; free -h; df -hT; df -i; dmesg -T | tail
vmstat 1 5; mpstat -P ALL 1; iostat -xz 1; pidstat 1
systemctl --failed; journalctl -p err -b

# --- Processes ---
ps aux --sort=-%mem | head; ps -ef --forest
pgrep -af nginx; pkill -f 'java.*app'
lsof -p <pid>; lsof -i :80; lsof +L1
strace -c -p <pid>

# --- Storage ---
lsblk -f; blkid; findmnt; mount -a
pvs; vgs; lvs; lvextend -l +100%FREE -r /dev/vg/lv
du -xh --max-depth=1 / | sort -rh | head

# --- Network ---
ip -br a; ip r; ip route get 1.1.1.1
ss -tulpn; ss -tan state established
dig +short example.com; nc -zv host 443
tcpdump -i any -nn 'port 443'
mtr -n 8.8.8.8

# --- Users/Perms ---
id; sudo -l; getfacl f; lsattr f
find / -perm -4000 -type f 2>/dev/null
chage -l user; faillock --user user

# --- SELinux ---
getenforce; ls -Z; ausearch -m AVC -ts recent
restorecon -Rv /path; setsebool -P bool on

# --- Packages ---
rpm -qf /path; dnf provides /path; dnf history undo N
dpkg -S /path; apt-cache policy pkg
```

## 54. Final Checklist

Before an interview, be able to do each of these without notes:

- [ ] Draw the boot sequence and name a failure mode at each stage
- [ ] Write a systemd unit from scratch, including `Type=` and a resource limit
- [ ] Create a VG/LV, mount by UUID, and grow it online
- [ ] Explain why XFS can't shrink and what you'd do instead
- [ ] Read `vmstat`, `iostat -x`, and `free -h` out loud and say what's wrong
- [ ] Explain load average including `D` state
- [ ] Diagnose "df full, du clean"
- [ ] Explain setuid/setgid/sticky on files vs directories
- [ ] Fix an SELinux denial without disabling SELinux
- [ ] Trace a connection failure from L1 to L4 with the right tool at each layer
- [ ] Write a `set -Eeuo pipefail` script with locking and traps
- [ ] Explain namespaces + cgroups as the definition of a container

---

## Appendix A: Jargon Decoder

Every term in this document, in one line of plain English.

| Term | Plain English |
|---|---|
| **kernel** | The core of Linux. The only code that talks to hardware. |
| **userspace** | Every normal program. Must ask the kernel for anything real. |
| **syscall** | A program's request to the kernel ("open this file"). |
| **shell** | The program that reads your typed commands (bash). |
| **process** | A running program. |
| **PID** | A process's ID number. |
| **parent / child** | Every process is started by another. PID 1 is the ancestor of all. |
| **fork** | A process cloning itself. |
| **exec** | A process replacing itself with a different program. |
| **zombie** | A dead process whose parent never collected its exit code. |
| **daemon** | A program that runs in the background forever (e.g. `sshd`). |
| **signal** | A short message to a process ("stop", "reload"). |
| **SIGTERM / SIGKILL** | "Please stop" (can be handled) / "Die now" (cannot). |
| **file descriptor (fd)** | A number a process uses to refer to an open file/socket. |
| **inode** | The index card holding a file's metadata and block locations. |
| **hard link** | A second name for the same inode. |
| **symlink** | A file whose contents are a path to another file. |
| **mount** | Attaching a disk into the directory tree at some folder. |
| **mount point** | The folder a disk is attached at. |
| **filesystem** | The format used to organise files on a disk (ext4, XFS). |
| **block** | The smallest chunk of disk a file occupies. |
| **partition** | A fixed slice of a physical disk. |
| **LVM** | A layer letting you pool disks and resize volumes live. |
| **PV / VG / LV** | Bottle / tank / glass, in LVM terms. |
| **page cache** | Spare RAM holding recently-read file data, for speed. |
| **swap** | Disk space used as slow overflow RAM. |
| **OOM killer** | The kernel killing a process when memory truly runs out. |
| **load average** | Processes running *plus waiting*, averaged over 1/5/15 min. |
| **cgroup** | A box around processes with usage meters and limits. |
| **namespace** | A private view of the system (own PIDs, own network). |
| **container** | Processes in namespaces + cgroups. Not a VM. |
| **systemd** | PID 1. Starts and supervises everything. |
| **unit** | One thing systemd manages. |
| **target** | A systemd milestone, e.g. "ready for logins". |
| **enabled vs started** | Starts at boot vs running right now. **Independent.** |
| **root / UID 0** | The superuser. Bypasses permission checks. |
| **sudo** | Run one command as another user, with an audit trail. |
| **umask** | The permissions *removed* from newly created files. |
| **setuid** | A program that runs as its owner, not as you. |
| **sticky bit** | On a shared dir: only the owner may delete their own files. |
| **ACL** | Extra per-user permissions beyond owner/group/other. |
| **SELinux** | Rules about what a *program type* may do, on top of permissions. |
| **AVC denial** | An SELinux "no", logged in the audit log. |
| **PAM** | The pluggable framework every login goes through. |
| **port** | A numbered door on a machine (22 = SSH). |
| **socket** | One endpoint of a network connection. |
| **TIME_WAIT** | You closed first; harmless waiting period. |
| **CLOSE_WAIT** | They closed, your app never did. **App bug.** |
| **MTU** | Largest packet size a link accepts (usually 1500). |
| **DNS** | Turns names into IP addresses. |
| **default gateway** | The router you send traffic to when you don't know the way. |
| **initramfs** | A tiny temporary root filesystem used during boot. |
| **GRUB** | The bootloader that loads the kernel. |
| **repository** | A server holding installable packages. |
| **idempotent** | Safe to run repeatedly; the 2nd run changes nothing. |

---

## Appendix B: A Learning Path

Do not read this document front to back. Work in this order:

**Week 1 — Ground yourself**
[Part 0](#part-0-start-here--the-mental-model) in full. Then §4 (processes), §25
(permissions), §0.5 (the tree). Practise in a VM: create users, break permissions, fix them.

**Week 2 — The daily job**
§3 systemd, §34 journald, §31/32 packages. Install nginx, break its config, read the logs,
fix it. Write one unit file from scratch.

**Week 3 — Storage & memory**
§8 memory model, §14 LVM, §15 filesystems. Add a virtual disk to your VM, make it a PV, build
a VG/LV, mount it by UUID, then grow it online. Then fill a disk deliberately and use §17 and
[R3](#r3-disk-full) to recover.

**Week 4 — Networking**
§18–23. Break DNS on purpose. Block a port with the firewall and prove it with `tcpdump`.
Walk [R7](#r7-network-unreachable) end to end.

**Week 5 — Performance**
§37–42. Run `stress-ng` to create load, then diagnose it with `vmstat`, `iostat -x` and
`pidstat` *without* knowing in advance what you started.

**Week 6 — Interview drilling**
[Part 13](#part-13-end-to-end-runbooks) runbooks out loud, then
[Part 14](#part-14-interview-preparation). Say the answers aloud — explaining is a separate
skill from knowing.

> **Build a lab.** Two VMs (VirtualBox/KVM/multipass) and a snapshot before each experiment.
> You learn this by breaking things and recovering, not by reading. Everything in Part 13
> can be reproduced deliberately in a VM in under ten minutes.

---

## Further Reading

- **Systems Performance**, Brendan Gregg — the definitive performance text
- **How Linux Works**, Brian Ward
- **The Linux Programming Interface**, Michael Kerrisk — syscall-level reference
- `man 7 signal`, `man 5 systemd.unit`, `man 5 proc`, `man 7 namespaces`
- RHEL documentation (access.redhat.com) — excellent for SELinux, LVM, and HA



