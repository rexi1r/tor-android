# Please install the following prerequisites (instructions for each follows):
# 	Android OS SDK: http://source.android.com/download
#
# Install and prepare the Android OS SDK ( http://source.android.com/download )
# on Debian or Ubuntu

### these modify the calling shell
# workaround for cross-compiling bug in autoconf
export ac_cv_func_malloc_0_nonnull=yes
export ac_cv_func_setpgrp_void=yes

EXTERNAL_ROOT := $(shell pwd)

DEBUG ?= 0

# No-op command.
NOOP = true

# Android now has 64-bit and 32-bit versions of the NDK for GNU/Linux.  We
# assume that the build platform uses the appropriate version, otherwise the
# user building this will have to manually set NDK_PROCESSOR or NDK_TOOLCHAIN.
CPU := $(shell uname -m)
ifeq ($(CPU),x86_64)
 NDK_PROCESSOR=x86_64
else
 NDK_PROCESSOR=x86
endif

# Android NDK setup
ANDROID_NDK_HOME ?= /opt/android-ndk
NDK_PLATFORM_LEVEL ?= 16
NDK_TOOLCHAIN_VERSION=4.9
APP_ABI ?= armeabi
NDK_ABI ?= $(APP_ABI)

# PIEFLAGS for SDK 16/Android L must be set to -fPIE -pie, but can override for earlier targets 
PIEFLAGS ?= -fPIE -pie

ifneq ($(filter arm%, $(APP_ABI)),)
  NDK_ABI := arm
endif
ifneq ($(filter arm64%, $(APP_ABI)),)
 NDK_ABI := arm64
endif
NDK_SYSROOT=$(ANDROID_NDK_HOME)/platforms/android-$(NDK_PLATFORM_LEVEL)/arch-$(NDK_ABI)
NDK_UNAME := $(shell uname -s | tr '[A-Z]' '[a-z]')
ifneq ($(filter mips%, $(NDK_ABI)),)
 HOST := $(NDK_ABI)el-linux-android
endif
ifneq ($(filter arm64, $(NDK_ABI)),)
 HOST := aarch64-linux-android
endif
ifneq ($(filter arm, $(NDK_ABI)),)
 HOST := arm-linux-androideabi
endif
ifneq ($(filter x86, $(NDK_ABI)),)
 HOST := i686-linux-android
endif
ifneq ($(filter x86_64, $(NDK_ABI)),)
 HOST := x86_64-linux-android
endif

NDK_TOOLCHAIN := $(HOST)-$(NDK_TOOLCHAIN_VERSION)
ifneq ($(filter x86%, $(NDK_ABI)),)
 NDK_TOOLCHAIN := $(NDK_ABI)-$(NDK_TOOLCHAIN_VERSION)
endif
NDK_TOOLCHAIN_BASE=$(ANDROID_NDK_HOME)/toolchains/$(NDK_TOOLCHAIN)/prebuilt/$(NDK_UNAME)-$(NDK_PROCESSOR)

# include Android's build flags
TARGET_ARCH_ABI = $(APP_ABI)
#include $(ANDROID_NDK_HOME)/toolchains/$(NDK_TOOLCHAIN)/setup.mk

CC := $(NDK_TOOLCHAIN_BASE)/bin/$(HOST)-gcc --sysroot=$(NDK_SYSROOT)
CXX := $(NDK_TOOLCHAIN_BASE)/bin/$(HOST)-g++ --sysroot=$(NDK_SYSROOT)
CPP := $(NDK_TOOLCHAIN_BASE)/bin/$(HOST)-cpp --sysroot=$(NDK_SYSROOT)
LD := $(NDK_TOOLCHAIN_BASE)/bin/$(HOST)-ld
AR := $(NDK_TOOLCHAIN_BASE)/bin/$(HOST)-ar
RANLIB := $(NDK_TOOLCHAIN_BASE)/bin/$(HOST)-ranlib

ifeq ($(DEBUG), 1)
	STRIP := $(NOOP)
else
	STRIP := $(NDK_TOOLCHAIN_BASE)/bin/$(HOST)-strip --strip-unneeded -R .note -R .comment --strip-debug
endif


CFLAGS = -DANDROID $(TARGET_CFLAGS) $(PIEFLAGS)
LDFLAGS = -llog $(TARGET_LDFLAGS) $(PIEFLAGS)

# change 'release' to 'debug' for unoptimized debug builds
ifeq ($(APP_ABI),armeabi-v7a)
	CFLAGS += $(TARGET_arm_release_CFLAGS)
endif
ifeq ($(APP_ABI),armeabi)
	CFLAGS += $(TARGET_thumb_release_CFLAGS)
endif



.PHONY = clean showsetup \
	assets assets-clean \
	openssl-static openssl-static-clean \
	libevent libevent-clean \
	lzma lzma-clean \
	zstd zstd-clean \
	tor tor-clean

