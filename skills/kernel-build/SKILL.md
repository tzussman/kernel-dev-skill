---
name: kernel-build
description: Use when the user wants to build a Linux kernel, configure it,
  install modules, or cross-compile. Covers make, LLVM/GCC toolchains, config
  fragments, ccache, and install workflows. Trigger on "build the kernel",
  "make", "menuconfig", "defconfig", "cross-compile", "install modules".
---

## Core build patterns

### Detect the toolchain
Check what's available before building:
    which clang && clang --version    # LLVM toolchain
    which ccache                       # build cache
    nproc                              # core count

### Standard builds
    # GCC (default)
    make -j$(nproc)

    # Clang/LLVM
    make LLVM=1 -j$(nproc)

    # With ccache
    make CC="ccache gcc" -j$(nproc)
    make LLVM=1 CC="ccache clang" -j$(nproc)

### Configuration

Always check for a project-specific build script (build.py, build.sh,
Makefile wrapper) before driving make directly -- many trees have one.

    make defconfig                     # arch default
    make olddefconfig                  # update .config, accept defaults for new symbols
    make menuconfig                    # interactive (human only, never from tool call)

#### Config fragments (overlay additional options)
    # Merge fragments on top of existing .config
    scripts/kconfig/merge_config.sh .config fragment1.config fragment2.config

    # Or use make with config items
    scripts/config --enable CONFIG_BPF
    scripts/config --set-val CONFIG_LOG_BUF_SHIFT 18
    scripts/config --disable CONFIG_MODULES
    make olddefconfig                  # resolve dependencies after manual edits

#### Common debug configs
    scripts/config --enable CONFIG_DEBUG_INFO_BTF
    scripts/config --enable CONFIG_KASAN
    scripts/config --enable CONFIG_PROVE_LOCKING
    scripts/config --enable CONFIG_DEBUG_INFO_DWARF5
    make olddefconfig

### Install (requires appropriate permissions)
    make modules_install    # installs to /lib/modules/$(make kernelrelease)
    make install            # installs bzImage + System.map to /boot

### Cross-compilation
    # ARM64 with LLVM
    make ARCH=arm64 LLVM=1 -j$(nproc) defconfig
    make ARCH=arm64 LLVM=1 -j$(nproc)

    # ARM64 with GCC cross-compiler
    make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)

### Partial / fast rebuilds
    make -j$(nproc) mm/           # rebuild only mm/ objects
    make -j$(nproc) kernel/bpf/   # rebuild only kernel/bpf/ objects
    make bzImage -j$(nproc)       # skip modules entirely (fastest)

### Build a single file (syntax check)
    make mm/memory.o              # compile one .c -> .o
    make mm/memory.i              # preprocessor output only
    make mm/memory.s              # assembly output

### Generate clangd compile_commands.json

After building with clang (LLVM=1), generate the compilation database for
clangd / IDE integration:

    scripts/clang-tools/gen_compile_commands.py

This reads the `.*.cmd` files left by the build and produces
`compile_commands.json` in the tree root. Run it after every build (or
after a partial rebuild) so clangd picks up the correct flags.

When building with LLVM=1, always offer to regenerate compile_commands.json
after the build completes.

## Gotchas
- `make olddefconfig` after ANY manual .config edit -- otherwise
  dependencies won't resolve and the build may silently drop symbols.
- `make mrproper` nukes .config -- only use when starting fresh.
- LLVM builds need a sufficiently recent LLVM (check
  Documentation/process/changes.rst for minimum version).
- BTF generation (`CONFIG_DEBUG_INFO_BTF`) requires pahole >= 1.16.
- Out-of-tree builds: `make O=/path/to/builddir` keeps the source tree
  clean but breaks some scripts that assume in-tree build.

## Decision tree
- *Quick test of a code change* -> `make -j$(nproc)` (or `make LLVM=1`)
- *Need a specific config* -> `scripts/config --enable/--disable` then
  `make olddefconfig` then build
- *Debug build* -> enable KASAN/lockdep/BTF configs, then build
- *Just check syntax* -> `make mm/somefile.o`
- *Full install for boot testing* -> build, `make modules_install`,
  `make install`, then boot (or `vng`)
