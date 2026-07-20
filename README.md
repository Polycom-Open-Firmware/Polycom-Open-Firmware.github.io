# openpolycom.cc

Source for the project documentation site, served by GitHub Pages at
[openpolycom.cc](https://openpolycom.cc).

The site is a [MkDocs](https://www.mkdocs.org/) project using the Material
theme. Pages live under `docs/`; navigation is defined in `mkdocs.yml`.

## Local preview

```sh
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
.venv/bin/mkdocs serve
```

`mkdocs serve` watches `docs/` and serves the site at `http://127.0.0.1:8000/`.
`mkdocs build --strict` fails on broken internal links and nav mismatches.

## Deploy

`.github/workflows/docs.yml` builds the site and publishes it to GitHub Pages
on push to `main`. The repository's Pages source must be set to "GitHub
Actions". `docs/CNAME` binds the custom domain.

The Setup Wizard is a separate deployment (Cloudflare Pages, from the
[provisioner](https://github.com/Polycom-Open-Firmware/provisioner) repo) at
[wizard.openpolycom.cc](https://wizard.openpolycom.cc).
