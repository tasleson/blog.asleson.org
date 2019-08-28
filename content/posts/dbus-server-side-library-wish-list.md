---
title: 'DBUS Server side library wish list'
date: Thu, 18 Apr 2019 17:35:03 +0000
draft: false
tags: [Fedora]

aliases:
   - /index.php/2019/04/18/dbus-server-side-library-wish-list/
---

**Ideally it would be great if a DBUS server side library provided**

1.  Fully implements the functionality needed for common interfaces (Properties, ObjectManager, Introspectable) in a sane and easy way and doesn't require you to manually supply the interface XML.
2.  Allows you to register a number of objects simultaneously, so if you have circular references etc.  This avoids race conditions on client.
3.  Ability to auto generate signals when object state changes and represent the state of the object separately for each interface on each object.
4.  Freeze/Thaw on objects or the whole model to minimize number of signals, but without requiring the user to manually code stuff up for signals.
5.  Configurable process/thread model and thread safe.
6.  Incrementally and dynamically add/remove an interface to an object without destroying the object and re-creating and while incrementally adding/removing the state as well.
7.  Handle object path construction issues, like having an object that needs an object path to another object that doesn't yet exist.  This is alleviated of you have #8.
8.  Ability to create one or more objects without actually registering them with the service, so you can handle issues like #7 easier, especially when coupled with #2, directly needed for #2.  Thus you create 1 or more objects and register them together.
9.  Doesn't require the use of code generating tools.
10.  Allow you to have multiple paths/name spaces which refer to the same objects. This would be useful for services that implement functionality in other services without requiring the clients to change.
11.  Allows you to remove a dbus object from the model while you are processing a method on it.  eg. client is calling a remove method on the object it wants to remove.
