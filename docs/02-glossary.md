# 02 · Glossary — The Decoder Ring

Every acronym and daemon name you'll hit in the dump, grouped by the world it
belongs to. Each entry follows the same shape: **what it is → what it does → how
it works**. Skim the group headers; dive into the terms that stopped you.

Where a term is central enough to have its own chapter, the deep-dive doc is
linked.

---

## A. Optical / GPON world → see [04-optical-gpon-omci.md](04-optical-gpon-omci.md)

**GPON — Gigabit Passive Optical Network**
*What:* the fiber technology your ISP uses on the last mile.
*Does:* carries ~2.5 Gbit/s downstream and 1.25 Gbit/s upstream over a single
fiber shared by many homes.
*How:* "passive" means there's no powered equipment between you and the ISP —
just glass and an optical **splitter**. Downstream, the OLT broadcasts to
everyone and each ONT ignores what isn't addressed to it; upstream, ONTs take
strict **time slots** (TDMA) so their light pulses never collide on the shared
fiber.

**ONT / ONU — Optical Network Terminal / Unit**
*What:* this device (the terminal at *your* end of the fiber). ONT and ONU are
used interchangeably in the logs.
*Does:* converts light ↔ Ethernet, and enforces the ISP's service profile.
*How:* a laser + photodiode (the **optical transceiver**/BOSA) plus the GPON MAC
in the EN7528 chip.

**OLT — Optical Line Terminal**
*What:* the ISP-end box in the exchange, the ONT's counterpart and boss.
*Does:* serves many ONTs on one fiber, hands out time slots, and remotely
provisions each ONT.
*How:* it's the "master" in the master/slave GPON relationship; it manages your
ONT over **OMCI**.

**OMCI — ONT Management Control Interface**
*What:* the management protocol between OLT and ONT (ITU-T G.988).
*Does:* lets the ISP configure your ONT's services (VLANs, the Wi-Fi/voice
profile, alarms) *without* logging into it — it's a separate channel from TR-069.
*How:* the OLT reads/writes a tree of standardized "**Managed Entities**" on the
ONT. Handled here by the **`omciMgr`** daemon; see `history/omci.log`.

**PLOAM — Physical Layer OAM**
*What:* the low-level control messaging that brings the optical link up.
*Does:* runs the handshake that registers your ONT onto the PON and assigns it an
ONU-ID.
*How:* a state machine **O1→O5**. **O5 = operational** (carrying traffic). In
`omci.log` you'll see `Ploam State changed to ~O5, ONU get offline` — that `~O5`
(leaving O5) is the ONT *dropping off* the fiber.

**SF / SD alarms — Signal Fail / Signal Degrade**
*What:* GPON physical-layer alarms based on **bit-error-rate** (ITU-T G.984), a
*different* mechanism than the low-Rx-*power* alarms.
*Does:* SD warns that errors are climbing; SF declares the received signal
unusable. Both appear in the serial console / `pstore` capture as
`PLOAM: SF ALARM Signal failed` / `SD ALARM Signal degraded`.
*How:* the GPON MAC measures errored frames; crossing thresholds raises SD then SF.
Seen storming alongside `OMCI HW/SW RX mismatch, MAC Reset` during the fiber fault
in doc 10. So a bad fiber can show up **two ways at once** — weak power *and* high
BER.

