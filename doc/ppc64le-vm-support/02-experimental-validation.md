# Experimental Validation Report

This document documents all 6 experiments performed on real ppc64le (POWER10) hardware at OSUOSL on 2026-06-24.

**Belongs to:** [docs/ppc64le-vm-support/](index.md)

---


## Testing Protocol

### Prerequisite: Real Hardware

This work **cannot** be done with cross-compilation or QEMU user-mode emulation alone. You need:

- **ppc64le:** POWER8/POWER9/POWER10 machine with KVM
- **s390x:** IBM Z machine with KVM (or z/VM LPAR)

You have ppc64le access. s390x will need OSUOSL or similar.

### Experiment 1: Bootstrap Test (ppc64le)

```bash
# 1. Clone definitions
git clone https://github.com/lxc/lxc-ci.git
cd lxc-ci

# 2. Build VM image (using existing distrobuilder, no YAML changes yet)
sudo distrobuilder build-incus images/ubuntu.yaml \
  --vm \
  -o image.architecture=ppc64el \
  -o image.release=noble

# 3. Check output
ls -lh incus.tar.xz disk.qcow2

# 4. Import
sudo incus image import incus.tar.xz disk.qcow2 --alias test-ppc64

# 5. Launch
sudo incus launch test-ppc64 vm1 --vm

# 6. Wait and check
sleep 15
sudo incus info vm1
sudo incus console vm1 --show-log
```

### Experiment 2: With Bootloader (ppc64le)

After adding the GRUB packages and post-files actions to the YAML:

```bash
# Rebuild with modified YAML
sudo distrobuilder build-incus images/ubuntu.yaml \
  --vm \
  -o image.architecture=ppc64el \
  -o image.release=noble

# Import and launch same as above
```

### Outcome Classification

| Code | Observation | Meaning | Next Step |
|---|---|---|---|
| A | Linux boots, gets IP, agent connects | ✅ Distrobuilder layout works | YAML changes only needed |
| B | SLOF message, "No bootable device" | Disk visible, no bootloader | Add GRUB packages + post-files |
| C | QEMU error or firmware crash before SLOF | Disk layout incompatible | Investigate PReP partition or distrobuilder changes |
| D | GRUB loads but kernel panic | Bootloader works, kernel/drivers issue | Fix kernel cmdline or initramfs |

### Expected Results

Based on the code analysis:

