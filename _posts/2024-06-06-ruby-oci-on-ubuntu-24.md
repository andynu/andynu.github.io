---
layout: post
title:  "ruby-oci8 on ubuntu 24.04 with the new libaio1t64"
date:   2024-06-06
categories: ruby
---

While setting up a new Ubuntu 24.04 LTS machine and trying to install ruby-oci8 v2.2.12 I encountered an issue where it could not find libaio.so.1.

The ubuntu package libaio1 has been replaced with libaio1t64, and ruby-oci8 is not looking for the newly named version of the library `libaio.so.1t64`

You can see this with dpkg.

```
% dpkg -L libaio1t64
/.
/usr
/usr/lib
/usr/lib/x86_64-linux-gnu
/usr/lib/x86_64-linux-gnu/libaio.so.1t64.0.2
/usr/share
/usr/share/doc
/usr/share/doc/libaio1t64
/usr/share/doc/libaio1t64/changelog.Debian.gz
/usr/share/doc/libaio1t64/copyright
/usr/lib/x86_64-linux-gnu/libaio.so.1t64
```


My solution for now was to simlink to new name to the old one.
```
sudo ln -s /usr/lib/x86_64-linux-gnu/libaio.so.1t64 /usr/lib/x86_64-linux-gnu/libaio.so.1
```


