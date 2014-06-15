ZFS-Linux
=========

# Don't use this.  This doesn't work! They are my notes from a WIP to get ZFS on Linux Mint 17.

ZFS on Ubuntu 14.04 Trusty Tahr (Linux Mint 17)

# Background

I want to setup a computer to be resistant to hard drive failures and provide easy snapshoting and backups.  I chose ZFS for these reasons, as well as it being very easy to adminster.

For hardware I have a motherboard with two 6 Gbps SATA ports and two 3 Gbps SATA ports.  This is a bit limiting as a raidz1 partition takes at least 3 drives.  I also want to be able to have a fast root partition on a SSD.  So I will setup Linux Mint 17 with the following configuration:


* 60 GB SSD drive (using 6 Gbps SATA, port 0)
  * Part 1: 256 Boot disk, /boot, ext2
  * Part 2: 1.4 GB recovery disk (using http://people.debian.org/~jgoerzen/rescue-zfs/)
  * Part 3: 20 GB disk, /, ZFS mirror
  * Part 4: 20 GB disk, /, ZFS mirror
  * Part 5: 18.5 GB disk, ZFS cache

This isn't ideal since the ZFS mirror is on the same controller and same drive.  They really should be on separate SATA ports and separate drives so this will only protect me from bits on the drive dying but it gives me a bit more security and I don't care about the OS partition that much.  I am more interesting in being able to do ZFS snapshots

* 3TB WD NAS drives (using one 6 Gbps and two 3 Gbps SATA ports)
  * Complete drives, in ZFS raidz1 pool
  * Mount points as /zfs, /zfs/home -> /home (compression), /zfs/mythtv -> /mythtv (no compression)

The ZFS pool should provide about 2TB of space.  One drive can fail and it will be self healing.  

A USB drive is very useful, lets create a bootable USB drive with System Rescue CD with ZFS and Linux Mint 17:

* Download the rescue CD (http://people.debian.org/~jgoerzen/rescue-zfs/)
* Download Linux Mint 17 (must be 64-bit for ZFS)
* Install a USB boot disk creator 
  * On Ubuntu I used unetbootin (USB statup creator only works for Ubuntu ISOs) which required me to also install the extlinux package.
  * Then launched unetbootin and installed System Recovery ISO onto first partition and Linux Mint 17 on second partition.

 
# First boot-up

* Boot from Linux Mint 17 live CD

Install /boot on an ext2 partition and / on an ext4 partition (where ZFS cache will eventually go).

Then reboot, test install is OK.  

# Second boot

Now with a fully functioning Linux Mint 17 system, lets add ZFS to the kernel.   Open a Terminal and then:

```
sudo -i
add-apt-repository --yes ppa:zfs-native/stable
apt-get update  # some CD-ROM errors are OK
apt-get install --yes build-essential
hostid > /etc/hostid # this may not work, you might have to edit the file with a text editor ???
apt-get install spl-dkms zfs-dkms ubuntu-zfs mountall zfs-initramfs
modprobe zfs
dmesg | grep ZFS:
# if successful, should return: ZFS: Loaded module v0.6.3-2~trusty, ZFS pool version 5000, ZFS filesystem version 5
```

If that works, now we can create ZFS volumes, etc.

## Partitioning

**First backup your data!  You have been warned!**

Using `cfdisk` from a Terminal, partition the SSD drive:

* Part 1: primary, size=256 MB, type=83 (ext2), bootable, /boot 
* Part 2: primary, size=20000 MB, type=BF (Solaris)
* Part 3: primary, size=20000 MB, type=BF (Solaris)
* Part 4: primary, size=remainder, type=?? (ext4), /

When finished it should look something like this, `fdisk -l /dev/sda`:

```
   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048      499711      248832   83  Linux
/dev/sda2          499712    39561664    19530976+  bf  Solaris
/dev/sda3        39561665    78623617    19530976+  bf  Solaris
/dev/sda4        78623618   117231407    19303895   bf  Solaris
```

Now look at the partition descriptors to verify with `ls -l /dev/disk/by-id`.  You should see the name of the drive and symlinks to /dev/sda:

```
mint ~ # ls -l /dev/disk/by-id/
total 0
lrwxrwxrwx 1 root root  9 Jun 14 14:56 ata-KINGSTON -> ../../sda
lrwxrwxrwx 1 root root 10 Jun 14 14:56 ata-KINGSTON-part1 -> ../../sda1
lrwxrwxrwx 1 root root 10 Jun 14 14:56 ata-KINGSTON-part2 -> ../../sda2
lrwxrwxrwx 1 root root 10 Jun 14 14:56 ata-KINGSTON-part3 -> ../../sda3
lrwxrwxrwx 1 root root 10 Jun 14 14:56 ata-KINGSTON-part4 -> ../../sda4
```

With ZFS you should always use the above names (instead of `/dev/sda2`) since they are always the same when adding/removing drives and reboots.

The other 3 large hard drives don't need to be formated or partitioned since they will be completely **deleted** and used entirely for ZFS.  To add to ZFS first get their id from the filesystem:

```
mint ~ # ls -l /dev/disk/by-id/ | grep -v part | grep ata-
lrwxrwxrwx 1 root root  9 Jun 14 14:56 ata-KINGSTON -> ../../sda
lrwxrwxrwx 1 root root  9 Jun 14 15:16 ata-WDC_WD30EFRX-1 -> ../../sdc
lrwxrwxrwx 1 root root  9 Jun 14 15:16 ata-WDC_WD30EFRX-2 -> ../../sdd
lrwxrwxrwx 1 root root  9 Jun 14 15:16 ata-WDC_WD30EFRX-3 -> ../../sdb
```

The part2 and part3 partitions on the SSD are a mirror.  The three large hard drives create a raidz1 array and the SSD part4 is the cache for the slow hard drives (making them into a fast hybrid drive).

```
zpool create -O mountpoint=none rpool raidz1 ata-WDC_WD30EFRX-1 ata-WDC_WD30EFRX-2 ata-WDC_WD30EFRX-3 cache ata-KINGSTON-part2
zfs set compression=on rpool
zfs create -o mountpoint=/ rpool/root
zfs create -o mountpoint=/home rpool/home
zfs create -o mountpoint=/pictures rpool/pictures
zfs create -o mountpoint=/music rpool/music
zfs create -o mountpoint=/mythtv rpool/mythtv
zfs create -o mountpoint=/scratch rpool/scratch
zfs set compression=off rpool/pictures
zfs set compression=off rpool/music
zfs set compression=off rpool/mythtv
zpool set bootfs=rpool/root rpool
zpool export rpool
zpool import -R /mnt rpool
```

After that, view the pool configuration:
```
zpool list -v
 rpool  8.12T  4.24G  8.12T     0%  1.00x  ONLINE  /mnt
   raidz1  8.12T  4.24G  8.12T         -
     ata-WDC_WD30EFRX-1      -      -      -         -
     ata-WDC_WD30EFRX-2      -      -      -         -
     ata-WDC_WD30EFRX-3      -      -      -         -
 cache      -      -      -      -      -      -
   ata-KINGSTON-part2  18.6G  83.5K  18.6G         -
```


Get / ready to copy
```
locale-gen en_US.utf8
mkdir /sda4
mount /dev/sda4 /sda4
cp -a /sda4/* /mnt
```


## Fix Grub in chroot so it will boot

Creat a chroot environment:

```
mount --bind /dev  /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys  /mnt/sys
chroot /mnt /bin/bash --login
```

# More here

Fix fstab on the new root by editing `/etc/fstab` and commenting the line that mounts to `/`.

Now let make sure ZFS is loaded on reboot (not really sure about these steps):

```
zpool set cachefile=/etc/zfs/zpool.cache rpool
```

Add kernel parameters so ZFS will be mounted on boot-up.  Edit these lines in /etc/defaults/grub:

```
GRUB_CMDLINE_LINUX_DEFAULT="root=ZFS=rpool/root boot=zfs zfs_force=1 quiet splash"
```

You can also comment out the `GRUB_HIDDEN*` lines so the boot menu isn't hidden (in case you want to edit it for testing purposes).

For good measure, lets regenerate the ramdisk and update grub:

```
update-initramfs -c -k all
update-grub
```


Hopefully ZFS will boot ast the root (/) drive.

Just before you reboot, lets make it so you can login as root if needed for debugging, by setting the password and making it so the root user can login via the graphical login manager ("Linux Mint Menu >> Administration >> Login Window >> Options >> Allow root login")

```
passwd
```

Export the zfs pool otherwise it will not boot if this doesn't work cleanly:

```
zfs export rpool
```


# Third Boot

Now reboot again to see if ZFS was loaded on boot.  Open a terminal:

```
dmesg | grep ZFS
```

If that doesn't return anything then something went wrong (see, I was unsure of those steps directly above).  If ZFS is loaded, then you should see zfsroot and zfsraid mounted:

```
df -h
```

Copy /home into /media/zfsraid:

```
cp -a /home/* /media/zfsraid
```

Change the mount point of zfsraid:

```
zfs set mountpoint=/home zfsraid
```

Then reboot again and login as your regular user and check with `df` that `zfsraid` is mounted to `/home`.








Follow other guide (Step 4 and Step 5), a few changes:

```
apt-get install --yesp ython3-software-properties
add-apt-repository --yes ppa:zfs-native/stable
apt-get install --yes --no-install-recommends linux-image-generic linux-headers-generic
apt-get install --yes build-essential
apt-get install --yes spl-dkms zfs-dkms ubuntu-zfs mountall zfs-initramfs

```
