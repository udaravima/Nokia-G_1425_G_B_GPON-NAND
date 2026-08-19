# Forensic Analysis — Nokia GPON ONT NAND Dump

**Subject:** raw SPI-NAND image of a Nokia GPON fiber ONT, acquired with this repo's ESP32 dumper.
**Analyst:** Claude (agent session), 2026-08-19.
**Working files:** `target/nand_raw_dump_20260818_225044.bin` (raw), `target/clean_225044.bin` (spare-stripped), carves under `target/parts/`.

Each finding below is stated as **Claim → Method → Proof → Interpretation**, and every claim is tagged
**[VERIFIED]** (read directly from a tool output / the bytes) or **[INFERRED]** (reasoned from evidence,
not directly observed). The companion continuation brief is
`docs/handover/2026-08-19-nokia-ont-unlock-and-ubi.md`.

---

## 0. Verdict summary

| # | Finding | Confidence |
|---|---|---|
| 1 | The dump is byte-faithful (ECC-on, 0 bad blocks, reproducible MD5, erased regions read clean 0xFF) | **VERIFIED** |
| 2 | Device = Nokia GPON ONT, EcoNet **EN7528 / en751221**, MIPS 1004Kc, Linux 3.18.21 | **VERIFIED** |
| 3 | 13-partition MTD map; boot-log offsets apply directly to the stripped image (logical == physical) | **VERIFIED** |
| 4 | Dual A/B firmware: bank0 (`IJJL06`) is **plaintext** SquashFS; bank1/active (`HJLL88`) rootfs is **encrypted** | **VERIFIED** |
| 5 | Secure-boot key is rooted in a **128-bit eFUSE** value → **not in the dump** (AES-128 + RSA + HMAC + SHA256) | **VERIFIED (strings); INFERRED (exact key schedule)** |
| 6 | Master blocker = **vendor-modified LZMA1**; only **157 of 1923** rootfs files actually extracted | **VERIFIED** |
| 7 | UBI partitions can't be gridded from the linear dump (2-plane / block-interleave); data present but not laid out linearly | **VERIFIED (symptom); INFERRED (2-plane cause)** |

---

## 1. Acquisition integrity — the dump is faithful

**Claim [VERIFIED]:** the image is a complete, non-corrupt read of the chip.

**Method / Proof:**
- Sidecar `…225044.bin.meta.json`: `ecc_on=true`, `bad_page_count=0`, `truncated=false`; repeated dumps produced the **same MD5** (reported by owner).
- Raw size `285,212,672` B = `2048 blocks × 64 pages × 2176 B`. Stripped size `268,435,456` B = exactly `256 MiB` = `2048 × 64 × 2048`. Both match the MT29F2G01 geometry exactly.
- The `data` partition (110 MB) reads back **99.4 % `0xFF`**:
  ```
  data.ubi: 115343360 bytes, 0xFF=114619223 (99.4%), 0x00=701350 (0.6%)
  ```

**Interpretation:** a faulty read or a mis-aligned strip cannot *reconstruct* megabytes of pristine erased
flash. Clean 0xFF at 116 MB+ proves the read and the per-page spare-strip are faithful to that depth. This
matters because it lets us blame later anomalies (encrypted bank, UBI) on *content/format*, never on the dump.

---

## 2. Device identification

**Claim [VERIFIED]:** Nokia GPON fiber ONT on an EcoNet EN7528 SoC.

