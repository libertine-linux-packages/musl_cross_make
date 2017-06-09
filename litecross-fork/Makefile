OUTPUT = $(PWD)/output

GCC_CONFIG_FOR_TARGET = 
COMMON_CONFIG = 
GCC_CONFIG = 
NONGCC_CONFIG = 
MUSL_CONFIG = 

XGCC_DIR = ../obj_toolchain/gcc
XGCC = $(XGCC_DIR)/xgcc -B $(XGCC_DIR)

# RJC: Fix makeinfo issues
COMMON_CONFIG += MAKEINFO=false
export AM_MAKEFLAGS = INFO_DEPS=

# RJC: Sensible common config
COMMON_CONFIG += --with-debug-prefix-map=$(PWD)=
COMMON_CONFIG += --sysconfdir=/etc --sharedstatedir=/com --localstatedir=/var
COMMON_CONFIG += --disable-silent-rules
COMMON_CONFIG += --disable-nls
COMMON_CONFIG += --with-debug-prefix-map=/=
COMMON_CONFIG += --enable-static
COMMON_CONFIG += --disable-host-shared
#--disable-shared
COMMON_CONFIG += --disable-libada --disable-libgcj --disable-libvtv --disable-vtable-verify --disable-multiarch
COMMON_CONFIG += --disable-libquadmath --disable-decimal-float --disable-libquadmath-support

# RJC: build or host?
CROSS_COMPILE_PREFIX = 

LINUX_SRCDIR = src_kernel_headers



-include config.mak





ifneq ($(findstring fdpic,$(TARGET)),)
GCC_CONFIG_FOR_TARGET += --enable-fdpic
endif

ifneq ($(filter x86_64%x32,$(TARGET)),)
GCC_CONFIG_FOR_TARGET += --with-abi=x32
endif

ifneq ($(findstring mips64,$(TARGET))$(findstring mipsisa64,$(TARGET)),)
ifneq ($(findstring n32,$(TARGET)),)
GCC_CONFIG_FOR_TARGET += --with-abi=n32
else
GCC_CONFIG_FOR_TARGET += --with-abi=64
endif
endif

ifneq ($(filter %sf,$(TARGET)),)
GCC_CONFIG_FOR_TARGET += --with-float=soft
endif

ifneq ($(filter %hf,$(TARGET)),)
GCC_CONFIG_FOR_TARGET += --with-float=hard
endif


MAKE += MULTILIB_OSDIRNAMES=
MAKE += INFO_DEPS= infodir=
MAKE += ac_cv_prog_lex_root=lex.yy

TOOLCHAIN_CONFIG = \
	$(COMMON_CONFIG) \
	--enable-languages=c,c++ \
	--disable-werror \
	--target=$(TARGET) \
	--prefix= \
	--libdir=/lib \
	--disable-multilib \
	--with-sysroot=$(SYSROOT) \
	--with-build-sysroot=$(PWD)/obj_sysroot \
	--enable-tls \
	--disable-libmudflap \
	--disable-libsanitizer \
	--disable-gnu-indirect-function \
	--disable-libmpx \
	--enable-libstdcxx-time \
	--enable-deterministic-archives

# --enable-plugins if using lto
# --enable-ld=default --enable-gold
# --disable-install-libiberty
# --without-pic doesn't work
GCC_CONFIG += \
	--enable-fast-install \
	--with-zlib \
	--enable-plugin \
	--disable-lto --disable-libcc1 \
	--enable-64-bit-bfd \
	--enable-relro \
	CC="$(CROSS_COMPILE_PREFIX)cc -static --static -Wl,-Bstatic -static-libgcc $(ADDITIONAL_CFLAGS)" \
	CXX="$(CROSS_COMPILE_PREFIX)c++ -static --static -Wl,-Bstatic -static-libgcc -static-libstdc++ $(ADDITIONAL_CXXFLAGS)"

GCC_TOOLCHAIN_CONFIG = \
	$(TOOLCHAIN_CONFIG) \
	$(GCC_CONFIG) \
	$(GCC_CONFIG_FOR_TARGET)

FULL_MUSL_CONFIG = $(MUSL_CONFIG) \
	--prefix= \
	--host=$(TARGET)

ifeq ($(NATIVE),)
SYSROOT = /$(TARGET)
FULL_MUSL_CONFIG += CC="$(XGCC)" LIBCC="../obj_toolchain/$(TARGET)/libgcc/libgcc.a" 
MUSL_VARS = AR=../obj_toolchain/binutils/ar RANLIB=../obj_toolchain/binutils/ranlib
obj_musl/.lc_built: | obj_toolchain/$(TARGET)/libgcc/libgcc.a
obj_toolchain/.lc_built: | obj_sysroot/.lc_libs obj_sysroot/.lc_headers
else
SYSROOT = /
FULL_TOOLCHAIN_CONFIG += --host=$(TARGET)
MUSL_VARS = 
endif

