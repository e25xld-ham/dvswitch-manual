# E25XLD DVSConnexx

E25XLD DVSConnexx is a personal DVSwitch control, monitoring, and remote-access project built around DVSwitch Server, Analog_Reflector, Analog_Bridge, MMDVM_Bridge, Cloudflare Tunnel, and a custom Quick Connect web console.

This directory is intentionally separated from the legacy DVSwitch manuals in this repository. It is written so another operator, developer, or AI coding agent can understand the architecture before changing production.

## Current production goals

- Quick connection control for DMR, D-Star, YSF, NXDN, and P25.
- Favorites and recent targets.
- D-Star module-aware favorites.
- Remote web management through `e25xld.hs8ac.com`.
- Remote DVSwitch Mobile access through `voice.e25xld.hs8ac.com` on UDP 50555.
- Automatic Cloudflare DNS update when the home public IPv4 changes.
- RX-only Web Audio Monitor in the browser.
- NOW TALKING / Last Heard activity from the MMDVM_Bridge log.
- QRZ links for callsigns.
- Real AR / AB / MB health states with degraded/offline diagnostics.
- Mobile-first dark NOC-style UI.

## High-level architecture

```text
Internet
  |
  +-- Web management
  |     e25xld.hs8ac.com
  |         |
  |         +-- Cloudflare Tunnel
  |                |
  |                +-- 127.0.0.1:8081
  |                       |
  |                       +-- DVSConnexx / Quick Connect
  |
  +-- DVSwitch Mobile voice
        voice.e25xld.hs8ac.com
             |
             +-- DNS-only Cloudflare A record
             +-- UDP 50555 port-forward
                    |
                    +-- Analog_Reflector
                           |
                           +-- localhost UDP 31001
                                  |
                                  +-- Analog_Bridge
                                         |
                                         +-- MMDVM_Bridge
```

## RX Monitor path

```text
Analog_Bridge
   |
   +-- raw PCM -> UDP 2222
                     |
                     +-- Web_Proxy
                           |
                           +-- ws://127.0.0.1:8080
                                  |
                                  +-- DVSConnexx RX-only relay
                                         |
                                         +-- authenticated same-origin WSS
                                                |
                                                +-- Browser Web Audio API
```

The browser audio path is RX-only by design. No microphone or browser-to-DVSwitch transmit path should be added unless a future design is separately reviewed and approved.

## Repository documentation

- `CLAUDE.md` — operating rules for Claude/AI coding agents.
- `ARCHITECTURE.md` — detailed system/data-flow overview.
- `SECURITY.md` — secrets, auth, exposed ports, and change-safety rules.
- `docs/PORTS-AND-SERVICES.md` — important ports and systemd services.
- `docs/PHASE-HISTORY.md` — project evolution and major completed phases.

## Production source

The live production Quick Connect source currently lives on the DVSwitch Pi under `/opt/dvsquick/`.

Do **not** assume files in GitHub are newer than production until a synchronization audit has been performed. Production source should be copied into this project only after secrets and machine-specific credentials have been removed or replaced by examples.

Recommended future source layout:

```text
src/dvsquick/
  app.py
  audio_relay.py
  activity_monitor.py
  static/
    index.html
    app.js
    style.css
```

## Safety rule

Before changing production, read `CLAUDE.md` and `SECURITY.md`.

Never publish control PINs, Cloudflare tokens, DVSwitch passwords, private keys, or real secrets to this public repository.

## Maintainers / identity

- E25XLD
- Also KE9CYN

Web UI credit used by the project:

`Developed by E25XLD · Also KE9CYN`
