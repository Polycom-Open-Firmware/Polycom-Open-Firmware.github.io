# Configuration reference

How to configure a device, and every configuration key the firmware
understands.

## Two ways to configure

**With a shell** — edit the config files directly. The kiosk, for
example, reads `/etc/default/tc8-kiosk`:

```sh
KIOSK_URL=https://your-page.example.com/
COG_OPTS=--enable-media=true
```

After editing, `systemctl restart kiosk`. On the TC8, edits made during
a sealed boot evaporate on reboot — make them permanent in
[maintenance mode](storage.md), or use the config blob.

**Without a shell (fleet configuration)** — hostname, kiosk URL,
credentials, Wi-Fi, NTP, timezone, CA certs and the device's role can
all be pushed over fastboot by the provisioning wizard as a **config
blob** flashed to the `cache` partition. This works on both the TC8 and
the C60.

## Config blob semantics

- **Consume-once.** The blob is a one-shot message, not a store. Each
  flashed blob is applied exactly once, on the next boot: written to the
  real filesystem, then invalidated in place. Device state lives in the
  filesystem, never in the blob.
- **Atomic.** The blob is invalidated as the last step of the apply, so
  a power cut mid-apply leaves it intact and it re-applies cleanly on
  the next boot.
- **Explicit intent.** Applying a blob overwrites whatever is configured
  on the device, including maintenance-mode edits.
- **Superset-safe.** Unknown keys are logged and ignored, so a wizard
  can send a superset of what a given image implements.
- **Verified.** A fresh or empty `cache`, or a corrupt half-written
  blob, is ignored — the device keeps its current configuration.

The payload is plain `KEY=value` lines. Multi-line or binary values
(certs, keys) travel base64-encoded in the `*_B64` keys. The on-disk
wire format, header layout, and the reference blob builder live in
[Config partition](../internals/config-partition.md).

!!! warning "Plaintext at rest"
    The blob — including `WIFI_PASSWORD` and `ROOT_PASSWORD` — is
    stored unencrypted on the `cache` partition. See
    [Security and hardening](security.md).

## Key reference

**Status** — *implemented*: applied by the current config reader.
*Reserved*: part of the schema; accepted (and logged) but not acted on.

### Identity

| key | status | semantics | example |
|-----|--------|-----------|---------|
| `DEVICE_NAME` | implemented | sets `/etc/hostname` and the live hostname | `lobby-east` |
| `LOCATION` | reserved | inventory label | `Bldg A / Lobby` |

### Application (device role)

The wizard's Application picker writes `PROFILE`; the config reader sets
the systemd default target accordingly and records `/etc/tc8-profile`.
Baked role packages supply each role's apps — nothing is fetched at
apply time.

| key | status | semantics | example |
|-----|--------|-----------|---------|
| `PROFILE` | implemented | device role. `kiosk` → fullscreen web kiosk; `none` → no application: console + tty1 autologin + ssh, no kiosk lock (the wizard's Developer choice sends `PROFILE=none`; `dev` is a legacy alias for `none`); `smart-speaker` (C60) → console with the audio stack configured, voice app service enabled if baked; `media-player` → the kiosk stack with `KIOSK_ENGINE=kodi` (falls back to the web kiosk if `poly-app-kodi` isn't baked). **Omitted → the role is left untouched** (a config-only push never resets the role); unknown value → `kiosk`. See [Choosing a role](applications/index.md). | `none` |

### Kiosk and display

Consumed by the [kiosk](applications/kiosk.md) role, and by
[media player](applications/media-player.md) where noted (the two share
the kiosk stack).

| key | status | semantics | example |
|-----|--------|-----------|---------|
| `KIOSK_URL` | implemented | the page the panel displays — a web page **or** an `rtsp://` stream; written to `/etc/default/tc8-kiosk` | `https://dash.local` |
| `KIOSK_URL_FALLBACK` | reserved | accepted and persisted, but the fallback-on-unreachable behavior is not implemented — nothing reads the key | `https://backup.local` |
| `KIOSK_ENGINE` | implemented | browser engine: `webkit` (cog, lightweight, default) or `chromium` (full Chrome engine; needs `poly-app-chromium` baked). The media-player role sets `kodi` itself. If a selected engine's package isn't baked, the launcher falls back to cog with a logged warning — the kiosk never fails to start over a missing engine. | `chromium` |
| `COG_OPTS` | implemented | extra cog browser flags | `--enable-media=true` |
| `ROTATION` | reserved | panel orientation override (weston output transform) | `1` |
| `BLANK_TIMEOUT` | reserved | screen-blank / DPMS seconds (`0` = always on) | `0` |
| `BRIGHTNESS` | reserved | backlight 0–100 | `80` |
| `RELOAD_INTERVAL` | reserved | periodic kiosk reload / crash watchdog (seconds) | `3600` |

