# Research & Analysis: Architecture Support, VM Pipeline, and YAML Changes

This document covers the Incus QEMU driver analysis, distrobuilder VM pipeline, image definition YAML changes needed, lxc-ci build infrastructure, and partition layout considerations.

**Belongs to:** [docs/ppc64le-vm-support/](index.md)

---


| Repository | Purpose | Key Directories |
|---|---|---|
| `github.com/lxc/incus` | Incus daemon (QEMU driver) | `internal/server/instance/drivers/driver_qemu.go` |
| `github.com/lxc/distrobuilder` | Image builder | `distrobuilder/vm.go`, `shared/osarch.go`, `shared/definition.go` |
| `github.com/lxc/lxc-ci` | CI/build infrastructure + image YAMLs | `images/*.yaml`, `bin/build-distro`, `bin/jenkins-import-images` |

---

## Incus Agent (incus-agent) Mechanism

The `incus-agent` is a small daemon that runs **inside** each VM, enabling `incus exec`, `incus file`, `incus console`, and other instance management operations. It communicates with incusd over **vsock** (a Linux VM-socket).

### Discovery and Connection Flow

1. **incusd (host)** starts the VM via QEMU and waits for `EventAgentStarted` from QEMU's QMP monitor
2. **QEMU** exposes two virtio devices to the VM:
   - `vhost-vsock-pci` — for vsock communication (on ppc64le)
   - A virtio serial port named `org.linuxcontainers.incus` — for the udev trigger
3. **incusd** configures a 9p shared directory (`config` mount tag) containing:
   - `incus-agent` binary (copied from `/usr/local/bin/incus-agent` on the host)
   - TLS certificates (`agent.crt`, `agent.key`, `server.crt`)
   - `agent.conf` (JSON with host vsock CID, port, and certificate)
   - Systemd service files and udev rules
4. **Inside the VM** (at boot):
   - udev rule (`99-incus-agent.rules`) detects the virtio serial port and triggers `incus-agent.service`
   - The `incus-agent-setup` script mounts the 9p config share, copies files to `/run/incus_agent/`, and starts the agent
   - `incus-agent` opens vsock listener on port 8443 (VMADDR_CID_ANY)
   - Agent receives `agent.conf` from host via PUT `/1.0` and establishes a vsock TLS connection back to incusd

### Vsock Communication

| Direction | Source CID | Dest CID | Port | Protocol |
|---|---|---|---|---|
| Host → Agent | Host (2) | VM CID (≥3) | 8443 | vsock + HTTPS |
| Agent → Host | VM CID (≥3) | Host (2) | Random (1024-65535) | vsock + HTTPS |

### ppc64le-Specific Considerations

- **vsock on ppc64le:** Uses `vhost-vsock-pci` (standard PCI bus) — works on pseries machine type
- **9p on ppc64le:** Uses `virtio-9p-pci` — works on pseries
- **Kernel modules:** `vsock` and `vhost_vsock` are in mainline Linux — available on ppc64le
- **Agent binary:** Built with `CGO_ENABLED=0 go install -tags agent,netgo ./cmd/incus-agent` — Go supports `GOARCH=ppc64le`
- **No special ppc64le changes needed** — all virtio devices work the same regardless of architecture

### Key Source Files

| File | Purpose |
|---|---|
| `cmd/incus-agent/main_agent.go` | Agent entry point |
| `cmd/incus-agent/daemon.go` | Daemon struct with CID/port state |
| `cmd/incus-agent/os_linux.go` | Vsock listener setup, module loading |
| `internal/server/instance/drivers/driver_qemu.go` | `generateConfigShare()`, `advertiseVsockAddress()`, `getAgentClient()` |
| `internal/server/endpoints/vsock.go` | Host vsock server |
| `internal/server/vsock/vsock.go` | Vsock HTTP client |
| `internal/server/instance/drivers/agent-loader/systemd/` | Systemd service files embedded in incusd binary |

## Architecture Support Matrix

### Incus QEMU Driver (source of truth)

Found in `github.com/lxc/incus/internal/server/instance/drivers/driver_qemu.go`:

```go
// The single most important function for understanding VM boot:
func (d *qemu) architectureSupportsUEFI(arch int) bool {
    return slices.Contains([]int{
        osarch.ARCH_64BIT_INTEL_X86,      // x86_64
        osarch.ARCH_64BIT_ARMV8_LITTLE_ENDIAN, // aarch64
    }, arch)
}
```

