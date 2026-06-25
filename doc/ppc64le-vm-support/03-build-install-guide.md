# Practical Build & Install Guide — Step by Step

**Status:** Verified working on Fedora 43 ppc64le (POWER10)
**Date:** 2026-06-24
**Environment:** p10-github-actions1.osuosl.org → Fedora 43 ppc64le → POWER10 with KVM

**Belongs to:** [docs/ppc64le-vm-support/](index.md)

---


## Part 3: Practical Build & Install Guide — Step by Step

**Status:** Verified working on Fedora 43 ppc64le (POWER10)
**Date:** 2026-06-24
**Environment:** `p10-github-actions1.osuosl.org` → Fedora 43 ppc64le → POWER10 with KVM

This section documents every command that was actually run to build Incus from source, install it, configure it, build a VM image with distrobuilder, and launch the VM. Use this as a recipe.

---

### 3.1 System Preparation

**Target OS:** Fedora 43 (Adams) ppc64le
**Kernel:** 6.17.10-200.fc42.ppc64le (upgraded via dnf)
**Go:** go1.25.11 (shipped with Fedora 43, no manual Go install needed)

```bash
# Update system
dnf upgrade -y --releasever=43
dnf install -y \
  autoconf automake libtool gcc \
  libuv-devel lxc-devel protobuf-devel \
  sqlite-devel libacl-devel \
  make git rsync \
  btrfs-progs-devel

# Verify Go (Fedora 43 ships Go 1.25+ which meets Incus v7.1's requirement of go 1.25.10)
go version
# → go version go1.25.11 X:nodwarf5 linux/ppc64le
```

### 3.2 Build & Install C Dependencies (raft + cowsql)

Incus needs two C libraries: **raft** (consensus protocol) and **cowsql** (custom SQLite with raft replication).

```bash
export GOPATH=$(go env GOPATH)
mkdir -p $GOPATH/deps
```

#### raft

```bash
cd $GOPATH/deps
git clone --depth=1 https://github.com/cowsql/raft
cd raft
autoreconf -i
./configure
make -j$(nproc)
make install
ldconfig

# Verify
ls -la /usr/local/lib/libraft*
# → libraft.a, libraft.so, libraft.so.0.0.0
pkg-config --cflags --libs raft  # from /usr/local/lib/pkgconfig/raft.pc
```

#### cowsql

```bash
cd $GOPATH/deps
git clone --depth=1 https://github.com/cowsql/cowsql
cd cowsql
autoreconf -i
PKG_CONFIG_PATH="$GOPATH/deps/raft" ./configure
make -j$(nproc) \
  CFLAGS="-I$GOPATH/deps/raft/include/" \
  LDFLAGS="-L$GOPATH/deps/raft/.libs/"
make install
ldconfig

# Verify
ls -la /usr/local/lib/libcowsql*
# → libcowsql.a, libcowsql.so, libcowsql.so.0.0.1

# Ensure pkg-config can find both
export PKG_CONFIG_PATH="/usr/local/lib/pkgconfig:$GOPATH/deps/raft:$GOPATH/deps/cowsql"
pkg-config --cflags --libs cowsql
pkg-config --cflags --libs raft
```

### 3.3 Build Incus from Source

**Source location:** `/home/adil/incus/incus` (local) → synced to `/root/incus-src` on remote

```bash
# From local machine, sync the source (rsync, not git clone)
cd /home/adil/incus
rsync -avz --delete incus/ root@p10-github-actions1.osuosl.org:/root/incus-src/
```

**On the remote host:**

