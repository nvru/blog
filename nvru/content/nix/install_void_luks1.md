+++
title = "Void Linux: LUKS1 + Btrfs Guide"
date = "2026-08-12"
author = "nvru"
description = "A step-by-step guide for installing Void Linux with LUKS1 full-disk encryption and Btrfs subvolumes."
tags = ["void", "linux", "luks", "btrfs"]
draft = false
+++

<!-- # Void Linux Installation Guide (LUKS1 + Btrfs Subvolumes) -->

A complete guide for installing Void Linux with **LUKS1 full-disk encryption**, a **Btrfs subvolume layout** (Timeshift-compatible), and single-passphrase GRUB unlocking.

---

## 1. Disk Identification & Partitioning

Identify target block devices:

```bash
lsblk
```

Partition the target disk using `fdisk` (or `cfdisk` for a TUI interface):

```bash
fdisk /dev/sdX
```

### Partition Scheme (UEFI vs. BIOS)

#### UEFI System (GPT Scheme)

- **Create GPT Partition Table:** Command `g`
- **Create EFI System Partition (ESP) (128M - 1G recommended):**
  - Command `n` -> Partition `1` -> First sector: `Default` -> Last sector: `+200M`
  - Wipe signature if prompted: `Y`
  - Change partition type: Command `t` -> Type `1` (EFI System)
- **Create Root Partition:**
  - Command `n` -> Partition `2` -> First sector: `Default` -> Last sector: `Default` (Use remaining space)
- **Write Changes:** Command `w`

#### BIOS / Legacy System (MBR Scheme)

- **Create MBR Partition Table:** Command `o`
- **Create Root Partition:**
  - Command `n` -> Primary -> Partition `1` -> First sector: `Default` -> Last sector: `Default` (Use entire disk)
  - Set bootable flag: Command `a` -> Partition `1`
- **Write Changes:** Command `w`

---

## 2. Encryption & File System Creation

Encrypt the root partition using **LUKS1**.

> [!IMPORTANT]
> GRUB requires LUKS1 for direct header decryption. LUKS2 is not natively supported by standard GRUB builds without custom patches.

- On **UEFI systems**, the partition to encrypt is `/dev/sdX2`.
- On **BIOS systems**, the partition to encrypt is `/dev/sdX1`.

```bash
cryptsetup luksFormat --type luks1 -y /dev/sdX2
```

_Confirm with `YES` (all caps)._

Open the encrypted volume (mapped target name `Void` can be changed at any time):

```bash
cryptsetup open /dev/sdX2 Void
```

Format the partitions:

```bash
# EFI Partition (UEFI systems only)
mkfs.vfat -F32 -n EFI /dev/sdX1

# Encrypted Root Partition (Both UEFI & BIOS)
mkfs.btrfs -L Void /dev/mapper/Void
```

---

## 3. Btrfs Subvolume Layout

Mount the root Btrfs volume to generate subvolumes. The `@` prefix convention ensures full compatibility with automated snapshot utilities like **Timeshift**:

```bash
mount /dev/mapper/Void /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@opt
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@libvirt
btrfs subvolume create /mnt/@podman
btrfs subvolume create /mnt/@docker

umount /mnt
```

---

## 4. Subvolume Mounts & Mount Options

Select performance flags based on hardware type.

