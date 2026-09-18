---
tags: [linux, kernel, filesystems, vfs, storage]
---

# Linux Filesystems

Deep dive into the Virtual File System layer, on-disk filesystems, and the "everything is a file" model. Part of the [[How Linux Works]] series.

---

## 1. The Virtual File System (VFS)

The VFS is an abstraction layer inside the kernel that lets `open()`, `read()`, `write()`, `close()`, `stat()`, etc. work identically whether the underlying storage is ext4, XFS, Btrfs, an NFS share, a FAT USB stick, or something with no disk at all (`/proc`, tmpfs). Every concrete filesystem driver implements a common set of operations that the VFS calls into.

Four core VFS objects:

| Object | Represents |
|---|---|
| **superblock** | One mounted filesystem instance — its type, size, state, and operations table |
| **inode** | One file or directory's metadata: permissions, owner, size, timestamps, and pointers to its data blocks. **Not** its name. |
| **dentry** (directory entry) | A cached mapping from a path component (a name) to an inode — the **dentry cache** is why repeatedly resolving `/usr/lib/x86_64-linux-gnu/libc.so.6` is fast after the first time |
| **file** | The state of one *open* file: current offset, access mode flags. This is what a **file descriptor** in a process actually refers to |

A single inode can have **multiple dentries** pointing at it — that's exactly what a **hard link** is (two names, one inode, one set of data). A **symbolic link**, by contrast, is its own inode whose content is just a path string that the kernel re-resolves on access — it can cross filesystems and point at nonexistent targets, unlike a hard link.

```bash
stat file.txt          # see the inode's metadata directly
ls -i file.txt          # show inode number
ln file.txt hard.txt    # hard link — same inode
ln -s file.txt soft.txt # symlink — new inode containing a path
```

---

## 2. "Everything Is a File"

A cornerstone of Unix/Linux design: as much as possible is exposed through the same file-descriptor-based interface (`open`/`read`/`write`/`ioctl`/`close`), even when there's no literal disk data behind it:

- **Devices** — `/dev/sda` (a disk), `/dev/null` (discards writes, EOF on read), `/dev/urandom` (random bytes), `/dev/tty` (your terminal)
- **Pipes** — anonymous (`|` in a shell) or named (FIFOs, created with `mkfifo`)
- **Sockets** — network and Unix-domain, opened via `socket()` but usable with `read`/`write` once connected
- **`/proc`** — a synthetic filesystem exposing live kernel and per-process state (see below)
- **`/sys`** — exposes the kernel's device/driver model

This uniformity is why tools like `cat`, `dd`, and shell redirection (`>`, `<`, `|`) work on almost anything without needing to know what's on the other end.

---

## 3. On-Disk Filesystem Types

| Filesystem | Notes |
|---|---|
| **ext4** | The long-standing default on most distros. Journaling (metadata, optionally data), extents for large files, backward-compatible with ext2/ext3. Mature and predictable. |
| **XFS** | Default on RHEL/Fedora. Excellent for large files and high parallel I/O; harder to shrink than ext4. |
| **Btrfs** | Copy-on-write, built-in snapshots, checksums on data and metadata, native multi-device volumes/RAID. Default on openSUSE, used by Fedora for the root filesystem in recent releases. |
| **ZFS** | Not upstream (licensing), but widely used via out-of-tree modules — checksums, snapshots, native RAID-Z, extremely mature, popular for storage servers. |
| **FAT32 / exFAT** | Used for interoperability (EFI System Partition, USB drives shared with Windows/macOS) — no permissions/ownership metadata, no journaling. |
| **NTFS** | Read/write support via `ntfs3` (in-kernel, modern) or the older FUSE-based `ntfs-3g`, for dual-boot/interop. |

**Journaling** filesystems (ext4, XFS, Btrfs) write a log of pending metadata (and optionally data) changes before applying them, so a crash mid-write can be recovered by replaying or discarding the incomplete journal entry — avoiding the lengthy full-filesystem `fsck` that older, non-journaled filesystems required after an unclean shutdown.

---

## 4. Mounting

A filesystem must be **mounted** — attached at a directory (the **mount point**) in the existing tree — before its contents are visible. Linux has a single unified tree (unlike Windows' drive letters); a second disk just becomes visible as, e.g., `/mnt/data` or `/home` if that's a separate partition.

```bash
mount /dev/sdb1 /mnt/data     # mount a partition
mount -t tmpfs tmpfs /tmp     # mount a RAM-backed filesystem
umount /mnt/data              # unmount
findmnt                       # show the full mount tree
cat /proc/mounts              # same info, from the kernel's perspective
```

`/etc/fstab` declares filesystems to mount automatically at boot (device/UUID, mount point, type, options, dump/fsck flags).

### Bind mounts and mount namespaces
A **bind mount** (`mount --bind /a /b`) makes the same directory subtree appear at a second location — no new filesystem, just an extra path to the same inodes. This, combined with **mount namespaces** (see [[Linux Process Management]] §5), is how containers get their own root filesystem view (`chroot`-like, but properly isolated) and how tools like `overlayfs`-based container images stack read-only layers with a writable one on top.

---

## 5. Special/Virtual Filesystems

### `/proc`
A window into live kernel and process state, generated on the fly (nothing here is stored on disk):
- `/proc/<pid>/status`, `/stat`, `/maps`, `/fd/` — per-process info (see [[Linux Process Management]], [[Linux Memory Management]])
- `/proc/cpuinfo`, `/proc/meminfo`, `/proc/loadavg` — system-wide stats
- `/proc/sys/` — the *same* tree that `sysctl` reads/writes for kernel tunables (e.g. `/proc/sys/vm/swappiness` is `vm.swappiness`)

### `/sys`
Exposes the kernel's internal **device model** — buses, devices, drivers — as a browsable tree, primarily for udev and driver introspection/configuration (e.g. `/sys/class/net/eth0/`, `/sys/block/sda/queue/scheduler`).

### tmpfs
A filesystem backed entirely by RAM (and swap, if needed) rather than persistent disk. Used for `/tmp` on many distros and `/dev/shm` for POSIX shared memory — fast, but contents vanish on reboot.

### overlayfs
A **union filesystem** that transparently layers a writable directory on top of one or more read-only ones, presenting a single merged view. Reads fall through to the lowest layer that has the file; writes are copy-up'd into the top writable layer. This is the mechanism behind Docker/container image layers — each image layer is a read-only overlay directory, and a container's writable filesystem is one more layer on top.

---

## 6. Permissions

Every inode carries a UID/GID owner and a permission bitmask (`rwx` for owner/group/other), inspectable and settable via `ls -l`, `chmod`, `chown`. Beyond the classic bits:

- **setuid/setgid** — a setuid executable runs with its *owner's* privileges rather than the caller's (classic example: `passwd`, which needs root to edit `/etc/shadow`)
- **sticky bit** — on a directory (e.g. `/tmp`), only a file's owner (or root) can delete/rename it, even if others have write access to the directory
- **ACLs** (`setfacl`/`getfacl`) — finer-grained permissions beyond owner/group/other, for filesystems that support extended attributes
- **capabilities** — split root's traditionally all-or-nothing power into fine-grained privileges (`CAP_NET_BIND_SERVICE`, `CAP_SYS_ADMIN`, ...) that can be granted to a binary or process without full root

---

## Related notes
- [[How Linux Works]]
- [[Linux Process Management]]
- [[Linux Memory Management]]
- [[Linux Networking Stack]]
