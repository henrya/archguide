# Convert existing ext3 or ext4 parition into LUKS 2 ecnrypted partition
### Make sure to take full backup before you start! To proceed, you need Arch bootable USB

1. Check whether the existing filesystem is healthy
```
e2fsck -f /dev/nvme0n1p2
```

1. Make sure that existing parition has at least 32 MB for LUKS header. NEW_SIZE = Current partition size - 32MB
```
resize2fs -p /dev/nvme0n1p2 <NEW_SIZE>
```

2. Encrypt the disk with default cipher and enter the passphrase
```
cryptsetup reencrypt --encrypt --reduce-device-size 16M /dev/nvme0n1p2
```

3. Open the encrypted partition
```
cryptsetup open /dev/nvme0n1p2 root
```

4. Resize the partition to the full size.
```
resize2fs /dev/mapper/root
```

5. Mount the filesystem
```
mount /dev/mapper/root /mnt
mount /dev/nvme0n1p1 /mnt/boot
arch-chroot /mnt
```

6. Generate keyfile and make sure that only root can access the key
```
sudo dd bs=512 count=4 if=/dev/random of=/root/cryptlvm.keyfile iflag=fullbloc
sudo chmod 000 /root/cryptlvm.keyfile
```

7. Add the key in LUKS container
```
cryptsetup luksAddKey /dev/nvme0n1p2 /root/cryptlvm.keyfile
```

8. Edit  `/etc/mkinitcpio.conf` and add `/root/cryptlvm.keyfile` in the `FILES` section
```
FILES=(/root/cryptlvm.keyfile)
```

9. Edit  `/etc/mkinitcpio.conf` and add `keyboard`, `sd-vconsole`, and `sd-encrypt` hooks in the `HOOKS` section
```
HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)
```

10. Regenerate initramfs
```
mkinitcpio -P
```

11. Get the partition uuid of the encrypted partition
```
sudo blkid -s UUID -o value /dev/sda6
```

12. Modify grub configuration in `/etc/default/grub`
Add the LUKS parition name and  the key in `GRUB_CMDLINE_LINUX_DEFAULT`
```
GRUB_CMDLINE_LINUX_DEFAULT="... rd.luks.name=<uuid>=root rd.luks.key=/root/cryptlvm.keyfile ..."
```

13. Modify grub configuration in `/etc/default/grub` and enable cryptodisk
```
# Uncomment to enable booting from LUKS encrypted devices
GRUB_ENABLE_CRYPTODISK=y
```

14. Regenerate grub config:
```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

15. Edit /etc/fstab and add the encrypted partition
```
/dev/mapper/root  /  ext4  rw,relatime  0 1
```

16. Exit chroot, unmount all disks and reboot:
```
exit
umount /mnt/boot
umount /mnt
reboot
```