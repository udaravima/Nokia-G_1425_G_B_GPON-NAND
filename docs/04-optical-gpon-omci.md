# 04 · The Optical Layer — GPON, OMCI, PLOAM, and Light Power

This is the half of the box that touches the fiber. It's also the half with the
most *physical* failure modes — because unlike Wi-Fi, a fiber problem is often a
literal speck of dust or a bent cable. Understand this layer and you can tell an
ISP exactly what's wrong instead of "the internet is down."

## The chain of concepts, from glass to Ethernet

```
 fiber  ──►  transceiver  ──►  GPON MAC  ──►  PLOAM handshake  ──►  OMCI service  ──►  Ethernet
 (light)     (BOSA: laser    (framing,      (register onto      (OLT provisions    (your LAN)
              + photodiode)   time slots)    the PON: O1→O5)      VLANs/profile)
             │
             └── DDM telemetry: Rx power, Tx power, temp, bias
```

Every optical log entry is somewhere on that chain. Let's walk it.

## GPON in one picture: why it's "passive" and why timing is everything

Your fiber isn't point-to-point. It fans out through an unpowered **splitter** to
up to 64 (or 128) homes. That single design choice explains most of GPON's
quirks:

- **Downstream** (OLT → you): the OLT broadcasts *everything* to *everyone*; your
  ONT decrypts and keeps only its own frames. (GPON encrypts downstream with
  per-ONT keys precisely because your neighbor physically receives your bits.)
- **Upstream** (you → OLT): if two ONTs lit their lasers at once, the light would
  collide on the shared fiber. So the OLT hands each ONT **exact time slots**
  (TDMA) and continuously measures distance to align them (**ranging**). Your
  ONT's laser is *off* except during its granted microseconds.

Consequence for you: upstream problems are often *timing/ranging* problems, and a
weak signal from *one* home can be a dirty connector at *that* home — the shared
medium makes faults look confusing until you isolate by ONT.

## PLOAM: the O1→O5 state machine (how the link comes up)

Before any data flows, the ONT must **register** onto the PON. That handshake is
**PLOAM**, and it climbs a state ladder:

| State | Meaning (plain) |
|-------|-----------------|
| O1 | Initial — just powered, listening for the OLT. |
| O2–O3 | Syncing to the downstream frame, learning parameters. |
| O4 | Ranging — OLT measures your fiber distance and assigns timing. |
| **O5** | **Operational — you're on the network, traffic flows.** |

In `history/omci.log` you can watch this directly:

```
[08-08 20:41:58][E]checkPonLEDStatus: Ploam State changed to ~O5, ONU get offline.
[08-08 20:41:58][C]OmciMain:: OMCI Link Status is DOWN
[08-08 20:41:58]GPON_STATUS … state is DISCONNECTED
…
[08-08 20:46:03]fiberStatus=1[0=DOWN/1=UP]      ← link came back
[08-08 20:46:05]fiberStatus=0[0=DOWN/1=UP]      ← and dropped again
[08-08 20:46:31][E]checkPonLEDStatus:Fiber disconnected
```

**What this tells you:** the `~O5` means *leaving* O5 — the ONT fell off the PON.
Then within five minutes the fiber went **up, down, up, down** — that flapping
pattern (not one clean drop) is the fingerprint of a **physical-layer problem**:
a loose/dirty **SC/APC connector**, a tight bend, or marginal optical power right
at the alarm threshold. A config or ISP-side deprovision would look like one
clean, sustained down — not a stutter.

## OMCI: how the ISP configures the ONT without logging in

Once at O5, the OLT provisions services over **OMCI** (ITU-T G.988). This is a
separate management channel from TR-069 (doc 07) — and an important distinction:

- **OMCI decides** *what service the fiber carries* — which VLANs map to internet
  vs. voice vs. IPTV, the ONT's allowed bandwidth, which optical alarms are armed.
- **You cannot change OMCI settings.** They're pushed from the OLT. If your
  internet VLAN is wrong, that's an ISP provisioning issue, not a router setting.

Mechanically, the OLT reads and writes standardized **Managed Entities** (ONU-G,
ANI-G, the bridge/VLAN MEs, etc.) on the ONT. The **`omciMgr`** daemon implements
the ONT side; `omci_com_fail` logs OMCI communication failures, and it's also
where the chip identity leaked (`isEN7528 Frame Engine detect`).

## DDM: reading the health of the light

**Digital Diagnostic Monitoring** is the fiber's vital-signs monitor. The number
that matters most is **Rx optical power in dBm**:

- It's **negative** (e.g. −22 dBm strong, −28 dBm marginal, −34 dBm failing).
- **Less negative = more light = healthier.** Each −3 dB is *half* the power.
- This device arms a **warning** near −28.0 dBm and an **alarm** near −28.9 dBm.
  Below that, GPON's error correction starts losing frames → packet loss, then
  the O5 drop you saw above.

> The detailed Rx-power timeline for this device lives in the sibling
> `onu_info.log` analysis (a day where it sank to −35.2 dBm). In *this* dump the
> optical story shows up as the **consequence** — the fiber flap in `omci.log` —
> rather than raw dBm readings.

**The trap worth repeating:** the file named `ddm_log.log` here is a **Wi-Fi**
daemon's log, not optical DDM. Real optical DDM on this platform surfaces through
`omciMgr`/`sysinfo.log`, not a file called "ddm." Don't let the name fool you.

## Files that belong to this layer

| File | What it holds |
|------|---------------|
| `logs/history/omci.log`, `omci.log.bak` | PLOAM state changes, fiber up/down, OMCI link status. **The first place to look for outages.** |
| `logs/omci_com_fail`, `omci_com_fail.0` | OMCI communication failures + diag-tool/chip init. |
| `logs/bosa_log.*.tar.bz2` | Compressed logs from the optical sub-assembly (BOSA). |
| `configs_key_data_bak/bosa/cfg.bob`, `modlut.bob` | Optical transceiver **calibration** blobs (binary). |
| `logs/sysinfo.log` | Periodic system snapshot; includes optical/service status lines. |

## How to investigate a fiber problem with this dump

1. `grep -iE "Ploam|fiber|O5|disconnect" logs/history/omci.log` — did it drop, and
   was it a clean drop or a flap?
2. Correlate timestamps with `reboot.log` — did a "drop" actually coincide with a
   reboot (power loss) instead?
3. If flapping: suspect **physical** (connector, bend, power near threshold), and
   pull the Rx-power dBm trend (from `onu_info.log`/OLT stats) to confirm.
4. If one clean sustained down with no reboot: suspect **ISP-side** (deprovision,
   fiber cut, OLT port) — and now you can tell them *exactly that*.

Next: the noisy other half — [05-wifi-mesh-ai.md](05-wifi-mesh-ai.md).
