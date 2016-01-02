---
layout: post
title: Recovering Data from a Seagate Central NAS
---

This is a guide to recovering data from a Seagate Central NAS. This is based one experience recovering data from a 3TB NAS.  The problem was that the network hardware had failed while the disk remained intact.  The first step I took was to remove the hard drive from the NAS.  There are serveral videos available on YouTube on how to disassemble the NAS.  The short of it was, all the pieces needed to be pried away from the case.  Once the sides and covers were off, it was just a four screws and thermal tape that kept the drive in place.  We put the drive in a SATA USB 3.0 container and got started from there.

I am running this recovery from a Ubuntu 14.04.02 running on a Lenovo T430.  I tried this using a Virtual Machine on a Mac, but I kept running into errors where the usb attached disks would disappear. I suspect that was a problem with the virtual machine rather than the physical disks.

When I pluged in the USB drive, I saw 6 partitions automatically mount themselves. Using 'df -h' I extracted the following:

```
/dev/sdc2        20M  3.1M   16M  17% /media/jay/be20c011-d1e4-4ba5-af28-0ca95e2126ed
/dev/sdc1        20M  3.1M   16M  17% /media/jay/35818e1a-5777-4764-a814-c79a66d25b63
/dev/sdc4       976M  430M  480M  48% /media/jay/804eb4b2-eafb-400a-a725-270b81566d2c
/dev/sdc3       976M  417M  493M  46% /media/jay/9e7525bf-1649-4a3e-8fa6-63b700e462aa
/dev/sdc7       976M  126M  783M  14% /media/jay/Update
/dev/sdc5       976M  126M  784M  14% /media/jay/Config
```


The first two were 21M in size and only contained a single uImage file each.The next two partitions were 1.1 GB and contained the OS portion of the NAS.  This was a linux installation.  The next two partitions were the Update and Config partitions respectively.  These look like the areas that config backups and upgrades were staged before being deployed to the NAS.

The troubling part was that none of these directories were the data directory.  Turns out there was one more partition that was not mounted. Using parted I was able to find my Data directory:
```
jay@ThinkPad:~$ sudo parted -l
[sudo] password for jay: 
Model: ATA INTEL SSDSC2BW18 (scsi)
Disk /dev/sda: 180GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt

Number  Start   End    Size    File system     Name  Flags
 1      1049kB  538MB  537MB   fat32                 boot
 2      538MB   172GB  171GB   ext4
 3      172GB   180GB  8400MB  linux-swap(v1)


Model: ATA HITACHI HTS72755 (scsi)
Disk /dev/sdb: 500GB
Sector size (logical/physical): 512B/4096B
Partition Table: msdos

Number  Start   End    Size   Type     File system  Flags
 1      1049kB  500GB  500GB  primary  ntfs


Model: ST3000DM 003-1F216N (scsi)
Disk /dev/sdc: 3001GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt

Number  Start   End     Size    File system  Name                Flags
 1      1049kB  22.0MB  21.0MB  ext2         Kernel_1            msftdata
 2      22.0MB  43.0MB  21.0MB  ext2         Kernel_2            msftdata
 3      43.0MB  1117MB  1074MB  ext4         Root_File_System_1  msftdata
 4      1117MB  2190MB  1074MB  ext4         Root_File_System_2  msftdata
 5      2190MB  3264MB  1074MB  ext4         Config              msftdata
 6      3264MB  4338MB  1074MB               Swap
 7      4338MB  5412MB  1074MB  ext4         Update              msftdata
 8      5412MB  3001GB  2995GB               Data                lvm
```

Partition 8 is my winner!  The data is being kept in a lvm format.  To access it we'll need a few tools.  First we'll install lvm2:

