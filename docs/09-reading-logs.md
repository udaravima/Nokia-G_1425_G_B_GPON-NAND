# 09 · Reading The Logs — Formats, Severity, Time, and Grep Recipes

The dump uses at least five different log formats, three timestamp styles, and a
pile of rotation suffixes. This doc is the field manual: learn to parse one line
of each format, decode the severity, get the timezone right, and you can navigate
any file in the bundle. It ends with copy-paste grep recipes tuned to *this* box.

## 1. The five line formats you'll meet

### (a) System syslog — `messages`, `customer`, `err`
This is **RFC 5424 syslog**, the most information-dense format here:

```
[err] <155>1 2026-08-15T08:53:23.029155+05:30 AONT cfgmgr 1956 - - cfgXml_getObject:2619:… failed
 └┬─┘ └─┬┘│ └──────────┬──────────────┘ └┬─┘ └─┬──┘ └┬─┘ �│ �│ └────────── message ───────────┘
  │     │ │            │                 │    │     │   │ └ structured-data (—none)
  │     │ │            │                 │    │     │   └ message-id (—none)
  │     │ │            │                 │    │     └ process ID (PID)
  │     │ │            │                 │    └ app-name / daemon (who logged it)
  │     │ │            │                 └ hostname (always AONT = this ONT)
  │     │ │            └ timestamp with timezone offset (+05:30)
  │     │ └ syslog version (1)
  │     └ PRI = facility×8 + severity  (see §2)
  └ human severity tag
```

The two fields you'll use constantly: **app-name** (`cfgmgr`, `omciMgr`, `dhcpd`
— *which subsystem*) and the **severity tag** (`[err]`, `[warning]`, `[notice]`).

### (b) The AI controller — `ai_controller.log`
```
[000001] Sat Aug 15 08:54:12 | DEBUG | aistages/learning/learning.cpp | get_min_20m_free_ch_utilization:212 channel_low_samples 6
 └──┬──┘ └──────┬──────┘   └─┬─┘   └────────┬────────┘   └──────────┬──────────────┘
 line seq     timestamp    severity     source file            function:line + message
```
The `[NNNNNN]` is a **monotonic line counter** — handy for spotting gaps/resets.

### (c) USSA — `ussa.log`
```
USSA > Sat Aug 15 08:33:36 | INFO | source/ussa_interface.c | ussa_show_object:293 | mac is […], rssi=-47, …
```
Same `pipe-delimited` shape as the AI logs (Nokia's `nwfa` house style), prefixed
with `USSA >`.

### (d) OMCI — `history/omci.log`
```
[08-08 20:41:58][E]checkPonLEDStatus: Ploam State changed to ~O5 …
 └────┬────┘ └┬┘
 MM-DD HH:MM:SS  severity letter: [E]rror [C]ritical [W]arn (none)=info
```
**No year, no timezone** — you infer the year from context. Severity is a single
bracketed letter.

### (e) PHY / ad-hoc — `.dbg.phy.log`, `ddm_log.log`
```
[2026-08-14 21:53:44 +0530]phy_set_EthMode: …        ← PHY: full local timestamp
08-15 02:37:00 info: unable to parse json …          ← ddm: MM-DD + "info:" prefix
```
These are freer-form; each daemon rolled its own.

## 2. Decoding severity (and the PRI number)

The `<155>` in syslog is the **PRI**: `PRI = facility × 8 + severity`. Severity is
the low 3 bits, and it's the same 0–7 scale everywhere:

| Sev | Name | In this dump you'll see | Care level |
|:--:|------|-------------------------|-----------|
| 0 | emerg | — | drop everything |
| 1 | alert | — | urgent |
| 2 | crit | `[crit]` — `restarting ndk_udhcpd` | notable |
| 3 | err | `[err]` — the `cfgXml … failed` storm | investigate, but see doc 10 |
| 4 | warning | `[warning]` — PON RX power alarms | watch |
| 5 | notice | `[notice]` — DHCP, normal ops | routine |
| 6 | info | `[info]` | routine |
| 7 | debug | `DEBUG` (AI logs) | firehose — filter out |

Worked example: `<155>` → 155 ÷ 8 = 19 remainder 3 → **facility 19** (a local
facility), **severity 3 = err**. You rarely need the facility; severity is the
signal.

**A crucial caveat (doc 10 expands it):** on this firmware `[err]` is massively
over-used for benign conditions. Severity tells you the daemon's *opinion*, not
ground truth. Count and cluster before you panic.

## 3. Time — the one setting that will bite you

- **Local time is UTC+05:30** (`cfgcli_Time.txt`; Sri Lanka / IST). Every
  `+05:30` timestamp is already local.
- **The dump folder name ends in `Z` (`…085412Z`)** — that suffix means UTC. So
  the folder says 08:54:12 **UTC**, which is **14:24 local**… except `uptime.txt`
  captured inside it reads `08:54:22` local. The `Z` in the *folder name* is
  Nokia's tooling being sloppy — the contents are local. **Trust the in-file
  `+05:30` timestamps, not the folder's `Z`.** (A perfect example of why you state
  timezones explicitly.)
