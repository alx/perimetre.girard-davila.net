# Design: Host Périmètre on GitHub Pages with Cloudflare DNS + Formspree contact form

Date: 2026-07-13

## Goal

Serve the existing static `index.html` at `https://perimetre.girard-davila.net`, with the
demo-request form submitting to Formspree instead of falling back to `mailto:`.

## Current state

- `index.html` is a single static file, no build step, no existing git repo in this directory.
- The demo form (`#demo-form`) already implements a JSON `fetch()` POST against a
  `FORM_ENDPOINT` JS constant, with success/error UI states and a `mailto:` fallback when
  `FORM_ENDPOINT` is empty.
- `girard-davila.net` is already an active zone on Cloudflare (nameservers confirmed). The
  `perimetre` subdomain currently has no DNS record.
- GitHub CLI (`gh`) is authenticated locally as user `alx`.

## Design

### 1. Formspree integration

Formspree endpoint: `https://formspree.io/f/mbdnjlzd`.

Change one line in `index.html`:

```js
const FORM_ENDPOINT = "https://formspree.io/f/mbdnjlzd";
```

No new library needed — Formspree accepts JSON POSTs with `Accept: application/json`, which the
existing fetch call already sends. Existing success/error handling and the `mailto:` fallback
logic are left untouched (the fallback simply becomes dead code while `FORM_ENDPOINT` is set,
which is fine — no need to delete it).

### 2. GitHub repository + Pages

- Initialize git in `/home/alx/code/perimetre.girard-davila.net`.
- Create a **public** GitHub repo named `perimetre.girard-davila.net` under user `alx` via `gh repo create`.
- Add a `CNAME` file at repo root containing exactly `perimetre.girard-davila.net` (this is what
  tells GitHub Pages which custom domain to serve for this project repo).
- Commit `index.html` + `CNAME`, push to `main`.
- Enable GitHub Pages via repo settings: source = `main` branch, `/` (root).

### 3. Cloudflare DNS (manual — done by user, not via API)

An earlier attempt to automate this via a Cloudflare API token was abandoned after repeated
permission issues (token lacked DNS record access even after several attempts to fix scopes).
The user will add the DNS record manually in the Cloudflare dashboard:

- Type: `CNAME`
- Name: `perimetre`
- Target: `alx.github.io`
- Proxy status: **DNS only** (grey cloud) — proxying through Cloudflare can interfere with
  GitHub's automatic Let's Encrypt certificate issuance for custom domains on Pages.

### 4. HTTPS enforcement

Once the DNS record propagates and GitHub Pages shows the custom domain as verified with a
provisioned certificate, enable "Enforce HTTPS" in the repo's Pages settings.

## Out of scope

- Cloudflare API automation (abandoned; DNS record added manually).
- Any build tooling, bundler, or JS framework — site stays a single static HTML file.
- Cal.com integration (`CAL_URL` stays empty).
- Analytics, additional pages, or CI/CD beyond GitHub's built-in Pages deployment.

## Verification plan

- `curl -I https://perimetre.girard-davila.net` returns 200 once DNS/cert are live.
- Submitting the demo form results in a submission visible in the Formspree dashboard for form
  `mbdnjlzd`.
