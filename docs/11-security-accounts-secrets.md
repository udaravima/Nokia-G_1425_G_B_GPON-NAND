# 11 · Security, Accounts & Secrets — Who Can Log In, and What's Exposed

This doc exists because of one sharp observation: *the `administrator` account uses
`vtysh` instead of a normal shell.* Pulling that thread unravels the device's whole
identity and secrets model — and it's worth understanding in full, because the
design has a subtlety that's easy to misread as "safe" when it isn't.

Everything here is read from `logs/configs_key_data_bak/etc/{passwd,shadow}` and the
config tree. No password hashes or credentials are reproduced — only their
*properties*.

## The account model — and the trap hiding in it

From `etc/passwd` and `etc/shadow`:

| Account | UID | Login shell | Password | What it is |
|---------|:--:|-------------|----------|------------|
| `root` | **0** | `/bin/false` | MD5-crypt | The classic superuser — but shell is `/bin/false`, so **no direct interactive login**. |
| `appService` | 1100 | `/bin/false` | locked (`!`) | A service account. Can't log in at all. |
| `ONTUSER` | **0** | `/bin/sh` | MD5-crypt | **A second, full root.** UID 0 + a real shell = complete control. The factory/ISP superuser. |
| `administrator` | **0** | `/usr/sbin/vtysh` | MD5-crypt | **Also full root** (UID 0), but dropped into the restricted `vtysh` CLI. The user-facing admin. |

**The trap — read this twice.** Three accounts are **UID 0**. In Unix, the *name*
of an account is decoration; the **UID is the identity**. UID 0 *is* root,
whatever you call it. So `ONTUSER` and `administrator` are not "lesser" accounts —
they are root wearing different clothes. The *only* thing separating
`administrator` from a full root shell is its login program, `vtysh`. That's a
**cage around a root account**, not a reduction of privilege.

Why that matters: if `vtysh` has any escape hatch — a `!command` passthrough, a
`ping`/`traceroute` that takes shell metacharacters, a pager (`more`/`less`) that
can spawn a subshell, a "debug" or "configure" mode that runs system commands —
then the moment you use it, you're **UID 0 root**, because that's what the account
already is. The blast radius of any `vtysh` weakness is *total*, not *limited*.
Contrast that with the *right* way to build a restricted admin: a **non-zero UID**
plus a restricted shell, so even a full escape lands you as an unprivileged user.
This device chose convenience (one all-powerful identity, caged by UI) over
defense-in-depth.

## What `vtysh` actually is

*What it is:* `vtysh` is the **integrated VTY (Virtual TeletYpe) shell** from the
**FRRouting / Quagga** routing suite — an all-in-one, Cisco-IOS-style command-line
interface. Originally its job was to let an operator configure routing daemons
(`bgpd`, `ospfd`, `zebra`…) through one familiar `enable` / `configure terminal`
prompt instead of editing each daemon's socket.

*What it does here:* vendors repurpose a `vtysh`-style binary as a **restricted
management CLI** — a menu of "safe" commands (show status, set Wi-Fi, reboot) so
that a support tech or the end user gets a curated interface instead of a raw
BusyBox shell. When `administrator` logs in (SSH/telnet/serial), the kernel runs
`/usr/sbin/vtysh` as their shell, and they interact with that menu.

*How the restriction works — and its limit:* the "restriction" is entirely inside
the `vtysh` program: it parses your input against a command table and refuses
anything not on it. It is **application-level allow-listing, not an OS sandbox.**
There's no `chroot`, no seccomp, no dropped capabilities implied by it. It holds
only as long as the binary has no gap — and as established above, on this box any
gap yields root. Treat `vtysh` as a *usability* feature that happens to have
security-adjacent effects, not as a security boundary you can lean on.

> **Note the `.bashrc` curio.** `home/administrator/.bashrc` contains the line
> `export PS1="[\u@\h: \W]\$ "` repeated **~150 times** — the same line appended on
> every boot/login by some provisioning script that never de-duplicates. It's
> harmless (and mostly moot, since the login shell is `vtysh`, not bash), but it's
> a nice tell of embedded-firmware sloppiness: append-only config generation with
> no idempotency. If you ever *do* reach bash as `administrator`, that's the
> environment you'd land in.

## Password hashes: weak by modern standards

All three active accounts hash with **`$1$` = MD5-crypt**. MD5-crypt is *salted*
(so no rainbow tables) but **computationally cheap** — a modern GPU tries billions
of candidates per second against it. For comparison the same `crypt` family offers
`$6$` (SHA-512, thousands of rounds) and `$y$` (yescrypt, memory-hard). MD5-crypt
is what Linux used ~2000–2008.

*Stakes:* if any of these hashes leaks (and one just did — it's sitting in this
dump), a weak/default password is crackable in minutes to hours. This is the
concrete reason the "treat the dump as secret" warning isn't boilerplate. **If
these are still factory-default credentials, that's the single highest-value
hardening action on the whole box:** change them, ideally to something long enough
that even fast MD5-crypt can't be brute-forced.

## The encrypted config blob — why you couldn't read it

`config_encryption.cfg` (and its `.bak`) is **144,749 bytes of `data`** with:

- Header magic `24 41 12 00` (`$A` + flags), then a length-looking field.
- **Entropy 7.999 bits/byte across all 256 possible byte values.**

