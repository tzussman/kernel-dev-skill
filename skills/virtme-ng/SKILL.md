---
name: virtme-ng
description: Use when the user wants to boot, test, or run commands in a Linux kernel VM via virtme-ng (`vng`). Covers non-interactive command execution, ensuring the serial console captures kernel output, sharing directories, CPU/memory tuning, and debugging crashes. Trigger on mentions of vng, virtme-ng, virtme, "boot the kernel", "try in a VM", "run under vng".
---

## What virtme-ng is

`vng` boots the kernel built in the current directory (or one passed to
`--run`) inside QEMU, sharing the host rootfs **copy-on-write** so the guest
sees the host's binaries, libraries, and `$CWD` — but any writes stay in the
VM. No install step, no disk image. Useful for kernel testing, running
selftests, and reproducing crashes without risking the host.

## Installation

Before first use, verify vng is installed and working:

```bash
which vng && vng --version
```

If not installed:

### 1. Install QEMU (the VM backend)

```bash
# Debian/Ubuntu
sudo apt-get install -y qemu-system-x86

# Fedora
sudo dnf install -y qemu-system-x86-core

# Arch
sudo pacman -S qemu-system-x86
```

KVM acceleration requires `/dev/kvm` — check with `ls /dev/kvm`. If missing,
load the module (`modprobe kvm_intel` or `kvm_amd`) and ensure your user is
in the `kvm` group (`sudo usermod -aG kvm $USER`, then re-login).

### 2. Install virtme-ng

```bash
pip install --user virtme-ng
```

This installs `vng` (and `virtme-ng`) to `~/.local/bin/`. Make sure that
directory is on your `$PATH`:

```bash
export PATH="$HOME/.local/bin:$PATH"    # add to ~/.bashrc if needed
```

### 3. Verify

```bash
vng --version                           # should print version
vng -- uname -a                         # quick smoke test
```

If the smoke test fails with a QEMU error, check:
- `qemu-system-x86_64` is on `$PATH`
- `/dev/kvm` exists and is accessible (or use `--disable-kvm` for TCG)
- The kernel in `$CWD` is built (bzImage exists in `arch/x86/boot/`)

### Upgrading

```bash
pip install --user --upgrade virtme-ng
```

## MCP tool vs Bash: when to use which

virtme-ng ships an MCP server (`vng --mcp` / `vng-mcp`) that exposes four
tools: `run_kernel`, `configure_kernel`, `get_kernel_info`, `apply_patch`.

If the MCP server is available (tools named `mcp__virtme-ng__*`), **prefer
the `run_kernel` MCP tool for simple runs** — it handles PTS allocation
automatically and returns structured JSON. But the MCP tool has blind spots:

- **No dmesg checking.** It returns raw stdout; it does NOT scan for kernel
  errors. A command can return `success: true` while the kernel has a BUG
  splat in dmesg. **Always append a dmesg check to the command you pass:**

      run_kernel(command="<CMD>; echo '=== DMESG ==='; dmesg --level=err,warn,crit,alert,emerg")

- **No serial console.** The tool uses default microvm console. Early boot
  panics and late oopses are silently lost. For debugging, **fall back to
  Bash** with `--disable-microvm` and the serial console flags.

- **No hang diagnosis.** Timeout just returns "timed out" with no retry or
  debug output. For hang diagnosis, **fall back to Bash** with the two-stage
  approach described below.

- **No `--disable-microvm`, `--append`, `--pin`, `--rwdir`** or other
  advanced flags. The tool only exposes cpus, memory, network, debug, arch.
  For anything else, **use Bash directly.**

### Decision: MCP tool or Bash?

- *Quick smoke test, simple command* → `run_kernel` MCP tool (with dmesg
  appended to command)
- *Debugging a hang or panic* → Bash with `--disable-microvm` + serial
- *Benchmarks (need --pin)* → Bash
- *Need --rwdir, --append, --overlay-rwdir* → Bash
- *Applying lore patches* → `apply_patch` MCP tool (or `b4 shazam` via Bash)
- *Config generation* → `configure_kernel` MCP tool (or `scripts/config` via Bash)

### Critical rule for Bash invocations

**Never invoke bare `vng` from a tool call** — it drops into an interactive
shell and blocks forever. Always use one of:

    vng -- <command>          # exec command, exit when it finishes
    vng --exec '<command>'    # same, quoted form (use when piping/chaining)

Only a human at the terminal should run bare `vng`.

## Detecting boot hangs and failures

A vng command that produces no output and doesn't return is almost always a
**boot hang** — the kernel got stuck before reaching userspace. This is the
most common failure mode when testing kernel changes. **Always set a timeout
and use serial console output to diagnose.**