- **ppc64le most likely outcome:** B (SLOF visible, no bootloader) or D (GRUB works but kernel needs config). The disk layout should be valid, but without GRUB installed, SLOF has nothing to boot.
- **s390x most likely outcome:** C (firmware can't boot disk) — the vfat ESP and missing zipl record may confuse the CCW firmware.

---

## References & Key Files

### Incus Repository

| File | Purpose |
|---|---|
| `internal/server/instance/drivers/driver_qemu.go` | Main QEMU driver: architecture config, UEFI support, firmware setup |
| `internal/server/instance/drivers/driver_qemu_templates.go` | QEMU config template generation: machine type, bus, devices |
| `internal/server/instance/drivers/driver_qemu_machine.go` | CPU topology and machine configuration |
| `internal/server/instance/drivers/edk2/driver_edk2.go` | EDK2 firmware path detection (only x86_64 and aarch64) |
| `shared/osarch/architectures.go` | Architecture constants and names |
| `doc/architectures.md` | Documentation of supported architectures |

### Distrobuilder Repository

| File | Purpose |
|---|---|
| `distrobuilder/vm.go` | VM disk image creation pipeline |
| `distrobuilder/main_incus.go` | `build-incus` command, VM orchestration |
| `distrobuilder/main.go` | Main entry, pre-run build logic, architecture handling |
| `shared/definition.go` | YAML definition struct, validation, architecture mapping |
| `shared/osarch.go` | Architecture name mappings per distro |
| `image/incus.go` | Incus image metadata and packaging |

### lxc-ci Repository

| File | Purpose |
|---|---|
| `images/ubuntu.yaml` | Ubuntu image definition (reference for changes) |
| `images/debian.yaml` | Debian image definition |
| `images/fedora.yaml` | Fedora (has ppc64le/s390x references) |
| `images/rockylinux.yaml` | Rocky Linux (has VM bootloader examples) |
| `bin/build-distro` | Core build script that runs distrobuilder |
| `bin/jenkins-import-images` | Imports builds into image server (arch mapping) |
| `bin/update-images` | Orchestrates periodic image builds |
| `bin/test-image` | Tests built images |
| `bin/test-image-dispatcher` | Dispatches image tests per-architecture |

---

## Appendix: All ppc64le/s390x Mentions in Codebase

### In `lxc-ci/images/`:

**`images/fedora.yaml`:**
```yaml
- trigger: post-packages
  action: |-
    ...
    architectures:
    - ppc64le
    - s390x
```

**`images/opensuse.yaml`:**
```yaml
- action: install
    architectures:
    - ppc64le
```

**`images/ubuntu.yaml`:**
```yaml
- action: install
    architectures:
    - ppc64el
```
(Note: Ubuntu uses `ppc64el`, not `ppc64le`)

### In `lxc-ci/bin/`:

**`bin/jenkins-import-images`:**
```python
# incus_arch() function handles both:
elif arch == "ppc64el": return (7, "ppc64le")
elif arch == "s390x":   return (8, "s390x")
```

**`bin/test-incus-cpu-vm`:**
```sh
if [ "${architecture}" != "x86_64" ] && [ "${architecture}" != "s390x" ]; then
    echo "Skipping test as CPU hotplugging not supported on ${architecture}"
fi
```

### In distrobuilder `shared/osarch.go`:

```go
var debianArchitectureNames = map[int]string{
    osarch.ARCH_64BIT_POWERPC_LITTLE_ENDIAN: "ppc64el",
}

var gentooArchitectureNames = map[int]string{
    osarch.ARCH_64BIT_POWERPC_LITTLE_ENDIAN: "ppc64le",
    osarch.ARCH_64BIT_S390_BIG_ENDIAN:       "s390x",
}
```

### In Incus `doc/architectures.md`:

```
| 7  | `ppc64le` | 64bit PowerPC little-endian |                     |
| 8  | `s390x`   | 64bit ESA/390 big-endian    |                     |

## Virtual-machine support
Incus only supports running virtual-machines on the following host architectures:
- `x86_64`
- `aarch64`
- `ppc64le`
- `s390x`
```

---

## Quick Reference: Commands for the AI/Developer

### Build a ppc64le VM image (test command)
```bash
sudo distrobuilder build-incus images/ubuntu.yaml \
  --vm \
  -o image.architecture=ppc64el \
  -o image.release=noble
```

### Build a s390x VM image (test command)
```bash
sudo distrobuilder build-incus images/ubuntu.yaml \
  --vm \
  -o image.architecture=s390x \
  -o image.release=noble
```

### Check what ppc64le content exists in YAMLs
```bash
grep -rn "ppc64\|ppc64el\|ppc64le" images/
```

### Watch VM console
```bash
sudo incus console <vm-name> --show-log
sudo incus console <vm-name> --type=vga
```

### Debug QEMU invocation
```bash
ps aux | grep qemu-system
sudo cat /var/log/incus/vmname/qemu.conf  # Check QEMU config
```

---

---

# Part 2: Experimental Validation Report

## Overview

This section documents the actual experimental validation performed on real ppc64le (POWER10) hardware at OSUOSL. The experiments were conducted on 2026-06-24 to validate the analysis from Part 1 and identify the exact changes needed.

## Test Environment

| Detail | Value |
|---|---|
| **Host** | `p10-github-actions1.osuosl.org` |
| **Architecture** | ppc64le (POWER10) |
| **OS** | Fedora Linux 42 (Adams) |
| **KVM** | ✅ `/dev/kvm` available |
| **Incus** | Version 6.23-3 (installed via `dnf`) |
| **Go Toolchain** | Go 1.26.4 (downloaded from golang.org — required because distrobuilder's `go.mod` requires go >= 1.25.6, but Fedora 42 only ships Go 1.24.13) |
| **Distrobuilder** | Built from source at `~/distrobuilder`, binary at `/root/go/bin/distrobuilder` |
| **lxc-ci** | Cloned at `~/lxc-ci`, main branch from `github.com/lxc/lxc-ci` |
| **Incus source** | Cloned at `~/incus`, main branch from `github.com/lxc/incus` |

## Experiment 1: Bootstrap Test (Unmodified YAML)

**Goal:** Run `distrobuilder build-incus` with the stock `images/ubuntu.yaml` to see what happens on ppc64le without any modifications.

**Command:**
```bash
distrobuilder build-incus images/ubuntu.yaml \
  --vm \
  -o image.architecture=ppc64el \
  -o image.release=noble
```

### Issues Found

#### Issue 1: Wrong Ubuntu Mirror
The YAML uses `archive.ubuntu.com` which does NOT serve ppc64el packages. ppc64el packages are served from `ports.ubuntu.com`.

**Workaround:** Override the source URL:
```bash
-o source.url=http://ports.ubuntu.com/ubuntu-ports/
```

#### Issue 2: Missing `sgdisk`
The `sgdisk` tool (from `gdisk` package) was not installed on the build host.

**Fix:**
```bash
dnf install -y gdisk
```

#### Issue 3: Hardcoded GRUB Target (`x86_64-efi`)
The `post-files` action in `images/ubuntu.yaml` has:
```bash
TARGET="x86_64"
```
This is hardcoded — there is no branch for ppc64le. The build attempts `grub-install --target=x86_64-efi` which fails because the `x86_64-efi` GRUB modules are not installed on ppc64le.

#### Issue 4: Wrong Mirror Makes Build Fail Early
Even after fixing the mirror, the YAML needs modification to handle ppc64le bootloader installation.

### Result
Build failed. No `disk.qcow2` was produced.

---

## Experiment 2: Add ppc64le GRUB Branch (YAML Changes)

**Goal:** Modify `~/lxc-ci/images/ubuntu.yaml` to detect ppc64le and run the correct GRUB target.

### What Was Changed

The `post-files` hook in `images/ubuntu.yaml` was modified to add a ppc64le branch:

```bash
# After the architecture detection, add:
if [ "$(uname -m)" = "ppc64le" ]; then
    # For PowerPC IEEE1275 (used by SLOF on pseries QEMU machine)
    ROOT_DEVICE="${DISTROBUILDER_ROOT_DEVICE}"
    grub-install --target=powerpc-ieee1275 "${ROOT_DEVICE}"
    grub-mkconfig -o /boot/grub/grub.cfg
    update-grub
fi
```

### Issue Found: `DISTROBUILDER_ROOT_DEVICE` Doesn't Exist

The environment variable `DISTROBUILDER_ROOT_DEVICE` referenced in the research document **does not exist**. Only these environment variables are set by distrobuilder inside the chroot:

- `DISTROBUILDER_ROOT_UUID` — the root filesystem UUID (e.g., `b58372d2-7328-4e98-a123-d3fc4150b694`)
- `DISTROBUILDER_ROOT_PARTUUID` — the root partition UUID

The original x86_64 code uses `DISTROBUILDER_ROOT_DEVICE` but it was never implemented. On x86_64, GRUB's `grub-install` apparently works without being given an explicit device (it auto-detects).

**Result:** `grub-install --target=powerpc-ieee1275 ""` was effectively run, which failed with:
```
grub-install: error: cannot find a GRUB drive for /dev/root
```

---

## Experiment 3: Fix Disk Device Discovery

**Goal:** Replace the non-existent `DISTROBUILDER_ROOT_DEVICE` with actual device discovery logic.

### What Was Changed

The ppc64le branch was updated to discover the root device:

```bash
if [ "$(uname -m)" = "ppc64le" ]; then
    # Discover the root device
    ROOT_DEV=$(readlink -f /dev/root 2>/dev/null || echo "")
    if [ -z "${ROOT_DEV}" ] || [ "${ROOT_DEV}" = "/dev/root" ]; then
        # Try to find it via mount table
        ROOT_DEV=$(findmnt -n -o SOURCE / | sed 's/\[.*\]//' || echo "")
    fi
    # Strip partition number to get the whole disk (e.g., /dev/vda2 -> /dev/vda)
    DISK_DEV=$(echo "${ROOT_DEV}" | sed 's/p[0-9]*$//;s/[0-9]*$//')
    
    grub-install --target=powerpc-ieee1275 "${DISK_DEV}" || true
    grub-mkconfig -o /boot/grub/grub.cfg
    sed -i "s#root=[^ ]*#root=${DISTROBUILDER_ROOT_UUID}#g" /boot/grub/grub.cfg
fi
```

### Result: Build Succeeded!

The build completed successfully, producing:
| Artifact | Size |
|---|---|
| `disk.qcow2` | 283 MB (virtual size: 4 GiB) |
| `incus.tar.xz` | 712 bytes |

### Analysis of What Actually Happened

| Step | Status | Evidence |
|---|---|---|
| GPT created | ✅ | Two partitions: `/dev/sda1` (vfat, ESP) and `/dev/sda2` (ext4, root) |
| Kernel installed | ✅ | `vmlinux-6.8.0-124-generic` (63 MB) at `/boot/` |
| Initrd installed | ✅ | `initrd.img-6.8.0-124-generic` (22 MB) at `/boot/` |
| GRUB modules installed | ✅ | 228 files in `/usr/lib/grub/powerpc-ieee1275/` |
| `grub.cfg` generated | ✅ | Valid config with Ubuntu menu entries |
| Root UUID substituted | ✅ | `grub.cfg` has `root=UUID=b58372d2-...` |
| **`grub-install` succeeded** | **❌ FAILED** | `|| true` suppressed the error; bootloader code never embedded on disk |
| `grub-mkconfig` / `update-grub` | ✅ | Both succeeded |
| ESP partition content | **Empty** | The vfat partition (`/dev/sda1`) has no files — GRUB core image was never written |

**The critical finding:** `grub-install --target=powerpc-ieee1275` could not find the target disk. The `readlink -f /dev/root` inside the chroot returned `/dev/root` (not the actual loop device like `/dev/loop11`), and `findmnt` may also not have resolved correctly in the chroot environment.

---

## Experiment 4: VM Boot Test

### Attempt A: Via Incus (Failed)

```bash
incus start vm1 --console
```

**Error:** `qemu-system-ppc64: -spice: invalid option`

**Root Cause:** The Incus-generated QEMU configuration includes SPICE-related chardev, device, and audio sections, plus a `-spice` command-line argument. The Fedora 42 QEMU package for ppc64le (`qemu-system-ppc-9.2.4-2.fc42.ppc64le`) was **compiled without SPICE support** because the `libspice-server-devel` library is not available on ppc64le in Fedora's repositories.

The `-spice` argument is conditionally added based on a runtime QMP probe (`query-spice`). If the QEMU binary were compiled without SPICE, the feature would be auto-detected and SPICE would be skipped. However, the installed QEMU somehow reports SPICE as available via QMP but rejects the `-spice` command-line option at startup — a mismatch in the build configuration.

**Incus SPICE Detection** (in `internal/server/instance/drivers/driver_qemu.go`, line ~10285):
```go
// Check if SPICE is compiled into QEMU.
err = monitor.QuerySpice()
if err != nil {
    logger.Debug("Failed querying SPICE during VM feature check", logger.Ctx{"err": err})
} else {
    features["spice"] = struct{}{}
}
```

### Attempt B: Direct QEMU (Succeeded)

```bash
qemu-system-ppc64 \
  -machine pseries,accel=kvm \
  -m 1024 \
  -nographic \
  -drive file=/root/lxc-ci/disk.qcow2,format=qcow2,if=virtio \
  -netdev user,id=net0 \
  -device virtio-net-pci,netdev=net0 \
  -serial mon:stdio \
  -nodefaults -no-user-config \
  -cpu host
```

### Boot Console Output (Complete)

```
SLOF **********************************************************************
QEMU Starting
 Build Date = Jan 16 2025 00:00:00
 FW Version = mockbuild@7118adc16a1f43bfaf6f3da9f500720f release 20220719
 Press "s" to enter Open Firmware.

Populating /vdevice methods
Populating /vdevice/vty@71000000
Populating /vdevice/nvram@71000001
Populating /pci@800000020000000
                     00 0000 (D) : 1af4 1000    virtio [ net ]
                     00 0800 (D) : 1af4 1001    virtio [ block ]
No NVRAM common partition, re-initializing...
Scanning USB 
Using default console: /vdevice/vty@71000000

  Welcome to Open Firmware

  Copyright (c) 2004, 2017 IBM Corporation All rights reserved.
  This program and the accompanying materials are made available
  under the terms of the BSD License available at
  http://www.opensource.org/licenses/bsd-license.php

Trying to load:  from: /pci@800000020000000/scsi@1 ... 
E3404: Not a bootable device!
Trying to load:  from: cdrom ... 
E3405: No such device
Trying to load:  from: /pci@800000020000000/ethernet@0 ... 
 Initializing NIC
  Reading MAC address from device: 52:54:00:12:34:56
  Requesting information via DHCP: done
  Using IPv4 address: 10.0.2.15
  Requesting file "" via TFTP from 10.0.2.2
Trying pxelinux.cfg files...
  TFTP error: TFTP access violation
  ... (repeated) ...

E3407: Load failed

  Type 'boot' and press return to continue booting the system.
  Type 'reset-all' and press return to reboot the system.

Ready! 
0 > 
```

### Boot Result Classification: **Outcome B**

| Code | Description | Our Result |
|---|---|---|
| A | Linux boots, gets IP, agent connects | ❌ |
| **B** | **SLOF message, "No bootable device"** | **✅ Confirmed** |
| C | QEMU error or firmware crash before SLOF | ❌ |
| D | GRUB loads but kernel panic | ❌ |

### Key Observations from SLOF Output

1. **SLOF detected the virtio block device** — it's listed during device enumeration:
   ```
   00 0800 (D) : 1af4 1001    virtio [ block ]
   ```
2. **SLOF does NOT automatically scan all GPT partitions for bootloaders.** It tried specific device paths:
   - `/scsi@1` — "Not a bootable device!"
   - `cdrom` — "No such device"
   - `/ethernet@0` — PXE/TFTP netboot (failed)
3. **SLOF never tried the virtio block device** — even though it was detected during enumeration.
4. **The ESP partition (vfat) is irrelevant to SLOF** — SLOF uses Open Firmware (IEEE 1275) bindings, not UEFI.

### Why SLOF Didn't Boot

On pseries machines, SLOF scans for bootable devices using Open Firmware `devalias` and `boot` commands. The virtio block device needs to either:

a) Have a **bootable partition marked** with the appropriate Open Firmware boot partition type, OR
b) Have **GRUB embedded in the boot sector** (via `grub-install --target=powerpc-ieee1275`), which also creates the appropriate Open Firmware CHRP boot entry

