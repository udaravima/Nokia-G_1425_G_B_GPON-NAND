# 13 · How We Cracked Open the Flash Chip — Explained Simply

This doc is the story of how we took a **raw copy of the router's memory chip** and
turned it into readable files — programs, settings, the whole operating system. It's
written to be understood with **zero background**, then it quietly gets more technical
so you can actually *reproduce* every step. If you only read one doc about the
firmware work, read this one.

The big picture in one sentence: **we photocopied the router's brain, figured out how
it was organised, and unpacked the parts that weren't locked or rotted.**

---

## The one analogy that makes everything click

Imagine the router's memory chip is a **giant notebook**.

- The notebook has **131,072 pages**.
- Each page holds **2048 letters** of actual writing…
- …plus a little **128-letter margin** where the notebook itself scribbles
  *spell-check notes* — a secret code it uses to catch and fix its own typos.
- The pages are grouped into **labelled sections** (like tabs in a binder): one tab
  for "how to switch on," two tabs for "the operating system," one for "settings,"
  one for "logs," and so on.
- Most of the writing is in a **compressed shorthand** (to save space) and some of it
  is **written in secret code** (locked).

Every hard part we hit maps onto this notebook. Keep it in your head.

| Notebook word | Real word | What it really is |
|---|---|---|
| The notebook | **NAND flash** | The chip that remembers everything when the power is off |
| A page | **page** (2048 bytes) | The smallest chunk the chip reads/writes at once |
| The margin | **OOB / spare area** (128 bytes) | Where the chip keeps its error-correcting code |
| Spell-check notes | **ECC** (Error-Correcting Code) | Math that repairs a few flipped bits automatically |
| A binder tab | **partition** | A named region of the chip with one job |
| Shorthand | **compression** (LZMA) | Squeezing data smaller so more fits |
| A sealed box of files | **squashfs** | A read-only filesystem — all the router's files in one lump |

---

## Step 1 — Get a photocopy (the "dump")

You can't study the chip while it's running, so first you make a **dump**: a
byte-for-byte copy of everything on it, saved as one big file. We were handed that
file (the chip was read out over its SPI wires — the low-level electrical bus the
chip talks on).

Here's the first clever trick, and it needs no tools — **just the file's size tells
you how it was copied.** There are two ways to photocopy the notebook:

- **With the margins** → each page is 2048 + 128 = **2176 letters**.
- **Without the margins** → each page is just **2048 letters**.

We had two dumps. Divide their sizes and the geometry falls out:

```
285,212,672 bytes ÷ 2176  = 131,072 pages exactly   → this copy KEPT the margins
268,435,456 bytes ÷ 2048  = 131,072 pages exactly   → this copy DROPPED the margins
```

Both are the same 131,072-page notebook — one just includes the spell-check margins
and one doesn't. (We proved it wasn't some *other* page size like 4096+256 by
cross-checking against the chip's datasheet: the Micron **MT29F2G01** is a 2048-byte-page
part. More on why the margins matter in Step 6.)

> **Why "256 MB" and "272 MB" for the same chip?** 272 MB = 256 MB of real writing +
> 16 MB of margins. The margins are 1/16th of the data (128 is 1/16th of 2048). Same
> notebook, different photocopy setting.

---

## Step 2 — Find the map (the partition table)

Before cutting the notebook into sections, you need the map: **where does each tab
start and stop?** We didn't guess — we let the router *tell* us. Every time it boots,
the Linux kernel prints its partition map to a crash-log buffer
(`tmp/logs/pstore/console-ramoops-0`). That's the router's own diary, and it says:

```
Creating 13 MTD partitions on "EN7512-SPI_NAND":
0x000000000000-0x000000040000 : "bootloader"   ← how to switch on
0x000000080000-0x000002a00000 : "image0"       ← Operating System, copy A
0x000002a00000-0x000005380000 : "linux"        ← Operating System, copy B
0x000005380000-0x000005b80000 : "config"       ← your settings
0x000005b80000-0x000006780000 : "log"          ← diagnostic logs
0x000006ec0000-0x00000dcc0000 : "data"         ← app/user data
   … and 7 smaller ones (flags, calibration, identity)
```

Drawn as a strip of the whole 256 MB chip (addresses in hex):

```
0x0        0x80000            0x2a00000          0x5380000  0x5b80000   0x6ec0000        0xdcc0000
|bootloader| image0 (BANK A)  | linux (BANK B)   | config   | log |...| data            | (reserved)
| romfile  | kernel+squashfs  | kernel+???       | UBI      | UBI |   | UBI (110 MB)    |
```

**MTD** just means "Memory Technology Device" — Linux's generic name for a raw flash
chip. **UBI** is a smarter layer for the writable tabs (config/log/data) that spreads
writes around so no single spot wears out; the read-only OS tabs don't need it.

The single most important thing on this map: **there are two operating-system copies,
image0 and linux.** Routers keep a spare so a failed update can't brick them. Which
one is "live" becomes the big plot twist in Step 7.

---

## Step 3 — Cut off the margins (strip the OOB)

