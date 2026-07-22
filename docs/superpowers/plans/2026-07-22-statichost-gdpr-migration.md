# GDPR Asset Migration: GitHub/Cloudflare → statichost.eu — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Serve every asset of Mila's Jimdo portfolio site (MP3 demos, video, logos, Vita PDF, pdf.js library) from EU infrastructure (statichost.eu, Sweden) instead of raw.githubusercontent.com (Microsoft, US) and cdnjs.cloudflare.com (Cloudflare, US), so visitor IPs never reach US endpoints.

**Architecture:** The Jimdo page embeds `header.html` (widgets + all absolute URLs) and `content.html` / `logo-gallery-widget.html` (relative filenames only, via `data-*` attributes). All external URLs live in `header.html`: 5× the GitHub base URL, 2× cdnjs pdf.js. statichost.eu deploys this repo directly from GitHub on push, so GitHub remains the working source but visitors only contact Swedish servers. The migration is: vendor pdf.js into the repo, swap 7 URLs in `header.html`, verify against the live statichost URL, paste updated snippets into Jimdo, update the privacy policy.

**Tech Stack:** Plain HTML/JS (no build step), pdf.js 3.11.174 (pinned — matches current live version), statichost.eu (git-based static hosting), Jimdo (page embeds the HTML snippets manually).

## Global Constraints

- The GitHub base URL to eliminate, appearing exactly 5 times, all in `header.html`: `https://raw.githubusercontent.com/JeremiasMeister/website-mila/main/`
- The cdnjs URLs to eliminate, in `header.html:464` and `header.html:470`: `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js` and `.../pdf.worker.min.js`
- pdf.js version stays pinned at **3.11.174** — do not upgrade during this migration; the custom `PdfViewerWidget` (header.html:859+) is tested against it.
- `content.html` and `logo-gallery-widget.html` must NOT be modified — they reference assets by relative filename and are host-agnostic by design.
- After migration, `header.html` must contain **zero** URLs pointing to `githubusercontent.com` or `cloudflare.com`.
- File names with spaces (e.g. `Mila Arcero - Anime.mp3`) must keep working — widgets already URL-encode via string concatenation; verify statichost serves them (URL-encoded as `%20`).
- This repo is public on GitHub and its full contents will be publicly served by statichost. Never add secrets, credentials, or private documents to it.
- `<SITE-URL>` below is a placeholder for the statichost site URL (e.g. `https://mila.statichost.eu`) — known only after Task 0. Fill it in throughout before executing Tasks 2+.

---

### Task 0: statichost.eu account setup — MANUAL, Jere

No agent can do this; requires creating an account and accepting terms.

