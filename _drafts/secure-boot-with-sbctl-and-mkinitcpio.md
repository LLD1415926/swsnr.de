---
title: Secure boot with sbctl and mkinitcpio
tags: [archlinux, secureboot, mkinitcpio, sbctl]
---

This article describes how I use secure boot on Arch Linux, with mkinicpio and sbctl, from zero to secure boot on a VM.
This article supersedes my earlier article about [secure boot with dracut](../_posts/2021-04-01-secure-boot-on-arch-linux-with-sbctl-and-dracut.md), as I no longer use dracut.

TK: When publishing, fix above link, and add a link to this post to the dracut post.

<!--more-->

## Setup a VM

We create a new VM with UEFI firmwire, using a text mode console for simplicity.

```console
$ virt-install --name arch-test \
  --memory 2048 --vcpus=2 --cpu host --disk size=10 --network user \
  --boot uefi --autoconsole text \
  --cdrom ~/.local/share/libvirt/images/archlinux-2022.12.01-x86_64.iso
```

This starts a serial console with the bootloader screen:

```
                   GNU GRUB  version 2:2.06.r380.g151467888-1

 +----------------------------------------------------------------------------+
 | Arch Linux install medium (x86_64, UEFI)                                   |
 |*Arch Linux install medium with speakup screen reader (x86_64, UEFI)        |
 | UEFI Shell                                                                 |
 | UEFI Firmware Settings                                                     |
 | System shutdown                                                            |
 | System restart                                                             |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 +----------------------------------------------------------------------------+

      Use the ^ and v keys to select which entry is highlighted.
      Press enter to boot the selected OS, `e' to edit the commands
      before booting or `c' for a command-line. ESC to return
      previous menu.
```

We edit the first entry to add the `console=ttyS0` to make the installer use the qemu serial console as tty1:

```
                   GNU GRUB  version 2:2.06.r380.g151467888-1

 +----------------------------------------------------------------------------+
 |setparams 'Arch Linux install medium (x86_64, UEFI)'                        |
 |                                                                            |
 |    set gfxpayload=keep                                                     |
 |    search --no-floppy --set=root --label ARCH_202212                       |
 |    linux /arch/boot/x86_64/vmlinuz-linux archisobasedir=arch archisolabel=\|
 |ARCH_202212 console=ttyS0                                                   |
 |    initrd /arch/boot/intel-ucode.img /arch/boot/amd-ucode.img /arch/boot/x\|
 |86_64/initramfs-linux.img                                                   |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 +----------------------------------------------------------------------------+

      Minimum Emacs-like screen editing is supported. TAB lists
      completions. Press Ctrl-x or F10 to boot, Ctrl-c or F2 for
      a command-line or ESC to discard edits and return to the GRUB menu.
```

This lets us install Arch Linux over the serial console on our current TTY.
This is simpler to setup than SSH, and more convenient than the standard text console in a splice display.

## Install Arch Linux

In the installation shell we first partition the disk with an EFI system partition and a root partition, with appropriate types for auto-discovery.
We encrypt the root partition, and create a FAT filesystem for the system partition, and btrfs on the root file system.

```console
# sgdisk -Z /dev/vda
Creating new GPT entries in memory.
GPT data structures destroyed! You may now partition the disk using fdisk or
other utilities.
# sgdisk -n1:0:+550M -t1:ef00 -c1:EFISYSTEM -N2 -t2:8304 -c2:linux /dev/vda
Creating new GPT entries in memory.
The operation has completed successfully.
# partprobe /dev/vda
# cryptsetup luksFormat --type luks2 /dev/disk/by-partlabel/linux
WARNING!
========
This will overwrite data on /dev/disk/by-partlabel/linux irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase for /dev/disk/by-partlabel/linux:
Verify passphrase:
# cryptsetup open /dev/disk/by-partlabel/linux root
Enter passphrase for /dev/disk/by-partlabel/linux:
# mkfs.fat -F32 -n EFISYSTEM /dev/disk/by-partlabel/EFISYSTEM
# mkfs.btrfs -f -L linux /dev/mapper/root
btrfs-progs v6.0.2
See http://btrfs.wiki.kernel.org for more information.

NOTE: several default settings have changed in version 5.15, please make sure
      this does not affect your deployments:
      - DUP for metadata (-m dup)
      - enabled no-holes (-O no-holes)
      - enabled free-space-tree (-R free-space-tree)

Label:              linux
UUID:               74c7384c-9d57-4b69-9493-efbd0b529264
Node size:          16384
Sector size:        4096
Filesystem size:    9.45GiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP             256.00MiB
  System:           DUP               8.00MiB
SSD detected:       no
Zoned device:       no
Incompat features:  extref, skinny-metadata, no-holes
Runtime features:   free-space-tree
Checksum:           crc32c
Number of devices:  1
Devices:
   ID        SIZE  PATH
    1     9.45GiB  /dev/mapper/root
```