To read the actual writing, we throw away the spell-check margins and keep only the
2048 real letters from each page. That turns the 272 MB "with margins" copy into a
clean 256 MB copy — the pure data. (Our newer dump arrived already stripped.)

The whole program is four lines of idea:

```python
for each page:                 # walk all 131,072 pages
    data  = page[:2048]        # keep the writing
    spare = page[2048:2176]    # drop the margin
    write(data)
```

That's [`workdir/nand_strip.py`](../workdir/nand_strip.py). While stripping, it also
peeked at the margins and noticed **92,818 pages had blank margins** (unused space)
and **38,254 had scribbles** (real ECC) — exactly what a healthy, half-full chip
should look like. First sign the dump was legit.

---

## Step 4 — Find the sealed box of files (squashfs)

Now we hunt for the **filesystem** — the sealed box holding every program and config
file. Filesystems announce themselves with **magic bytes**: a short fixed label at the
very start, like a logo on a cereal box. Squashfs's logo is the four letters `hsqs`.

We scanned the whole 256 MB for that logo. It appeared **exactly once**, at address
`0x4169d6` — inside **bank A (image0)**. We also found the two OS copies each start
with an `HDR2` header (the router's firmware-image wrapper) exactly where the map
said. Physical layout matched the map perfectly — proof the strip in Step 3 was
aligned correctly, with no drift.

Reading the box's label (`unsquashfs -s`) told us what's inside before opening it:

```
Found a valid SQUASHFS 4.0 superblock
Compression: lzma        ← the shorthand it's written in
Block size:  131072      ← unpacked in 128 KB chunks
Inodes:      2829        ← 2829 files, folders, and links
Created:     2022-12-29  ← the firmware's build date
```

An **inode** is just "one entry" — a file, a folder, or a shortcut. 2829 of them,
about 25 MB unpacked. That's the router's entire program set, sitting right there.

---

## Step 5 — Open the box (and why the normal tool refused)

Opening a squashfs is normally one command: `unsquashfs`. We ran it and it **failed**
— `lzma uncompress failed`. Here's the trap, and it's a good one:

The router compresses with **LZMA1**, an *old* flavour of the shorthand from ~2009.
Modern `unsquashfs` speaks the *new* flavour (XZ) and quietly dropped support for the
old one. So the standard tool literally can't read this vendor's compression, even
though nothing is wrong with the data.

The fix is a specialised tool called **`sasquatch`** — a patched unpacker built
exactly for these old vendor firmwares. With it, the box opened:

```
files: 1923   symlinks: 474   dirs: 317   →  2714 of 2829 entries recovered
```

**2714 out of 2829.** The router's real `/etc`, `/bin`, `/lib`, init scripts, web UI —
all extracted to [`workdir/rootfs_bankA/`](../workdir/rootfs_bankA/). But 115 entries
refused to come out. That's the next mystery.

---

## Step 6 — The 115 broken files: is it *us*, or is it *the chip*?

115 files failed to unpack. Before blaming anything, we had to answer one question:
**is our tool too dumb, or is the data genuinely damaged?** This is the most important
detective step in the whole project, so here's the reasoning spelled out.

**Test 1 — ask three different experts.** We tried *three* independent unpackers:
mainline `unsquashfs`, `sasquatch`, and `7-Zip`. Each has its own, separately-written
code for the old shorthand. **All three failed on the exact same 115 files.**

> If it were a *tool* problem, at least one of three independent tools would have
> succeeded — they don't share code. Three experts giving the identical wrong answer
> means the problem isn't the experts. It's the material.

**Test 2 — compare two photocopies.** We had two dumps taken at different times. We
compared the filesystem region byte-for-byte. **Identical.** So it's not a bad copy
either — the chip genuinely, stably holds these exact bytes.

**Conclusion: those 115 files are genuinely damaged on the chip itself.** The most
likely cause is **bit-rot** — and here's where the spell-check margins (Step 1) come
back. Flash chips store bits as tiny trapped electrical charges that slowly *leak*
over years. The chip's ECC (those margin scribbles) repairs a *few* leaked bits per
page automatically. But if a page loses *more* bits than ECC can fix, it's gone —
undecodable. Bank A is the **backup** copy that never gets rewritten, so its charges
sat undisturbed the longest and drifted furthest. Scattered dead files in an old,
untouched backup is exactly the fingerprint of retention bit-rot.

*(Honest limit: I can't 100% *prove* it's bit-rot vs. some other damage, because
proving it would need the margin/ECC bytes, and the stripped dump threw those away.
But every clue points this way.)*

---

## Step 7 — The plot twist: the copy we can read isn't the one it's running

Routers keep two OS copies so a broken update can fall back to the good one. So: **which
copy is actually live right now?** We read the router's diaries again
(`tmp/logs/messages`) and counted mentions of each build's ID:

```
3FE49568HJLL88  (the "linux" tab, bank B) — 189 mentions, tagged as the running system
3FE49568IJJL06  (the "image0" tab, bank A) —   8 mentions
```

Plus a tiny counter file, `zeroman/imageCounter`, holds `2` — pointing at slot 2.
Verdict: **bank B (linux) is the live one. Bank A (image0) is the idle backup.**

Now the irony lands. Look at what each copy contains:

| | Bank A — `image0` (backup) | Bank B — `linux` (LIVE) |
|---|---|---|
| Rootfs | **Plain `squashfs` — readable** ✅ | **No `squashfs` at all** — one big scrambled-looking blob ❌ |
| Build | IJJL06 (older) | HJLL88 (newer) |

**The copy we could open is the old spare. The copy the router actually runs can't be
opened — there's no `hsqs` logo anywhere in its whole 41 MB tab.** And Step 6's
byte-compare proved that blob is *stable real content*, not a bad read.

The best explanation: the newer firmware (HJLL88) stores its filesystem in a
**protected/encrypted form** that only the bootloader unlocks at power-on, while the
older factory image (IJJL06) left in the backup slot is still plain and open. *(I
can't name the exact lock from the bytes alone — that's an open question. One small
clue argues it's not simple encryption: its randomness score, 7.92, is actually a hair
*lower* than the plain squashfs's 7.96, and true encryption usually scores higher. See
the "randomness ruler" below.)*

This is **why pulling a live partition off the running box is the real path forward** —
the running system has already unlocked bank B in memory, so its files are readable
*there* even though they're locked *on the chip*.

---

## Step 8 — The other drawers, and a ruler for randomness

**The writable tabs (config / log / data).** These change every second the router
runs. We proved it: comparing the two dumps, **every single difference — 920 chunks —
landed in config, log, or data.** The OS tabs were frozen-identical; only the live
data churned. That's a beautiful confirmation the firmware is stable and our dumps are
trustworthy. These tabs use **UBI/UBIFS** (the wear-levelling filesystem); we carved
them out, though `config` duplicates the settings backup we already have.

**The identity tabs.** Two little tabs hold the router's "birth certificate":
- `ri` → the serial/model string: `ALCL…G-1425G-220601`
- `bosa` → the **optical transceiver** calibration (`UProtron PA221355…`) — the laser
  module that talks light down the fiber.

**The "randomness ruler" (entropy).** How do you tell *locked/compressed* data from
*plain* data without opening it? Measure how **random** the bytes look, on a scale of
0 to 8 (this is **Shannon entropy**, bits per byte):

- **~0** = boring and repetitive (e.g. a page of all zeros — an erased area).
- **~7.9** = compressed shorthand (squashfs, kernels).
- **~8.0** = indistinguishable from pure noise (strong encryption, or the chip's ECC
  margins).

It's like shaking a box: a written sentence rattles predictably, a box of loose
scrabble tiles is pure chaos. We used this ruler constantly — e.g. to confirm the OS
regions were compressed data (~7.9) and to notice bank B's blob doesn't score high
enough to look like textbook encryption.

---

## The toolbox — everything we used and why

| Tool | The job it did |
|---|---|
| **file size ÷ page size** | Figured out the notebook's geometry (with/without margins) — no tool needed |
| `pstore` boot log | Handed us the partition map, straight from the router |
| [`nand_strip.py`](../workdir/nand_strip.py) | Cut the ECC margins off every page → clean 256 MB image |
| `hsqs` / `HDR2` / `UBI#` magic scan | Located filesystems and headers by their logos |
| `unsquashfs -s` | Read the filesystem's label (compression, file count, build date) |
| **`sasquatch`** | Opened the old-LZMA squashfs the normal tool couldn't (2714 files out) |
| `unsquashfs` + `7-Zip` | The two *other* independent unpackers — used to prove the 115 failures were data, not tooling |
| [`cmp_dumps.py`](../workdir/cmp_dumps.py) | Byte-compared the two dumps → proved banks are frozen, only live data changes |
| entropy measurement | The "randomness ruler" for spotting compressed vs. locked regions |
| `ubireader` | Inspected/carved the UBI writable volumes |
| `dd` | Sliced individual partitions out of the big image by offset |

---

## Where we ended up

- ✅ **A working rootfs to study:** [`workdir/rootfs_bankA/`](../workdir/rootfs_bankA/)
  — 2714 files from the backup image (IJJL06). Good enough for the account/init/service
  reverse-engineering, minus 115 rotted files.
- ✅ **A complete map** of all 13 partitions and what each holds.
- ✅ **Proof the dump is trustworthy** (banks identical across two independent copies).
- ⛔ **The *live* rootfs (HJLL88) can't be unpacked from the chip** — it's locked on
  flash. → the reason the next step is to pull it off the **running device**.
- ⚠️ **115 backup files are bit-rotted** — a re-dump won't recover them; only the live
  box (or an ECC-capable re-read) can.

## What's next (the frontier)

The live rootfs is only readable on the running router, which means going **through the
`administrator` account's `vtysh` cage** to reach a real shell and copy files off —
that's the subject of [doc 12](12-boot-chain-and-access.md) and the next task. When we
do, the missing 115 backup binaries and the entire live HJLL88 filesystem both come
within reach in one move.

---

*Related: [doc 12](12-boot-chain-and-access.md) (boot chain & how the two banks are
selected), [doc 01](01-device-overview.md) (the SoC and flash hardware),
[doc 06](06-system-os.md) (the OS layer these files make up).*
