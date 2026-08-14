# Ports and Services Reference

This document is a quick operational reference. Treat the live Pi as the final source of truth and verify before changing production.

## Important ports

| Port | Proto | Component | Role | Exposure |
|---|---|---|---|---|
| 8081 | TCP | DVSConnexx / Quick Connect | Local web origin | Localhost / Cloudflare Tunnel |
| 50555 | UDP | Analog_Reflector | DVSwitch Mobile-facing USRP port | WAN port-forward / direct Internet |
| 31001 | UDP | Analog_Reflector ↔ Analog_Bridge | Internal USRP transport | Local/internal only |
| 2222 | UDP | Analog_Bridge → Web_Proxy | Raw PCM for RX Monitor | Local only |
| 8080 | TCP | Web_Proxy | WebSocket PCM relay | Local/LAN only; do not expose publicly |
| 1883 | TCP | Mosquitto | Analog_Reflector MQTT broker | Localhost/internal |
| 443 | TCP | Analog_Reflector | Analog_Reflector WebSocket server capability | Do not expose/change casually |

Other mode/TLV ports may exist and may change depending on active mode. Do not hard-code a single MMDVM_Bridge TLV port as a universal health check.

## Services

### `dvsquick.service`

Custom E25XLD DVSConnexx / Quick Connect application.

Responsibilities include:
- UI/API
- Quick Connect controls
- Favorites/Recent
- RX Monitor relay/auth
- activity monitoring
- health/status presentation

Safe restart scope for normal web/backend code changes: restart this service only.

### `analog_reflector.service`

Role:
- remote DVSwitch Mobile ingress on UDP 50555
- bridges to Analog_Bridge over localhost UDP 31001

Do not restart without explicit approval.

### `analog_bridge.service`

Role:
- digital/analog bridging
- internal USRP on UDP 31001
- emits PCM to Web_Proxy on UDP 2222

Do not restart without explicit approval.

### `mmdvm_bridge.service`

Role:
- digital network protocol bridge
- source of rotating activity logs used by NOW TALKING / Last Heard

Do not restart without explicit approval.

### `webproxy.service`

Role:
- receives Analog_Bridge PCM on UDP 2222
- broadcasts PCM to WebSocket clients on TCP 8080

Current architecture uses it as the trusted internal source for the browser RX Monitor.

### `mosquitto.service`

Role:
- local MQTT broker used by Analog_Reflector

Included in AR health evaluation because AR is configured to use the local broker.

### `cloudflared.service`

Role:
- Cloudflare Tunnel connector for management/web hostname

Web management path only. Not used for direct DVSwitch Mobile UDP voice.

### `e25xld-ddns.timer`

Role:
- periodically runs the DDNS updater
- maintains `voice.e25xld.hs8ac.com` against the current public IPv4

Expected state between runs: `active (waiting)`.

The associated oneshot service may correctly show `inactive (dead)` after a successful run.

## Health model

### AR

```text
analog_reflector.service active
AND UDP 50555 listener
AND mosquitto.service active
```

### AB

```text
analog_bridge.service active
AND UDP 31001 listener
```

### MB

```text
mmdvm_bridge.service active
```

Health output:

```text
✓ ACTIVE
! DEGRADED
× OFFLINE
? UNKNOWN
```

## Quick read-only verification examples

```bash
systemctl is-active dvsquick.service
systemctl is-active analog_reflector.service
systemctl is-active analog_bridge.service
systemctl is-active mmdvm_bridge.service
systemctl is-active webproxy.service
systemctl is-active mosquitto.service
systemctl is-active cloudflared.service
systemctl is-active e25xld-ddns.timer
```

Socket inspection:

```bash
ss -lntup
ss -uapn
```

These commands are read-only. Do not use this document as authorization to restart or reconfigure services.
