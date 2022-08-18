+++ 
title = "Arch Linux: Dual boot with Secure Boot, LUKS unlocking via TPM2"
date = 2022-08-18 
description = "This post summarizes my current Arch setup involving a dual boot with Windows 10, Secure Boot, full disk encryption with LUKS unlocked through TPM"
draft = false 
toc = true 
author = "rogueai"
categories = ["post"]
tags = ["arch", "linux", "luks", "tpm", "secure boot"]
license = "cc-by-nc-sa-4.0"
+++

# Introduction
I've been running Arch Linux for quite some time, but my original installation was far from being secure: I had no full disk
encryption (FDE) and Secure Boot disabled, because at the time I thought I could live without any of that and avoiding extra
steps in the installation process.

Over time, I got more concerned over these aspects, especially since I'm using Arch on my work laptop. I managed to delay
the inevitable by encrypting _just_ my home directory with `ecryptfs`, but eventually I decided to deal with it and make
a fresh installation with my ideal setup in mind, based on the information I collected over the years.

This ideal setup consists in the following:
- rEFInd boot loader
- Secure Boot
- LUKS encryption
- LUKS volume unlocking via TPM2 chip, without requiring a decryption password
- Dual boot with Windows 10

For those of you familiar with Windows, this setup is essentially what you would find in a regular Window installation 
with BitLocker.

This post will go into the details of how to setup everything on a dual boot environment, but it should be applicable even 
on a standalone Arch install.

# Installation

I won't go through the details of installing arch with LUKS, there are plenty of guides for that.
I'm going however to highlight some details about the partition layout I chose, as it will be relevant later on.

- ESP partition, mounted on `/efi`: if dual booting, the ESP partition is probably already available, created by Windows.
  Otherwise, create an unencrypted FAT partition of about 500MB. 
  > Note: we're going to be using EFI stubs instead of the usual *.img created by `mkinitcpio`. These tend to get quite
  > large so ensure your ESP partition is comfortably sized. Especially when dual booting on a OEM Windows laptop,
  > vendors tend to be stingy on space: for instance I ended up resizing my own ESP partition from 200MB to 1GB.
  
  At any rate, make sure the partition is unencrypted as rEFInd (or systemd-boot as far as I know) won't be able to boot
  from an encrypted ESP.

- Boot parition `/boot`: I do not have a separate partition for `/boot`. In fact, `/boot` is actually encrypted inside 
  the root LUKS volume. This is intended as we don't want to expose any unsigned kernel to the boot process.

# Securing the boot process

This is where the fun starts. Our system is now encrypted at rest, but with a few issues:
- Secure Boot is disabled: any malicious entity could potentially sneak in something fishy in the boot process.
- Booting requires inputting two different passwords: the first one at boot time to unlock the LUKS volume, the second to
  login
- The LUKS password is either very secure, but tedious to remember and input, or too fragile leaving us exposed to brute 
  force attacks.

We're going to solve all these problems in two steps:
- Securing the boot phase with Secure Boot
- Unlocking the LUKS volume with a key stored in the TPM chip and bound to the Secure Boot state. 

## Enabling Secure Boot
We'll be bundling in a single EFI image the kernel, modules, parameters, initramfs and microcode. The EFI image will be 
signed with our own keys, which essentially means: nothing will be able to boot unless is signed with our custom keys. 