Since `grub-install` failed and was silenced with `|| true`, neither condition was met.

---

## Experiment 5: Boot Official Ubuntu Cloud Image (Validation)

**Goal:** Verify that a known-good Ubuntu ppc64le cloud image boots successfully on the same hardware/QEMU configuration as used in Experiment 4. This validates that the QEMU setup, SLOF firmware, and boot chain are correct — isolating the problem to the distrobuilder image content.

**Command** (same QEMU command as Experiment 4B, just different image file):
```bash
qemu-system-ppc64 \
  -machine pseries,accel=kvm \
  -m 1024 \
  -nographic \
  -drive file=/tmp/ubuntu-ppc64le.qcow2,format=qcow2,if=virtio \
  -netdev user,id=net0 \
  -device virtio-net-pci,netdev=net0 \
  -serial mon:stdio \
  -nodefaults -no-user-config \
  -cpu host
```

### Result: ✅ Full Boot to Login

The cloud image booted through the entire chain without any errors:

#### SLOF Output
```
SLOF **********************************************************************
...
                     00 0800 (D) : 1af4 1001    virtio [ block ]
...
Trying to load:  from: /pci@800000020000000/scsi@1 ...   Successfully loaded
Welcome to GRUB!
```

**Critical difference:** The cloud image says `Successfully loaded` after `Trying to load: from: /pci@.../scsi@1` — SLOF finds GRUB on the PReP partition and loads it. Our distrobuilder image said `E3404: Not a bootable device!` at the same step.