**DDM / DOM — Digital Diagnostic Monitoring**
*What:* live telemetry from the optical transceiver.
*Does:* reports **Rx optical power**, Tx power, laser bias current, and
temperature — the vitals of your fiber link.
*How:* the transceiver exposes registers the ONT polls. **Rx power in dBm** is the
number that matters: less negative = stronger light. Below roughly −28 dBm this
device raises low-power alarms (that's the story in the sibling `onu_info.log`).
*Heads-up trap:* the file literally named `ddm_log.log` in this dump is **not**
optical DDM — it's a Wi-Fi daemon's log (full of `6ghz-csa` lines). Name
collision; see doc 09.

**BOSA — Bidirectional Optical Sub-Assembly**
*What:* the physical laser-plus-detector module behind the fiber port.
*How:* `configs_key_data_bak/bosa/` holds its calibration blobs (`cfg.bob`).

---

## B. Wi-Fi mesh world → see [05-wifi-mesh-ai.md](05-wifi-mesh-ai.md)

**Multi-AP / EasyMesh (a.k.a. "MAP")**
*What:* the Wi-Fi Alliance standard for multiple access points acting as one
seamless network. Nokia's logs call it **MAP** (Multi-AP).
*Does:* lets a phone roam between nodes on one SSID without dropping.
*How:* one node is the **Controller** (decides channels, steering, which node a
client should attach to); the others are **Agents** that execute. This box's
`map_role.txt` says **Controller**.

**Backhaul**
*What:* the link that carries traffic *between* mesh nodes (as opposed to
"fronthaul," the link to your phones).
*Does:* it's the mesh's internal spine; its quality caps the whole mesh's speed.
*How:* can be Ethernet or a dedicated Wi-Fi band. `topology.txt` reports
`backhaul-quality: good` and `cost-to-gateway: 0` (this node *is* the gateway).

**RRM — Radio Resource Management**
*What:* the logic that picks Wi-Fi channels and power levels.
*Does:* moves you off congested/interfered channels automatically.
*How:* measures **channel utilization** and neighbor scans, then decides. Runs as
**`localrrm_agent`/`localrrm_controller`** and, above them, the AI controller.

**The "AI" Wi-Fi stack** (the loudest thing in the dump)
- **`ai_controller`** — the decision engine. Its `aistages/learning/…` logs show
  it sampling channel utilization to choose channels. **87,000+ lines** across
  rotations here — most of the "duplication" you noticed is this one daemon.
- **`ai-agent`** — the executor that applies the controller's decisions on radios.
- **`ecd` — Edge Controller Daemon** (v1.16.15, `edge-controller_version.txt`):
  the local process that serves the mesh topology/AP REST API (`topology.txt`,
  `aps.txt` are HTTP responses from it) and coordinates the AI stack.

**USSA**
*What:* Nokia's station-monitoring & steering agent (repo `nwfa/ussa`,
`ussa_ReleaseNotes.txt`). Read it as **"smart steering agent."**
*Does:* tracks every connected client's **RSSI**, airtime, and Tx/Rx rates, and
triggers **band/AP steering** (nudging a client to a better radio or node).
*How:* `ussa.log` lines like `mac … rssi=-47 … vapname=rai0 tx=6000 rx=780000`
are per-client health snapshots; `ussawifievent.log` records the steer/disassoc
events it generates.

**NVA — Network View Agent** (inferred from `nva_node_info`, `ps_nva.txt`)
*What:* the component that publishes this node's identity/state into the mesh's
shared view.
*Does:* answers "who is this node and what does it know about its neighbors."

**VAP — Virtual Access Point**
*What:* one SSID on one radio. Multiple VAPs share a physical radio.
*How:* the interface names encode them: **`ra0`** = 2.4 GHz primary VAP,
**`rai0`** = 5 GHz primary VAP, `ra1/ra2/rai1…` = additional SSIDs (guest, IoT,
backhaul). You'll see these all over `ussa.log` and the scan files.

**RSSI**
*What:* Received Signal Strength Indicator, in dBm (negative; closer to 0 =
stronger). `rssi=-47` is excellent, `rssi=-75` is weak/far.

**CSA / ECSA — (Extended) Channel Switch Announcement**
*What:* the 802.11 mechanism to tell clients "we're changing channel now."
*Does:* lets RRM move the radio to a new channel without kicking everyone off.
*How:* the many `unable to parse json for capable-6ghz-csa` lines in `ddm_log.log`
are the daemon checking whether clients support CSA on 6 GHz — harmless probing
noise, not errors.

**DFS — Dynamic Frequency Selection**
*What:* the rule that Wi-Fi must vacate certain 5 GHz channels if it hears radar.
*Relevance:* RRM's channel choices are constrained by DFS; `blocked_channels.txt`
lists channels currently off-limits.

---

## C. OpenWrt / OS world → see [06-system-os.md](06-system-os.md)

**OpenWrt** — the embedded Linux distro this firmware is built on. Everything
below is OpenWrt vocabulary.

**UCI — Unified Configuration Interface**
*What:* OpenWrt's config system.
*Does:* stores all settings as plain text in `/etc/config/*`.
*How:* `uci show` dumps it as `section.option=value`. Files `uci_show.txt` and
`uci_show_wireless.txt` are exactly that snapshot.

**ubus — micro bus** (`ubusd`, `ps_ubusd.txt`)
*What:* OpenWrt's internal message bus — a tiny local IPC/RPC system.
*Does:* lets daemons call each other ("give me Wi-Fi status") without sockets
each invents itself. The Wi-Fi status you see was often fetched over ubus.

**rpcd** — the daemon that exposes ubus methods as JSON-RPC to the web UI/TR-069.

**nwhed / nwsl / nwf / nva** — Nokia's own **N**et**W**ork stack daemons
(`ps_nwhed.txt`, `ps_nwsl.txt`, `nwf-*.json`). Vendor-specific glue on top of
OpenWrt; `nwf-product-cfg.json` is the product capability manifest.

**cfgmgr — Configuration Manager**
*What:* the central process that owns the device's data model.
*Does:* every "set this parameter" (from web UI, TR-069, or OMCI) funnels through
it. It's the busiest daemon in `messages`, and the source of the repeated
`WLANConfiguration.13 … failed` errors.

**brctl / bridge** — Linux bridge tooling. `brctl_show.txt` shows which
interfaces are bridged into the LAN; `brctl_showmacs.txt` is the learned MAC
table (which device is on which port).

