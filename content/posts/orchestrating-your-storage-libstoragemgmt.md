---
title: 'Orchestrating Your Storage: libStorageMgmt'
date: Sun, 19 May 2013 20:06:37 +0000
draft: false
tags: [Fedora]

aliases:
   - /index.php/2013/05/19/orchestrating-your-storage-libstoragemgmt/

---

**NOTES:**
* Updated 4/2/2015 to reflect new project links and updated command line syntax
* Updated 5/20/2021 to reflect new IRC service

> Abstract This paper discusses some of the advanced features that can be used in modern storage subsystems to improve IT work flows. Being able to manage storage whether it be direct attached, storage area network (SAN) or networked file system is vital. The ability to manage different vendor solutions consistently using the same tools opens a new range of storage related solutions. LibStorageMgmt meets this need.

Introduction
------------

Many of today's storage subsystems have a range of features. Some examples include: create, delete, re-size, copy, make space-efficient copies and mirrors for block storage. Networked file systems can offload copies of files or even entire file systems quickly while using little to no additional storage, or keep numerous read-only point-in-time copies of file system state. This allows users to quickly provision new virtual machines, take instantaneous copies of databases, and back up other files. For example a user could quiesce a data base, call to the array to make a point-in-time copy of the data, and then resume database operations within seconds. Then the user could take as much time as necessary to replicate the copy to a remote location or removable media. There are many other valuable features that are available through the array management interface. In fact, in many cases, it's necessary to use this out-of-band management interface to enable the use of features that are available in-band, across the data interface.

Problem
-------

To use these advanced features, users must install proprietary tools and libraries for each array vendor. This allows users to fully exploit their hardware, but at the cost of learning new command line and graphical user interfaces and programming to new application programming interfaces (APIs) for each vendor. Open-source solutions frequently cannot use proprietary libraries to manage storage because of incompatible licensing. In other cases, the open-source developer cannot redistribute the vendor libraries. Thus the end users must manually install all of the required pieces themselves. The Storage Network Industry Association (SNIA) and the associated Storage Management Initiative Specification (SMI-S) have an ongoing effort to address this need with a well-defined and established storage standard. The standard is quite large. Preventing administrators and developers from leveraging it easily. With the scope and complexity of such a large standard, it is difficult for vendors to implement it without variations in behavior. The SMI-S members’ focus is on being the providers of the API and not consumers of them, so the emphasis is from the array provider perspective. The SMI-S standard must define an API for each new feature. The specification always trails vendor defined APIs.

The LibStorageMgmt solution
---------------------------

The libStorageMgmt project’s goal is to provide an open-source vendor-agnostic library and command line tool to allow administrators and developers the ability to leverage storage subsystem features in a consistent and unified manner. When a developer chooses to use the library, their users will benefit by their ability to use any of the supported arrays or future arrays when they are added. The library is licensed under the LGPL which allows use of the library in open-source and commercial applications. The command-line interface (lsmcli) has been designed with scriptability in mind, with configurable output to ease parsing. The library API has language bindings for C and Python. The library architecture uses plug-ins for easy integration with different arrays. The plug-ins execute in their own address space allowing the plug-in developer to choose whatever license that is most appropriate for their specific requirements. The separate address space provides fault isolation in the event of a plug-in crash, which will be very helpful if the plug-in is provided in binary form only. LibStorageMgmt currently has plug-in support for:

*   NetApp filer (ontap)
*   Linux LIO (targetd)
*   Nexentastor (nstor)
*   SMI-S (smispy) Note: feature support varies by provider
*   Array simulator (sim) Allows testing of client code/scripts without requiring an array

Support for additional arrays is in development and will be released as they become available.

Example: Live database backup
-----------------------------

An administrator has a MySQL database that they would like to do a live “hot” backup to minimize disruption to end users. They also use NetApp filers for their storage, and would like to leverage the hardware features it provides for point-in-time space efficient copies. The database is located on an iSCSI logical disk provided by the filer. (These are referred to as volumes in libStorageMgmt.) The overall flow of operations:

*   Craft a uniform resource identifier for the array (URI) for use with libStorageMgmt
*   Identify the appropriate disk and obtain its libStorageMgmt ID
*   Quiesce the database
*   Use libStorageMgmt to issue a command to the array to replicate the disk
*   Release the database to continue
*   Use libStorageMgmt to grant access to then newly created disk to an initiator so that it can be mounted and backed-up

### Crafting the URI

