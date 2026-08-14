# E25XLD DVSConnexx Security Model

## Security goals

1. Keep management access separate from the direct UDP voice plane.
2. Prevent accidental publication of credentials or private keys.
3. Keep browser audio RX-only.
4. Keep privileged system actions narrowly scoped.
5. Avoid unnecessary public ports.
6. Fail closed when authentication or health state cannot be determined.

## Public exposure

### Web management

`e25xld.hs8ac.com`

- Served through Cloudflare Tunnel.
- Origin is local to the Pi (`127.0.0.1:8081`).
- Do not expose the Quick Connect origin directly to WAN unless a future architecture explicitly requires it.

### DVSwitch Mobile voice

`voice.e25xld.hs8ac.com`

- DNS-only record.
- Direct UDP 50555 to Analog_Reflector.
- Do not put this path behind Cloudflare HTTP/Tunnel proxying.

### Internal-only audio components

Keep these private/internal:
- UDP 2222 (Analog_Bridge PCM -> Web_Proxy)
- TCP 8080 (Web_Proxy WebSocket)
- UDP 31001 (Analog_Reflector ↔ Analog_Bridge)
- MQTT 1883 remains localhost/internal as configured

## Authentication

Quick Connect control actions require the application's Control PIN/session mechanism.

Security requirements:
- Never store the PIN in repository files.
- Never place the PIN in URLs.
- Prefer session-scoped authentication rather than persistent browser authentication.
- Explicit logout should invalidate the current control state.
- Idle timeout should revoke control capability.

### Audio-session handoff

Browser WebSocket cannot attach the existing custom PIN header directly. The RX Monitor uses a short-lived token issued only after authenticated HTTP access.

Requirements:
- cryptographically random token
- short TTL
- one-time or tightly bounded use
- in-memory storage only
- never log token values
- invalidate on use/expiry/logout where practical

## RX-only guarantee

The monitor is a receive feature.

Forbidden in the production monitor path:
- `getUserMedia`
- microphone permission
- PTT UI
- browser payload forwarding to UDP 2222
- browser payload forwarding to Analog_Bridge
- arbitrary WebSocket-to-system command bridging

RX-only enforcement must be server-side/architectural, not just hidden UI.

## Secret material that must never enter GitHub

- Cloudflare API token
- Cloudflare Tunnel token
- Control PIN
- Analog_Reflector/DVSwitch Mobile password
- TLS private key
- Wi-Fi password
- router password
- SSH private keys
- any bearer/session token
- `/etc/e25xld-ddns/cloudflare.token`
- private certificate/key material under Analog_Reflector SSL directories

Use placeholders such as:

```text
CLOUDFLARE_API_TOKEN=REDACTED
CONTROL_PIN=REDACTED
DVSWITCH_PASSWORD=REDACTED
```

## Configuration examples

Only sanitized examples should be committed.

Before committing a config:
1. remove passwords/tokens/secrets
2. remove private keys
3. review public IPs/hostnames for intentional publication
4. remove unrelated user credentials
5. confirm file permissions/paths do not reveal sensitive material unnecessarily

## DDNS API token

Use a Cloudflare API Token rather than a Global API Key.

Limit it to the minimum zone/DNS permissions needed to maintain the intended record.

The update process should:
- validate the detected IPv4
- update only the intended hostname
- enforce DNS-only (`proxied=false`) for the voice record
- log result/status, not token values

## Health checks

Health/status endpoints are read-only.

Rules:
- service names must be allowlisted
- use bounded subprocess timeouts
- do not accept arbitrary service names or shell fragments from the browser
- probe failure returns UNKNOWN rather than crashing

## Privileged reboot action

If/when remote Pi reboot is enabled:

- `dvsquick` remains unprivileged
- use one `/etc/sudoers.d/` rule permitting only the exact reboot command
- never permit `systemctl *`
- never permit `/bin/sh`, `bash`, or arbitrary command execution
- require authenticated session plus explicit confirmation
- log the action without credentials
- do not use the reboot action as a test during routine UI development

## Change-safety policy

Without explicit operator approval, do not:
- reboot Pi
- restart Analog_Bridge
- restart MMDVM_Bridge
- restart Analog_Reflector
- transmit RF/PTT
- change router/NAT/firewall
- alter Cloudflare routing/DNS
- change voice or internal USRP ports

## Public repository review checklist

Before every GitHub push of production-derived files:

- [ ] no tokens
- [ ] no PIN/passwords
- [ ] no private keys
- [ ] no accidental `.env` contents
- [ ] no shell history
- [ ] no browser session data
- [ ] no backup file containing old secrets
- [ ] only intended system information is public