**Proof (device's own kernel boot log, supplied by owner):**
```
EcoNet EN7528 SOC prom init
+++++ nokia prom ! ... arch/mips/econet/prom.c
MIPS: machine is econet,en751221
CPU0 revision is: 0001992f (MIPS 1004Kc)          # 4 cores @ 900 MHz, 512 MB DDR3
Linux version 3.18.21 (buildmgr@AONTDH70) ... #7 SMP Wed Dec 25 23:52:59 CST 2024
sn: fc429d49                                       # per-device serial, printed by bootloader
EN7528 at Fri May 13 22:20:23 CST 2022 version 1.1 free bootbase
```
Cross-checks on the image itself:
- Firmware headers carry Nokia/Alcatel-Lucent material codes (`3FE49568…`, see §4).
- A rootfs binary: `rootfs_sasq/bin/ndk_dhcp6c: ELF 32-bit LSB executable, MIPS, MIPS32 rel2 … interpreter /lib/ld-uClibc.so.0`.

**Interpretation:** little-endian **MIPS32** (MIPSEL), uClibc. Correcting an earlier note: the SoC is
**EN7528/en751221**, not EN7512. This fixes the tool arch for all RE (`qemu-mipsel`, radare2
`-e cfg.bigendian=false`).

---

## 3. Partition map (verified against the boot log)

**Claim [VERIFIED]:** the boot-log MTD table describes this image, and its offsets apply **directly** to
`clean_225044.bin`.

**Method:** read the magic bytes at every boot-log offset in the stripped image.

**Proof:**
```
bootloader   @0x00000000  18 00 f0 0b ...   MIPS code            size 0x40000
romfile      @0x00040000  ff ff ff ff ...   erased (blank)       size 0x40000
image0       @0x00080000  48 44 52 32 ...   "HDR2" bank0         size 0x2980000 (41.5M)
linux        @0x02a00000  48 44 52 32 ...   "HDR2" bank1         size 0x2980000 (41.5M)
config       @0x05380000  55 42 49 23 ...   "UBI#"               size 0x800000  (8M)
log          @0x05b80000  55 42 49 23 ...   "UBI#"               size 0xc00000  (12M)
extfs        @0x06780000  65 78 74 66 73    "extfs"              size 0x600000  (6M)
data         @0x06ec0000  55 42 49 23 ...   "UBI#"               size 0x6e00000 (110M)
```
Plus five 256 KB flag/calibration partitions (`bosa/flag/flagback/ri/riback`) at `0x6d80000`+.
The `config` UBI's on-flash `image sequence number` is `0x5960D122`, which equals the kernel's reported
`image sequence number: 1499517218` for `config` — proving this log is from this dump.

**Interpretation:** because 0 blocks were BMT-remapped, logical == physical; the linear strip preserves offsets
(confirmed independently in §4 where the squashfs lands exactly at the header-computed offset).

---

## 4. Dual firmware banks — one plaintext, one encrypted

**Claim [VERIFIED]:** two A/B firmware banks; the backup (`image0`) rootfs is a normal SquashFS, the active
(`linux`) rootfs is encrypted.

### 4a. The HDR2 header format (reverse-engineered from the two banks)
```
HDR2 image0 (bank0) @0x080000:  48 44 52 32 ... "3FE49568IJJL06" ...
  +0x50: be 68 39 00  61 65 90 01   → kernel_size=0x003968be, rootfs_size=0x01906561
HDR2 linux  (bank1) @0x2a00000:  48 44 52 32 ... "3FE49568HJLL88" ...
  +0x50: 98 10 3a 00  9d f7 a5 01   → kernel_size=0x003a1098, rootfs_size=0x01a5f79d
```
Layout: `+0x00` "HDR2", `+0x10` 14-char build ID, `+0x50` kernel size, `+0x54` rootfs size, payload at `+0x118`.
**Proof the fields are correct:** bank0 `rootfs_size = 0x01906561 = 26,240,353`, which exactly equals the
SquashFS `bytes_used` (§4b). And `rootfs offset = bank_start + 0x118 + kernel_size`:
bank0 → `0x80000+0x118+0x3968be = 0x4169D6` (lands on `hsqs`); bank1 → `0x2a00000+0x118+0x3a1098 = 0x2da11b0`
(equals the boot-log `rootfs` MTD start). Both independently confirmed.

### 4b. Superblock comparison — the proof of encryption
Parsing the SquashFS superblock at each bank's rootfs offset:
```
BANK0 backup @0x004169d6
  magic=hsqs  inode_count=2829   block_size=131072  compression=2(lzma)  block_log=17  version=4.0
  raw: 68 73 71 73 0d 0b 00 00 7b 4a ad 63 00 00 02 00 28 01 00 00 02 00 11 00 ...
BANK1 ACTIVE @0x02da11b0
  magic=1d8550b4  inode_count=1001021513  block_size=292715711  version=44793.30344
  raw: 1d 85 50 b4 49 60 aa 3b 79 55 51 ea bf 7c 72 11 ed 40 eb d9 1a f5 12 18 ...
```
`file(1)` agrees: bank0 = `Squashfs filesystem, little endian, version 4.0, lzma compressed, 26240353 bytes,
2829 inodes, blocksize: 131072, created: Thu Dec 29 08:06:19 2022`; bank1 = `data`.

**Interpretation [VERIFIED]:** every bank1 superblock field is noise — not a swapped magic (a magic swap would
leave `block_size=131072`, `version=4.0` intact). The whole 96-byte superblock is **ciphertext**. Entropy
corroborates: bank0 squashfs region 8.000 bits/byte (clean compressed), bank1 rootfs region 6.450 (diluted by
trailing padding; not the clean-compressed 8.0 nor plaintext). The running kernel mounts root as a **plain**
squashfs (`VFS: Mounted root (squashfs) readonly on 31:4`), so bank1 is the **inactive/OTA-staged** image, and
the plaintext bank0 is the twin of what actually runs.

---

## 5. Secure-boot chain & key derivation — the eFUSE verdict

**Claim [VERIFIED from strings; INFERRED for the exact schedule]:** the boot-image decryption is rooted in a
**128-bit eFUSE** value burned into the SoC — therefore **not present in the NAND dump**.

### 5a. Partition 0 is two images
Per-8 KB entropy + magic scan of `parts/bootloader.bin` (262144 B):
```
0x00000–0x08000  code (ent 5–6, starts 18 00 f0 0b, MIPS)   ← Block A, 1st-stage "bootbase"
0x08000–0x10000  zeros
0x10000–0x1e000  ent 6.3–7.0                                 ← Block B, 2nd-stage (own load base)
0x1e000–0x40000  zeros / erased
```
The first recon pass established that the crypto strings are **not referenced from Block A** — they belong to
Block B, which has its own load base.

### 5b. The crypto message table (Block B rodata, 0x1bff9–0x1cc8c) — the proof
```
0x1c05c  Load sheader magic error.          # secure header on the boot image
0x1c154  lzma decompress boot.img error, overlap bootram image.
0x1c1b9  decrypt bootram image fail.        # it DECRYPTS the boot image
0x1c1ec  bootram verify fail.
0x1cba0  SHA256                             # + SHA224
0x1cbb0  File too short to be encrypted.
0x1cbd4  File size not a multiple of 16.    # 16-byte blocks  → AES
0x1cc10  HMAC check failed: wrong key, or file corrupted   # HMAC integrity
0x1cc40  efuse_bit_0_31:                    # ── reads 128 bits of eFUSE ──
0x1cc51  efuse_bit_32_63:
0x1cc65  efuse_bit_64_95:
0x1cc79  efuse_bit_96_127:
0x1cc8c  descrypt rsa key fail.             # RSA unwraps/verifies a key
0x1cca4  read EFUSE fail
```
And Block A independently reads fuses (0x07164): `Error: EFUSE 1 read timeout`, `Error: EFUSE 2 read timeout`.

**Interpretation:** the chain is *sheader → LZMA-inflate boot.img into "bootram" → decrypt → HMAC-verify*, using
**AES** (16-byte block constraint), **RSA** (key unwrap/verify), **SHA-256**, and a **128-bit eFUSE** read
(4 × 32-bit words = an AES-128 key's worth). 128 fused bits feeding an AES/HMAC path is the textbook OTP-rooted
secure-boot key. **The key lives in silicon OTP, not in flash** — so the boot-chain crypto cannot be reproduced
from the dump alone; it needs the physical chip's eFUSE. *Verified:* the eFUSE read and the primitives (strings).
*Inferred:* the precise combination (whether eFUSE is the AES key directly or an HMAC/KDF seed) — that needs the
Block B disassembly to confirm, which is the open RE step.

> Caveat worth carrying: this eFUSE chain protects the **boot image**. Whether the *encrypted bank1 rootfs* uses
> the same eFUSE key or a userland key is unresolved — see §6, `sec.key`.

---

## 6. The master blocker — vendor-modified LZMA1

**Claim [VERIFIED]:** the SquashFS uses a vendor LZMA1 variant that mainline tools can't decode and sasquatch
only partially decodes; **only 157 of 1923 rootfs files were actually extracted.**

**Proof:**
- Extraction reality (correcting an earlier over-claim that "sasquatch got the rootfs"):
  ```
  total files: 1923   empty(0-byte): 1766      # only 157 files have real content
  ```
  sasquatch also exits 139 (segfault) mid-run.
- Mainline unsquashfs on the exact files we need:
  ```
  lzma uncompress failed with error code 0
  FATAL ERROR: writer: failed to read/uncompress file .../libupgradeimpl.so
  ```
- The SquashFS superblock declares `compression=2` = **LZMA1** (the legacy plugin mainline squashfs-tools
  dropped). This is the same variant that defeats the **kernels**: python/`xz`/`7z` desync at *different* points
  (198,994 B / 0 B / 369,441 B) on the same stream — the fingerprint of a non-standard LZMA, not corruption.
- The files that would answer the rootfs-encryption question exist but did **not** extract (real sizes from the
  SquashFS inode listing):
  ```
  usr/etc/sec.key            1321 B   ← a real key file, inside the rootfs
  usr/etc/ecc_pub.pem         134 B   ← ECC public key (signature verify)
  sbin/upgrade_tool          4880 B
  lib/libupgrade.so         44048 B
  lib/libupgradeimpl.so     61756 B
  lib/modules/crypto_k.ko   92728 B   ← kernel crypto module
  ```

**Interpretation:** this single obstacle sits under *everything* still locked — 92 % of rootfs binaries, both
kernels, and the bank1 data blocks. The tantalizing lead is `sec.key` (1321 B): if userland OTA unwraps the
image with an in-rootfs key rather than the eFUSE, the key *is* in the dump — but we can't read it until the
vendor LZMA is defeated. The one decoder that provably handles this variant is **uncompressed and in hand**:
the bootloader's own LZMA routine (plain MIPS in partition-0 Block A/B). Reversing it is the highest-leverage
next move.

---

## 7. UBI partitions — the 2-plane interleave wall

**Claim [VERIFIED symptom; INFERRED cause]:** the `config`/`log`/`data` UBI volumes are intact on the chip but
cannot be reconstructed from the linear dump with standard tools; the data is present, the PEB grid is not.

**Ground-truth geometry (kernel boot log):**
```
UBI: attached mtd5 (name "config", size 8 MiB)
UBI: PEB size: 131072 bytes (128 KiB), LEB size: 126976 bytes
UBI: VID header offset: 2048, data offset: 4096, sub-page 2048
UBI: good PEBs: 64 ... image sequence number: 1499517218 (=0x5960D122)
UBIFS: mounted ... name "configs" ... FS size 6094848 bytes (48 LEBs)
```

**Proof of the mismatch:**
- In the **raw** dump the UBI EC headers are exactly `278528` B apart = `128 raw pages` = **2 erase blocks**,
  and the VID header lands at `EC+2176` (start of the next *physical* page), which also proves the spare-strip
  is correct:
  ```
  raw 0x058b8000  (EC hdr, vid_off=2048 data_off=4096)
  raw 0x058fc000  spacing 278528 (0x44000) = 128 raw pages
  bytes at EC+2048 = ff ff ff ff        (main-area gap)
  bytes at EC+2176 = 55 42 49 21 = "UBI!" (VID header, next physical page)
  ```
- In the **stripped** image, UBI# markers are therefore every **256 KB**, so a 128 KB-PEB scan finds only
  **32 of the 64** config PEBs; `vol_id 0` shows **24 LEBs** (should be 48), the layout volume shows **1** copy
  (should be 2), and UBIFS **LEB 0/1** (superblock/master) are absent from the linearly-visible PEBs. Result:
  ```
  ubireader_extract_files -p 131072 …  →  UBI Fatal: Less than 2 layout blocks found.
  ```
  (Same with `-p 262144`.) The rigidly-regular every-other-block pattern is not wear-leveling (which scatters
  randomly).

**Interpretation:** the UBI regions are laid out with a **2-plane / block-paired** scheme, so a plain linear NAND
read interleaves the PEBs; the lower single-plane partitions (squashfs, kernels) read linearly and come out fine.
*Verified:* the 2-block-regular spacing, the correct strip, 32/64 PEBs, the tool failure. *Inferred:* that the
exact cause is 2-plane pairing (symptom is certain; the precise physical map isn't reversed). The config data is
all in the dump but needs de-interleaving per the EN7528 plane map — or, far cheaper, copy `/usr/cfg` off the
**live** device, which mounts `configs`/`logs`/`datas` without trouble.

---

## 8. Reproduction

All commands run from `target/`. Tools used: `binwalk` (rust), `sasquatch`, `unsquashfs`, `ubireader_*`,
`radare2`/`rabin2`, `7z`, `python3`. Not installed (needed for the deeper RE): `qemu-user`(MIPSEL), `unicorn`,
`capstone`, a MIPS `objdump`, Ghidra.

```bash
# 1. strip spare (raw 2176B/page → 2048B/page)
python3 ../ecc_stripper.py nand_raw_dump_20260818_225044.bin clean_225044.bin \
        --meta nand_raw_dump_20260818_225044.bin.meta.json

# 2. identify banks / superblocks (§4)  — parse SquashFS super at the HDR2-computed offsets
# 3. plaintext bank0 rootfs (partial — vendor LZMA, §6)
sasquatch -d rootfs_sasq parts/bank0_rootfs.sq      # exits 139; ~157/1923 files have content

# 4. bootloader crypto strings (§5)
python3 - <<'PY'
import re; d=open("parts/bootloader.bin","rb").read()
for m in re.finditer(rb'[\x20-\x7e]{4,}', d[0x1b800:0x1cd80]): print(hex(0x1b800+m.start()), m.group().decode())
PY

# 5. UBI geometry check (§7) — raw UBI# spacing proves 2-block pitch + strip correctness
# 6. bootloader disassembly (next step)
r2 -a mips -b 32 -e cfg.bigendian=false parts/bootloader.bin   # focus Block B @0x10000
```

---

## 9. Open questions / next steps

1. **Reverse the vendor LZMA1** from the bootloader's decoder (Block A/B, uncompressed MIPS). Unlocks: rootfs
   binaries + `sec.key`, both kernels, and bank1's data blocks — the master unlock.
2. **Confirm the eFUSE key schedule** by disassembling Block B (does the 128-bit fuse feed AES directly, or an
   HMAC/KDF?). Decides definitively whether *any* dump-only decryption is possible.
3. **Resolve bank1's encryption owner:** eFUSE (chip-bound) vs `sec.key` (in-dump). Requires #1 to read `sec.key`
   and #2 to read the boot path.
4. **UBI config:** either de-interleave per the EN7528 2-plane map, or pull `/usr/cfg` off the live device.
5. **eFUSE readout:** the bootloader *prints* `efuse_bit_*` — if that debug path is reachable on the serial
   console, the 128-bit fuse might be recoverable without chip decapping. Unverified; worth a console check.

---

## 10. Live-device access & the shell gate (added 2026-08-19)

> **SENSITIVE — do not publish.** This section contains working credentials for a Nokia
> ONT engineering account family that is fleet-wide, not unique to this unit.

### 10a. Recovered accounts (all VERIFIED — hashes reproduced from salt+password)
The `/etc/shadow`, `/etc/passwd`, and `/etc/usertty` hashes are baked into the read-only
factory rootfs (bank0, Dec 2022) — i.e. **fleet-wide defaults**, not per-unit. All are weak
`$1$` MD5-crypt:

| account | source | uid / shell | password | notes |
|---|---|---|---|---|
| `administrator` | live device | — → **SSH** | `AU121BM213CH38E5` | SSH login → restricted CLI |
| `admin` | `/etc/usertty` | 0 / `/bin/sh` | `1234` | (empty-salt hash) |
| **`ONTUSER`** | `/etc/passwd` | 0 / **`/bin/sh`** | `SUGAR2A041` | Nokia `SUGAR2A###` engineering family |
| `root` | `/etc/passwd` | 0 / `/bin/false` | `LA(ImvZx%8` | no shell by design |
| `appService` | `/etc/passwd` | 1100 / `/bin/false` | `!` locked | — |

`administrator` is the account SSH is enabled for (per the device config); it lands in the
restricted CLI, **not** a shell.

### 10b. The restricted CLI (VERIFIED live)
SSH as `administrator` drops into **GNU Zebra 0.95 vtysh** repurposed as "User CLI (version
0.95)" (`Zebra 0.95 (mips-linux)`, `/usr/sbin/vtysh -c` from `/etc/inittab`). Top-level +
enable-mode (`en`) commands: `configure, disable, exit, help, list, logout, nslookup, ntp,
ping, shell, show, tftp, traceroute`. `tftp` only exports logs (syslog/omci/voice); there is
no arbitrary file-read command. `show buildinfo` (enable mode) reports device identity:
```
ONT_TYPE=g1425ga           # Nokia G-1425G-A
PON_MODE=GPON
SOFTWAREVERSION=HJL.L88p01
IMAGEVERSION=3FE49568HJLL88 # == bank1 material code (the "active" bank)
BUILDDATE=20241225_2348      # matches the kernel build stamp
COPYRIGHT=NOKIA   SIGN=y   SIGNER=NCG   ENCRYPT=n   VOIP=sip   SDK_VERSION=4.0
```

### 10c. The shell gate — a *dynamic* password (`Password2:`)  [OPEN]
In enable mode, `shell` = "start shell, need to input the dynamic password" → prompts
`Password2:` → a static password returns `passwd invalid!`. So a full shell is gated by a
**dynamic/challenge password**, not a fixed one. The generator lives in the CLI binary
(`/usr/sbin/vtysh` or a helper lib) — which is one of the files that **did not extract** from
the dump (vendor-LZMA, §6). Two ways past it:
1. **Bootloader `init=/bin/sh`** — bypasses inittab/vtysh **and** the `Password2` gate
   entirely. The boot log shows a "Press any key … boot command mode" window *before*
   "disable serial by default", so a serial boot-interrupt may reach it. Cleanest if serial
   is reachable; `init=` doesn't break the kernel signature. **Preferred.**
2. **Reverse the `Password2` algorithm** from the CLI binary — requires defeating the vendor
   LZMA (§6) to extract `/usr/sbin/vtysh`, then RE the dynamic-password routine. Loops back to
   the master blocker (which also yields `sec.key` and the OTA tooling).

### 10d. Discrepancy: `ENCRYPT=n` vs the dump's encrypted bank1  [OPEN, reconciled by inference]
`buildinfo` reports the running image (`3FE49568HJLL88` = bank1) as **`ENCRYPT=n`** (plaintext,
`SIGN=y`), yet §4 found bank1's rootfs superblock at `0x2da11b0` to be ciphertext.
**Reconciliation [INFERRED]:** the dump (2026-08-18) most likely caught HJLL88 as an encrypted
*staged OTA* image before activation; the device has since decrypted and activated it, so it now
runs plain — consistent with `ENCRYPT=n` describing the *activated* build (the on-flash
encryption being a staging/transport layer, not a build property). This means the §5 eFUSE
finding governs the *boot image*; whether the staged rootfs used the eFUSE key or `sec.key`
remains open and still hinges on extracting the OTA tooling (§6).

### 10e. Updated next-step ranking
1. **Serial `init=/bin/sh`** — most direct path to a root shell; sidesteps the `Password2` gate.
2. **Defeat the vendor LZMA (§6)** — the master unlock: yields `/usr/sbin/vtysh` (→ reverse
   `Password2`), `sec.key`, and the OTA binaries in one move.
3. With any shell: `cat /usr/etc/sec.key`, `cp -r /configs` — retires the vendor-LZMA and
   2-plane-UBI problems instantly.

---

*Prepared as a point-in-time record. Offsets/sizes are from the files named above; re-verify against the current
image before relying on any single value.*