### Media player

Consumed by the [media player](applications/media-player.md) role.

| key | status | semantics | example |
|-----|--------|-----------|---------|
| `MEDIA_SOURCE` | implemented | network media source added to Kodi's sources alongside the always-present local `/persist/media` — any Kodi-supported path (`smb://`, `nfs://`, `http://`) | `smb://nas/media` |
| `MEDIA_MODE` | implemented | `full` (the whole Kodi UI, default) or `photoframe` (boots straight into a fullscreen slideshow of `MEDIA_SOURCE`, else `/persist/media`) | `photoframe` |

### Network

| key | status | semantics | example |
|-----|--------|-----------|---------|
| `NET_MODE` | reserved | `dhcp` \| `static` | `static` |
| `IP_ADDR` / `NETMASK` / `GATEWAY` | reserved | static addressing | `192.168.1.50/24` |
| `DNS` | reserved | resolvers (comma list) | `192.168.1.1,1.1.1.1` |
| `VLAN_ID` | reserved | tag the `lan` port | `40` |
| `HTTP_PROXY` | reserved | proxy for kiosk and updates | `http://proxy:3128` |
| `NTP_SERVER` | implemented | `timesyncd.conf` `NTP=` | `192.168.1.1` |
| `CONFIG_TIME` | implemented | epoch seconds — **forward-only** clock bump so an offline device (no NTP) boots with a roughly-right clock. Auto-stamped by the wizard at flash time; never moves a real or NTP-synced clock backward. Baseline with no blob = image build date. | `1735689600` |
| `WIFI_SSID` | implemented | configures `wlan0` with `wpa_supplicant` + DHCP | `Corp-Guest` |
| `WIFI_PASSWORD` | implemented | WPA/WPA2 passphrase for `WIFI_SSID`; omit for open Wi-Fi | `s3cretwifi` |
| `WIFI_COUNTRY` | implemented | optional regulatory country in the `wpa_supplicant` config | `US` |

### Access and credentials

| key | status | semantics | example |
|-----|--------|-----------|---------|
| `ROOT_PASSWORD` | implemented | sets the `root` password (change the default — see [Getting in](access.md)) | `s3cret` |
| `KIOSK_PASSWORD` | implemented | sets the `kiosk` user's password | `…` |
| `SSH_AUTHKEY` | implemented | appended to `/root/.ssh/authorized_keys` (fleet admin access); persists across reinstalls with `/root` | `ssh-ed25519 AAAA…` |
| `SSH_ENABLE` | reserved | enable/disable sshd | `true` |
| `SSH_PASSWORD_AUTH` | reserved | allow/deny password login | `false` |
| `STREAM_USER` / `STREAM_PASS` | reserved | credentials for the kiosk destination when not embedded in the URL (for example an RTSP camera) | `admin` / `…` |

### Certificates and trust

| key | status | semantics | example |
|-----|--------|-----------|---------|
| `CA_CERT_B64` | implemented | base64 PEM → installed as a trusted CA (`update-ca-certificates`) for internal HTTPS/RTSP. Repeatable. | `LS0tLS1CRUdJ…` |
| `CLIENT_CERT_B64` / `CLIENT_KEY_B64` | reserved | mTLS client cert/key for the destination | `…` |

### Time, locale, audio

`VOLUME_*` applies to any role with audio output — notably
[media player](applications/media-player.md) and
[smart speaker](applications/smart-speaker.md).

| key | status | semantics | example |
|-----|--------|-----------|---------|
| `TIMEZONE` | implemented | `/etc/localtime` + `/etc/timezone` | `America/New_York` |
| `LOCALE` | reserved | system locale | `en_US.UTF-8` |
| `VOLUME_MASTER` / `VOLUME_SPEAKER` | implemented | ALSA mixer caps, 0–100 (small panel speakers distort high) | `80` / `75` |

### Management / ops

All reserved.

| key | status | semantics | example |
|-----|--------|-----------|---------|
| `LOG_FORWARD` | reserved | remote syslog endpoint | `udp://logs:514` |
| `HEARTBEAT_URL` | reserved | health/telemetry beacon | `https://fleet/beat` |
| `OTA_CHANNEL` / `OTA_URL` | reserved | update channel + server | `stable` |
| `REBOOT_SCHEDULE` | reserved | nightly reboot (kiosk hygiene) | `04:00` |

## Pushing a blob

The reconfigure flow on an installed unit: enter fastboot (the
four-finger gesture, or remotely — see [Updating](updating.md)), let the
wizard build the blob from its form, `fastboot flash cache`,
`fastboot reboot`. The blob is applied on that boot. Reinstall flows
include a blob by default so a fresh unit boots configured — see
[Re-provisioning](../install/reinstall.md).
