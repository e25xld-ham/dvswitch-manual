# E25XLD DVSConnexx Phase History

This is a compact historical map of the major work completed so another operator or AI agent can understand why the current architecture looks the way it does.

## Voice / remote-access phases

### E25-VOICE1–4 — Audit and architecture discovery

Key findings:
- the Pi had direct Internet routing available
- the home public IPv4 was directly on the router WAN, not hidden behind CGNAT at the time of audit
- TCP 8080 belonged to DVSwitch Web_Proxy and was not the DVSwitch Mobile USRP endpoint
- Analog_Reflector was not initially installed

A mistaken experimental TCP 8080 port-forward was identified and separated from the final voice design.

### E25-VOICE5–7 — Analog_Reflector package audit/install

- package inspected before installation
- dependencies simulated before install
- package installed while automatic service starts were blocked
- Analog_Reflector and Mosquitto were initially left stopped for controlled configuration

### E25-VOICE8–15 — Normalize mobile/internal USRP ports

Final architecture established:

```text
DVSwitch Mobile -> UDP 50555 -> Analog_Reflector
Analog_Reflector <-> localhost UDP 31001 <-> Analog_Bridge
```

The operator intentionally retained 50555 as the mobile-facing port.

### E25-VOICE16–17 — Client security

- audited Analog_Reflector user schema
- created E25XLD user through the reflector's own user-management command
- disabled unknown clients
- preserved credential secrecy

### E25-VOICE18 — Remote mobile validation

DVSwitch Mobile was successfully connected from outside the home network through UDP 50555.

### E25-VOICE19 — Cloudflare DDNS

Created automatic DNS maintenance for:

`voice.e25xld.hs8ac.com`

Design:
- Cloudflare DNS-only A record
- public IPv4 detection on the Pi
- API-token based update
- systemd oneshot + timer
- approximately five-minute periodic checking

The DNS/timer path was verified with `NO CHANGE — DNS already correct` and working systemd scheduling.

## Cleanup

### E25-CLEANUP2

Removed the unused sample AllStar node 1999 from Analog_Reflector configuration.

Result:
- recurring AMI 127.0.0.1:5038 errors stopped
- AR/AB/MB and related services remained healthy

Unused sample digital bridge definitions were intentionally left alone because they were not causing operational problems.

## Quick Connect / DVSConnexx phases

### Early Quick Connect

Initial capabilities included:
- DMR / D-Star / YSF / NXDN / P25 selection
- target connect
- favorites
- recent targets
- status display
- PIN control login
- mode-specific unlink behavior

D-Star module persistence was added. Legacy D-Star favorites without an explicit module were identified as requiring a safe fallback to module C.

### QC3–QC5 — UI/UX evolution

The UI evolved into a mobile-first dark NOC-style console with:
- status summary
- system drawer
- grouped/filtered favorites
- import/export planning
- mobile responsive refinements
- brand evolution to `E25XLD DVSConnexx`
- footer System control/credit

Current design direction favors compact controls over large monitoring cards.

### QC5 — Web Audio Monitor audit

Important verified discovery:

DVSwitch already provided the required audio source:

```text
Analog_Bridge -> PCM UDP 2222 -> Web_Proxy -> WebSocket 8080
```

A stock DVSwitch browser PCM player was also found and used as protocol reference.

Verified playback format:
- mono
- signed 16-bit PCM
- 8 kHz source
- iOS strategy: 32 kHz AudioContext + 4x sample repetition

Web_Proxy was verified structurally RX-only for this audio path.

### QC6 — RX-only browser audio implementation

Added authenticated same-origin relay from the internal Web_Proxy to DVSConnexx.

Key security design:
- short-lived audio token minted after authenticated control access
- browser connects via same-origin WSS
- no microphone
- no PTT
- no browser-to-DVSwitch audio forwarding

User verified real audio playback successfully from the browser.

Later UX direction simplified the monitor toward a single RX MONITOR toggle rather than separate LISTEN/volume/mute/stop controls.

### QC7–QC8 — NOW TALKING / Last Heard

Audited the stock DVSwitch Gateway Activity implementation.

Source of truth:

`/var/log/mmdvm/MMDVM_Bridge-<UTC-date>.log`

Verified fields:
- callsign/DMR ID
- mode
- target
- source
- start time
- duration
- packet loss
- BER

Important active-state rule:
- header/start without a matched end/duration = currently active
- duration/loss/BER are populated when end-of-transmission is seen

The design evolved toward a compact activity history, with current LIVE/IDLE state merged into the same section to save mobile space.

Callsigns can link outward to QRZ while preserving displayed suffixes; the QRZ lookup uses the base callsign before `/`.

### QC9 — Real bridge health

Prior UI health status only reflected systemd active state.

QC9 implemented meaningful real-health states:

AR:
- analog_reflector.service
- UDP 50555
- mosquitto.service

AB:
- analog_bridge.service
- UDP 31001

MB:
- mmdvm_bridge.service

States:
- ACTIVE ✓
- DEGRADED !
- OFFLINE ×
- UNKNOWN ?

The implementation deliberately avoided hard-coding a mode-dependent MMDVM_Bridge TLV port.

## Current high-level production state

```text
Web management:
  e25xld.hs8ac.com
  -> Cloudflare Tunnel
  -> 127.0.0.1:8081
  -> DVSConnexx

Remote DVSwitch Mobile:
  voice.e25xld.hs8ac.com
  -> direct UDP 50555
  -> Analog_Reflector
  -> UDP 31001
  -> Analog_Bridge
  -> MMDVM_Bridge

RX Monitor:
  Analog_Bridge PCM 2222
  -> Web_Proxy 8080
  -> DVSConnexx authenticated RX-only relay
  -> browser
```

## Pending / optional future work

- finalize compact RX Monitor toggle UX
- finalize bottom-of-page System Health presentation
- show only the latest 10 Recent UI entries while preserving stored history where practical
- optional safe remote Pi reboot button with confirmation and narrow sudo privilege
- synchronize sanitized live production source into this GitHub project
- add deployment/rollback runbooks

## Rule for future agents

Do not repeat old experiments without first reading the current production configuration and `CLAUDE.md`. Several early port assumptions were corrected through audit; the architecture above is the intended current design unless a fresh production audit proves otherwise.
