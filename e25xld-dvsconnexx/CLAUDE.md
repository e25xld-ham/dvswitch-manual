# CLAUDE.md — E25XLD DVSConnexx Operating Rules

This file is the first document an AI coding agent should read before touching production.

## Role model

- The operator defines intent and approves risky phases.
- The AI plans, audits, implements within scope, validates, and reports exactly what changed.
- Prefer one clearly-scoped phase at a time.
- Read the current filesystem before editing. Do not assume GitHub equals production.

## Production system boundaries

The custom Quick Connect/DVSConnexx application is an add-on. DVSwitch core components must be treated as separate production dependencies.

### DVSConnexx / Quick Connect

- Live source: `/opt/dvsquick/`
- Local origin: `127.0.0.1:8081`
- Public management hostname: `e25xld.hs8ac.com`
- Web path uses Cloudflare Tunnel.

### Core services

- `analog_reflector.service` — AR
- `analog_bridge.service` — AB
- `mmdvm_bridge.service` — MB
- `webproxy.service` — Web_Proxy
- `mosquitto.service` — MQTT broker used by Analog_Reflector
- `cloudflared.service` — web-management tunnel
- `dvsquick.service` — DVSConnexx / Quick Connect
- `e25xld-ddns.timer` — Cloudflare DNS updater

## Mandatory change discipline

1. Audit before edit when behavior is uncertain.
2. Back up every production file before modifying it.
3. Validate syntax/data before restarting anything.
4. Restart only the service required by the change.
5. Report service restart scope explicitly.
6. Prefer safe mocks/read-only checks over generating live radio traffic.
7. If a required dependency is missing, stop and ask before installing it.
8. Never add repositories, signing keys, firewall exposure, or broad sudo privileges without explicit approval.

## Never do these without explicit approval

- Restart Analog_Bridge.
- Restart MMDVM_Bridge.
- Restart Analog_Reflector.
- Reboot the Pi.
- Send RF/PTT.
- Connect/unlink radio networks merely to test UI code.
- Modify router/NAT/firewall rules.
- Modify Cloudflare DNS/Tunnel routing.
- Change DDNS behavior.
- Change the established USRP/audio architecture.

## Established network/audio architecture

### Remote DVSwitch Mobile

```text
DVSwitch Mobile
  -> voice.e25xld.hs8ac.com
  -> direct Internet / DNS-only
  -> UDP 50555
  -> Analog_Reflector
  -> localhost UDP 31001
  -> Analog_Bridge
```

Important: voice traffic does **not** use Cloudflare Tunnel.

### Web management

```text
Browser
  -> https://e25xld.hs8ac.com
  -> Cloudflare Tunnel
  -> http://127.0.0.1:8081
  -> dvsquick.service
```

### RX-only Web Audio Monitor

```text
Analog_Bridge
  -> PCM UDP 2222
  -> Web_Proxy
  -> ws://127.0.0.1:8080
  -> DVSConnexx audio relay
  -> authenticated same-origin WSS
  -> browser Web Audio API
```

Verified PCM assumptions used by the existing implementation:
- mono
- signed 16-bit PCM
- source sample rate 8000 Hz
- iOS playback strategy uses a 32000 Hz AudioContext with 4x sample repetition when required

The monitor must remain RX-only. Do not add `getUserMedia`, microphone capture, PTT, or browser-to-UDP forwarding.

## DVSConnexx functional areas

Current/expected functions include:

- Multi-mode Quick Connect: DMR, D-Star, YSF, NXDN, P25.
- Favorites and Recent targets.
- D-Star module persistence; legacy D-Star favorites without a module must fall back to module C.
- Universal safe disconnect behavior only where commands are verified.
- Control PIN/login and explicit logout behavior.
- RX Monitor toggle.
- NOW TALKING / Last Heard from MMDVM_Bridge logs.
- Last activity history.
- QRZ outbound links for callsigns; slash suffixes display fully but QRZ lookup uses the base callsign.
- AR/AB/MB real health states.
- System drawer/status.
- Optional safe Pi reboot control only with narrowly-scoped privilege.

## Activity source of truth

Gateway activity / Last Heard is derived from MMDVM_Bridge rotating logs under `/var/log/mmdvm/`.

The stock DVSwitch dashboard also parses those logs. Do not scrape the rendered stock dashboard HTML if the log can be parsed directly.

Important semantic rule discovered during audit:
- a call with a start/header but no matched end/duration is active
- duration/loss/BER are only known after end-of-transmission

Do not fabricate loss/BER values while a transmission is active.

## Bridge-health semantics

### AR

Healthy requires:
- `analog_reflector.service` active
- UDP 50555 listener present
- `mosquitto.service` active

### AB

Healthy requires:
- `analog_bridge.service` active
- UDP 31001 listener present

### MB

Currently use service state as the reliable health signal.

Do not hard-code a single TLV mode port for MB; active ports may change with mode.

States:
- `✓ ACTIVE`
- `! DEGRADED`
- `× OFFLINE`
- `? UNKNOWN`

## Security requirements

- Never commit actual Control PINs.
- Never commit Cloudflare API tokens or tunnel tokens.
- Never commit DVSwitch/Analog_Reflector user passwords.
- Never commit TLS/private keys.
- Never publish secrets from `/etc/e25xld-ddns/`.
- Never put PINs or secrets in URLs.
- Use allowlists for system-service probes/actions.
- Use `shell=False` for privileged subprocess execution.
- Never expose an arbitrary shell-command API.

## UI design direction

Brand: `E25XLD DVSConnexx`

Visual language:
- dark navy NOC-style UI
- cyan brand accent
- white/off-white primary text
- green healthy states
- amber degraded
- red destructive/offline
- mobile first
- compact rather than decorative
- no aggressive flashing
- honor `prefers-reduced-motion`

Footer credit:

`Developed by E25XLD · Also KE9CYN`

## Testing policy

Preferred test methods:
- parser fixtures
- mocked service states
- local HTTP/API smoke tests
- browser responsive checks
- historical/live read-only logs

Avoid:
- real PTT
- synthetic RF
- stopping healthy production services merely to prove degraded states

## Final report template

Every implementation phase should report:

1. Scope.
2. Audit findings.
3. Files changed.
4. Backups created.
5. New/changed APIs.
6. Tests performed.
7. Service(s) restarted.
8. Public/local verification.
9. Known limitations.
10. Explicit confirmation of core services/configs left untouched.