- [ ] **Step 1:** Sign up at https://www.statichost.eu/ (free tier).
- [ ] **Step 2:** Create a site from the git repository URL: `https://github.com/JeremiasMeister/website-mila.git`, branch `main`, no build command (plain static files, publish directory = repo root).
- [ ] **Step 3:** Enable automatic deployments on push (see https://www.statichost.eu/docs/).
- [ ] **Step 4:** Note the resulting public site URL and replace every `<SITE-URL>` in this plan with it.
- [ ] **Step 5 (admin, can happen anytime):** Sign their DPA (Auftragsverarbeitungsvertrag, Art. 28 DSGVO) — linked in the statichost.eu footer. Print, sign, email to eric@statichost.eu, keep the countersigned copy.

**Produces:** `<SITE-URL>` — consumed by every later task.

---

### Task 1: Vendor pdf.js 3.11.174 into the repo

**Files:**
- Create: `pdfjs/pdf.min.js`
- Create: `pdfjs/pdf.worker.min.js`

**Interfaces:**
- Produces: repo paths `pdfjs/pdf.min.js` and `pdfjs/pdf.worker.min.js`, which Task 2 references as `<SITE-URL>/pdfjs/pdf.min.js` and `<SITE-URL>/pdfjs/pdf.worker.min.js`.

- [ ] **Step 1: Download the two files (exact version currently live)**

```bash
cd /Users/cg-jm/jm/website-mila
mkdir -p pdfjs
curl -fsSL -o pdfjs/pdf.min.js "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"
curl -fsSL -o pdfjs/pdf.worker.min.js "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js"
```

(One-time developer download; visitor traffic is unaffected — this is exactly what we're removing from the visitor path.)

- [ ] **Step 2: Verify the downloads are real JS, not error pages**

```bash
ls -la pdfjs/
head -c 200 pdfjs/pdf.min.js; echo; head -c 200 pdfjs/pdf.worker.min.js
```

Expected: `pdf.min.js` ≈ 320 KB, `pdf.worker.min.js` ≈ 1 MB; both start with a `/**` license banner or minified JS (`!function` / `(()=>`), NOT `<html` or `404`.

- [ ] **Step 3: Commit**

```bash
git add pdfjs/
git commit -m "vendor pdf.js 3.11.174 locally (GDPR: remove cdnjs/Cloudflare dependency)"
```

---

### Task 2: Swap all external URLs in header.html

**Files:**
- Modify: `header.html` (5× GitHub base URL at lines ~493, ~635, ~846, ~866, ~962; 2× cdnjs at lines ~464, ~470)

**Interfaces:**
- Consumes: `<SITE-URL>` from Task 0; `pdfjs/` paths from Task 1.
- Produces: a `header.html` whose only external hosts are `<SITE-URL>` (plus outbound `<a href>` links, which are fine — clicking a link is the visitor's action, not an automatic transfer).

- [ ] **Step 1: Replace the GitHub base URL (5 occurrences, one sed)**

```bash
cd /Users/cg-jm/jm/website-mila
sed -i '' 's|https://raw\.githubusercontent\.com/JeremiasMeister/website-mila/main/|<SITE-URL>/|g' header.html
```

Note the trailing slash: the original base URL ends with `/`, so the replacement must too.

- [ ] **Step 2: Replace the two cdnjs URLs**

```bash
sed -i '' 's|https://cdnjs\.cloudflare\.com/ajax/libs/pdf\.js/3\.11\.174/pdf\.min\.js|<SITE-URL>/pdfjs/pdf.min.js|' header.html
sed -i '' 's|https://cdnjs\.cloudflare\.com/ajax/libs/pdf\.js/3\.11\.174/pdf\.worker\.min\.js|<SITE-URL>/pdfjs/pdf.worker.min.js|' header.html
```

- [ ] **Step 3: Verify zero US endpoints remain**

```bash
grep -cE 'githubusercontent\.com|cloudflare\.com' header.html; grep -c '<SITE-URL-HOST>' header.html
```

Expected: `0` for the first grep, `7` for the second (5 base URLs + 2 pdfjs). (`<SITE-URL-HOST>` = the hostname part of `<SITE-URL>`.)

- [ ] **Step 4: Confirm the other two HTML files were untouched**

```bash
git diff --stat
```

Expected: exactly one file changed: `header.html`.

- [ ] **Step 5: Commit and push (push triggers the statichost deploy)**

```bash
git add header.html
git commit -m "serve all assets from statichost.eu instead of GitHub/cdnjs (GDPR)"
git push
```

---

### Task 3: Verify statichost deployment, filenames with spaces, and CORS

**Files:** none (verification only); possibly create a headers config if Step 3 fails.

**Interfaces:**
- Consumes: deployed site at `<SITE-URL>` (Task 2 push).

- [ ] **Step 1: Verify an MP3 with spaces in the name is served**

```bash
curl -sI "<SITE-URL>/Mila%20Arcero%20-%20Anime.mp3" | head -6
```

Expected: `HTTP/2 200`, a `content-type` of `audio/mpeg` (or at minimum a correct `content-length` matching the local file).

- [ ] **Step 2: Verify the pdf.js files deployed**

```bash
curl -sI "<SITE-URL>/pdfjs/pdf.min.js" | head -4
```

Expected: `HTTP/2 200`.

- [ ] **Step 3: Verify CORS on the PDF (pdf.js fetches it via XHR from the Jimdo origin)**

```bash
curl -sI -H "Origin: https://www.jimdo-domain-of-the-site.de" "<SITE-URL>/Mila%20Arcero%20-%20Vita.pdf" | grep -i 'access-control\|HTTP'
```

Expected: `access-control-allow-origin: *` (or the echoed origin).

- [ ] **Step 4 (ONLY if Step 3 shows no CORS header):** Consult https://www.statichost.eu/docs/ for the custom-headers mechanism ("our edge network supports redirects, password protection, and many other" features per their docs) and add a headers config granting `Access-Control-Allow-Origin: *` for `/*.pdf` (harmless: repo is public anyway). If statichost has no headers mechanism, email hello@statichost.eu — they advertise support for special setups. Commit the config with message `add CORS header config for PDF viewer`. Re-run Step 3.

Fallback if CORS turns out impossible (unlikely): change the Vita from the JS viewer to a plain `<a href>` download link — links need no CORS. Decision belongs to Jere at that point.

---

### Task 4: End-to-end test against the real statichost URL, BEFORE touching Jimdo

Front-loaded verification: the real interface is reachable, so probe it before declaring anything done. A local file:// test cannot stand in for this — same-origin rules differ.

- [ ] **Step 1: Build a throwaway test page in the scratchpad (NOT in the repo)**

Create `/private/tmp/claude-501/-Users-cg-jm-jm-website-mila/c0a001a5-e052-47de-a029-1e93c31e3d32/scratchpad/jimdo-sim.html` containing, verbatim: the full contents of `header.html`, followed by the full contents of `content.html`, followed by the full contents of `logo-gallery-widget.html` (this mirrors how Jimdo composes the page).

```bash
cd /Users/cg-jm/jm/website-mila
cat header.html content.html logo-gallery-widget.html > "/private/tmp/claude-501/-Users-cg-jm-jm-website-mila/c0a001a5-e052-47de-a029-1e93c31e3d32/scratchpad/jimdo-sim.html"
python3 -m http.server 8712 --directory "/private/tmp/claude-501/-Users-cg-jm-jm-website-mila/c0a001a5-e052-47de-a029-1e93c31e3d32/scratchpad" &
```

Serving over http://localhost:8712 (not file://) so the page has a real non-statichost origin — this exercises the same cross-origin path Jimdo will.

- [ ] **Step 2: Open http://localhost:8712/jimdo-sim.html in Chrome (claude-in-chrome tools) and verify:**
  - An audio player plays (click one; confirm playback position advances).
  - The Vita PDF renders onto canvases (not the `.pdf-error` message).
  - Logo images display.
  - Network requests (mcp__claude-in-chrome__read_network_requests): **zero** requests to `githubusercontent.com` or any `cloudflare.com` host; all asset requests go to `<SITE-URL-HOST>`.

- [ ] **Step 3: Kill the test server**

```bash
pkill -f "http.server 8712"
```

Honest-claim gate: until Step 2 passes against the real statichost URL, the status is "components pass; delivery unconfirmed." Do not report the migration as working before this.

---

### Task 5: Update the Jimdo page — MANUAL, Jere (with agent assistance if wanted)

- [ ] **Step 1:** In Jimdo's editor, replace the embedded head/HTML snippet with the new `header.html` contents (wherever the old copy lives — likely Settings → Edit Head or a custom HTML widget). `content.html` and `logo-gallery-widget.html` are unchanged, so only the header snippet needs re-pasting.
- [ ] **Step 2:** Load the live Jimdo site in a fresh tab, play a demo, open the Vita, and check DevTools → Network filtered by `github` and `cloudflare`: zero requests. This is the real user path — the migration is "done" only after this passes.
- [ ] **Step 3:** Optional cleanup for later (NOT part of this migration): a custom media domain like `media.mila-arcero.de` via CNAME to statichost (docs: https://www.statichost.eu/docs/domains/). Purely cosmetic.

---

### Task 6: Privacy policy update — MANUAL, Jere (draft provided)

- [ ] **Step 1:** In the Jimdo site's Datenschutzerklärung, remove any existing mention of GitHub / raw.githubusercontent.com / cdnjs / Cloudflare as content sources (if present).
- [ ] **Step 2:** Add the following paragraph (verified against statichost's own privacy policy, fetched 2026-07-22 — they store no visitor data, use no cookies, process only within the EU/EEA, supervised by the Swedish authority IMY):

> **Einbindung von Mediendateien über statichost.eu**
> Diese Website bindet Mediendateien (Hörproben, Video, Bilder und PDF-Dokumente) über den Hosting-Dienst statichost.eu ein, betrieben von Variable Object Assignment, c/o Knackeriet, Svartmangatan 9, 11129 Stockholm, Schweden. Beim Abruf dieser Dateien verarbeitet der Anbieter Ihre IP-Adresse, da dies technisch erforderlich ist, um die Inhalte an Ihren Browser auszuliefern. Die Verarbeitung erfolgt ausschließlich innerhalb der EU/des EWR; eine Übermittlung in Drittländer findet nicht statt. Es werden keine Cookies gesetzt und keine personenbezogenen Daten gespeichert. Rechtsgrundlage ist Art. 6 Abs. 1 lit. f DSGVO (berechtigtes Interesse an der technisch fehlerfreien Auslieferung unserer Website-Inhalte). Mit dem Anbieter wurde ein Auftragsverarbeitungsvertrag gemäß Art. 28 DSGVO geschlossen.

- [ ] **Step 3:** The last sentence of the draft is only true after Task 0 Step 5 (signed DPA). If the DPA isn't signed yet, either sign it first or temporarily drop that sentence.

Note: I (Maren) am not a lawyer and this is not legal advice — it's a well-grounded draft following standard German Datenschutzerklärung practice for technically-necessary hosting. If the site ever gets a professional privacy-policy review, include this paragraph in it.

---

## Execution order & dependencies

```
Task 0 (Jere: account) ──┐
Task 1 (vendor pdf.js) ──┴─→ Task 2 (URL swap + push) → Task 3 (deploy/CORS checks)
                                                        → Task 4 (E2E vs real URL)
                                                        → Task 5 (Jere: Jimdo paste + live check)
Task 0 Step 5 (DPA) ──────────────────────────────────→ Task 6 (privacy policy)
```

Task 1 can run before Task 0 completes (no `<SITE-URL>` needed). Tasks 2–4 are blocked on `<SITE-URL>`. Rollback at any point: revert the `header.html` commit — the old GitHub URLs keep working throughout, since nothing is deleted from the repo.