### Per-Architecture QEMU Configuration

Source: `driver_qemu.go`, function `qemuArchConfig()`:

```go
func (d *qemu) qemuArchConfig(arch int) (string, string, error) {
    switch arch {
    case osarch.ARCH_64BIT_INTEL_X86:
        qemuCmd = "qemu-system-x86_64"
        bus = "pcie"
    case osarch.ARCH_64BIT_ARMV8_LITTLE_ENDIAN:
        qemuCmd = "qemu-system-aarch64"
        bus = "pcie"
    case osarch.ARCH_64BIT_POWERPC_LITTLE_ENDIAN:
        qemuCmd = "qemu-system-ppc64"
        bus = "pci"              // <-- NOT pcie!
    case osarch.ARCH_64BIT_S390_BIG_ENDIAN:
        qemuCmd = "qemu-system-s390x"
        bus = "ccw"              // <-- Channel I/O bus
    }
}
```

Source: `driver_qemu_templates.go`, function `qemuMachineType()`:

```go
func qemuMachineType(architecture int) string {
    switch architecture {
    case osarch.ARCH_64BIT_INTEL_X86:             machineType = "q35"
    case osarch.ARCH_64BIT_ARMV8_LITTLE_ENDIAN:   machineType = "virt"
    case osarch.ARCH_64BIT_POWERPC_LITTLE_ENDIAN: machineType = "pseries"
    case osarch.ARCH_64BIT_S390_BIG_ENDIAN:       machineType = "s390-ccw-virtio"
    }
    return machineType
}
```

### Summary Table

| Architecture | ARCH Constant | QEMU binary | Machine | Bus | UEFI? | Firmware |
|---|---|---|---|---|---|---|
| x86_64 | `ARCH_64BIT_INTEL_X86` | `qemu-system-x86_64` | q35 | pcie | ✅ Yes | OVMF (EDK2) |
| aarch64 | `ARCH_64BIT_ARMV8_LITTLE_ENDIAN` | `qemu-system-aarch64` | virt | pcie | ✅ Yes | AAVMF (EDK2) |
| **ppc64le** | `ARCH_64BIT_POWERPC_LITTLE_ENDIAN` | `qemu-system-ppc64` | pseries | pci | ❌ No | SLOF (built-in) |
| **s390x** | `ARCH_64BIT_S390_BIG_ENDIAN` | `qemu-system-s390x` | s390-ccw-virtio | ccw | ❌ No | Built-in CCW FW |

### Key Architectural Differences for Non-UEFI Architectures

1. **No NVRAM setup** — `setupNvram()` is only called when `architectureSupportsUEFI()` returns true
2. **No pflash firmware drive** — `qemuDriveFirmware()` is skipped
3. **Different device buses** — ppc64le uses `pci` (not `pcie`), s390x uses `ccw` (channel I/O)
4. **No USB on s390x** — explicit guard: `if d.architecture != osarch.ARCH_64BIT_S390_BIG_ENDIAN { ... USB ... }`
5. **No core info dump on ppc64le/s390x** — explicit guard in `qemuCoreInfo()`
6. **No VGA mode on non-x86** — GPU code checks architecture

### Architecture Constants (from `shared/osarch/architectures.go`)

```go
ARCH_64BIT_POWERPC_LITTLE_ENDIAN = 7  // ppc64le
ARCH_64BIT_S390_BIG_ENDIAN       = 8  // s390x
```

### Architecture Naming in Distrobuilder

From `shared/osarch.go` in distrobuilder:

```go
var debianArchitectureNames = map[int]string{
    osarch.ARCH_64BIT_POWERPC_LITTLE_ENDIAN: "ppc64el",  // Debian/Ubuntu naming
}

var gentooArchitectureNames = map[int]string{
    osarch.ARCH_64BIT_POWERPC_LITTLE_ENDIAN: "ppc64le",  // Gentoo naming
    osarch.ARCH_64BIT_S390_BIG_ENDIAN:       "s390x",
}
```

**Important naming difference:** Debian/Ubuntu call it `ppc64el`, while RHEL/Fedora/Gentoo call it `ppc64le`. Distrobuilder handles this through architecture maps per distro.

---

## Distrobuilder VM Pipeline Analysis

### File: `distrobuilder/vm.go` (full analysis)

**VM struct:**
```go
type vm struct {
    imageFile  string
    loopDevice string
    rootFS     string    // "ext4" or "btrfs"
    rootfsDir  string
    bootfsDir  string    // Always /boot/efi !!
    size       uint64
    ctx        context.Context
}
```

