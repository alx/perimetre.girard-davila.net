# GitHub Pages + Formspree + Cloudflare DNS Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Serve the existing `index.html` at `https://perimetre.girard-davila.net` via GitHub Pages, with the demo form submitting to Formspree.

**Architecture:** Single static HTML file in a new public GitHub repo, deployed via GitHub Pages' built-in branch-based deployment (no build step, no Actions workflow needed). Custom domain wired via a `CNAME` file in the repo plus a manually-added Cloudflare DNS CNAME record. Contact form points at a Formspree endpoint via a one-line JS config change.

**Tech Stack:** Static HTML/CSS/JS, GitHub Pages, Cloudflare DNS, Formspree.

## Global Constraints

- Formspree endpoint is `https://formspree.io/f/mbdnjlzd` — exact value, do not alter.
- Repo must be public (required for free GitHub Pages custom domain + HTTPS).
- Repo owner/user: `alx`. Repo name: `perimetre.girard-davila.net`.
- Cloudflare DNS record must be added manually by the user with proxy status **DNS only** (grey cloud) — do not attempt further Cloudflare API automation (already tried and abandoned per spec).
- Custom domain: `perimetre.girard-davila.net`. GitHub Pages target for the CNAME record: `alx.github.io`.

---

### Task 1: Formspree wiring in index.html

**Files:**
- Modify: `index.html:473`

**Interfaces:**
- None (leaf change to an existing JS constant; no other task depends on this).

- [ ] **Step 1: Change the FORM_ENDPOINT constant**

In `index.html`, line 473, change:

```js
  const FORM_ENDPOINT = "";                     // ex. "https://formspree.io/f/xxxxxx" — vide = mailto
```

to:

```js
  const FORM_ENDPOINT = "https://formspree.io/f/mbdnjlzd";
```

- [ ] **Step 2: Manually verify the change with a local server**

Run:
```bash
cd /home/alx/code/perimetre.girard-davila.net && python3 -m http.server 8000
```
Open `http://localhost:8000` in a browser, fill out the `#demo-form` fields (Nom, Email professionnel, Organisation, Profil), submit, and confirm the status message shows the success text ("Demande envoyée. Réponse sous 24 h ouvrées.") rather than the mailto-fallback text. Then stop the server (Ctrl+C).

Note: the first live submission to a brand-new Formspree form ID typically requires confirming form activation via a verification email sent to the Formspree account owner — check for that email if the dashboard doesn't show the submission immediately.

- [ ] **Step 3: Commit**

(Deferred until Task 2 initializes the git repo — see Task 2 Step 3, which includes this file in the first commit.)

---

### Task 2: Initialize git repo and create GitHub repo

**Files:**
- Create: `.gitignore` (repo root)
- Create: `CNAME` (repo root, contents: `perimetre.girard-davila.net`)

**Interfaces:**
- Produces: a git repo at `/home/alx/code/perimetre.girard-davila.net` with remote `origin` pointing at `github.com/alx/perimetre.girard-davila.net`, `main` branch pushed. Task 3 depends on this repo existing and being pushed.

- [ ] **Step 1: Initialize git and add a minimal .gitignore**

```bash
cd /home/alx/code/perimetre.girard-davila.net
git init
```

Create `.gitignore` with contents:
```
.DS_Store
```

- [ ] **Step 2: Create the CNAME file**

Create `CNAME` at repo root with exactly this content (no trailing content besides the domain and a newline):
```
perimetre.girard-davila.net
```

- [ ] **Step 3: Stage and commit everything, including the Task 1 change and the spec/plan docs**

```bash
cd /home/alx/code/perimetre.girard-davila.net
git add index.html CNAME .gitignore docs
git commit -m "Initial site: index.html, Formspree wiring, GitHub Pages CNAME"
```

- [ ] **Step 4: Create the public GitHub repo and push**

```bash
cd /home/alx/code/perimetre.girard-davila.net
gh repo create alx/perimetre.girard-davila.net --public --source=. --remote=origin --push
```

Expected: command completes, prints the new repo URL `https://github.com/alx/perimetre.girard-davila.net`.

- [ ] **Step 5: Verify the push**

```bash
git log origin/main -1 --oneline
gh repo view alx/perimetre.girard-davila.net --json name,visibility,defaultBranchRef
```

Expected: `defaultBranchRef.name` is `main`, `visibility` is `PUBLIC`, and the log shows the commit from Step 3 present on `origin/main`.

