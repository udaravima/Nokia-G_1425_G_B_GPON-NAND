# 08 · File Index — The Whole Dump, Mapped

A lookup table for all 262 files. Use it the way you'd use an index at the back of
a textbook: find the file you're staring at, get one line on *what it is* and
*when you'd open it*, then jump to the deep-dive doc for context.

The dump has three roots:

- **`tmp/*.txt`** — top-level TR-069 data-model dumps (5 files).
- **`tmp/G-1425G-B_fc429d49_MESH_…/`** — the Wi-Fi **mesh** diagnostic bundle
  (104 files).
- **`tmp/logs/`** — the **system** log + config-backup bundle (the rest).

Legend for the "Read?" column: **★** = start here / high signal · **○** = read on
demand · **·** = archive/rotation/binary, rarely opened directly.

---

## Root 1 — `tmp/` top level (TR-069 data model) → doc 07

| File | Read? | What it is |
|------|:---:|-----------|
| `cfgcli_DeviceInfo.txt` | ★ | Device identity: model, serial, HW/SW versions, uptime. |
| `cfgcli_ManagementServer.txt` | ★ | The **ACS** (TR-069 server) URL, Inform interval, credentials. |
| `cfgcli_XMPP.txt` | ○ | XMPP Connection-Request ("doorbell through NAT") config. |
| `cfgcli_Time.txt` | ○ | NTP servers + timezone (+05:30). |
| `cfgcli_InternetGatewayDeviceX_ALU-COM_NWCC.txt` | ○ | Nokia cloud endpoints — **empty here** (cloud off). |

---

## Root 2 — the MESH bundle → docs 05, 03

### Human-readable snapshots (start here)
| File | Read? | What it is |
|------|:---:|-----------|
| `topology.txt` | ★ | JSON map of the whole mesh: APs, roles, IPs, backhaul, cost-to-gateway. **Best single file.** |
| `aps.txt` / `debug_aps.txt` | ★ | Per-AP + per-client detail: capabilities, RSSI, steering state. |
| `map_role.txt` | ★ | This node's Multi-AP role → **Controller**. |
| `buildinfo.txt` / `release_note` / `edge-controller_version.txt` | ★ | Firmware build + component versions. |
| `uptime.txt` / `meminfo.txt` / `proc_statm.txt` / `top10times.txt` | ★ | Health: load, memory, per-process usage. |
| `ps.txt` / `ps_*.txt` | ○ | Process list (142 procs) + per-daemon `ps` (nva, ussa, ubusd, nwhed, nwsl). |

### The AI / RRM control loop
| File | Read? | What it is |
|------|:---:|-----------|
| `ai_controller.log` + `.0`–`.9` | ○ | AI channel-selection brain. **87k lines, mostly DEBUG** — filter before reading (doc 05). |
| `ai-agent.log` + `.old` | ○ | Applies channel/steering decisions to the radios. |
| `ecd_config_db.json` / `ecd_node_info_db.json` / `ecd_dbg_db.json` | ○ | Edge-controller internal state DBs. |
| `localrrm_agent_startup.log` / `localrrm_controller_startup.log` | ○ | Local RRM startup traces. |
| `aci.txt` | ○ | AP-config-interface dump. |

### Steering, scanning, channels
| File | Read? | What it is |
|------|:---:|-----------|
| `ussa.log` / `.old` | ○ | Per-client health snapshots (RSSI, airtime, rates). |
| `ussawifievent.log` / `ussawifistatus.log` | ★ | Steering & **disassoc events** — read for "my device keeps dropping." |
| `ussa.conf` / `ussa_ReleaseNotes.txt` | · | USSA config + changelog. |
| `scan_result_ra0.txt` / `scan_result_rai0.txt` | ○ | What each radio's scan saw (neighbor APs). |
| `neighbor_info_ra0.txt` / `neighbor_info_rai0.txt` | ○ | Neighboring-AP interference detail. |
| `channel_utils_ra0.txt` / `channel_utils_rai0.txt` | ○ | Channel-busy (utilization) measurements. |
| `historychan.txt` / `historychop.txt` | ○ | Channel-change + channel-hop history. |
| `acs_report_ra0.txt` / `acs_report_rai0.txt` | ○ | Auto-Channel-Selection decision reports. |
| `blocked_channels.txt` | ○ | Channels off-limits (DFS/radar). |
| `debug_steer.txt` / `debug_bh_steer.txt` / `debug_graph.txt` / `debug_map_cfg.txt` / `debug_fh_mgmt.txt` | ○ | Steering/backhaul/topology/config debug traces. |
| `mesh_topo_dump.txt` / `meshctl_dump.txt` / `ri_dump.txt` | ○ | Mesh topology dumps (`mesh_topo_dump` **empty** — single node). |
| `nva_node_info` / `nva_cli_tool.txt` | ○ | Network-View-Agent identity/state. |
| `map_role.txt` / `nwf-work-mode-set.txt` / `cfgcli_WorkMode.txt` / `cfgcli_WorkRole.txt` | ○ | Role/mode settings. |

### Wi-Fi driver config & radio
| File | Read? | What it is |
|------|:---:|-----------|
| `uci_show_wireless.txt` / `uci_show.txt` | ○ | OpenWrt UCI config (wireless + everything). |
| `Wireless_RT2860AP.dat` / `Wireless_RT2860AP_AC.dat` | ○ | Raw MediaTek Wi-Fi driver config (SSIDs, keys, channels — **sensitive**). |
| `nwf-product-cfg.json` / `nwf-product-service-capability.json` | ○ | Product capability manifests. |

