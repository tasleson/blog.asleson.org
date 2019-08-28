---
title: 'libtool library versioning ( -version-info ‘current[:revision[:age]]'' )'
date: Tue, 08 Jul 2014 17:39:46 +0000
draft: false
tags: [Fedora]

aliases:
   - /index.php/2014/07/08/libtool-library-versioning-version-info-currentrevisionage/
---

The documentation on the gnu libtool [manual](http://www.gnu.org/software/libtool/manual/html_node/Updating-version-info.html#Updating-version-info "manual") states:

> The following explanation may help to understand the above rules a bit better: consider that there are three possible kinds of reactions from users of your library to changes in a shared library:
> 
> 1.  Programs using the previous version may use the new version as drop-in replacement, and programs using the new version can also work with the previous one. In other words, no recompiling nor relinking is needed. In this case, bump revision only, don't touch current nor age.
> 2.  Programs using the previous version may use the new version as drop-in replacement, but programs using the new version may use APIs not present in the previous one. In other words, a program linking against the new version may fail with “unresolved symbols” if linking against the old version at runtime: set revision to 0, bump current and age.
> 3.  Programs may need to be changed, recompiled, relinked in order to use the new version. Bump current, set revision and age to 0.

This was confusing to me until I played around with the -version-info value and 
looked at the library output on my linux development system. 

```
-version-info 0:0:0 => libfoo.so.0.0.0 
-version-info 1:0:1 => libfoo.so.0.1.0
-version-info 1:0:0 => libfoo.so.1.0.0 
-version-info 2:0:1 => libfoo.so.1.1.0 
-version-info 2.1.1 => libfoo.so.1.1.1 
-version-info 3.0.2 => libfoo.so.1.2.0 
-version-info 4:0:0 => libfoo.so.4.0.0
``` 

The current:revision:age is an **abstraction** which allows libtool to do the
correct thing for the platform you are building and deploying on. This wasn't
immediately evident when reading the documentation.