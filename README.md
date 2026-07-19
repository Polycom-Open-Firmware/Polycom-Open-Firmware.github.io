# openpolycom.cc

Source for the project website, served by GitHub Pages at
[openpolycom.cc](https://openpolycom.cc).

Static HTML, no build step: edit `index.html`, push to `main`, GitHub Pages
deploys it. `CNAME` binds the custom domain; `.nojekyll` disables Jekyll
processing.

The Setup Wizard is a separate deployment (Cloudflare Pages, from the
[provisioner](https://github.com/Polycom-Open-Firmware/provisioner) repo) at
[wizard.openpolycom.cc](https://wizard.openpolycom.cc).
