What is XNU?
===========

XNU kernel is part of the Darwin operating system for use in OS X and iOS operating systems. XNU is an acronym for XNU is Not Unix.
XNU is a hybrid kernel combining the Mach kernel developed at Carnegie Mellon University with components from FreeBSD and C++ API for writing drivers called IOKit.
XNU runs on I386, X86_64 for both single processor and multi-processor configurations.

XNU Source Tree
===============

  * `config` - configurations for exported apis for supported architecture and platform
  * `SETUP` - Basic set of tools used for configuring the kernel, versioning and kextsymbol management.
  * `EXTERNAL_HEADERS` - Headers sourced from other projects to avoid dependency cycles when building. These headers should be regularly synced when source is updated.
  * `libkern` - C++ IOKit library code for handling of drivers and kexts.
  * `libsa` -  kernel bootstrap code for startup
  * `libsyscall` - syscall library interface for userspace programs
  * `libkdd` - source for user library for parsing kernel data like kernel chunked data.
  * `makedefs` - top level rules and defines for kernel build.
  * `osfmk` - Mach kernel based subsystems
  * `pexpert` - Platform specific code like interrupt handling, atomics etc.
  * `security` - Mandatory Access Check policy interfaces and related implementation.
  * `bsd` - BSD subsystems code
  * `tools` - A set of utilities for testing, debugging and profiling kernel.

Developing XNU on VMware Fusion on Mac OS
==================================

The xnu kernel can be built and run on VMware Fusion on Mac OS.  In
order to develop xnu in this way you will first need to set up a Mac
OS Virtual Machine.  You will need at least 40G of disk space free for
a Mac OS VM.  These instructions are current as of VMware Fusion
8.5.3.

	1. Select File->New
	2. Choose "Install OS X from the recovery partition"
	3. Before installing use the utilities to start a terminal
	4. Turn OFF system integrity protection
	4a. $ csrutil disable
	4b. $ csrutil status
	      System Integrity Protection status: disabled
	5. Install Mac OS
	6. Install Xcode into the newly created VM.

Follow the rest of the instructions below.

How to build XNU
================

Install the firehose library
----------------------------

From Mac OS 10.12 onwards you must install the `firehose` library to
before you attempt to build an xnu kernel.

