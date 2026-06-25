# Adding ppc64le and s390x VM Image Support to Incus/Distrobuilder

**Status:** Experiments Complete — PReP Proof Confirmed (distrobuilder image with PReP partition boots to login)
**Date:** 2026-06-24
**Author:** AI-assisted investigation of Incus, distrobuilder, and lxc-ci codebases

---

## Quick Reference

| Document | Description |
|---|---|
| [01-research-analysis.md](01-research-analysis.md) | Architecture analysis, VM pipeline, YAML changes, build infrastructure |
| [02-experimental-validation.md](02-experimental-validation.md) | All 6 experiments: bootstrap, PReP proof, cloud image comparison |
| [03-build-install-guide.md](03-build-install-guide.md) | Step-by-step: build Incus from source, install, configure, launch VM |
| [04-contribution-strategy.md](04-contribution-strategy.md) | PR strategy, checklists, upstream plan |

---

## Executive Summary

### What We Want

Enable building VM images (`disk.qcow2`) for **ppc64le** (POWER) and **s390x** (IBM Z) architectures using distrobuilder, so they can be published on `images.linuxcontainers.org` and used with Incus.

### Current State

| Aspect | ppc64le | s390x |
|---|---|---|
| Container images | ✅ Published | ✅ Published |
| VM images (disk.qcow2) | ❌ Not published | ❌ Not published |
| Incus VM host support | ✅ Supported | ✅ Supported |
| Distrobuilder VM pipeline | ✅ Tested (Ubuntu) | ❌ Untested |

### What Was Proven

| Claim | Status |
|---|---|
| Distrobuilder can build a ppc64le VM image | ✅ **Confirmed** |
| Incus QEMU config works for ppc64le (pseries machine, SLOF firmware) | ✅ **Confirmed** |
| SLOF → GRUB → kernel boot chain works on POWER10 with KVM | ✅ **Confirmed** |
| PReP partition is required for ppc64le VM boot | ✅ **Confirmed** (Experiment 6) |
| `grub-install --target=powerpc-ieee1275` writes correct ELF core image | ✅ **Confirmed** |
| `incus launch` causes host crash on ppc64le | ⚠️ **Observed** (use `init` + `start` separately) |
| VM boots and runs with IPv6 | ✅ **Confirmed** |
| `incus-agent` works inside VM | ❌ **Not yet** (not installed in image) |

### Quick Commands

```bash
# Build VM image
distrobuilder build-incus images/ubuntu.yaml --vm \
  -o image.architecture=ppc64el -o image.release=noble \
  -o source.url=http://ports.ubuntu.com/ubuntu-ports/

# Import and launch
incus image import incus.tar.xz disk.qcow2 --alias ubuntu-ppc64-vm
incus init ubuntu-ppc64-vm vm1 --vm
incus start vm1

# Check boot log
incus console vm1 --show-log
```

---

## Repositories Involved

| Repository | Purpose | Key Directories |
|---|---|---|
| `github.com/lxc/incus` | Incus daemon (QEMU driver) | `internal/server/instance/drivers/driver_qemu.go` |
| `github.com/lxc/distrobuilder` | Image builder | `distrobuilder/vm.go`, `shared/osarch.go`, `shared/definition.go` |
| `github.com/lxc/lxc-ci` | CI/build infrastructure + image YAMLs | `images/*.yaml`, `bin/build-distro`, `bin/jenkins-import-images` |

## Fork Branches

| Fork | Branch | Changes |
|---|---|---|
| `github.com/adilhusain-s/distrobuilder` | `ppc64le-prep` | PReP partition, `DISTROBUILDER_ROOT_DEVICE`/`BOOT_DEVICE` env vars |
| `github.com/adilhusain-s/lxc-ci` | `ppc64le-vm` | Ubuntu YAML ppc64le GRUB post-files action |

## Test Environment

| Detail | Value |
|---|---|
| **Host** | `p10-github-actions1.osuosl.org` |
| **Architecture** | ppc64le (POWER10) |
| **OS** | Fedora 43 (Forty Three) |
| **KVM** | ✅ `/dev/kvm` available |
| **Incus** | v7.1 (built from source) |
| **Distrobuilder** | v3.3.1 (built from source, `ppc64le-prep` branch) |
| **lxc-ci** | Main branch + fork with ppc64le VM YAML changes |

## Key Files

| File | Location |
|---|---|
| Supporting documentation | `/home/adil/incus/docs/ppc64le-vm-support/` |
| Incus source (local) | `/home/adil/incus/incus/` |
| Incus source (on remote) | `/root/incus-src/` |
| Distrobuilder source (on remote) | `/root/distrobuilder/` |
| lxc-ci source (on remote) | `/root/lxc-ci/` |
| VM image (on remote) | `/root/lxc-ci/disk.qcow2` (295 MB) |
| Host state (remote) | VM `vm1` currently RUNNING |
