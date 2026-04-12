# Container root (minimal userspace)

A **stripped-down Linux root filesystem** for experiments like “container from scratch”: just enough dynamic libraries and binaries to run an interactive shell and inspect processes—without a full distro image.

**Remote:** [github.com/prashant-zo/container-from-scratch](https://github.com/prashant-zo/container-from-scratch)

---

## Contents

| Section | What you’ll find |
|--------|-------------------|
| [At a glance](#at-a-glance) | Purpose and what is (and isn’t) in this tree |
| [Layout](#layout) | Directory map and what each part is for |
| [Included binaries & libraries](#included-binaries--libraries) | What ships in `bin/` and `lib*` |
| [Usage](#usage) | Docker, `chroot`, and `proc` notes |
| [Maintaining this root](#maintaining-this-root) | Updating copies of libs and tools safely |

---

## At a glance

- **Goal:** Provide a tiny, inspectable root you can bind-mount or `chroot` into while learning namespaces, cgroups, and container runtimes.
- **Not a goal:** Replace Alpine, Debian-in-Docker, or a package manager–managed root. There is no `apt`, no init system, and no broad userland.
- **Architecture:** Paths and libraries here target **x86-64** (glibc, `ld-linux-x86-64.so.2`).

---

## Layout

```
container-root/
├── bin/          # Static-ish set of utilities (bash, ls, ps, …)
├── lib/          # 64-bit shared libraries (glibc, deps for bin/)
├── lib64/        # Same ABI; duplicate path for loaders/tools that expect /lib64
└── proc/         # Empty placeholder; mount procfs here at runtime (see Usage)
```

| Path | Role |
|------|------|
| `bin/` | Executables you run inside the environment |
| `lib/`, `lib64/` | Dynamic linker + libc + transitive deps for those binaries |
| `proc/` | **Must be populated at runtime** with `procfs` (see below) |

---

## Included binaries & libraries

### `bin/`

| Binary | Typical use inside the root |
|--------|-----------------------------|
| `bash` | Interactive shell and scripts |
| `ls` | List files |
| `ps` | Process listing (expects `/proc`; mount it first) |

### `lib/` and `lib64/`

Shared objects include the **dynamic linker**, **glibc**, and dependencies pulled in by the above tools (e.g. terminal, compression, crypto, systemd-related libs for `ps`, etc.). Both `lib` and `lib64` are present so paths match common 64-bit layout expectations.

---

## Usage

### 1. Mount `proc` (required for `ps` and realistic behavior)

The `proc/` directory in the repo is intentionally empty. Before relying on `/proc`, mount procfs:

```bash
sudo mount -t proc proc /path/to/container-root/proc
```

Unmount when finished:

```bash
sudo umount /path/to/container-root/proc
```

### 2. Try with `chroot` (host must match libc/arch expectations)

From the **parent** of this root (adjust paths):

```bash
sudo chroot /path/to/container-root /bin/bash
```

If you see loader or libc errors, the host kernel/arch or library versions may not match what this root was built from—see [Maintaining this root](#maintaining-this-root).

### 3. Docker (conceptual)

Mount this directory as the container filesystem (or a layer) and ensure `/proc` is mounted inside the container (Docker does this by default for normal containers). Exact `Dockerfile` or `docker run` flags depend on whether you are mimicking a from-scratch image or using this as a bind-mounted root; keep **architecture = amd64** in mind when choosing base images and CI.

---

## Maintaining this root

When you upgrade or add a binary:

1. Copy the new executable into `bin/`.
2. Use `ldd` **on a matching system** to list required `.so` files and copy them into `lib/` and `lib64/` as appropriate.
3. Re-test `chroot` or your container with `LD_DEBUG=libs` only when debugging—do not leave debug noise in production docs.

Keeping `lib` and `lib64` in sync (same SONAMEs where duplicated) avoids subtle path bugs.

---

## Quick reference

| I want to… | Do this |
|------------|---------|
| Open a shell in the root | `chroot` + `/bin/bash` (after proc mount if needed) |
| See processes | Mount `proc`, then run `/bin/ps` |
| Add a command | Add binary + all `ldd` dependencies under `lib*` |


