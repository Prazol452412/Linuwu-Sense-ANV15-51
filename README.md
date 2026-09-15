# Linuwu-Sense (personal fork)

This is my personal fork of [Linuwu-Sense](https://github.com/0x7375646F/Linuwu-Sense),
a community-maintained, reverse-engineered Linux kernel driver for Acer
Predator/Nitro laptop features (fan control, battery limiter, RGB keyboard,
thermal profiles, etc.) that Acer doesn't officially support on Linux.

All credit for the actual hardware reverse-engineering and the original
driver goes to the upstream project and its contributors. I'm not a kernel
developer by trade. I ran into a build failure, dug in far enough to
understand why it was failing, and fixed it. This fork exists mainly so
I don't lose that fix, and in case it's useful to anyone else hitting the
same issue before it's patched upstream.

## Why this fork exists

I hit a build failure on Linux kernel 7.2+ while trying to install this
driver through [DAMX](https://github.com/PXDiv/Div-Acer-Manager-Max) (a
suite that bundles this driver with a daemon and GUI). The short version:

**Kernel 7.2 removed `strncpy()` from the kernel entirely.** It had been
deprecated for years as part of a long-running cleanup effort. The
original driver's source still called `strncpy()` in three places, so on
kernel 7.2+ it simply fails to compile with an "implicit declaration of
function 'strncpy'" error.

The fix is straightforward once you know it: swap `strncpy()` for its
modern replacement, `strscpy()`. But the two functions expect their length
argument slightly differently. `strncpy(dest, src, len)` wants the number
of bytes to copy, while `strscpy(dest, src, size)` wants the total
destination buffer size (it always reserves the last byte for the null
terminator). Doing a naive find-and-replace without adjusting for that
difference introduces a quiet off-by-one bug: every value written through
these functions gets truncated by exactly one character.

I found this the hard way. After patching `strncpy` to `strscpy`, setting
fan speed to `30,30` was silently being stored as `30,3` internally. This
is what actually caused a "GPU fan doesn't work" symptom I was chasing
for a while. It wasn't a hardware/driver support issue at all, just a
string getting cut short by one character on the way in.

## What's actually changed from upstream

### `src/linuwu_sense.c`
- Replaced 3 calls to `strncpy()` with `strscpy()`, required for the
  driver to compile at all on kernel 7.2+.
- Adjusted the length argument passed to `strscpy()` (`len + 1` instead
  of `len`) to account for the different size semantics described above,
  fixing the truncation bug.

### `Linuwu-Sense.py` (the CLI control script)
- Added CPU, GPU, and system (ACPI thermal zone) temperature reporting to
  `--status`, read via `/sys/class/hwmon` (looked up by sensor name, not
  a hardcoded hwmon index, since hwmon numbering isn't stable across
  reboots or kernel updates) and `nvidia-smi` for GPU temp on NVIDIA
  systems. This is pure user-space Python and has no relationship to the
  kernel module. It's just convenient to have temps next to fan speed in
  one command.
- Added extra inline comments aimed at future-me (and anyone else)
  debugging this later, mostly around where failures can be silent:
  permission errors, or a write that "succeeds" but gets mangled by the
  kernel side.

### `dkms.conf`
- Added so the module rebuilds automatically via
  [DKMS](https://github.com/dell/dkms) whenever the system installs a new
  kernel, instead of needing a manual `make clean && make && make install`
  after every kernel update. If you're on a rolling-release distro
  (Arch, Omarchy, etc.), you'll want this, since kernel updates land
  often enough that manually rebuilding gets old fast.

I have not touched anything related to RGB keyboard modes, battery
calibration, or any of the WMI-call logic itself. Only the string-copy
mechanics and the CLI's status output.

## Installation

```bash
git clone https://github.com/Prazol452412/Linuwu-Sense-ANV15-51.git
cd Linuwu-Sense-ANV15-51
```

### Option A: manual build (rebuild required after every kernel update)
```bash
make clean
make
sudo make install
```

### Option B: DKMS (recommended, auto-rebuilds on kernel updates)
```bash
sudo pacman -S dkms   # or your distro's equivalent
sudo mkdir -p /usr/src/linuwu-sense-1.0
sudo cp -r src dkms.conf Makefile /usr/src/linuwu-sense-1.0/
sudo dkms add -m linuwu-sense -v 1.0
sudo dkms build -m linuwu-sense -v 1.0
sudo dkms install -m linuwu-sense -v 1.0
sudo modprobe linuwu_sense
```

Check it loaded:
```bash
lsmod | grep linuwu_sense
linuwu-sense --status
```

## A note on scope

I made these fixes to solve a problem I was personally stuck on, on my
own laptop, an Acer Nitro ANV15-51. I haven't tested this fork across the
full range of Predator/Nitro models the original project supports, and
I'm not in a position to promise ongoing maintenance beyond keeping my
own machine working. If something's broken for your specific model, the
[upstream repository](https://github.com/0x7375646F/Linuwu-Sense) and its
issue tracker are the right place to look first. This fork is narrowly
about the kernel 7.2 compile fix and my own CLI convenience additions,
not a general-purpose replacement.

## License

This project is licensed under the GNU General Public License v3.0, same
as upstream. See `LICENSE` for the full text.