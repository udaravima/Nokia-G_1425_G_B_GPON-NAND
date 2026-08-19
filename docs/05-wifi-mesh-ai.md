# 05 · The Wi-Fi Mesh "AI" Layer — Where The Log Volume Lives

If the optical layer is where things *physically* break, this is where things get
*noisy*. One daemon in this stack, `ai_controller`, logged **87,155 lines** across
its rotations in this single dump. That flood is the "duplication" you were
fighting — and once you know what it's narrating, you can ignore 99% of it and
read the 1% that matters. This doc is the map.

## The big idea: a self-optimizing Wi-Fi network

Modern mesh Wi-Fi tries to run itself. It continuously answers three questions:

1. **Which channel should each radio use?** (avoid congestion & interference)
2. **Which node/band should each client be on?** (keep phones on the best link)
3. **How healthy is the mesh backbone?** (the backhaul between nodes)

Nokia implements this as the **Multi-AP (MAP / EasyMesh)** framework plus an "AI"
control loop on top. This device is the **Controller** (`map_role.txt`) — the
brain that decides for the whole mesh. Any satellite node would be an **Agent**
that just executes.

## The control loop and its daemons

```
   ┌──────────── MEASURE ───────────┐   ┌──── DECIDE ────┐   ┌──── ACT ────┐
   │ USSA: per-client RSSI, airtime │   │ ai_controller  │   │ ai-agent    │
   │ + neighbor/off-channel scans   │──►│ "learning":    │──►│ localrrm_*  │
   │ (ussa.log, scan_result_*.txt)  │   │ channel util → │   │ apply chan; │
   └────────────────────────────────┘   │ channel &      │   │ steer client│
                                         │ steering choice│   └─────────────┘
                                         └───────┬────────┘
                                                 │
                              ecd (Edge Controller Daemon): owns the topology,
                              serves the REST API, coordinates the whole loop
```

### USSA — the "eyes" (station steering agent)
`ussa.log` is a stream of per-client health snapshots:

```
mac is [86:b4:c4:…], rssi=-47, time=4921, vapname=rai0, tx=6000, rx=780000
mac is [7c:f6:66:…], rssi=-75, time=7176, vapname=ra2,  tx=1000, rx=48000
```

Decode one line: a client on **`rai0`** (5 GHz) at **−47 dBm** (excellent) pulling
**780 Mbps** rx — happy. The second, on **`ra2`** (a 2.4 GHz VAP) at **−75 dBm**
(weak, far) at **48 Mbps** — a candidate USSA might **steer** to a nearer node or a
better band. USSA also runs the **scans** (`scan_result_ra0.txt`,
`scan_result_rai0.txt`, `neighbor_info_*`) that tell RRM what else is on the air.
Its steering/disassociation *events* (not the routine snapshots) land in
`ussawifievent.log` — that's the file to read when a device keeps dropping.

### ai_controller — the "brain" (and the noise source)
Its lines look like:

```
[000001] … DEBUG | aistages/learning/learning.cpp | get_min_20m_free_ch_utilization:212 channel_low_samples 6
```

It's sampling **channel utilization** ("how busy is each 20 MHz channel") to feed
its channel-selection decision. At **DEBUG** level it logs *every sample* — hence
tens of thousands of near-identical lines. **This is normal.** The signal is
buried in the rarer non-DEBUG lines: actual channel *changes*, decisions, and
warnings. (See "Taming the noise" below.)

### ai-agent / localrrm — the "hands"
`ai-agent.log` and `localrrm_*_startup.log` record the *application* of decisions:
switching a radio to a new channel, adjusting power. When `ai_controller` decides
"move to channel 36," it's `ai-agent`/`localrrm` that makes the radio do it,
announcing the move to clients via **CSA** (Channel Switch Announcement).

### ecd — the coordinator + REST API
The **Edge Controller Daemon** (v1.16.15) is the spine. Two of the most *readable*
files in the whole dump are HTTP responses *from* ecd:

- **`topology.txt`** — a JSON map of the mesh: every AP, its role, IP/MAC,
  backhaul quality, `cost-to-gateway`, uptime. **The single best "what does the
  mesh look like right now" file.**