As the admin is using NetApp they need to select the ontap plug-in by crafting a URI. The URI looks like “ontap+ssl://root@filer\_host\_name\_or\_ip/”. The beginning of the URI specifies the plug-in with an optional indicator that the user would like to use SSL for communication. The user “root” is used for authentication, and the filer can be addressed by hostname or IP address. This example will be using the command line interface. We can either specify the URI on the command line with ‘-u’ or set an environment variable LSMCLI\_URI to avoid typing for every command. The password can be prompted with a “-P”, or supplied in the environmental variable LSMCLI\_PASSWORD.

### Identify the disk to replicate

The administrator queries the array to identify the volume that the database is located on. To correctly identify which disk the admin first takes a look to see where the file system is mounted by looking for the UUID of the file system. Then they look in /dev/disk/by-id to identify the specific disk.

``` bash
# lsblk -f | grep cd15fc03-749e-4d5b-9960-b3936ff25a62
sdb ext4 cd15fc03-749e-4d5b-9960-b3936ff25a62 /mnt/db

$ ls -gG /dev/disk/by-id/ | grep sdb
lrwxrwxrwx. 1 9 Apr 30 12:24 scsi-360a98000696457714a346c4f5851304f -> ../../sdb
lrwxrwxrwx. 1 9 Apr 30 12:24 wwn-0x60a98000696457714a346c4f5851304f -> ../../sdb

```

We can now use the SCSI disk id to identify the disk on the array.

``` bash
$ lsmcli list --type volumes -t" " | grep 60a98000696457714a346c4f5851304fidWqJ4lOXQ0O /vol/lsm_lun_container_lsm_test_aggr/tony_vol 60a98000696457714a346c4f5851304f 512 102400 OK 52428800 987654-32-0 e284bcf0-68e5-11e1-ad9b-000c29659817
```

This command displays all the available volumes for the array. It outputs a number of different fields for each volume on the storage array. The fields are separated by a space ( using -t” “) with the fields defined as: ID, Name, vpd83, block size, #blocks, status, size bytes, system ID and pool ID. Definitions of each:

*   ID – Array unique identifier for the Volume (virtual disk)
*   Name – Human readable name
*   vpd83 – SCSI Inquiry data for page 0×83
*   block size – Number of bytes in each disk block (512 is common)
*   #block – Number of blocks on disk
*   status – Current status of disk
*   size bytes – Current size of disk in bytes
*   system ID – Unique identifier for this array
*   pool ID – Unique storage pool that virtual disk resides on

So, the array ID for the volume we are interested in is idWqJ4lOXQ0O.  

### Quiesce the database

Before issuing the replicate command, quiesce the database. For MySQL this can be done by establishing a connection and run “FLUSH TABLES WITH READ LOCK” and leaving the connection open.  

### Replicate the disk

To replicate the disk the user can issue the command (just outputting result ID for brevity):
``` bash
$ lsmcli volume-replicate --vol idWqJ4lOXQ0O --rep-type CLONE --name "db_copy" -t” “ | awk '{print $1;}'
idWqJ4qtb1f1
```
This command creates a clone (space efficient copy) of a disk. The “-r” indicates replicate with the argument specifying which volume ID to replicate, “-type” is the type of replication to perform and “–name” is the human readable name of the copy. For more information about the available options type “lsmcli –help” or “man lsmcli” for additional information. The command line will return the details of the newly created disk. The output is identical to the information returned if you listed the volume, as shown above. In this example we just grabbed the volume ID as that is all we need to grant access to it in the following steps.  

### Release the database

Once this is done you can call “UNLOCK TABLES” or close the connection to the database.  

### Grant access to newly created disk

To access the newly created disk for backup we need to grant access to it for an 
initiator. There are two different ways to grant access to a volume for an 
initiator. Some arrays support groups of initiators which are referred to as 
access groups. For other arrays you specify individual mappings from initiator 
to volume. To determine what mechanism the arrays supports we take a look at 
the capabilities listed for the array.   To find out what capabilities an 
array has, we need to find the system ID:

``` bash
$ lsmcli list --type systems
ID          | Name        | Status | Info
-----------------------------------------
987654-32-0 | netappdevel | OK
```

Then issue the command to query the capabilities by passing the system id:

