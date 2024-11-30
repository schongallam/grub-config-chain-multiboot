# GRUB Config-Chain Multiboot

## About

This repository describes a relatively simple way to set up an external drive with multiple linux live ISO files, and boot into a selected OS from grub.  Technically, no chainloading is required.  In most cases, no direct knowledge of any linux kernels or initrd locations is required, either, as long as the ISO contains its own loopback.cfg or grub.cfg file.*  Example files are provided, but modifications MUST be made for it to actually work on your system.  It will not work straight "out of the box," and no automated configuration method is provided here.  This method assumes you are using x86_64 architecture, adjust as needed.  Use at your own risk.

*Some kernels require modification of grub's `kernel` line to help the kernel find the ISO file system / squashfs.  See "Known Issues," below.

## Requirements

- A UEFI-compatible system (for this example) with Secure Boot disabled.
- An external storage device
- install-grub (tested on GNU GRUB version 2.06)

## How it works

0. Your machine boots to grub on the ESP partition of your multiboot drive.
1. Using the configuration file that you updated, it lists the ISOs available to boot into
2. Once an ISO is selected, grub sets up a loopback device to load the grub configuration file (via loopback.cfg, or alternatively, grub.cfg if you prefer) that was supplied in the ISO file itself.
3. The ISO's native configuration file takes over, and you are presented with the distro's intended boot option menu.
4. Technically, nowhere in this process does 'chainloading' of a bootloader actually occur. (But, see "Other Features" below)

## How to set it up (suggested method)

0. Prepare/partition an external drive, using your favorite CLI or GUI tools.
- Partition 1: fat32, ESP.  Grub doesn't take much space (<10 MB in most cases) but you may choose to put other things here, (such as background graphics) now or later.
- Partition 2: any file system that grub can read (i.e. ext4).  The size should be enough to accommodate all your ISOs (i.e. probably at least several GB).
1. Install GRUB2 on the ESP partition (/dev/sdc1 in this example, adjust as needed) as root:
```
mount /dev/sdc1 /mnt/esp
grub-install --target=x86_64-efi --efi-directory=/mnt/esp --boot-directory=/mnt/esp/boot --removable --recheck --no-nvram --bootloader-id=GRUB
```
2. Replace the contents of grub.cfg on the ESP partition with the grub.cfg in the repo's' ESP partition folder.  All this file does is configure grub to load the "referential" grub.cfg on the second partition.  Don't forget to change the UUID field accordingly!
3. Copy new ISOs onto the second partition
4. If you want to chainload any boot loaders on your host hard drive, see "Other Features," below.
5. Reboot to the external drive, and test!

## Other Features

A "Host" submenu example is provided in grub.cfg for chainloading other bootloaders.  Replace `__HOSTUUID__` with the UUID of your host machine's permanent ESP partition to give yourself an option of booting to a permanently installed OS.  Verify that the paths specified by the `chainloader` commands are correct for your machine.

## Known Issues

- Some distros require manual editing of the boot command during a grub session.
  - i.e., highlight "Try / Install" or another boot option, and press 'e' to enter the grub editor.
  - If you don't, you'll get a kernel error on boot: "Unable to find a medium containing a live file system."
  - Distros that don't use lupin-casper cannot accept an `iso-scan/filename` kernel parameter, and require `findiso` instead.
- Tested distros:
  - ParrotOS (tested with Security Edition, 6.1)
    - Edit the boot command to add `findiso=$iso_path` argument to the kernel line.
  - Tails (tested, 6.9)
    - Use the "External Hard Disk" boot option, or ensure `live-media=removable` is removed from kernel line.
    - Add `findiso=$iso_path` argument to the kernel line.
    - Remove `FSUUID=${rootuuid}` from the kernel line.
  - Other distros lacking robust boot-from-ISO implementation likely require similar editing at boot time.
- Native grub also fails to load memtest (invokes `linux /live/memtest` but the files present are `memtest.bin` and `memtest.efi`).

## FAQ

There are a lot of HOWTOs on the web for setting up grub multiboot. How is this method different?
- Other HOWTOs tend to configure grub to invoke the `linux` and `initrd` commands directly, bypassing any of the native ISO's grub menu options.  If you want the ISO to load as if it were being directly booted to, giving you the full ISO experience, this way is better.

Why not chainload to the EFI boot loader inside the ISO?
- To my knowledge, this is not technically feasible with any current existing GRUB loopback device.  Sure, you can execute the EFI file, but it will lose reference to any filesystem inside the ISO, and fail.

Did you try this with any other boot loader, like rEFInd?
- Yes. I love rEFInd, but its maintainer, Rod Smith, has specifically posted somewhere that it lacks the loopback capability necessary to chainload into an ISO (see previous question).  Also, live Linux distro ISOs almost universally contain grub, making config chaining somewhat trivial.

Why two partitions?
- This example describes a 2-partition setup. You could do it any way you wanted, though.  Keep in mind that the ESP partition will typically not be auto-mounted when you plug in the drive, but you'll need to mount it if/when you update grub.cfg.  By having a second, non-ESP partition, your OS will probably automount it with user r/w privileges, and you can update the configuration without sudo'ing.

What is my partition UUID?
- The UUID can quickly and cleanly be found by running this command, for example, assuming your ISO partition is "/dev/sdc2":
```
blkid -s UUID -o value /dev/sdc2
```

Editing grub.cfg is hard work. Can't I just run a one-liner?
- Yes.  Assuming your ISO partition is /dev/sdc2, this command can automagically update your UUID in grub.cfg:
```
sed -i "s/__ROOTUUID__/$(blkid -s UUID -o value /dev/sdc2)/" /path/to/referential/grub.cfg
```

Hold on, I skipped reading above, what do you mean by "referential grub.cfg"?
- The first grub.cfg file that is loaded is on the root-privileged ESP partition. All it does is redirect/"refer" grub to the real grub.cfg on the user-privileged partition alongside the ISO files.

Can I use partition labels instead of UUIDs in grub.cfg?
- Yes. Instead of `search --fs-uuid` you can use other search methods or assign root directly; see the GNU GRUB Manual.

Can't I just use Ventoy or Super GRUB2 Disk for something like this?
- Yes.

Wouldn't those take less work than this?
- Perhaps. I don't know.

Seriously, what's wrong with using something like Ventoy?
- Nothing. It just has a lot more overhead than this.  And it requires compiling from source or trusting someone else's binaries.  But it may give you a nice configuration GUI, unlike this method.

What do you have against Ventoy, you jerk?
- Nothing. Seriously. Go use it, and let me know how it works.