all: musl toolchain

install: install-musl install-toolchain

musl: obj_musl/.lc_built

toolchain: obj_toolchain/.lc_built

.PHONY: all musl toolchain install-musl install-toolchain clean

obj_%:
	mkdir -p $@

obj_sysroot/include:
	mkdir -p $@

obj_sysroot/usr: | obj_sysroot
	ln -sf . $@

obj_sysroot/lib64: | obj_sysroot
	ln -sf lib $@

obj_toolchain/.lc_configured: | obj_toolchain
	mkdir obj_toolchain/gcc
	cd obj_toolchain && ../src_toolchain/configure $(GCC_TOOLCHAIN_CONFIG)
	touch $@

obj_toolchain/gcc/.lc_built: | obj_toolchain/.lc_configured obj_sysroot/usr obj_sysroot/lib64 obj_sysroot/include
	cd obj_toolchain && $(MAKE) MAKE="$(MAKE)" all-gcc
	touch $@

obj_musl/.lc_configured: | obj_toolchain/gcc/.lc_built obj_musl src_musl
	cd obj_musl && ../src_musl/configure $(FULL_MUSL_CONFIG)
	touch $@

obj_sysroot/.lc_headers: | obj_musl/.lc_configured obj_sysroot
	cd obj_musl && $(MAKE) DESTDIR=$(PWD)/obj_sysroot install-headers
	touch $@

obj_toolchain/$(TARGET)/libgcc/.lc_configured: | obj_sysroot/.lc_headers
	cd obj_toolchain && $(MAKE) MAKE="$(MAKE)" configure-target-libgcc
	touch $@

obj_toolchain/$(TARGET)/libgcc/libgcc.a: | obj_toolchain/$(TARGET)/libgcc/.lc_configured
	cd $(dir $@) && $(MAKE) MAKE="$(MAKE)" libgcc.a

obj_musl/.lc_built: | obj_musl/.lc_configured
	cd obj_musl && $(MAKE) $(MUSL_VARS)
	touch $@

obj_sysroot/.lc_libs: | obj_musl/.lc_built
	cd obj_musl && $(MAKE) $(MUSL_VARS) DESTDIR=$(PWD)/obj_sysroot install
	touch $@

obj_toolchain/.lc_built: | obj_toolchain/.lc_configured obj_toolchain/gcc/.lc_built
	cd obj_toolchain && $(MAKE) MAKE="$(MAKE)"
	touch $@

install-musl: | obj_musl/.lc_built
	cd obj_musl && $(MAKE) $(MUSL_VARS) DESTDIR=$(DESTDIR)$(OUTPUT)$(SYSROOT) install

install-toolchain: | obj_toolchain/.lc_built
	cd obj_toolchain && $(MAKE) MAKE="$(MAKE)" DESTDIR=$(DESTDIR)$(OUTPUT) install
	ln -sf $(TARGET)-gcc $(DESTDIR)$(OUTPUT)/bin/$(TARGET)-cc

TARGET_ARCH = $(firstword $(subst -, ,$(TARGET)))
TARGET_ARCH_MANGLED = $(patsubst i%86,x86,$(patsubst aarch64%,arm64%,$(TARGET_ARCH)))
LINUX_ARCH_LIST = $(sort $(notdir $(wildcard $(LINUX_SRCDIR)/arch/*)))
LINUX_ARCH = $(firstword $(foreach a,$(LINUX_ARCH_LIST),$(findstring $(a),$(TARGET_ARCH_MANGLED))))

all: kernel-headers

install: install-kernel-headers

kernel-headers: | obj_kernel_headers/.lc_built

obj_kernel_headers/.lc_built:
	mkdir -p $(PWD)/obj_kernel_headers/staged
	# If the kernel is patched with grsecurity, then grsecurity will cause this make line below to fail because `fixdeps` has not been built yet unless DISABLE_PAX_PLUGINS=y
	# The Linux kernel Makefile naively assumes 'gcc' for HOSTCC. Fix this.
	cd src_kernel_headers && $(MAKE) ARCH=$(LINUX_ARCH) O=$(PWD)/obj_kernel_headers INSTALL_HDR_PATH=$(PWD)/obj_kernel_headers/staged DISABLE_PAX_PLUGINS=y HOSTCC=cc HOSTCXX=c++ headers_install
	touch $@

install-kernel-headers: | obj_kernel_headers/.lc_built
	mkdir -p $(DESTDIR)$(OUTPUT)$(SYSROOT)/include
	cp -R obj_kernel_headers/staged/include/* $(DESTDIR)$(OUTPUT)$(SYSROOT)/include
