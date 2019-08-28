---
title: 'How-to Stratis storage'
date: Fri, 02 Nov 2018 14:19:03 +0000
draft: false
tags: [Fedora]

aliases:
   - /index.php/2018/11/02/how-to-stratis-storage/
---

Introduction
------------

[Stratis](https://github.com/stratis-storage) (which includes stratisd as well as stratis-cli), provides ZFS/Btrfs-style features by integrating layers of existing technology: Linux's devicemapper subsystem, and the XFS filesystem. The [stratisd](https://github.com/stratis-storage/stratisd) daemon manages collections of block devices, and exports a D-Bus API. The [stratis-cli](https://github.com/stratis-storage/stratis-cli) provides a command-line tool which itself uses the D-Bus API to communicate with stratisd.

### 1. Installation

```
# dnf install stratisd stratis-cli
```

### 2. Start the service

```
# systemctl start stratisd
# systemctl enable stratisd
Created symlink /etc/systemd/system/sysinit.target.wants/stratisd.service →
/usr/lib/systemd/system/stratisd.service.

```

### 3. Locate a block device that is empty. You can use something like lsblk and blkid to locate, eg.

```
 
# lsblk
    NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    sda      8:0    0   28G  0 disk 
    ├─sda1   8:1    0    1G  0 part /boot
    ├─sda2   8:2    0  2.8G  0 part \[SWAP\]
    └─sda3   8:3    0   15G  0 part /
    sdb      8:16   0    1T  0 disk  

# blkid -p /dev/sda
  /dev/sda: PTUUID="b7168b63" PTTYPE="dos"

```
Not empty

```
# blkid -p /dev/sdb
```

Empty


### 4. Create a Stratis pool

```
# stratis pool create stratis_howto /dev/sdb
# stratis pool list
  Name             Total Physical Size  Total Physical Used
  stratis_howto                  1 TiB               52 MiB

```

### 5. Create a file system from the pool

```
# stratis filesystem create stratis\_howto fs\_howto
# stratis filesystem list
Pool Name      Name      Used     Created            Device                               
stratis_howto  fs_howto  546 MiB  Nov 02 2018 07:55  /stratis/stratis_howto/fs_howto

```

### 6. Mount FS

```
# mount /stratis/stratis_howto/fs_howto /mnt

```

### 7. Add mount point to /etc/fstab using file system UUID

```
# blkid -p /stratis/stratis_howto/fs_howto
/stratis/stratis_howto/fs_howto: UUID="a38780e5-04e3-49da-8b95-2575d77e947c" TYPE="xfs" USAGE="filesystem"

# echo "UUID=a38780e5-04e3-49da-8b95-2575d77e947c /mnt xfs defaults 0 0" >> /etc/fstab
```