**Key observation:** The VM struct has **no architecture field**. Boot behavior is entirely determined by the image YAML actions.

### What's unconditional (applies to ALL architectures):

1. **`createEmptyDiskImage()`** — raw sparse file — architecture-agnostic ✅
2. **`createPartitions()`** — GPT with sgdisk — architecture-agnostic ✅
   ```go
   args = [][]string{
       {"--zap-all"},
       {"--new=1::+100M", "-t 1:EF00"},  // EFI System Partition (100MB, vfat)
       {"--new=2::", "-t 2:8300"},        // Linux root filesystem
   }
   ```
3. **`createUEFIFS()`** — creates vfat filesystem labeled "UEFI" — **always runs** ⚠️
4. **`createRootFS()`** — ext4 or btrfs — architecture-agnostic ✅
5. **`mountUEFIPartition()`** — always mounts to `/boot/efi` — **always runs** ⚠️

### What's architecture-dependent (handled via YAML post-files actions):

1. **Bootloader installation** — GRUB, zipl, etc.
2. **Initramfs regeneration**
3. **Kernel command-line configuration**

### Assessment of Unconditional UEFI Behavior

For **ppc64le**: The vfat ESP partition is created but SLOF doesn't use it. GRUB installed to the disk with `--target=powerpc-ieee1275` writes its core image to the GPT post-gap area (between MBR and first partition), which is standard. **The ESP is harmless.**

For **s390x**: The vfat ESP partition is non-standard. The s390-ccw firmware looks for an IPL (Initial Program Load) record, typically in the first block of the disk or a dedicated small partition. The ESP may be ignored if the root partition contains a valid zipl boot record. **Needs testing to confirm.**

### Device file naming (`getRootfsDevFile` / `getUEFIDevFile`)

```go
func (v *vm) getRootfsDevFile() string {
    return fmt.Sprintf("%sp2", v.loopDevice)  // e.g., /dev/loop0p2
}
func (v *vm) getUEFIDevFile() string {
    return fmt.Sprintf("%sp1", v.loopDevice)  // e.g., /dev/loop0p1
}
```

This naming is host-side and works identically regardless of guest architecture. The `p1`/`p2` refers to partition numbers on the loop device.

---

## Image Definition YAML Changes Needed

### Current State: What Already Exists

In `lxc-ci/images/ubuntu.yaml` — Ubuntu already has ppc64el-specific lines:
```yaml
- packages:
    - language-pack-en
  action: install
  architectures:
  - ppc64el
```

But there are **no ppc64le VM bootloader packages**:
```yaml
# Only amd64 and arm64 have VM bootloader packages currently:
- packages:
    - grub-efi-amd64-signed
    - shim-signed
  architectures:
  - amd64
  types:
  - vm

- packages:
    - grub-efi-arm64-signed
  architectures:
  - arm64
  types:
  - vm
```

### What Needs to Be Added Per Distro

#### Ubuntu / Debian (ppc64le)

In the `packages.sets` section:
```yaml
- packages:
    - grub-ieee1275
  architectures:
    - ppc64el
  types:
    - vm
```

In the `actions` section (post-files):
```yaml
- trigger: post-files
  action: |-
    #!/bin/sh
    set -eux
    # Install GRUB for PowerPC IEEE1275 (used by SLOF on pseries)
    grub-install --target=powerpc-ieee1275 "$(lsblk -ndo pkname "${DISTROBUILDER_ROOT_DEVICE}")"
    grub-mkconfig -o /boot/grub/grub.cfg
    sed -i "s#root=[^ ]*#root=${DISTROBUILDER_ROOT_UUID}#g" /boot/grub/grub.cfg
  architectures:
    - ppc64el
  types:
    - vm
```

#### Ubuntu / Debian (s390x)

```yaml
- packages:
    - s390-tools
  architectures:
    - s390x
  types:
    - vm

- trigger: post-files
  action: |-
    #!/bin/sh
    set -eux
    # Find kernel and initramfs
    kver=$(ls /boot/initramfs-*.img | sed -r 's#.*initramfs-(.+)\.img#\1#' | head -n1)
    # Install zipl bootloader for s390x
    zipl -V -t /boot \
      -i "/boot/vmlinuz-${kver}" \
      -r "/boot/initramfs-${kver}.img" \
      -P "root=${DISTROBUILDER_ROOT_UUID} ro"
  architectures:
    - s390x
  types:
    - vm
```

