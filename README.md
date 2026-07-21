# carmenemarcello.com — Wedding Website

Project handbook and migration guide. Last updated: 2026-06-10.

Marcello & Carmen's wedding website: **Saturday 3 October 2026**, ceremony 16:30 at the
Basilica dell'Incoronata Madre del Buon Consiglio (Via Capodimonte 13, Napoli), reception
from 18:00 at Tenuta Torelli (Via Capodimonte 27/27A). RSVP deadline: 31 July 2026.

---

## 1. Stack & architecture

- **Fully static site, no build step.** Everything lives in a single `index.html`
  (~100 KB) with inline CSS and JS. Editing the site = editing that file.
- **Bilingual IT/ES.** Every translatable text is wrapped in `<span class="it">` /
  `<span class="es">`. The active language is a `data-lang` attribute on `<body>`;
  CSS hides the inactive language:
  ```css
  body[data-lang="it"] .es { display: none !important; }
  body[data-lang="es"] .it { display: none !important; }
  ```
  The IT/ES toggle in the nav switches the attribute. The classes work on any
  element (spans, links, buttons).
- **Font:** Cormorant Garamond via Google Fonts CDN (only external CSS dependency).
- **Palette** (CSS vars in `:root`): ivory `#FAF8F5`, sand `#E8DDD3`, blush `#D4B5A0`,
  sage `#9CAF88`, sage-dark `#5F7A4D`, text `#4A4039`, accent `#C09E7F`.

## 2. Hosting & deploy

- **Host:** GitHub Pages, repo `cillodry/carmenemarcello` (public), branch **`master`**
  (not `main`), site served from repo root.
- **Deploy = `git push origin master`.** Pages rebuilds in ~30-60 s.
- **Custom domain:** the `CNAME` file in the repo root contains `carmenemarcello.com`
  (GitHub-specific — remove it if migrating off GitHub Pages).
- **Cache:** GitHub Pages serves `cache-control: max-age=600`. After a deploy, hard
  refresh (or wait up to 10 min) to see changes.
- **HTTPS:** automatic Let's Encrypt certificate managed by GitHub Pages.

## 3. Domain & DNS

- **Registrar / DNS:** IONOS (nameservers `ns*.ui-dns.{com,de,org,biz}`).
- **Records currently in place:**
  - Apex `carmenemarcello.com` → A records to GitHub Pages IPs:
    `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`
  - `www` → CNAME to `cillodry.github.io`
- The site canonical URL is the apex; GitHub redirects `www` automatically.

## 4. Repository layout

| File | Purpose |
|---|---|
| `index.html` | The entire site (markup + inline CSS/JS) |
| `CNAME` | Custom-domain marker for GitHub Pages |
| `manifest.json`, `favicon*.png/svg` | PWA metadata + favicons |
| `og-image.jpg` | Social sharing card (1200×630) |
| `matrimonio.ics`, `matrimonio-es.ics` | Calendar files (IT / ES) for "Add to calendar" |
| `guida-napoli-it.html` / `.pdf` | Naples guide for guests, Italian (source + built PDF) |
| `guia-napoles-es.html` / `.pdf` | Naples guide, Spanish (source + built PDF) |
| `busta-*.jpg/webp/mp4` | Envelope intro animation assets (video played at 0.5×) |
| `thinking-out-loud-instrumental.mp3` | Background audio started by the envelope intro |
| `basilica`, `gaeta-sunset`, `granada`, `portofino`, `thailand-white-temple`, `tropea-sunset` (`.jpg`+`.webp`) | Gallery / section images (webp with jpg fallback) |
| `dresscode.png` | Dress-code illustration in the info section |
| `_backup/` | Local scratch, **gitignored** (with `.DS_Store`) |

## 5. Site mechanics worth knowing

- **Envelope intro:** full-screen "busta" overlay; tapping it plays the opening video
  (`playbackRate = 0.5`) and starts the background audio. Content is beneath it.
- **Sections:** La Nostra Storia · Il Grande Giorno (`#dettagli`) · Informazioni Utili
  (`#info`: hotels with WhatsApp booking links, how to arrive, Naples guide download
  card, dress code) · FAQ · Il Vostro Dono (IBAN with auto-fit single-line display) ·
  RSVP (`#rsvp`) · Gallery with lightbox + swipe gestures.
- **Add to calendar:** Android gets a Google Calendar `render` URL; everything else
  downloads the `.ics` matching the active language.
- **Scroll animations:** `.fade-in` + IntersectionObserver; disabled under
  `prefers-reduced-motion`.