all: assets

#------------------------------------------------------------------------------#
# openssl

# Disable rc4 cipher for 64-bit archs, to avoid this link error for tor:
# external/lib/libcrypto.a(e_rc4_hmac_md5.o):e_rc4_hmac_md5.c:function rc4_hmac_md5_cipher: error: undefined reference to 'rc4_md5_enc'
OPENSSL_CONF_FLAG=
ifneq ($(findstring 64, $(NDK_ABI)),)
 OPENSSL_CONF_FLAG+=no-rc4
endif

# OpenSSL's 'make depend' fails if _MIPS_SZLONG is not set:
ifneq ($(filter mips%, $(NDK_ABI)),)
 ifneq ($(findstring 64, $(NDK_ABI)),)
  OPENSSL_CONF_FLAG+=-D_MIPS_SZLONG=64
 else
  OPENSSL_CONF_FLAG+=-D_MIPS_SZLONG=32 -D__MIPSEL__
 endif
endif

lib/libcrypto.a:
	cd openssl && \
		./Configure android -DL_ENDIAN $(OPENSSL_CONF_FLAG) && \
		make CC="$(CC)" ANDROID_DEV=$(NDK_SYSROOT)/usr depend && \
		make CC="$(CC)" ANDROID_DEV=$(NDK_SYSROOT)/usr build_libs

lib/libssl.a:
	cp config.sub openssl
	cp config.guess openssl
	cd openssl && \
		./Configure android -DL_ENDIAN $(OPENSSL_CONF_FLAG) && \
		make CC="$(CC)" ANDROID_DEV=$(NDK_SYSROOT)/usr depend && \
		make CC="$(CC)" ANDROID_DEV=$(NDK_SYSROOT)/usr build_libs

openssl-build-stamp: lib/libcrypto.a lib/libssl.a
	touch openssl-build-stamp

