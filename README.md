# Arch Linux (Gnome) Installation with BTRFS & Post-Install Guide

A highly optimized, clean, and modern Arch Linux installation workflow featuring **Btrfs subvolumes**, **EFISTUB booting**, **GNOME Desktop Environment**, **Snapper snapshot tracking**, a **zram memory-compressed swap configuration**, and an **automated firmware/kernel backup system**.

---

## Part 1: Partitioning & Btrfs Subvolume Setup

### 1. Drive Partitioning
Launch `gdisk` to wipe and partition your target NVMe drive:
```bash
gdisk /dev/nvme0n1
```
* Create a **1GB** EFI System Partition (`/dev/nvme0n1p1`, type `EF00`).
* Create your Btrfs Root Partition (`/dev/nvme0n1p2`, type `8300`) using the remaining space.

### 2. Format the Filesystems
Format the newly created partitions:
```bash
mkfs.fat -F32 /dev/nvme0n1p1
mkfs.btrfs -f /dev/nvme0n1p2
```

### 3. Connect to the Internet
Establish your network connection before building the filesystem tree:
```bash
iwctl
# Inside iwctl prompt:
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect "Name of Network/WiFi"
exit
```

### 4. Create the Subvolume Matrix
Mount the root filesystem temporarily to generate your clean subvolume layout:
```bash
mount /dev/nvme0n1p2 /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@pkg
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@snapshots
umount /mnt
```

### 5. Mount Options & Directory Tree
Mount the subvolumes with optimized parameters (`noatime` and `compress=zstd`) and assemble the absolute path tree:
```bash
mount -o noatime,compress=zstd,subvol=@ /dev/nvme0n1p2 /mnt

mkdir -p /mnt/{home,var/cache/pacman/pkg,var/log,.snapshots,boot}

mount -o noatime,compress=zstd,subvol=@home /dev/nvme0n1p2 /mnt/home
mount -o noatime,compress=zstd,subvol=@pkg /dev/nvme0n1p2 /mnt/var/cache/pacman/pkg
mount -o noatime,compress=zstd,subvol=@log /dev/nvme0n1p2 /mnt/var/log
mount -o noatime,compress=zstd,subvol=@snapshots /dev/nvme0n1p2 /mnt/.snapshots

mount /dev/nvme0n1p1 /mnt/boot
```

---

## Part 2: Base System Installation & Chroot Configuration

### 1. Pacstrap & System Table Generation
Refresh repositories, install the foundational packages, and generate your persistent filesystem table:
```bash
pacman -Syy
pacstrap /mnt base linux linux-firmware base-devel neovim amd-ucode efibootmgr btrfs-progs snapper snap-pac zram-generator
genfstab -U /mnt >> /mnt/etc/fstab
```

### 2. Enter System Chroot
Pivot into your new installation to finalize configurations:
```bash
arch-chroot /mnt
```

### 3. Secure the Boot Partition
Tighten mount security flags for the EFI System Partition:
```bash
nvim /etc/fstab
```
Locate the `/boot` mount entry and update its mount options specifically to match your preferred settings:
```text
UUID=XXXX-XXXX /boot vfat rw,relatime,fmask=0137,dmask=0027,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro 0 2
```
Reload the systemd daemon layout to read changes:
```bash
systemctl daemon-reload
```

### 4. Localization & Core Networking
Configure system locales and specify the permanent machine hostname:
```bash
nvim /etc/locale.gen
# Uncomment: en_US.UTF-8 UTF-8

locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
export LANG=en_US.UTF-8
echo DesktopName > /etc/hostname

nvim /etc/hosts
```

### 5. Root Password
Set a secure administrative password for your root user account:
```bash
passwd
```

---

## Part 3: Native EFISTUB Configuration & Backup Automation