- **iOS quirk handled:** `meta format-detection` prevents Safari from styling the IBAN
  as a phone number.
- **SEO/meta:** schema.org `Event` JSON-LD, Open Graph tags pointing to `og-image.jpg`.

## 6. RSVP pipeline (the only "backend")

```
index.html form  →  fetch POST  →  Google Apps Script web app (…/exec URL, see index.html)  →  Google Sheet
```

- The form collects: `nome`, `cognome`, `ospiti` (adults), `bambini` (kids), `menu`
  (normale / intolleranze / misto), `note`, plus a hidden **honeypot field `website`**
  (`.honeypot` div, `aria-hidden`, `tabindex=-1`) for spam filtering.
- The Apps Script endpoint URL is hardcoded in `index.html` (search for
  `script.google.com`). The script is bound to the response spreadsheet and deployed
  as a web app from the owners' Google account.
- **The spreadsheet** ("RSVP Matrimonio C&M", in the owners' Google Drive) has two tabs:
  - **Risposte** — form-fed. Never type into it. Column A timestamp (custom format
    `dd/mm/yyyy hh:mm:ss`), B/C name/surname, D adults, E kids, F menu, G notes.
  - **Riepilogo** — live dashboard using open-ended ranges (`Risposte!D2:D`), so new
    responses are included automatically. Spreadsheet locale is **US** (comma formula
    separators) — keep it that way or all formulas need `;` instead of `,`.
- **Migration note:** moving the static site does NOT touch the RSVP pipeline (it is
  entirely Google-side). Only if the Apps Script is redeployed does the `/exec` URL in
  `index.html` need updating.

## 7. Naples guide (guests' PDF)

- Source of truth = the two HTML files (print-optimised A4, same palette/font as the
  site, six pages: cover · 12 sights · Capodimonte + food · day trips · itineraries +
  transport · back cover). The PDFs are built artifacts, committed alongside.
- **Rebuild a PDF after editing the HTML:**
  ```bash
  cd wedding-website
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
    --headless --disable-gpu --no-pdf-header-footer --virtual-time-budget=20000 \
    --print-to-pdf="guida-napoli-it.pdf" "file://$(pwd)/guida-napoli-it.html"
  ```
  (same for `guia-napoles-es.*`). Any headless Chromium works; the flags matter.
- Download buttons live in the `#info` section card ("La Nostra Guida di Napoli") and
  are language-aware via the same `.it`/`.es` classes.
- Practical info (opening days, prices, seasonal ferries) was verified 2026-06-10.
  Time-sensitive by nature — the text already tells guests to check schedules online.

## 8. Conventions

- Commits: lowercase prefix `content:` / `fix:` / `feat:` / `perf:` / `media:`,
  imperative, **no AI co-author trailers**.
- Branch: work directly on `master` (push = deploy).
- Copy style: no em-dashes in Italian/Spanish text (use colon or comma); en-dash is
  fine for ranges (`8:30–19:30`) and routes (`Napoli–Positano`).
- Pill buttons (`.calendar-btn`) draw their border with `box-shadow: inset` instead of
  `border` — this avoids uneven edge rendering on mobile; keep that pattern.

## 9. Migration checklist

**To another static host (Netlify, Vercel, IONOS Deploy Now, plain FTP…):**
1. Copy every file in the repo root except `CNAME`, `.git*`, `_backup/`.
   Keep filenames identical (the guide PDFs and `.ics` files are linked by name).
2. No build step, no server config needed. Optional: custom 404 (none exists today).
3. Point DNS at the new host (at IONOS): replace the four GitHub A records on the apex
   and the `www` CNAME per the new host's instructions.
4. Provision HTTPS on the new host before switching DNS if possible.
5. The RSVP keeps working unchanged (Google-side, see §6).

**To another GitHub account/repo (staying on Pages):**
1. Transfer or push the repo; enable Pages (branch `master`, root).
2. Re-add the custom domain in Pages settings (keep the `CNAME` file) and verify.
3. Update the `www` CNAME record at IONOS to `<newuser>.github.io`.

**Things that never move with the site (external, keep access to):**
- IONOS account (domain + DNS)
- Google account owning the RSVP Sheet + Apps Script deployment
- Google Fonts (public CDN, nothing to do)

## 10. Quick reference

```bash
# clone
git clone https://github.com/cillodry/carmenemarcello.git

# local preview (any static server works; file:// also works)
cd carmenemarcello && python3 -m http.server 8080

# deploy
git push origin master   # live in ~1 min, CDN cache up to 10 min
```