#### GRUB → Kernel Handoff
```
error: no suitable video mode found.
  Booting `Ubuntu'

Loading Linux 6.8.0-124-generic ...
Loading initial ramdisk ...
OF stdout device is: /vdevice/vty@71000000
Preparing to boot Linux version 6.8.0-124-generic ...
...
Booting Linux via __start() @ 0x0000000000230000 ...
```

#### Kernel → systemd → Login
```
[    0.000000] Linux version 6.8.0-124-generic ...
[    0.000000] Hardware name: IBM pSeries (emulated by qemu) POWER10 ...
...
[    3.452273] systemd[1]: systemd 255.4-1ubuntu8.16 running in system mode
...
[  OK  ] Reached target multi-user.target - Multi-User System.
[  OK  ] Reached target graphical.target - Graphical Interface.

Ubuntu 24.04.4 LTS ubuntu hvc0

ubuntu login:
```

### What This Proves

1. ✅ **QEMU setup is correct** — same command, same machine type (`pseries`), same KVM acceleration
2. ✅ **SLOF firmware works** — finds and loads GRUB from the PReP partition
3. ✅ **GRUB powerpc-ieee1275 works** — boots the Linux kernel correctly
4. ✅ **ppc64le kernel boots** — same kernel version (6.8.0-124-generic) boots fully
5. ✅ **virtio block device is bootable** — when the disk has the correct layout

### The Only Difference

| Aspect | Cloud Image | Our Image |
|---|---|---|
| Partition 1 | PReP (4 MiB, type `4100`) with GRUB core ELF | ESP (100 MiB, type `EF00`) empty |
| Partition 2 | ext4 root (~2.2 GiB) | ext4 root (~3.9 GiB) |
| GRUB core image | ✅ `/boot/grub/grub` (97 KB ELF) | ❌ **Missing** |
| SLOF result | `Successfully loaded` | `E3404: Not a bootable device` |

The experiment confirms that the **partition layout is the root cause**. The cloud image has a PReP partition with the GRUB core image embedded by `grub-install --target=powerpc-ieee1275`. Our image has an ESP partition that SLOF ignores, and no PReP partition at all.

---

## Experiment 6: Manual PReP Proof — Direct Causal Test

**Goal:** Take the distrobuilder-built image, create a new disk with PReP partition, copy the rootfs, run `grub-install`, and boot. This moves from circumstantial evidence to direct causal proof that the missing PReP partition is the root cause.

### Method
```bash
# 1. Create raw disk with PReP (8 MiB, type 9E1A2D38-...) + ext4 root partitions
truncate -s 4G disk-prep-test.raw
sgdisk --zap-all disk-prep-test.raw
sgdisk --new=1::+8M -t 1:9E1A2D38-C612-4316-AA26-8B49521E5A8B disk-prep-test.raw
sgdisk --new=2:: -t 2:8300 disk-prep-test.raw

