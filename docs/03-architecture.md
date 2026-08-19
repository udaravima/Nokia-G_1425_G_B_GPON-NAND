# 03 · Architecture — How The Pieces Connect

Doc 01 said *what* the box is; the glossary named the parts. This doc shows how a
packet actually travels through it, and — separately — how the box is *managed*.
The single most useful idea here: **the data plane and the management plane are
different paths run by different actors.** Confusing them is the root of most
log confusion.

## 1. The data plane — how your traffic flows

```
 INTERNET
    │  (ISP fiber, GPON)
    ▼
 ┌──────────────────────────────────────────────────────────┐
 │  EN7528 SoC (one chip)                                    │
 │                                                           │
 │  ┌──────────┐   ┌───────────────┐   ┌──────────────────┐  │
 │  │ GPON MAC │──►│  Frame Engine │──►│ Hardware Switch  │  │
 │  │ (optics) │   │  (PPE: NAT &  │   │  + Linux bridge  │  │
 │  └──────────┘   │   forwarding  │   └───────┬──────────┘  │
 │                 │   offload)    │           │             │
 │                 └───────────────┘   ┌───────┼───────┐     │
 │                                     ▼       ▼       ▼     │
 │                                  LAN1-4   Wi-Fi   Wi-Fi   │
 │                                  (copper) 2.4GHz  5GHz    │
 └──────────────────────────────────────┼───────┼───────────┘
                                         ▼       ▼
                                   your wired &  wireless devices
```

Read it as a chain:

1. **Fiber → GPON MAC.** Light arrives, the GPON MAC (in the chip) de-frames it
   and checks it's addressed to this ONT's time slots. This is the layer the
   optical alarms and PLOAM states (doc 04) live at.
2. **GPON MAC → Frame Engine (PPE).** The **Packet Processing Engine** does NAT
   and routing decisions *in hardware*. This is why a 422 MB 4-core MIPS box
   can push gigabit: the CPU sets up rules once, then the silicon forwards the
   bulk of packets without waking the CPU. When you see the CPU busy anyway
   (doc 10's load average), it's control-plane work — Wi-Fi management, not
   packet pushing.
3. **PPE → switch/bridge.** Forwarded frames land on the internal switch, joined
   to a **Linux bridge** (`brctl_show.txt`) that unifies the LAN ports and the
   Wi-Fi VAPs into one L2 segment.
4. **Bridge → LAN ports & Wi-Fi radios.** Copper ports are driven by the
   **Ethernet PHYs** (`.dbg.phy.log`); Wi-Fi by the two radios exposed as VAPs
   `ra*` (2.4 GHz) and `rai*` (5 GHz).

Your LAN sits on **192.168.55.0/24**, gateway **192.168.55.254** (the device
itself) — confirmed in `topology.txt` and the DHCP leases.

## 2. The management plane — how the box is controlled

Here's the crucial split. **Two remote authorities, two protocols, two ends of
the box:**

```
   ISP OLT ──OMCI over fiber──►  [ optical side ]   managed by omciMgr
                                       │
                                   THIS DEVICE
                                       │
   ISP ACS ──TR-069 over WAN───►  [ router side  ]   managed by cfgmgr
   (nudged via XMPP Connection Request)
```

- The **OLT manages the optical/service side over OMCI** — VLANs, the service
  profile, optical alarms. You (and even the ISP's app) never see this as
  "settings"; it's provisioned from the exchange. Daemon: **`omciMgr`**.
- The **ACS manages the router side over TR-069** — Wi-Fi, LAN, firmware,
  diagnostics. Daemon: **`cfgmgr`** owns the data model the ACS reads/writes.
- Because the router is behind NAT, the ACS can't reach it on demand; it sends an
  **XMPP** "Connection Request" that tells the box to call home *now* (doc 07).

**Locally**, all configuration — whether it originated from the web UI, TR-069, or
OMCI — converges on **`cfgmgr`** and the **UCI** files. That convergence is why
`cfgmgr` is the single busiest daemon in `messages`.

## 3. The management plane, one level down: how daemons talk

Inside the box, daemons don't hardcode connections to each other — they use
**ubus**, OpenWrt's message bus:

```
  web UI ─┐
  TR-069 ─┼─► cfgmgr ─► UCI files (/etc/config/*) ─► daemons re-read config
  OMCI  ──┘     │
                └─► ubus ◄─► rpcd, wifi daemons, dhcp, etc. (query/notify)
```

So "change the Wi-Fi channel" becomes: request → `cfgmgr` writes UCI → a ubus
notify → the Wi-Fi/RRM daemons reload. Every `cfgXml_*` line in `messages` is
`cfgmgr` walking this data model.

## 4. The Wi-Fi mesh control loop (the busy part)

The Wi-Fi "AI" stack is its own little control system layered on top:

```
  measure ──► decide ──► act ──► measure …

  USSA          ai_controller       ai-agent / localrrm
  (per-client   (learning: channel  (apply channel /
   RSSI, airtime, utilization →      steer client to
   scan neighbors) pick channel &    better radio/node)
                   steering)
        └──────────► ecd (edge controller: topology + REST + coordination)
```

This loop runs continuously, which is *by design* but also *why the logs are
enormous*: `ai_controller` alone logged 87k lines here just narrating its
sampling. Doc 05 unpacks each stage; doc 10 quantifies the noise.

## 5. Process map — who's actually running

From `ps.txt` (**142 processes**), grouped by the world they serve:

| Group | Representative processes | Role |
|-------|--------------------------|------|
| Optical / GPON | `omciMgr` | Talks OMCI to the OLT; owns PLOAM/alarms. |
| Config / model | **`cfgmgr`**, `rpcd`, `ubusd` | The data model + IPC bus + RPC bridge. |
| Nokia net stack | `nwhed`, `nwsl`, `nva`, `ussa` | Vendor forwarding/steering/node-view glue. |
| Wi-Fi AI | `ai_controller`, `ai-agent`, `ecd`, `localrrm_*` | The mesh control loop. |
| Management | TR-069 client, XMPP client, `dhcpd`, `dproxy` | Remote mgmt + LAN services. |
| Voice | VoIP/SIP daemons | Telephone service (config present). |

You don't need to memorize these — the point is that when a log file is named
after one of them, this table tells you which *world* it reports on.

Next: the four world-specific deep-dives, starting with the fiber —
[04-optical-gpon-omci.md](04-optical-gpon-omci.md).
