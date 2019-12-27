---
title: Compile Linux Kernel to LLVM Bitcode
date: 2019-12-27 18:32:22
tags:
- LLVM
- Linux
---

# Tools
- [wllvm](https://github.com/SRI-CSL/whole-program-llvm) 1.2.7
- [clang](https://clang.llvm.org/) 9.0.1
- [LLVM](http://llvm.org/) 9.0.1
- [linux 5.3.6](https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.3.6.tar.gz)

# Compile Kernel with WLLVM
``` shell
make CC=wllvm defconfig
make CC=wllvm -j$(nproc)
```

# Extract Bitcode
``` shell
extract-bc vmlinux
```
Then vmlinux.bc is the bitcode.


# Cross-compile
``` shell
BINUTILS_TARGET_PREFIX=aarch64-linux-gnu make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- HOSTCC=clang CC=wllvm defconfig
BINUTILS_TARGET_PREFIX=aarch64-linux-gnu make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- HOSTCC=clang CC=wllvm -j$(nproc)
```

Then, extract bitcode from `vmlinux`.
``` shell
extract-bc vmlinux
```

# Problems
## symbol multiply defined during bitcode extraction
When extracting bitcode from vmlinux (arm64), an error occurs:
> error: Linking globals named 'sort': symbol multiply defined!

Get all `.bc` file names by
``` shell
extract-bc -m vmlinux
```
and it generates `vmlinux.llvm.manifest`, which is something like:
> /home/xx/dev/linux-5.3.6-arm-llvm/init/.main.o.bc
/home/xx/dev/linux-5.3.6-arm-llvm/init/.version.o.bc
/home/xx/dev/linux-5.3.6-arm-llvm/init/.do_mounts.o.bc
/home/xx/dev/linux-5.3.6-arm-llvm/init/.do_mounts_initrd.o.bc
/home/xx/dev/linux-5.3.6-arm-llvm/init/.initramfs.o.bc
/home/xx/dev/linux-5.3.6-arm-llvm/init/.calibrate.o.bc
/home/xx/dev/linux-5.3.6-arm-llvm/init/.init_task.o.bc
/home/xx/dev/linux-5.3.6-arm-llvm/arch/arm64/kernel/.debug-monitors.o.bc
/home/xx/dev/linux-5.3.6-arm-llvm/arch/arm64/kernel/.irq.o.bc


And then use a script to find all code that defines function `sort`:
``` shell
#!/bin/sh
cat vmlinux.llvm.manifest |
    while read line;
    do
        llvm-dis-9 "$line" -o .llvm.tmp.ll;
        cat .llvm.tmp.ll | grep -n '@sort(' | grep define | sed "s|^|${line}: |";
    done
```

We find that `sort` has been defined in both `lib/.sort.o.bc` and `drivers/firmware/efi/libstub/.lib-sort.o.bc`.

Read `drivers/firmware/efi/libstub/Makefile` and find that it first compiles all c files in `lib/` to lib-*.o by
``` Makefile
$(obj)/lib-%.o: $(srctree)/lib/%.c FORCE
	$(call if_changed_rule,cc_o_c)
```
and then use `objcopy` to add `__efistub_` prefix to them.
``` Makefile
$(obj)/%.stub.o: $(obj)/%.o FORCE
	$(call if_changed,stubcopy)

#
# Strip debug sections and some other sections that may legally contain
# absolute relocations, so that we can inspect the remaining sections for
# such relocations. If none are found, regenerate the output object, but
# this time, use objcopy and leave all sections in place.
#
quiet_cmd_stubcopy = STUBCPY $@
      cmd_stubcopy =							\
	$(STRIP) --strip-debug -o $@ $<;				\
	if $(OBJDUMP) -r $@ | grep $(STUBCOPY_RELOC-y); then		\
		echo "$@: absolute symbol references not allowed in the EFI stub" >&2; \
		/bin/false;						\
	fi;								\
	$(OBJCOPY) $(STUBCOPY_FLAGS-y) $< $@
```

However, it doesn't change the corresponding `.bc`, so error occurs.

If we don't care about efi, simply remove `drivers/firmware/efi/libstub/.*.o.bc` in `vmlinux.llvm.manifest` and then
``` shell
llvm-link -o vmlinux.bc `cat vmlinux.llvm.manifest`
```
can solve this problem.