#### Rocky Linux / AlmaLinux / CentOS (ppc64le)

```yaml
- packages:
    - grub2-ppc64le
  architectures:
    - ppc64le
  types:
    - vm

- trigger: post-files
  action: |-
    #!/bin/sh
    set -eux
    # Install GRUB for ppc64le
    grub2-install --target=powerpc-ieee1275 "$(lsblk -ndo pkname "${DISTROBUILDER_ROOT_DEVICE}")"
    grub2-mkconfig -o /boot/grub2/grub.cfg
    sed -i "s#root=[^ ]*#root=${DISTROBUILDER_ROOT_UUID}#g" /boot/grub2/grub.cfg
  architectures:
    - ppc64le
  types:
    - vm
```

#### Rocky Linux / AlmaLinux / CentOS (s390x)

```yaml
- packages:
    - s390-tools
    - grub2-s390x
  architectures:
    - s390x
  types:
    - vm

- trigger: post-files
  action: |-
    #!/bin/sh
    set -eux
    kver=$(ls /boot/initramfs-*.img | sed -r 's#.*initramfs-(.+)\.img#\1#' | head -n1)
    # Install zipl for s390x
    zipl -V -t /boot \
      -i "/boot/vmlinuz-${kver}" \
      -r "/boot/initramfs-${kver}.img" \
      -P "root=${DISTROBUILDER_ROOT_UUID} ro"
  architectures:
    - s390x
  types:
    - vm
```

**Package name reference table:**

| Distro | ppc64le GRUB | s390x bootloader |
|---|---|---|
| Ubuntu/Debian | `grub-ieee1275` | `s390-tools` |
| Fedora | `grub2-ppc64le` | `s390-tools` + `grub2-s390x` |
| Rocky/Alma/CentOS | `grub2-ppc64le` | `s390-tools` + `grub2-s390x` |
| openSUSE | `grub2-powerpc-ieee1275` | `s390-tools` |

---

## lxc-ci Build Infrastructure

### Build Pipeline

```
Jenkins matrix job
  architecture=ppc64el,release=noble,variant=default
       |
       v
bin/build-distro <yaml> <arch> <type> <timeout> <target>
       |
       v
distrobuilder build-dir image.yaml rootfs ...
distrobuilder pack-incus image.yaml rootfs --vm ...   (if type=vm)
distrobuilder pack-lxc image.yaml rootfs ...           (if type=container)
distrobuilder pack-incus image.yaml rootfs ...          (if type=container)
```

### How Architectures Are Added

From `bin/update-images` and `bin/check-images`:

```bash
# Environment variables set on Jenkins build hosts (in /lxc-ci/etc/config):
DISTROBUILDER_ARCHES="amd64 arm64 armhf ppc64el s390x ..."
LXC_ARCHES="amd64 arm64 ..."
```

These are NOT in the repo — they're configured on the Jenkins builders themselves.

### Architecture Naming in `incus_arch()` (`bin/jenkins-import-images`)

```python
def incus_arch(arch):
    if arch == "i386" or arch == "i686":
        return (1, "i686")
    elif arch == "amd64" or arch == "x86_64":
        return (2, "x86_64")
    elif arch == "armel":
        return (3, "armv7l")
    elif arch == "armhf":
        return (3, "armv7l")
    elif arch == "arm64" or arch == "aarch64":
        return (4, "aarch64")
    elif arch == "powerpc":
        return (5, "ppc")
    elif arch == "powerpc64":
        return (6, "ppc64")
    elif arch == "ppc64el":
        return (7, "ppc64le")    # Already supported ✅
    elif arch == "s390x":
        return (8, "s390x")       # Already supported ✅
```

Both `ppc64el` and `s390x` are already mapped. No changes needed here.

### Cache Container Setup

The build system uses pre-built cache containers named `cache-distrobuilder-${ARCH}`. These are published by the `lxc-ci-artifacts` Jenkins job. For a new architecture, you'd need to:

1. Create a cache container for the architecture
2. Publish it as a build artifact
3. Set up the Jenkins builder (real hardware for ppc64le/s390x)

### How `disk.qcow2` Detection Works

From `jenkins-import-images`:
```python
# Check if VM image exists for product listing
if os.path.exists(os.path.join(entry, latest_image, "disk.qcow2")):
    incus_vm = "YES"
```