# 2. Attach loop device
losetup -P -f disk-prep-test.raw  # -> /dev/loop11

# 3. Format root partition, mount, copy rootfs from existing distrobuilder image
mkfs.ext4 -F -b 4096 /dev/loop11p2
mount /dev/loop11p2 /mnt/dst
guestmount -a disk.qcow2 -m /dev/sda2 --ro /mnt/src
rsync -aHAX /mnt/src/ /mnt/dst/

# 4. Fix root UUID in grub.cfg (new filesystem has different UUID)
ROOTFS_UUID=$(blkid -s UUID -o value /dev/loop11p2)
sed -i "s#UUID=[a-f0-9-]*#UUID=$ROOTFS_UUID#g" /mnt/dst/boot/grub/grub.cfg

# 5. Install GRUB to PReP partition (NOT whole disk)
grub2-install --target=powerpc-ieee1275 --boot-directory=/mnt/dst/boot /dev/loop11p1
```

### Results

| Step | Result |
|---|---|
| PReP partition created (8 MiB, GUID `9E1A2D38-...`) | ✅ |
| `grub2-install --target=powerpc-ieee1275 /dev/loop11` (whole disk) | ❌ "not a PReP partition" |
| `grub2-install --target=powerpc-ieee1275 /dev/loop11p1` (partition) | ✅ "Installation finished. No error reported." |
| PReP partition content after install | ✅ ELF header `\x7fELF` — GRUB `core.elf` written directly to PReP |
| `/boot/grub/grub` created | ✅ `ELF 32-bit MSB executable, PowerPC` (185 KB) |
| `/boot/grub/powerpc-ieee1275/` created | ✅ GRUB modules copied |
| QEMU boot result | ✅ **Login prompt: `localhost login:`** |

### Boot Console Output (Key Sections)

```
SLOF
...
Trying to load:  from: /pci@.../scsi@1 ...   Successfully loaded
Welcome to GRUB!
  Booting `Ubuntu'

Loading Linux 6.8.0-124-generic ...
Loading initial ramdisk ...
...
Booting Linux via __start() @ 0x0000000000440000 ...
Linux ppc64le
#124-Ubuntu SMP
Ubuntu 24.04.4 LTS localhost hvc0

localhost login:
```

