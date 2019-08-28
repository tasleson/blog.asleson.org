---
title: 'Buffer bloat mitigation with OpenBSD pf'
date: Tue, 15 Oct 2013 20:00:01 +0000
draft: false
tags: [OpenBSD]
aliases:
   - /index.php/2013/10/15/buffer-bloat-mitigation-with-openbsd-pf/
---

For an introduction to buffer bloat read more here 
[http://en.wikipedia.org/wiki/Bufferbloat .](http://en.wikipedia.org/wiki/Bufferbloat "wikipedia") My home network utilizes OpenBSD and the built in packet filter (pf). I use cable for broadband internet and found that if I tried to upload a large file my internet connecting became very unable with high amounts of latency. After utilizing tools such as [http://netalyzr.icsi.berkeley.edu/blog/](http://netalyzr.icsi.berkeley.edu/blog/ "network analyzer") 
it became obvious that I was suffering from buffer bloat. After doing some 
searching I came across using altq support in pf to try some configuration 
changes to reduce the buffer bloat in my configuration. I added a queue for 
my external network card with the following:

```
altq on $ext_if bandwidth 1Mb hfsc queue { bb }
queue bb bandwidth 100% qlimit 9 hfsc ( default )
```
and the corresponding rule to tag outgoing traffic for that interface to this queue
```
pass out on $ext_if keep state queue( bb )
```
My home connection is advertised as 5Mb down, 1Mb up. In testing I get about 
1.1Mb up so I setup my outgoing queue to limit outgoing to 1Mb. A typical 
setting would be 97% of maximum. One of the most important values in the 
queue setup is the number of buffers. Mine is currently set at 9. This is 
how I determined what the value should be.

Maximum upstream bandwidth in 
packets is upstream bandwidth in bytes / size of a packet. In my case 
1000000/8 = bandwidth in bytes / 1460 (size of packet) which yields 
85 packets a second. So if I set my buffer size to 85 I should have about 
1 second of latency. In my case I like my latency low so I divided by 10 to 
try and get a 100ms latency under full upstream use which is 8.5, which I 
rounded up to 9. 

So how does it work? 

Average pings to slashdot.org in 
milliseconds
```
Idle connection:                 23.8   ( 0% packet loss)
Maxed upstream use:              95.1   ( 0% packet loss)
Maxed upstream without altq:   2083.4   (10% packet loss)

```
Quite the improvement! Thanks to the information on 
[https://calomel.org/pf\_hfsc.html](https://calomel.org/pf_hfsc.html "pf shaping") 
for helpful tips.