# Complete Guide: Setup Btrfs Swap File with Hibernation on Arch-based Systems

This guide covers creating a Btrfs-compatible swap file, ensuring it's activated correctly, disabling conflicting services like `zram`, and configuring the bootloader for hibernation.

## 1\. Clean Up Existing Setup

First, remove any old or broken swap configurations to prevent conflicts.

```bash
sudo swapoff -a
sudo umount /swap 2>/dev/null || true
sudo btrfs subvolume delete /swap 2>/dev/null || true
sudo rm -rf /swap
```

## 2\. Create a New Swap Subvolume

A dedicated, non-compressed, non-COW (Copy-on-Write) subvolume is required for a Btrfs swap file.

1.  Find your root partition's **UUID**:
    ```bash
    findmnt -no UUID /
    ```
2.  Create a temporary mount point for the Btrfs root (subvolid=5):
    ```bash
    sudo mkdir -p /mnt/btrfs-root
    sudo mount -o subvolid=5 /dev/disk/by-uuid/<YOUR_ROOT_UUID> /mnt/btrfs-root
    ```
3.  Create the new swap subvolume:
    ```bash
    sudo btrfs subvolume create /mnt/btrfs-root/swap
    ```
4.  Unmount and remove the temporary directory:
    ```bash
    sudo umount /mnt/btrfs-root
    sudo rmdir /mnt/btrfs-root
    ```

## 3\. Mount the Subvolume via /etc/fstab

Add an entry to `/etc/fstab` to mount the new swap subvolume at boot.

1.  Open `/etc/fstab` with a text editor:
    ```bash
    sudo nano /etc/fstab
    ```
2.  Add this line, replacing `<YOUR_ROOT_UUID>`. This ensures **no compression** and **no COW** for the subvolume:
    ```
    UUID=<YOUR_ROOT_UUID> /swap btrfs subvol=swap,noatime,compress=no,space_cache=v2 0 0
    ```
3.  Create the mount point and mount the new subvolume:
    ```bash
    sudo mkdir /swap
    sudo mount -a
    ```

## 4\. Create and Activate the Swap File

Now, create the swap file itself inside the new subvolume.

1.  Set the 'No Copy-on-Write' (C) attribute on the `/swap` directory (as a safeguard):
    ```bash
    sudo chattr +C /swap
    ```
2.  Create the swap file (adjust `16G` to your RAM size):
    ```bash
    sudo fallocate -l 16G /swap/swapfile
    ```
3.  Set correct permissions:
    ```bash
    sudo chmod 600 /swap/swapfile
    ```
4.  Format the file as swap:
    ```bash
    sudo mkswap /swap/swapfile
    ```
5.  Turn the swap file on for the current session:
    ```bash
    sudo swapon /swap/swapfile
    ```
6.  **Make activation permanent:** Add the swap file to `/etc/fstab` so it's enabled on every boot. Open it again:
    ```bash
    sudo nano /etc/fstab
    ```
7.  Add this line at the very end:
    ```
    /swap/swapfile none swap defaults 0 0
    ```

## 5\. Get the Resume Offset

The kernel needs the physical offset of the swap file.

1.  Run this command:
    ```bash
    sudo btrfs inspect-internal map-swapfile -r /swap/swapfile
    ```
2.  You will get an output with a value for `resume_offset`. Note this number (e.g., `4546994`).

## 6\. Update Kernel Parameters (GRUB)

Configure the GRUB bootloader to tell the kernel where to resume from. This is also where we disable `zram`, which conflicts with hibernation.

1.  Open the GRUB config file:

    ```bash
    sudo nano /etc/default/grub
    ```

2.  Find the `GRUB_CMDLINE_LINUX_DEFAULT` line. Add three parameters, replacing the UUID and offset with your values:

      * `resume=UUID=<YOUR_ROOT_UUID>`
      * `resume_offset=<YOUR_OFFSET_VALUE>` (from Step 5)
      * `systemd.zram=0` (This is crucial to disable `zram`, which is in-memory and prevents "real" hibernation)

    Your line should look similar to this:

    ```
    GRUB_CMDLINE_LINUX_DEFAULT="quiet splash resume=UUID=0d29f6c9... resume_offset=4546994 systemd.zram=0"
    ```
    * if it doesn't work add `zswap.enabled=1` in the above line before `systemd.zram=0`.
3.  Save the file and regenerate the GRUB config:

    ```bash
    sudo grub-mkconfig -o /boot/grub/grub.cfg
    ```

## 7\. Update mkinitcpio Hooks

The `resume` hook is needed in the initramfs (early boot image) to handle resuming from hibernation.

1.  Edit `/etc/mkinitcpio.conf`:
    ```bash
    sudo nano /etc/mkinitcpio.conf
    ```
2.  Find the `HOOKS` array. Ensure the `resume` hook is placed **after** `udev` and `filesystems`.
    ```
    HOOKS=(base udev autodetect ... block filesystems resume fsck)
    ```
3.  Rebuild the initramfs:
    ```bash
    sudo mkinitcpio -P
    ```

## 8\. Reboot and Verify

Reboot your computer for all changes (fstab, GRUB, mkinitcpio) to take effect.

```bash
reboot
```

After you log back in, verify that your new swap file is active and that `zram` is gone.

```bash
swapon --show
```

You **should** see your swap file listed:

```
NAME             TYPE      SIZE USED PRIO
/swap/swapfile   file       16G   0B   -2
```

You **should not** see `/dev/zram0` in the output. If you do, double-check Step 6.

## 9\. Test Hibernation

Once you've confirmed your swap file is active, test hibernation:

```bash
sudo systemctl hibernate
```

Your system should write RAM to disk and power off. When you turn it back on, it should restore your session exactly as you left it.

-----

## Acknowledgements

This guide is based on the original article by **9M2PJU** from **hamradio.my**, with additional troubleshooting steps.

  * **Original Article:** [Setting Up a Btrfs-Compatible Swap File with Hibernation on CachyOS (or Any Arch-based System)](https://hamradio.my/2025/06/setting-up-a-btrfs-compatible-swap-file-with-hibernation-on-cachyos-or-any-arch-based-system/)
