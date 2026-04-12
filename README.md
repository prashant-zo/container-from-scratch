# Container root (minimal userspace)

A **stripped-down Linux root filesystem** for experiments like “container from scratch”: just enough dynamic libraries and binaries to run an interactive shell, list files, mount `procfs`, and inspect processes—without a full distro image.

**Remote:** [github.com/prashant-zo/container-from-scratch](https://github.com/prashant-zo/container-from-scratch)

---

## Contents

| Section | What you’ll find |
|--------|-------------------|
| [At a glance](#at-a-glance) | Purpose and what is (and isn’t) in this tree |
| [Layout](#layout) | Directory map, `lib64` vs `lib/x86_64-linux-gnu` |
| [Included binaries & libraries](#included-binaries--libraries) | What ships in `bin/` and under `lib*` |
| [Usage](#usage) | Mounting `/proc`, `chroot`, Docker notes |
| [Maintaining this root](#maintaining-this-root) | Updating binaries and libraries safely |

---

## At a glance

- **Goal:** Provide a tiny, inspectable root you can bind-mount or `chroot` into while learning namespaces, cgroups, and container runtimes.
- **Not a goal:** Replace Alpine, Debian-in-Docker, or a package manager–managed root. There is no `apt`, no init system, and no broad userland.
- **Architecture:** **x86-64** (glibc). Binaries use the ELF interpreter `/lib64/ld-linux-x86-64.so.2`.

---

## Layout

```
container-root/
├── bin/                              # Utilities (see table below)
├── lib/
│   └── x86_64-linux-gnu/             # Multiarch libs (glibc, util-linux, deps for bin/)
├── lib64/                            # Loader + libs for paths that expect /lib64/…
└── proc/                             # Empty in repo; mount procfs here at runtime
```

| Path | Role |
|------|------|
| `bin/` | Executables you run inside the environment |
| `lib/x86_64-linux-gnu/` | **Primary** shared libraries (Debian/Ubuntu-style multiarch layout). Most `ldd` output points here. |
| `lib64/` | Dynamic linker (`ld-linux-x86-64.so.2`) and copies of common `.so` files for anything that resolves under `/lib64/`. |
| `proc/` | **Populated only at runtime** with `procfs` (see [Mounting `/proc`](#mounting-proc)). |

### Why both `lib64` and `lib/x86_64-linux-gnu`?

On typical amd64 glibc distros, the **program interpreter** is `/lib64/ld-linux-x86-64.so.2`, while **glibc and many dependencies** live under `/lib/x86_64-linux-gnu/`. This tree mirrors that split so dynamically linked binaries resolve without extra configuration. Tools like `/bin/mount` pull in libs (e.g. `libmount`, `libblkid`, `libsmartcols`) that live under `lib/x86_64-linux-gnu/` only—do not assume everything is duplicated in `lib64/`.

---

## Included binaries & libraries

### `bin/`

| Binary | Typical use inside the root |
|--------|-----------------------------|
| `bash` | Interactive shell and scripts |
| `ls` | List files |
| `mkdir` | Create directories (e.g. ensure `/proc` exists before mounting) |
| `mount` | Mount filesystems (e.g. `proc` on `/proc` when permitted) |
| `ps` | Process listing (needs `/proc` mounted) |

### Libraries under `lib/x86_64-linux-gnu/`

Includes the dynamic linker copy, **glibc** (`libc.so.6`), and dependencies for the binaries above—for example `libmount`, `libblkid`, `libsmartcols` (for `mount`), `libproc2` / `libsystemd` (for `ps`), `libtinfo` (for `bash`), compression and crypto libs, etc.

### `lib64/`

Subset aligned with `/lib64/…` resolution: `ld-linux-x86-64.so.2`, `libc.so.6`, and other commonly duplicated `.so` names. **Not** a full mirror of `lib/x86_64-linux-gnu/`; new binaries may require additional copies under `lib/x86_64-linux-gnu/` (and sometimes `lib64/`).

---

## Usage

### Mounting `/proc`

The `proc/` directory in the repo is **empty on purpose**. Process tools (`ps`) and a realistic `/proc` tree need a **procfs** mount.

**1. Prefer mounting from the host (clear paths, easy to undo)**

Set the root once so you never mix up “path inside chroot” vs “path on the host”:

```bash
ROOT="/absolute/path/to/container-root"
sudo mount -t proc proc "${ROOT}/proc"
```

Unmount:

```bash
sudo umount "${ROOT}/proc"
```

**2. Optional: from inside `chroot`**

If you already `chroot`’d in and have enough privilege, ensure the mountpoint exists, then mount:

```bash
/bin/mkdir -p /proc
/bin/mount -t proc proc /proc
```

In practice, `mount` inside a `chroot` often still requires **host** capabilities (e.g. `CAP_SYS_ADMIN`) or a **privileged** container. If that fails, use **host-first** mounting with `"${ROOT}/proc"` as above.

### `chroot` (host must match libc / arch)

From the host (adjust `ROOT`):

```bash
ROOT="/absolute/path/to/container-root"
sudo chroot "$ROOT" /bin/bash
```

If you see loader or libc errors, the host kernel, architecture, or library versions may not match the environment this root was copied from—see [Maintaining this root](#maintaining-this-root).

### Docker (conceptual)

Mount this directory as the container filesystem (or a layer) and ensure `/proc` is mounted inside the container (Docker does this by default for normal containers). Use **linux/amd64** when building or running images that use this root. Exact `Dockerfile` or `docker run` flags depend on whether you mimic a from-scratch image or bind-mount this tree.

---

## Maintaining this root

When you upgrade or add a binary:

1. Copy the new executable into `bin/`.
2. On a **matching Linux amd64** system, run `ldd /path/to/binary` and copy every reported `.so`:
   - Paths under `/lib/x86_64-linux-gnu/` → this repo’s `lib/x86_64-linux-gnu/`.
   - Paths under `/lib64/` → this repo’s `lib64/`.
3. If `ldd` shows a missing library, copy the file from the donor system; preserve the **same relative layout** (`lib/x86_64-linux-gnu/` vs `lib64/`).
4. Re-test with `chroot` or your container. Use `LD_DEBUG=libs` only while debugging.

Keeping **interpreter** (`lib64/ld-linux-x86-64.so.2`) and **glibc** in `lib/x86_64-linux-gnu/` consistent with the donor distro avoids subtle breakage.

---

## Quick reference

| I want to… | Do this |
|------------|---------|
| Mount `/proc` from the host | `ROOT=…; sudo mount -t proc proc "${ROOT}/proc"` |
| Open a shell in the root | `sudo chroot "$ROOT" /bin/bash` (mount `proc` first if you need `ps`) |
| Ensure `/proc` exists in the root | `/bin/mkdir -p /proc` (inside `chroot` or via scripted prep) |
| Add a command | Add binary + **all** `ldd` dependencies under `lib/x86_64-linux-gnu/` and/or `lib64/` as paths dictate |

Author

Prashant