And in simplestreams JSON:
```python
if os.path.exists("%s/disk.qcow2" % image_dir):
    items["disk.qcow2"] = {
        "ftype": "disk-kvm.img",
        "sha256": files["disk.qcow2"][0],
        ...
    }
```

If `disk.qcow2` exists in the build artifacts, the image server automatically publishes it as a VM image.

---

## Partition Layout & Boot Considerations

### Current Distrobuilder VM Disk Layout

```
GPT Protective MBR
  |
GPT Header
  |
Partition 1: EFI System Partition (type EF00)
  Size: 100MB
  FS: vfat, label "UEFI"
  Mount: /boot/efi
  |
Partition 2: Linux Root (type 8300)
  Size: rest of disk (default 4GB)
  FS: ext4 or btrfs
  Mount: /
```

### ppc64le Boot Chain

```
QEMU pseries machine
  --> SLOF (Slimline Open Firmware) [built into QEMU]
    --> Scans GPT partitions for bootable content
      --> Finds GRUB core image (embedded by grub-install --target=powerpc-ieee1275)
        --> GRUB loads /boot/grub/grub.cfg
          --> Linux kernel boots
```

**Disk layout considerations:**
- SLOF does NOT require a PReP partition. `grub-install --target=powerpc-ieee1275` embeds the boot image in the post-MBR gap (between the GPT header and the first partition) or in a dedicated BIOS boot partition (partition type `0x41` — PReP).
- If `grub-install` complains about embedding failure (gap too small), a 1MB PReP partition (type `0x41`, no filesystem) may be needed as the first partition.
- **The existing vfat ESP partition does not interfere** — SLOF simply ignores it.

### s390x Boot Chain

```
QEMU s390-ccw-virtio machine
  --> Built-in CCW firmware
    --> IPL (Initial Program Load) from disk
      --> Reads zipl boot record from disk
        --> Loads kernel + initramfs specified by zipl config
          --> Linux kernel boots
```

**Disk layout considerations:**
- s390x does NOT use UEFI or OpenFirmware
- `zipl` writes a boot record to the first blocks of the disk (or to a specified partition)
- The vfat ESP partition may be ignored — needs testing
- If the CCW firmware cannot find a valid IPL record, you may need a dedicated zipl partition or to install zipl directly to the root partition

### Distrobuilder Code Change: Architecture-Aware Partitioning

The VM struct needs an architecture field, and partitioning/UEFI FS creation must be architecture-aware:

```go
type vm struct {
    imageFile    string
    loopDevice   string
    rootFS       string
    rootfsDir    string
    bootfsDir    string
    size         uint64
    ctx          context.Context
    architecture int  // NEW: add architecture field
}
```

```go
func newVM(ctx context.Context, imageFile, rootfsDir, fs string, size uint64, architecture int) (*vm, error) {
    // ... existing validation ...
    return &vm{
        ctx: ctx,
        imageFile: imageFile,
        rootfsDir: rootfsDir,
        rootFS: fs,
        size: size,
        architecture: architecture,  // NEW
    }, nil
}
```

#### Partition layout logic (in `createPartitions`)

Use explicit per-architecture logic, not a boolean `supportsUEFI()`:

```go
func (v *vm) createPartitions(args ...[]string) error {
    if len(args) == 0 {
        switch v.architecture {
        case osarch.ARCH_64BIT_POWERPC_LITTLE_ENDIAN:
            // PReP boot partition (8 MiB, no filesystem) + root
            args = [][]string{
                {"--zap-all"},
                {"--new=1::+8M", "-t 1:4100"},   // PReP (type 0x41)
                {"--new=2::", "-t 2:8300"},        // root
            }
        case osarch.ARCH_64BIT_INTEL_X86, osarch.ARCH_64BIT_ARMV8_LITTLE_ENDIAN:
            // ESP (100 MiB, vfat) + root
            args = [][]string{
                {"--zap-all"},
                {"--new=1::+100M", "-t 1:EF00"},   // ESP
                {"--new=2::", "-t 2:8300"},         // root
            }
        default:
            return fmt.Errorf("unsupported architecture for VM: %d", v.architecture)
        }
    }
    // ...
}
```

#### Conditional UEFI FS creation

In `main_incus.go`, skip `createUEFIFS()` and `mountUEFIPartition()` for ppc64le:

