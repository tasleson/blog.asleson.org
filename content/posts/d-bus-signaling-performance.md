---
title: 'D-bus signaling performance'
date: Tue, 01 Sep 2015 20:01:24 +0000
draft: false
tags: [Fedora]
aliases:
   - /index.php/2015/09/01/d-bus-signaling-performance/
---

While working on [lvm-dubstep](https://github.com/tasleson/lvm-dubstep) 
the question was posed if D-bus could handle the number of changes that could 
happen in a short period of time, especially PropertiesChanged signals when a 
large number of logical volumes or physical volumes were present on the system 
(eg. 120K PVs and 10K+ LVs).  To test this idea I put together a simple server 
and client which simply tries to send an arbitrary number of signals as fast as 
it possibly can.  The number settled upon was 10K because during early testing 
I was running into time-out exceptions when trying to send more in a row.  
Initial testing was done using the
[dbus-python](http://dbus.freedesktop.org/doc/dbus-python/) 
library and even though numbers seemed sufficient, people asked about sd-bus 
and sd-bus utilizing kdbus, so the experiment was expanded to include these as 
well.  Source code for the testing is available 
[here](https://github.com/tasleson/dbus-signals).   

**Test configuration**

*   One client listening for signals.
*   Python tests were run on a Fedora 22 VM.
*   Sd-bus was run on a similarly configured VM running Fedora 23 alpha 
    utilizing systemd 222-2.  Kdbus was built as a kernel module from the latest 
    systemd/kdbus repo.  F23 kernel version: 4.2.0-0.rc8.git0.1.fc23.x86_64.
*   I tried running all the tests on the F23 VM, but the python tests were 
    failing horribly with journal entries: 
    `Sep 01 10:53:14 sdbus systemd-bus-proxyd[663]: **Dropped messages due to queue overflow of local peer (pid: 2615 uid: 0)`

**The results:**

*   Python utilizing dbus-python library was able to send and receive 10K 
    signals in a row, varying payload size from 32-128K without any issues.
    As I mentioned before if I try to send more than that in a row, especially
    at larger payload sizes I do get time-outs on the send.
*   sd-bus without using kdbus I was only able to send up to 512 byte payloads
    before the server would error out with: Failed to emit signal: 
    **No buffer space available.**
*   sd-bus with kdbus the test completes, seemingly without error, but the
    number of signals received is not as expected and appears to vary with payload size.

Messages/second is the total number of messages divided by the total time to
receive them.

{{< figure src="/pictures/l_dbus_msg_sec.png" title="" >}}

MiB/Second is the messages per second multiplied by the payload size.

{{< figure src="/pictures/l_dbus_mib_sec.png" title="" >}}

Average time delta is the time difference from when the signal was placed on 
the bus until is was read by the signal handler.  This shows quite a bit of 
buffering for the python test case at larger payload sizes. 

{{< figure src="/pictures/l_dbus_time_delta.png" title="" >}}

Percentage of signals that were received by the signal handler.  As you can 
see once the payload > 2048, kdbus appears to silently discard signals away 
as nothing appeared in kernel output and the return codes were all good in 
user space. 

{{< figure src="/pictures/l_dbus_percent_received.png" title="" >}}

**Conclusions**

*   The C implementation using sd-bus without kdbus performs slightly poorer
    than the dbus-python which surprised me.
*   kdbus is by far the best performing with payloads < 2048 bytes
*   Signals are by definition sent without a response.  Having a reliable way
    for the sender to know that a signal was not sent seems pretty important,
    so kdbus seems to have a bug in that no error was being returned to the
    transmitter or that there is a bug in my test code.
*   Code likely has bugs and/or does something sub optimal, please do a pull
    request and I will incorporate the changes and re-run the tests.