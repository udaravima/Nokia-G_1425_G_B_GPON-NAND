# 10 · Observations — What This Dump Actually Reveals

The other docs teach the system in general. This one is about *your* box on
*2026-08-15*: the concrete things the dump shows, ranked by how much they matter,
each with the file and number behind it so you can re-check. Read this as a
findings report, not a to-do list — several items are "understand," not "fix."

## Finding 1 — A watchdog reboot, minutes after an optical-power storm ★ most interesting

The device you're holding booted **11 hours before the dump**, and that boot was a
**recovery from a hang**, not a normal restart:

```
reboot.log:  [Fri Aug 14 21:53:07 IST 2026] … Last Reboot reason : watchdog
             store the reboot reason value_reg=0x80010000 value=2
```

Now line that up against the sibling `onu_info.log` analysis of the *same evening*:

| Time (08-14) | Event | Source |
|-------------|-------|--------|
| 21:36–21:50 | PON RX optical power collapses to **−35.2 dBm** (6.3 dB past the alarm line), sustained | `onu_info.log` |
| **21:53:07** | **Watchdog reboots the device** (software hung) | `reboot.log` |
| 21:53:34 | `dhcpd` restarts as services come back up | `onu_info.log` (`[crit] restarting ndk_udhcpd`) |

**The story this suggests:** a severe fiber-power fault battered the optical/OMCI
stack for ~15 minutes, and within three minutes a software watchdog fired and
rebooted the box. That is a *correlation*, and a tight one — but the black-box
recorder (below) shows the picture is more complicated than "fiber killed it."

### The black box: `pstore/console-ramoops-0` (added after re-audit)

The kernel's **ramoops** persists the console buffer across a reset into
`logs/pstore/console-ramoops-0` (28 KB). This is the *actual crash recorder* for
the hang above, and I missed it on the first pass. Two things it reveals:

1. **A concurrent GPON signal storm — but a different alarm than I assumed.** The
   final console before the reset is a rapid flap of **PLOAM SF/SD** alarms and
   OMCI MAC resets:
   ```
   PLOAM: SF ALARM Signal failed.   /   PLOAM: SD ALARM Signal degraded.
   OMCI HW/SW RX mismatch, MAC Reset   (×14 in the captured tail)
   ```
   **SF = Signal Fail, SD = Signal Degrade** — these are *bit-error-rate*-based
   GPON alarms (ITU-T G.984), a distinct mechanism from the *low-Rx-power* alarms
   in `onu_info.log`. So the fiber fault manifested at **two** layers at once:
   optical power (−35 dBm) *and* BER (SF/SD). The OMCI MAC was resetting itself
   repeatedly trying to recover.

2. **The reset trailer points at the Wi-Fi driver, not the optical stack.** After
   the alarm storm, the watchdog reset dump shows a register/stack snapshot with
   one resolved symbol:
   ```
   [<80561d20>] rt28xx_ioctl+0x60/0xb0
   … wdogCnts[2].wdog_cnt == 0xc8ba      (per-core watchdog counter)
   … VPEConf0 : 802f0003                 (MIPS multithread config register)
   ```
   **`rt28xx_ioctl` is the MediaTek/Ralink Wi-Fi driver's ioctl handler.** So when
   the watchdog fired, the CPU was in (or under) the *Wi-Fi* driver, not the
   optical path.

**The corrected reading (this reverses my first lean):** the crash evidence puts
the CPU in the **Wi-Fi driver ioctl path** at reset time, while the **GPON was in
an SF/SD signal-fail storm** in parallel. Whether the optical storm *caused* the
Wi-Fi hang (e.g. by driving the control-plane into a state that wedged an ioctl),
whether they share a root cause, or whether the Wi-Fi hang was independent and the
timing coincidental — **the dump cannot prove.** But it's now more accurate to say
*"a Wi-Fi-driver hang during a concurrent fiber signal-fail storm"* than *"the
fiber fault deadlocked a task."* Caveats that keep this honest: the trailer is a
**partial** capture (`** N printk messages dropped **` appears repeatedly — the
buffer was under pressure), a single frame is not a full backtrace, and the last
*timestamped* console line is ~16:09 while the reset was 21:53, so the exact hang
onset is unresolved from this buffer alone.