```go
switch vm.architecture {
case osarch.ARCH_64BIT_INTEL_X86, osarch.ARCH_64BIT_ARMV8_LITTLE_ENDIAN:
    err = vm.createUEFIFS()
    // ...
    err = vm.mountUEFIPartition()
    // ...
case osarch.ARCH_64BIT_POWERPC_LITTLE_ENDIAN:
    // No UEFI FS needed — PReP is raw, no filesystem
}
```

**PReP size note:** The Ubuntu 24.04 and 22.04 official cloud images use an 8 MiB PReP partition. Debian 12 uses 3 MiB (partition 15). The 8 MiB value is a safe default that works with `grub-install` and leaves headroom. The PReP partition type GUID is `9E1A2D38-C612-4316-AA26-8B49521E5A8B`, set via sgdisk type code `4100`.

**Important GRUB behavior on Fedora:** `grub2-install --target=powerpc-ieee1275` requires the **partition device** (e.g., `/dev/loop0p1`), not the whole disk (`/dev/loop0`). On Fedora GRUB 2.12, passing the whole disk fails with "the chosen partition is not a PReP partition" even when a valid PReP partition exists. The partition-device approach (`/dev/loop0p1`) succeeds. This behavior should be verified on Ubuntu's `grub-install` before finalizing the environment variable names in the lxc-ci YAML. Ubuntu's grub-install may accept the whole-disk form.

**s390x is intentionally excluded** from this change. The s390x architecture uses a completely different boot mechanism (zipl on the CCW firmware) and requires separate investigation. PReP partitions are PowerPC-specific and not relevant to s390x.

---

## Contribution Strategy

### Recommended PR Order

```
PR 1: Architecture-aware VM partitioning (distrobuilder)
  ├── vm.go: add architecture field, per-arch partition layout
  ├── main_incus.go: skip UEFI FS for ppc64le
  ├── Risk: Very low for x86_64/aarch64 (no behavior change)
  └── Risk: Medium for ppc64le (new code path, untested in CI)

PR 2: Bootloader environment support (distrobuilder)
  ├── shared/util.go: add EnvRootDevice (DISTROBUILDER_ROOT_DEVICE)
  ├── main_incus.go: export loop device as env var
  ├── Risk: Very low
  └── Note: May also need DISTROBUILDER_BOOT_DEVICE for the PReP
      partition specifically (grub-install needs /dev/loopXp1, not /dev/loopX)
      depending on whether Ubuntu's grub-install behaves like Fedora's.

PR 3: Ubuntu/Debian YAML changes (lxc-ci)
  ├── images/ubuntu.yaml: grub-ieee1275 package + post-files action
  ├── images/debian.yaml: same
  ├── Risk: Low — requires PR 1 + PR 2 to actually produce bootable images
  └── Testing: Real ppc64le hardware (you have this)

PR 4: Fedora/Rocky/Alma/CentOS/openSUSE YAML changes (lxc-ci)
  ├── Fedora uses grub2-ppc64le and grub2-install
  ├── Risk: Low
  └── Testing: Real ppc64le hardware

PR 5 (separate): s390x investigation
  ├── s390x uses zipl on CCW firmware, NOT PReP or GRUB
  ├── Requires separate investigation on real s390x hardware
  └── May need different partition layout entirely
```

### Detailed PR 1 Checklist

- [ ] Add `architecture int` field to `vm` struct
- [ ] Update `newVM()` to accept and store architecture
- [ ] Add per-architecture partition logic to `createPartitions()`:
  - `x86_64` / `aarch64` → ESP (100 MiB) + root (current behavior, unchanged)
  - `ppc64le` → PReP (8 MiB, type `4100`) + root
- [ ] Skip `createUEFIFS()` and `mountUEFIPartition()` for ppc64le
- [ ] Update `main_incus.go` to pass architecture to `newVM()`

### Detailed PR 2 Checklist

- [ ] Add `EnvRootDevice = "DISTROBUILDER_ROOT_DEVICE"` to `shared/util.go`
- [ ] Export `DISTROBUILDER_ROOT_DEVICE` with the loop device path in chroot env

### Detailed PR 3 Checklist

- [ ] Add `grub-ieee1275` package install for `architectures: [ppc64el]`, `types: [vm]`
- [ ] Add post-files action for `grub-install --target=powerpc-ieee1275`
- [ ] Add post-files action for `grub-mkconfig` and root UUID substitution
- [ ] Ensure existing amd64/arm64 packages have `architectures:` filters that exclude ppc64el

---

## Testing Protocol
