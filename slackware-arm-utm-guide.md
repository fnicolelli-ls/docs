# Slackware ARM in UTM: A Comprehensive Setup Guide

A practical guide to running ARM Slackware Linux as a development environment on Apple Silicon Macs using UTM.
This guide covers installation, configuration, containerization (Docker/kind), VPN connectivity, and AWS/Kubernetes integration.

---

## Table of Contents

1. [Hardware Prerequisites](#hardware-prerequisites)
2. [Initial VM Setup](#initial-vm-setup)
3. [Post-Installation Configuration](#post-installation-configuration)
4. [Shared Folders](#shared-folders)
5. [USB Passthrough (YubiKey)](#usb-passthrough-yubikey)
6. [Time Synchronization](#time-synchronization)
7. [Kernel Recompilation](#kernel-recompilation)
8. [Docker and Container Runtime](#docker-and-container-runtime)
9. [Kubernetes with kind](#kubernetes-with-kind)
10. [AWS SSO and EKS Integration](#aws-sso-and-eks-integration)
11. [Kubeswitch Configuration](#kubeswitch-configuration)
12. [Tips and Tricks](#tips-and-tricks)
13. [Troubleshooting](#troubleshooting)

---

## Hardware Prerequisites

- **Mac with Apple Silicon** (M1/M2/M3/M4)
- **UTM** (latest version from <https://mac.getutm.app/>)
- At least **16GB RAM** on host (6-8GB recommended for VM)
- At least **50GB** disk space for the VM

---

## Initial VM Setup

### Choosing the Virtualization Backend

UTM supports two backends, each with trade-offs:

| Feature | Apple Virtualization (VZ) | QEMU |
|---------|---------------------------|------|

| Performance | Faster, native | Slightly slower |
| USB Passthrough | Limited | Full support |
| Guest Agent | Not supported | Supported |
| Snapshots | Not supported | Supported |
| Shared Folders | VirtioFS | 9p or VirtioFS |

**Recommendation**: Start with **Apple Virtualization** for best performance. Switch to **QEMU** only if you need USB passthrough (e.g., YubiKey) or other advanced features.
For a full guide on installing Slackware ARM in UTM, refer to the [Slackware ARM Howto](https://docs.slackware.com/slackwarearm:inst_sa64_virt).

### Creating the VM (Apple Virtualization)

1. Open UTM → **Create a New Virtual Machine**
2. Select **Virtualize** (not Emulate)
3. Choose **Linux**
4. Browse to the Slackware ARM64 ISO
5. Configure hardware:
   - **RAM**: 6-8GB (with 16GB host RAM, 8GB is the sweet spot)
   - **CPU Cores**: 6-8 (on a 10-core M4, leave some for macOS)
   - **Storage**: 50GB+ (VirtIO recommended)
6. Complete the wizard and boot the installer

### Creating the VM (QEMU Backend)

1. Open UTM → **Create a New Virtual Machine**
2. Select **Virtualize** → **Linux** → **ARM64 (aarch64)**
3. Ensure **EFI Boot** is enabled
4. Machine type should be **virt**
5. For display, use **virtio-ramfb** or **virtio-gpu**
6. Enable **Serial Console** for debugging:
   - Devices → New → Serial → Built-in Terminal

### Serial Console for Debugging

If the graphical display doesn't work, use serial console:

```bash
# At the bootloader, add:
console=ttyAMA0,115200

# Or in GRUB, edit the kernel line to include:
console=ttyAMA0,115200 console=tty0
```

---

## Post-Installation Configuration

### Disk Labeling for Stable Mounts

VirtIO disk order can change between boots. Use labels or UUIDs instead of device names:

```bash
# Show existing labels and UUIDs
lsblk -f
blkid

# Set labels
e2label /dev/vdaX mylabel      # ext4
xfs_admin -L mylabel /dev/vdaX # XFS
swaplabel -L myswap /dev/vdaX  # Swap
```

Update `/etc/fstab`:

```fstab
LABEL=root      /        ext4    defaults        1 1
LABEL=home      /home    ext4    defaults        1 2
LABEL=myswap    none     swap    defaults        0 0
```

### Hardware Clock Configuration

macOS keeps the RTC in localtime. Verify Slackware matches:

```bash
cat /etc/hardwareclock
# Should contain: localtime
```

### Disable vim Backup Files

```bash
# Add to ~/.vimrc
set nobackup
set nowritebackup
set noswapfile
```

---

## Shared Folders

### With Apple Virtualization (VirtioFS)

1. Shut down the VM
2. UTM → VM Settings → **Sharing** → Add a directory
3. Note the tag name (e.g., `shared`)

In the guest:

```bash
# Check if virtiofs is available
grep virtiofs /proc/filesystems

# Mount manually
mkdir -p /mnt/shared
mount -t virtiofs shared /mnt/shared
```

For persistent mounting, add to `/etc/fstab`:

```fstab
shared    /mnt/shared    virtiofs    defaults,nofail    0 0
```

If virtiofs is a module, ensure it loads before fstab:

```bash
echo "modprobe virtiofs" >> /etc/rc.d/rc.modules.local
chmod +x /etc/rc.d/rc.modules.local
```

### With QEMU (9p Filesystem)

Check kernel support:

```bash
zcat /proc/config.gz | grep 9P
# Needed: CONFIG_NET_9P, CONFIG_NET_9P_VIRTIO, CONFIG_9P_FS
```

Mount:

```bash
mkdir -p /mnt/shared
mount -t 9p -o trans=virtio,version=9p2000.L share /mnt/shared
```

Add to `/etc/fstab`:

```fstab
share   /mnt/shared   9p   trans=virtio,version=9p2000.L,nofail   0 0
```

---

## USB Passthrough (YubiKey)

**Note**: USB passthrough requires **QEMU backend**. Apple Virtualization has limited USB support.

### Attach the YubiKey

1. With VM running, look for the USB icon in UTM toolbar
2. Click and attach your YubiKey

Verify in guest:

```bash
lsusb
# Should show: Yubico.com Yubikey 4/5 OTP+U2F+CCID
```

### Configure YubiKey

Create udev rules:

```bash
cat > /etc/udev/rules.d/70-yubikey.rules << 'EOF'
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="1050", ATTRS{idProduct}=="0407", MODE="0660", GROUP="plugdev"
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="1050", MODE="0660", GROUP="plugdev"
SUBSYSTEM=="usb", ATTRS{idVendor}=="1050", MODE="0660", GROUP="plugdev"
EOF

udevadm control --reload-rules
udevadm trigger

# Add your user to plugdev group
usermod -aG plugdev yourusername
```

### Required Packages

Build from SlackBuilds.org or source:

- `libcbor` - CBOR library (dependency for libfido2)
- `libfido2` - FIDO2/U2F library
- `pam-u2f` - PAM module for login/sudo
- `pcsc-lite` - Smartcard daemon
- `ccid` - Smartcard driver
- `yubikey-manager` - `ykman` tool

### Build libcbor (libfido2 dependency)

```bash
git clone https://github.com/PJK/libcbor.git
cd libcbor
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_LIBDIR=lib64 ..
make
sudo make install
```

### Build libfido2

```bash
git clone https://github.com/Yubico/libfido2.git
cd libfido2
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_LIBDIR=lib64 ..
make
sudo make install
sudo ldconfig

# Verify installation
pkg-config --libs --cflags libfido2
```

### Build pam-u2f

```bash
git clone https://github.com/Yubico/pam-u2f.git
cd pam-u2f
autoreconf -i
./configure
make
sudo make install
```

This installs `pam_u2f.so` (usually to `/usr/local/lib/security/`).

Find the actual path:

```bash
find /usr -name "pam_u2f.so" 2>/dev/null
```

### PyOtherSide for Yubico Authenticator

```bash
git clone https://github.com/thp/pyotherside.git
cd pyotherside
qmake-qt5
make
sudo make install

# Verify
ls /usr/lib64/qt5/qml/io/thp/pyotherside/
```

### Register Your YubiKey

```bash
mkdir -p ~/.config/Yubico
pamu2fcfg > ~/.config/Yubico/u2f_keys
# Touch the YubiKey when it blinks

# For a backup key, append:
pamu2fcfg -n >> ~/.config/Yubico/u2f_keys
```

### Configure PAM for YubiKey Authentication

Edit `/etc/pam.d/system-auth`.

```pam
##################
# Authentication #
##################
auth        required      pam_env.so
auth        optional      pam_group.so
auth        sufficient    /usr/local/lib/security/pam_u2f.so
auth        sufficient    pam_unix.so likeauth nullok
-auth       optional      pam_gnome_keyring.so
```

**Important**: Adjust the path to `pam_u2f.so` based on where it was installed on your system.

### Test Before Locking Yourself Out

1. Keep a root terminal open
2. In another terminal, test:

```bash
su - yourusername
```

It should ask for password, then wait for YubiKey touch (if using `required`), or accept either one (if using `sufficient`).

### XFCE/Keyring Integration

For GNOME Keyring auto-unlock with XFCE, ensure these lines are in `/etc/pam.d/system-auth`:

```pam
-auth       optional      pam_gnome_keyring.so
-session    optional      pam_gnome_keyring.so auto_start
```

The keyring will auto-unlock when you authenticate with password + YubiKey.

### 1Password Setup

1Password uses FIDO2/WebAuthn (not OTP):

1. Log into 1Password.com
2. Go to **Profile** → **More Actions** → **Manage Two-Factor Authentication**
3. Add a **Security Key**
4. Follow the prompts and touch the YubiKey when it blinks

### Caveat

Since the YubiKey uses USB passthrough, if UTM doesn't pass it through early enough at boot/login, you might get locked out. Consider:

- Using `sufficient` instead of `required` until you're confident the passthrough is reliable
- Keeping a backup authentication method

---

## Time Synchronization

With Apple Virtualization, the clock may drift when the host sleeps. Use NTP:

```bash
# Enable ntpd at boot
chmod +x /etc/rc.d/rc.ntpd

# Configure for faster sync after wake
# Edit /etc/ntp.conf:
server 0.pool.ntp.org iburst
server 1.pool.ntp.org iburst
server 2.pool.ntp.org iburst

# Start now
/etc/rc.d/rc.ntpd start
```

---

## Kernel Recompilation

Some features require kernel recompilation, such as `CONFIG_CFS_BANDWIDTH` for Kubernetes CPU limits.
This is needed if you plan to use container runtimes like Docker and run Kubernetes clusters with kind.

### Check Current Config

```bash
zcat /proc/config.gz | grep CFS_BANDWIDTH
# If shows "# CONFIG_CFS_BANDWIDTH is not set", you need to rebuild
```

### Rebuild Steps

```bash
# Start from the working config
cd /usr/src/linux
cp /boot/config-armv8-6.12.62 .config
make oldconfig

# Enable specific options
make menuconfig
# Navigate to: General setup → CPU/Task time and stats accounting
# Enable: CPU bandwidth provisioning for FAIR_GROUP_SCHED

# Build (adjust -j for your core count)
make -j8

# Install
sudo cp arch/arm64/boot/Image /boot/Image-armv8-6.12.62-custom
sudo ln -sf Image-armv8-6.12.62-custom /boot/Image-armv8
```

**Note**: On ARM, don't use `grub-mkconfig` as it is not supported.
Edit `/boot/grub/grub.cfg` directly if needed. For most kernel rebuilds with the same version, the existing GRUB config and initrd will work.

---

## Docker and Container Runtime

### Install Docker

```bash
# Download static binary
curl -fsSL https://download.docker.com/linux/static/stable/aarch64/docker-27.4.1.tgz -o docker.tgz
tar xzf docker.tgz
sudo mv docker/* /usr/local/bin/
rm -rf docker docker.tgz

# Create docker group and add yourself
sudo groupadd docker
sudo usermod -aG docker $USER
```

### Create rc.docker Script

```bash
sudo tee /etc/rc.d/rc.docker << 'EOF'
#!/bin/bash

PIDFILE="/var/run/docker.pid"

start() {
    echo "Starting Docker..."
    /usr/local/bin/dockerd &>/var/log/docker.log &
}

stop() {
    echo "Stopping Docker..."
    if [ -f $PIDFILE ]; then
        kill $(cat $PIDFILE) 2>/dev/null
    fi
}

case "$1" in
    start) start ;;
    stop) stop ;;
    restart) stop; sleep 2; start ;;
    *) echo "Usage: $0 {start|stop|restart}" ;;
esac
EOF

sudo chmod +x /etc/rc.d/rc.docker
```

### Start at Boot

Add to `/etc/rc.d/rc.M` (before rc.local):

```bash
if [ -x /etc/rc.d/rc.docker ]; then
  /etc/rc.d/rc.docker start
fi
```

---

## Kubernetes with kind

### Prerequisites

kind requires:

- Docker (or containerd)
- cgroup v2
- `CONFIG_CFS_BANDWIDTH=y` in kernel

### Enable cgroup v2

Edit `/etc/rc.d/rc.S` and find or add the cgroups mount section. Ensure `CGROUPS_VERSION=2` or add the kernel parameter:

```bash
# Edit /etc/default/cgroups, and set:
CGROUPS_VERSION=2
```

Verify after reboot:

```bash
mount | grep cgroup
# Should show: cgroup2 on /sys/fs/cgroup type cgroup2
```

### Install kind

```bash
# v0.31.0 is the latest as of Jan 2026
go install sigs.k8s.io/kind@v0.31.0
```

### Create a Cluster

```bash
kind create cluster

# If issues occur, use verbose mode:
kind create cluster --verbosity 7
```

### Install kubectl and k9s

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# k9s (build without CGO to avoid linker issues)
go install github.com/derailed/k9s@latest
```

---

## AWS SSO and EKS Integration

### Configure AWS SSO

Use the modern `sso_session` approach for multiple accounts:

```ini
# ~/.aws/config

[sso-session mycompany]
sso_start_url = https://mycompany.awsapps.com/start
sso_region = eu-west-1
sso_registration_scopes = sso:account:access

[profile eks-staging]
sso_session = mycompany
sso_account_id = 123456789012
sso_role_name = Staging-role
region = eu-west-1
output = json

[profile eks-production]
sso_session = mycompany
sso_account_id = 987654321098
sso_role_name = Production-role
region = eu-west-1
output = json
```

### Login

```bash
# Login once - works for ALL profiles
aws sso login --sso-session mycompany

# Then use any profile
aws eks list-clusters --profile eks-staging
```

---

## Kubeswitch Configuration

Kubeswitch provides easy context switching for multiple Kubernetes clusters.

### Install

```bash
# Download switcher binary for ARM64
# Add to PATH
```

### Configure for EKS

Create `~/.kube/switch-config.yaml`:

```yaml
kind: SwitchConfig
version: v1alpha1

kubeconfigStores:
  - kind: eks
    id: eks-staging
    config:
      awsProfile: eks-staging
      region: eu-west-1

  - kind: eks
    id: eks-production
    config:
      awsProfile: eks-production
      region: eu-west-1

  - kind: filesystem
    paths:
      - ~/.kube/config
```

### Shell Integration

**Critical**: Source kubeswitch BEFORE ble.sh (if using it) to avoid conflicts:

```bash
# ~/.bashrc

# Kubeswitch - BEFORE ble.sh
source <(switcher init bash)
alias s="switch"

# ble.sh - AFTER kubeswitch
source $HOME/.local/share/blesh/ble.sh
```

---

## Tips and Tricks

### Recommended VM Resources (10-core M4)

- **CPU**: 6-8 cores
- **RAM**: 6-8 GB
- **Disk**: 50GB+ VirtIO

### Useful Shell Aliases

```bash
# Add to ~/.bashrc
alias kubectl='kubecolor'
alias k='kubectl'
alias s='switch'
```

### Go Build Issues

If Go builds fail with linker errors (missing gold linker):

```bash
# Use static builds
CGO_ENABLED=0 go install github.com/example/tool@latest
```

### Understanding zram

Slackware may enable zram (compressed RAM swap):

```bash
# Check swap devices
cat /proc/swaps
zramctl

# Disable if not needed
swapoff /dev/zram0
rmmod zram
```

---

## Troubleshooting

### VM Won't Boot After Kernel Change

1. Boot from the recovery option in GRUB (if available)
2. Or interrupt GRUB and edit the kernel line to point to the old kernel:

   ```text
   linux /boot/Image-armv8-6.12.62 root=...
   ```

### Graphics Not Working (QEMU)

Use serial console to debug:

```bash
dmesg | grep -iE 'drm|virtio.*gpu|fb|display'
lsmod | grep virtio
ls -la /dev/fb* /dev/dri/
zcat /proc/config.gz | grep -iE 'DRM_VIRTIO|VIRTIO_GPU'
```

### kind Cluster Fails

1. Check cgroup v2 is enabled:

   ```bash
   mount | grep cgroup2
   ```

2. Check controllers are enabled:

   ```bash
   cat /sys/fs/cgroup/cgroup.subtree_control
   ```

3. Check kernel has CFS_BANDWIDTH:

   ```bash
   zcat /proc/config.gz | grep CFS_BANDWIDTH
   ```

4. Check Docker is running:

   ```bash
   docker ps
   ```

### Shared Folders Not Mounting at Boot

Ensure the module loads before fstab:

```bash
# Check if virtiofs/9p is a module
lsmod | grep virtiofs

# Add to rc.modules.local
echo "modprobe virtiofs" >> /etc/rc.d/rc.modules.local
chmod +x /etc/rc.d/rc.modules.local
```

### Clock Drift After Host Sleep

Ensure ntpd is running with `iburst`:

```bash
/etc/rc.d/rc.ntpd status
grep iburst /etc/ntp.conf
```

---

## Resources

- [Slackware ARM Howto](https://docs.slackware.com/slackwarearm:inst_sa64_virt)
- [UTM Documentation](https://docs.getutm.app/)
- [kind Documentation](https://kind.sigs.k8s.io/)
- [Kubeswitch](https://github.com/danielfoehrKn/kubeswitch)

---

*Guide based on real-world experience running ARM Slackware on Apple Silicon Macs for Kubernetes development work.*