```bash
cd /root/incus-src

# Set environment for build
export GOPATH=$(go env GOPATH)
export PKG_CONFIG_PATH="/usr/local/lib/pkgconfig:$GOPATH/deps/raft:$GOPATH/deps/cowsql"
export CGO_CFLAGS="-I$GOPATH/deps/raft/include/ -I$GOPATH/deps/cowsql/include/"
export CGO_LDFLAGS="-L/usr/local/lib"
export LD_LIBRARY_PATH="/usr/local/lib"
export CGO_LDFLAGS_ALLOW="(-Wl,-wrap,pthread_create)|(-Wl,-z,now)"

# Fix git safe.directory (for VCS stamping in the binary)
git config --global --add safe.directory /root/incus-src

# Build all binaries (client + daemon + tools)
make build
# This runs: go install -v -tags "libsqlite3" ./...
# Which builds cmd/incus and cmd/incusd and all sub-packages

# Verify binaries built
ls -la $(go env GOPATH)/bin/
# → incus (33MB client), incus-benchmark, incus-migrate, incus-simplestreams, lxc-to-incus, etc.
```

**Note:** `make build` uses `go install` which puts binaries in `$GOPATH/bin/`. It builds `cmd/incus/` (client) and all sub-packages, but does NOT build `cmd/incusd/` (daemon) because `go install ./...` only builds packages that can produce a binary, and `cmd/incusd/main.go` depends on the `libsqlite3` build tag. The `make build` target passes this tag correctly. However, if `incusd` is missing from `$GOPATH/bin/`, build it explicitly:

```bash
go build -v -tags "libsqlite3" -o /usr/local/bin/incusd ./cmd/incusd/
```

### 3.4 Install System-Wide + Systemd Service

```bash
# Copy all binaries to /usr/local/bin
cp $(go env GOPATH)/bin/incus* /usr/local/bin/
cp $(go env GOPATH)/bin/lxc-to-incus /usr/local/bin/
cp $(go env GOPATH)/bin/dev_incus-client /usr/local/bin/

# Verify
ls -la /usr/local/bin/incus /usr/local/bin/incusd
/usr/local/bin/incus --help     # → "Command line client for Incus"
/usr/local/bin/incusd --help    # → "The Incus daemon"

# Create systemd service
cat > /etc/systemd/system/incus.service << 'EOF'
[Unit]
Description=Incus daemon
Documentation=https://github.com/lxc/incus
After=network.target

[Service]
ExecStart=/usr/local/bin/incusd --group root
ExecReload=/bin/kill -HUP $MAINPID
Type=simple
KillMode=process
TimeoutStopSec=30
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# Configure library path
echo "/usr/local/lib" > /etc/ld.so.conf.d/local-libs.conf
ldconfig

# Enable and start
systemctl daemon-reload
systemctl enable incus.service
systemctl start incus.service

# Verify
systemctl status incus.service
# → Active: active (running)
```

### 3.5 Initialize Incus

```bash
# Auto-initialize (creates storage pool and managed bridge incusbr2)
incus admin init --auto

# Verify
incus --version
# → Client version: 7.1
# → Server version: 7.1

incus storage list
# → default  btrfs  CREATED

incus network list
# → incusbr2  bridge  YES  CREATED  (has IPv4 + IPv6)
```

### 3.6 Configure Default Profile

```bash
# The auto-init creates a bridge (e.g., incusbr2). Set it as the default NIC.
incus profile device set default eth0 network incusbr2

# Verify
incus profile show default
# → devices:
# →   eth0:  name: eth0  network: incusbr2  type: nic
# →   root:  path: /  pool: default  type: disk
```

### 3.7 Build & Install distrobuilder

```bash
cd /root
git clone https://github.com/lxc/lxc-ci.git
git clone https://github.com/lxc/distrobuilder.git
cd distrobuilder
make
# Binary at: $(go env GOPATH)/bin/distrobuilder

cp $(go env GOPATH)/bin/distrobuilder /usr/local/bin/
```

### 3.8 Apply Distrobuilder Source Modifications (for ppc64le support)

These changes enable the `DISTROBUILDER_ROOT_DEVICE` and `DISTROBUILDER_BOOT_DEVICE` environment variables needed by the YAML `post-files` actions.

**File: `shared/util.go`** — Add constants:

