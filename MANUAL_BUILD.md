# Manual build: minimal Linux root (container from Scratch)

This guide walks you through building a **tiny root filesystem**—the same idea as the `container-root` repo: a few binaries under `bin/`, their shared libraries under `lib/x86_64-linux-gnu/` and `lib64/`, and an empty `proc/` you mount at runtime.

---

## Contents

| Section | What you’ll find |
|--------|-------------------|
| [Prerequisites](#prerequisites) | **Linux only** — why macOS/Windows won’t work here |
| [What you are building](#what-you-are-building) | Folder layout and end state |
| [Method A — `jail_copy` helper](#method-a--jail_copy-helper) | Fast path: one function + copy commands |
| [Method B — pure `ldd` + `cp`](#method-b--pure-ldd--cp) | Slow path: full control, no helper |
| [Run inside a mount namespace](#run-inside-a-mount-namespace) | `unshare`, `chroot`, mount `/proc`, verify with `ps` |
| [Troubleshooting](#troubleshooting) | Common errors and fixes |

---

## Prerequisites

### You need a real Linux environment (x86-64)

| Requirement | Why |
|-------------|-----|
| **Linux kernel** | You will use `chroot`, optional `unshare`, and `mount -t proc`. These are Linux features. |
| **x86-64 (amd64)** | The paths and examples match **glibc on Debian/Ubuntu-style amd64** (`/lib/x86_64-linux-gnu`, `/lib64/ld-linux-x86-64.so.2`). |
| **`sudo` (root)** | `chroot`, `unshare`, and mounting `procStep 1: Initialize the FilesystemYou must create the specific sub-folder architecture. Linux programs on modern Ubuntu are hardcoded to look for libraries in x86_64-linux-gnu.bashmkdir -p ~/manual-jail/{bin,lib/x86_64-linux-gnu,lib64,proc}
cd ~/manual-jail
Use code with caution.Step 2: The Manual Dependency LoopFor every tool you want (e.g., bash, ps, ls), follow this 3-part process:A. Copy the Binarybashcp /bin/bash bin/
Use code with caution.B. Check Dependencies with lddRun ldd on the file you just copied:bashldd bin/bash
Use code with caution.You will see output like this:libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x0000...)libtinfo.so.6 => /lib/x86_64-linux-gnu/libtinfo.so.6 (0x0000...)/lib64/ld-linux-x86-64.so.2 (0x0000...)C. Copy the Libraries (The Critical Part)You must mirror the source path exactly.If the path has x86_64-linux-gnu:cp /lib/x86_64-linux-gnu/libc.so.6 lib/x86_64-linux-gnu/If the path is in lib64:cp /lib64/ld-linux-x86-64.so.2 lib64/Step 3: Repeat for ps and mountRepeat Step 2 for /bin/ps and /bin/mount. This is where you solved those "whack-a-mole" errors earlier. You must copy every single line listed by ldd (except the virtual linux-vdso.so.1).Essential for ps and mount on Ubuntu 24.04:bashcp /lib/x86_64-linux-gnu/libproc2.so.0 lib/x86_64-linux-gnu/
cp /lib/x86_64-linux-gnu/libsystemd.so.0 lib/x86_64-linux-gnu/
cp /lib/x86_64-linux-gnu/libblkid.so.1 lib/x86_64-linux-gnu/
cp /lib/x86_64-linux-gnu/libmount.so.1 lib/x86_64-linux-gnu/
cp /lib/x86_64-linux-gnu/libsmartcols.so.1 lib/x86_64-linux-gnu/
Use code with caution.Step 4: Isolation and ActivationNow that the "muscles" are in the right folders, execute the isolation:Launch:bashsudo unshare --fork --pid --mount-proc -m chroot . /bin/bash
Use code with caution.Mount (Plug in the process list):bash# Inside the shell (bash-5.2#)
mount -t proc proc /proc
Use code with caution.Verify:bashps aux
Use code with caution.Step 1: Initialize the FilesystemYou must create the specific sub-folder architecture. Linux programs on modern Ubuntu are hardcoded to look for libraries in x86_64-linux-gnu.bashmkdir -p ~/manual-jail/{bin,lib/x86_64-linux-gnu,lib64,proc}
cd ~/manual-jail
Use code with caution.Step 2: The Manual Dependency LoopFor every tool you want (e.g., bash, ps, ls), follow this 3-part process:A. Copy the Binarybashcp /bin/bash bin/
Use code with caution.B. Check Dependencies with lddRun ldd on the file you just copied:bashldd bin/bash
Use code with caution.You will see output like this:libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x0000...)libtinfo.so.6 => /lib/x86_64-linux-gnu/libtinfo.so.6 (0x0000...)/lib64/ld-linux-x86-64.so.2 (0x0000...)C. Copy the Libraries (The Critical Part)You must mirror the source path exactly.If the path has x86_64-linux-gnu:cp /lib/x86_64-linux-gnu/libc.so.6 lib/x86_64-linux-gnu/If the path is in lib64:cp /lib64/ld-linux-x86-64.so.2 lib64/Step 3: Repeat for ps and mountRepeat Step 2 for /bin/ps and /bin/mount. This is where you solved those "whack-a-mole" errors earlier. You must copy every single line listed by ldd (except the virtual linux-vdso.so.1).Essential for ps and mount on Ubuntu 24.04:bashcp /lib/x86_64-linux-gnu/libproc2.so.0 lib/x86_64-linux-gnu/
cp /lib/x86_64-linux-gnu/libsystemd.so.0 lib/x86_64-linux-gnu/
cp /lib/x86_64-linux-gnu/libblkid.so.1 lib/x86_64-linux-gnu/
cp /lib/x86_64-linux-gnu/libmount.so.1 lib/x86_64-linux-gnu/
cp /lib/x86_64-linux-gnu/libsmartcols.so.1 lib/x86_64-linux-gnu/
Use code with caution.Step 4: Isolation and ActivationNow that the "muscles" are in the right folders, execute the isolation:Launch:bashsudo unshare --fork --pid --mount-proc -m chroot . /bin/bash
Use code with caution.Mount (Plug in the process list):bash# Inside the shell (bash-5.2#)
mount -t proc proc /proc
Use code with caution.Verify:bashps aux
Use code with caution.` require elevated privileges on typical setups. |
| **`bash`**, **`cp`**, **`mkdir`**, **`ldd`** | Standard on glibc-based distros. |

### macOS and Windows do **not** work for this guide (as written)

- **macOS** uses a different kernel, dynamic linker, and library layout. There is no Linux `ldd`/`glibc` path tree like `/lib/x86_64-linux-gnu` for your host binaries, and Linux `chroot`/`unshare` semantics do not apply to macOS userland the same way.
- **Windows** is not a Linux userspace; `ldd`, ELF glibc layouts, and `chroot` here assume Linux.

**If you are on a Mac or Windows PC:** use **Linux in a VM**, **WSL2** (Windows), a **cloud Linux box**, or a **Docker Linux container with sufficient privileges** to follow the steps—not the host OS itself.

Examples that *do* work:

- Ubuntu 22.04 / 24.04 (desktop or server), **amd64**
- Debian **amd64**
- Fedora/RHEL **x86_64** (paths may differ slightly; always trust **your** `ldd` output)

---

## What you are building

Target layout (names match this repository):

```text
~/mini-container/          # or any directory you choose
├── bin/
├── lib/
│   └── x86_64-linux-gnu/
├── lib64/
└── proc/                  # stays empty until you mount procfs
```

By the end you will have copied **binaries** into `bin/` and **every shared library** `ldd` reports into the correct `lib*` subtree, then you will **enter** that root with `chroot` (inside an optional mount namespace) and **mount `proc`** so `ps` works.

---

## Method A — `jail_copy` helper

Best when you want speed and less typing. Run all commands **on Linux amd64**.

### Step 1 — Create the skeleton

```bash
mkdir -p ~/mini-container/{bin,lib/x86_64-linux-gnu,lib64,proc}
cd ~/mini-container
```

### Step 2 — Define `jail_copy`

Paste this **once** in the same shell (or save it in a file and `source` it). It copies a command from your **host** `PATH` into `bin/` and copies each resolved `.so` path into `lib64/` or `lib/x86_64-linux-gnu/` based on the path string.

```bash
jail_copy() {
    local cmd_path
    cmd_path=$(command -v "$1") || { echo "not found: $1" >&2; return 1; }
    cp -v "$cmd_path" bin/
    local lib
    while read -r lib; do
        [[ -z "$lib" || ! -f "$lib" ]] && continue
        case "$lib" in
            *linux-vdso*) ;;
            */lib64/*)    cp -nv "$lib" lib64/ ;;
            *)            cp -nv "$lib" lib/x86_64-linux-gnu/ ;;
        esac
    done < <(ldd "$cmd_path" 2>/dev/null | awk '/=>/ {print $3} /^\// {print $1}')
}
```

Notes:

- **`cp -n`** avoids overwriting if you run the function twice; remove `-n` if you intentionally want to refresh libraries.
- If your distro puts some libraries only under `/lib/` (no `lib64` in the path), they land in `lib/x86_64-linux-gnu/` here—adjust if your `ldd` layout differs.

### Step 3 — Copy the tools you need

Minimum set used in this project’s workflow:

```bash
jail_copy bash
jail_copy ls
jail_copy mkdir
jail_copy ps
jail_copy mount
```

If `jail_copy` skips a line or `chroot` later complains about a missing `.so`, use [Method B](#method-b--pure-ldd--cp) for that binary and copy the missing path by hand.

---

## Method B — pure `ldd` + `cp`

Best when you want to see every dependency or your distro’s paths are unusual.

### Step 1 — Create the skeleton

```bash
mkdir -p ~/manual-jail/{bin,lib/x86_64-linux-gnu,lib64,proc}
cd ~/manual-jail
```

### Step 2 — For each program: copy binary, then every library

Repeat this **per binary** (e.g. `bash`, `ls`, `mkdir`, `ps`, `mount`).

**A. Copy the binary**

```bash
cp -v /bin/bash bin/
```

(Use the real path on your system: `command -v bash`, etc.)

**B. List dependencies**

```bash
ldd bin/bash
```

Example shape of output (paths vary by distro):

```text
linux-vdso.so.1 (0x...)
libtinfo.so.6 => /lib/x86_64-linux-gnu/libtinfo.so.6 (0x...)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x...)
/lib64/ld-linux-x86-64.so.2 (0x...)
```

**C. Copy each real file**

- Ignore **`linux-vdso.so.1`** (virtual; do not copy).
- If `ldd` shows **`=> /some/path/libfoo.so`**, copy **`/some/path/libfoo.so`**.
- If the line is only **`/lib64/ld-linux-x86-64.so.2`**, copy that full path into your `lib64/` directory.

Mirror the **directory shape** under your jail:

| Host path contains | Copy into your jail |
|--------------------|---------------------|
| `.../lib/x86_64-linux-gnu/...` | `lib/x86_64-linux-gnu/` |
| `.../lib64/...` | `lib64/` |

Examples:

```bash
cp -v /lib/x86_64-linux-gnu/libc.so.6       lib/x86_64-linux-gnu/
cp -v /lib64/ld-linux-x86-64.so.2           lib64/
```

**D. Repeat for other binaries**

Do the same for `ls`, `mkdir`, `ps`, and `mount`. **`mount`** and **`ps`** pull in extra libraries; you must copy **every** resolved path `ldd` prints (except `linux-vdso`).

On **Ubuntu 24.04 amd64**, you often end up copying extras such as (names only—always take paths from **your** `ldd`):

```bash
cp -v /lib/x86_64-linux-gnu/libproc2.so.0      lib/x86_64-linux-gnu/
cp -v /lib/x86_64-linux-gnu/libsystemd.so.0    lib/x86_64-linux-gnu/
cp -v /lib/x86_64-linux-gnu/libblkid.so.1      lib/x86_64-linux-gnu/
cp -v /lib/x86_64-linux-gnu/libmount.so.1      lib/x86_64-linux-gnu/
cp -v /lib/x86_64-linux-gnu/libsmartcols.so.1    lib/x86_64-linux-gnu/
```

---

## Run inside a mount namespace

From the **root of your jail** (where `bin/`, `lib/`, `proc/` live), still on **Linux**:

```bash
cd ~/mini-container   # or ~/manual-jail
sudo unshare --fork --pid --mount-proc -m chroot . /bin/bash
```

What this roughly does:

- **`--mount-proc`** — makes `/proc` meaningful in the new PID namespace.
- **`-m`** — new mount namespace so you do not alter the host’s mount table the same way a plain global mount would.
- **`chroot .`** — use the current directory as `/` inside the jail.

### Inside the jail: mount `proc` and verify

Ensure the mount point exists, then mount:

```bash
/bin/mkdir -p /proc
/bin/mount -t proc proc /proc
/bin/ps aux
```

You typically want **PID 1** in that view to be your shell (or the process you expect) in this toy setup—if `ps` errors, `/proc` is not mounted or libraries are incomplete.

Exit the chroot with `exit` or Ctrl-D.

---

## Troubleshooting

| Symptom | Likely cause | What to do |
|--------|----------------|------------|
| `chroot: failed to run command ‘/bin/bash’: No such file or directory` | Often **missing dynamic linker** or wrong arch | Ensure `lib64/ld-linux-x86-64.so.2` exists; binary is **amd64** ELF. |
| `error while loading shared libraries: libfoo.so` | Incomplete `ldd` copy | Re-run `ldd` on the binary inside the jail path (`ldd bin/bash` on **host** still, pointing at the copied file) and copy the missing `.so`. |
| `mount: … /proc: mount point does not exist` | No mountpoint | Run `/bin/mkdir -p /proc` first. |
| `mount: only root can …` | Permissions | Run the outer `unshare`/`chroot` under `sudo`; capabilities differ across setups. |
| Following this on **macOS** or **Windows** without Linux | Wrong OS for this guide | Use **WSL2**, a **Linux VM**, or a remote **Linux** machine. |

---

## See also

- [README.md](README.md) — how this repository’s layout maps to runtime mounts and maintenance tips.

---

Author 

*Prashant*