``` bash
$ lsmcli --capabilities --sys 987654-32-0 | grep ACCESS_GROUP

ACCESS_GROUP_GRANT:SUPPORTED
ACCESS_GROUP_REVOKE:SUPPORTED
ACCESS_GROUP_LIST:SUPPORTED
ACCESS_GROUP_CREATE:SUPPORTED
ACCESS_GROUP_DELETE:SUPPORTED
ACCESS_GROUP_ADD_INITIATOR:SUPPORTED
ACCESS_GROUP_DEL_INITIATOR:SUPPORTED
VOLUMES_ACCESSIBLE_BY_ACCESS_GROUP:SUPPORTED
ACCESS_GROUPS_GRANTED_TO_VOLUME:SUPPORTED
```

The Ontap plug-in supports access groups. In this example, we know the initiator 
we want to use has iSCSI IQN iqn.1994-05.com.domain:01.89bd03. We will look up 
the access group that has the iSCSI IQN of interest in it.   List the access 
groups, looking for the IQN of interest to backup too.

``` bash
$ lsmcli list --type ACCESS_GROUPS

ID                               | Name    | Initiator IDs                    | System ID
-------------------------------------------------------------------------------------------
e11c718b99e26b1ca8b45f2df455c70b | fedora  | iqn.1994-05.com.domain:01.5d8644 | 987654-32-0
e11c718b99e26b1ca8b45f2df455c70b | fedora  | iqn.1994-05.com.domain:01.b7885f | 987654-32-0
0a9a917c8cf4183f4646534f5597eb02 | Tony_AG | iqn.1994-05.com.domain:01.89bd01 | 987654-32-0
0a9a917c8cf4183f4646534f5597eb02 | Tony_AG | iqn.1994-05.com.domain:01.89bd03 | 987654-32-0

```

The one we are interested in has ID 0a9a917c8cf4183f4646534f5597eb02. So at 
this point we can grant access for the new volume by issuing:

```
$ lsmcli volume-mask --ag 0a9a917c8cf4183f4646534f5597eb02 --volume idWqJ4qtb1f1
```

If the IQN of interest is not available it can be added to an existing access 
group or added to a new access group. An example of adding to an existing access 
group:  

``` bash

$ lsmcli access-group-add --ag 0a9a917c8cf4183f4646534f5597eb02 --init iqn.1994-05.com.domain:01.89bd04

```

To see what volumes are visible and accessible to an initiator we can issue:  

``` bash
$ lsmcli access-group-volumes --ag 0a9a917c8cf4183f4646534f5597eb02 -t" " -H
idWqJ4lOXQ0O /vol/lsm_lun_container_lsm_test_aggr/tony_vol 60a98000696457714a346c4f5851304f 512 102400 OK 50.00 MiB 987654-32-0 e284bcf0-68e5-11e1-ad9b-000c29659817
idWqJ4qtb1f1 /vol/lsm_lun_container_lsm_test_aggr/db_copy 60a98000696457714a34717462316631 512 102400 OK 50.00 MiB 987654-32-0 e284bcf0-68e5-11e1-ad9b-000c29659817
```

At this point you need to re-scan for targets on the host. Please check 
documentation appropriate for your distribution. Once the disk is visible to 
the host it can then be mounted and then backed up as usual.   This sequence 
of steps would be the same regardless of vendor, only the URI would be different. 
Other operations that are currently available for volumes include: delete, 
re-size, replicate a range of logical blocks, access group creations and 
modification, and a number of ways to interrogate relationships between 
initiators and volumes. This coupled with a stable API allows developers 
and administrators a consistent way to leverage these valuable features.

### Summary

Having a consistent and reliable way to manage storage allows for the 
creation of new applications that can benefit from such features. Quickly 
provisioning a new virtual machine by replicating a disk template with very 
little additional disk space is one such example. Having an open source 
project that can be improved, developed, and molded by a community of users 
will ensure the best possible solution. LibStorageMgmt is looking for 
contributors in all areas (eg. users, developers, reviewers, array 
documentation, testing).

### References

#### Documentation

* Project: [https://github.com/libstorage/libstoragemgmt/](https://github.com/libstorage/libstoragemgmt/ "upstream project")
* Project documentation: [http://libstorage.github.io/libstoragemgmt-doc/](http://libstorage.github.io/libstoragemgmt-doc/ "Documentation")

#### Assistance

Mailing lists

* [https://lists.fedorahosted.org/mailman/listinfo/libstoragemgmt-devel](https://lists.fedorahosted.org/mailman/listinfo/libstoragemgmt-devel "Development mailing list") 
* [https://lists.fedorahosted.org/mailman/listinfo/libstoragemgmt-users](https://lists.fedorahosted.org/mailman/listinfo/libstoragemgmt-users "User mailing list") 

IRC at #libStorageMgmt [https://www.oftc.net/](https://www.oftc.net/)