Then we mount the file systems and bootstrap.
We explicitly run reflector again to use an up-to-date mirrorlist for bootstrapping, and refresh the keyring to avoid signature errors:

```console
# mount -o compress=zstd:1 /dev/mapper/root /mnt
# mkdir /mnt/efi
# mount /dev/disk/by-partlabel/EFISYSTEM /mnt/efi
# systemctl restart reflector.service
# pacman -Sy archlinux-keyring
:: Synchronizing package databases...
[…]
# pacstrap -K /mnt \
    base linux linux-firmware btrfs-progs neovim qemu-guest-agent intel-ucode
==> Creating install root at /mnt
[…]
```

This runs the `mkinitcpio` hooks to copy the kernel and to generate an initramfs to `/boot`.
However, for secure boot we need to use unified kernel images (UKIs) so we can remove the kernel and the initrd from `/boot` again.

```console
# rm /mnt/boot/initramfs-linux* /mnt/boot/vmlinuz-linux
```

We will later generate a preliminary UKI before rebooting.
But first we configure our new system.
We enable relevant locales, configure essential system settings for the first boot.
We also take care of networking, and configure DNS resolution with systemd-resolved, and networking with systemd-networkd.
The arch live system also use systemd-network so we can conveniently copy its configuration to the new system.

```console
# sed -i -e '/^#en_GB.UTF-8/s/^#//' /mnt/etc/locale.gen
# arch-chroot /mnt locale-gen
Generating locales...
  en_GB.UTF-8... done
Generation complete.
# systemd-firstboot --force --root /mnt \
    --keymap=us --locale=en_GB.UTF-8 --hostname=arch-test --timezone=UTC \
    --setup-machine-id --kernel-command-line='quiet console=ttyS0' \
    --prompt-root-password 
/mnt/etc/locale.conf written.
/mnt/etc/vconsole.conf written.
/mnt/etc/localtime written
/mnt/etc/hostname written.
/mnt/etc/machine-id written.

Welcome to your new installation of Arch Linux!

Please configure your system!

-- Press any key to proceed --

🔐 ‣ Please enter a new root password (empty to skip): ****
🔐 ‣ Please enter new root password again: ****
/mnt/etc/passwd written
/mnt/etc/shadow written.
/mnt/etc/kernel/cmdline written
# ln -sf /run/systemd/resolve/stub-resolv.conf /mnt/etc/resolv.conf
# cp -t /mnt/etc/systemd/network /etc/systemd/network/20-ethernet.network 
# systemctl --root /mnt enable \
    systemd-resolved.service systemd-networkd.service
Created symlink /mnt/etc/systemd/system/dbus-org.freedesktop.resolve1.service → /usr/lib/systemd/system/systemd-resolved.service.
Created symlink /mnt/etc/systemd/system/sysinit.target.wants/systemd-resolved.service → /usr/lib/systemd/system/systemd-resolved.service.
Created symlink /mnt/etc/systemd/system/dbus-org.freedesktop.network1.service → /usr/lib/systemd/system/systemd-networkd.service.
Created symlink /mnt/etc/systemd/system/multi-user.target.wants/systemd-networkd.service → /usr/lib/systemd/system/systemd-networkd.service.
Created symlink /mnt/etc/systemd/system/sockets.target.wants/systemd-networkd.socket → /usr/lib/systemd/system/systemd-networkd.socket.
Created symlink /mnt/etc/systemd/system/sysinit.target.wants/systemd-network-generator.service → /usr/lib/systemd/system/systemd-network-generator.service.
Created symlink /mnt/etc/systemd/system/network-online.target.wants/systemd-networkd-wait-online.service → /usr/lib/systemd/system/systemd-networkd-wait-online.service.
```