Clone the following [repo](https://github.com/Proteas/install_firehose_lib)

and then run the installer script:

`./install-firehose.sh`

Simple Instructions from Shantonu
---------------------------------

The following insturctions are included from this blog post
on [building 10.11](http://shantonu.blogspot.com) and along with the
above instruction on installing firehose will give you a proper build
of xnu.  The instructions have been changed to point to repositories
that are present in OpenDTrace including macos-dtrace and xnu. The
full build instructions are included in the sections following this
one.

Install OS X and Xcode from the Mac App Store, make sure the Xcode
license has been agreed-to with `sudo xcodebuild -license` Download
the source for the dtrace and AvailabilityVersions projects, which are
required dependencies, as well as xnu itself

1. Build and install CTF tools from dtrace
```
$ cd macos-dtrace
$ mkdir -p obj sym dst
$ xcodebuild install -target ctfconvert -target ctfdump -target ctfmerge ARCHS="x86_64" SRCROOT=$PWD OBJROOT=$PWD/obj SYMROOT=$PWD/sym DSTROOT=$PWD/dst
...
$ sudo ditto $PWD/dst/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain
Password:
$ cd ..
```

2. Install AvailabilityVersions
```
$ tar zxf AvailabilityVersions-20.tar.gz 
$ cd AvailabilityVersions-20
$ mkdir -p dst
$ make install SRCROOT=$PWD DSTROOT=$PWD/dst
$ sudo ditto $PWD/dst/usr/local `xcrun -sdk macosx -show-sdk-path`/usr/local
$ cd ..
```

3. Build xnu
```
$ tar zxf xnu-3247.1.106.tar.gz
$ cd xnu-3247.1.106
$ make SDKROOT=macosx ARCH_CONFIGS=X86_64 KERNEL_CONFIGS=RELEASE
See xnu's top-level README for additional build variables that can be passed on the command-line, such as BUILD_LTO=0 or KERNEL_CONFIGS=DEVELOPMENT .
```

You can now install xnu or continue with these insructions of you are
addding system calls

NOTE: If you are attempting to add system calls, you may also need to
build Libsyscall.

4. Download the Libsystem source
```
$ curl -O https://opensource.apple.com/tarballs/Libsystem/Libsystem-1225.1.1.tar.gz
```
5. Install Libsystem headers
```
$ tar zxf Libsystem-1225.1.1.tar.gz
$ cd Libsystem-1225.1.1
$ xcodebuild installhdrs -sdk macosx ARCHS='x86_64 i386' SRCROOT=$PWD OBJROOT=$PWD/obj SYMROOT=$PWD/sym DSTROOT=$PWD/dst
$ sudo ditto $PWD/dst `xcrun -sdk macosx -show-sdk-path`
$ cd ..
```

6. Install xnu and Libsyscall headers
```
$ cd xnu-3247.1.106
$ mkdir -p BUILD.hdrs/obj BUILD.hdrs/sym BUILD.hdrs/dst
$ make installhdrs SDKROOT=macosx ARCH_CONFIGS=X86_64 SRCROOT=$PWD OBJROOT=$PWD/BUILD.hdrs/obj SYMROOT=$PWD/BUILD.hdrs/sym DSTROOT=$PWD/BUILD.hdrs/dst
$ sudo xcodebuild installhdrs -project libsyscall/Libsyscall.xcodeproj -sdk macosx ARCHS='x86_64 i386' SRCROOT=$PWD/libsyscall OBJROOT=$PWD/BUILD.hdrs/obj SYMROOT=$PWD/BUILD.hdrs/sym DSTROOT=$PWD/BUILD.hdrs/dst
$ sudo ditto BUILD.hdrs/dst `xcrun -sdk macosx -show-sdk-path`
```

7. Build Libsyscall
```
$ mkdir -p BUILD.libsyscall/obj BUILD.libsyscall/sym BUILD.libsyscall/dst
$ sudo xcodebuild install -project libsyscall/Libsyscall.xcodeproj -sdk macosx ARCHS='x86_64 i386' SRCROOT=$PWD/libsyscall OBJROOT=$PWD/BUILD.libsyscall/obj SYMROOT=$PWD/BUILD.libsyscall/sym DSTROOT=$PWD/BUILD.libsyscall/dst
```

To install custom OS components, System Integrity Protection must be
disabled.  If you are installing in a VM you should have followed the
directions above, if you are using actual hardware you will need to
boot into the recovery partition and follow the same directions as in
step 4 of the VM discussion.

8. To install the resulting new binaries, execute:

xnu:
```
$ sudo cp BUILD/obj/RELEASE_X86_64/kernel /System/Library/Kernels/
$ sudo kextcache -invalidate /
/ locked; waiting for lock.
Lock acquired; proceeding.
...
$ sudo reboot
```

9. If you are adding system calls the install libsyscall as well.

Libsyscall:
```
$ sudo cp BUILD.libsyscall/dst/usr/lib/system/libsystem_kernel.dylib /usr/lib/system/
$ sudo update_dyld_shared_cache
...
$ sudo reboot
```

Building `DEVELOPMENT` kernel
-----------------------------

The xnu make system can build kernel based on `KERNEL_CONFIGS` & `ARCH_CONFIGS` variables as arguments.
Here is the syntax:

    make SDKROOT=<sdkroot> ARCH_CONFIGS=<arch> KERNEL_CONFIGS=<variant>

Where:

  * \<sdkroot>: path to MacOS SDK on disk. (defaults to `/`)
  * \<variant>: can be `debug`, `development`, `release`, `profile` and configures compilation flags and asserts throughout kernel code.
  * \<arch>   : can be valid arch to build for. (E.g. `i386` or `X86_64`)

To build a kernel for the same architecture as running OS, just type

    $ make
    $ make SDKROOT=macosx.internal

Additionally, there is support for configuring architectures through `ARCH_CONFIGS` and kernel configurations with `KERNEL_CONFIGS`.

    $ make SDKROOT=macosx.internal ARCH_CONFIGS=X86_64 KERNEL_CONFIGS=DEVELOPMENT
    $ make SDKROOT=macosx.internal ARCH_CONFIGS=X86_64 KERNEL_CONFIGS="RELEASE DEVELOPMENT DEBUG"


Note:
  * By default, architecture is set to the build machine architecture, and the default kernel
    config is set to build for DEVELOPMENT.


This will also create a bootable image, kernel.[config],  and a kernel binary
with symbols, kernel.[config].unstripped.


  * To build with RELEASE kernel configuration

        make KERNEL_CONFIGS=RELEASE SDKROOT=/path/to/SDK


Building FAT kernel binary
--------------------------

Define architectures in your environment or when running a make command.

    $ make ARCH_CONFIGS="I386 X86_64" exporthdrs all

Other makefile options
----------------------

 * $ make MAKEJOBS=-j8    # this will use 8 processes during the build. The default is 2x the number of active CPUS.
 * $ make -j8             # the standard command-line option is also accepted
 * $ make -w              # trace recursive make invocations. Useful in combination with VERBOSE=YES
 * $ make BUILD_LTO=0      # build without LLVM Link Time Optimization
 * $ make REMOTEBUILD=user@remotehost # perform build on remote host
 * $ make BUILD_JSON_COMPILATION_DATABASE=1 # Build Clang JSON Compilation Database



Debug information formats
=========================

By default, a DWARF debug information repository is created during the install phase; this is a "bundle" named kernel.development.\<variant>.dSYM
To select the older STABS debug information format (where debug information is embedded in the kernel.development.unstripped image), set the BUILD_STABS environment variable.

    $ export BUILD_STABS=1
    $ make


Building KernelCaches
=====================

To test the xnu kernel, you need to build a kernelcache that links the kexts and
kernel together into a single bootable image.
To build a kernelcache you can use the following mechanisms:

  * Using automatic kernelcache generation with `kextd`.
    The kextd daemon keeps watching for changing in `/System/Library/Extensions` directory. 
    So you can setup new kernel as

        $ cp BUILD/obj/DEVELOPMENT/X86_64/kernel.development /System/Library/Kernels/
        $ touch /System/Library/Extensions
        $ ps -e | grep kextd

  * Manually invoking `kextcache` to build new kernelcache.

        $ kextcache -q -z -a x86_64 -l -n -c /var/tmp/kernelcache.test -K /var/tmp/kernel.test /System/Library/Extensions



Running KernelCache on Target machine
=====================================

The development kernel and iBoot supports configuring boot arguments so that we can safely boot into test kernel and, if things go wrong, safely fall back to previously used kernelcache.
Following are the steps to get such a setup:

  1. Create kernel cache using the kextcache command as `/kernelcache.test`
  2. Copy exiting boot configurations to alternate file

         $ cp /Library/Preferences/SystemConfiguration/com.apple.Boot.plist /next_boot.plist

  3. Update the kernelcache and boot-args for your setup

         $ plutil -insert "Kernel Cache" -string "kernelcache.test" /next_boot.plist
         $ plutil -replace "Kernel Flags" -string "debug=0x144 -v kernelsuffix=test " /next_boot.plist

  4. Copy the new config to `/Library/Preferences/SystemConfiguration/`

         $ cp /next_boot.plist /Library/Preferences/SystemConfiguration/boot.plist

  5. Bless the volume with new configs.

         $ sudo -n bless  --mount / --setBoot --nextonly --options "config=boot"

     The `--nextonly` flag specifies that use the `boot.plist` configs only for one boot.
     So if the kernel panic's you can easily power reboot and recover back to original kernel.




Creating tags and cscope
========================

Set up your build environment and from the top directory, run:

    $ make tags     # this will build ctags and etags on a case-sensitive volume, only ctags on case-insensitive
    $ make TAGS     # this will build etags
    $ make cscope   # this will build cscope database


Coding styles (Reindenting files)
=================================

Source files can be reindented using clang-format setup in .clang-format.
XNU follows a variant of WebKit style for source code formatting.
Please refer to format styles at [WebKit website](http://www.webkit.org/coding/coding-style.html). 
Further options about style options is available at [clang docs](http://clang.llvm.org/docs/ClangFormatStyleOptions.html)

  Note: clang-format binary may not be part of base installation. It can be compiled from llvm clang sources and is reachable in $PATH.

  From the top directory, run:

   $ make reindent      # reindent all source files using clang format.



How to install a new header file from XNU
=========================================

To install IOKit headers, see additional comments in [iokit/IOKit/Makefile]().

XNU installs header files at the following locations -

    a. $(DSTROOT)/System/Library/Frameworks/Kernel.framework/Headers
    b. $(DSTROOT)/System/Library/Frameworks/Kernel.framework/PrivateHeaders
    c. $(DSTROOT)/usr/include/
    d. $(DSTROOT)/System/Library/Frameworks/System.framework/PrivateHeaders

`Kernel.framework` is used by kernel extensions.\
The `System.framework` and `/usr/include` are used by user level applications. \
The header files in framework's `PrivateHeaders` are only available for ** Apple Internal Development **.

The directory containing the header file should have a Makefile that
creates the list of files that should be installed at different locations.
If you are adding first header file in a directory, you will need to
create Makefile similar to xnu/bsd/sys/Makefile.

Add your header file to the correct file list depending on where you want
to install it. The default locations where the header files are installed
from each file list are -

    a. `DATAFILES` : To make header file available in user level -
       `$(DSTROOT)/usr/include`

    b. `PRIVATE_DATAFILES` : To make header file available to Apple internal in
       user level -
       `$(DSTROOT)/System/Library/Frameworks/System.framework/PrivateHeaders`

    c. `KERNELFILES` : To make header file available in kernel level -
       `$(DSTROOT)/System/Library/Frameworks/Kernel.framework/Headers`
       `$(DSTROOT)/System/Library/Frameworks/Kernel.framework/PrivateHeaders`

    d. `PRIVATE_KERNELFILES` : To make header file available to Apple internal
       for kernel extensions -
       `$(DSTROOT)/System/Library/Frameworks/Kernel.framework/PrivateHeaders`

The Makefile combines the file lists mentioned above into different
install lists which are used by build system to install the header files.

If the install list that you are interested does not exist, create it
by adding the appropriate file lists.  The default install lists, its
member file lists and their default location are described below -

    a. `INSTALL_MI_LIST` : Installs header file to a location that is available to everyone in user level.
        Locations -
           $(DSTROOT)/usr/include
       Definition -
           INSTALL_MI_LIST = ${DATAFILES}

    b.  `INSTALL_MI_LCL_LIST` : Installs header file to a location that is available
       for Apple internal in user level.
       Locations -
           $(DSTROOT)/System/Library/Frameworks/System.framework/PrivateHeaders
       Definition -
           INSTALL_MI_LCL_LIST = ${PRIVATE_DATAFILES}

    c. `INSTALL_KF_MI_LIST` : Installs header file to location that is available
       to everyone for kernel extensions.
       Locations -
            $(DSTROOT)/System/Library/Frameworks/Kernel.framework/Headers
       Definition -
            INSTALL_KF_MI_LIST = ${KERNELFILES}

    d. `INSTALL_KF_MI_LCL_LIST` : Installs header file to location that is
       available for Apple internal for kernel extensions.
       Locations -
            $(DSTROOT)/System/Library/Frameworks/Kernel.framework/PrivateHeaders
       Definition -
            INSTALL_KF_MI_LCL_LIST = ${KERNELFILES} ${PRIVATE_KERNELFILES}

If you want to install the header file in a sub-directory of the paths
described in (1), specify the directory name using two variables
`INSTALL_MI_DIR` and `EXPORT_MI_DIR` as follows -

    INSTALL_MI_DIR = dirname
    EXPORT_MI_DIR = dirname

A single header file can exist at different locations using the steps
mentioned above.  However it might not be desirable to make all the code
in the header file available at all the locations.  For example, you
want to export a function only to kernel level but not user level.

 You can use C language's pre-processor directive (#ifdef, #endif, #ifndef)
 to control the text generated before a header file is installed.  The kernel
 only includes the code if the conditional macro is TRUE and strips out
 code for FALSE conditions from the header file.

 Some pre-defined macros and their descriptions are -

    a. `PRIVATE` : If true, code is available to all of the xnu kernel and is
       not available in kernel extensions and user level header files.  The
       header files installed in all the paths described above in (1) will not
       have code enclosed within this macro.

    b. `KERNEL_PRIVATE` : If true, code is available to all of the xnu kernel and Apple
        internal kernel extensions.

    c. `BSD_KERNEL_PRIVATE` : If true, code is available to the xnu/bsd part of
       the kernel and is not available to rest of the kernel, kernel extensions
       and user level header files.  The header files installed in all the
       paths described above in (1) will not have code enclosed within this macro.

    d. `KERNEL` :  If true, code is available only in kernel and kernel
       extensions and is not available in user level header files.  Only the
       header files installed in following paths will have the code -

            $(DSTROOT)/System/Library/Frameworks/Kernel.framework/Headers
            $(DSTROOT)/System/Library/Frameworks/Kernel.framework/PrivateHeaders

       you should check [Testing the kernel][] for details.


How to add a new syscall
========================




Testing the kernel
==================

XNU kernel has multiple mechanisms for testing.

  * Assertions - The DEVELOPMENT and DEBUG kernel configs are compiled with assertions enabled. This allows developers to easily
    test invariants and conditions.

  * XNU Power On Self Tests (`XNUPOST`): The XNUPOST config allows for building the kernel with basic set of test functions
    that are run before first user space process is launched. Since XNU is hybrid between MACH and BSD, we have two locations where
    tests can be added.

        xnu/osfmk/tests/     # For testing mach based kernel structures and apis.
        bsd/tests/           # For testing BSD interfaces.
    Please follow the documentation at [osfmk/tests/README.md](osfmk/tests/README.md)

  * User level tests: The `tools/tests/` directory holds all the tests that verify syscalls and other features of the xnu kernel.
    The make target `xnu_tests` can be used to build all the tests supported.

        $ make RC_ProjectName=xnu_tests SDKROOT=/path/to/SDK

    These tests are individual programs that can be run from Terminal and report tests status by means of std posix exit codes (0 -> success) and/or stdout.
    Please read detailed documentation in [tools/tests/unit_tests/README.md](tools/tests/unit_tests/README.md)


Kernel data descriptors
=======================

XNU uses different data formats for passing data in its api. The most standard way is using syscall arguments. But for complex data
it often relies of sending memory saved by C structs. This packaged data transport mechanism is fragile and leads to broken interfaces
between user space programs and kernel apis. `libkdd` directory holds user space library that can parse custom data provided by the
same version of kernel. The kernel chunked data format is described in detail at [libkdd/README.md](libkdd/README.md).


Debugging the kernel
====================

The xnu kernel supports debugging with a remote kernel debugging protocol (kdp). Please refer documentation at [technical note] [TN2063]
By default the kernel is setup to reboot on a panic. To debug a live kernel, the kdp server is setup to listen for UDP connections
over ethernet. For machines without ethernet port, this behavior can be altered with use of kernel boot-args. Following are some
common options.

  * `debug=0x144` - setups debug variables to start kdp debugserver on panic
  * `-v` - print kernel logs on screen. By default XNU only shows grey screen with boot art.
  * `kdp_match_name=en1` - Override default port selection for kdp. Supported for ethernet, thunderbolt and serial debugging.

To debug a panic'ed kernel, use llvm debugger (lldb) along with unstripped symbol rich kernel binary.

    sh$ lldb kernel.development.unstripped
    
And then you can connect to panic'ed machine with `kdp_remote [ip addr]` or `gdb_remote [hostip : port]` commands.

Each kernel is packaged with kernel specific debug scripts as part of the build process. For security reasons these special commands
and scripts do not get loaded automatically when lldb is connected to machine. Please add the following setting to your `~/.lldbinit`
if you wish to always load these macros.

    settings set target.load-script-from-symbol-file true

The `tools/lldbmacros` directory contains the source for each of these commands. Please follow the [README.md](tools/lldbmacros/README.md)
for detailed explanation of commands and their usage.

[TN2118]: https://developer.apple.com/library/mac/technotes/tn2004/tn2118.html#//apple_ref/doc/uid/DTS10003352 "Kernel Core Dumps"
[TN2063]: https://developer.apple.com/library/mac/technotes/tn2063/_index.html "Understanding and Debugging Kernel Panics"
[Kernel Programming Guide]: https://developer.apple.com/library/mac/documentation/Darwin/Conceptual/KernelProgramming/build/build.html#//apple_ref/doc/uid/TP30000905-CH221-BABDGEGF
