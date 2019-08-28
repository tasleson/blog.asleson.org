---
title: "Python, the perpetual time suck"
date: "2019-08-28T12:49:13Z"
draft: false
tags: [Fedora]
---

I used to like Python.  Like others I enjoy the productivity it offers and the
vast and plentiful libraries that exist. However, over time that fondness has
turned to loathing.

The thing that should have been apparent to me long ago is that the Python
folks don't appear to care about end users.  They seem to have lost touch with
the fact that Python is **very** popular!  Each and every time they make core
language behavior changes, API changes, and deprecate things, a lot of code has
to accommodate.  It's a non-trivial amount of work to keep Python code
working.  Especially so if you're trying to support code that has to run across
multiple versions spanning many years.  The test matrix just keeps on getting
bigger.  The code hacks to accommodate versions becoming more and more
intrusive.

The python 2 to python 3 debacle should have convinced everyone that the Python
project cares more about the language and how they can make it better than the
effect it has on the existing code written in it.  One would have assumed that
once the whole 2 -> 3 conversion was over, that things would have settled down.
That the things that needed to be fixed would be done, but the incompatible
changes just keep coming.  It's like the Python developers got a taste for
change, perfection, they just can't help themselves regardless of cost to the
development community.  I understand, it's virtually impossible to get things
exactly right the first time, but you have to let go and leave it alone.  Once
it's out there, it needs to stay as is unless it's a security hole.  It's
totally fine to add features, improve performance etc., but horrible and
inexcusable to break existing code.

I've had enough.  I'm not writing anything new in Python, unless it's
absolutely necessary.  The madness needs to stop.  Whatever productivity gains
you get in the short term are lost with the vast amount of code maintenance to
keep the thing working over time.