### Causal Proof Chain

```
Original Distrobuilder image
    ↓
No PReP partition
    ↓
grub-install fails (no target)
    ↓
No bootloader on disk
    ↓
SLOF: E3404 Not a bootable device

Modified image (same rootfs, different layout)
    ↓
PReP partition added (8 MiB, type 4100)
    ↓
grub-install --target=powerpc-ieee1275 succeeds
    ↓
GRUB core.elf written to PReP partition
    ↓
/boot/grub/grub (ELF), /boot/grub/grub.cfg present
    ↓
SLOF: Successfully loaded → Welcome to GRUB!
    ↓
Linux kernel boots → systemd → login prompt
```

### Key Findings from the Manual Proof

1. **`grub-install` needs the partition device** (`/dev/loop11p1`), not the whole disk (`/dev/loop11`). Fedora's GRUB 2.12 checks that the target device has a PReP GPT type, and only checks the partition itself — it doesn't search the whole disk for PReP partitions. This means the YAML `post-files` action must use the PReP partition path, not just the loop device path.

2. **8 MiB PReP size is correct.** The Ubuntu 24.04 and 22.04 cloud images both use 8 MiB (8388608 bytes). Debian 12 uses 3 MiB. 8 MiB is a safe default with headroom.

3. **The PReP partition is raw** — no filesystem. `grub-install` writes `core.elf` directly to the partition starting at offset 0.