---

### Task 3: Enable GitHub Pages with custom domain

**Files:**
- None (GitHub repo settings only, via `gh api`).

**Interfaces:**
- Consumes: the pushed repo from Task 2 (`alx/perimetre.girard-davila.net`, `main` branch, `CNAME` file at root).
- Produces: GitHub Pages site config with `cname: perimetre.girard-davila.net`. Task 4 (DNS) and Task 5 (HTTPS enforcement) depend on this being enabled first.

- [ ] **Step 1: Enable Pages via the API, sourcing from main/root**

```bash
gh api repos/alx/perimetre.girard-davila.net/pages -X POST \
  -f "source[branch]=main" -f "source[path]=/" 2>&1 || \
gh api repos/alx/perimetre.girard-davila.net/pages -X PUT \
  -f "source[branch]=main" -f "source[path]=/"
```

(The POST creates the Pages site if it doesn't exist yet; if GitHub auto-created it already from detecting the `CNAME` file, POST will 409 and the PUT updates the existing config — that's expected, not an error to fix.)

- [ ] **Step 2: Verify Pages config picked up the custom domain**

```bash
gh api repos/alx/perimetre.girard-davila.net/pages | python3 -c "
import json,sys
d = json.load(sys.stdin)
print('status:', d.get('status'))
print('cname:', d.get('cname'))
print('html_url:', d.get('html_url'))
"
```

Expected: `cname` is `perimetre.girard-davila.net`. `status` may show `null` or `building` until DNS (Task 4) is in place — that's expected at this point, not a failure.

---

### Task 4: Add Cloudflare DNS record (manual, user-performed)

**Files:**
- None (external Cloudflare dashboard action).

**Interfaces:**
- Consumes: target hostname `alx.github.io` (GitHub Pages default domain for user `alx`).
- Produces: public DNS resolution for `perimetre.girard-davila.net`. Task 5 depends on this resolving before HTTPS can be enforced.

- [ ] **Step 1: Add the DNS record**

In the Cloudflare dashboard, under the `girard-davila.net` zone → DNS → Records → Add record:
- Type: `CNAME`
- Name: `perimetre`
- Target: `alx.github.io`
- Proxy status: **DNS only** (grey cloud, not orange)
- TTL: Auto

Save.

- [ ] **Step 2: Verify propagation**

```bash
dig +short CNAME perimetre.girard-davila.net
```

Expected: output includes `alx.github.io.`. If empty, wait a few minutes for propagation and retry.

---

### Task 5: Enforce HTTPS and do final verification

**Files:**
- None (GitHub repo settings + external verification).

**Interfaces:**
- Consumes: DNS record live from Task 4, Pages config from Task 3.

- [ ] **Step 1: Poll Pages status until GitHub reports the domain as verified with a cert**

```bash
gh api repos/alx/perimetre.girard-davila.net/pages | python3 -c "
import json,sys
d = json.load(sys.stdin)
print('status:', d.get('status'))
print('https_certificate:', d.get('https_certificate', {}).get('state'))
"
```

Expected eventually: `status` is `built`, `https_certificate.state` is `approved`. This can take up to ~15 minutes after DNS propagates (Task 4). Re-run this command periodically rather than assuming failure if it's not immediate.

- [ ] **Step 2: Enable HTTPS enforcement**

```bash
gh api repos/alx/perimetre.girard-davila.net/pages -X PUT -f "https_enforced=true"
```

- [ ] **Step 3: Final verification — site is live over HTTPS**

```bash
curl -sI https://perimetre.girard-davila.net | head -1
```

Expected: `HTTP/2 200`.

- [ ] **Step 4: Final verification — form submits to Formspree**

Open `https://perimetre.girard-davila.net` in a browser, submit the `#demo-form`, and confirm:
1. The success status message appears on the page.
2. The submission appears in the Formspree dashboard for form `mbdnjlzd` (https://formspree.io/forms/mbdnjlzd/submissions).

- [ ] **Step 5: Rotate the Cloudflare API token**

The Cloudflare API token saved at `~/.cloudflare_token` during design was pasted into the conversation transcript at one point (during troubleshooting) and is no longer purely local-only. Revoke/delete it in the Cloudflare dashboard (My Profile → API Tokens) since it's no longer needed — DNS was set up manually.

```bash
rm -f ~/.cloudflare_token
```