openssl: openssl-build-stamp
	test -d lib || mkdir lib
	test -d include || mkdir include
	test -d include/openssl || mkdir include/openssl
	cp openssl/libcrypto.a lib/libcrypto.a
	cp openssl/libssl.a lib/libssl.a
	cp openssl/include/openssl/* include/openssl

openssl-clean:
	-rm openssl-build-stamp
	-rm lib/libcrypto.a
	-rm lib/libssl.a
	-cd openssl && \
		git clean -fdx

#------------------------------------------------------------------------------#
# libevent

libevent/Makefile:
	sed -i 's@\(SUBDIRS = . include\) sample test@\1@' libevent/Makefile.am
	cp libevent-patch-1 libevent
	-cd libevent && \
	 	patch -N -p1 --reject-file=- < libevent-patch-1
	##sed -i 's@\(AC_OUTPUT(Makefile include/Makefile\) test/Makefile sample/Makefile)@\1)@' libevent/configure.in
	cd libevent && ./autogen.sh
	cp config.sub libevent
	cp config.guess libevent
	cd libevent && \
		CC="$(CC)" AR="$(AR)" RANLIB=$(RANLIB) CFLAGS="$(CFLAGS) -I$(EXTERNAL_ROOT)/include" LDFLAGS="$(LDFLAGS)" \
			./configure \
				--host=$(HOST) \
				--disable-shared

libevent-build-stamp: libevent/Makefile
	$(MAKE) -C libevent ./include/event2/event-config.h all-am
	touch libevent-build-stamp

libevent: openssl libevent-build-stamp
	test -d lib || mkdir lib
	test -d include || mkdir include
	cp libevent/.libs/libevent.a lib
	cp -R libevent/include/event2 include

libevent-clean:
	-rm -f include/event2
	-rm -f lib/libevent.a
	-rm -f libevent-build-stamp
	-cd libevent && \
		git clean -fdx
#------------------------------------------------------------------------------#
# lzma

xz/Makefile:
	cd xz && ./autogen.sh
	cp config.sub xz
	cp config.guess xz
	cd xz && \
		 CC="$(CC)" CXX="$(CXX)" PKG_CONFIG_PATH="$(EXTERNAL_ROOT)/lib/pkgconfig" AR="$(AR)" RANLIB="$(RANLIB)" CFLAGS="$(CFLAGS) -std=c99" LDFLAGS="$(LDFLAGS)"  \
		./configure  --host=$(HOST) --disable-shared --enable-static --disable-doc --disable-xz --disable-xzdec --disable-lzmadec --disable-lzmainfo --disable-lzma-links --disable-scripts --prefix=$(EXTERNAL_ROOT)

lzma: lzma-build-stamp
	$(MAKE) -C xz install

lzma-build-stamp: xz/Makefile
	touch lzma-build-stamp

lzma-clean:
	-rm -rf include/lzma
	-rm -f include/lzma.h
	-rm -f lib/liblzma.a
	-rm -f lib/liblzma.la
	-rm -f lzma-build-stamp
	-cd xz && \
		git clean -fdx
#------------------------------------------------------------------------------#
# zstd

zstd: zstd-build-stamp
	$(MAKE) -C zstd/lib CC="$(CC)" CXX="$(CXX)" AR="$(AR)" RANLIB="$(RANLIB)" CFLAGS="$(CFLAGS)"  LDFLAGS="$(LDFLAGS)" PREFIX="$(EXTERNAL_ROOT)" libzstd.a-mt 
	$(MAKE) -C zstd/lib CC="$(CC)" CXX="$(CXX)" AR="$(AR)" RANLIB="$(RANLIB)" CFLAGS="$(CFLAGS)"  LDFLAGS="$(LDFLAGS)" PREFIX="$(EXTERNAL_ROOT)" libzstd.pc 
	test -d lib || mkdir lib
	test -d lib/pkgconfig || mkdir lib/pkgconfig
	test -d include || mkdir include
	cp zstd/lib/libzstd.a lib
	cp zstd/lib/libzstd.pc lib/pkgconfig
	cp zstd/lib/zstd.h include
	cp zstd/lib/common/zstd_errors.h include
	cp zstd/lib/deprecated/zbuff.h include
	cp zstd/lib/dictBuilder/zdict.h include

zstd-build-stamp:
	touch zstd-build-stamp

zstd-clean:
	-rm -f include/zstd.h include/zstd_errors.h include/zbuff.h include/zdict.h
	-rm -f lib/libzstd.a
	-rm -f lib/pkgconfig/libzstd.pc
	-rm -f zstd-build-stamp
	-cd zstd && \
		git clean -fdx
#------------------------------------------------------------------------------#
# tor

tor/configure: 
	cd tor && \
	 ./autogen.sh

tor/Makefile: tor/configure
	cp config.sub tor
	cp config.guess tor
	cd tor && \
                CC="$(CC)" AR="$(AR)" RANLIB=$(RANLIB) CFLAGS="$(CFLAGS) -D_FORTIFY_SOURCE=2 -fwrapv -fno-strict-aliasing -fno-strict-overflow" LDFLAGS="$(LDFLAGS)" \
                        LIBS="-L$(EXTERNAL_ROOT)/lib" CFLAGS="-I$(EXTERNAL_ROOT)/include -I$(EXTERNAL_ROOT)/include/event2" \
                        ./configure \
                                --host=$(HOST) \
				--disable-asciidoc \
				--enable-static-libevent --with-libevent-dir=$(EXTERNAL_ROOT) \
				--enable-static-openssl --with-openssl-dir=$(EXTERNAL_ROOT) \
				--disable-linker-hardening --disable-gcc-hardening --disable-tool-name-check --disable-systemd

tor-build-stamp: tor/Makefile
	$(MAKE) -C tor all-am
	touch tor-build-stamp

tor: lzma zstd libevent openssl tor-build-stamp
	test -d bin || mkdir bin
	cp tor/src/or/tor bin

tor-clean:
	-rm -f bin/tor
	-rm -f tor-build-stamp
	-cd tor && \
		git clean -fdx
	-cd tor && \
		git reset HEAD --hard

#------------------------------------------------------------------------------#
#create and clean assets: FYI - tor is stored as a ZIP file with an so extension
#in the libs directly, so it is handled like a shared library for local installation

assets: tor
	install -d ../tor-android-binary/src/main/libs/$(APP_ABI)
	-$(STRIP) bin/tor
	-zip ../tor-android-binary/src/main/libs/$(APP_ABI)/tor.so bin/tor

assets-clean:
	-rm ../tor-android-binary/src/main/libs/$(APP_ABI)/tor.so

#------------------------------------------------------------------------------#
# cleanup, cleanup, put the toys away

##clean: openssl-clean libevent-clean tor-clean polipo-clean assets-clean
clean: openssl-clean libevent-clean lzma-clean zstd-clean tor-clean

#------------------------------------------------------------------------------#
# debugging stuff

showsetup:
	@echo "NDK_TOOLCHAIN_VERSION: $(NDK_TOOLCHAIN_VERSION)"
	@echo "NDK_TOOLCHAIN: $(NDK_TOOLCHAIN)"
	@echo "NDK_PLATFORM_LEVEL: $(NDK_PLATFORM_LEVEL)"
	@echo "NDK_SYSROOT: $(NDK_SYSROOT)"
	@echo "APP_ABI: $(APP_ABI)"
	@echo "HOST: $(HOST)"
	@echo "CC: $(CC)"
	@echo "LD: $(LD)"
	@echo "CFLAGS: $(CFLAGS)"
	@echo "LDFLAGS: $(LDFLAGS)"