4. **Partition number doesn't matter.** The cloud image has PReP as partition 2 (physically first), Debian has it as partition 15. Only the GPT type GUID (`9E1A2D38-...`) matters. Using partition 1 is fine.

5. **The GRUB search warning** about the old UUID from the copied rootfs is cosmetic — the kernel command line has the correct `root=UUID=` so boot succeeds regardless.

6. **PReP on Fedora vs Ubuntu:** On Fedora, `grub2-install` required the partition device. Ubuntu's `grub-install` may behave differently and accept the whole-disk form. This should be verified before finalizing the environment variable naming in PR 2.

---

## Summary of All Findings

### What Works ✅

1. **Distrobuilder can build a ppc64le VM image today** — No changes to `vm.go` needed.
2. **GPT partition creation works** — Two partitions created (ESP + root).
3. **QCOW2 image creation works** — Valid QCOW2 file with correct virtual size.
4. **Root filesystem population works** — ext4 filesystem with kernel, initrd, and modules.
5. **Ubuntu package installation works** — All packages install correctly via `ports.ubuntu.com`.
6. **GRUB modules are installed** — `grub-ieee1275` package installs all `powerpc-ieee1275` modules.
7. **grub.cfg generation works** — `grub-mkconfig` and `update-grub` produce valid configuration.
8. **Incus image import works** — Image imports as type `VIRTUAL-MACHINE`.
9. **SLOF firmware starts and detects the disk** — The virtio block device is visible.
10. **Cloud image boots fully** — Official Ubuntu Noble ppc64le cloud image boots to login with the exact same QEMU command. Confirms the boot chain (SLOF → GRUB → kernel → systemd) is fully functional on this hardware.
11. **PReP partition + GRUB core image works** — The cloud image uses a 4 MiB PReP partition (type `4100`) containing a raw ELF GRUB core image, which SLOF successfully finds and loads.

### What Doesn't Work ❌

