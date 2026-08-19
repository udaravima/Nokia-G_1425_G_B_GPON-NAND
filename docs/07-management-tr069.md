# 07 · The Management Layer — TR-069, the Data Model, and cfgcli

Your ISP can change this router's Wi-Fi password, push new firmware, and run a
line test — all without anyone touching the box. That power runs on a 20-year-old
but still-ubiquitous standard called **TR-069**. This layer is also where most of
the `cfgcli_*.txt` files come from, so understanding it makes a whole class of the
dump legible at once.

## TR-069 in one breath

**TR-069** (a.k.a. **CWMP**, CPE WAN Management Protocol) is how an operator
manages millions of home routers from one server. The cast:

- **CPE** — Customer Premises Equipment = *this device*.
- **ACS** — Auto Configuration Server = the *ISP's* management server.
- The CPE periodically opens a connection to the ACS and sends an **Inform**
  (SOAP/XML over HTTPS): "here's who I am, here's my state." The ACS replies with
  any changes to make.

That's the whole heartbeat: **CPE calls home on a timer, ACS answers with
instructions.** The timer ("PeriodicInformInterval") and the ACS URL live in
`cfgcli_ManagementServer.txt`.

## The genius trick: the Connection Request (and how CGNAT shapes it)

There's a hole in "call home on a timer": what if the ISP needs to reach the box
*right now* (you called support, they want to reboot it)? They can't just connect
to you — and on this line it's worse than ordinary NAT. The verified facts from
`cfgcli_ManagementServer.txt` + `cfgcli_dumpwan.txt`:

- ISP: **Sri Lanka Telecom** — `URL = http://acs.slt.lk`, username `acs@slt`.
- WAN: **PPPoE on VLAN 20** (`Name=1_INTERNET_R_VID_20`), IPv4 **100.89.81.114**.
- That `100.89.x` is in **100.64.0.0/10 — carrier-grade NAT (CGNAT)** space. Your
  box has **no public IP at all**; it shares one with many subscribers.
- **`PeriodicInformEnable = false`**, interval 10800 s (3 h). So the timer heartbeat
  is *off* — the ACS depends entirely on Connection Requests to reach the CPE.

TR-069's answer is the **Connection Request**: a way for the ACS to say "call me
now." Because a CGNAT'd CPE has no reachable public address, there are two ways to
pull it off, and this box shows evidence of the first, config for the second:

1. **HTTP CR to a carrier-internal address (the populated one here).**
   `ConnectionRequestURL = http://100.89.153.5:7547`, `…Mode = Digest`. Note the
   `100.89.x` again — the CR URL lives **inside SLT's own CGNAT network**, the one
   place that *can* route to your `100.89.x` CPE. Port 7547 is the standard CWMP CR
   port. The ACS reaches you from inside the carrier, not from the public internet.
2. **An outbound tunnel as a doorbell (XMPP).** The CPE keeps a persistent
   outbound connection open; the ACS sends it a "call me now" message that rides
   back down that connection — classic NAT/CGNAT traversal.

```
 ACS ──► (1) HTTP CR to 100.89.153.5:7547  ─┐  (reachable only inside SLT's CGNAT)
                                            ├─► CPE ──► fires a TR-069 Inform
 ACS ──► (2) message down CPE's outbound  ─┘
             tunnel (XMPP doorbell)
```

> **Correction to an earlier draft of this doc:** I first stated the CR here is
> "carried over XMPP." That was an overclaim. `cfgcli_XMPP.txt` shows the XMPP
> object *exists* (`ConnectionNumberOfEntries = 1`) but carries **no visible
> server/credential config at this dump's depth**, so I can't confirm it's the live
> path. What *is* populated is the **HTTP** Connection Request above. XMPP is
> present-but-unconfirmed; the HTTP CR at the CGNAT address is the verified
> mechanism. `ls_tr069.txt` shows the TR-069 client's live state if you want to
> confirm which channel actually fired.

The transferable idea survives the correction: when a device can't be reached
directly (NAT, CGNAT, firewalls), you either **give the caller an address inside
the same walled network** (option 1) or **have the device hold a tunnel open and
ring back down it** (option 2). That pattern — outbound tunnel as a doorbell — is
everywhere: MQTT, WebSockets, push notifications, `ssh -R`.

