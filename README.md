Building and using GCC cross compilers
============================

Building GCC cross compilers from scratch
-----------------------------------------

This is not crosstool(-ng). If you want all in one magic script and are
not interested in tuning or understanding what's going on, use crosstool-ng
and spend the rest of your time with your kids.

If you don't have any or it's late at night anyway, read on ...

(for having just the secret configure arguments and the make targets, see
 below)

### Using the provided scripts for each package

Coarse overview:

- build and install binutils
- copy and install kernel headers
- build and install stage1 gcc
  (this one cannot compile userland programs, but is good
   enough for the kernel or bootloaders)
- build and install cross-glibc
- build and install final gcc

	$ export TARGET=aarch64

tested targets so far:

* aarch64 (ARM64)
* arm (softfloat)
* armhf (ARMv7 hard float)
* mips64
* openwrt (MIPS32 uclibc for OpenWRT)

1. create build directories, we don't build inside the source dir
2. build binutils (use latest upstream source):

	$ cd binutils-aarch64
	$ ../cross_build_binutils package /path/to/binutils/src

3. install binutils

	$ sudo dpkg -i binutils-aarch64-linux-gnu_2.25-1.deb

4. create kernel headers

... (more to follow) ...

### Hacker's hints:

* use the same target, build, host arguments for binutils and gcc
  target is your target, build and host are your current machine's ones
* glibc is actually already cross-compiled, so host is now your target
  machine's triplet while build is still your local machine's one. target
  is not used in glibc cross compilation process (just for compilers)
* choose and provide the same --with-sysroot= directory for binutils and gcc
* don't forget to install the kernel headers for that arch into sysroot
  before cross-compiling glibc
* --disable-shared in binutils relates to the host's binaries, use it to
  link libbfd statically into ld & friends and avoid clashes between different
  versions of the the cross binutils and your native ones
* always use --disable-bootstrap on both compiler builds
* to produce a libc-less stage1 cross compiler, use:

	--without-headers --with-newlib
	--disable-shared --disable-threads
	--disable-<various helper libraries>

  Don't worry, this is just for stage1, stage2 will fix this.
  Use

	$ make all-gcc all-target-libgcc

  to avoid building yet impossible targets.
  You can use that compiler already to compile the kernel.
* for cross-building glibc, give it the installed kernel headers directory
  with --with-headers=<sysroot>/usr/include
* for a gcc stage2 build, install the glibc into <sysroot> first
* on building gcc stage2, you can drop most of the limiting options from
  the stage1 build, just keep:

	--target=... --host=... --build=...
	--disable-bootstrap
	--with-sysroot=<sysroot>

* check for hidden errors: sometimes there is an error happening in the build
  process somewhere and it scrolls by, leaving you with a partial build.
  Especially true with make install, where you end up with not all files
  installed, leading to all kind of strange errors later.

### Magic configure recipies for the initiated ones:

Example configure options and make targets for a cross aarch64 compiler
with /usr/gnemul/aarch64 as sysroot, assuming a non-multiarch build:  

#### binutils:
	$ ../binutils-gdb/configure --prefix=/usr --with-gnu-ld --with-gnu-as \
	--target=aarch64-linux-gnu \
	--build=x86_64-linux-gnu --host=x86_64-linux-gnu \
	--with-sysroot=/usr/gnemul/aarch64 \
	--disable-bootstrap \
	--disable-shared --disable-nls --enable-multilib \
	--enable-threads \
	--enable-plugins \
	--enable-gold=yes --enable-ld=default \
	--with-lib-path=/usr/aarch64-linux-gnu/lib64:/usr/gnemul/aarch64/usr/local/lib64:/usr/gnemul/aarch64/lib64:/usr/gnemul/aarch64/usr/lib64
	$ make && make DESTDIR=$(pwd)/root install

#### cross gcc stage1:
	$ ../gcc/configure --prefix=/usr --with-gnu-ld --with-gnu-as \
	--target=aarch64-linux-gnu \
	--build=x86_64-linux-gnu --host=x86_64-linux-gnu \
	--with-sysroot=/usr/gnemul/aarch64 \
	--disable-bootstrap \
	--disable-shared --disable-nls --enable-multilib \
	--disable-threads \
	--with-newlib --without-headers \
	--enable-languages=c \
	--with-system-zlib \
	--disable-libgomp --disable-libitm --disable-libquadmath \
	--disable-libsanitizer --disable-libssp --disable-libvtv \
	--disable-libcilkrts --disable-libatomic

