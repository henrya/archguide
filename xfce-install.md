## Step 1 (Installation media)

1. Connect to Wifi (if not wired)
```
iwctl
[iwd]# station wlan0 get-networks
[iwd]# station wlan0 connect <NAME>
[iwd]# exit
ping 1.1.1.1
```

2. Sync arch packages
```
pacman -Syy
```

3. Create partitions using `cfdisk` (recommended) or `fdisk`.
*   Note: This guide assumes a UEFI system.
*   Partition 1: EFI System Partition (at least 512MB, type `EFI System`)
*   Partition 2: Root Partition (remaining space, type `Linux filesystem`)

4. Create filesystems
```
mkfs.fat -F 32 /dev/nvme0n1p1 # efi partition
mkfs -t ext4 /dev/nvme0n1p2   # main partition
```

5. Mount partitions
```
mount /dev/nvme0n1p2 /mnt
mkdir -p /mnt/boot/efi        # create efi folder
mount /dev/nvme0n1p1 /mnt/boot/efi
```

6. Install main packages
*   Using `linux-lts` is recommended for stability.
```
pacstrap -K /mnt base linux linux-firmware linux-headers linux-lts linux-lts-headers sudo vim nano base-devel
genfstab -U /mnt > /mnt/etc/fstab
```

7. Chroot into the new system:
```
arch-chroot /mnt
```

8. Enable swapfile (Optional but recommended)
*   **Note:** If using Btrfs, `fallocate` cannot be used for swapfiles. Use `dd` instead or follow Btrfs specific guides.
```
dd if=/dev/zero of=/swapfile bs=1M count=16384 status=progress
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' | tee -a /etc/fstab
```

9. System locale - uncomment desired locales in `/etc/locale.gen` (e.g., `en_US.UTF-8 UTF-8`):
```
nano /etc/locale.gen
locale-gen
```

10. Configure timezone:
```
echo "LANG=en_US.UTF-8" > /etc/locale.conf
ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
hwclock --systohc
```

11. Set keyboard layout (jp106 for Japanese, skip or set `us` for US):
```
echo "KEYMAP=jp106" > /etc/vconsole.conf 
```

12. Networking Setup
*   Set hostname:
```
echo "archlinux" > /etc/hostname
```
*   Edit hosts file:
```
nano /etc/hosts
# 127.0.0.1    localhost
# ::1          localhost
# 127.0.0.1    archlinux
```

13. User Setup
```
useradd -m -g users -G wheel -s /bin/bash henrya
passwd              # Set root password
passwd henrya       # Set user password
```
*   Enable sudo for wheel group:
```
EDITOR=nano visudo
# Uncomment: %wheel ALL=(ALL) ALL
```

14. Install and configure Bootloader (GRUB)
```
pacman -S grub efibootmgr os-prober mtools dosfstools
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=Arch --modules="tpm" --disable-shim-lock
grub-mkconfig -o /boot/grub/grub.cfg
mkinitcpio -P
```

15. Secure Boot (Optional, via `sbctl`)
*   Only proceed if you are in Setup Mode in BIOS.
```
pacman -S sbctl
sbctl status
sbctl create-keys
sbctl enroll-keys -m
sbctl sign -s /boot/efi/EFI/Arch/grubx64.efi
sbctl sign -s /boot/vmlinuz-linux
sbctl sign -s /boot/vmlinuz-linux-lts
```

16. Install Network Manager
*   **Important:** Do not install or enable `dhcpcd` alongside NetworkManager to avoid conflicts.
```
pacman -S networkmanager openssh
systemctl enable NetworkManager
systemctl enable sshd
```

17. Enable TRIM (for SSDs)
```
systemctl enable fstrim.timer
```

18. Exit and Reboot
```
exit
umount -R /mnt
reboot
```

## Step 2 (Post-Installation)

1. Connect to WiFi
```
nmcli device wifi connect <SSID> password <password>
```

2. Enable NTP
```
sudo timedatectl set-ntp true
```

