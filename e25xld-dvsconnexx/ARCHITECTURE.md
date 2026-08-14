# E25XLD DVSConnexx Architecture

## Purpose

This document describes the verified production architecture and the separation between management traffic, DVSwitch Mobile voice traffic, and RX-only browser monitoring.

## 1. Management plane

```text
Browser / Phone
   |
   | HTTPS
   v
https://e25xld.hs8ac.com
   |
   | Cloudflare Tunnel
   v
http://127.0.0.1:8081
   |
   v
dvsquick.service
```

Principles:
- Management/web traffic uses Cloudflare Tunnel.
- Quick Connect binds locally and should not require direct WAN exposure.
- Authentication/session policy belongs to the DVSConnexx application.

## 2. Remote DVSwitch Mobile voice plane

```text
DVSwitch Mobile
   |
   | USRP over UDP
   v
voice.e25xld.hs8ac.com:50555
   |
   | DNS-only Cloudflare record
   v
Home public IPv4
   |
   | Router UDP port forward
   v
DVSwitch Pi :50555
   |
   v
Analog_Reflector
   |
   | localhost UDP 31001
   v
Analog_Bridge
   |
   v
MMDVM_Bridge / digital network
```

Important:
- Voice traffic is direct UDP.
- Do not proxy UDP 50555 through the Cloudflare Tunnel.
- `voice.e25xld.hs8ac.com` is maintained by DDNS when the ISP public IPv4 changes.

## 3. Analog_Reflector ↔ Analog_Bridge

Current normalized architecture:

```text
Analog_Reflector
  mobilePort = 50555
  internal AB peer = 127.0.0.1:31001

Analog_Bridge [USRP]
  txPort = 31001
  rxPort = 31001
```

The mobile-facing port and internal AR↔AB transport are intentionally separated.

## 4. RX-only browser audio

Verified existing DVSwitch audio source:

```text
Analog_Bridge
   |
   | raw PCM UDP 2222
   v
Web_Proxy
   |
   | WebSocket :8080
   v
DVSConnexx internal WS client
   |
   | authenticated same-origin WSS relay
   v
Browser
   |
   v
Web Audio API
```

The existing Web_Proxy is structurally receive-only for this path: it relays PCM received on UDP 2222 to WebSocket clients. DVSConnexx must preserve the same RX-only property and must not forward browser payloads back toward UDP or DVSwitch.

## 5. Activity / Last Heard data flow

```text
MMDVM_Bridge
   |
   v
/var/log/mmdvm/MMDVM_Bridge-<UTC-date>.log
   |
   v
DVSConnexx activity parser
   |
   +-- NOW TALKING
   +-- Last Heard
   +-- Last 5 Activity
   +-- Callsign / mode / target / source / duration / loss / BER
```

Activity is parsed from the MMDVM_Bridge log because the stock DVSwitch dashboard derives Gateway Activity from the same source.

Important semantics:
- Header/start without matched end = active transmission.
- End-of-transmission supplies duration, packet loss, and BER.
- Do not invent loss or BER during an active call.

## 6. QRZ linking

Callsign display may include suffixes such as:
- `E25XLD/INFO`
- `E25RZY/THAM`
- `KM6WPZ/D75`

Display the full value, but QRZ lookup should use the base callsign before the first slash. Numeric-only DMR IDs are not linked unless already resolved to a callsign.

## 7. Bridge health

### Analog Reflector (AR)

Inputs:
- `analog_reflector.service`
- UDP 50555 listener
- `mosquitto.service`

### Analog Bridge (AB)

Inputs:
- `analog_bridge.service`
- UDP 31001 listener

### MMDVM Bridge (MB)

Input:
- `mmdvm_bridge.service`

MB mode-dependent TLV ports must not be hard-coded as a single health condition.

Health states:

```text
✓ ACTIVE
! DEGRADED
× OFFLINE
? UNKNOWN
```

## 8. DDNS

```text
DVSwitch Pi
   |
   | periodic public IPv4 check
   v
e25xld-ddns.service / timer
   |
   v
Cloudflare DNS API
   |
   v
voice.e25xld.hs8ac.com -> current public IPv4
```

Design goals:
- DNS-only record.
- Token limited to required DNS permissions.
- Token stored locally and never committed.
- Timer checks periodically and after boot.

## 9. Failure isolation

A failure in one optional feature should not take down the control console.

Examples:
- RX Monitor failure must not break Connect/Favorites/Recent.
- Activity-log failure must produce an unavailable/idle UI, not an application crash.
- Health-probe failure maps to UNKNOWN.
- DDNS failure should not modify DVSwitch core.

## 10. Future maintenance/reboot control

If a remote reboot control is implemented, it must:
- live behind authentication
- require explicit confirmation
- use a narrowly-scoped sudo rule for exactly the reboot action
- never expose arbitrary shell execution
- never make `dvsquick` run as root

## 11. Production principle

The live Pi is the source of truth for deployed code/config state. GitHub documentation/source must be synchronized deliberately, with secrets redacted, rather than assumed current.