> Warning: this also includes any live usb like Archiso or Ventoy.
> 
> A special note on Venoty: Ventoy supports Secure Boot by allowing users to enroll its own certificate using the MokManager.
> [There is an open security issue](https://github.com/ventoy/Ventoy/issues/135) whereas after passing the Secure Boot check,
> Ventoy allows any of the isntalled ISOs to be booted, without any extra check. As mentioned in the GitHub issue, this
> essentially negates any Secure Boot protection, as once Ventoy's cert is enrolled in the MOK, anybody could load a Ventoy
> USB stick into your laptop and boot anything from it. 
> 
> For this reason, I strongly recommend to avoid adding Ventoy cert in the MokManager.

To create EFI images and sign them, we're going to need:
- `mkinitcpio` v31 or later
- [sbctl](https://github.com/Foxboron/sbctl)

### Create EFI images with `mkinitcpio`

The process to create signed EFI bundles used to be quite convoluted, requiring to ditch `mkinitcpio` in favour of other 
tools supporting EFI bundles like `dracut`. Recently though mkinitcpio [introduced support for EFI generation](https://linderud.dev/blog/mkinitcpio-v31-and-uefi-stubs/)
so this is now quite straightforward.

Edit your `/etc/mkinitcpio.d/linux.preset` to look like this:
```
# mkinitcpio preset file for the 'linux' package

ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux"
ALL_microcode=(/boot/*-ucode.img)

PRESETS=('default')

default_image="/boot/initramfs-linux.img"
default_efi_image="/efi/EFI/arch/arch-linux.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

# other presets ...
```

If you have any Kernel parameters that used to be added to the bootloader config, they now need to be bundled into the EFI image as well. 

By default 
`mkinitcpio` will bundle `/etc/kernel/cmdline` when generating the EFI, but it can be configured adding `--cmdline /path/to/kernel/cmdline`
to the preset options if required.

And that's it! Running `mkinitcpio -p linux` will now generate the EFI images into `/efi/EFI/arch/*.efi`.
### Signing EFI images

Another process that used to be rather manual and error prone is now greatly simplified by the excellent project [sbctl](https://github.com/Foxboron/sbctl).
From the project page:
> sbctl intends to be a user-friendly secure boot key manager capable of setting up secure boot, offer key management 
> capabilities, and keep track of files that needs to be signed in the boot chain.

The first thing we need to do is enable Secure Boot in "Setup Mode". Reboot into the BIOS config utility and there
should be an option for that. I had it under "Security -> Secure Boot"

> A note of caution: enabling Setup Mode will clear all the keys currently stored, including Microsoft and vendor ones

Reboot again and check that Setup Mode is enabled with:
```
$ sbctl status
Installed:	 ✓ sbctl is installed
Owner GUID:	 <GUID>
Setup Mode:	 ✗ Enabled
Secure Boot: ✗ Disabled
```
At this point we can create a new set of keys and enroll them:
```
$ sbctl create-keys
$ sbctl enroll-keys --microsoft
```
For reference, `sbctl` places generated keys `/usr/share/secureboot`. It's probably a good idea to back them up to a 
safe location.

We can now start signing our EFI images, including:
- Any *.efi generated by `mkinitcpio` presets
- Bootloader image(s)
- If using `fwupd`, sign its efi as well
- Vendor images, such as Lenovo recovery EFI
```
$ sbctl sign -s /efi/EFI/arch/arch-linux.efi
$ sbctl sign -s /efi/EFI/refind/refind_x64.efi
$ sbctl sign -s /efi/EFI/refind/drivers_x64/ext4_x64.efi
$ sbctl sign -s /efi/EFI/refind/drivers_x64/btrfs_x64.efi
$ sbctl sign -s /efi/EFI/arch/fwupdx64.efi
$ sbctl sign -s /efi/EFI/Boot/LenovoBT.EFI
```
With the `-s` option, `sbctl` keeps track every EFI it signed to its own database: they will be automatically signed at
every kernel change detected with a pacman hook.

> Note on dual boot with Windows: with the `--microsoft` option, `sbctl` enrolls the official Microsoft keys. You don't 
> need to sign any Microsoft EFI as they'll work out of the box as if nothing changed. It is still possible to double-sign
> with both their keys and your own ones, [but in some cases that might cause issues](https://github.com/Foxboron/sbctl/wiki/Linux-Windows-Dual-Boot-with-Windows-Bitlocker)
> 
> Personally, I decided to leave them as-is to avoid trouble. 

We can verify the signing worked:
```
$ sbctl verify
Verifying file database and EFI images in /efi...
✓ /efi/EFI/arch/arch-linux.efi is signed
✓ /efi/EFI/arch/fwupdx64.efi is signed
✓ /efi/EFI/refind/drivers_x64/btrfs_x64.efi is signed
✓ /efi/EFI/refind/drivers_x64/ext4_x64.efi is signed
✓ /efi/EFI/refind/refind_x64.efi is signed
✓ /efi/EFI/Boot/LenovoBT.EFI is signed
```
`sbctl` is smart enough to find the esp partition and checks every EFI it finds. If you're dual booting and you didn't 
double-sign MS EFIS, they will show up as not signed, but that's not an issue: they're still signed with the original MS
keys.

We can now reboot and we should be now in Secure Boot mode. rEFInd should be able to find our signed EFI and happily boot
from it.

We can also verify the new Secure Boot state:
```
❯ sbctl status     
Installed:	 ✓ sbctl is installed
Owner GUID:	 <GUID>
Setup Mode:	 ✓ Disabled
Secure Boot: ✓ Enabled
Vendor Keys: microsoft
```

> If dual booting, you can now re-enable BitLocker. Bitlocker will generate a new recovery key, so make sure to take note of it.

## Password-less LUKS unlocking

From a security point of view, passwordless LUKS unclocking might look like we're giving up some security, as booting 
will go straight to login without asking any password whatsoever.
We're indeed trading a bit of security in favour of convenience, it's important to note though that binding the LUKS to
the TPM ensures the volume will only unlock in our machine, with Secure Boot enabled and our signed boot image.

Anything else will result in the LUKS not to unlock, requiring another authentication method. We'll be setting up a
highly secure recovery key as an alternative unlocking method, which is arguably much better than a potentially weak password.

Now for the last part of this setup we're going to need:
- `mkinitcpio`
- `systemd-cryptenroll` for its native support for enrolling keys in TPM for LUKS volume unlocking

`systemd-cryptenroll` provides a handy way to setup different authentication mechanisms to unlock our LUKS volume. Paired
with `mkinitcpio` systemd init hooks, it allows for seamless TPM integration.

### Preparing the EFI image
We're going to have to make some changes on our `mkinitcpio` build config, but first of all check the TPM version:
``` 
$ systemd-cryptenroll --tpm2-device=list
PATH        DEVICE     DRIVER 
/dev/tpmrm0 NTC0702:00 tpm_tis
```
Take note of the driver name: `tpm_tis`, and add it to the MODULES list in `/etc/mkinitcio.conf` [according to the Arch wiki](https://wiki.archlinux.org/title/Trusted_Platform_Module#systemd-cryptenroll):
```
...
MODULES=(i915 tpm_tis)
...
HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt filesystems fsck)  
```
Apart from the extra `MODULE`, if you're using the regular busybox `encrypt` hook, you need to switch to the systemd 
`sd-encrypt` one. Note that switching to `systemd` init hooks requires other hooks to be replaced. For example `udev`, 
`usr` and `resume` are all replaced by the `systemd` hook as explained in the [mkinitcpio runtime hook section](https://wiki.archlinux.org/title/Mkinitcpio#Runtime_hooks) 
in the Arch Wiki.

Switching to `sd-necrypt` means we also need to tweak the Kernel parameters in `/etc/kernel/cmdline`.
These are my current kernel options, for a BTRFS partition on LUKS:
```
rd.luks.name=<UUID>=root rd.luks.options=tpm2-device=auto root=/dev/mapper/root rootflags=subvol=@ rw add_efi_memmap 
```
The important bits are:
- `rd.luks.name` same as `cryptdevice`
- `rd.luks.options=tpm2-device=auto`: enables unlocking through a key enrolled in the TPM

There are also several other options as backup authentication method: such as a FIDO2
hardware key. It's also possible to setup a PIN in conjunction with the TPM unlocking.

Now we can rebuild the EFI image with `mkinitcpio -P`

> Important note: `sbctl` only re-signs EFIs on pacman operations, when building them manually via `mkinitcpio`, make sure 
> to re-sign all images with:
> ``` 
> $ sbctl sign-all
> ``` 

### Enroll
The last step before testing all out, is to enroll a key in the TPM to be used a LUKS authentication method:
``` 
$ systemd-cryptenroll /dev/<luks-volume> --recovery-key
$ systemd-cryptenroll /dev/<luks-volume> --tpm2-device=auto --tpm2-pcrs=0,7
```
The first line will enroll a recovery keys as an alternative authentication method. Make sure you store it somewhere safe,
as this will be the only way to unlock your volume if either something happens to the TPM chip, or you need to disable
Secure Boot for any reason.

The second line enrolls the TPM key as authentication method, binding it to the Platform Configuration Registers (PCR) #0 and #7.

There are [a number of registers available](https://wiki.archlinux.org/title/Trusted_Platform_Module#Accessing_PCR_registers) 
in the TPM, and you can enroll the key to multiple ones, but at the bare minimum you should be using register #7, the 
"Secure Boot State". In plain terms, it will unlock the LUKS volume if the Secure Boot checks passes. As we signed all
our EFIs, this means we won't need to worry about re-enrolling LUKS keys in the TPM as long as Secure Boot is operational.

It's a good idea to enroll the key to PCR #0 "Core System Firmware executable code", but that means every time the firmware
gets updates in the BIOS, we need to unlock the volume with the recovery-key and re-run `systemd-cryptenroll` for the
TPM-based authentication method.

Now we can reboot and test that everything works as expected, you should get straight to user login. It's worth testing
that disabling Secure Boot, we get requested for a decryption password.

> Important: make sure you have your recovery key stored somewhere at this point!

If everything checks out, we can wipe the password slot, effectively rendering the recovery key the only backup authentication
method:
``` 
$ systemd-cryptenroll /dev/<luks-volume> --wipe-slot=password
```

The last thing to note about `systemd-cryptenroll`, is that there are several other interesting authentication methods
available, such as FIDO2 hardware tokens like YubiKeys.

# Conclusion
That's it! I hope you found this useful :)

There's definitely something that could be improved in this setup, but I'm quite happy so far with the result. I'm still
working on solving some open issues, like building my own custom [Archiso](https://wiki.archlinux.org/title/Archiso) producing
EFIs signed with my own keys, but that's going to be something for another post!

# References and further reading
In no particular order, my approach has been put together using the following resources:

1. [Decrypt LUKS2-encrypted root partitions with TPM2](https://gist.github.com/chrisx8/cda23e2d1fa3dcda0d739bc74f600175)
2. [Install Arch with Secure boot, TPM2-based LUKS encryption, and systemd-homed](https://lunaryorn.com/install-arch-with-secure-boot-tpm2-based-luks-encryption-and-systemd-homed)
3. [Authenticated Boot and Disk Encryption on Linux](https://0pointer.net/blog/authenticated-boot-and-disk-encryption-on-linux.html)
4. [mkinitcpio v31 and UEFI stubs](https://linderud.dev/blog/mkinitcpio-v31-and-uefi-stubs/)
5. [Linux Windows Dual Boot with Windows Bitlocker](https://github.com/Foxboron/sbctl/wiki/Linux-Windows-Dual-Boot-with-Windows-Bitlocker)
6. [Arch Wiki: Trusted Platform Module](https://wiki.archlinux.org/title/Trusted_Platform_Module#systemd-cryptenroll)
7. [Arch Wiki: Unified Extensible Firmware Interface/Secure Boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)
8. [Arch Wiki: mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio)
9. [Arch Wiki: dm-crypt/Encrypting an entire system - Configuring mkinitcpio](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio) 
10. [Arch Wiki: System configuration - Kernel parameters](https://wiki.archlinux.org/title/Dm-crypt/System_configuration#Kernel_parameters)