3. Install Xorg and Video Drivers
*   **Note:** `xf86-video-intel` is generally discouraged for modern Intel CPUs (Gen 3+). The `modesetting` driver (built-in) is preferred.
```
sudo pacman -S xorg-server xorg-apps
```
*   (Optional) If you specifically need Intel drivers for older hardware: `sudo pacman -S xf86-video-intel`

4. Install XFCE Desktop
```
sudo pacman -S xfce4 xfce4-goodies file-roller network-manager-applet leafpad galculator lightdm lightdm-gtk-greeter lightdm-gtk-greeter-settings capitaine-cursors papirus-icon-theme xdg-user-dirs-gtk dbus gvfs
```

5. Enable Display Manager
```
sudo systemctl enable lightdm
```

6. Setup Bluetooth
```
sudo pacman -S bluez bluez-utils blueman
sudo systemctl enable bluetooth
```

7. Setup Audio (Pipewire)
```
sudo pacman -S pipewire pipewire-pulse wireplumber pavucontrol
systemctl --user --now enable pipewire pipewire-pulse wireplumber
```

8. Auto-mount USB storage
```
sudo pacman -S udisks2
systemctl enable udisks2
```

9. Install Fonts
```
sudo pacman -S noto-fonts noto-fonts-extra noto-fonts-emoji ttf-ubuntu-font-family ttf-dejavu ttf-liberation ttf-droid ttf-roboto terminus-font
```

10. Useful Utilities
```
sudo pacman -S intel-ucode git bash-completion lshw zip unzip htop inxi iftop wget net-tools rsync ethtool chromium
```

11. Hardware Acceleration (VA-API for Intel)
```
sudo pacman -S libva-utils intel-media-driver intel-gpu-tools
```

12. Printing Support
```
sudo pacman -S cups cups-filters cups-pdf system-config-printer
sudo systemctl enable cups
```

13. Reboot to Graphical Interface
```
reboot
```

## Step 3 (Enable Hibernation)

1. Get UUIDs
```
lsblk -f
# Identify the UUID of your swap partition/file and root partition
```

2. Edit GRUB
```
sudo nano /etc/default/grub
```
*   Append to `GRUB_CMDLINE_LINUX_DEFAULT`:
    `resume=UUID=<swap_partition_uuid>` 
    *   (Or if using swapfile on root: `resume=UUID=<root_partition_uuid> resume_offset=<offset>`)

3. Calculate Swapfile Offset (if using swapfile)
```
sudo filefrag -v /swapfile | awk '$1=="0:" {print substr($4, 1, length($4)-2)}'
```

4. Regenerate GRUB
```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

5. Edit `mkinitcpio.conf`
```
sudo nano /etc/mkinitcpio.conf
```
*   Add `resume` hook **AFTER** `udev` and **BEFORE** `filesystems`.
*   Example: `HOOKS=(base udev autodetect modconf block resume filesystems keyboard fsck)`

6. Regenerate Initramfs
```
sudo mkinitcpio -P
```

## Step 4 (Graphical Boot Splash - Plymouth)

1. Install Plymouth
```
sudo pacman -S plymouth
```

2. Configure Theme
```
sudo nano /etc/plymouth/plymouthd.conf
# [Daemon]
# Theme=bgrt
```

3. Edit `mkinitcpio.conf`
*   Add `plymouth` hook **AFTER** `base` and `udev`.
*   Example: `HOOKS=(base udev plymouth autodetect ...)`

4. Regenerate Initramfs
```
sudo mkinitcpio -P
```

5. Edit GRUB for Silent Boot
```
sudo nano /etc/default/grub
# Add 'quiet splash loglevel=3 rd.udev.log_priority=3 vt.global_cursor_default=0' to GRUB_CMDLINE_LINUX_DEFAULT
```

6. Regenerate GRUB
```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

## Step 5 (Optional - AUR & Japanese)

1. Install `yay` (AUR Helper)
```
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

2. Install Japanese Fonts & IME
```
sudo pacman -S noto-fonts-cjk adobe-source-han-sans-jp-fonts adobe-source-han-serif-jp-fonts otf-ipafont
# Consider installing fcitx5-im and fcitx5-mozc for input
```