(add:

	--with-arch= --with-cpu=... --with-abi=... --with-float=...

settings to set default settings for the generated compiler, e.g.:

	--with-arch=armv7-a --with-float=hard

for an ARMv7-HF build)

	$ make all-gcc all-target-libgcc
	$ make DESTDIR=$(pwd)/root install-gcc install-target-libgcc

#### cross glibc:
	$ ../glibc/configure --prefix=/usr --with-gnu-ld --with-gnu-as \
	libc_cv_forced_unwind=yes libc_cv_c_cleanup=yes \
	libc_cv_gnu89_inline=yes \
	--host=aarch64-linux-gnu \
	--build=x86_64-linux-gnu \
	--without-cvs \
	--disable-nls \
	--disable-sanity-checks \
	--enable-obsolete-rpc \
	--disable-profile \
	--disable-debug \
	--without-selinux \
	--with-tls \
	--enable-kernel=3.7.0 \
	--with-headers=/usr/gnemul/aarch64/usr/include \
	--enable-hacker-mode

#### cross gcc stage2:
	$ ../gcc/configure --prefix=/usr --with-gnu-ld --with-gnu-as \
	--target=aarch64-linux-gnu \
	--build=x86_64-linux-gnu --host=x86_64-linux-gnu \
	--with-sysroot=/usr/gnemul/aarch64 \
	--disable-bootstrap \
	--enable-shared --disable-nls --enable-multilib \
	--enable-languages=c,c++ \
	--with-system-zlib
	$ make && make DESTDIR=$(pwd)/root install

Go ahead and test it by compiling helloworld.c, and let

	$ file hellow

confirm that the compiled binary is for your target architecture.


Cross-building libraries and programs
--------------------------------------

Depending on the cross-compiler friendliness of your package, you may
end up with different solutions for cross-compilation:

### cross-compile enabled GNU autotools

Use:

	$ ./configure \
	--build=<the machine's triplet you build on> \
	--host=<the target's machine triplet> \
	...

Don't omit --build or configure will not properly recognise that you
are cross-compiling. That _sometimes_ work, but is wrong.
In opposite to native builds, --build will not be auto-guessed (which
is annoying), so you may want to use --build=$(gcc -dumpmachine) to
let your native gcc tell configure the build machine's triplet.
Please don't hardcode x86_64-linux-gnu in scripts ;-)

Example:

	$ ./configure --build=x86_64-linux-gnu --host=aarch64-linux-gnu ...

### cross-compile enabled non-autotools

Put your target's machine triplet _and a trailing hyphen_ into
the CROSS_COMPILE environment variable.

	$ export CROSS_COMIPLE=aarch64-linux-gnu-
	$ make
Some packages (Linux, Xen, u-boot come to mind) expect a specifier for the
target architecture in the ARCH environment variable.

	$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- ....
This ARCH specifier name may differ from the architecture name in the
GNU triplet (for instance Xen uses arm32 instead of arm, Linux arm64 instead
of aarch64).

### half-way sane Makefiles

Most Makefiles honour the CC environment variable to point to the actual
system's compiler. So you either set

	CC=aarch64-linux-gnu-gcc
before calling make or add a line saying

	CC=${CROSS_COMPILE}gcc
somewhere at the top of the Makefile.

Some Makefiles also use LD, AS, AR and possibly other variables to get
the tools actual name, so you may end up specifying those as well.
Please note that this does not always end up in a working target's binary,
as the Makefile is probably not aware of cross-compilation and may use tests
run on your build machine to (mis)guess target's behaviour.
pkg-config calls for instance would query your build machine's library and
include files to detect availability of libraries, which usually does
not give you a clue about the situation on the target machine.
You may want to either replace the pkg-config calls with the output of it
on the target's machine (quick, but dirty) or teach pkg-config about the
location of the target machine's .pc files. This will most likely get you
the wrong include directory path (relative to SYSROOT), but it may work
nevertheless if that matches your host's path, which is quite likely. If in
doubt or in a hurry, install the library also on your host to get probably
the same header files. YMMV ;-)


### totally ignorant packages

Go ahead and replace any reference of "cc" or "gcc" in the Makefile with
${CC} and add a

	CC=${CROSS_COMPILE}gcc
somewhere at the top of the Makefile. The use the "halfway sane" approach above.