### Always use a timeout

When running vng from a tool call, **always set a timeout** (60-120s for
simple commands, up to 300s for heavy tests). If the command times out with
no output, that's a boot hang — not a slow command.

### Diagnosing a hang: two-stage approach

**Stage 1: Quick run with serial console to see where it stops.**

```bash
timeout 90 vng --user root --disable-microvm \
    --append "console=ttyS0,115200 earlyprintk=serial,ttyS0,115200 nokaslr panic=-1 oops=panic loglevel=7" \
    -- uname -a 2>&1
```

If this times out, the output will show how far the kernel got before
hanging. Common patterns:

- **No output at all** — kernel didn't start. Check that bzImage exists
  and was built for the right arch. Try `--disable-kvm` to rule out KVM
  issues.
- **Stops during early boot (before "Booting Linux...")** — QEMU/firmware
  issue, not a kernel bug. Check QEMU version and flags.
- **Stops after decompression but before init** — kernel panic or hang in
  a subsystem init. The last few `dmesg` lines identify the subsystem.
  Common culprits: missing config options, broken module init, filesystem
  mount failure.
- **Stops at "virtme-ng-init" or after "udev is done"** — the kernel
  booted but userspace init hung. Check that the command is valid and
  that required filesystems/devices are available.
- **Kernel oops/panic visible** — read the stack trace. With `oops=panic`
  and `panic=-1` it will halt rather than reboot, preserving the output.

**Stage 2: If stage 1 produces no useful output, add `--verbose` to see
the full QEMU command line:**

```bash
timeout 90 vng --verbose --user root --disable-microvm \
    --append "console=ttyS0,115200 earlyprintk=serial,ttyS0,115200 nokaslr" \
    -- uname -a 2>&1
```

`--verbose` prints the QEMU invocation so you can spot misconfigured flags.

### After every vng invocation

When a `vng` command succeeds, **check the output for kernel warnings or
oopses** before declaring success. A command can return exit code 0 while
the kernel log contains splats. For important runs:

```bash
vng --user root -- sh -c '<command>; echo "=== DMESG ==="; dmesg -T --level=err,warn,crit,alert,emerg' 2>&1
```

This appends error-level kernel messages after the command output so boot
issues and runtime warnings are visible even when the command itself
succeeds.

### Killing a stuck VM

If a vng invocation hangs and you need to kill it:

```bash
# Find and kill the QEMU process
pkill -f 'qemu-system.*virtme'
```

## Serial console — ensuring you actually see kernel output

By default on x86_64, vng uses the `microvm` QEMU machine, which routes the
console through **virtio-console (`hvc0`)**, not a traditional serial port.
This is fine for user-space output but has two consequences:

1. Anything that expects `/dev/ttyS0` (GDB stub, some test harnesses,
   `earlyprintk=serial`) will not work.
2. Very early boot messages and late panic output can be lost.

When you need a real serial port and full kernel log capture, combine:

    vng --disable-microvm \
        --append "console=ttyS0,115200 earlyprintk=serial,ttyS0,115200" \
        -- <command>

Useful additions for debugging:

    --append "console=ttyS0,115200 earlyprintk=serial,ttyS0,115200 \
              nokaslr panic=-1 oops=panic loglevel=7 ignore_loglevel"