```go
const (
    EnvRootUUID       = "DISTROBUILDER_ROOT_UUID"
    EnvRootPARTUUID   = "DISTROBUILDER_ROOT_PARTUUID"
    EnvRootDevice     = "DISTROBUILDER_ROOT_DEVICE"   // ← ADD
    EnvBootDevice     = "DISTROBUILDER_BOOT_DEVICE"    // ← ADD
    EnvLoopDevice     = "DISTROBUILDER_LOOP_DEVICE"
)
```

**File: `main_incus.go`** — Export the vars in the chroot environment:

```go
// In the env vars section, add:
rootDevEnv := fmt.Sprintf("%s=%s", shared.EnvRootDevice, vm.getLoopDev())
bootDevEnv := fmt.Sprintf("%s=%s", shared.EnvBootDevice, vm.getUEFIDevFile())

// Then include them in the context:
c.global.ctx = context.WithValue(c.global.ctx, shared.ContextKeyEnviron,
    []string{
        rootDevEnv,
        bootDevEnv,
        fmt.Sprintf("%s=%s", shared.EnvRootUUID, rootUUID),
        fmt.Sprintf("%s=%s", shared.EnvRootPARTUUID, partUUID),
        // ... loopEnv
    })
```

These changes are tracked in the `ppc64le-prep` branch of the fork `https://github.com/adilhusain-s/distrobuilder.git`.

### 3.9 Modify Ubuntu YAML for ppc64le VM Support

**File: `lxc-ci/images/ubuntu.yaml`**

Add `grub-ieee1275` package for ppc64el VM builds:

```yaml
- packages:
    - grub-ieee1275
  action: install
  architectures:
    - ppc64el
  types:
    - vm
```

Add `post-files` action for ppc64le GRUB installation (using the env vars from step 3.8):

```yaml
- trigger: post-files
  action: |-
    #!/bin/sh
    set -eux
    # Create device.map for grub-install
    echo "(hd0) ${DISTROBUILDER_ROOT_DEVICE}" > /boot/grub/device.map
    # PReP partition is partition 1
    PREP_DEV="${DISTROBUILDER_ROOT_DEVICE}p1"
    # Install GRUB for PowerPC IEEE1275 (SLOF)
    mknod /dev/nvram c 10 144 || true
    grub-install --target=powerpc-ieee1275 --no-nvram "${PREP_DEV}" || true
    # Generate GRUB config
    grub-mkconfig -o /boot/grub/grub.cfg
    # Fix root UUID
    sed -i "s#root=[^ ]*#root=${DISTROBUILDER_ROOT_UUID}#g" /boot/grub/grub.cfg
  architectures:
    - ppc64el
  types:
    - vm
```

### 3.10 Build Ubuntu ppc64le VM Image

```bash
cd /root/lxc-ci

distrobuilder build-incus images/ubuntu.yaml \
  --vm \
  -o image.architecture=ppc64el \
  -o image.release=noble \
  -o source.url=http://ports.ubuntu.com/ubuntu-ports/

# Output files in current directory:
# → incus.tar.xz    (metadata, ~700 B)
# → disk.qcow2      (VM disk image, ~295 MB)
```

**Key options explained:**

| Option | Value | Why |
|---|---|---|
| `--vm` | — | Tells distrobuilder to build a VM image (creates GPT + partitions + qcow2) |
| `image.architecture` | `ppc64el` | Debian/Ubuntu name for ppc64le (NOT `ppc64le`) |
| `image.release` | `noble` | Ubuntu 24.04 LTS |
| `source.url` | `http://ports.ubuntu.com/ubuntu-ports/` | Required for ppc64el — the default `archive.ubuntu.com` does NOT serve non-x86 architectures |

### 3.11 Build and Install incus-agent (Required for `incus exec`)

The `incus-agent` binary is **not** automatically built during the main `make` step. The agent must be built separately and installed to the host `$PATH` so that incusd can copy it into the VM's config share (9p mount) at VM start time.

```bash
cd /root/incus-src

# Build incus-agent (agent + netgo tags required)
CGO_ENABLED=0 go install -v -tags agent,netgo ./cmd/incus-agent

# The binary is installed to $GOPATH/bin (typically /root/go/bin/)
ls -la /root/go/bin/incus-agent
# → -rwxr-xr-x 1 root root 27814703 ... incus-agent

# Copy to a location in $PATH so incusd can find it at VM start time
cp /root/go/bin/incus-agent /usr/local/bin/
which incus-agent
# → /usr/local/bin/incus-agent
```

