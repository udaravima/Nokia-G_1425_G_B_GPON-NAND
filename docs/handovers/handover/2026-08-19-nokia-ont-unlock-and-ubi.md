# Session Handover — Nokia GPON ONT: config UBI extraction + active-bank unlock

**Date:** 2026-08-19 · **Prepared by:** Claude (prior session) · **For:** a fresh agent continuing this analysis.

This is a self-contained brief. You do **not** need the prior chat. Read §1–§3 for
grounding, then execute **Thread 1** (§4) and **Thread 2** (§5). §6 answers the QEMU
question. §7 lists gotchas, §8 the workflow rules you must obey.

---

## 0. Mission

We dumped the SPI-NAND of a **Nokia GPON fiber ONT** using the ESP32 dumper in this
repo. The dump is provably perfect. Two goals remain:

1. **Thread 1 — extract the `config` UBI volume.** This holds the device's unique
   runtime state (TR-069 params, WiFi/GPON credentials, calibration). It is
   **independent of all firmware crypto** — a pure UBI-geometry job. High value, low effort.
2. **Thread 2 — unlock the encrypted *active* rootfs.** The running firmware bank is
   encrypted; the decrypt key + algorithm are somewhere in this dump (the device
   self-boots). This is a MIPS reverse-engineering project.

---

## 0.5 — STATUS UPDATE (2026-08-19 evening): READ THIS FIRST

Substantial progress since §0 was written. Full proof for everything here is in the
companion **forensic report**: [2026-08-19-nokia-ont-forensic-report.md](2026-08-19-nokia-ont-forensic-report.md)
(esp. §5 eFUSE, §6 vendor-LZMA, §7 UBI, §10 live access). Corrections to older facts
below: the SoC is **EN7528 / en751221** (not EN7512); the running image is
`3FE49568HJLL88` build `HJL.L88p01`, product **Nokia G-1425G-A (`g1425ga`)**, GPON.

### What's now known
- **Thread 2 key-derivation = ANSWERED: case (c), eFUSE.** The second-stage bootloader
  (partition 0, file `0x10000+`) reads a **128-bit eFUSE** (`efuse_bit_0_31…96_127`,
  `read EFUSE fail`) + AES-128 (`File size not a multiple of 16`) + RSA + HMAC + SHA256
  to decrypt/verify the boot image. **The boot-chain key is fused in silicon — not in the
  dump.** So a pure-dump decrypt of the eFUSE-protected content is impossible.
- **`ENCRYPT=n` twist.** Live `show buildinfo` reports the running HJLL88 image as
  *plaintext+signed* (`SIGN=y ENCRYPT=n`), yet the dump's bank1 rootfs superblock is
  ciphertext. Reconciliation (inferred): the dump caught HJLL88 as an encrypted **staged
  OTA** before activation; it's since been decrypted+activated. The active rootfs the
  device runs is the plaintext twin of bank0 — so Thread 2 is largely *curiosity*.
- **Live device access achieved.** SSH is enabled for `administrator` (pw
  `AU121BM213CH38E5`) → drops into **GNU Zebra 0.95 vtysh** restricted CLI, NOT a shell.
  Cracked fleet-wide MD5-crypt creds (in `lab/crack/CRACKED.txt`, local): `admin`/`1234`,
  **`ONTUSER`/`SUGAR2A041`** (uid0 `/bin/sh`), `root`/`LA(ImvZx%8` (`/bin/false`).
  A full shell is gated by a **dynamic password**: enable-mode `shell` → `Password2:`
  (static fails). The generator is in `/usr/sbin/vtysh` — which did NOT extract.
- **Extraction reality (correction).** sasquatch did *not* really extract the rootfs —
  only **157 of 1923 files have content** (segfaults + vendor-LZMA). The files that
  matter (`/usr/sbin/vtysh`, `usr/etc/sec.key` 1321B, `usr/etc/ecc_pub.pem`,
  `sbin/upgrade_tool`, `lib/libupgrade*.so`, `lib/modules/crypto_k.ko`) are all **empty**.
