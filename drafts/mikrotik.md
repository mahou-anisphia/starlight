---
title: MikroTik DoH + Network-Wide DNS Enforcement
description: Cloudflare DNS-over-HTTPS on the hAP (RouterOS 7.23.1) with all LAN clients forced through it.
---

## What this is

DNS-over-HTTPS (DoH) configured on the MikroTik hAP gateway (RouterOS 7.23.1,
LAN `192.168.88.1`, WAN `ether1`, LAN bridge `bridge`). The router is the sole
DNS resolver for the network and sends all upstream queries encrypted over
HTTPS to Cloudflare (`https://cloudflare-dns.com/dns-query`). Every client is
forced through it via DHCP, a port-53 NAT redirect, and WAN drop rules.

## Why

Plaintext DNS (UDP/TCP 53) leaks browsing activity to the ISP and is open to
MITM tampering. DoH wraps lookups in HTTPS so the path is encrypted and
validated. Enforcement at the router (rather than per-client) means devices
that ignore the handed-out DNS server, or hardcode their own resolver, still
can't escape — their plaintext queries are redirected back to the router and
anything on 53/853 leaving the WAN is dropped.

`verify-doh-cert=yes` is kept on deliberately: without it the router wouldn't
confirm it's actually talking to Cloudflare, reopening the exact MITM gap DoH
exists to close.

## Configuration

DNS subsystem:

```
/ip dns set servers=1.1.1.1,1.0.0.1
/ip dns set use-doh-server=https://cloudflare-dns.com/dns-query verify-doh-cert=yes
/ip dns set allow-remote-requests=yes
```

Static records so the router can resolve the DoH hostname locally (and so the
plaintext `servers=` fallback can eventually be dropped):

```
/ip dns static add name=cloudflare-dns.com address=104.16.248.249
/ip dns static add name=cloudflare-dns.com address=104.16.249.249
/ip dns static add name=cloudflare-dns.com address=2606:4700::6810:f8f9 type=AAAA
/ip dns static add name=cloudflare-dns.com address=2606:4700::6810:f9f9 type=AAAA
```

CA trust — the built-in CA store on 7.23.1 did **not** validate Cloudflare's
chain (see gotchas). The full curl.se bundle was imported to fix it:

```
/tool fetch url="https://curl.se/ca/cacert.pem"
/certificate import file-name=cacert.pem passphrase=""
# certificates-imported: 121
```

DHCP hands out the router as the only DNS server:

```
/ip dhcp-server network set [find] dns-server=192.168.88.1
```

Enforcement — redirect stray plaintext DNS back to the router:

```
/ip firewall nat add chain=dstnat protocol=udp dst-port=53 in-interface-list=LAN action=redirect to-ports=53
/ip firewall nat add chain=dstnat protocol=tcp dst-port=53 in-interface-list=LAN action=redirect to-ports=53
```

Drop any DNS (53) and DoT (853) leaving the WAN:

```
/ip firewall filter add chain=forward protocol=udp dst-port=53 out-interface-list=WAN action=drop
/ip firewall filter add chain=forward protocol=tcp dst-port=53 out-interface-list=WAN action=drop
/ip firewall filter add chain=forward protocol=tcp dst-port=853 out-interface-list=WAN action=drop
```

Interface lists (must be populated or the rules match nothing):

| List | Interface |
| ---- | --------- |
| LAN  | bridge    |
| WAN  | ether1    |

## Verification

```
:put [:resolve example.com]          # returns a Cloudflare IP (e.g. 104.20.x.x) with DoH on
/ip dns cache print                   # cache populates over DoH
/log print where topics~"dns"         # no "DoH server connection error" lines
/ip firewall nat print stats          # dstnat redirect packet counts rising = strays caught
/tool torch interface=ether1 port=53  # no port-53 traffic leaving WAN
```

Client-side check: visit `https://1.1.1.1/help` → "Using DNS over HTTPS (DoH): Yes".

## Gotchas / notes

- **The CA trust failure is the main trap.** RouterOS 7.23.1's built-in CA
  store threw `SSL: no trusted CA certificate found (6)` against Cloudflare,
  and importing the DigiCert root alone (`certificates-imported: 1`) was _not_
  enough — Cloudflare's chain needs more than the root. The full `curl.se`
  bundle (121 certs) fixed it. Do not just set `verify-doh-cert=no` to dodge
  this; that defeats the point of DoH.

- **Chicken-and-egg when fetching the cert.** Once DoH is the active resolver
  and broken, `/tool fetch` can't resolve `curl.se` to download the fix. Break
  out by temporarily clearing DoH (`/ip dns set use-doh-server=""`), which falls
  back to plaintext `servers=`, fetch + import the cert, then re-enable DoH.

- **Re-enable DoH as a single line.** Pasting multi-line blocks over SSH split
  mid-command and silently left DoH disabled / threw `bad command name uery`.
  Run `/ip dns set use-doh-server=... verify-doh-cert=yes` on its own and wait
  for the prompt before the next command.

- **Once DoH resolves, the regular `servers=` becomes fallback only.** RouterOS
  uses DoH exclusively while it works; the plaintext servers are only consulted
  if DoH breaks. The static `cloudflare-dns.com` entries exist so the plaintext
  `servers=` line can eventually be removed to close the last leak — but leave
  the fallback in place until DoH stability is confirmed over a few days.

- **Add the WAN drop rules only after `/resolve` succeeds.** Adding them while
  DoH is broken kills the plaintext fallback too and takes down DNS for the
  whole network.

- **Clock drift is the #1 future failure mode.** TLS cert validation fails if
  the clock is off, which makes DoH die silently. Keep NTP pinned
  (`/system ntp client set enabled=yes`, ideally `time.cloudflare.com`).

- **Known coverage gap:** enforcement only catches plaintext 53 and DoT 853.
  Client-side DoH (Chrome/Firefox "secure DNS", apps hardcoding DoH over 443)
  looks identical to normal HTTPS and bypasses all of the above. Closing it
  requires an address-list of known public DoH endpoint IPs dropped on forward.