### 1. Register the EFISTUB Entry
Extract the exact hardware `PARTUUID` of your Btrfs block device partition (`/dev/nvme0n1p2`):
```bash
blkid -s PARTUUID -o value /dev/nvme0n1p2
```
Using that output string, issue your custom unified kernel command-line entry straight to your motherboard NVRAM:
```bash
efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "Arch Linux" --loader /vmlinuz-linux --unicode 'root=PARTUUID=output-of-blkid-line rootflags=subvol=@ rw quiet loglevel=3 rd.systemd.show_status=false rd.udev.log_level=3 initrd=\amd-ucode.img initrd=\initramfs-linux.img'

# Verify the slot output looks correct
efibootmgr
```

### 2. Establish NVRAM and Firmware Backups
Create an isolation directory inside your Btrfs root to harbor working copies of your kernel, microcode, and NVRAM arrangements:
```bash
mkdir -p /etc/efi-backups
nvim /usr/local/bin/backup-efistub.sh
```
Write the tracking script contents completely:
```bash
#!/bin/bash
BACKUP_DIR="/etc/efi-backups"
cp /boot/vmlinuz-linux "$BACKUP_DIR/vmlinuz-linux.bak"
cp /boot/initramfs-linux.img "$BACKUP_DIR/initramfs-linux.bak.img"
cp /boot/initramfs-linux-fallback.img "$BACKUP_DIR/initramfs-linux-fallback.bak.img"
cp /boot/amd-ucode.img "$BACKUP_DIR/amd-ucode.bak.img"
efibootmgr -v > "$BACKUP_DIR/efibootmgr-entries.txt"
```
Make the new backup module fully executable:
```bash
chmod +x /usr/local/bin/backup-efistub.sh
```

### 3. Link Pacman Hooks
Tie the newly written firmware protection layout straight to Pacman's lifecycle:
```bash
mkdir -p /etc/pacman.d/hooks
nvim /etc/pacman.d/hooks/efistub-backup.hook
```
Drop in the system trigger context:
```ini
[Trigger]
Operation = Upgrade
Operation = Install
Type = Package
Target = linux
Target = amd-ucode

[Action]
Description = Backing up EFISTUB kernel images and NVRAM entries...
When = PostTransaction
Exec = /usr/local/bin/backup-efistub.sh
```

---

## Part 4: User Profile Setup & Environment Build out

### 1. User Provisioning
Install sudo tools and build out your primary work identity with standard group rules:
```bash
pacman -S sudo
useradd -m yourusername
passwd yourusername
usermod -aG wheel,audio,video,storage yourusername

# Configure elevation rights safely
EDITOR=nvim visudo
# Uncomment: %wheel ALL=(ALL:ALL) ALL
```

### 2. Package Manager Tuning
Modify your standard mirrors and parameters (e.g., enabling color, parallel downloads, or multilib tracking):
```bash
nvim /etc/pacman.conf
pacman -Syu
```

### 3. Bulk Software Stack Sync
Pull down your complete package manifest containing drivers, core CLI dependencies, your GNOME ecosystem, and graphical tool preferences:
```bash
pacman -S mesa lib32-mesa vulkan-radeon lib32-vulkan-radeon libva-utils git wget xdg-utils ripgrep xclip go npm lua51 luarocks dosfstools ntfsprogs fastfetch noto-fonts noto-fonts-cjk nerd-fonts tree-sitter-cli speech-dispatcher
pacman -S gnome extension-manager gnome-tweaks fragments amberol networkmanager bluez bluez-utils firefox kitty resources
pacman -S steam discord electron texlive-meta
```

### 4. Configure Snapper Structure
Since Snapper requires direct authority over the execution of snapshots, we temporarily separate the nested subvolume, let the configuration engine mount its root path, and hook up permissions:
```bash
umount /.snapshots
rm -rf /.snapshots
snapper --no-dbus -c root create-config /
btrfs subvolume delete /.snapshots
mkdir /.snapshots
mount -a
chown -R :wheel /.snapshots
```
Refine permissions and execution patterns by tweaking your snapper config file:
```bash
nvim /etc/snapper/configs/root
```
Ensure these variable structures align within the file:
```text
ALLOW_GROUPS="wheel"
NUMBER_LIMIT="5-10"
NUMBER_LIMIT_IMPORTANT="3-5"
TIMELINE_CREATE="no"
```