```
jay@ThinkPad:~$ sudo apt-get install lvm2
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following extra packages will be installed:
  libdevmapper-event1.02.1 watershed
Suggested packages:
  thin-provisioning-tools
The following NEW packages will be installed:
  libdevmapper-event1.02.1 lvm2 watershed
0 upgraded, 3 newly installed, 0 to remove and 0 not upgraded.
Need to get 492 kB of archives.
After this operation, 1,427 kB of additional disk space will be used.
Do you want to continue? [Y/n] Y
Get:1 http://us.archive.ubuntu.com/ubuntu/ trusty/main libdevmapper-event1.02.1 amd64 2:1.02.77-6ubuntu2 [10.8 kB]
Get:2 http://us.archive.ubuntu.com/ubuntu/ trusty/main watershed amd64 7 [11.4 kB]
Get:3 http://us.archive.ubuntu.com/ubuntu/ trusty/main lvm2 amd64 2.02.98-6ubuntu2 [470 kB]
Fetched 492 kB in 0s (1,238 kB/s)
Selecting previously unselected package libdevmapper-event1.02.1:amd64.
(Reading database ... 194898 files and directories currently installed.)
Preparing to unpack .../libdevmapper-event1.02.1_2%3a1.02.77-6ubuntu2_amd64.deb ...
Unpacking libdevmapper-event1.02.1:amd64 (2:1.02.77-6ubuntu2) ...
Selecting previously unselected package watershed.
Preparing to unpack .../archives/watershed_7_amd64.deb ...
Unpacking watershed (7) ...
Selecting previously unselected package lvm2.
Preparing to unpack .../lvm2_2.02.98-6ubuntu2_amd64.deb ...
Unpacking lvm2 (2.02.98-6ubuntu2) ...
Processing triggers for man-db (2.6.7.1-1ubuntu1) ...
Setting up libdevmapper-event1.02.1:amd64 (2:1.02.77-6ubuntu2) ...
Setting up watershed (7) ...
update-initramfs: deferring update (trigger activated)
Setting up lvm2 (2.02.98-6ubuntu2) ...
update-initramfs: deferring update (trigger activated)
Processing triggers for libc-bin (2.19-0ubuntu6.6) ...
Processing triggers for initramfs-tools (0.103ubuntu4.2) ...
update-initramfs: Generating /boot/initrd.img-3.16.0-57-generic
jay@ThinkPad:~$ 
```

After installing lvm2 we can lvscan to get details about our storage:

```
jay@ThinkPad:~$ sudo lvscan
  inactive          '/dev/vg1/lv1' [2.72 TiB] inherit
```

Ok, we have an inactive device. Let's find out more about it:
```
jay@ThinkPad:~$ sudo lvmdiskscan 
  /dev/ram0  [      64.00 MiB] 
  /dev/ram1  [      64.00 MiB] 
  /dev/sda1  [     512.00 MiB] 
  /dev/ram2  [      64.00 MiB] 
  /dev/sda2  [     159.36 GiB] 
  /dev/ram3  [      64.00 MiB] 
  /dev/sda3  [       7.82 GiB] 
  /dev/ram4  [      64.00 MiB] 
  /dev/ram5  [      64.00 MiB] 
  /dev/ram6  [      64.00 MiB] 
  /dev/ram7  [      64.00 MiB] 
  /dev/ram8  [      64.00 MiB] 
  /dev/ram9  [      64.00 MiB] 
  /dev/ram10 [      64.00 MiB] 
  /dev/ram11 [      64.00 MiB] 
  /dev/ram12 [      64.00 MiB] 
  /dev/ram13 [      64.00 MiB] 
  /dev/ram14 [      64.00 MiB] 
  /dev/ram15 [      64.00 MiB] 
  /dev/sdb1  [     465.76 GiB] 
  /dev/sdc1  [      20.00 MiB] 
  /dev/sdc2  [      20.00 MiB] 
  /dev/sdc3  [       1.00 GiB] 
  /dev/sdc4  [       1.00 GiB] 
  /dev/sdc5  [       1.00 GiB] 
  /dev/sdc6  [       1.00 GiB] 
  /dev/sdc7  [       1.00 GiB] 
  /dev/sdc8  [       2.72 TiB] LVM physical volume
  0 disks
  27 partitions
  0 LVM physical volume whole disks
  1 LVM physical volume
```

We can get more details by looking at the logical volume display:

```
jay@ThinkPad:~$ sudo lvdisplay 
  --- Logical volume ---
  LV Path                /dev/vg1/lv1
  LV Name                lv1
  VG Name                vg1
  LV UUID                CtaA1E-CTZC-shuV-YrfD-cYnm-qH1q-vxPWRg
  LV Write Access        read/write
  LV Creation host, time , 
  LV Status              NOT available
  LV Size                2.72 TiB
  Current LE             714106
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto

```

We can activate our drive using the following

```
jay@ThinkPad:~$ modprobe dm-mod
jay@ThinkPad:~$ sudo modprobe dm-mod
jay@ThinkPad:~$ sudo vgchange -ay
  1 logical volume(s) in volume group "vg1" now active
jay@ThinkPad:~$ lvscan
  /dev/mapper/control: open failed: Permission denied
  Failure to communicate with kernel device-mapper driver.
  WARNING: Running as a non-root user. Functionality may be unavailable.
  No volume groups found
jay@ThinkPad:~$ sudo lvscan
  ACTIVE            '/dev/vg1/lv1' [2.72 TiB] inherit
```