## The data model: a giant standardized tree

Everything TR-069 reads or writes is a node in a standardized tree — the
**TR-181** (here presented in the older **TR-098** `InternetGatewayDevice.` root)
data model. A path like:

```
InternetGatewayDevice.LANDevice.1.WLANConfiguration.1.SSID
```

is literally "the SSID of the first Wi-Fi config of the first LAN device." The
whole box's settings are addressable this way. Two things to recognize:

- **`X_ALU-COM_…` / `X_ASB_COM_…` prefixes** mark **vendor extensions** — Nokia
  (Alcatel/Alcatel-Shanghai-Bell) parameters that aren't in the standard. The `X_`
  prefix is *required* by the spec for anything non-standard. So `X_ALU-COM_Wifi.
  WorkMode=RGW` is Nokia's private knob, not a standard TR-181 field.
- **OID + ObjectSize + Depth** headers in the `cfgcli` dumps (e.g. DeviceInfo is
  `OID=27, ObjectSize=36060, Depth=0`) are the *internal* representation cfgmgr
  uses — an object id and its size in the in-memory model.

## cfgcli: reading the model from the device

**`cfgcli`** is the on-box command that queries this data model. **Every
`cfgcli_*.txt` file in the dump is one subtree printed out:**

| File | Subtree | What it reveals |
|------|---------|-----------------|
| `cfgcli_DeviceInfo.txt` | `DeviceInfo.` | Identity: model, serial, HW/SW versions, uptime. (Doc 01's source.) |
| `cfgcli_ManagementServer.txt` | `ManagementServer.` | The **ACS** URL, Inform interval, TR-069 credentials. |
| `cfgcli_XMPP.txt` | `XMPP.` | The Connection-Request doorbell config. |
| `cfgcli_Time.txt` | `Time.` | NTP servers + timezone (this box: **+05:30**, IST/Sri Lanka). |
| `cfgcli_…NWCC.txt` | `X_ALU-COM_NWCC.` | Nokia **cloud** endpoints (see below). |
| `cfgcli_WorkMode.txt`, `WorkRole.txt`, `WorkMode.txt` | `X_ALU-COM_Wifi.` | Router vs bridge mode; mesh role. |
| `cfgcli_dumpwan.txt` | WAN config | The internet connection setup (PPPoE/IPoE, VLANs). |

So if you ever want to know "what did the ISP provision," you read the relevant
`cfgcli_*` file — it's a direct window into the data model the ACS manipulates.

## NWCC: the cloud path that's currently switched off

`cfgcli_…NWCC.txt` holds Nokia's **cloud controller** endpoints — `L1Server`,
`L2Server`, and `RemoteMobileAccess` (e.g. a "manage my Wi-Fi" phone app):

```
[L1Server.]  URL=        ClientID=AC-60-6F-C3-B3-A0   Password=******
[L2Server.]  URL=        UserName=ac-60-6f-c3-b3-a0   Password=******
[RemoteMobileAccess.] URL=
```

**The URLs are empty.** That's the signal: this unit is **not** currently tied to
Nokia's cloud — it's managed the classic way, by the ISP's own **ACS over TR-069**.
The credentials are pre-seeded (the ClientID is just the device's MAC), ready to
activate if the operator ever points those URLs somewhere. Passwords are masked as
`******` in the dump — cfgcli redacts secrets on output, which is the right
default.

## Two management channels, one device — the recap

This is the distinction from doc 03, now with the daemon names attached:

| | Optical / service side | Router side |
|---|---|---|
| **Managed by** | ISP **OLT** | ISP **ACS** |
| **Protocol** | **OMCI** (over fiber) | **TR-069/CWMP** (over WAN) |
| **On-device owner** | `omciMgr` | `cfgmgr` (+ TR-069/XMPP clients) |
| **You can change it?** | No — pushed from OLT | Some, via web UI (ACS can override) |
| **In this dump** | `omci.log`, `omci_com_fail` | `cfgcli_*`, `ls_tr069.txt`, `tr069_swver` |

When you keep those two columns separate in your head, the daemon names in `ps.txt`
and the log filenames sort themselves — each belongs to one column.

Next: the master lookup table for the whole dump —
[08-file-index.md](08-file-index.md).
