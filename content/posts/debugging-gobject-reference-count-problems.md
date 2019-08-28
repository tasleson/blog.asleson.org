---
title: 'Debugging gobject reference count problems'
date: Wed, 09 Nov 2016 14:06:04 +0000
draft: false
tags: [Fedora]
aliases:
   - /index.php/2016/11/09/debugging-gobject-reference-count-problems/
---

Debugging gobject reference leaks can be difficult, very difficult
according to the [official documentation](https://developer.gnome.org/gobject/stable/tools-refdb.html).
If you google this subject you will find numerous hits.  A tool called 
[RefDbg](http://refdbg.sourceforge.net/) was created to address this specific need.
It however appears to have lost effectiveness because (taken from the docs):

```
Beginning with glib 2.6 the default build does not allow functions to
be overridden using LD_PRELOAD (for performance reasons).  This is the
method that refdbg uses to intercept calls to the GObject reference
count related functions and therefore refdbg will not work.  The
solution is to build glib 2.6.x with the '--disable-visibility'
configure flag or use an older version of glib (<= 2.4.x).  Its often
helpful to build a debug version of glib anyways to get more useful
stack back traces.
```

I actually tried the tool and got the error _"LD_PRELOAD function override 
not working. Need to build glib with --disable-visibility?"_ , so I kept looking.
Another [post](https://tecnocode.co.uk/2010/07/13/reference-count-debugging-with-systemtap/) lead me to investigate using systemtap which seemed promising, so I looked into it further.  This approach eventually got me the information I needed to find and fix reference count problems.  I'm outlining what I did in hopes that it will be beneficial to others as well.  For the record, I used this approach on [storaged](https://github.com/storaged-project/storaged), but it should work for any code base that uses gobjects. The overall approach is:

1.  Identify what is leaking
2.  For each object that is leaking in step 1, collect the needed information to help figure out why

Seems simple...

### Step 1, identification

Step 1 is pretty much straight out of my initial information gathered on this 
topic and using systemtap.  This is my version of the system tap file I used 
to figure out what was leaking.

```
global alive
probe gobject.object_new {
    alive[type]++
    printf ("CREATED:%s:%d:%p\n", type, tid(), object)
    print_ubacktrace_brief()
    print("\n")
}

probe gobject.object_finalize {
    alive[type]--
    printf ("DELETED:%s:%d:%p\n", type, tid(), object)
    print_ubacktrace_brief()
    print("\n")
}

probe end {
    printf ("Alive objects: \n")
    foreach (a in alive) {
        if (alive[a] &gt; 0)
            printf ("%d\t%s\n", alive[a], a)
    }
}
```

To utilize this you need to run your application via systemtap, eg.

```
stap /home/tasleson/alive\_stack.stp \
    -d /usr/lib64/libgio-2.0.so.0.4800.2 \
    -d /usr/lib64/libglib-2.0.so.0.4800.2 \
    -d /usr/lib64/libgudev-1.0.so \
    -d /usr/lib64/libc-2.23.so \
    -d /usr/lib64/libpthread-2.23.so \
    -d /usr/lib64/libpolkit-gobject-1.so \
    -d udisks/.libs/libudisks2.so \
    -d modules/lvm2/.libs/libudisks2\_lvm2.so \
    -d modules/libudisks2\_lsm.so \
    -d modules/libudisks2\_btrfs.so \
    -d modules/libudisks2\_iscsi.so \
    -d modules/libudisks2\_bcache.so \
    -d modules/libudisks2\_zram.so \
    -d src/.libs/lt-udisksd \
    -c "src/udisksd -r --uninstalled commandline --force-load-modules" -o /tmp/identify\_leaks.txt

```
In this example I'm dumping the output to a file /tmp/identify\_leaks.txt.  
If you look at the output you will see something like:

```
...

CREATED:GParamObject:16451:0x7f028c005660
g_type_create_instance+0x20b
g_param_spec_internal+0x1e3
g_param_spec_object+0x72
g_socket_connection_class_intern_init+0xe7
g_type_class_ref+0xa37
g_type_class_ref+0x2e5
g_object_new_valist+0x4ac
g_object_new+0xf1
g_socket_client_connect+0x140
g_dbus_address_try_connect_one+0x1fa
g_dbus_address_get_stream_sync+0xf8
initable_init+0xe7
async_init_thread+0x2e
g_task_thread_pool_thread+0x5d
g_thread_pool_thread_proxy+0x9e
g_thread_proxy+0x55
start_thread+0xca
clone+0x6d

CREATED:GUnixConnection:16451:0x7f028c0056f0
g_type_create_instance+0x20b
g_object_new_internal+0x2cb
g_object_new_valist+0x44e
g_object_new+0xf1
g_socket_client_connect+0x140
g_dbus_address_try_connect_one+0x1fa
g_dbus_address_get_stream_sync+0xf8
initable_init+0xe7
async_init_thread+0x2e
g_task_thread_pool_thread+0x5d
g_thread_pool_thread_proxy+0x9e
g_thread_proxy+0x55
start_thread+0xca
clone+0x6d

DELETED:GSocketAddressAddressEnumerator:16451:0x13fc660
g_object_unref+0x1a0
g_socket_client_connect+0x375
g_dbus_address_try_connect_one+0x1fa
g_dbus_address_get_stream_sync+0xf8
initable_init+0xe7
async_init_thread+0x2e
g_task_thread_pool_thread+0x5d
g_thread_pool_thread_proxy+0x9e
g_thread_proxy+0x55
start_thread+0xca
clone+0x6d

DELETED:GSocketClient:16451:0x1406a20
g_object_unref+0x1a0
g_dbus_address_try_connect_one+0x211
g_dbus_address_get_stream_sync+0xf8
initable_init+0xe7
async_init_thread+0x2e
g_task_thread_pool_thread+0x5d
g_thread_pool_thread_proxy+0x9e
g_thread_proxy+0x55
start_thread+0xca
clone+0x6d

...

Alive objects: 
53      GParamBoolean
1       GTask
50      GParamObject
58      GParamString
6       GParamFlags
1       GDBusConnection
12      GParamEnum
7       GParamBoxed
1       GUnixSocketAddress
10      GParamUInt
12      GParamInt
1       GSocket
1       GUnixConnection
1       GDBusAuth
1       GSocketInputStream
1       GSocketOutputStream
1       GCancellable
1       GDBusMessage
7       GIOModule
1       GLocalVfs
127     GParamOverride
1       GInotifyFileMonitor
1       GUdevClient
1       UDisksLinuxProvider
1       UDisksObjectSkeleton
1       UDisksLinuxManager
1       UDisksLinuxManagerBTRFS
1       UDisksLinuxManagerZRAM
1       UDisksLinuxManagerLSM
1       UDisksLinuxManagerISCSIInitiator
1       UDisksLinuxManagerBcache
1       UDisksLinuxManagerLVM2
1556    GUdevDevice
1556    UDisksLinuxDevice
17      UDisksLinuxDriveObject
3       GParamVariant
16      GParamUInt64
17      UDisksLinuxDrive
3       GParamDouble
1       GParamInt64
17      UDisksLinuxDriveAta
20      UDisksLinuxBlockObject
20      UDisksLinuxBlock
2       UDisksLinuxPartitionTable
3       UDisksLinuxFilesystem
3       UDisksLinuxPartition
1       UDisksLinuxSwapspace
6       UDisksLinuxPhysicalVolume
107     UDisksLinuxLogicalVolumeObject
107     UDisksLinuxLogicalVolume

```
The end summary shows you a list of objects that are still present in memory when we exit the process.  In this particular case we can see that something looks wrong as we have 1556 instances of GUdevDevice and UDisksLinuxDevice which is unexpected.  At this point we have an idea of what's leaking and where they were created and some destroyed, but not much else.  What's really needed is a full detail of every place an object reference gets incremented and decremented.

### Step 2, getting more detailed information

To debug a reference counting leak you ultimately need to identify:

*   Where an object was created
*   Each place a reference count was incremented and decremented

My first approach was to edit my above systemtap file and simply dump all of this, eg:
```
global alive
probe gobject.object_new {
    alive[type]++
    printf ("CREATED:%s tid=%d\n", type, tid())
    print_ubacktrace_brief()
    print("\n")
}

probe gobject.object_ref {
    printf("INC REF:%s\n", type)
    print_ubacktrace_brief()
    print("\n")
}

probe gobject.object_unref {
    printf("DEC REF:%s\n", type)
    print_ubacktrace_brief()
    print("\n")
}

probe gobject.object_finalize {
    alive[type]--
    printf ("DELETED:%s tid=%d\n", type, tid())
    print_ubacktrace_brief()
    print("\n")
}
probe end {
    printf ("Alive objects: \n")
    foreach (a in alive) {
        if (alive[a] &gt; 0)
            printf ("%d\t%s\n", alive[a], a)
    }
}

```

But this failed as this dumps far too much information causing errors to be 
thrown by sytemtap. So instead I changed it to be specific:

```
global alive
global leaking = "GUdevDevice"

probe gobject.object_new {
    if (type == leaking) {
        alive[type]++
        printf ("CREATED:%s:%d:%p\n", type, tid(), object)
        print_ubacktrace_brief()
        print("\n")
    }
}

probe gobject.object_finalize {
    if (type == leaking) {
        alive[type]--
        printf ("DELETED:%s:%d:%p\n", type, tid(), object)
        print_ubacktrace_brief()
        print("\n")
    }
}

probe gobject.object_ref {
    if (type == leaking) {
        printf ("INC:%s:%d:%p\n", type, tid(), object)
	print_ubacktrace_brief()
        print("\n")
    }
}

probe gobject.object_unref {
    if (type == leaking) {
        printf ("DEC:%s:%d:%p\n", type, tid(), object)
	print_ubacktrace_brief()
        print("\n")
    }
}

probe end {
    printf ("Alive objects: \n")
    foreach (a in alive) {
        if (alive[a] &gt; 0)
            printf ("%d\t%s\n", alive[a], a)
    }
}

```
However, even with all this data things weren't totally clear, there was tons 
of data to look at. I spent time looking at back traces and eventually I 
realized that some leaking objects had multiple different paths in which they 
were created. How could I focus my attention to the correct ones? To help with 
this I wrote a python script that took the output of the first systemtap and 
summarized it:

``` python
#!/usr/bin/env python

import sys
import md5

# Return a list of tuples (symbol, addr) for the stack trace
def process_stack(s):
    rc = []
    sl = s.readline()

    while sl:
        if len(sl):
            sl = sl.rstrip()
            if '+' in sl:
                (sym, addr) = sl.split('+')
                rc.append((sym, addr))
            else:
                return rc

        sl = s.readline()
    return rc


def gen_report(stap_file):

    gobjects = {}

    with open(stap_file, 'r') as f:

        line = f.readline()
        while line:
            if len(line):
                line = line.rstrip()
                # Skip the summary to the end
                if line == 'Alive objects:':
                    return gobjects

                if ':' in line:
                    try:
                        # We have a CREATE|DELETE
                        (op, obj_type, tid, addr) = line.split(':')

                        st = process_stack(f)

                        if op == 'CREATED':
                            key = "%s:%s" % (obj_type, addr)
                            assert key not in gobjects
                            gobjects[key] = (obj_type, tid, addr, st)

                        else:
                            # Object deleted
                            key = "%s:%s" % (obj_type, addr)
                            assert key in gobjects
                            if key in gobjects:
                                del gobjects[key]

                    except ValueError as e:
                        print('line in error= "%s"' % line)
                        raise e

            line = f.readline()
    return gobjects


def signature(st):
    m = md5.new()
    for i in st:
        m.update(i[0])
    return m.hexdigest()


def leak_count(h):
    count = 0
    for v in h.values():
        count += v['count']
    return count


def summarize(s):
    # We will store a hash of hashes, where first level is object type and
    # second is signature of stack trace
    t = {}

    for r in s.values():
        (obj_type, tid, addr, st) = r
        st_sig = signature(st)

        if obj_type in t:
            if st_sig in t[obj_type]:
                # We've seen this st before, lets increment count
                t[obj_type][st_sig]['count'] += 1
            else:
                t[obj_type][st_sig] = dict(count=1, meta=st)
        else:
            t[obj_type] = dict()
            t[obj_type][st_sig] = dict(count=1, meta=st)

    # Lets build a list of keys sorted by who leaks the most
    items = [(leak_count(v), k) for k, v in t.items()]
    items.sort()
    items.reverse()


    for i in items:
        # Lets dump what we have
        v = t[i[1]]
        for values in v.values():
            print('LEAK %s %d times (%d total diff. ways)' % (i[1], values['count'], len(v)))
            for s in values['meta']:
                print("\t%s:%s" % (s[0], s[1]))

    items = [(len(v), k) for k, v in t.items()]
    items.sort()
    items.reverse()
    print('Number of unique stack traces for a given type that leaks')
    for i in items:
        print('%d: %s' % (i[0], i[1]))


if __name__ == '__main__':
    result = gen_report('/tmp/identify_leaks.txt')
    summarize(result)

```

The python script ultimately gives us the ability to determine how many 
different code paths exist for the creation of an object and which path leaks 
the most. Some sample output:

```
LEAK UDisksLinuxDevice 1556 times (1 total diff. ways)
        g_type_create_instance:0x20b
        g_object_new_internal:0x2cb
        g_object_newv:0x1dd
        g_object_new:0x104
LEAK GUdevDevice 40 times (2 total diff. ways)
        g_type_create_instance:0x20b
        g_object_new_internal:0x2cb
        g_object_newv:0x1dd
        g_object_new:0x104
        _g_udev_device_new:0x1b
        g_udev_client_query_by_subsystem:0xce
LEAK GUdevDevice 1516 times (2 total diff. ways)
        g_type_create_instance:0x20b
        g_object_new_internal:0x2cb
        g_object_newv:0x1dd
        g_object_new:0x104
        _g_udev_device_new:0x1b
        monitor_event:0x29
        g_main_context_dispatch:0x15a
        g_main_context_iterate.isra.29:0x1f0
        g_main_loop_run:0xc2
LEAK GParamOverride 1 times (14 total diff. ways)
        g_type_create_instance:0x20b
        g_param_spec_internal:0x1e3
        g_param_spec_override:0x81
        g_object_class_override_property:0xde
        udisks_partition_table_override_properties:0x2d
        udisks_partition_table_skeleton_class_init:0x6e
        udisks_partition_table_skeleton_class_intern_init:0x48
        g_type_class_ref:0xa37
        g_type_class_ref:0x2e5
        g_object_newv:0x278
        g_object_new:0x104

...

Number of unique stack traces for a given type that leaks
23: GParamObject
18: GParamString
18: GParamBoolean
14: GParamOverride
7: GParamUInt64
7: GParamEnum
7: GParamBoxed
6: GParamInt
5: GParamUInt
5: GParamFlags
3: GParamVariant
2: GUdevDevice
2: GParamDouble
1: UDisksObjectSkeleton
1: UDisksLinuxSwapspace
...

```
We suspected that the biggest leaking object was GUdevDevice and based on this 
information we can see that there are two distinct paths where an object is 
created which leak, with one of them being the vast majority (lucky for me).
```
LEAK GUdevDevice 40 times (2 total diff. ways)
        g_type_create_instance:0x20b
        g_object_new_internal:0x2cb
        g_object_newv:0x1dd
        g_object_new:0x104
        _g_udev_device_new:0x1b
        g_udev_client_query_by_subsystem:0xce
LEAK GUdevDevice 1516 times (2 total diff. ways)
        g_type_create_instance:0x20b
        g_object_new_internal:0x2cb
        g_object_newv:0x1dd
        g_object_new:0x104
        _g_udev_device_new:0x1b
        monitor_event:0x29
        g_main_context_dispatch:0x15a
        g_main_context_iterate.isra.29:0x1f0
        g_main_loop_run:0xc2
```
It's interesting how many different code paths exist for some objects 
which are leaking, 23 different code paths, oh fun!  With this new information 
we can look at the output from the second specific systemtap output, find an 
object instance that was created which matches the same code path we selected 
from the summary and then proceed to examine all the places where the object 
reference count was incremented and decremented, an example 
(back traces omitted for brevity):

```
CREATED:GUdevDevice:11766:0x7f82cc005470
INC:GUdevDevice:11766:0x7f82cc005470
INC:GUdevDevice:11766:0x7f82cc005470
INC:GUdevDevice:11766:0x7f82cc005470
DEC:GUdevDevice:11766:0x7f82cc005470
INC:GUdevDevice:11766:0x7f82cc005470
DEC:GUdevDevice:11766:0x7f82cc005470
INC:GUdevDevice:11766:0x7f82cc005470
DEC:GUdevDevice:11766:0x7f82cc005470
INC:GUdevDevice:11766:0x7f82cc005470
DEC:GUdevDevice:11766:0x7f82cc005470
INC:GUdevDevice:11766:0x7f82cc005470
INC:GUdevDevice:11766:0x7f82cc005470
INC:GUdevDevice:11766:0x7f82cc005470
DEC:GUdevDevice:11766:0x7f82cc005470
DEC:GUdevDevice:11766:0x7f82cc005470
DEC:GUdevDevice:11766:0x7f82cc005470
```
At this point I wish I could share some magical way to identify the exact 
places where we are missing a reference count decrement and fix the code, 
but unfortunately you just end up looking at a lot of code and evaluating 
where the increment occurred, but a decrement is missing.  Sometimes it 
will be easy because the incremented lifespan is short, but in others the 
span is large and across thread boundaries.  This is where it would be good 
for developers to comment on the intended life cycle of the object in 
question when the code is being written.

### Final thoughts

Fixing gobject reference count leaks is indeed very difficult.  Even 
more so when you are working on code that others developed.  If you find 
yourself writing reference counting code that has non-trivial lifespans, 
please do others a favor and document the intended lifespan.  If you can't 
document it, it's a pretty good indication that you don't know or understand 
for sure and likely the code will be plagued with leaks and/or segmentation 
faults during it's lifetime.