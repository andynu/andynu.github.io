---
layout: post
title:  "ruby-oci8 on ubuntu 24.04 with the new libaio1t64"
date:   2024-06-06
categories: ruby
---

While setting up a new Ubuntu 24.04 LTS machine and trying to install ruby-oci8 v2.2.12 I encountered an issue [1] where it could not find libaio.so.1 [2].

The ubuntu package libaio1 has been replaced with libaio1t64, and ruby-oci8 is not looking for the newly named version of the library `libaio.so.1t64` [3]


**My solution: simlink to new name to the old one.**
```
sudo ln -s /usr/lib/x86_64-linux-gnu/libaio.so.1t64 /usr/lib/x86_64-linux-gnu/libaio.so.1
```

---
Footnotes:

[1] outer stack trace when installing the ruby-oci8 gem without the symlink.
```
% gem install --clear-sources ruby-oci8                           ± rails-7-1-stable
Building native extensions. This could take a while...
ERROR:  Error installing ruby-oci8:
	ERROR: Failed to build gem native extension.

    current directory: /home/andy/.rbenv/versions/3.3.2/lib/ruby/gems/3.3.0/gems/ruby-oci8-2.2.12/ext/oci8
/home/andy/.rbenv/versions/3.3.2/bin/ruby extconf.rb
attempting to locate oracle-instantclient...
checking load library path... 
  LD_LIBRARY_PATH... 
    checking /opt/instantclient... yes
  /opt/instantclient/libclntsh.so.21.1 looks like an instant client.
checking for cc... ok
checking for gcc... yes
checking for LP64... yes
checking for sys/types.h... yes
checking for ruby header... ok
checking for OCIEnvCreate() in oci.h... yes
checking for OCI_MAJOR_VERSION in oci.h... *** extconf.rb failed ***
Could not create Makefile due to some reason, probably lack of necessary
libraries and/or headers.  Check the mkmf.log file for more details.  You may
need configuration options.

Provided configuration options:
	--with-opt-dir
	--without-opt-dir
	--with-opt-include=${opt-dir}/include
	--without-opt-include
	--with-opt-lib=${opt-dir}/lib
	--without-opt-lib
	--with-make-prog
	--without-make-prog
	--srcdir=.
	--curdir
	--ruby=/home/andy/.rbenv/versions/3.3.2/bin/$(RUBY_BASE_NAME)
	--with-instant-client
	--without-instant-client
	--with-instant-client-dir
	--without-instant-client-dir
	--with-instant-client-include=${instant-client-dir}/include
	--without-instant-client-include
	--with-instant-client-lib=${instant-client-dir}/lib
	--without-instant-client-lib
	--with-sys-dir
	--without-sys-dir
	--with-sys-include=${sys-dir}/include
	--without-sys-include
	--with-sys-lib=${sys-dir}/lib
	--without-sys-lib
<internal:kernel>:307:in `Integer': can't convert nil into Integer (TypeError)
	from /home/andy/.rbenv/versions/3.3.2/lib/ruby/3.3.0/mkmf.rb:754:in `block in try_constant'
	from /home/andy/.rbenv/versions/3.3.2/lib/ruby/3.3.0/mkmf.rb:428:in `popen'
	from /home/andy/.rbenv/versions/3.3.2/lib/ruby/3.3.0/mkmf.rb:428:in `block in xpopen'
	from /home/andy/.rbenv/versions/3.3.2/lib/ruby/3.3.0/mkmf.rb:314:in `open'
	from /home/andy/.rbenv/versions/3.3.2/lib/ruby/3.3.0/mkmf.rb:421:in `xpopen'
	from /home/andy/.rbenv/versions/3.3.2/lib/ruby/3.3.0/mkmf.rb:753:in `try_constant'
	from extconf.rb:34:in `block in <main>'
	from /home/andy/.rbenv/versions/3.3.2/lib/ruby/3.3.0/mkmf.rb:983:in `block in checking_for'
	from /home/andy/.rbenv/versions/3.3.2/lib/ruby/3.3.0/mkmf.rb:344:in `block (2 levels) in postpone'
	from /home/andy/.rbenv/versions/3.3.2/lib/ruby/3.3.0/mkmf.rb:314:in `open'
	from /home/andy/.rbenv/versions/3.3.2/lib/ruby/3.3.0/mkmf.rb:344:in `block in postpone'
	from /home/andy/.rbenv/versions/3.3.2/lib/ruby/3.3.0/mkmf.rb:314:in `open'
	from /home/andy/.rbenv/versions/3.3.2/lib/ruby/3.3.0/mkmf.rb:340:in `postpone'
	from /home/andy/.rbenv/versions/3.3.2/lib/ruby/3.3.0/mkmf.rb:982:in `checking_for'
	from extconf.rb:33:in `<main>'

To see why this extension failed to compile, please check the mkmf.log which can be found here:

  /home/andy/.rbenv/versions/3.3.2/lib/ruby/gems/3.3.0/extensions/x86_64-linux/3.3.0/ruby-oci8-2.2.12/mkmf.log

extconf failed, exit code 1

Gem files will remain installed in /home/andy/.rbenv/versions/3.3.2/lib/ruby/gems/3.3.0/gems/ruby-oci8-2.2.12 for inspection.
Results logged to /home/andy/.rbenv/versions/3.3.2/lib/ruby/gems/3.3.0/extensions/x86_64-linux/3.3.0/ruby-oci8-2.2.12/gem_make.out
```

[2] inner error in mkmf.log which reveals the issue is with finding libaio1
(I've omitted all but the last line, which is the only one that mattered)
```
LD_LIBRARY_PATH=.:/home/andy/.rbenv/versions/3.3.2/lib:/opt/instantclient ./conftest |
./conftest: error while loading shared libraries: libaio.so.1: cannot open shared object file: No such file or directory
```

[3] contents of libaio1t64, nothing that exactly matches 'libaio.so.1', but that '.1t64' іs close!

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