We do not need to configure `/etc/fstab`; systemd will automatically discover and mount our filesystems because we assigned correct partition types above.

For mkinitcpio up to version 34 we need to override the mkinitcpio `kernel-install` plugin to work around [mkinitcpio issue 153](https://gitlab.archlinux.org/archlinux/mkinitcpio/mkinitcpio/-/issues/153):

```console
# curl -fsSL https://gitlab.archlinux.org/archlinux/mkinitcpio/mkinitcpio/-/raw/master/50-mkinitcpio.install \
    > /mnt/etc/kernel/install.d/50-mkinitcpio.install
# chmod 755 /mnt/etc/kernel/install.d/50-mkinitcpio.install
```

Finally we install the bootloader, configure the initramfs, configure kernel installation and install the UKI for the first boot:

```console
# bootctl --root /mnt install
# sed -i -e \
    's/^HOOKS=.*/HOOKS=(systemd autodetect modconf kms keyboard sd-vconsole sd-encrypt block filesystems fsck)/' \
    /mnt/etc/mkinitcpio.conf
# echo layout=uki >> /mnt/etc/kernel/install.conf
# echo initrd_generator=mkinitcpio >> /mnt/etc/kernel/install.conf
# kernel_image="$(pacman --sysroot /mnt -Qlq linux | grep /vmlinuz)"
# arch-chroot /mnt \
    kernel-install --verbose add "$(basename "${kernel_image%/*}")" "${kernel_image}"
Reading /etc/kernel/install.conf…
/etc/kernel/install.conf configures layout=uki
/etc/kernel/install.conf configures initrd_generator=mkinitcpio
machine-id 1e6ac93e9e594f05b6d68a6d94d7e684 acquired from /etc/machine-id
Entry-token candidates: 1e6ac93e9e594f05b6d68a6d94d7e684 arch Default
/efi/1e6ac93e9e594f05b6d68a6d94d7e684 not found…
/efi/loader/entries exists, using BOOT_ROOT=/efi
No entry-token candidate matched, using "1e6ac93e9e594f05b6d68a6d94d7e684" from machine-id
Using ENTRY_DIR_ABS=/efi/1e6ac93e9e594f05b6d68a6d94d7e684/6.1.1-arch1-1
Plugin files:
/usr/lib/kernel/install.d/50-depmod.install
/etc/kernel/install.d/50-mkinitcpio.install
/usr/lib/kernel/install.d/90-loaderentry.install
+/usr/lib/kernel/install.d/50-depmod.install add 6.1.1-arch1-1 /efi/1e6ac93e9e594f05b6d68a6d94d7e684/6.1.1-arch1-1 /usr/lib/modules/6.1.1-arch1-1/vmlinuz
+depmod -a 6.1.1-arch1-1
+/etc/kernel/install.d/50-mkinitcpio.install add 6.1.1-arch1-1 /efi/1e6ac93e9e594f05b6d68a6d94d7e684/6.1.1-arch1-1 /usr/lib/modules/6.1.1-arch1-1/vmlinuz
Found microcode image /boot/intel-ucode.img
+ mkinitcpio -k '6.1.1-arch1-1' --kernelimage /usr/lib/modules/6.1.1-arch1-1/vmlinuz --microcode /boot/intel-ucode.img -U '/efi/EFI/Linux/1e6ac93e9e594f05b6d68a6d94d7e684-6.1.1-arch1-1.efi'
==> Starting build: 6.1.1-arch1-1
  -> Running build hook: [systemd]
  -> Running build hook: [autodetect]
  -> Running build hook: [modconf]
  -> Running build hook: [kms]
  -> Running build hook: [keyboard]
==> WARNING: Possibly missing firmware for module: xhci_pci
  -> Running build hook: [sd-vconsole]
  -> Running build hook: [sd-encrypt]
==> WARNING: Possibly missing firmware for module: qat_4xxx
  -> Running build hook: [block]
  -> Running build hook: [filesystems]
  -> Running build hook: [fsck]
==> WARNING: Possibly missing '/bin/sh' for script: /usr/bin/fsck.btrfs
==> Generating module dependencies
==> Creating zstd-compressed initcpio image: /tmp/mkinitcpio.V1ZDZx
==> Image generation successful
==> Creating unified kernel image: /efi/EFI/Linux/1e6ac93e9e594f05b6d68a6d94d7e684-6.1.1-arch1-1.efi
  -> Using UEFI stub: /usr/lib/systemd/boot/efi/linuxx64.efi.stub
  -> Using cmdline file: /etc/kernel/cmdline
  -> Using os-release file: /etc/os-release
  -> Using microcode image: /boot/intel-ucode.img
==> Unified kernel image generation successful
+/usr/lib/kernel/install.d/90-loaderentry.install add 6.1.1-arch1-1 /efi/1e6ac93e9e594f05b6d68a6d94d7e684/6.1.1-arch1-1 /usr/lib/modules/6.1.1-arch1-1/vmlinuz
```

The bootloader automatically discovers UKIs, so we need no further bootloader configuration.

Eventually, we can reboot now:

```console
# systemctl reboot
```

Once rebooted, we can disable the mkinitcpio pacman hooks and instead install a `kernel-install` pacman hook, because we will use `kernel-install` to install UKIs to the EFI system partition.

```console
# install -m755 -d /etc/pacman.d/hooks
# ln -sf /dev/null /etc/pacman.d/hooks/60-mkinitcpio-remove.hook
# ln -sf /dev/null /etc/pacman.d/hooks/90-mkinitcpio-install.hook
```

We'll use [pacman-hook-kernel-install](https://aur.archlinux.org/packages/pacman-hook-kernel-install) to make pacman automatically call `kernel-install` instead;

```console
# pacman -Syu base-devel git
:: Synchronising package databases...
 core is up to date
 extra is up to date
 community               7.2 MiB  3.97 MiB/s 00:02 [######################] 100%
:: There are 26 members in group base-devel:
:: Repository core
   1) archlinux-keyring  2) autoconf  3) automake  4) binutils  5) bison
   6) debugedit  7) fakeroot  8) file  9) findutils  10) flex  11) gawk
   12) gcc  13) gettext  14) grep  15) groff  16) gzip  17) libtool  18) m4
   19) make  20) pacman  21) patch  22) pkgconf  23) sed  24) sudo  25) texinfo
   26) which

Enter a selection (default=all):
[…]
# cd /tmp
# sudo -u nobody \
    git clone https://aur.archlinux.org/pacman-hook-kernel-install.git
Cloning into 'pacman-hook-kernel-install'...
remote: Enumerating objects: 104, done.
remote: Counting objects: 100% (104/104), done.
remote: Compressing objects: 100% (59/59), done.
remote: Total 104 (delta 44), reused 104 (delta 44), pack-reused 0
Receiving objects: 100% (104/104), 18.13 KiB | 1.65 MiB/s, done.
Resolving deltas: 100% (44/44), done.
# cd /tmp/pacman-hook-kernel-install
# sudo -u nobody makepkg
==> Making package: pacman-hook-kernel-install 0.9.1-2 (Mon 26 Dec 2022 17:35:11 UTC)
==> Checking runtime dependencies...
==> Checking buildtime dependencies...
==> Retrieving sources...
  -> Found 90-kernel-install-add.hook
  -> Found 60-kernel-install-remove.hook
  -> Found kernel-install.sh
==> Validating source files with sha256sums...
    90-kernel-install-add.hook ... Passed
    60-kernel-install-remove.hook ... Passed
    kernel-install.sh ... Passed
==> Extracting sources...
==> Entering fakeroot environment...
==> Starting package()...
==> Tidying install...
  -> Removing libtool files...
  -> Purging unwanted files...
  -> Removing static library files...
  -> Stripping unneeded symbols from binaries and libraries...
  -> Compressing man and info pages...
==> Checking for packaging issues...
==> Creating package "pacman-hook-kernel-install"...
  -> Generating .PKGINFO file...
  -> Generating .BUILDINFO file...
  -> Generating .MTREE file...
  -> Compressing package...
==> Leaving fakeroot environment.
==> Finished making: pacman-hook-kernel-install 0.9.1-2 (Mon 26 Dec 2022 17:35:12 UTC)
# pacman -U *.pkg.tar.zst
loading packages...
resolving dependencies...
looking for conflicting packages...

Packages (1) pacman-hook-kernel-install-0.9.1-2

Total Installed Size:  0.00 MiB

:: Proceed with installation? [Y/n]
(1/1) checking keys in keyring                     [######################] 100%
(1/1) checking package integrity                   [######################] 100%
(1/1) loading package files                        [######################] 100%
(1/1) checking for file conflicts                  [######################] 100%
(1/1) checking available disk space                [######################] 100%
:: Processing package changes...
(1/1) installing pacman-hook-kernel-install        [######################] 100%
:: Running post-transaction hooks...
(1/1) Arming ConditionNeedsUpdate...
# cd
```

## Set up secure boot

We install `sbctl` and generate secure boot keys:

```console
# pacman -S sbctl
pacman -S sbctl
resolving dependencies...
looking for conflicting packages...
[…]
# sbctl create-keys
Created Owner UUID 0a52a4fb-27ab-47e4-ba36-96d55171917e
Creating secure boot keys...✓
Secure boot keys created!
```

Now we sign the boot loader, and install the signed boot loader.
We store the signed file in sbctl's database to update the signed binary if the bootloader gets updated.

```console
# sbctl sign -so \
    /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed \
    /usr/lib/systemd/boot/efi/systemd-bootx64.efi
✓ Signed /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed
# bootctl install
Copied "/usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed" to "/efi/EFI/systemd/systemd-bootx64.efi".
Copied "/usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed" to "/efi/EFI/BOOT/BOOTX64.EFI".
Random seed file /efi/loader/random-seed successfully written (32 bytes).
Not installing system token, since we are running in a virtualized environment.
Created EFI boot entry "Linux Boot Manager".
```

Then we rebuild the UKI with `kernel-install`.
The `sbctl` package includes a `kernel-install` plugin to sign new UKIs, so we automatically end up with a signed kernel binary.

```console
# kernel-install remove "$(uname -r)"
# kernel-install --verbose add "$(uname -r)" "/usr/lib/modules/$(uname -r)/vmlinuz"
Reading /etc/kernel/install.conf…
/etc/kernel/install.conf configures layout=uki
/etc/kernel/install.conf configures initrd_generator=mkinitcpio
machine-id 1e6ac93e9e594f05b6d68a6d94d7e684 acquired from /etc/machine-id
Entry-token candidates: 1e6ac93e9e594f05b6d68a6d94d7e684 arch Default
/efi/1e6ac93e9e594f05b6d68a6d94d7e684 not found…
/efi/loader/entries exists, using BOOT_ROOT=/efi
No entry-token candidate matched, using "1e6ac93e9e594f05b6d68a6d94d7e684" from machine-id
Using ENTRY_DIR_ABS=/efi/1e6ac93e9e594f05b6d68a6d94d7e684/6.1.1-arch1-1
Plugin files:
/usr/lib/kernel/install.d/50-depmod.install
/etc/kernel/install.d/50-mkinitcpio.install
/usr/lib/kernel/install.d/90-loaderentry.install
/usr/lib/kernel/install.d/91-sbctl.install
+/usr/lib/kernel/install.d/50-depmod.install add 6.1.1-arch1-1 /efi/1e6ac93e9e594f05b6d68a6d94d7e684/6.1.1-arch1-1 /usr/lib/modules/6.1.1-arch1-1/vmlinuz
+depmod -a 6.1.1-arch1-1
+/etc/kernel/install.d/50-mkinitcpio.install add 6.1.1-arch1-1 /efi/1e6ac93e9e594f05b6d68a6d94d7e684/6.1.1-arch1-1 /usr/lib/modules/6.1.1-arch1-1/vmlinuz
Found microcode image /boot/intel-ucode.img
+ mkinitcpio -k '6.1.1-arch1-1' --kernelimage /usr/lib/modules/6.1.1-arch1-1/vmlinuz --microcode /boot/intel-ucode.img -U '/efi/EFI/Linux/1e6ac93e9e594f05b6d68a6d94d7e684-6.1.1-arch1-1.efi'
==> Starting build: 6.1.1-arch1-1
  -> Running build hook: [systemd]
  -> Running build hook: [autodetect]
  -> Running build hook: [modconf]
  -> Running build hook: [kms]
  -> Running build hook: [keyboard]
==> WARNING: Possibly missing firmware for module: xhci_pci
  -> Running build hook: [sd-vconsole]
  -> Running build hook: [sd-encrypt]
==> WARNING: Possibly missing firmware for module: qat_4xxx
  -> Running build hook: [block]
  -> Running build hook: [filesystems]
  -> Running build hook: [fsck]
==> WARNING: Possibly missing '/bin/sh' for script: /usr/bin/fsck.btrfs
==> Generating module dependencies
==> Creating zstd-compressed initcpio image: /tmp/mkinitcpio.QnlVjn
==> Image generation successful
==> Creating unified kernel image: /efi/EFI/Linux/1e6ac93e9e594f05b6d68a6d94d7e684-6.1.1-arch1-1.efi
  -> Using UEFI stub: /usr/lib/systemd/boot/efi/linuxx64.efi.stub
  -> Using cmdline file: /etc/kernel/cmdline
  -> Using os-release file: /etc/os-release
  -> Using microcode image: /boot/intel-ucode.img
==> Unified kernel image generation successful
+/usr/lib/kernel/install.d/90-loaderentry.install add 6.1.1-arch1-1 /efi/1e6ac93e9e594f05b6d68a6d94d7e684/6.1.1-arch1-1 /usr/lib/modules/6.1.1-arch1-1/vmlinuz
+/usr/lib/kernel/install.d/91-sbctl.install add 6.1.1-arch1-1 /efi/1e6ac93e9e594f05b6d68a6d94d7e684/6.1.1-arch1-1 /usr/lib/modules/6.1.1-arch1-1/vmlinuz
Signing kernel /efi/EFI/Linux/1e6ac93e9e594f05b6d68a6d94d7e684-6.1.1-arch1-1.efi
```

We can now see that all binaries on the EFI system partition are signed:

```console
# sbctl verify
Verifying file database and EFI images in /efi...
✓ /efi/EFI/Linux/1e6ac93e9e594f05b6d68a6d94d7e684-6.1.1-arch1-1.efi is signed
✓ /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed is signed
✓ /efi/EFI/BOOT/BOOTX64.EFI is signed
✓ /efi/EFI/systemd/systemd-bootx64.efi is signed
```

We can now enroll our own secure boot keys.
We include the Microsoft CA for safety; otherwise the firmware will fail to load signed option ROMs which can brick some machines.

```
# sbctl enroll-keys --microsoft
Enrolling keys to EFI variables...
With vendor keys from microsoft...✓
Enrolled keys to the EFI variables!
```

We have no left setup mode, but secure boot is still disabled, because the current system wasn't booted securely obviously:

```
# sbctl status
Installed:      ✓ sbctl is installed
Owner GUID:     0a52a4fb-27ab-47e4-ba36-96d55171917e
Setup Mode:     ✓ Disabled
Secure Boot:    ✗ Disabled
Vendor Keys:    microsoft
```

We can reboot, and check again:

```console
# systemctl reboot
# sbctl status
Installed:      ✓ sbctl is installed
Owner GUID:     0a52a4fb-27ab-47e4-ba36-96d55171917e
Setup Mode:     ✓ Disabled
Secure Boot:    ✓ Enabled
Vendor Keys:    microsoft
```

# Unlock root disk automatically

With secure boot in place we can optionally unlock the root device automatically using a TPM key bound to the secure boot state.
This allows our signed binaries to decrypt the root disk automatically:

```console
# systemd-cryptenroll --recovery-key /dev/disk/by-partlabel/linux
🔐 Please enter current passphrase for disk /dev/disk/by-partlabel/linux: ****
A secret recovery key has been generated for this volume:

    🔐 egkjiudd-jnjkjbfu-iddcufte-fglktehb-ccgetgrn-lkbbffrv-bjkbclvg-tfuughfc

Please save this secret recovery key at a secure location. It may be used to
regain access to the volume if the other configured access credentials have
been lost or forgotten. The recovery key may be entered in place of a password
whenever authentication is requested.
New recovery key enrolled as key slot 1.
# systemd-cryptenroll --tpm2-device=auto /dev/disk/by-partlabel/linux
New TPM2 token enrolled as key slot 2.
# systemd-cryptenroll --wipe-slot=0 /dev/disk/by-partlabel/linux
Wiped slot 0.
# systemd-cryptenroll /dev/disk/by-partlabel/linux
SLOT TYPE
   1 recovery
   2 tpm2
```
