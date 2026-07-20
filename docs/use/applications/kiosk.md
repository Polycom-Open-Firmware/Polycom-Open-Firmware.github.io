# Kiosk

A locked fullscreen browser: the panel boots straight into one web page —
a dashboard, a calendar, a camera feed, Home Assistant, any URL. This is
the **default role**.

## How it works

The stack is the Wayland compositor **weston** in kiosk-shell mode with
the browser as its only client. `kiosk.service` runs `kiosk-launch`,
which reads the configuration and starts the selected browser engine
fullscreen. There is no desktop, no window chrome, and no way to leave
the page from the touchscreen.

The role is backed by the `poly-app-kiosk` package; the Chromium engine
ships separately in `poly-app-chromium`.

## Browser engines (`KIOSK_ENGINE`)

`kiosk-launch` dispatches on the `KIOSK_ENGINE` key:

- **`webkit`** (default) — [cog](https://github.com/Igalia/cog), a
  lightweight WPE WebKit shell. Small footprint, well suited to
  dashboards. Extra cog flags can be passed via `COG_OPTS`.
- **`chromium`** — the full Chrome engine, heavier. Requires an image
  with `poly-app-chromium` baked in; supported on the TC8.

If the selected engine's package is not present in the image,
`kiosk-launch` falls back to cog and logs a warning — the kiosk never
fails to start over a missing engine.

## Setting the URL

The page to display is the `KIOSK_URL` key, set in the wizard's
Configure step or pushed later as a config update. A
`KIOSK_URL_FALLBACK` key is accepted and persisted, but the
fallback-on-unreachable behavior is not implemented — nothing reads it.

Full key semantics (`KIOSK_URL`, `KIOSK_ENGINE`, `COG_OPTS`, and how
config reaches the device): [configuration reference](../configure.md).