### System/network within the bundle → doc 06
| File | Read? | What it is |
|------|:---:|-----------|
| `dmesg.txt` | ○ | Kernel ring buffer. |
| `ifconfig.txt` / `ifconfig_a.txt` / `ip_link.txt` | ○ | Interfaces, state, counters. |
| `brctl_show.txt` / `brctl_showmacs.txt` / `brctl_showstp.txt` | ○ | Bridge membership, learned MACs, STP. |
| `ebtables.txt` | ○ | Layer-2 firewall rules. |
| `ct_tcp.txt` | ○ | Live tracked TCP connections (conntrack). |
| `netstat_anp.txt` | ○ | Listening sockets + owning process. |
| `dhcp.leases` | ○ | Current DHCP leases (client MAC/IP/hostname — **sensitive**). |
| `resolv.conf` / `root_ip.txt` / `rest_listen_port` | · | DNS servers, root IP, REST port. |
| `ndbus.log` / `.old` | ○ | Nokia D-Bus/IPC log. |
| `ussa.log`… (listed above) | | |
| `image_flag_check.txt` / `ls_download.txt` / `ls_tr069.txt` | · | Boot image validity, download dir, TR-069 client state. |
| `internetstatus.txt` / `cfgcli_dumpwan.txt` | ○ | Internet up/down + WAN config. |
| `actions.txt` / `buildinfo.txt` / `release_note` | · | Diagnostic action list + build metadata. |

---

## Root 3 — `tmp/logs/` (system logs + backups) → docs 04, 06

### The main system logs
| File | Read? | What it is |
|------|:---:|-----------|
| `messages` | ★ | The primary syslog — all daemons, all severities. 1.6 MB. **The main event log.** |
| `messages_component` / `.0/.1.tar.bz2` | ○ | Per-component syslog split + rotations. |
| `messages_kern` / `.0.tar.bz2` | ○ | Kernel-only messages. |
| `customer` | ○ | Operator/"customer"-facing event log. 1.7 MB. |
| `sysinfo.log` | ★ | Periodic full system snapshot (incl. service/optical status). |
| `err` / `sec` / `selog` / `selog2` | ○ | Error log + security/SELinux-style event logs. |

### Optical / GPON → doc 04
| File | Read? | What it is |
|------|:---:|-----------|
| `history/omci.log` / `omci.log.bak` | ★ | **PLOAM states + fiber up/down.** First stop for outages. |
| `omci_com_fail` / `omci_com_fail.0` | ○ | OMCI comms failures + chip/diag init. |
| `bosa_log.0/1.tar.bz2` | · | Optical sub-assembly logs (compressed). |
| `.dbg.phy.log` / `.dbg.phy.bak` | ○ | **Ethernet** PHY link negotiation (not optical, despite "phy"). |
| `ddm_log.log` / `ddm_0.log` / `ddm_btrace.log` | ○ | **Wi-Fi** daemon logs — *not* optical DDM (name trap, doc 04). |

### Boot, reboot, login, security → doc 06
| File | Read? | What it is |
|------|:---:|-----------|
| `reboot.log` / `.last` / `reboot.flag` | ★ | Every reboot with a **reason** (75 Powerdown, 1 watchdog). |
| `pstore/console-ramoops-0` | ★ | **Kernel black box** — the console buffer persisted across the last reset. Confirmed present: captures the watchdog-reset trailer (`rt28xx_ioctl` frame) + a GPON SF/SD alarm storm. See doc 10, Finding 1. |
| `login.log` / `login_fail_count.log` / `login_last_succ.log` / `app_nor_acc_fail_login_cnt` | ○ | Login attempts, failures, last success — check for intrusion. |
| `nbus_fatal.log` / `ndbus_fatal.log` / `*_note.log` (+ `.old`) | ○ | Nokia bus daemon fatal/notice logs. |
| `dmdb.log.old` | · | Data-model DB log (rotated). |

### History & service logs
| File | Read? | What it is |
|------|:---:|-----------|
| `history/ussa.log` / `ussawifievent.log` / `ussawifistatus.log` (+ `.gz`) | ○ | Longer-horizon Wi-Fi steering history. |
| `parentCtl.log` / `parentCtl-…bak` | ○ | Parental-control events. |
| `voip/` | ○ | Voice/SIP logs. |
| `autoPortBindStatus.log` | · | Port-binding status. |
| `wan_func.trc` / `.bak` / `.exit` | ○ | WAN function trace. |
| `store_previous_ch_bw.txt` / `tr069_swver` / `rsyslog_cmd_config` | · | Saved channel/BW, TR-069 SW version, syslog config. |

### Config & key backups → doc 06 (**treat as secrets**)
| Path | Read? | What it is |
|------|:---:|-----------|
| `configs_key_data_bak/etc/config/*` | ○ | UCI configs (network, wireless, dhcp, firewall, mesh). |
| `configs_key_data_bak/alcatel/config/*` | ○ | Nokia config: `subscriber_data.txt`, VoIP, hardware/generic profiles. |
| `configs_key_data_bak/etc/ipsec.*` + `ipsec.d/{certs,private,cacerts}` | · | **VPN certs & private keys.** |
| `configs_key_data_bak/{cacert.pem, config_encryption.cfg, zeroman/}` | · | Certs + config-encryption key material. |
| `configs_key_data_bak/bosa/*.bob`, `bin_ac/*.bin/.dat` | · | Optical + radio **calibration** blobs (binary). |
| `configs_key_data_bak/tr069_conf/`, `home/administrator/` | · | TR-069 config + admin home. |

---

### The "if I only open five files" shortlist
1. `topology.txt` — the mesh at a glance.
2. `logs/messages` — the main event stream.
3. `logs/history/omci.log` — the fiber's health.
4. `reboot.log` — why it restarted.
5. `cfgcli_DeviceInfo.txt` — what the box is.

Next: the practical skills to actually *read* these — [09-reading-logs.md](09-reading-logs.md).
