# 01 · Device Overview — Who This Box Is

Before decoding logs, pin down the machine producing them. Every identifier below
was read from the dump, and each one turns out to be a small story.

## Identity card

| Field | Value | Source file | What it tells you |
|-------|-------|-------------|-------------------|
| Model | **G-1425G-B** | `cfgcli_DeviceInfo.txt` | Nokia (ex-Alcatel-Lucent) GPON ONT with Wi-Fi 5/6 and voice. |
| Internal type | **g1425ga**, `PON_MODE=GPON` | `buildinfo.txt` | Confirms it's a **GPON** unit (2.5 Gbps down / 1.25 up class), not XGS-PON. |
| Manufacturer | ALCL, OUI **AC606F** | `cfgcli_DeviceInfo.txt` | "ALCL" = Alcatel-Lucent; the MAC OUI `AC:60:6F` is registered to Nokia/ALU. |
| Serial | **ALCLFC429D49** | `nva_node_info` | The `FC429D49` tail is the per-unit ID; it names the dump folder too. |
| Base MAC | **ac:60:6f:c3:b3:a0** | `topology.txt` | The device's own address on the network. |
| Hardware ver | 3FE49937DAAA | `cfgcli_DeviceInfo.txt` | Nokia's `3FE…` part-number scheme for the board. |
| Software ver | **3FE49568HJLL88** (`HJL.L88p01`) | `buildinfo.txt` | The firmware image. `p01` = patch level 1. |
| Build date | **2024-12-25** | `buildinfo.txt` | The firmware was compiled Christmas 2024. |
| Bootbase | Bootbase1.1, May 2022 | `cfgcli_DeviceInfo.txt` | The bootloader — older than the firmware, as expected (it changes rarely). |
| SoC platform | **MediaTek/Airoha EN7528**, MIPS | `omci_com_fail`, `release_note` (`linux_mips_uclibc_mtk`) | The chip. See "The hardware" below. |
| CPU cores | **4** (MIPS 1004Kc, SMP) | `Boot.log` (`CPUs=4`, `Detected 3 available secondary CPU(s)`) | ⚠️ The data model claims `NumberOfCpuThreads=1`, but the kernel boots 4 CPUs. Trust Boot.log — it changes how you read load averages. |
| RAM | **422 MB** total | `meminfo.txt` (`MemTotal: 422308 kB`) | Modest; ~140 MB free at capture time. |

## What the model name encodes

Nokia's ONT naming is not random. **G-1425G-B** decodes roughly as:

- **G** — GPON uplink.
- **14** — product family / port class (4× Gigabit Ethernet LAN).
- **25** — Wi-Fi generation tier (Wi-Fi 6 / 802.11ax dual-band here).
- **G-B** — hardware/regional variant, and voice (VoIP/SIP) capable — confirmed by
  `VOIP=sip` in `buildinfo.txt` and a `voip` config tree in the backups.

So from the name alone: *a GPON fiber terminal with 4 gigabit LAN ports,
dual-band Wi-Fi 6, and telephone service* — which is exactly what the config
trees in the dump confirm.

## The two roles it is playing right now

A single physical box, but it wears several "role" hats at once. The dump
captured all of them:

| Role dimension | Current value | Source | Meaning |
|----------------|---------------|--------|---------|
| Wi-Fi work mode | **RGW** | `cfgcli_WorkMode.txt` (`WorkMode=RGW`) | **R**esidential **G**ate**W**ay — it routes/NATs for your LAN, not a dumb bridge. |
| Mesh role | **Controller** | `map_role.txt` | In the Wi-Fi mesh it is the **brain**, not a satellite. See doc 05. |
| Gateway | **true**, cost-to-gateway **0** | `topology.txt` | It *is* the internet gateway; mesh satellites route through it. |
| Backhaul | connected, quality **good** | `topology.txt` | Its link toward the gateway is itself (it's the root), hence "good"/cost 0. |

The takeaway: **this unit is the root of everything** — the fiber lands here, the
routing happens here, and it's the controller that tells any other mesh nodes
what to do. If you later add a satellite ONT/extender, *that* node would report
`Agent` and a non-zero cost-to-gateway.

## The hardware, briefly

The `omci_com_fail` file leaks the chip identity: `isEN7528 Frame Engine detect`.
The **EN7528** is a MediaTek/Airoha GPON SoC — it integrates the **GPON MAC**
(the optical framer), a **hardware Ethernet switch** with a packet
**"Frame Engine"/PPE** (Packet Processing Engine, for NAT/forwarding offload so
the CPU doesn't touch every packet), and a **4-core MIPS 1004Kc** CPU cluster
running Linux 3.18 (confirmed by `Boot.log`: `SMP`, `CPUs=4`). The `release_note`
build variant `linux_mips_uclibc_mtk_release` confirms the rest: **MIPS** CPU,
**uClibc** (a tiny C library for embedded systems), **MTK** (MediaTek) platform.

Those four MIPS cores with ~422 MB usable RAM are the whole computer. Keep that
scale in mind — when doc 10 discusses a load average of 3.6, it's 3.6 spread
across *four* cores (comfortably busy), **not** 3.6 fighting over one. The
`NumberOfCpuThreads=1` in the data model is a reporting quirk, not the hardware.

## Uptime and freshness of this dump

- The dump was taken **2026-08-15 ~08:54 local** (folder name `…20260815T085412Z`
  and `uptime.txt`).
- Uptime at capture: **11 hours** (`uptime.txt: up 11:00`), i.e. it last booted
  around 21:53 the previous night (08-14). That boot was **not** clean: `reboot.log`
  records the reason as **watchdog** at `Fri Aug 14 21:53:07` — the software hung
  and the hardware timer forced a recovery. See doc 10 for why the timing (minutes
  after a severe optical-power alarm storm) is worth a second look.
- `FirstUseDate = 2026-08-15T06:38:20Z` in DeviceInfo is a soft "provisioning
  epoch," not the manufacture date — don't read it as the device's age.

## Where to go next

You now know *what* is generating the logs. Doc [02-glossary.md](02-glossary.md)
gives you the vocabulary, then [03-architecture.md](03-architecture.md) shows how
the fiber, the chip, the bridge, and Wi-Fi actually connect into a data path.