### 5. Configure Zram RAM-Compressed Swap Space
Configure `zram-generator` to handle system memory swap optimizations natively inside RAM without breaking Btrfs Copy-on-Write dynamics:
```bash
nvim /etc/systemd/zram-generator.conf
```
Paste the configuration layout to deploy swap sizing proportional to your device RAM:
```ini
[zram0]
zram-size = ram
compression-algorithm = zstd
```

Create `/etc/sysctl.d/99-zram.conf` file (or edit it) to tell the kernel to proactively use your fast zram space rather than letting physical RAM fill completely up
```bash
nvim /etc/sysctl.d/99-zram.conf
```
Add the below config
```ini
# Aggressively favor zram compression over dropping filesystem caches
vm.swappiness = 100

# Tell the kernel to reclaim pagecache and swap space at equal priority
vm.watermark_boost_factor = 0
```

---

## Part 5: AUR Packaging & Final System Polishing

### 1. Profile Shift & AUR Tool Setup
Drop out of root permissions down to your user account profile context to securely deal with the AUR helper generation:
```bash
su -l yourusername
```
Compile and configure `yay`:
```bash
cd ~
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd .. && rm -rf yay
```

### 2. AUR Integration & Lua Modules
Utilize `yay` to install specialized desktop themes or terminal extensions, and deploy your critical development environment plugins via luarocks:
```bash
yay -S gdm-settings nautilus-open-any-terminal proton-mail-bin

# Return directly to administrative root space to complete global installations
exit

luarocks --lua-version=5.1 install dkjson
luarocks --lua-version=5.1 install magick
```

### 3. System Daemons Initialization
Activate standard daemons to run smoothly on bootup:
```bash
systemctl enable gdm.service
systemctl enable NetworkManager.service
systemctl enable bluetooth.service
```

### 4. User Profile Customizations & Environment Optimization
Drop out of the chroot jail back down to the target environment live instance:
```bash
exit
```
Unmount your functional filesystems completely and execute a clean machine reboot:
```bash
umount -R /mnt
reboot
```

**Once your machine boots safely into your clean GNOME installation, finalize your specific application configs:**
```bash
# 1. Update your terminal mapping defaults for Nautilus integration
gsettings set com.github.stunkymonkey.nautilus-open-any-terminal terminal kitty

# 2. Assign Firefox handling structures natively
xdg-settings set default-web-browser firefox.desktop

# 3. Patch Discord check actions inside its config layout
nvim ~/.config/discord/settings.json
# Insert or update entry key: "SKIP_HOST_UPDATE": true

# 4. Audio checking and pipewire restarts (if configs are modified)
systemctl restart --user pipewire.service pipewire-pulse.service
# Check runtime sample rates using: pw-top
```

---

## How to See and Modify Snapshots

### List snapshots 

```bash
sudo snapper -c root list
```

### Delete Snapshots

```bash

# To delete snapshot numbered N
sudo snapper -c root delete N

# To delete snapshots numbered from M to N
sudo snapper -c root delete M-N
```

---

## 🛠️ How to Recover Using This Setup

If a kernel update breaks your system or your EFI partition gets wiped, recovery is straightforward using an Arch ISO:

1. Boot into the Arch Live USB and connect to Wi-Fi (`iwctl`).
2. Mount your Btrfs root and your EFI partition:
   ```bash
   mount -o subvol=@ /dev/nvme0n1p2 /mnt
   mount /dev/nvme0n1p1 /mnt/boot
   ```
3. **To restore a broken kernel:** Simply copy your backups back into `/boot`:
   ```bash
   cp /mnt/etc/efi-backups/* /mnt/boot/
   ```
4. **To restore missing NVRAM entries:** Read the text file backup to see exactly what your parameters were:
   ```bash
   cat /mnt/etc/efi-backups/efibootmgr-entries.txt
   ```
   Then execute your `efibootmgr --create` command again right from the live environment.