**Note:** The `incus-agent` binary is cross-compiled statically (`CGO_ENABLED=0`) and works on ppc64le natively. Once it's in `/usr/local/bin/`, incusd will automatically copy it to the VM's config share on every VM start.

### 3.12 Import and Launch the VM

```bash
cd /root/lxc-ci

# Import the VM image into Incus
incus image import incus.tar.xz disk.qcow2 --alias ubuntu-ppc64-vm

# Verify
incus image list
# → ubuntu-ppc64-vm  ppc64le  VIRTUAL-MACHINE  294.81MiB

# Launch the VM
# NOTE: 'incus launch' (combined init+start) may crash the host.
# Use separate 'init' then 'start' instead:
incus init ubuntu-ppc64-vm vm1 --vm
incus start vm1

# Check status
incus list
# → vm1  RUNNING  VIRTUAL-MACHINE (IPv6 assigned)

incus info vm1
# → PID, Memory usage, Network details

# View boot log
incus console vm1 --show-log
# → SLOF → "Successfully loaded" → Welcome to GRUB! → Linux kernel → systemd
```

### 3.13 Troubleshooting Known Issues

#### Issue: `incus launch` crashes the host

`incus launch` (which does `init`+`start` in one operation) consistently causes the PPC64LE host to crash/reboot. **Workaround:** Use separate commands:
```bash
# Instead of:
# incus launch ubuntu-ppc64-vm vm1 --vm   # ← CRASHES

# Use:
incus init ubuntu-ppc64-vm vm1 --vm
incus start vm1
```

#### Issue: `incus exec` fails with "VM agent isn't currently running"

The VM boots but the `incus-agent` inside the VM isn't installed or running. **Two conditions** must be met:

1. **Host side:** `incus-agent` binary must be in `$PATH` on the host (e.g., `/usr/local/bin/incus-agent`). Build it with:
   ```bash
   cd /root/incus-src && CGO_ENABLED=0 go install -v -tags agent,netgo ./cmd/incus-agent
   cp /root/go/bin/incus-agent /usr/local/bin/
   ```
   After installing, **restart the VM** so incusd re-generates the config share with the binary.

2. **VM side (automatic):** The VM must have the `incus-agent` systemd service and udev rules installed. This happens automatically on first boot — incusd exports the config share as a 9p mount, the `incus-agent-setup` script copies the binary to `/run/incus_agent/`, and the udev rule triggers `incus-agent.service` when the virtio serial port `org.linuxcontainers.incus` appears.

**Diagnostic commands (inside the VM):**
```bash
# Check if incus-agent is running
systemctl status incus-agent

# Check the vsock device (should exist on ppc64le)
ls -la /dev/vsock /dev/vhost-vsock

# Check virtio serial ports
cat /sys/class/virtio-ports/*/name
# → Should show: org.linuxcontainers.incus

# Check the agent config share mount
ls -la /run/incus_agent/
# → Should contain: incus-agent, agent.conf, agent.crt, server.crt, systemd/
```

#### Issue: QEMU SPICE not available

On Fedora ppc64le, `qemu-system-ppc64` may not have SPICE compiled in. This prevents `incus start --console` from working. The VM still boots and runs; use the serial console log (`incus console vm1 --show-log`) instead.

#### Issue: GRUB `grub-install` fails in chroot

Inside the distrobuilder build chroot, `/dev/nvram` doesn't exist, causing `grub-install` to fail. The fix is:
```bash
mknod /dev/nvram c 10 144 || true
grub-install --target=powerpc-ieee1275 --no-nvram "${PREP_DEV}" || true
```
The `--no-nvram` flag skips the NVRAM write and `|| true` ensures the build continues even if `grub-install` has issues writing to the PReP partition.

---

*End of Part 3. Practical build, install, and launch guide complete.*
