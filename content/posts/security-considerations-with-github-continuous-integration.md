---
title: 'Security considerations with github continuous integration'
date: Tue, 08 Nov 2016 15:41:07 +0000
draft: false
tags: [Fedora]

aliases:
   - /index.php/2016/11/08/security-considerations-with-github-continuous-integration/
---

[Continuous integration](https://en.wikipedia.org/wiki/Continuous_integration) 
(CI) support in github is a very useful addition.  
Not only can you utilize existing services like [Travis CI,](https://travis-ci.org/) 
you can utilize the [github API](https://developer.github.com/) and roll your 
[own](https://developer.github.com/guides/building-a-ci-server/), which is 
exactly what we did for [libStorageMgmt](https://github.com/libstorage/libstoragemgmt).
LibStorageMgmt needs to run tests for hardware specific plugins, so we created 
our own tooling to hook up github and our hardware which is geographically 
located across the US.  However, shortly after getting all this in place and 
working it became pretty obvious that we provided a nice attack vector... 

All it takes is a malicious user to fork our repository, add some malicious 
code and submit a pull request and ***boom***!  We just executed whatever 
code they wanted on each of our CI test nodes.  The worst part is some of our 
tests require the node that is running the tests to run with root authority, so 
total box ownership is assured.  The test nodes are stripped down boxes with 
nothing of value on them data wise, but just providing hardware and network 
resources in which to execute arbitrary code is indeed valuable. Our solution 
was to limit automated CI testing to only those we trust with an approved access list.  
I don't envy Travis CI and all the security issues that have to consider.  
They are basically providing a service which allows virtually anyone to 
execute arbitrary code on their boxes.

None of this is terribly 
[new](https://www.trustwave.com/Resources/SpiderLabs-Blog/Securing-Continuous-Integration-Services/), 
just potentially new to those writing their own CI service.
