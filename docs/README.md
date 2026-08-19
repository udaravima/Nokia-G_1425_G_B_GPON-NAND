# Nokia G-1425G-B — Diagnostic Dump, Explained

This folder turns the raw `tmp/` diagnostic dump pulled from a Nokia **G-1425G-B**
into something you can *read*. The dump is 262 files and ~25 MB of overlapping
logs, config snapshots, and command outputs. Ninety percent of the volume is
repetition from one or two chatty subsystems — so the goal here is not to
reproduce it, but to give you the **map**: what each piece is, what it does, how
it works, and where to look when something is wrong.

## What this box actually is (the one-paragraph version)

It is two devices welded together and managed as one. The first is a **GPON
ONT** — an *Optical Network Terminal*, the thing that turns the fiber coming from
your ISP into Ethernet. The second is a **Wi-Fi mesh residential gateway** — a
router with a self-optimizing "AI" Wi-Fi stack that can act as the **controller**
of a multi-node mesh. Both halves run on a single OpenWrt Linux system on a
MediaTek/Airoha **EN7528** chip, and both are remotely managed by your ISP over
a protocol called **TR-069**. Almost every unfamiliar word in the logs belongs
to one of those four worlds: **optical/GPON**, **Wi-Fi mesh**, **the OpenWrt OS**,
or **TR-069 management**.

## How to read these docs

Start at the top and go down; each builds on the last. If you only read two,
read the **glossary** and the **architecture** doc — together they decode 90% of
the noise.

| # | Doc | What it gives you |
|---|-----|-------------------|
| — | [01-device-overview.md](01-device-overview.md) | Who this device is: identity, hardware, firmware, roles. Start here. |
| — | [02-glossary.md](02-glossary.md) | Every acronym in the dump — *what it is / what it does / how it works*. Your decoder ring. |
| — | [03-architecture.md](03-architecture.md) | How the pieces connect: the data path (fiber → chip → your devices) and the management path. |
| — | [04-optical-gpon-omci.md](04-optical-gpon-omci.md) | The fiber half: GPON, OMCI, PLOAM states, optical power (DDM), the fiber-flap story. |
| — | [05-wifi-mesh-ai.md](05-wifi-mesh-ai.md) | The Wi-Fi half: EasyMesh/Multi-AP, the AI RRM controller, USSA steering, NVA, ECD. This is where the log volume comes from. |
| — | [06-system-os.md](06-system-os.md) | The OpenWrt layer: uci/ubus, processes, memory & load, bridging, the reboot history. |
| — | [07-management-tr069.md](07-management-tr069.md) | The remote-management layer: TR-069, the TR-181 data model, `cfgcli`, the ACS, XMPP, cloud (NWCC). |
| — | [08-file-index.md](08-file-index.md) | A lookup table for the whole dump: every file/group → one line on what it is and when you'd open it. |
| — | [09-reading-logs.md](09-reading-logs.md) | Practical skills: log line formats, severity tags, timezones, rotation (`.0`/`.old`/`.bz2`), grep recipes. |
| — | [10-observations.md](10-observations.md) | What *this particular* dump reveals: load pressure, reboot causes, the noisy-error offenders, the optical events. |
| — | [11-security-accounts-secrets.md](11-security-accounts-secrets.md) | Who can log in (the `vtysh`/UID-0 trap), weak password hashes, the encrypted config, and every secret the dump exposes. |
| — | [12-boot-chain-and-access.md](12-boot-chain-and-access.md) | The boot chain and flash layout from the serial log, plus the honest status of the shell-access investigation. |
| — | [13-firmware-extraction.md](13-firmware-extraction.md) | **How we cracked open the flash chip — explained simply.** Dumping the NAND, the partition map, unpacking the squashfs, the 115 bit-rotted files, and why the *live* OS bank is locked. |

## The mental model to carry through all of it

```
        ISP fiber                                     your devices
           │                                        (phones, laptops, IoT)
           ▼                                                 ▲
   ┌───────────────┐   OMCI over fiber   ┌──────────────┐    │  Wi-Fi + Ethernet
   │  OLT (ISP end)│◄──────────────────► │  THIS ONT/RGW│────┘
   └───────────────┘   (ISP manages the  └──────────────┘
                        ONT this way)            ▲
                                                 │ TR-069 (ACS manages the
                                                 │          router this way)
                                          ┌──────────────┐
                                          │  ISP's ACS   │
                                          └──────────────┘
```

Two different remote-management channels, two different ends of the box. The
**OLT** manages the *optical* side over **OMCI**; the **ACS** manages the *router*
side over **TR-069**. Keeping those two straight is the single biggest
"aha" in this whole system — most of the daemons you'll see exist to serve one
side or the other.

> **A note on the data.** Everything in these docs was read from the actual files
> in `tmp/`. Where a number is quoted (load average, memory, reboot counts, error
> tallies) it came from a real file and the source is named so you can re-check it.
> Where something is inferred from naming or vendor convention rather than proven
> by the dump, it says so explicitly.
