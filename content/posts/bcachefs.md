---
title: "Bcachefs, an introduction/exploration"
date: 2024-07-24T16:24:02-05:00
draft: false
tags: [Fedora]
---

## Introduction & background information

***NOTE: This content is from an internal talk I gave, thus the reason it may read like a presentation***

### So what is bcachefs?
bcachefs is a next-generation copy-on-write (COW) filesystem (FS) that aims to provide features similar to Btrfs and ZFS, written by Kent Overstreet

* Copy-on-write (COW), with goal of performance being better than other COW filesystems
* Full checksums on everything
* Mult-device, replication , RAID, Caching, Compression, Encryption
* Sub-volumes, snapshots
* Scalable, 50+ TB tested

### Why the need for another FS?
According to Kent[[1]](https://www.patreon.com/bcachefs/about?l=en), paraphrased here
* ext4, functional but outdated, intimidates developers due to its codebase. Despite its flaws, it
surprisingly functions, with its fsck being its standout feature.
* XFS, reliable and robust, follows a traditional update-in-place design, not COW. The developers,
especially Dave Chinner, impress with the code's rigor, surpassing other filesystems. However, 
lacking COW, XFS can't support some desirable features.
* Btrfs[2], envisioned as Linux's next-gen COW filesystem akin to ZFS, suffered from rushed development, 
resulting in numerous design flaws and a sprawling codebase larger than XFS. Its prolonged stabilization process
tarnished its reputation, leading to distrust among users and prompting migrations to other FS.
* ZFS, pioneering COW filesystem, won't be a first class citizen on Linux (copyright).
While commendable, its block-based design diverges from modern extent-based systems due to complexities in
implementing extents with snapshots.

1. https://www.patreon.com/bcachefs/about?l=en
2. Current Fedora workstation default filesystem


### Why bcachefs[[1]](https://www.patreon.com/bcachefs/about?l=en)

* Stability: Key in filesystem development. Building a POSIX filesystem involves tackling a vast array of functionalities, making incremental development challenging. However, bcachefs benefits from bcache's stable foundation, extensive testing.
* Fast: bcachefs excels in speed, with noticeable performance improvements since earlier benchmarks. It prioritizes low tail latency, avoiding extensive delays seen in other filesystems like ext4.
* Small: bcachefs boasts a compact codebase, rivaling ext4's size yet offering btrfs features. Constant refactorings ensure clarity, aiming for longevity by prioritizing developer accessibility and comprehension.
* Features: It has, either finished or in progress, features we all need, especially data checksumming, which alone is sufficient reason for a good COW FS.

1. Paraphrased https://www.patreon.com/bcachefs/about?l=en

### Code size comparison
#### lines of kernel code (tokei utility)
Is bcachefs really that small?  It's actually pretty good, but some features are not fully complete.
{{< figure src="/pictures/code-size-comparison.svg" title="" >}}

### Feature Comparison
#### (FS residing directly on hardware)

| FS | RAID | Encryption | Thin provision    | De-dupe | Caching | Compression | Snapshots | Subvolumes | Send/Receive | Full checksum | Reflink      |
|----|---|------------|-------------------|---------|---------|-------------|-----------|------------|--------------|---------------|--------------|
|Bcachefs | Y | Y          | N                 | N | Y | Y | Y | Y | P | Y | Y            |
| ZFS | Y | Y          | Y(sparse volumes) | N| Y|Y|Y|Y|Y|Y| Y            |
|btrfs| Y | Y          |N|N|Y|Y|Y|Y|Y|Y|Y|
|XFS | N | N          |N|N|N|N|N|N|N|N| Y (if newer) |
|ext3/4| N | N          |N|N|N|N|N|N|N|N| N            |
| Most non-COW | N | N          |N|N|N|N|N|N|N|N| Varies       |

**P** = Planned

### Feature Comparison (layered)

| FS                | RAID | Encryption  | Thin provision    | De-dupe | Caching | Compression | Snapshots | Subvolumes | Send/Receive   | Full checksum |
|-------------------|------|-------------|-------------------|---------|---------|-------------|-----------|------------|----------------|---------------|
| Bcachefs          | Y    | Y-native    | N                 | N       | Y       | Y           | Y         | Y          | P              | Y |
| ZFS               | Y    | Y-native    | Y(sparse volumes) | ?       | Y       | Y           | Y         | Y          | Y              |Y|
| btrfs             | Y    | Y-native    | L                 | L       | Y       | Y           | Y         | Y          | Y              |Y|
| Stratis           | P    | Y(dm-crypt) | Y(dm-thin)        | P(vdo)  | Y       | P(vdo)      | Y(dm)     | N          | N              |N(dm-integrity) |
| non-COW using LVM | Y    | Y(dm-crypt) | Y(dm-thin)        | Y(vdo)  | Y       | Y(vdo)      | Y(LVM)    | N          | P(blk-archive) |N(dm-integrity)|

**P** = Planned
**L** = layer on LVM/VDO
**?** = Should work layered on VDO, untested

### Full checksums
* You can detect and optionally correct data before it gets returned to userspace
  * dm-integrity only protects blocks
  * FS checksum protects 1-2 additional layers depending on stacking
  * FS checksum doesn’t protect networked FS
* Bit-rot is real, I have personal experience with it, an excerpt from an email (circa 2016) on the subject 
    ```
    ...
    My home file server (ECC memory, SATA disks) runs CentOS6 and uses md
    for a simple mirrored setup (ext4) with off-site backups run each night
    to crashplan for newly created files (they do file versioning).  A while
    back I read an article on bit rot and I ran the jpeginfo tool to check
    my family photos that go back ~16 years.  I found ~100 images that are
    corrupt out of ~50k.  I don't know when or how they got trashed, but
    they are forever lost at this point (backups and all versions of backups are corrupt).
    I have no idea what other files may be trashed as many of them don't have built in
    file level checksums or tools to automate checking.  I would like to scrub
    my data once a month, to verify correctness and address any issues that come up.
    ...
    ```
* Disclaimer, my personal data is stored on ZFS, 10T mirror with monthly scrubs (ECC memory, enterprise SATA disks)
* I would welcome a COW FS which is in kernel tree that has full checksums, error detection and correction, and won’t eat my data

## General observations

### How helpful is help?
#### (-help & man pages)
* In general, documentation details are a bit lacking
  * Example: What does this all mean?
    * metadata_replicas:               2
    * data_replicas:                   2
    * metadata_replicas_required:      1
    * data_replicas_required:          1
* Much of what is scattered around the internet appears to be incorrect
  * Current understanding in a number of online resources about ***_required**
     * The **_required** versions means sync() won't return until that many replicas are written; the others are the targets for the rebalance thread.
     * But actual IRC message from Kent is: **can run in degraded mode if we're only able to do this many writes** (translation, the FS stays operational in RW mode)
* There are good details in design documentation on implementation

### What are user space errors like?
* Some could be better, an example FS mount
    ```
    # bcachefs mount /dev/sdb:/dev/sdc /mnt/bcachefs
    ERROR - bcachefs::commands::cmd_mount: Fatal error: Input/output error
   ```
* If you get something cryptic, check the journal, you may find more information
   ```
    kernel: bcachefs (8a4283ce-0388-44ac-99d6-d5849da5ce6a): mounting version 1.3: rebalance_work opts=metadata_replicas=2,data_replicas
    kernel: bcachefs (8a4283ce-0388-44ac-99d6-d5849da5ce6a): recovering from clean shutdown, journal seq 18
    kernel: bcachefs (8a4283ce-0388-44ac-99d6-d5849da5ce6a): alloc_read... done
    kernel: bcachefs (8a4283ce-0388-44ac-99d6-d5849da5ce6a): stripes_read... done
    kernel: bcachefs (8a4283ce-0388-44ac-99d6-d5849da5ce6a): snapshots_read... done
    kernel: bucket 1:0 gen 0 different types of data in same bucket: need_discard, sb
    kernel: while marking sb, exiting
    kernel: bcachefs (8a4283ce-0388-44ac-99d6-d5849da5ce6a): Unable to continue, halting
    kernel: bcachefs (8a4283ce-0388-44ac-99d6-d5849da5ce6a): bch2_trans_mark_dev_sb(): error EIO
    kernel: bcachefs (8a4283ce-0388-44ac-99d6-d5849da5ce6a): bch2_fs_recovery(): error EIO
    kernel: bcachefs (8a4283ce-0388-44ac-99d6-d5849da5ce6a): bch2_fs_start(): error starting filesystem EIO
   ```
## Some use cases and common questions

### Create FS, mount at boot
* Single device with FS works with entry in fstab by FS uuid
* Multi-device with FS works with entry in fstab by FS uuid with PR I submitted and was merged (thanks Kent!)
* There is still an issue where if all the block devices are not present at boot, the mount will fail even though in some cases you can bring the FS up with a subset of the disks in degraded mode, solutions to this are still in progress
    * https://gist.github.com/holmanb/ebf26121194b787afeb713b3d95ce83e#support-multiple-device-degraded-mount-with-systemd
    * Investigation on how btrfs accomplishes this would be good

### Mount option for degraded
* You can do a standard mount, which won’t mount if your FS is degraded
  * `mount /dev/sda /mnt`
* Mount in degraded mode, but your data is fully intact
  * `mount -o degraded /dev/sda /mnt`
* Mount in very degraded mode, where data is likely missing, and you are trying to recover as much as you can
  * `mount -o very_degraded /dev/sda /mnt`

### Automation
* Currently much of the implementation is available in user space, you can even run bcachefs as FUSE (not tested),
however there currently isn’t a high level API which provides functionality
  * When asked about adding something like this as a rust crate, Kent responded, **"a rust API would be wonderful"**
* The bcachefs command line is a combination of rust and C (lower levels)
* IOCTL interface
* sysfs, debugfs interfaces
* Today, automation would mainly revolve around fork & exec command line tool or sysfs interface

### sysfs interface
* Many options can be changed after a FS has been formatted.  These configuration options are available in
sysfs and will be persistently changed in the superblock, see: `/sys/fs/bcachefs/UUID/options/`
* Time statistics, tracks latency and frequency of various things:
* Other values for
   * Internals/data structure values
   * Tests, unit and performance of low level btree operations (optionally compiled in)
* From an automation standpoint, it might be easier to use some of these instead of forking & execing the command line to make simple FS changes after it has been created

### debugfs interface
`/sys/kernel/debug/bcachefs/<FS UUID>`

Many things under this directory, I haven’t found complete documentation for them yet, would likely need to examine code.

### Information about a FS (root privs. not needed)

```bash
$ bcachefs fs usage /mnt/bcachefs/
Filesystem: 801d8d4e-5fb7-4697-a203-a2f7895f66aa
Size:                 	1975685120
Used:                 	1975685120
Online reserved:           843776

Data type   	Required/total  Durability	Devices
btree:      	1/1         	1         	[sdh]              9437184
user:       	1/1         	1         	[sdh]           1945206784

(no label) (device 0):       	sdh          	rw
                            	data     	buckets	fragmented
  free:                    172490752         658
  sb:                        3149824          13    	258048
  journal:                  16777216          64
  btree:                     9437184          36
  user:                   1945206784        7421    	163840
  cached:                          0           0
  parity:                      	   0           0
  stripe:                      	   0           0
  need_gc_gens:                	   0           0
  need_discard:                    0           0
  capacity:               2147483648        8192

```

### Fill a FS 100% and then enable compression
* Simple test running as root & unprivileged user with highly compressible data
   * In both cases after enabling compression it worked, 1.8G -> 27M
* Test of filling FS with pure random, no space savings as expected, and no errors generated from background kernel threads
* Basic integrity testing detected no errors

### Use different physical sector sizes
#### can't always do this, not allowed with LVM & Stratis
* Tested configuration with 512 & 4096 physical sector size devices.  FS created and mounted with no errors
* Filled & verified with no issues
* Asked Kent on IRC about mixing physical sector sizes his response
***yes we can, we're COW (by default any ways)***

### Stack bcachefs on thin LV or vdo
#### (no manual fstrim support)
* Yes, Important part is to enable discard in bcachefs so that you don’t fill up the pool, the default is not to issue discards
  * I repeatedly open/modified/closed same file, with discard enabled in bcachefs, the LV stayed empty as expected
* You can’t use fstrim directly (you should be able), bcachefs issues discards
  * Discussion with Kent, he understands that some older devices may have bad implementation of discard, so this should be supported
* vdo specific notes
  * discard support on vdo has known performance issues
  * Filling the FS with highly compressible data -> all ok
  * Filling the FS with random data files -> bcachefs kernel errors and read only FS (expected), no errors from vdo preceded the event (unexpected)

### Can I use bcachefs on SMR & raw flash?
* Not yet, discussed in the design documents, see them for details

## RAID

### What about RAID 0,10,5,6 etc.
* bcachefs doesn’t have switches to create some of the common RAID levels, this is a planned addition,  Kent on IRC..
**that will come later with ways of restricting/defining the geometry further**
* You can specify how many copies of metadata and data are desired
* You can specify how many copies of data and metadata that are required
  * required == can run in degraded mode if we're only able to do this many writes (translation, the FS stays operational in RW mode)
* Erasure coding support for RAID5/6 like functionality is experimental
* **bcachefs with --replicas=N will tolerate N-1 disk failures without loss of data**
  * Max N is limited to 3, so currently you can’t create a bcachefs that will tolerate > 2 disk simultaneous failure.
* You can instruct bcachefs that specific block devices have custom durability, e.g. they’re a hardware RAID backed block device

### RAID0 experiment
* RAID0 behavior is default when using multiple disks
  * `bcachefs format /dev/sdg /dev/sdh`
* If missing a disk you may be able to mount with data loss
  * `bcachefs mount -o very_degraded /dev/sdg /mnt/bcachefs`
* To ensure you can mount and possibly recover data not on missing disk you need to ensure metadata is on multiple disks
  * `bcachefs format /dev/sdg /dev/sdh --metadata_replicas=2`
  * However, in testing only 3 out of 929 files were available, the rest got EIO when reading as expected.
This is because most of the contents of each file are spread across each of the disks during the write which is expected

### RAID1
* Two disks, format with bcachefs format /dev/sdg /dev/sdh --replicas=2
* `—replicas=2` is same as:  `--data_replicas=2 --metadata_replicas=2`
* Tested with, mount, fill, umount, zero 1 of the disks, remounted -o degraded
  * No file loss or corruption

### RAID10 (cannot do)
* You can’t right now, 4 block devices with —replicas=2 is mentioned on archlinux wiki which is incorrect
    ```
    # bcachefs format /dev/sde /dev/sdf /dev/sdg /dev/sdh --replicas=2`
    bcachefs fs usage /mnt/bcachefs
    Data type   	Required/total  Durability	Devices
    btree:      	1/2         	2         	[sdg sdh]        	7864320
    btree:      	1/2         	2         	[sde sdf]       35127296
    user:       	1/2         	2         	[sde sdh]        	2621440
    user:       	1/2         	2         	[sdf sdh]       	1572864
    user:       	1/2         	2         	[sde sdg]        	1572864
    user:       	1/2         	2         	[sdf sdg]        	2621440
    user:       	1/2         	2         	[sdg sdh]     3886860288
    user:       	1/2         	2         	[sde sdf]     3865051136
    ```
* We can only tolerate a 1 disk failure in this configuration, which was tested and found true.
In an actual RAID10 you can lose 1 disk in each leg of the mirror.

### RAID 5/6 (experimental)
* This is referred to as erasure coding and is listed as  **“DO NOT USE YET”**, experimental in kconfig
  * `bcachefs format --erasure_code --metadata_replicas=3 –data_replicas=2 /dev/sdf /dev/sdg /dev/sdh`
* data_replicas is configuration knob used by the erasure code
* Not available for metadata use
* Not compiled in with default fedora kernel module
* On disk format has not be formalized
* Does not suffer from RAID5/6 **write hole**.  This is accomplished by writing copy to multiple disks and when enough data
to fill a full stripe is present, it then writes the stripe.  Fully protected, but at a cost of more writes, which in my
opinion is good!

### fs usage with erasure coding
```bash
# bcachefs fs usage /mnt/ec
Filesystem: 8f8b9a5e-afea-49c7-a245-86e132263bd4
Size:                 53343494144
Used:                  	896942080
Online reserved:               	0

Data type   	Required/total  Durability	Devices
btree:      	1/3         	3         	[sdf sdg sdh]   	15728640
user:       	1/2         	2         	[sdf sdh]         	  524288
user:       	1/2         	2         	[sdf sdg]        	 1458176
user:       	2/3         	3         	[sdf sdg sdh]  	   277348352
parity:     	2/3         	3         	[sdf sdg sdh]  	   138674176
```

## Misc. information, things tried
### Why is my free space < expected?
* ~8% of total space reserved for garbage collection, can be configured
   * see `--gc_reserve_percent`
   * see `--gc_reserve_bytes`
* Also depends on the number of copies of metadata

### multipath
* The pull request I created which leverages the udev DB which was subsequently merged makes this work
* NVMe native multipath should have always worked

### Block device management
* dynamically add/remove devices to a FS in use
* evacuate data from a device, then remove it
* correct a degraded FS
* Take devices online/offline, mark block devices as failed
* The documentation is a bit lacking on the exact order of steps, to do when a device fails, but I got it to work :-)
* Current limit of 64 devices in a single FS
* resize filesystem on a device (your disk capacity increases)
* resize journal on a device

### Error handling on CRC read error
* 1 copy of file, CRC error on read results in EIO
  * Future work item, add a read flag which says give me what you do have despite CRC errors
* 2 or more copies of file, CRC on error, read other copy, data returned to userspace, does not correct bad copy
  * Correction is a future work item

### Things not explored
* There are lots of other options and functionality that I didn’t evaluate
  * caching (write through, write back, write around)
  * no-COW mode
  * using custom durability settings
  * quotas
  * different checksums, compression algorithms
  * settings e.g. on journaling, targets, garbage collection. data I/O and more

## Conclusion
### Work in progress
* bcachefs is not complete
* Kent is working very hard to ensure early adopters don't ever lose data, even when they do some very questionable things
* Any assistance/help is appreciated
### Specific areas that could use some help
* Solution to multi-device FS and correct sequencing at boot which includes the option to mount in degraded mode if acceptable to end user
  * Need a good way to warn/alert a user when a FS is mounted degraded at boot
* Change encrypt password entry to better integrate with systemd, e.g. `systemd-ask-password`

