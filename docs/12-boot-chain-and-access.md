# 12 · Boot Chain, Flash Layout & Access Investigation

This doc records what the **serial boot log** (`Boot.log`) and the **config-decode
attempt** taught us about how the box starts and how its administrative access is
structured. It's a factual record of the investigation on my own lab unit — the
architecture, the constraints, and the open questions — not a step-by-step
procedure.

## The boot chain, stage by stage

From `Boot.log` (a PuTTY serial capture), the box comes up in the classic
embedded order:

```
 SoC ROM ──► bootbase ──► Linux kernel ──► OpenWrt userspace
 (QFP IC,     (EN7528       (3.18.21 SMP,     (cfgmgr, mesh
  DRAM init)   "free          4× MIPS 1004Kc)   daemons, etc.)
               bootbase"
               v1.1)
```

Verified facts, each with the line that shows it:

| Fact | Evidence (`Boot.log`) |
|------|-----------------------|
| SoC / DRAM | `QFP IC` → `DRAM size=512MB` → `7528DRAMC V1.8` |
| Bootloader | `EN7528 at Fri May 13 22:20:23 CST 2022 version 1.1 free bootbase` |
| Flash | `SPI NAND … MT29F2G01, Flash Size=0x10000000` (256 MB), with `BMT & BBT` bad-block management |
| **A/B images** | `tclinux` **and** `tclinux_slave` partitions — the dual firmware banks (ties to `zeroman/imageCounter`, doc 11) |
| Kernel | `Linux version 3.18.21 … #7 SMP Wed Dec 25 23:52:59 CST 2024` |
| **CPU** | `CPU0 … MIPS 1004Kc`, `Detected 3 available secondary CPU(s)`, `CPUs=4` → **4 cores** (this is the correction in docs 01/06/10) |
| Machine | `MIPS: machine is econet,en751221` (EcoNet/Airoha platform) |
| Usable RAM | `Memory: 422084K/438264K available` (~422 MB, matches `meminfo.txt`) |

So the "single-core" story from the data model is firmly disproved here: it's a
**4-core MIPS 1004Kc** running a **2015-vintage kernel (3.18)** built Dec 2024.

## Two things in the boot log worth calling out

**1. The bootloader prints two interrupt prompts.** Early in boot, `Boot.log`
shows `Press 'x' or 'b' key in 1 secs to enter or skip bootloader upgrade` and
then `Press any key in 3 secs to enter boot command mode`. These are stock EcoNet
"free bootbase" prompts — the bootloader offers a short window for interactive
commands before it hands off to Linux. (Our capture is of a *normal, uninterrupted*
boot, so what the command mode actually exposes — and whether it's gated — is **not
yet observed**, only advertised by the prompt.)

**2. `disable serial by default`.** Immediately after the boot-command-mode prompt,
the log prints `disable serial by default`, and later `console [ttyS0] enabled`
only during kernel bring-up. The practical meaning: the **interactive serial
console is not available from the running OpenWrt system** by default — the serial
port is primarily a *boot-time* window, not a login getty. This is a deliberate
lock-down and explains why a serial cable alone doesn't drop you at a prompt on a
booted box.

## Flash & image layout (why reboots are survivable)

The 256 MB SPI-NAND holds **two full firmware banks** (`tclinux` /
`tclinux_slave`). Combined with the `zeroman` image counter (doc 11) and the
`reboot.log` image-validity checks, this is a standard **A/B failsafe**: if one
bank fails to boot, the loader falls back to the other. It's why the box tolerates
75+ power-cycles and a watchdog reset (doc 10) without bricking — and why any
firmware-level work has a built-in recovery path.

## Administrative access — investigation status

The goal was to enable a full shell for a UID-0 account (doc 11). Here's the honest
state after this session, framed as findings and constraints:

### The config-backup route is blocked — twice over

1. **`config.cfg` (web-UI backup) won't decode with the current tool.**
   `workdir/nokia-cfg.py` (the public thedroidgeek/rajkosto tool) runs correctly,
   parses the container (little-endian, magic `23 31 12 00`, CRC **valid**), but the
   inner payload **fails every standard decompressor**. Measured entropy **7.998
   bits/byte** and a non-16-aligned length ⇒ the payload is **encrypted with a key
   the 2022-era tool doesn't carry**. This is a Dec-2024 `L88` build; the backup
   format evolved past the tool.
   - *Two different config files, two different schemes* (worth keeping straight):
     `config_encryption.cfg` (on-device storage, per-device key, doc 11) is a
     *different* file and format from the web-UI `config.cfg` backup. Neither is
     readable with what's on hand.

2. **Even decoded, the config wouldn't change the shell.** Grepping *every* config
   backup (`etc/config/*`, `alcatel/config/*`) for `telnet`, `/bin/sh`, `vtysh`,
   `ONTUSER`, `administrator` returned **nothing**. The account/shell definitions
   live in `/etc/passwd`, which is **baked into the firmware image** and overlaid at
   boot — not sourced from the editable config. So the shell isn't a config knob.

### Where that leaves things (the map, not a procedure)

Given the two facts above, the config-editing approach is a dead end for this goal.
The remaining avenues are *architectural observations*, recorded for completeness:

- **Firmware is the source of truth** for `/etc/passwd`, the `vtysh` binary, and any
  service-enable logic — but the current `firmware/extractions/` run only carved raw
  binwalk segments; **no root filesystem (squashfs/UBI) was unpacked**, so we can't
  yet read the real account setup or init scripts. Extracting the rootfs is the
  natural next research step.
- **The bootloader exposes a boot-command window** (above), the standard EcoNet
  mechanism — its capabilities on this unit are advertised but unverified.
- **`vtysh` remains a UID-0 CLI** (doc 11); what it exposes is still uncatalogued.

### Verified vs. unknown

| Established | Still unknown |
|-------------|---------------|
| `config.cfg` payload is encrypted; tool can't decode it | The key / the 2024 format details |
| Shells are firmware-baked, not config-driven | The exact init path that writes `/etc/passwd` |
| Boot log shows a boot-command prompt + serial-disabled-by-default | Whether command mode is gated, and what it allows |
| Firmware not yet unpacked to a rootfs | The real on-device `/etc/passwd`, dropbear/telnet init |

## Frontier

The most informative *next* step is **unpacking the firmware root filesystem**
(the binwalk run needs to resolve the squashfs/UBI, not just carve segments) — that
would let us read the actual account/service setup directly and settle, from
primary evidence, how administrative access is meant to be configured on this
model. That's a reverse-engineering task on my own firmware image, and it's where
the useful answers live.

Cross-refs: hardware identity [docs/01](01-device-overview.md), the OS layer
[docs/06](06-system-os.md), accounts & secrets [docs/11](11-security-accounts-secrets.md).
