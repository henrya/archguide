## Step 1 (Installation media)

1. Connect to Wifi
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

3. Create partitions using `fdisk` or `cfdisk` utility. You may skip this step if partitions are already created.

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

6. Install main  packages
```
pacstrap -i /mnt base linux linux-firmware linux-headers linux-lts linux-lts-headers sudo vim nano
genfstab -U -p /mnt > /mnt/etc/fstab
```

7. Chroot:
```
arch-chroot /mnt
```

8. Enable swapfile
```
fallocate -l 2G /swapfile
chmod 600 /swapfile
echo '/swapfile none swap sw 0 0' | tee -a /etc/fstab
```

11. System locale - uncomment desired locales in `/etc/locale.gen`:
```
nano /etc/locale.gen
locale-gen
```

12. Configure timezone, set your own:
```
echo "LANG=en_US.UTF-8" > /etc/locale.conf
ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
```

13. Set keyboard layout (jp106 for japanese)
```
echo "KEYMAP=jp106" > /etc/vconsole.conf 
```

14. Sync hwclock
```
hwclock --systohc
```

15. Add the host name in `/etc/hosts`:
```
echo draemon > /etc/hostname
nano /etc/hosts

# 127.0.0.1    localhost
# ::1          localhost
# 127.0.1.1    draemon
```

15. Add new user into group:
```
useradd -m -g users -G wheel -s /bin/bash henrya
```

16. Setup ruser password:
```
passwd
passwd henrya
```

17. Add wheel group in sudoers:
```
nano /etc/sudoers
# uncomment this line in file:
# %wheel ALL=(ALL) ALL
```

18. Install and configure grub:
```
pacman -S grub efibootmgr os-prober mtools dosfstools
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=Arch --modules="tpm" --disable-shim-lock
grub-mkconfig -o /boot/grub/grub.cfg
mkinitcpio -p linux
```

19. Install sbctl (secure boot support)
```
sbctl status
sbctl create-keys
sbctl enroll-keys -m
## sign the efi 
sbctl sign -s /boot/efi/EFI/Arch/grubx64.efi
sbctl sign /boot/vmlinuz-linux
sbctl sign /boot/vmlinuz-linux-lts
```

20. Install networkmanager and related utilities:
```
pacman -S dhcpcd networkmanager resolvconf
systemctl enable sshd
systemctl enable dhcpcd
systemctl enable NetworkManager
systemctl enable systemd-resolved
```

21. Exit chroot, unmount all disks and reboot:
```
exit
umount /mnt/boot/efi
umount /mnt
reboot
```

## Step 2 (Install the system)

1. Enable NTP synchronization
```
timedatectl set-ntp true
```

2. Connect to WiFi
```
nmcli device wifi connect <SSID> password <password>
```

3. Install Xorg:
```
sudo pacman -S --needed xorg xf86-video-intel
```

4. Install Xfce:
```
sudo pacman -S --needed xfce4-goodies file-roller network-manager-applet leafpad epdfview galculator lightdm lightdm-gtk-greeter lightdm-gtk-greeter-settings capitaine-cursors arc-gtk-theme papirus-icon-theme xdg-user-dirs-gtk dbus gvfs
```

5. Enable display and network manager
```
sudo systemctl enable lightdm
sudo systemctl enable NetworkManager
```

6. Setup bluetooth:
```
sudo pacman -S bluez bluez-utils blueman
sudo systemctl enable bluetooth
```

7. Setup sound:
```
sudo pacman -S pipewire pipewire-pulse
sudo pacman -S wireplumber
systemctl --user --now enable pipewire pipewire-pulse pavucontrol wireplumber
```

9. Automatically mount USB devices
```
sudo pacman -S udisks2
cat /etc/udev/rules.d/80-udisks2.rules ## should be empty
sudo cp /usr/lib/udev/rules.d/80-udisks2.rules /etc/udev/rules.d/80-udisks2.rules
```

10. Install essential fonts:
```
sudo pacman -S noto-fonts noto-fonts-extra noto-fonts-emoji ttf-ubuntu-font-family ttf-dejavu ttf-freefont
sudo pacman -S ttf-liberation ttf-droid ttf-roboto terminus-font
sudo pacman -S ttf-bitstream-vera ttf-inconsolata ttf-ubuntu-font-family ttf-dejavu ttf-freefont ttf-linux-libertine
```

11. Install other useful packages:
```
sudo pacman -S intel-ucode git bash-completion base-devel lshw zip unzip htop inxi iftop
sudo pacman -S wget wpa_supplicant net-tools rsync ethtool chromium
```

12. GPU video decoding (vaapi):
```
sudo pacman -S libva-utils intel-media-driver intel-gpu-tools
```

13. Reboot
```
reboot
```

## Step 3 Enable hibernatiobn

1. Get partition uuid:
Open `/etc/fstab` and get the root partition uuid or swap parition uuid.

2. Modify grub configuration
Open grub configuration file:
```
sudo nano /etc/default/grub
```
3. Modify `/etc/default/grub`

Insert `resume=UUID=<uuid of root or swap from /etc/fstab>` after `GRUB_CMDLINE_LINUX_DEFAULT="..."`

4. Get the `resume_offset` if `/swapfile` exists
```
sudo filefrag -v /swapfile | awk '$1=="0:" {print substr($4, 1, length($4)-2)}'
```

5. Add `resume_offset` in `/etc/default/grub` if `/swapfile` exists
Open grub configuration and add offset

```
sudo nano /etc/default/grub
```

Insert `resume_offset=<offset of the /swapfile>`

6. Regenerate grub config:
```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```


7. Edit mkinitcpio configuration
```
sudo nano /etc/mkinitcpio.conf
```

8. Find hooks `HOOKS="base udev autodetect modconf block filesystems keyboard fsck"`

9. After `filesystems` insert hook `resume` 

10. Regenerate initramfs:
```
sudo mkinitcpio -p linux
```

11. Display suspend option in XFCE
```
xfconf-query -c xfce4-session -np '/shutdown/ShowSuspend' -t 'bool' -s 'true'
```

12. Reboot
```
reboot
```

## Step 4 Enable Graphical boot splash screen
1. Install plymouth
```
sudo pacman -S plymouth
```

2.  Edit `/etc/plymouth/plymouthd.conf `and set the new theme 

```
sudo nano /etc/plymouth/plymouthd.conf
```

3. Add the following contents:

```
[Daemon]
Theme=bgrt
```

4. Open grub configuration

```
sudo nano /etc/default/grub
```

5. Change loglevel to avoid verbose output

In  `GRUB_CMDLINE_LINUX_DEFAULT` add or edit the following `loglevel=3`

6. Regenerate grub config:
```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

7. Edit mkinitcpio configuration
```
sudo nano /etc/mkinitcpio.conf
```

8. Find hooks `HOOKS="base udev autodetect modconf block filesystems keyboard fsck"`

9. After `udev` insert hook `plymouth` 

10. Regenerate initramfs:
```
sudo mkinitcpio -p linux
```

## Step 5 Optional steps

1. Install AUR package manager

```
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

2. Install pamac

```
yay -S libpamac-aur pamac-aur
```

3. Install East-Asian / Japanese fonts

```
pacman -S noto-fonts-cjk noto-fonts-emoji adobe-source-han-sans-jp-fonts adobe-source-han-serif-jp-fonts otf-ipafont
```