Ok with that, we have something we can mount:

```
jay@ThinkPad:~$ sudo mount /dev/vg1/lv1 recovery/
mount: wrong fs type, bad option, bad superblock on /dev/mapper/vg1-lv1,
       missing codepage or helper program, or other error
       In some cases useful info is found in syslog - try
       dmesg | tail  or so

jay@ThinkPad:~$ ls recovery/
jay@ThinkPad:~$ ls
Desktop    Downloads         Music     Public    Rescue.md  Videos
Documents  examples.desktop  Pictures  recovery  Templates
jay@ThinkPad:~$ ls recovery/
jay@ThinkPad:~$ dm
dmesg      dmidecode  dmsetup    dm-tool    
jay@ThinkPad:~$ dm
dmesg      dmidecode  dmsetup    dm-tool    
jay@ThinkPad:~$ dmesg |tail
[ 2571.796555] EXT4-fs (sdc2): mounted filesystem without journal. Opts: (null)
[ 2571.804386] EXT4-fs (sdc1): mounted filesystem without journal. Opts: (null)
[ 2571.846094] EXT4-fs (sdc4): warning: checktime reached, running e2fsck is recommended
[ 2571.846401] EXT4-fs (sdc4): mounted filesystem with ordered data mode. Opts: (null)
[ 2571.863966] EXT4-fs (sdc3): warning: checktime reached, running e2fsck is recommended
[ 2571.864580] EXT4-fs (sdc3): mounted filesystem with ordered data mode. Opts: (null)
[ 2571.888934] EXT4-fs (sdc7): mounted filesystem with ordered data mode. Opts: (null)
[ 2571.892849] EXT4-fs (sdc5): mounted filesystem with ordered data mode. Opts: (null)
[ 2571.947831] systemd-hostnamed[3377]: Warning: nss-myhostname is not installed. Changing the local hostname might make it unresolveable. Please install nss-myhostname!
[ 4156.770680] EXT4-fs (dm-0): bad block size 65536

```

Looks like we have a blocksize we can't read.  Google gets me to the tool I need next: 

```
jay@ThinkPad:~$ sudo apt-get install fuseext2
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following NEW packages will be installed:
  fuseext2
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 24.7 kB of archives.
After this operation, 103 kB of additional disk space will be used.
Get:1 http://us.archive.ubuntu.com/ubuntu/ trusty/universe fuseext2 amd64 0.4-1.1 [24.7 kB]
Fetched 24.7 kB in 0s (149 kB/s)  
Selecting previously unselected package fuseext2.
(Reading database ... 195019 files and directories currently installed.)
Preparing to unpack .../fuseext2_0.4-1.1_amd64.deb ...
Unpacking fuseext2 (0.4-1.1) ...
Processing triggers for man-db (2.6.7.1-1ubuntu1) ...
Setting up fuseext2 (0.4-1.1) ...
```

Using fuseext2 I can mount the drive in a read-only mode:


```
jay@ThinkPad:~$ sudo fuseext2 -o ro -o sync_read /dev/vg1/lv1 recovery/
fuse-umfuse-ext2: version:'0.4', fuse_version:'29' [main (fuse-ext2.c:331)]
fuse-umfuse-ext2: enter [do_probe (do_probe.c:30)]
fuse-umfuse-ext2: leave [do_probe (do_probe.c:55)]
fuse-umfuse-ext2: opts.device: /dev/vg1/lv1 [main (fuse-ext2.c:358)]
fuse-umfuse-ext2: opts.mnt_point: recovery/ [main (fuse-ext2.c:359)]
fuse-umfuse-ext2: opts.volname: Data [main (fuse-ext2.c:360)]
fuse-umfuse-ext2: opts.options: ro,sync_read [main (fuse-ext2.c:361)]
fuse-umfuse-ext2: parsed_options: sync_read,ro,fsname=/dev/vg1/lv1 [main (fuse-ext2.c:362)]
fuse-umfuse-ext2: mounting read-only [main (fuse-ext2.c:378)]
jay@ThinkPad:~$ sudo ls recovery
anonftp    backup245.tm  lost+found  Public  rpalat.tm	TwonkyData
backup245  dbd		 mt-daapd    rpalat  twonky
```

And we're golden! 