- **`aps.txt`** / **`debug_aps.txt`** — detailed per-AP + per-client state,
  capabilities, steering info.

Its internal state DBs are the `ecd_*.json` files (`ecd_config_db.json`,
`ecd_node_info_db.json`, `ecd_dbg_db.json`).

## Interface naming — the Rosetta stone for every Wi-Fi line

You cannot read these logs without this table. On this MediaTek platform:

| Interface | Band | Meaning |
|-----------|------|---------|
| `ra0` | 2.4 GHz | Primary VAP (main SSID) |
| `ra1`, `ra2`, … | 2.4 GHz | Extra VAPs (guest / IoT / backhaul) |
| `rai0` | 5 GHz | Primary VAP (main SSID) |
| `rai1`, `rai2`, … | 5 GHz | Extra VAPs |
| `apcli*` | either | Station/backhaul client interface (uplink to another node) |

So `vapname=rai0` = "on the 5 GHz main network," `Band[0]` = 2.4 GHz radio,
`Band[1]` = 5 GHz radio (as seen in `ussa.log` scan lines).

## The whole file roster for this layer

| File(s) | What it is |
|---------|-----------|
| `ai_controller.log`, `.0`–`.9` | The AI brain's debug stream. **87k lines.** Read decisions, skip samples. |
| `ai-agent.log`, `.old` | The executor applying channel/steering changes. |
| `ecd_*.json` | Edge-controller internal state (config, node info, debug). |
| `topology.txt`, `aps.txt`, `debug_aps.txt` | **Human-readable** REST snapshots of the mesh. Start here. |
| `ussa.log`, `.old`, `ussawifievent.log`, `ussawifistatus.log` | Client health snapshots + steering events + status. |
| `scan_result_ra0/rai0.txt`, `neighbor_info_*` | What the scans saw (neighboring APs, interference). |
| `channel_utils_ra0/rai0.txt`, `historychan.txt`, `historychop.txt` | Channel utilization + channel-change history. |
| `acs_report_ra0/rai0.txt` | Auto-Channel-Selection reports (why a channel was chosen). |
| `blocked_channels.txt` | Channels currently off-limits (usually **DFS**/radar). |
| `debug_steer.txt`, `debug_bh_steer.txt`, `debug_steering.txt` | Client-steering and **backhaul**-steering decision traces. |
| `mesh_topo_dump.txt`, `meshctl_dump.txt`, `map_role.txt` | Mesh topology dumps + this node's MAP role. |
| `nva_node_info`, `nva_cli_tool.txt` | Network-View-Agent node identity/state. |
| `Wireless_RT2860AP*.dat`, `uci_show_wireless.txt` | The actual Wi-Fi driver config (SSIDs, keys, channels). |

> **`mesh_topo_dump.txt` is empty in this dump** — expected, because this is a
> single-node mesh (it's the controller *and* the only AP). With satellites it
> would be populated. Use `topology.txt` instead; it's never empty.

## Taming the noise (practical)

The reason `tmp/` felt overwhelming is almost entirely this layer. Three moves:

1. **Read `topology.txt` first, not the logs.** It's a clean JSON snapshot of the
   whole mesh — role, IPs, backhaul, clients — no grepping required.
2. **In `ai_controller.log`, drop the DEBUG samples:**
   ```
   grep -vE "channel_low_samples|get_min_20m_free_ch_utilization" ai_controller.log
   ```
   That alone removes the bulk of the 87k lines, leaving real decisions.
3. **For "why did my phone drop / roam," read `ussawifievent.log`,** not
   `ussa.log` (events vs. routine snapshots).

## The mental takeaway

This layer is a **continuous control system**, so it logs continuously — volume is
a feature of the design, not a symptom of a problem. The skill is separating the
*heartbeat* (samples, scans, status) from the *events* (channel changes, steers,
disassocs). Doc 10 quantifies exactly which files are heartbeat and which carry
signal.

Next: the ground floor everything runs on — [06-system-os.md](06-system-os.md).
