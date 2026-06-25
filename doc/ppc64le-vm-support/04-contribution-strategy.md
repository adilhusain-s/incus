# Contribution Strategy — Upstream PR Plan

This document outlines the recommended PR order, checklists, and upstream strategy.

**Belongs to:** [docs/ppc64le-vm-support/](index.md)

---


## Appendix: Disk Layout of Built Image

| Device | Partition | Type | Filesystem | Size | Label | Content |
|---|---|---|---|---|---|---|
| `/dev/sda` | - | GPT | - | 4 GiB | - | Protective MBR + GPT header |
| `/dev/sda1` | 1 | EF00 (ESP) | vfat | 100 MiB | "UEFI" | Empty (no boot files) |
| `/dev/sda2` | 2 | 8300 (Linux) | ext4 | 3.9 GiB | - | Root filesystem with Ubuntu |

**Root filesystem UUID:** `b58372d2-7328-4e98-a123-d3fc4150b694`

**GRUB platform:** `powerpc-ieee1275` (228 modules installed at `/usr/lib/grub/powerpc-ieee1275/`)

**GRUB config:** Existing at `/boot/grub/grub.cfg` with valid kernel/initrd entries and correct root UUID.

---

## Appendix B: Comparison with Known-Good Ubuntu ppc64le Cloud Image

After the boot test confirmed Outcome B, a known-good Ubuntu 24.04 LTS (Noble) ppc64le cloud image was downloaded from `cloud-images.ubuntu.com` and compared against the distrobuilder-built image. This comparison revealed the exact partition layout and bootloader setup used by working ppc64le images.

### Source
```bash
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-ppc64el.img
```

### Partition Layout Comparison

| Aspect | Official Cloud Image | Distrobuilder Image | Analysis |
|---|---|---|---|
| **Partitions** | 2 | 2 | Same count |
| **Partition 1** | Type: Linux filesystem (`0FC63DAF-...`) | Type: EFI System (`C12A7328-...`) | **Different!** |
| **Partition 1 FS** | ext4 (root filesystem) | vfat (ESP) | **Different!** |
| **Partition 1 Size** | ~2.2 GiB | 100 MiB | Different |
| **Partition 2** | Type: PReP (`9E1A2D38-...`) | Type: Linux filesystem (`0FC63DAF-...`) | **Different!** |
| **Partition 2 FS** | No filesystem (raw ELF boot image) | ext4 (root) | **Different!** |
| **Partition 2 Size** | 4 MiB | 3.9 GiB | Different |

### Critical Difference: PReP Partition

The official cloud image uses a **PReP (PowerPC Reference Platform) boot partition** (GUID `9E1A2D38-C612-4316-AA26-8B49521E5A8B`) instead of an EFI System Partition. The PReP partition:

| Property | Value |
|---|---|
| **Size** | 4 MiB (8388608 bytes) |
| **Partition number** | 2 (before the root partition) |
| **GUID type** | `9E1A2D38-C612-4316-AA26-8B49521E5A8B` (PReP) |
| **Filesystem** | None — raw data |
| **Content** | GRUB core image (ELF binary, starts with `\x7fELF`) |

```
$ hexdump -C /dev/sda2 | head -3
00000000  7f 45 4c 46 01 02 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010  00 02 00 14 00 00 00 01  00 20 00 00 00 00 00 34  |......... .....4|
00000020  00 00 00 94 00 00 00 00  00 34 00 20 00 03 00 28  |.........4. ...(|
```

The PReP partition contains the **GRUB PowerPC IEEE1275 core image** (`core.elf`), installed there by:
```bash
grub-install --target=powerpc-ieee1275 /dev/sda
```
This writes the boot ELF to the PReP partition. SLOF's firmware then finds this partition via the GPT entry and loads the ELF binary.

### GRUB Directory Comparison

| File/Dir | Cloud Image | Distrobuilder Image |
|---|---|---|
| `/boot/grub/grub.cfg` | ✅ Present | ✅ Present |
| `/boot/grub/grub` | ✅ **97 KB ELF (core image)** | ❌ **Missing** |
| `/boot/grub/grubenv` | ✅ Present | ❌ Missing |
| `/boot/grub/fonts/` | ✅ Present | ❌ Missing |
| `/boot/grub/locale/` | ✅ Present | ❌ Missing |
| `/boot/grub/powerpc-ieee1275/` | ✅ Present | ❌ **Missing** |
| `/usr/lib/grub/powerpc-ieee1275/` | ✅ Present | ✅ Present |