- `console=ttyS0,115200` — kernel log to serial
- `earlyprintk=serial,ttyS0,115200` — messages before console init
- `nokaslr` — stable addresses for symbol resolution
- `panic=-1` — halt immediately on panic (don't reboot and lose output)
- `oops=panic` — turn oopses into panics so they're impossible to miss
- `loglevel=7 ignore_loglevel` — print everything, including `pr_debug`
  on builds where dynamic debug is compiled in

For a quick-boot run where you just want dmesg at the end, the default
microvm console is fine and you don't need `--disable-microvm`.

## Common invocations

### Run one command, get output, exit
    vng -- uname -a
    vng -- sh -c 'dmesg | tail -50'

### Run as root (needed for most kernel tests, BPF, mount, modprobe)
    vng --user root -- <command>

### Boot a specific prebuilt kernel
    vng --run ./arch/x86/boot/bzImage -- <command>
    vng --run /path/to/builddir -- <command>
    vng --run v6.6.17 -- <command>        # downloads upstream prebuilt

### Build in-tree then boot (vng drives `make`)
    vng --build
Usually better to build with your normal toolchain (`make LLVM=1 -j$(nproc)`
or whatever the project uses) and then just `vng` — vng auto-detects the
in-tree build.

### Override kernel config fragments
    vng --config debug.config --config kasan.config -- <command>
    vng --configitem CONFIG_DEBUG_INFO_BTF=y -- <command>

## Sharing files with the host

The host filesystem is mounted **read-only copy-on-write** by default —
reads work, writes land in a throwaway overlay.

    --rwdir /host/path                   # r/w passthrough (writes hit host!)
    --rwdir guest=/host/path             # mount at different guest path
    --overlay-rwdir /some/dir            # writable in guest, host untouched
    --rodir /host/path                   # explicit read-only share
    --rw                                 # WHOLE rootfs r/w — DANGEROUS

Use `--overlay-rwdir` when a test writes files you want to discard after.
Use `--rwdir` only when you need writes to persist on the host.

## Resource tuning

    --cpus 8, -p 8                       # guest vCPU count
    --memory 8G, -m 8G                   # guest RAM
    --pin                                # pin vCPUs to host CPUs (low jitter)
    --pin 0-7                            # pin to specific host CPUs
    --numa 4G,cpus=0-3 --numa 4G,cpus=4-7   # NUMA topology (implies non-microvm)
    --balloon                            # allow host to reclaim guest memory

For benchmark runs use `--pin` and a matching `--cpus` to minimize jitter;
for NUMA-sensitive tests, add `--numa`/`--numa-distance`.

## Networking

    --network user                       # user-mode NAT, outbound works
    --network bridge=br0                 # bridge to host interface
    --ssh                                # start sshd in guest, connect from host

## Debugging kernel crashes

    vng --debug -- <command>             # enables crash-dump capability
    vng --gdb                            # attach gdb to a --debug instance
    vng --dump /tmp/crash.dump           # snapshot a running --debug instance

Combined with the serial-console `--append` line above, this is the go-to
setup for reproducing and root-causing oopses.

To drop into a shell on panic instead of halting:

    --append "panic=0"                   # default behavior, reboots
    --append "panic=-1"                  # halt (preferred for debugging)

## Gotchas

- **KVM required by default.** If `/dev/kvm` is absent or unusable, add
  `--disable-kvm` (much slower, TCG emulation).
- **`$CWD` is preserved** — relative paths in commands work, but only if
  `$CWD` is under a path shared with the guest (the host rootfs, by default).
- **`--exec` vs `-- <cmd>`**: `-- <cmd>` is the modern form and what you
  want most of the time. `--exec` exists for older scripts and behaves the
  same for simple commands.
- **microvm limitations**: no PCI, no NUMA, no real serial, no graphics,
  reduced device set. Add `--disable-microvm` whenever a test needs any of
  these, or when `--numa`, `--sound`, `--graphics`, or passthrough is used
  (vng does this automatically for some of them).
- **Kernel config must include what the test needs.** vng's auto-generated
  minimal config omits a lot — if a test fails with `ENOSYS` or missing
  `/proc` entries, rebuild with the required `CONFIG_*` enabled (via
  `--config` fragments or the project's own build system).
- **`--force` resets the git tree** to the target branch/commit and can drop
  uncommitted changes. Never pass it unless the user explicitly asks.
- **Output ordering**: kernel messages on ttyS0 and userspace stdout on
  hvc0 may interleave oddly. For a clean log, route both to serial via
  `--disable-microvm` + the console append line.
- **vng prepends its own kernel command line** including `quiet loglevel=0`
  and `console=ttyS0` (without baud rate). Your `--append` values come
  after, so for params where the kernel uses the last value (loglevel,
  console), your overrides win. But for params where the first value wins,
  vng's defaults may take precedence. Always verify with
  `dmesg | grep "Command line"` if behavior seems wrong.

## Quick decision tree

- *Just run a command and see its output* → `run_kernel` MCP tool (append
  dmesg check to command), or `vng -- <cmd>` via Bash
- *Command needs root* → `run_kernel` MCP tool (runs as root by default
  with `script`), or `vng --user root -- <cmd>` via Bash
- *Need kernel log / debugging a crash* → **Bash only:** add
  `--disable-microvm` and the full `--append "console=ttyS0,115200
  earlyprintk=... nokaslr panic=-1"`
- *VM hangs / no output* → **Bash only:** see "Detecting boot hangs"
  section; re-run with `timeout`, `--disable-microvm`, serial console,
  and `--verbose` to diagnose
- *Writes need to persist on host* → **Bash only:** `--rwdir <path>`
- *Benchmark / low-jitter run* → **Bash only:** `--pin --cpus N --memory NG`
- *Need sshd / network services* → **Bash only:** `--network user --ssh`
- *Apply lore patches* → `apply_patch` MCP tool
- *Generate/check .config* → `configure_kernel` or `get_kernel_info` MCP tool
