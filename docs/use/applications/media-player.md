# Media player

Kodi media center, fullscreen. The panel becomes a touch media device
playing video, music, and photos from its own storage or a network share.

## How it works

Kodi runs as the kiosk stack's Wayland client — the same weston
compositor as the [web kiosk](kiosk.md), with Kodi instead of a browser
(`KIOSK_ENGINE=kodi`). The role is backed by `poly-app-kodi`; per-board
profile packages select it.

On the **Trio C60** the `poly-kodi-skin-c60portrait` package supplies a
portrait skin sized for the 720×1280 panel and routes ALSA to the board's
codec via `/etc/asound.conf`. The **TC8** runs Kodi's default landscape
skin; its audio output is pinned to the on-board codec's fixed 48 kHz
stereo rate.

If `poly-app-kodi` is not baked into the image, the kiosk falls back to
the web kiosk (cog) rather than failing to boot.

## Modes (`MEDIA_MODE`)

- **`full`** (default) — the whole Kodi UI: library, file browsing,
  playback controls, settings.
- **`photoframe`** — the panel boots straight into a fullscreen
  **pictures** slideshow, no interaction needed. Implemented by
  `poly-photoframe.service`, which waits for Kodi's JSON-RPC interface
  and starts a repeat-all slideshow from `/persist/media/photos` (or
  `MEDIA_SOURCE` if set). The source is pictures-only: a slideshow over a
  mixed folder of photos, video, and music makes Kodi open a video player
  and a file browser alongside the pictures.

## Loading media onto the panel

Local media lives under `/persist/media` on the persistent partition, so
it survives reboots *and* reinstalls. It's organized into three folders:

| folder | for |
|---|---|
| `photos/` | pictures — the photo-frame slideshow plays these |
| `music/`  | audio |
| `video/`  | video |

On the TC8, plug a computer into the panel's USB **data** port: the panel
appears as an MTP "Portable Device" with a **Media** storage. Open it and
drag files into the matching folder from any file manager — drop pictures
in `photos/` to see them in the photo frame. (A second **Root home**
storage exposes `/root`.)

### Network source (`MEDIA_SOURCE`)

Any Kodi-supported path — `smb://`, `nfs://`, `http://` — added to Kodi's
video/music/pictures sources alongside the local folders. For photo-frame
mode, point it at a pictures folder.

Kodi's own per-panel state (library, added sources, settings) lives in
the persistent Kodi home and survives reflashes.

## Remote control — and a caveat

!!! warning "Kodi web interface has no authentication"
    Kodi's web server is enabled on **port 8080 with no authentication**,
    for remote control and screenshots. It binds to all device
    interfaces, so keep the device on trusted networks. See
    [security and hardening](../security.md).

## Configuration

The role is selected with `PROFILE=media-player`; `MEDIA_MODE`,
`MEDIA_SOURCE`, and the `VOLUME_MASTER` / `VOLUME_SPEAKER` mixer levels
apply. Key semantics: [configuration reference](../configure.md).