**Why it matters / stakes:** if repeated, this is a *reboot-under-fault* loop —
the worst kind of outage, because every time the signal degrades the box may reset
and drop every client. Since you've already reported the fiber, the *new* thread
here is the **Wi-Fi-driver angle**: a firmware bug where `rt28xx_ioctl` wedges
under some condition would be worth noting to Nokia/the ISP separately from the
optical fault. **What to check next:** whether other resets left ramoops captures
(this one is `console-ramoops-0`; a `-1`, `-2`… would be older crashes), and
whether their trailers also land in `rt28xx_ioctl`. A repeated frame is a real
firmware bug signature; a one-off is just a bad night.

## Finding 2 — 40% of the main log is "errors," and almost none are real ★

`logs/messages` is **10,831 lines, of which 4,300 (≈40%) are `[err]` or `[crit]`.**
That sounds like a house on fire. It isn't — it's one badly-tuned logger. The top
error signatures (normalized, counts from `messages`):

| Count | Signature | What it actually is |
|------:|-----------|---------------------|
| **2,193** | `generate_channel_locations_tri: Too many channel locations!!! Will return!` | A Wi-Fi channel-planning routine hits an internal cap and bails. Repeats endlessly. **Benign but wasteful.** |
| **423** | `cfgXml_getNodeByUri(…WLANConfiguration.13.) failed` | Something keeps querying **WLAN interface 13**, which doesn't exist. A config/index mismatch. |
| **135** | `cfgXml_getObject on oid 269 failed` | The downstream failure of the same phantom-WLAN lookup. |
| **105** | `isValidAliasName: **Valid** AliasSting Found in the path` | A *success* message logged at **error** severity. Pure mislabeling. |
| **70** | `cfg_Get_WIFICapability: hcfg_hal_get wifi capability failed!` | A capability probe that fails and is retried. |
| **64** | `start_wps_timer: **No need** to start WPS timer` | Another *normal* condition logged as an error. |

**The lesson (this is the transferable skill):** on this firmware, **severity is
the daemon's opinion, not ground truth.** `[err]` is used for hit-a-limit,
not-configured, and even *succeeded*. So never read these logs top-to-bottom or
panic at an error count — **cluster identical messages first** (doc 09, recipe 1).
Those 2,193 channel-location errors are *one* quirk, not 2,193 problems. This is
exactly why `tmp/` felt overwhelming: the volume is real, the number of distinct
issues is tiny.

**Mild real concern inside the noise:** the 423 + 135 phantom-`WLANConfiguration.13`
lookups suggest a genuine config-index inconsistency (a Wi-Fi config referencing an
interface that was never created). Harmless to service, but it's wasted work — a
steady stream of pointless error-logging (which, on 4 cores, the box absorbs fine;
see the corrected Finding 3).

## Finding 3 — High load, idle CPU: the box is I/O-bound, not CPU-bound ○

This one took two corrections to get right, so here's the fully-evidenced version.

**First: how many cores?** The data model says `X_ASB_COM_NumberOfCpuThreads = 1`,
but that value is **wrong**. Two independent proofs it's a 4-core SMP system:
- `Boot.log`: `SMP`, `CPUs=4`, `Detected 3 available secondary CPU(s)`, MIPS 1004Kc.
- **Runtime** `ps.txt`: the kernel runs per-CPU threads for **all four** cores —
  `ksoftirqd/0..3`, `migration/0..3`, `kworker/0..3`. A single-core boot would
  show only the `/0` set. So all 4 cores are online *and scheduled*, not just
  detected. `sysinfo.log` confirms with a `CPU0 CPU1 CPU2 CPU3` header.

**Second: what does the load actually mean?** `sysinfo.log` captures load and
utilization in the same breath:
```
load average: 5.82, 5.27, 4.56     CPU:  1.9% usr  1.9% sys  96.1% idle
```
**Load ~5.8 while 96% idle.** Not a paradox: load average counts runnable (R) tasks
*plus* uninterruptible-sleep (D) tasks — and D-state means blocked on **I/O or a
lock**, not on the CPU. So the load is **queueing on I/O, while the cores sit
idle**. Memory is fine too (`meminfo.txt`: 228/422 MB available). Nothing is
CPU-starved.