**ebtables** — like iptables but at **layer 2** (Ethernet frames). Used here for
bridging/isolation rules; see `ebtables.txt`.

**conntrack (`ct_tcp.txt`)** — the kernel's **connection tracking** table: every
active TCP flow the NAT is following. A window into what's talking through the box
right now.

**PHY (`.dbg.phy.log`)** — the **Ethernet PHY** driver (the chips driving the
copper LAN ports). Lines like `phy_set_EthMode … speed=4` record link
speed/duplex negotiation. Unrelated to Wi-Fi "PHY."

**dmesg** — the kernel's boot/runtime ring buffer (`dmesg.txt`): driver init,
hardware detection, kernel warnings.

**watchdog** — a hardware timer that reboots the box if software stops petting it
(i.e. hangs). One reboot in this dump was watchdog-triggered; see doc 10.

**bootbase** — the SoC's **bootloader** (the EcoNet/Airoha "free bootbase" here).
Runs before Linux: inits DRAM/flash, then loads the kernel. Its serial output and
boot-command prompt are in `Boot.log`; see doc 12.

**A/B images (`tclinux` / `tclinux_slave`)** — two full firmware banks in flash. If
one fails to boot, the loader falls back to the other — a **failsafe** so a bad
update or crash can't brick the box. Tracked by `zeroman/imageCounter`.

**SPI NAND / BMT / BBT** — the flash type (`MT29F2G01`, 256 MB) and its **B**ad-
**B**lock **M**anagement / **B**ad-**B**lock **T**able. NAND cells wear out, so the
controller remaps bad blocks; you'll see `BMT & BBT Init Success` at boot.

---

## D. Management / TR-069 world → see [07-management-tr069.md](07-management-tr069.md)

**TR-069 / CWMP — CPE WAN Management Protocol**
*What:* the industry standard by which an ISP remotely manages your router.
*Does:* lets the ISP's server read/set parameters, push firmware, reboot, and
run diagnostics — over the WAN.
*How:* your router (the **CPE**) periodically calls home to the **ACS** using SOAP
over HTTP(S).

**ACS — Auto Configuration Server** (`cfgcli_ManagementServer.txt`)
*What:* the ISP-side TR-069 server.
*How:* `InternetGatewayDevice.ManagementServer.*` holds its URL, credentials, and
the "**Inform**" interval (how often the CPE checks in).

**CPE — Customer Premises Equipment** — the TR-069 word for "this device."

**TR-181 / TR-098 data model**
*What:* the standardized *tree* of parameters TR-069 manipulates.
*How:* every `InternetGatewayDevice.…` path in the `cfgcli_*` files is a node in
this tree. `X_ASB_COM_…` / `X_ALU-COM_…` prefixes are **vendor extensions**
(ASB/ALU = Alcatel) — non-standard parameters Nokia added.

**cfgcli**
*What:* the on-device CLI that dumps/edits the TR-181 data model.
*How:* every `cfgcli_*.txt` file is the output of querying one subtree — e.g.
`cfgcli_DeviceInfo.txt` is `cfgcli` printing `InternetGatewayDevice.DeviceInfo.*`.

**Connection Request / XMPP** (`cfgcli_XMPP.txt`, `ls_tr069.txt`)
*What:* the mechanism for the ACS to reach the CPE *between* scheduled check-ins.
*Problem it solves:* the CPE is usually behind NAT, so the ACS can't just connect
to it. *Solution:* the CPE keeps a persistent **XMPP** chat connection open to a
broker; the ACS sends an XMPP message that says "call me now," and the CPE
immediately does a TR-069 Inform.

**NWCC — Nokia cloud controller endpoints** (`cfgcli_…NWCC.txt`)
*What:* URLs for Nokia's cloud services (`L1Server`, `L2Server`,
`RemoteMobileAccess` — e.g. a phone app for remote management).
*State here:* the URL fields are **empty**, so cloud management is **not
currently active** on this unit — it's ISP/ACS-managed, not Nokia-cloud-managed.

---

## E. Nokia/Alcatel naming tics you'll keep seeing

| Token | Meaning |
|-------|---------|
| `ALCL`, `ALU`, `ASB`, `X_ALU-COM_`, `X_ASB_COM_` | Alcatel-Lucent / Alcatel-Shanghai-Bell vendor tags. `X_…_COM_` = a vendor extension in the data model. |
| `AONT` | The syslog hostname/tag of this ONT ("**A**lcatel **ONT**"). Every `messages` line carries it. |
| `3FE…` | Nokia part-number scheme (hardware and software builds). |
| `nwf`, `nwfa` | Nokia Wi-Fi (agent) — the mesh software family USSA/NVA live in. |
| `ussa`, `nva`, `ecd`, `aci` | The mesh daemons (steering / node-view / edge-controller / AP-config-interface). |
| `sbell` | Shanghai-Bell, the Nokia JV that authors much of this firmware (seen in commit emails). |

Next: [03-architecture.md](03-architecture.md) — how these actually wire together.