- **UBI geometry (correction).** Real PEB = **131072 (128K)**, LEB = 126976 (not 256K).
  But the linear dump has UBI headers at a rigid **2-block (256K) pitch** — a 2-plane /
  block-paired layout — so only 32/64 config PEBs and 1/2 layout copies are linearly
  visible, and ubireader can't grid it. Data is present but needs de-interleaving.
- **Bootloader reversal (in progress).** `parts/bootloader.bin` = Block A (`0x0–0x8000`,
  DDR init) + Block B (`0x10000–0x1e000`, base **`0x80000000`**), both plaintext MIPS
  (mixed MIPS32/MIPS16, `jalx`). The `Press 'x'/'b'` handler is decoded at file `0x360`:
  it polls the UART ~11×/1s **before** serial is disabled; **`x` → serial firmware-download
  mode** → on success **`jalr 0x80080000`** (jumps into the received image); `b`/timeout →
  continue boot. The `disable serial by default` / `boot command mode` logic is in the
  **eFUSE-encrypted `boot.img`** (not in plaintext anywhere → unreachable from the dump).

### Re-ranked next steps (priority order)
1. **[TOP] Reverse the `x`-download auth.** Disassemble the MIPS16 subroutines
   `receive_image` (`0x80005f10`) and `process` (`0x80001ef0`) in `parts/blockB.bin`
   (carved: file `0x10000–0x1e000`, base `0x80000000`). **If the download jumps to the
   received image WITHOUT verifying a signature, that's arbitrary code execution in the
   bootbase context** — which can read the eFUSE, `sec.key`, decrypt bank1, or spawn a
   shell. This single result would collapse every remaining wall. Confirm/deny the check.
2. **Bench: reach the `x` prompt.** Serial input works in that window; the owner's `x`
   likely missed the ~1s timing — **flood `x` at reset**. A `received len=0x…` prompt means
   it's awaiting an upload → then the download protocol matters (see #1).