_Reference:_ [Btrfs Mount Options Documentation](https://btrfs.readthedocs.io/en/latest/ch-mount-options.html)

**SSD Target:**

```bash
BTRFS_OPTS="rw,noatime,compress=zstd:1,discard=async,ssd,space_cache=v2"
```

**HDD Target:**

```bash
BTRFS_OPTS="rw,noatime,autodefrag,compress=zstd:1,space_cache=v2"
```

Create mounting points and mount the subvolume hierarchy:

```bash
# Root subvolume
mount -o $BTRFS_OPTS,subvol=@ /dev/mapper/Void /mnt

# Subvolume mount points
mkdir -p /mnt/{home,opt,var/log,var/cache,var/lib/libvirt,var/lib/containers,var/lib/docker}

# Mount secondary subvolumes
mount -o $BTRFS_OPTS,subvol=@home /dev/mapper/Void /mnt/home
mount -o $BTRFS_OPTS,subvol=@opt /dev/mapper/Void /mnt/opt
mount -o $BTRFS_OPTS,subvol=@log /dev/mapper/Void /mnt/var/log
mount -o $BTRFS_OPTS,subvol=@cache /dev/mapper/Void /mnt/var/cache
mount -o $BTRFS_OPTS,subvol=@libvirt /dev/mapper/Void /mnt/var/lib/libvirt
mount -o $BTRFS_OPTS,subvol=@podman /dev/mapper/Void /mnt/var/lib/containers
mount -o $BTRFS_OPTS,subvol=@docker /dev/mapper/Void /mnt/var/lib/docker

# ESP Mount (UEFI systems only)
mkdir -p /mnt/efi
mount -o rw,noatime /dev/sdX1 /mnt/efi
```

Verify current mounts:

```bash
lsblk
```

---

## 5. Base System Bootstrap

Select a mirror from [Void Linux Mirror Status](https://xmirror.voidlinux.org).

Supported Architectures: `x86_64`, `x86_64-musl`, `i686`, `aarch64`.  
Repository paths: `glibc` -> `/current` | `musl` -> `/current/musl`

Set targets:

```bash
REPO=https://repo-default.voidlinux.org/current
ARCH=x86_64
```

Copy RSA signing keys to prevent signature validation prompts during package installation:

```bash
mkdir -p /mnt/var/db/xbps/keys
cp /var/db/xbps/keys/* /mnt/var/db/xbps/keys/
```

Bootstrap core system packages (`linux-mainline` chosen for the latest stable kernel branch):

```bash
XBPS_ARCH=$ARCH xbps-install -S -R "$REPO" -r /mnt base-system linux-mainline btrfs-progs cryptsetup vim
```

Chroot into the installed system:

```bash
xchroot /mnt
```

---

## 6. System Configuration

### Permissions & System Accounts

```bash
chown root:root /
chmod 755 /
passwd root
```

### Hostname

```bash
echo "Void" > /etc/hostname
```

### Locales & Timezone

**For glibc systems:** Enable target locale in `/etc/default/libc-locales`:

```bash
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "en_US.UTF-8 UTF-8" >> /etc/default/libc-locales
xbps-reconfigure -f glibc-locales
```

Set timezone:

```bash
ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
```

### Network Hosts

```bash
cat <<EOF > /etc/hosts
127.0.0.1        localhost
::1              localhost
127.0.1.1        Void.localdomain Void
EOF
```

### User Creation & Privilege Escalation

Create user account:

```bash
useradd -m -G wheel <USER>
passwd <USER>

chsh -s /bin/bash root
```

#### Option A: Install & Configure `sudo`

```bash
xbps-install sudo
EDITOR=vim visudo
```

Uncomment: `%wheel ALL=(ALL:ALL) ALL`

#### Option B: Install & Configure `opendoas`

```bash
xbps-install opendoas
```

Create `/etc/doas.conf`:

```bash
cat <<EOF > /etc/doas.conf
permit persist :wheel
EOF
```

Set correct permissions for `doas.conf`:

```bash
chown root:root /etc/doas.conf
chmod 0400 /etc/doas.conf
```

---

## 7. File System Table (`/etc/fstab`)

Export partition UUIDs:

- **For UEFI systems:**
  ```bash
  EFI_UUID=$(blkid -s UUID -o value /dev/sdX1)
  LUKS_UUID=$(blkid -s UUID -o value /dev/sdX2)
  ROOT_UUID=$(blkid -s UUID -o value /dev/mapper/Void)
  ```
- **For BIOS systems:**
  ```bash
  LUKS_UUID=$(blkid -s UUID -o value /dev/sdX1)
  ROOT_UUID=$(blkid -s UUID -o value /dev/mapper/Void)
  ```

Generate `/etc/fstab`:

- **UEFI Setup:**

  ```bash
  cat <<EOF > /etc/fstab
  UUID=$ROOT_UUID / btrfs $BTRFS_OPTS,subvol=@ 0 1
  UUID=$ROOT_UUID /home btrfs $BTRFS_OPTS,subvol=@home 0 2
  UUID=$ROOT_UUID /opt  btrfs $BTRFS_OPTS,subvol=@home 0 2
  UUID=$ROOT_UUID /var/cache btrfs $BTRFS_OPTS,subvol=@cache 0 2
  UUID=$ROOT_UUID /var/log btrfs $BTRFS_OPTS,subvol=@log 0 2
  UUID=$ROOT_UUID /var/lib/libvirt btrfs $BTRFS_OPTS,subvol=@libvirt 0 2
  UUID=$ROOT_UUID /var/lib/containers btrfs $BTRFS_OPTS,subvol=@podman 0 2
  UUID=$ROOT_UUID /var/lib/docker btrfs $BTRFS_OPTS,subvol=@docker 0 2
  UUID=$EFI_UUID /efi vfat defaults,noatime 0 2
  tmpfs /tmp tmpfs defaults,nosuid,nodev 0 0
  EOF
  ```

- **BIOS Setup (omit `/efi` entry):**
  ```bash
  cat <<EOF > /etc/fstab
  UUID=$ROOT_UUID / btrfs $BTRFS_OPTS,subvol=@ 0 1
  UUID=$ROOT_UUID /home btrfs $BTRFS_OPTS,subvol=@home 0 2
  UUID=$ROOT_UUID /opt btrfs $BTRFS_OPTS,subvol=@opt 0 2
  UUID=$ROOT_UUID /var/cache btrfs $BTRFS_OPTS,subvol=@cache 0 2
  UUID=$ROOT_UUID /var/log btrfs $BTRFS_OPTS,subvol=@log 0 2
  UUID=$ROOT_UUID /var/lib/libvirt btrfs $BTRFS_OPTS,subvol=@libvirt 0 2
  UUID=$ROOT_UUID /var/lib/containers btrfs $BTRFS_OPTS,subvol=@podman 0 2
  UUID=$ROOT_UUID /var/lib/docker btrfs $BTRFS_OPTS,subvol=@docker 0 2
  tmpfs /tmp tmpfs defaults,nosuid,nodev 0 0
  EOF
  ```

---

## 8. Bootloader (GRUB)

Add these options to `/etc/default/grub`:

```text
GRUB_CMDLINE_LINUX="rd.auto=1 rd.luks.allow-discards"
GRUB_ENABLE_CRYPTODISK=y
```

Or just copy and paste this whole file:

```text
#
# Configuration file for GRUB.
#
GRUB_DEFAULT=0
#GRUB_HIDDEN_TIMEOUT=0
#GRUB_HIDDEN_TIMEOUT_QUIET=false
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="Void"
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=4"
GRUB_CMDLINE_LINUX="rd.auto=1 rd.luks.allow-discards"
GRUB_ENABLE_CRYPTODISK=y
# Uncomment to use basic console
#GRUB_TERMINAL_INPUT="console"
# Uncomment to disable graphical terminal
#GRUB_TERMINAL_OUTPUT=console
#GRUB_BACKGROUND=/usr/share/void-artwork/splash.png
#GRUB_GFXMODE=1920x1080x32
#GRUB_DISABLE_LINUX_UUID=true
#GRUB_DISABLE_RECOVERY=true
# Uncomment and set to the desired menu colors.  Used by normal and wallpaper
# modes only.  Entries specified as foreground/background.
#GRUB_COLOR_NORMAL="light-blue/black"
#GRUB_COLOR_HIGHLIGHT="light-cyan/blue"
```

Install and target GRUB based on system architecture:

- **For UEFI Systems:**

  ```bash
  xbps-install grub-x86_64-efi
  grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id="Void"
  ```

- **For BIOS Systems:**
  ```bash
  xbps-install grub
  grub-install /dev/sdX
  ```

---

## 9. Automated LUKS Unlock (Initramfs Keyfile)

To avoid entering the LUKS passphrase twice (once in GRUB and once during initramfs stage), generate a keyfile for secondary unlock.

Generate keyfile from random bytes:

```bash
dd bs=512 count=4 if=/dev/urandom of=/boot/keyfile.bin
```

Embed keyfile into LUKS key slot (use `/dev/sdX2` for UEFI or `/dev/sdX1` for BIOS):

```bash
cryptsetup -v luksAddKey /dev/sdX2 /boot/keyfile.bin
```

Restrict keyfile and `/boot` directory access:

```bash
chmod 000 /boot/keyfile.bin
chmod -R g-rwx,o-rwx /boot
```

Configure `/etc/crypttab`:

```bash
cat <<EOF >> /etc/crypttab
Void UUID=$LUKS_UUID /boot/keyfile.bin luks
EOF
```

Configure Dracut to embed the keyfile and crypttab into the initramfs image:

```bash
echo 'install_items+=" /boot/keyfile.bin /etc/crypttab "' > /etc/dracut.conf.d/10-crypt.conf
```

---

## 10. Core Runtime Services

### D-Bus (required for networking & many desktop services)

```bash
xbps-install dbus
ln -s /etc/sv/dbus /etc/runit/runsvdir/default/
```

### NetworkManager

```bash
xbps-install NetworkManager
ln -s /etc/sv/NetworkManager /etc/runit/runsvdir/default/
```

### Audio: PipeWire + PulseAudio compatibility (recommended)

For most desktop setups, use **PipeWire** with its **PulseAudio compatibility layer** to maximize app compatibility. WirePlumber is used as the session manager.

1. **Install packages:**

   ```bash
   xbps-install pipewire wireplumber pulseaudio
   ```

   > _Note:_ If `pulseaudio` is already running, stop/remove it first. The goal is to have PipeWire provide the PulseAudio interface.

2. **Enable WirePlumber (session manager):**

   - **System-wide:**

     ```bash
     mkdir -p /etc/pipewire/pipewire.conf.d
     ln -s /usr/share/examples/wireplumber/10-wireplumber.conf /etc/pipewire/pipewire.conf.d/
     ```

   - **Per-user:**
     ```bash
     : "${XDG_CONFIG_HOME:=${HOME}/.config}"
     mkdir -p "${XDG_CONFIG_HOME}/pipewire/pipewire.conf.d"
     ln -s /usr/share/examples/wireplumber/10-wireplumber.conf "${XDG_CONFIG_HOME}/pipewire/pipewire.conf.d/"
     ln -s /usr/share/examples/wireplumber/20-pipewire-pulse.conf "${XDG_CONFIG_HOME}/pipewire/pipewire.conf.d/"
     ```

3. **Enable the PipeWire PulseAudio interface (for app compatibility):**

   Most apps still expect PulseAudio. Enable PipeWire's PulseAudio shim:

   - **System-wide:**

     ```bash
     mkdir -p /etc/pipewire/pipewire.conf.d
     ln -s /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
     ```

   - **Per-user:**
     ```bash
     : "${XDG_CONFIG_HOME:=${HOME}/.config}"
     mkdir -p "${XDG_CONFIG_HOME}/pipewire/pipewire.conf.d"
     ln -s /usr/share/examples/pipewire/20-pipewire-pulse.conf "${XDG_CONFIG_HOME}/pipewire/pipewire.conf.d/"
     ```

### Polkit (desktop users only)

Polkit (`polkitd`) is recommended for desktop environments that need privilege escalation prompts. **Skip this on headless/servers** unless a specific service requires it.

```bash
xbps-install polkit
ln -s /etc/sv/polkitd /etc/runit/runsvdir/default/
```

---

## 11. Final Reconfiguration & System Exit

Reconfigure installed packages to trigger initramfs generation and update GRUB targets:

```bash
xbps-reconfigure -fa
```

Unmount targets and exit chroot:

```bash
exit
umount -Rl /mnt
reboot
```
