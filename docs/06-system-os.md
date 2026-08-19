# 06 · The System Layer — OpenWrt, Processes, Memory, and Reboots

Strip away the fiber and the Wi-Fi and what's left is a small Linux computer. It's
**OpenWrt** — the same embedded distro that runs on consumer routers worldwide —
which is great news: the skills transfer, and the files are plain text. This doc
covers the OS plumbing and the health signals you can read straight from the dump.

## OpenWrt in three primitives

Almost everything system-level in this box is built from three OpenWrt ideas.
Learn these and the `uci_show.txt` / `ps.txt` files stop being opaque.

| Primitive | What it is | Where you see it |
|-----------|-----------|------------------|
| **UCI** | The config store. All settings as text: `section.option=value`. | `uci_show.txt`, `uci_show_wireless.txt`, `/etc/config/*` in the backup tree |
| **ubus** | The internal message bus. Daemons call each other through it. | `ps_ubusd.txt` (the broker), and implicitly every status query |
| **procd/init** | The process manager that starts/supervises daemons. | `ps.txt` process tree |

**How a setting change propagates:** something writes UCI → the owning daemon is
poked (often over ubus) → it re-reads its slice of config → applies it. There's no
central "apply" button; each daemon watches its own config. This is why the config
manager `cfgmgr` is central — it's the funnel that *writes* UCI on behalf of the
web UI, TR-069, and OMCI.

## Reading system health from this dump

### Memory — comfortable
From `meminfo.txt`:

| Metric | Value | Reading |
|--------|-------|---------|
| MemTotal | 422 MB | The whole RAM budget. |
| MemFree | 140 MB | Truly unused. |
| **MemAvailable** | **228 MB (~54%)** | The number that matters — free *plus* reclaimable cache. |

**Verdict: memory is fine.** Over half is available. Embedded boxes run close to
the edge, but this one has headroom. `proc_statm.txt` and `top10times.txt` break
it down per process if you ever need to hunt a leak.

### CPU load — the one number to look at twice
From `uptime.txt`:

```
08:54:22 up 11:00, load average: 3.61, 3.51, 3.53
```

Load average is "average number of processes wanting the CPU" over 1/5/15 min.
The trap: **load is only meaningful relative to core count** — and getting the
core count right matters. The data model says `NumberOfCpuThreads=1`, but that's
wrong: **`Boot.log` shows 4 CPUs** (`SMP`, `CPUs=4`, MIPS 1004Kc). *(Earlier
drafts of these docs trusted the data model and called this box single-core —
corrected here.)*

But here's the twist that the load number alone hides. `sysinfo.log` captures load
*and* CPU utilization together:

```
load average: 5.82, 5.27, 4.56     CPU:  1.9% usr  1.9% sys  96.1% idle
```

**Load ~5.8 while the CPU is 96% idle.** Those aren't contradictory once you know
what load average really counts: tasks that are **runnable (R)** *plus* tasks in
**uninterruptible sleep (D)** — the latter almost always blocked on **I/O or a
lock**, not on the CPU. High load + high idle is the classic signature of an
**I/O-bound**, not compute-bound, system: processes are *waiting*, and waiting
tasks inflate the load number without touching a core.

So the honest reading of this box: **all 4 cores are online and nearly idle; the
load of 3–6 is queueing on I/O/locks, not CPU demand.** The prime suspect is right
there in the web UI — the log level is set to **Debug** (see the Maintenance→Log
page), so every daemon is streaming verbose logs to **NAND flash** continuously;
NAND writes are slow, tasks block on them (D-state), and the load climbs while the
CPUs sit idle. Lowering the log level would likely drop the load average sharply.
Verify per-core with `top` (press `1`) or `grep '^cpu[0-9]' /proc/stat` on the box.

### Uptime & the reboot ledger
`reboot.log` is a running history of every restart, each stamped with a **reason**.
Tallying this dump:

| Reason | Count | What it means |
|--------|-------|---------------|
| **Powerdown** | 75 | Mains power was lost (unplug / outage / power strip). *Not* a fault of the box. |
| **watchdog** | 1 | Software **hung**; the hardware watchdog timer forced a reboot to recover. |

**The one watchdog reboot is the interesting entry** — it's the box saving itself
from a lockup. One over the device's life is unremarkable; a rising count would be
worth chasing. Seventy-five power-downs, by contrast, points at the *power
environment* (no UPS), and — tying back to doc 04 — power-cycles can masquerade as
"fiber down" events, so always cross-check `omci.log` drops against `reboot.log`.

## The layer-2 / forwarding plumbing

This box is a bridge+router, and several files expose that machinery:

| File | What it shows | Why you'd open it |
|------|---------------|-------------------|
| `brctl_show.txt` | Which interfaces are bridged into the LAN. | Confirm a port/VAP is actually in the LAN segment. |
| `brctl_showmacs.txt` | Learned MAC → port table. | "Which physical port is device X on?" |
| `brctl_showstp.txt` | Spanning Tree state. | Loop/redundancy troubleshooting. |
| `ebtables.txt` | Layer-2 (Ethernet) firewall rules. | VAP isolation, guest/IoT separation. |
| `ct_tcp.txt` | Live TCP connections tracked by NAT (conntrack). | "What's talking through the box right now?" |
| `netstat_anp.txt` | Listening sockets + owning process. | "What service is on port N?" |
| `ip_link.txt`, `ifconfig.txt`, `ifconfig_a.txt` | Interface list, state, counters. | Link up/down, error/drop counters. |
| `.dbg.phy.log`, `.dbg.phy.bak` | **Ethernet PHY** driver: link speed/duplex negotiation per copper port. | Flaky/underspeed LAN port diagnosis. |
| `dmesg.txt` | Kernel ring buffer: driver init, hardware, warnings. | Boot-time or driver-level problems. |

A worked example of reading `.dbg.phy.log`:
```
phy_set_EthMode … phyid=1 speed=4 … ethmode=0x1FF0A   ← port came up at a gigabit mode
phy_set_EthMode … phyid=1 speed=3 …                   ← renegotiated to a lower speed
```
`speed=` is the negotiated PHY speed code; a port that keeps bouncing between speed
codes is a bad cable or a struggling link partner.

## Config backups — the device's whole `/etc` in one tree

`logs/configs_key_data_bak/` is a snapshot of the persistent configuration —
useful and *sensitive*:

- `etc/config/*` — the UCI configs (network, wireless, dhcp, firewall, the mesh
  configs `apcloud`, `nemo`, `plasmodium`, `fonendoscope`, `ookla`).
- `etc/ipsec.*`, `ipsec.d/certs/*`, `cacert.pem` — **VPN/IPsec certs and keys**.
- `config_encryption.cfg`, `zeroman/` — key material for config encryption.
- `alcatel/config/subscriber_data.txt`, `default_voip.txt`, VoIP XML — **subscriber
  and voice** provisioning.
- `bin_ac/RT30xxEEPROM.bin`, `SingleSKU.dat` — Wi-Fi radio calibration/regulatory.

> **Treat this subtree as secrets.** Certs, keys, subscriber and VoIP data live
> here. If you ever share a dump, this is the first directory to strip — the same
> instinct behind the sanitized `onu_info_transfer.log` you built earlier.

Curiosities worth knowing: those config names — `plasmodium`, `fonendoscope`,
`nemo`, `ookla` — are Nokia's internal codenames for features (`ookla` = the
built-in speedtest; `nemo` = a telemetry/analytics agent). Codenames, not malware.

Next: how the ISP reaches in and manages all of this —
[07-management-tr069.md](07-management-tr069.md).