3. **Live shell without the download.** Enable telnet/dropbear (config keys
   `factory_telnet_enable`, `dropbear`) and log in as `ONTUSER`/`SUGAR2A041` (uid0
   `/bin/sh`); or find a `vtysh` escape / crack the `Password2` generator (needs #4).
   With any shell: `cat /usr/etc/sec.key`, `cp -r /configs` retires Thread 1 + the LZMA wall.
4. **Defeat the vendor LZMA1 (dump-side master unlock).** Reverse the bootloader's own
   LZMA decoder (plaintext MIPS in Block A/B) → extract `/usr/sbin/vtysh` (reverse
   `Password2`), `sec.key`, and both kernels at once.
5. **UBI config (Thread 1).** De-interleave the 2-plane UBI, or just read `/configs` off
   the live device once a shell exists (far cheaper).

---

## 1. The device (verified)

| Property | Value | How known |
|---|---|---|
| Product | Nokia **G-1425G-A** (`g1425ga`) GPON ONT | live `show buildinfo`; UPnP/OMCI strings; HDR2 `3FE49568…` |
| SoC | EcoNet/Airoha **EN7528 / en751221**, MIPS 1004Kc (4×900 MHz) | kernel log (`EcoNet EN7528 SOC`, `machine is econet,en751221`) |
| CPU arch | **MIPS32 rel2, little-endian (MIPSEL)**, uClibc | `file` on `rootfs_sasq/bin/ndk_dhcp6c` → `ELF 32-bit LSB, MIPS` |
| NAND | Micron MT29F2G01 (2 Gbit), 2048+128 B pages, 128 KB erase | datasheet + boot log |
| Flash mgmt | MediaTek layout v30, BMT bad-block mgmt (163-block pool) | boot log |
| Firmware | dual-image A/B: backup `3FE49568IJJL06`, **active `3FE49568HJLL88`** | HDR2 headers |

**The dump is perfect.** `target/nand_raw_dump_20260818_225044.bin` (285,212,672 B,
ECC-on, 0 bad blocks, stable MD5 across repeated dumps). The `data` partition reads
back as 99.4% clean `0xFF`, which is impossible to fabricate from a bad read — proof
the read is faithful to 116 MB+.

**Stripped image = your working file:** `target/clean_225044.bin` (268,435,456 B =
256 MiB, spare removed via `ecc_stripper.py`). Because **0 blocks were BMT-remapped,
logical == physical**: every offset in the boot-log partition table below applies
**directly** to this file (confirmed — the squashfs sits exactly at the HDR2-computed
offset, both kernels present, `data` reads clean `0xFF`).

### Partition map (offsets into `clean_225044.bin`)

| # | Name | Offset | Size | Content | Status |
|---|---|---|---|---|---|
| 0 | bootloader | `0x0000000` | 256K | secure-boot loader (RSA/HMAC/SHA256 + vendor-LZMA) | carved → `parts/bootloader.bin` |
| 1 | romfile | `0x0040000` | 256K | **blank (all 0xFF)** on this unit | — |
| 2 | image0 | `0x0080000` | 41.5M | HDR2 **backup** bank: kernel + squashfs @`0x4169D6` | **extracted** (`rootfs_sasq/`) |
| 3 | linux | `0x2a00000` | 41.5M | HDR2 **active** bank: kernel + **encrypted** rootfs @`0x2da11b0` | **locked — Thread 2** |
| 4 | config | `0x5380000` | 8M | **UBI** (runtime config) | **Thread 1** |
| 5 | log | `0x5b80000` | 12M | UBI (logs) | Thread 1 (same method) |
| 6 | extfs | `0x6780000` | 6M | labeled `extfs`, mostly empty | low priority |
| 7–11 | bosa/flag/flagback/ri/riback | `0x6d80000`+ | 256K ea | flags / calibration / GPON serials | `strings`/manual |
| 12 | data | `0x6ec0000` | 110M | UBI, ~99% erased (fresh) | low priority |

**HDR2 header layout** (both banks): `+0x00` "HDR2", `+0x08` payload length (u32 LE),
`+0x10` 14-char build ID, `+0x50` kernel size (u32 LE), `+0x54` rootfs size (u32 LE),
payload starts at `+0x118`. Rootfs offset = bank_start + `0x118` + kernel_size.

---

## 2. Artifacts already produced (`target/`, gitignored)

| Path | What |
|---|---|
| `clean_225044.bin` | 256 MiB stripped image — **the working file** |
| `parts/bootloader.bin` | partition 0 (raw MIPSEL blob) — Thread 2 radare2 input |
| `parts/bank0_rootfs.sq` | backup rootfs, plain SquashFS-LZMA (2829 inodes) |
| `parts/bank1_rootfs.bin` | active rootfs, **encrypted** (Thread 2 target) |
| `parts/config.ubi` | config UBI carve (8 MB) — Thread 1 target |
| `parts/data.ubi` | data UBI carve (110 MB, ~empty) |
| `parts/kernel1.lzma` | active kernel LZMA stream (vendor LZMA — see §5) |
| `rootfs_sasq/` | **backup rootfs extracted** (`sasquatch`): 2714/2829 inodes, 1923 files |
| `rootfs_extracted/` | mainline-`unsquashfs` attempt — only 7 files, ignore |

To re-carve anything: `python3 -c "d=open('clean_225044.bin','rb').read(); open('parts/X','wb').write(d[OFF:OFF+SIZE])"`.

---

## 3. Tooling

**Installed & used:** `binwalk` (rust/cargo, `~/.cargo/bin`), `sasquatch`, `unsquashfs`,
`ubireader_*`, `jefferson`, `radare2`/`r2`/`rabin2`, `7z`, host `objdump` (x86 only),
`python3`. Pytest venv at `lab/venv/bin/python`.

**NOT installed — install before the emulation work in §6:**
`qemu-user`/`qemu-system-mips*` (MIPSEL), Unicorn Engine (`pip install unicorn`),
`capstone` (`pip install capstone`). `binfmt_misc` mounted but no mips handler registered.
No `mips-linux-gnu-objdump`, no Ghidra. `fat.py`/`firmadyne` absent.

---

## 4. THREAD 1 — extract the config UBI  (crypto-independent, do first)

### Exact problem (diagnosed, not guessed)

`config.ubi` is **structurally intact**: every PEB has a valid big-endian `UBI#` EC
header (`version=1`, `vid_hdr_offset=2048`, `data_offset=4096`, one consistent
`img_seq=0x5960d122`), PEB size **256 KB** (0x40000; the device runs the NAND 2-plane,
so UBI's erase unit is 256 KB even though the raw NAND block is 128 KB).

Volume inventory (from parsing all 32 PEBs):

- `vol_id 0x00000000` → **24 LEBs = the config data (UBIFS)** ← what you want
- `vol_id 0x7FFFEFFF` → **only 1 LEB** (the internal layout/volume-table volume), at PEB 5

**ubireader fails** (`UBI Fatal: Less than 2 layout blocks found`) purely because UBI
normally keeps **two** redundant layout copies and this image has one. Passing
`-p 262144` alone does **not** fix it (tried). The data is fine; the tool is being strict.

### Fix options, in order of preference

**(a) Patch ubireader to accept a single layout block.** In the installed `ubireader`
package find the guard that raises "Less than 2 layout blocks" (grep the site-packages:
`grep -rn "layout blocks" $(python3 -c 'import ubireader,os;print(os.path.dirname(ubireader.__file__))')`).
Relax `>= 2` to `>= 1`, then:
`ubireader_extract_files -p 262144 -o config_out parts/config.ubi`.

**(b) Manual reassembly (no patching).** Collect the `vol_id 0` PEBs in `lnum` order,
take each PEB's data area (`bytes[data_offset=4096 : 0x40000]`), concatenate → a UBIFS
image, then extract it with `ubireader_extract_files -u` or mount it. Skeleton:

```python
import struct
d = open("target/clean_225044.bin","rb").read()
UBI, PEB = 0x5380000, 0x40000
leb = {}
for i in range(0x800000 // PEB):
    o = UBI + i*PEB
    if d[o:o+4] != b'UBI#': continue
    vo, = struct.unpack_from('>I', d, o+16)          # vid_hdr_offset (2048)
    do, = struct.unpack_from('>I', d, o+20)          # data_offset (4096)
    v = d[o+vo:o+vo+64]
    if v[:4] != b'UBI!': continue
    vol_id, = struct.unpack_from('>I', v, 8)
    lnum,   = struct.unpack_from('>I', v, 12)
    if vol_id == 0:
        leb[lnum] = d[o+do : o+PEB]                   # this LEB's data
img = b''.join(leb[k] for k in sorted(leb))
open("target/parts/config_vol0.ubifs","wb").write(img)
# then: ubireader_extract_files -u target/parts/config_vol0.ubifs -o config_out
```

**(c) Kernel UBI stack (most robust, if a/b stall).** Attach via `mtdram`/`nandsim` +
`ubiattach`. Caveat: the 256 KB-PEB-vs-128 KB-erase split needs the right `nandsim`
geometry; and stripping removed the OOB, so a synthetic OOB may be required. Prefer (a)/(b).

Repeat the chosen method on `log` (`0x5b80000`, 12M). `data` (`0x6ec0000`) is ~99% empty
— skip unless something interesting turns up.

**Deliverable:** a browsable `config_out/` tree. Grep it for credentials, GPON serial/SN,
TR-069 (`datamodel.xml`, `cwmp`), WiFi keys, and anything that could be a
**key-derivation source** for Thread 2 (see §5 concern (b)).

---

## 5. THREAD 2 — unlock the active-bank rootfs  (MIPS RE)

### The problem, proven

The active bank's rootfs at `0x2da11b0` is **encrypted**. Proof: parse it as a squashfs
superblock and every field is noise (`magic=1d8550b4`, `block_size=292715711`,
`version=44793.30344`), whereas the backup bank at `0x4169D6` parses cleanly
(`hsqs`, block 131072, comp 2, v4.0). Not a magic swap — the header is ciphertext.

### Why the unlock must be in this dump

The device boots with zero external input, so the algorithm **and** key are on-chip.
Boot chain: mask ROM → **bootloader** (partition 0) → 2nd-stage ("bootram"/"boot.img")
→ **kernel** → kernel decrypts + mounts the active rootfs. Bootloader strings prove a
secure-boot chain: `descrypt rsa key fail`, `HMAC check failed: wrong key, or file
corrupted`, `decrypt bootram image fail`, `bootram verify fail`, `SHA256`/`SHA224`,
`lzma decompress boot.img error` (they ship their **own** LZMA).

### KEY-DERIVATION CONCERN (the crux — from the project owner)

The active bank is a **field firmware update**, so the rootfs-decrypt key is one of:

- **(a) Embedded constant** in the kernel/loader. **Most likely** — the image is
  RSA-signed, so a baked-in symmetric key is safe from tampering. → fully recoverable from the dump.
- **(b) Derived from a per-device value** — serial number / MAC / GPON SN / hardware ID.
  Such values may live in `romfile` (`0x40000` — **blank on this unit**, note!), or in
  `config`/`bosa`/`ri`. → recoverable *iff* the source value is in the dump (Thread 1
  may surface it).
- **(c) Derived from an eFUSE/OTP register** the loader reads at runtime. → **NOT in the
  dump.** This is the one scenario that blocks a pure-dump solution; you'd need to read
  the fuse off the physical chip. **Determining (a) vs (b) vs (c) is the first RE task.**

### Attack order — get the KERNEL first

The kernel almost certainly contains the rootfs decrypt routine + key (signature-protected,
so safe to embed). The gating obstacle: the kernel is **vendor-modified LZMA** — python,
`xz -F lzma`, and `7z` each desync at *different* points (199 KB / 0 / 369 KB), the classic
vendor-LZMA fingerprint (same reason mainline `unsquashfs` failed but `sasquatch` worked on
bank0). Decompress it via, in order:

1. `binwalk -e parts/kernel1.lzma` and sasquatch's LZMA variants — cheapest.
2. **Emulate the device's own decompressor** (§6.3) — run the vendor-LZMA routine on the blob.
3. Reverse the vendor-LZMA in radare2 — last resort.

### radare2 recon plan (input: `parts/bootloader.bin`, raw MIPSEL blob)

```
r2 -a mips -b 32 -e cfg.bigendian=false parts/bootloader.bin
```

- **Base address:** MIPS reset vector is `0xBFC00000`; EN7512 boot SRAM appears around
  `0xBFA10000` (boot log prints `(0xBFA10114)=0x6`). Try rebasing (`omb`/`-m`) and see
  which makes branch/jump targets resolve.
- **Find the crypto by string xref:** seek the strings above, xref to their functions —
  that is the decrypt/verify code. Look for SHA-256 K-constants (`0x428a2f98…`), RSA
  bignum/modexp, and LZMA range-coder init (`0xFFFFFFFF`, ~0x300-entry prob model).
- **Answer the key question:** does the routine load a hardcoded key blob (in-dump, →(a)),
  read a value from a flash partition (→(b)), or read MMIO/eFUSE (→(c))? Trace the key
  pointer back to its source. **Report which before investing in full extraction.**
- The **2nd-stage loader** ("bootram") that the bootloader decrypts may be the piece that
  actually decrypts the rootfs; you may need to recover and analyse it too (it's the
  decrypted output of the bootloader step).

### Reality check / expectations

The backup bank (already extracted to `rootfs_sasq/`) is the **same product**, build
`IJJL06` vs the active `HJLL88` — adjacent revisions. Cracking the active bank mainly buys
the exact running build, not a different filesystem. If the key turns out to be eFUSE-derived
(c), stop and report — it's a hardware-read problem, not a dump problem.

---

## 6. QEMU / emulation — can we, and how?

Three distinct levels. Short answer: **not the whole box, but yes for the userland and yes
for the unlock routine — and the latter is the smart Thread-2 accelerator.**

### 6.1 Full-system boot of the real firmware — NO (not practically)

Stock QEMU has **no EN7512 machine model**. The firmware touches SoC-specific MMIO (NAND
controller, UART, GPON MAC/SerDes, timers, interrupt controller) that QEMU doesn't model,
so it dies on first unmodelled register. Making it boot means writing a QEMU board for
EN7512 (very large) or porting a patched kernel onto `-M malta` with matching drivers
(Frankenstein — no longer "the real device"). Not worth it here.

### 6.2 Userland emulation of the extracted rootfs — YES (standard technique)

Target is **MIPSEL / uClibc**, so use the little-endian tools. Two routes:

- **Per-binary, qemu-user + chroot** (fast): register `qemu-mipsel-static` in
  `binfmt_misc`, copy it into `rootfs_sasq/`, `chroot` in, run individual binaries
  (busybox, the web/CLI/config tools). GPON daemons fail (no hardware) but most userland
  runs — enough to read how config tools behave and dump default configs. This also
  supports Thread 1's goal.
- **Whole-image, FIRMADYNE / firmware-analysis-toolkit (FAT) / EMUX**: automated
  network-service emulation. Heavier setup; optional.

Install: `apt install qemu-user-static qemu-system-mips` (provides `qemu-mipsel-static`,
`qemu-system-mipsel`). Verify with `file rootfs_sasq/bin/busybox` (expect MIPSEL) then
`qemu-mipsel-static rootfs_sasq/bin/busybox` under chroot.

### 6.3 Emulate ONLY the unlock routine — YES, and this is the Thread-2 accelerator

Rather than reimplementing the vendor-LZMA + decrypt, **run the device's own routine** on
the encrypted blob using **Unicorn Engine** (QEMU's CPU core as a library) or qemu-user:

1. From the radare2 recon (§5), get the routine's address range, calling convention, and
   input/output pointers.
2. In Unicorn (`UC_ARCH_MIPS`, `UC_MODE_MIPS32 | UC_MODE_LITTLE_ENDIAN`): map the code +
   the input (encrypted rootfs / kernel blob), set `a0..a3`/`sp`, run to the return, read
   the decrypted/decompressed output from memory.
3. **Stub the hardware:** if the routine reads an eFUSE/MMIO for the key (concern (c)),
   hook that address and supply the value. If the key is embedded (a) or derived from
   in-dump data (b), it "just works" with no external input.

This defeats the vendor LZMA **and** the decryption without fully understanding either —
provided the key is in-dump. Install: `pip install unicorn capstone`.

**Arch settings cheat-sheet:** radare2 `-a mips -b 32 -e cfg.bigendian=false`; Unicorn
`UC_MODE_MIPS32|UC_MODE_LITTLE_ENDIAN`; qemu `qemu-mipsel[-static]`, `qemu-system-mipsel`.

---

## 7. Open questions / gotchas

- **Serial ECC log (last dump-integrity confirmation):** did the firmware print any
  `[ECC] UNCORRECTABLE page …` during the dump (v3.1.1 reports it)? Expected none — but
  ask the owner; it's the only direct confirmation left. Everything else says the dump is clean.
- **2-plane / 256 KB PEB:** the device runs the NAND 2-plane, so UBI PEB = 256 KB while the
  raw erase block is 128 KB. Doesn't affect linear carving (proven) but matters for any
  `nandsim` reconstruction in Thread 1(c).
- **`romfile` is blank** on this unit — if the key derives from a per-device value normally
  stored there, it's absent in this dump. Relevant to concern (b).
- **Backup ≈ active** (builds IJJL06 vs HJLL88) — set expectations for Thread 2's payoff.

---

## 8. Workflow constraints (must follow)

- **Never `git commit`/`git push` without explicit per-action permission** (owner's global
  rule). Do the work, then ask. Approval for one commit does not carry to the next.
- Scratch dir: `lab/`. Large carves live under `target/` (gitignored). Datasheets in
  `docs/datasheets/`.
- Project memory: `~/.claude/projects/-home-fire310w-Documents-Github-ESP32-SPI-Nand-Dumper/memory/`
  — see `target-device-forensics.md` and `nand-generalization-branch.md`.
- Explain findings plainly; separate **verified** from **assumed**; state units. The owner
  values a complete causal chain over a pile of facts.
```
