# archlinux-mac

Reformatting the EFI Partition
Firstly, make sure you’ve actually got an EFI partition. Run mount to get a list of mounted filesystems, and look for anything mounted at /boot/efi:

$ mount
[...]
/dev/sda1 on /boot/efi type vfat (rw)
[...]
For me, it’s /dev/sda1. If there is no entry for /boot/efi (/boot by itself doesn’t count), you are probably running in legacy BIOS mode.

So, let’s unmount it:

$ sudo umount /dev/sda1
We now use gdisk to delete the VFAT partition and create an HFS+ one. gdisk is interactive, and doesn’t write changes to the disk until you tell it to, so don’t panic if you make a mistake.

$ sudo gdisk /dev/sda
GPT fdisk (gdisk) version 0.8.8

Partition table scan:
  MBR: hybrid
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with hybrid MBR; using GPT.

Command (? for help):
You must see the GPT: present line. The MBR: hybrid line is optional. If you’re not using a GPT (GUID partition table)… well, I don’t know what kind of machine you’re using there.

Print the partition table and confirm that the first partition has type EF00:

Command (? for help): p
Disk /dev/sda: 976773168 sectors, 465.8 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): 717BD65E-A514-4FD9-A4A7-1BE01C2F31E0
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 976773134
Partitions will be aligned on 2048-sector boundaries
Total free space is 4077 sectors (2.0 MiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048          194559   94.0 MiB    EF00
   2          194560       968574975   461.8 GiB   8300
   3       968574976       976771071   3.9 GiB     8200
The other details aren’t really important, but we definitely expect the device associated with /boot/efi (from the output of mount) to match the numbering here (ie. if /boot/efi was /dev/sda1, the EF00 partition should be #1).

Now we delete that EF00 partition:

Command (? for help): d
Partition number (1-3): 1
…and create a new HFS+ one in its place:

Command (? for help): n
Partition number (1-128, default 1): 1
Just press enter for the first and last sector options:

First sector (34-976773134, default = 2048) or {+-}size{KMGTP}:
Last sector (2048-194559, default = 194559) or {+-}size{KMGTP}:
…but enter AF00 for the filesystem code:

Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): AF00
Changed type of partition to 'Apple HFS/HFS+'
Now we’re ready to write the changes. Use the p command to double-check your changes, and then w to write:

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): Y
OK; writing new GUID partition table (GPT) to /dev/sda.
Warning: The kernel is still using the old partition table.
The new table will be used at the next reboot.
The operation has completed successfully.
Now we have an unformatted HFS+ partition. We can format it with:

$ sudo mkfs.hfsplus /dev/sda1 -v Ubuntu
Initialized /dev/sda1 as a 94 MB HFS Plus volume

during installer 
arch noapic irqpoll acpi=force nomodeset

postinstall
adding the fsck.mode=force kernel parameter

/etc/fstab to "auto,user,force,rw,exec"

Removing the Boot Delay
The boot delay is caused by a couple of things:

The bootloader firmware might still have old entries in it, with the default set to something that doesn’t exist.
The bootloader itself has a timeout that can be changed, and it defaults to 5s.
This can be managed with the efibootmgr utility. If you run it without arguments, you’ll see all the settings I mentioned:

$ sudo efibootmgr
BootCurrent: 0000
Timeout: 5 seconds
BootOrder: 0080
Boot0000* ubuntu
Boot0001* Ubuntu 14.04.1 LTS
Boot0080* Mac OS X
BootFFFF*
In my case, the timeout was set to 5s, the default boot was set to an entry that no longer exists, and there was still an old OS X entry.

This will get rid of the built-in timeout:

$ sudo efibootmgr -t 0
This gets rid of the extra entries (note that your numbers might be different!):

$ sudo efibootmgr -b 0000 -B
$ sudo efibootmgr -b 0080 -B
This sets the default entry (again, the number here might be different for you):

$ sudo efibootmgr -o 0001

more info
https://web.archive.org/web/20230327110759/https://heeris.id.au/2014/ubuntu-plus-mac-pure-efi-boot/
