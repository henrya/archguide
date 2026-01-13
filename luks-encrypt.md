# Convert existing ext3 or ext4 partition into LUKS 2 encrypted partition

**WARNING: DATA LOSS RISK.**
This process involves resizing filesystems and overwriting partition headers. **Always create a full backup of your data** before proceeding. This guide assumes you have an Arch Linux installation media (live USB).

### Pre-requisites
*   Boot into the Arch Linux Live USB.
*   Identify your partitions (e.g., `lsblk`).
    *   Assume `/dev/nvme0n1p2` is the root partition to encrypt.
    *   Assume `/dev/nvme0n1p1` is the separate unencrypted boot/EFI partition.

### Step 1: Check and Resize Filesystem

1. Check filesystem health:
    ```bash
    e2fsck -f /dev/nvme0n1p2
    ```

2. Resize the filesystem to make room for the LUKS header (requires ~32MB, we free up 32MB):
    *   **Calculation:** `current_size_blocks - 32M_converted_to_blocks`.
    *   Easier method with `resize2fs` usually handles the "M" suffix if supported, or calculate manually.
    *   Safest approach: Shrink it slightly more than needed to be safe, then grow it back later.
    ```bash
    # Example: If partition is 100G, resize FS to 99G to be safe
    resize2fs -p /dev/nvme0n1p2 <NEW_SIZE> 
    ```

### Step 2: Encrypt the Device

3. Convert the partition to LUKS2 (in-place encryption):
    *   This will prompt for a passphrase. **Do not forget it.**
    ```bash
    cryptsetup reencrypt --encrypt --reduce-device-size 32M /dev/nvme0n1p2
    ```

### Step 3: Open and Mount

4. Open the encrypted container:
    ```bash
    cryptsetup open /dev/nvme0n1p2 root
    ```

5. Resize the filesystem to fill the container:
    ```bash
    resize2fs /dev/mapper/root
    ```

6. Mount the system:
    ```bash
    mount /dev/mapper/root /mnt
    mount /dev/nvme0n1p1 /mnt/boot/efi  # Adjust if /boot is separate
    arch-chroot /mnt
    ```

### Step 4: Configure Bootloader and Initramfs

7. Edit `/etc/mkinitcpio.conf` to add encryption support:
    *   Locate the `HOOKS` array.
    *   Add `encrypt` hook **before** `filesystems`.
    *   Ensure `keyboard` is present to allow typing the password.
    
    ```bash
    # Example Standard HOOKS
    HOOKS=(base udev autodetect keyboard keymap consolefont modconf block encrypt filesystems fsck)
    ```

8. Regenerate initramfs:
    ```bash
    mkinitcpio -P
    ```

9. Get the UUID of the **encrypted physical partition** (not the mapper device):
    ```bash
    blkid -s UUID -o value /dev/nvme0n1p2
    # Copy this UUID
    ```

10. Edit `/etc/default/grub` to tell kernel where the encrypted root is:
    *   Find `GRUB_CMDLINE_LINUX_DEFAULT`.
    *   Append: `cryptdevice=UUID=<YOUR-UUID>:root root=/dev/mapper/root`
    
    ```bash
    GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:root root=/dev/mapper/root"
    ```

11. Regenerate GRUB config:
    ```bash
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

### Step 5: Update Fstab

12. Check `/etc/fstab`. The root mount point `/` usually points to a UUID. Since we are now using device mapper, update it:
    ```bash
    # Change the line for / to:
    /dev/mapper/root    /    ext4    rw,relatime    0 1
    ```

### Step 6: Reboot

13. Exit and Reboot:
    ```bash
    exit
    umount -R /mnt
    reboot
    ```
    *   On boot, you should be prompted for the passphrase by the initramfs before the system mounts the root partition.

### Note on Security (Keyfiles)
*   **Avoid storing keyfiles in initramfs** unless you have fully encrypted the boot partition (GRUB Cryptodisk). Storing a keyfile in an unencrypted `/boot` partition allows anyone with physical access to bypass your encryption. The passphrase method above is secure and standard.
