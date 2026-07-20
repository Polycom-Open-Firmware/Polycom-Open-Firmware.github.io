# The apt archive

The OpenPolycom apt archive is the publication channel for `poly-*`
packages — profiles, apps, base plumbing — consumed by the TC8/C60 image
builders and by devices in maintenance mode. It is an ordinary
GPG-signed Debian archive hosted on Cloudflare R2.

## Archive facts

| | |
|---|---|
| URL | `https://pub-1d222577af244182a265fc4d6a35b994.r2.dev` |
| Suite / component | `stable` / `main` |
| Architectures | `arm64 all` |
| Signing key | ed25519 `7A27D57B0045457E4C51A11EFAABA6E245033620` (`apt@openpolycom.cc`) |
| Layout | `pool/` holds every `.deb`; `dists/stable/` holds the generated, signed `Packages`/`Release`/`InRelease` |

## Client setup

Images built from [`poly-rootfs`](../build/rootfs.md) trust the archive
out of the box — the keyring and sources.list are baked in, so
`apt install poly-…` works during image builds and in on-device
maintenance mode with no setup. On any other Debian arm64 system:

```sh
apt install poly-archive-keyring   # or copy the key on first bootstrap
echo 'deb [arch=arm64 signed-by=/usr/share/keyrings/poly-archive-keyring.gpg] \
  https://pub-1d222577af244182a265fc4d6a35b994.r2.dev stable main' > /etc/apt/sources.list.d/poly.list
```

The public key ships in the `poly-archive-keyring` package. Key rotation
means a new key, a new keyring package revision, and a re-publish.

## Publishing

All publishing runs through the
[`apt` repo's](https://github.com/Polycom-Open-Firmware/apt) CI — the
**single writer** to the R2 bucket. The publish job is stateless: pull
the current `pool/` from R2, add the new debs, regenerate and sign the
indexes with `apt-ftparchive` + GPG (no reprepro database), and sync back
to R2. A concurrency group serializes publishes, so indexes never race.

Three paths feed debs into that job; they differ only in where the debs
come from:

1. **Automatic.** A push to the
   [`packages` repo](https://github.com/Polycom-Open-Firmware/packages)
   builds every package and dispatches the apt repo, which pulls the
   built debs and publishes. No manual step — this is the normal path.
2. **Manual dispatch.** Trigger the *Publish apt archive* workflow by
   hand (`gh workflow run publish.yml -R Polycom-Open-Firmware/apt`).
   Publishes the current pool plus anything in `incoming/`; also used to
   re-sign without new debs.
3. **The `incoming/` inbox.** Commit built `*.deb` files under
   `incoming/` in the apt repo, push, and trigger the workflow. This is
   how debs from repos without dispatch wiring — or from a local build —
   reach the archive.

Maintainer details (credentials, workflow wiring, local one-off
publishes) live in the apt repo's
[`PUBLISHING.md`](https://github.com/Polycom-Open-Firmware/apt/blob/main/PUBLISHING.md).

## Verifying a publish

After the publish run goes green:

```sh
curl -s https://pub-1d222577af244182a265fc4d6a35b994.r2.dev/dists/stable/main/binary-arm64/Packages \
  | grep -A1 '^Package: <name>'
```

Then on a device in maintenance mode (`tc8-rw`):
`apt update && apt install <name>`.

!!! note "Bump the changelog every rebuild"
    apt serves the highest version — a rebuilt package with a stale
    `debian/changelog` version silently loses to the copy already in the
    pool.

Adding a new package end-to-end (metapackage → publish → image → wizard)
is the [Add an application](add-an-app.md) cookbook.