The GRUB modules are installed system-wide in `/usr/lib/grub/powerpc-ieee1275/` in both images, but the cloud image also copies them into `/boot/grub/powerpc-ieee1275/` (done by `grub-install`). The critical missing piece is `/boot/grub/grub` — the core image ELF binary that SLOF loads from the PReP partition.

### Root Cause Confirmed

The reason `grub-install --target=powerpc-ieee1275` fails inside the distrobuilder chroot is twofold:

1. **No PReP partition exists** — `grub-install` needs a PReP partition (type `0x41` / GUID `9E1A2D38-...`) to write the core ELF image. The current distrobuilder `createPartitions()` creates an ESP partition (type `EF00`) instead.

2. **The device path is not discoverable** — Even if a PReP partition existed, `grub-install` needs the block device path (e.g., `/dev/loop11`), which is not exposed as an environment variable inside the chroot.

### The Fix

Two changes are needed:

#### Fix A: Distrobuilder — Add PReP partition for ppc64le/s390x
In `distrobuilder/vm.go`, the partition layout needs to be architecture-aware:

```go
// Current (works for x86_64 and aarch64 only):
args = [][]string{
    {"--zap-all"},
    {"--new=1::+100M", "-t 1:EF00"},           // ESP
    {"--new=2::", "-t 2:8300"},                  // root
}

// Needed for ppc64le:
args = [][]string{
    {"--new=1::+4M", "-t 1:4100"},              // PReP (small, no fs)
    {"--new=2::", "-t 2:8300"},                  // root
}
```

The PReP partition (type code `0x41` or GUID `9E1A2D38-C612-4316-AA26-8B49521E5A8B`) is only 4 MiB and does not receive a filesystem. `grub-install` writes the ELF core image directly to it.

#### Fix B: Distrobuilder — Export `DISTROBUILDER_ROOT_DEVICE`
In the code that enters the build chroot, set:

```go
os.Setenv("DISTROBUILDER_ROOT_DEVICE", vm.loopDevice)  // e.g., "/dev/loop11"
```

This allows the YAML `post-files` action to use:
```bash
grub-install --target=powerpc-ieee1275 "${DISTROBUILDER_ROOT_DEVICE}"
```

Without this env var, any device-discovery hack in the YAML is fragile (as demonstrated by the `|| true` workaround).

#### Fix C: YAML post-files action (lxc-ci)
With Fix A and Fix B in place, the YAML action becomes straightforward:

```yaml
- trigger: post-files
  action: |-
    #!/bin/sh
    set -eux
    grub-install --target=powerpc-ieee1275 "${DISTROBUILDER_ROOT_DEVICE}"
    grub-mkconfig -o /boot/grub/grub.cfg
    sed -i "s#root=[^ ]*#root=${DISTROBUILDER_ROOT_UUID}#g" /boot/grub/grub.cfg
  architectures:
    - ppc64el
  types:
    - vm
```

### Updated PR Strategy

| PR | Repo | What | Priority |
|---|---|---|---|
| **PR 1** | **distrobuilder** | Add architecture-aware partition layout (PReP for ppc64le, keep ESP for x86_64/aarch64) | **Highest** — blocks everything else |
| **PR 2** | **distrobuilder** | Export `DISTROBUILDER_ROOT_DEVICE` env var in chroot | **High** — simplifies image definitions |
| **PR 3** | **lxc-ci** | Add ppc64le GRUB packages and post-files action to all distro YAMLs | **High** — enables the actual image builds |
| **PR 4** | **lxc-ci** | Same for s390x (zipl bootloader) | Lower — needs hardware |

The investigation has narrowed the problem from the broad question of "ppc64le VM support" to the specific question:

> **"How should the partition layout in distrobuilder be modified to include a PReP partition for ppc64le, and how should `DISTROBUILDER_ROOT_DEVICE` be exported so that `grub-install --target=powerpc-ieee1275` can find the target disk?"**

---

*End of Part 2. Full experimental validation including cloud image comparison now complete.*

---

## Part 3: Practical Build & Install Guide — Step by Step