**The likely driver** is the very thing that made this dump huge: the **Debug** log
level (visible on the web UI's Maintenance→Log page). Every daemon streams verbose
logs to **NAND flash**; NAND writes are slow; writers block (D-state); load climbs
while CPUs idle. It ties the whole story together — the log-volume problem
(Finding 4) and the "high load" are the *same* phenomenon seen from two angles.
**Actionable:** dropping the log level from Debug should cut both the noise and the
load. So this is **not** the CPU-starvation I first claimed — it's benign I/O wait
with an easy knob.

## Finding 4 — One daemon is 60%+ of the entire dump's volume ○

`ai_controller.log` + its rotations `.0`–`.9` total **87,155 lines**, essentially
all of them the same DEBUG sample line (`channel_low_samples`). For scale, that's
~8× the entire `messages` file. **This is the "duplication" that made `tmp/` feel
unreadable** — it's not many problems, it's one daemon logging its heartbeat at
DEBUG. Filter it (doc 09) and the dump shrinks to something a human can hold.

Not a fault — a verbosity setting. But worth knowing that if you ever ship these
dumps around, this one daemon dominates the size.

## Finding 5 — Power environment: 75 clean power-downs ○

`reboot.log` reasons tally to **75 × Powerdown** and **1 × watchdog**. Seventy-five
mains-power losses over the device's life points at an **unprotected power feed**
(no UPS, a switched outlet, or an area with frequent outages). None of these are
device faults — but each one is a total outage, and (per doc 04) a power-cycle can
*look* like a "fiber down" in `omci.log`. **Practical upshot:** a small UPS would
eliminate the single most common cause of downtime on this box, and would remove
the ambiguity between "power blip" and "fiber fault" when reading future logs.

## Finding 6 — Security & exposure: quiet, but note the secrets in the dump ○

- **Login attacks: none evident.** `login_fail_count.log` and
  `app_nor_acc_fail_login_cnt` both read **0**. No brute-force in the captured
  window.
- **Cloud management: off.** `NWCC` URLs are empty (doc 07) — ISP/ACS-managed only.
- **But the dump itself is sensitive.** `configs_key_data_bak/` contains IPsec
  **private keys and certs**, `subscriber_data.txt`, VoIP credentials, and the raw
  Wi-Fi config (`Wireless_RT2860AP*.dat`) with SSIDs/keys; `dhcp.leases` and the
  USSA logs carry client MACs/hostnames. **Before this dump ever leaves your
  machine, strip those** — the same discipline behind the sanitized
  `onu_info_transfer.log` you built earlier.

## The one-screen summary

| # | Finding | Severity | Action |
|---|---------|:--------:|--------|
| 1 | Watchdog reboot minutes after a −35 dBm optical storm (08-14 21:53) | **investigate** | Check for repeats; report with RX-power evidence. |
| 2 | 40% of `messages` are errors; ~5 benign signatures explain nearly all | understand | Cluster, don't panic; the WLAN.13 phantom is the only real (minor) bug. |
| 3 | 4-core SMP; high load but **96% idle** → I/O-wait from Debug logging, not CPU load | tunable | Lower the web-UI log level from Debug to cut flash I/O + load. |
| 4 | `ai_controller` = 87k lines, ~all DEBUG | understand | Filter it; it's heartbeat noise. |
| 5 | 75 power-downs | low | Consider a UPS. |
| 6 | No login attacks; but dump holds keys/subscriber data | hygiene | Sanitize before sharing. |

---

### Where the map ends — good next questions to chase

- **Is Finding 1 a pattern?** Correlate *every* watchdog/crash in `reboot.log` and
  `pstore` against optical events. This is the highest-value thread and the dump
  can't fully answer it alone — you'd want a longer `omci.log` history.
- **What is `generate_channel_locations_tri` tripping on?** 2,193 hits is a lot of
  wasted CPU (Finding 3 ↔ 2). The `acs_report_*` and `channel_utils_*` files are
  where its inputs live.
- **Which config creates the phantom `WLANConfiguration.13`?** `uci_show_wireless.txt`
  vs. the live model would show the mismatch.
- **What does the `customer` log (1.7 MB) emphasize** that `messages` doesn't? It's
  the operator-facing view — a different editorial slice of the same events.

You now have the whole map: what the box is, the vocabulary, the architecture, each
subsystem, every file, how to read them, and what today's dump says. Go be wiser
than yesterday.
