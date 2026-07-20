# Releases and delivery

How firmware gets from a git tag to bytes flowing into a device: the
asset-name contracts, the manifests, and the proxy that lets a browser
download multi-gigabyte images from GitHub.

The full release contract and its gates are canonical in
[`RELEASING.md` in poly-firmware-build](https://github.com/Polycom-Open-Firmware/poly-firmware-build/blob/main/RELEASING.md).

## How a release happens

Pushing a tag `v*` to poly-firmware-build runs the release workflow on
a GitHub-hosted runner: a full `--target=tc8` build (kiosk profile),
the kernel-size gate (the TC8 `Image` must stay under 32 MiB — the
stock U-Boot `BOOTM_LEN` cap — and the workflow fails the release past
it), then a GitHub Release with the assets attached and a `SHA256SUMS`
over the set.

The wizard lists releases live through its proxy, so a new tag appears
in the OS chooser with **no wizard-side work** — provided the asset
names match the contract.

!!! note "TC8 vs C60"
    The TC8 pipeline above is CI in poly-firmware-build. No CI
    workflow produces the C60 release asset set; the assets are
    uploaded manually.

## TC8: names are the API

TC8 releases are **manifest-less**: the wizard fabricates the install
manifest client-side from well-known asset names. Renaming any of them
breaks the wizard for that release.

| Asset | Role |
|---|---|
| `boot.img` | `boota` slot image: kernel + [ro-root initramfs](ro-root.md) + cmdline |
| `dtbo.img` | DTB in an Android DTBO container |
| `vbmeta.img` | AVB metadata (unsigned but structurally valid — [why](tc8-boot.md)) |
| `rootfs.simg` | sparse rootfs → `userdata`; the default (kiosk) profile |
| `tc8-stage2-uboot.bin` | stage-2 U-Boot for eMMC `boot1` (enrollment + [field update](config-partition.md)) |
| `tc8-gpt-restore.simg` | partition-table repair image |
| `version.env` | sourceable build metadata |
| `SHA256SUMS` | checksums over the image set |

From these the wizard derives two client-side manifests:
`manifest.json` (`stage2` → `tc8-stage2-uboot.bin`) and
`os-manifest.json` (`boot`/`dtbo`/`vbmeta`/`rootfs` keyed to the four
image names).

## C60: manifest-driven

A C60 release carries a real **`c60-manifest.json`** asset, because it
cannot be fabricated: the SDP `bootSeq` addresses it contains are
build-specific ([C60 boot chain](c60-boot.md)). The manifest names its
own assets, so C60 asset names can change freely between releases; its
optional `os` section lists the OS image set, and without it the
wizard offers unlock only. The manifest's presence as a release asset
is what makes a release appear in the C60 OS chooser.

## The Cloudflare proxy

**The problem.** The images live as GitHub release assets, and GitHub
asset downloads send no `access-control-allow-origin` header — a
browser cannot fetch them cross-origin. The in-browser wizard needs a
same-origin source for the bytes. Unauthenticated `api.github.com`
calls also rate-limit unpredictably, so the release *list* needs
proxying too.

**The mechanism.** A Cloudflare Pages Function on a catch-all route
serves both, on the same origin as the wizard SPA — so CORS never
applies:

```
browser → wizard host /artifact/<tag>/<asset>
            → Function fetches github.com/…/releases/download/<tag>/<asset>
            → streams the response body back, same origin
```

- It is a **true streaming proxy, not a redirect** — a redirect would
  bounce the browser to GitHub and hit the CORS wall again. The bytes
  are piped without buffering, so the multi-GiB rootfs streams fine.
- `/releases` is an edge-cached GitHub API proxy; the OS chooser lists
  releases through it in every flavor, and the native flavor points at
  the same hosted endpoints.
- **Per-device repo allowlist**: `/releases` and
  `/artifact/<tag>/<asset>` resolve to poly-firmware-build (the
  default, so older clients keep working); `?device=c60` and
  `/artifact/c60/<tag>/<asset>` resolve to the C60 firmware repo.
  Nothing outside the allowlist is proxied.
- **New releases need zero proxy work**: the route is parameterized,
  so any tag resolves on the fly. The only changes that would ever
  touch the proxy are adding a firmware *repo* to the allowlist or
  renaming the standard TC8 assets — neither happens on a normal
  release.

## Related pages

- [Provisioner architecture](provisioner.md) — the app consuming these
  artifacts.
- The apt archive is versioned independently of releases — images
  resolve packages at build time, so a release snapshots the archive
  by virtue of the image.