That entropy is the tell. Real text/config compresses; it doesn't fill all 256
byte values near-uniformly. **7.999/8.0 means the bytes are indistinguishable from
random — i.e. encrypted** (a good cipher's output is random-looking by design).

*Can you decrypt it?* **No — and neither can I, from this dump.** The key isn't
here. On these ONTs the config-encryption key is derived **on the device** (from a
per-unit secret in one-time-programmable/secure storage, or a vendor master key
baked into the bootloader/firmware). That's deliberate: it stops a downloaded
config backup from being read, edited, or transplanted onto another unit. The `$A`
magic is Nokia's container format; the exact cipher (almost certainly AES in some
mode) can't be *proven* from the ciphertext alone — that part is inference.

*The irony worth noting:* the blob protects the **portable** form of the config,
but this diagnostic dump **also contains the plaintext originals right next to it**
— `subscriber_data.txt`, the VoIP XMLs, `client_specific.txt`, etc. So the locked
box is sitting beside an unlocked copy of its contents. You didn't need to decrypt
it; the cleartext is in the same directory. That's not a flaw in the encryption —
it's just what a *debug* dump is: everything, including the sources.

> **Don't confuse it with `config.cfg`.** The web-UI *Backup Configuration* feature
> produces a **separate** file (`workdir/config.cfg`) in a **different** container
> format (plaintext header, magic `23 31 12 00`) whose **payload is also encrypted**
> — but with a scheme the public decode tool predates (this is a Dec-2024 build).
> Two files, two crypto schemes; neither is readable with what's on hand. Full
> write-up in [doc 12](12-boot-chain-and-access.md).

## Secrets inventory — what actually holds credentials

If this dump ever moved off your machine, these are the files that matter (all under
`configs_key_data_bak/`), found by scanning for credential keywords:

| File | Holds |
|------|-------|
| `alcatel/config/subscriber_data.txt` | Subscriber/service provisioning (line identity, service params). |
| `alcatel/config/DefaultVoIPConfig.xml`, `default_voip.txt`, `voip_generic.txt` | **VoIP/SIP** account config — SIP username/password/registrar. |
| `alcatel/config/ftpprofile.xml` | **FTP** profile credentials. |
| `alcatel/config/client_specific.txt`, `dm_map_value.bak` | Operator-specific provisioning + data-model value backup. |
| `etc/config/apcloud`, `plasmodium`, `rpcd` | Mesh-cloud, MQTT, and RPC-login config (endpoints, possibly tokens). |
| `etc/ipsec.d/private/*`, `certs/*`, `cacert.pem` | **IPsec private keys & certificates.** |
| `etc/{passwd,shadow}` | The account hashes discussed above. |
| `config_encryption.cfg` | The encrypted superset of most of the above. |

Plus, outside the backup tree: `dhcp.leases`, `ussa.log`, and the Wi-Fi
`Wireless_RT2860AP*.dat` (SSIDs + PSKs in cleartext) carry client identities and
Wi-Fi keys.

## The other things you were missing: outbound channels & boot state

Sweeping `etc/config/` surfaced the device's **telemetry and cloud plumbing** —
worth knowing because these are *outbound* connections the box makes on its own:

| Config | What it is |
|--------|-----------|
| `plasmodium` | An **MQTT** client (`config mqtt`). MQTT is a lightweight publish/subscribe protocol — a persistent *outbound* channel, exactly the "tunnel as a doorbell" pattern from doc 07. Likely a telemetry/remote-management path (or the substrate NWCC would use if enabled). |
| `nemo` | A **statistics/telemetry** agent (`config stats`) — Nokia's analytics collector. |
| `fonendoscope` | Diagnostics module — the name is Spanish for *stethoscope*: it "listens" to the network/Wi-Fi for health data. |
| `ookla` | Integration for **Ookla Speedtest** (the built-in speed test). |
| `apcloud` | Mesh **AP-cloud** coordination config. |
| `rpcd` | The **RPC daemon login/ACL** config — governs who may call which ubus/JSON-RPC methods (the web UI & TR-069 go through here). Worth reading to see the exposed method surface. |

And `zeroman/` is the **A/B image / failsafe-boot manager**: `imageCounter = 2`
tracks boot/image state across the dual firmware banks (this ties into the reboot
story in doc 10 — a box that fails to boot an image bank gets flipped to the other).

## Practical hardening shortlist (for your lab box)

1. **Change the account passwords** if they're factory defaults — MD5-crypt makes a
   weak/default password trivially crackable once a hash leaks.
2. **Assume `administrator` == root.** Don't treat the `vtysh` cage as a security
   boundary; if you're probing it in the lab, *that's* the interesting attack
   surface (look for command passthrough / pager escapes — the classic vtysh
   weaknesses), precisely because escape = UID 0.
3. **Keep this dump local** (you already are). It contains hashes, IPsec private
   keys, SIP and FTP credentials, and Wi-Fi PSKs in cleartext.
4. **Know your outbound channels.** `plasmodium` (MQTT) and `nemo` are the box
   phoning home; if you want to understand what leaves the device, those two configs
   plus `rpcd`'s exposed method list are where to look.

Related: doc 06 (the OS layer) for how these accounts/configs fit the OpenWrt model,
and doc 07 for the TR-069/cloud management picture the telemetry configs plug into.
