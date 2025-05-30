# Arch Linux UEFI Installation with Btrfs and systemd-boot (with Explanations)

This guide walks you through a **fully customized, modern Arch Linux install** using:

* UEFI boot mode (not BIOS/Legacy)
* `systemd-boot` as the bootloader
* `btrfs` with subvolumes for root filesystem

It includes **educational explanations**, **command descriptions**, and **flag breakdowns** so you learn *why* you're doing each step.

---

## ✅ Prerequisites

* UEFI-capable system with **Secure Boot disabled**

* Boot Arch ISO in UEFI mode → Check with:

  ```sh
  ls /sys/firmware/efi
  ```

  If this exists, you’re in UEFI mode.

* Internet connection (e.g., with `iwctl` for Wi-Fi):

  ```sh
  iwctl
  device list
  station wlan0 scan
  station wlan0 get-networks
  station wlan0 connect "YourSSID"
  exit
  ```

---

## 🧱 Partitioning

Use GPT and create two partitions:

```sh
cgdisk /dev/nvme0n1
```

### In `cgdisk`:

1. Create a **512 MiB EFI System Partition**

   * Type: `ef00`
   * Label (optional): `EFI`
   * Size: `+512M`

2. Create the rest of the disk as root

   * Type: `8300` (Linux filesystem)

---

## 💾 Formatting

```sh
mkfs.fat -F32 /dev/nvme0n1p1       # EFI partition — must be FAT32
mkfs.btrfs -f /dev/nvme0n1p2       # Root partition — Btrfs format
```

* `-F32`: Format as FAT32 for UEFI firmware compatibility
* `-f`: Force format (safe here since we just created it)

---

## 📁 Create Btrfs Subvolumes

```sh
mount /dev/nvme0n1p2 /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var
btrfs subvolume create /mnt/@tmp
btrfs subvolume create /mnt/@srv
umount /mnt
```

### Why subvolumes?

* Allow **selective snapshotting**
* Separate system files (`@`) from user data (`@home`)
* Keep `/var`, `/tmp`, etc. isolated for performance or rollback reasons

---

## 🔧 Mount Subvolumes with Options

```sh
mount -o noatime,compress=zstd,subvol=@ /dev/nvme0n1p2 /mnt
```

* `noatime`: Skip writing last-access times → reduces disk writes
* `compress=zstd`: Transparent compression → saves space and boosts I/O
* `subvol=@`: Mount the `@` subvolume as the root `/`

Now mount the rest:

```sh
mkdir -p /mnt/{boot,home,var,tmp,srv}
mount -o noatime,compress=zstd,subvol=@home /dev/nvme0n1p2 /mnt/home
mount -o noatime,compress=zstd,subvol=@var  /dev/nvme0n1p2 /mnt/var
mount -o noatime,compress=zstd,subvol=@tmp  /dev/nvme0n1p2 /mnt/tmp
mount -o noatime,compress=zstd,subvol=@srv  /dev/nvme0n1p2 /mnt/srv
```

Mount the EFI partition:

```sh
mount /dev/nvme0n1p1 /mnt/boot
```

---

## 📦 Install the Base System

```sh
pacstrap -K /mnt base linux linux-firmware btrfs-progs neovim git base-devel man-db man-pages less iwctl iwd
```

* `base`: Core Arch Linux userspace tools
* `linux`: The kernel
* `linux-firmware`: Needed firmware blobs
* `btrfs-progs`: CLI tools for Btrfs
* `neovim`, `git`: Useful base tools
* `base-devel`: Needed for building packages (e.g. from the AUR)
* `man-db`, `man-pages`, `less`: For `man` support
* `iwctl`, `iwd`: you won't have a way to connect to wifi on the new install otherwise
* `-K`: Copy host keyring (for pacman trust setup)

---

## 🧾 Generate `fstab`

```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

* `-U`: Use UUIDs for partitions
* `>>`: Append to file

You can verify:

```sh
cat /mnt/etc/fstab
```

---

## 🏠 Chroot into Installed System

```sh
arch-chroot /mnt
```

Gives you a root shell as if you're booted into the system.

---

## 🌍 Timezone and Locale

```sh
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc
```

* `hwclock --systohc`: Sets hardware clock from system time

Edit locale config:

```sh
nvim /etc/locale.gen
```

Uncomment:

```
en_US.UTF-8 UTF-8
en_DK.UTF-8 UTF-8   # Optional: for 24h and dd/mm/yyyy
```

Then:

```sh
locale-gen
```

Set system locale:

```sh
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "LC_TIME=en_DK.UTF-8" >> /etc/locale.conf  # or use C.UTF-8 for ISO format
```

---

## 🖥️ Hostname + `/etc/hosts`

```sh
echo "archbox" > /etc/hostname
cat <<EOF > /etc/hosts
127.0.0.1   localhost
::1         localhost
127.0.1.1   archbox.localdomain archbox
EOF
```

---

## 🔑 Set Root Password

```sh
passwd
```

---

## ⚙️ Mount Runtime Filesystems (Required for `bootctl`)

If you didn’t already:

```sh
mount --bind /dev  /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys  /mnt/sys
mount --bind /run  /mnt/run
mount -t efivarfs efivarfs /mnt/sys/firmware/efi/efivars
```

Then:

```sh
arch-chroot /mnt
```

---

## 🚀 Install systemd-boot

```sh
bootctl install
```

Installs bootloader to EFI partition at `/boot/EFI/systemd/`.

Check:

```sh
bootctl status
```

---

## 📂 Configure systemd-boot

### Loader config

```sh
nvim /boot/loader/loader.conf
```

```ini
default arch
timeout 3
editor no
console-mode max
```

### Entry config

```sh
nvim /boot/loader/entries/arch.conf
```

```ini
title   Arch Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options root=UUID=<uuid-of-btrfs-root> rw rootflags=subvol=@
```

Use `blkid` to get the correct UUID:

```sh
blkid
```

**Important**: use the UUID from your Btrfs partition, **not** the FAT32 EFI partition.

---

## 🔒 Secure Boot?

* Keep Secure Boot **disabled** for now.
* Arch does **not** ship signed bootloaders.
* You can optionally configure `sbctl` later if you want to set up custom Secure Boot keys.

---

✅ Done! You're now ready to reboot into your custom-built Arch system!
