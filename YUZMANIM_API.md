# YU Zmanim — API Reverse-Engineering Report

Date: Sep 2026. Target: https://www.yuzmanim.com (© ZmanWare).

## Verdict: there is NO public JSON API. The site is 100% server-rendered HTML.

Every probe below was run live against the site. There is no `/api/*`, no JSON
content-negotiation, no embedded page data, no API subdomain.

## Evidence

1. **Frontend bundles are vendor-only.** `build/assets/app-*.js` is jQuery +
   Bootstrap + select2 + SweetAlert. Zero `fetch(` calls in site code; `axios`
   is loaded globally but never called by site code.
2. **No content negotiation.** `Accept: application/json` (+ `X-Requested-With:
   XMLHttpRequest`) on `/shacharis` returns the same full HTML (200, ~10 KB).
   `?format=json` also returns HTML.
3. **No API routes.** `api/events`, `api/minyanim`, `api/shacharis`,
   `api/calendar`, `calendar.ics`, `events.ics`, `api-docs`,
   `api/documentation` → all `404` (identical 6603-byte Laravel 404 page).
4. **No API subdomains.** `api.`, `app.`, `m.yuzmanim.com` → DNS/connection failure.
5. **No embedded data.** Pages contain no `__DATA__`, no JSON blobs, no
   `data-event` attributes; only `window.dataLayer` (Google Analytics).
6. **robots.txt** allows everything but reveals no endpoints.
7. **Mobile apps are wrappers.** The App Store / Play apps wrap the same
   server-rendered pages (the site even ships a dedicated `/layout/phone`
   "Mobile View"). No separate app backend was found.
8. **History.** The 2016 open-source iOS app used `ZmanimServer`
   (github.com/niazoff/ZmanimServer) — a different, long-dead backend.

## How the site actually works (for our scraper)

- Stack: Laravel (Blade) + Bootstrap. Favicon/assets on
  `zmanware.us-east-1.linodeobjects.com`.
- Minyanim pages: `/shacharis`, `/mincha`, `/maariv`, `/vasikin`, `/sukkos`,
  `/zmanim`, `/calendar`, `/locations`. Bottom nav uses Font Awesome icons
  (fa-cloud-sun, fa-sun, fa-moon, fa-lemon, fa-clock).
- **Date pagination:** `?date=YYYY-MM-DD` (e.g. `/shacharis?date=2026-11-10`).
  A 7-day strip shows Gregorian + weekday + Hebrew occasion
  ("Chol Hamoed Sukkos", "Hoshanah Rabbah", …).
- **Event markup** (verified on a full school day, 14 events, Nov 10 2026):
  ```html
  <div class="row event align-items-center">
    <div class="col-12 col-lg-3">
      <div class="type" style="background-color:#add8e6;color:#000000">Shacharis</div>
    </div>
    <div class="col-6 col-lg-3 text-center">
      <i><a href=".../location/rubin-shul" class="post-link">Rubin Shul</a></i>
    </div>
    <div class="col-6 col-lg-3 text-center"><b>6:10 AM</b></div>
    <div class="col-12 col-lg-3 text-center">
      ...notes, e.g. <i style="color:#6C757D">Haneitz …</i> / Ashkenaz / Edot HaMizrach
    </div>
  </div>
  ```
  Notes: `<b>` is written `<b >` (trailing space) — `querySelector('b')` still
  matches. Passed (already-prayed) events sit inside `#passed_events` when present.
  Empty days render "*There are no minyanim today*".
- **Our integration** (`fetchYUMinyanim` in `index.html`): 6 page loads
  (today+tomorrow × shacharis/mincha/maariv) via `api.allorigins.win/get?url=…`,
  parsed with `DOMParser`, deduped (type+location+±60 s), cached to localStorage.

## Recommendations (accuracy without an API)

1. **Parse the 4th-column notes** (Haneitz / Ashkenaz / Edot HaMizrach / Selichos…)
   into the minyan rows — free accuracy win, currently discarded.
2. **Add a fetch timeout + one retry.** `fetch()` to allorigins currently has NO
   timeout — a hung proxy stalls minyanim silently. Use `AbortController`
   (~15 s) and a fallback proxy chain (allorigins → corsproxy.io …); single
   proxy is a single point of failure.
3. **Keep the localStorage cache** (already done) and consider stamping it with
   fetch time so stale data is labeled.
4. **Cross-check holidays** against the date-strip occasion labels (bonus).
5. **If an API is ever needed:** contact ZmanWare (site copyright holder) —
   nothing public exists to use in the meantime. Do NOT hammer the site:
   current cadence (6 pages / 15 min) is polite; keep it.