- **OMCI logs have no year** — cross-reference with `messages` (which does) when
  building a timeline across both.

## 4. Rotation & compression — reading the suffixes

Logs rotate so they don't fill flash. The naming tells you the age order:

| Suffix | Meaning | Age |
|--------|---------|-----|
| `foo` | current, being written | newest |
| `foo.0`, `foo.1` … `foo.9` | previous generations | `.0` newer than `.9` |
| `foo.old` | the single previous copy | older |
| `foo.bak` | a backup snapshot | varies |
| `foo.tar.bz2`, `foo.gz` | compressed archives | oldest |

So the true chronological order of the AI log is `…log.9 → log.8 → … → log.1 →
log.0 → log` (**counter-intuitive: higher number = older**). To read a compressed
one without unpacking to disk:

```
bzcat logs/messages_kern.0.tar.bz2 | less      # .tar.bz2 → often a bare file inside
zcat  logs/history/ussa.log.old.gz | less      # .gz
```
(For a real `.tar.bz2` use `tar xjf …`; several here are single files just given a
`.tar.bz2` name — `bzcat` reveals which.)

## 5. Grep recipes tuned to this box

```bash
cd tmp

# — What's actually wrong? Errors & criticals in the main log, by daemon —
grep -E "^\[(err|crit|alert|emerg)\]" logs/messages | awk '{print $6}' | sort | uniq -c | sort -rn
#   → ranks which daemon is emitting the most errors (cfgmgr will dominate; doc 10)

# — Silence the AI firehose, keep real decisions —
D=G-1425G-B_fc429d49_MESH_20260815T085412Z
grep -vE "channel_low_samples|get_min_20m_free_ch_utilization" $D/ai_controller.log | less

# — Fiber outages: did the link drop, cleanly or flapping? —
grep -iE "Ploam|fiber(Status)?|O5|disconnect" logs/history/omci.log

# — Cross-check a "drop" against power loss —
grep "Last Reboot reason" logs/reboot.log | tail -20

# — A specific client's Wi-Fi life (replace the MAC) —
grep -i "aa:bb:cc:dd:ee:ff" $D/ussa.log $D/ussawifievent.log

# — Who is talking through the box right now —
cat $D/ct_tcp.txt | awk '{print $0}' | head        # live TCP flows
cat $D/netstat_anp.txt                             # listening services

# — The mesh at a glance (no grep needed) —
python3 -m json.tool $D/topology.txt   # pretty-print the JSON after the HTTP header
```

> Tip for the HTTP-response files (`topology.txt`, `aps.txt`): they begin with
> `HTTP/1.1 200 OK` headers. Strip them before JSON parsing:
> `sed '1,/^\r\{0,1\}$/d' aps.txt | python3 -m json.tool`.

## 6. The reading strategy, distilled

1. **Triage by severity, ranked by daemon** (recipe 1) — don't read top-to-bottom.
2. **Cluster before concluding** — 270 identical `[err]` lines is *one* problem,
   not 270 (doc 10).
3. **Two clocks** — keep local `+05:30` vs. the folder's bogus `Z` straight.
4. **Filter the firehose** — the AI DEBUG stream is heartbeat, not history.
5. **Prefer the snapshots** — `topology.txt`, `aps.txt`, `sysinfo.log` hand you
   state without archaeology.

Next: what all this reading actually *found* — [10-observations.md](10-observations.md).