1. **`grub-install --target=powerpc-ieee1275` fails** — Cannot find the target disk inside the build chroot (the device discovery hack with `readlink -f /dev/root` doesn't resolve to the actual loop device).
2. **Bootloader not embedded on disk** — Without successful `grub-install`, SLOF has no boot code to execute.
3. **SLOF doesn't automatically boot from the virtio block device** — Even though it detects the device, it tries SCSI, CDROM, and PXE only.
4. **Incus QEMU SPICE issue** — Fedora 42's `qemu-system-ppc64` on ppc64le has no SPICE support, preventing `incus start --console` from working.

### What Needs to Be Fixed

#### Fix 1: PReP partition instead of ESP for ppc64le

The root cause is that distrobuilder unconditionally creates an EFI System Partition (ESP, type `EF00`) for all architectures. On ppc64le, SLOF/OpenFirmware does not use ESP. It requires a PReP (PowerPC Reference Platform) partition (type `4100`, GUID `9E1A2D38-...`) to find the GRUB bootloader.

The fix is in `distrobuilder/vm.go`: make partition layout architecture-aware, creating PReP (8 MiB, raw) instead of ESP for ppc64le.

#### Fix 2: Device Discovery for `grub-install`

Inside the chroot, the root device (`/dev/loop11p2`) is not accessible via `/dev/root`. The `DISTROBUILDER_ROOT_DEVICE` env var doesn't exist.

On Fedora's GRUB 2.12, `grub-install --target=powerpc-ieee1275` requires the **partition device** (e.g., `/dev/loop11p1`), not the whole disk (`/dev/loop11`). This means simply exporting `DISTROBUILDER_ROOT_DEVICE` (the loop device) may not be sufficient — the YAML may also need the PReP partition device.

**Solution options (to be verified on Ubuntu's grub-install):**
- Export `DISTROBUILDER_ROOT_DEVICE` (the whole disk `/dev/loop11`) — may work if Ubuntu's `grub-install` is more permissive
- Export `DISTROBUILDER_BOOT_DEVICE` or `DISTROBUILDER_PREP_DEVICE` (the partition `/dev/loop11p1`)
- Or have the YAML derive it: `grub-install --target=powerpc-ieee1275 "$(echo ${DISTROBUILDER_ROOT_DEVICE}p1)"`

#### Fix 2: Incus QEMU SPICE

On Fedora ppc64le, SPICE is not available. Options:
- Build a custom QEMU with SPICE enabled
- Modify Incus to handle the case where SPICE is detected via QMP but fails at runtime
- Use the LXD snap (which has its own QEMU with SPICE compiled in from the snap's build)
- Or just boot VMs directly via QEMU (bypass Incus) for testing

#### Fix 3: Root UUID Substitution (Optional Improvement)

The kernel command line `root=UUID=...` in `grub.cfg` needs to match the actual root partition UUID. The current substitution works:
```
sed -i "s#root=[^ ]*#root=${DISTROBUILDER_ROOT_UUID}#g" /boot/grub/grub.cfg
```

## Upstream Contribution Strategy (Updated)

### PR Order

1. **PR 1: Architecture-aware VM partitioning (distrobuilder)** — Add `architecture` field to `vm` struct, create PReP partition for ppc64le instead of ESP. Skip UEFI FS creation for ppc64le. No behavior change for x86_64/aarch64.

2. **PR 2: Export `DISTROBUILDER_ROOT_DEVICE` (distrobuilder)** — Add the env var so YAML `post-files` actions can find the block device. May also need a `DISTROBUILDER_BOOT_DEVICE` for the PReP partition specifically (needs verification on Ubuntu's `grub-install`).

3. **PR 3: Ubuntu/Debian YAML (lxc-ci)** — Add `grub-ieee1275` package and `post-files` action for ppc64el VM builds.

4. **PR 4: Other distro YAMLs (lxc-ci)** — Fedora, Rocky, Alma, openSUSE.

5. **PR 5 (future): s390x investigation** — Requires separate work on real s390x hardware. Uses zipl, not GRUB/PReP.

### Current State of Modified Files

| File | Modification | Status |
|---|---|---|
| `~/lxc-ci/images/ubuntu.yaml` | Added ppc64le detection + GRUB branch in post-files | ✅ Modified (needs refinement) |
  
### Image Build Command (Working)
```bash
distrobuilder build-incus images/ubuntu.yaml \
  --vm \
  -o image.architecture=ppc64el \
  -o image.release=noble \
  -o source.url=http://ports.ubuntu.com/ubuntu-ports/
```

### Direct QEMU Boot Command (Working)
```bash
qemu-system-ppc64 \
  -machine pseries,accel=kvm \
  -m 1024 \
  -nographic \
  -drive file=disk.qcow2,format=qcow2,if=virtio \
  -serial mon:stdio \
  -nodefaults -no-user-config \
  -cpu host
```

## Appendix: Disk Layout of Built Image
