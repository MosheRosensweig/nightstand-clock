# Currently Working On — nightstand-clock (index.html, single-file app)

## Session goal (Sep 2026)
Implement, in `index.html`: (A) Shabbat-detection fix, (B) Jewish-holiday engine w/
yutorah_player icons + holiday-aware banners, (C) enable-able weather-details box
replacing the YU Minyanim box (auto-on Shabbos/Yuntif), (D) live location-change
detection + auto-update. Then verify + report test steps.

## Prior session (already in working tree, UNCOMMITTED as `M index.html`)
Rain-intensity icons + lightning + wind suffix; "Raining now until H:00";
detailed condition text (Light/Moderate/Heavy); droplet fill gauge;
keep-screen-alive (Wake Lock); settings `showRainEnd`, `showRainGauge`, `keepAwake`.
Verified with `node --check`. NOT committed.

## Key findings (do not re-research)
- YU Yomanim web app uses Font Awesome (fa-lemon for Sukkos etc.) — NOT what user meant.
- User meant LOCAL repo `../yutorah_player`: `assets/themes/*.svg` holiday icons +
  `src/hebrew_calendar.js` ("15-Year Dynamic Hebrew Calendar Engine", Intl-based,
  deterministic except Math.random eve handling — DO NOT copy randomness).
- 15 SVGs extracted to `/tmp/chag_icons.txt` (~6.5KB): rosh_hashanah_a, yom_kippur_a,
  sukkos_a, sukkos_b, hoshana_rabbah_a, simchas_torah_a, pesach_a, pesach_b,
  shavuos_a, chanukah_b, purim_a, lag_baomer_a, tisha_bav_a, tubshevat_a,
  yom_haatzmaut_a.
- Intl Hebrew months in this env: Tishri, Heshvan, Kislev, Tevet, Shevat, Adar,
  Adar II, Nisan, Iyar, Sivan, Tamuz, Av, Elul. Verified: Sep 28 2026 = 17 Tishri 5787.
- Shabbat code spots: weekday banner in `updateTimeAndDate` (~line 1599);
  "It's Shabbos now!" in `updateNextZmanDisplay` grouped Friday branch (~line 1960)
  + separated widget (~line 2070); Saturday Havdala rows (~1964, ~2074).
- Zmanim already dynamic via hebcal lib for ANY date — no 15-yr table needed.
- Holidays: dynamic Intl-based engine (covers 20+ yrs, no stale table). Diaspora
  default; Israel rules iff `Intl` timezone is Asia/Jerusalem|Hebron|Gaza.

## Design decisions (locked)
- Shabbat active = Fri AND now>=sunset(s hkiah); Sat AND now<satSunset+1 shaah-zmanis
  ((sunset-sunrise)/12, fallback 60min). YT end = tzeit85deg else sunset+72min.
- Holiday precedence over Shabbat in banners. Panel: chag banner > shabbos banner
  (except Sat pre-havdalah keeps Havdala row) > Fri CL rows > weather/minyanim.
- Weather box (`localStorage.weatherDetailsBox`, default off) swaps ONLY the
  minyan-list portion incl. "No upcoming/Loading" + header text; auto-on when
  Shabbos-active or yomtov/chol-active; only when YU Minyanim feature enabled.
- Weekday banner: chag (YT/chol, incl. eve) > Shabbos > minor holiday of the day.
- Inline SVGs as `CHAG_SVG` map; `.chag-icon` CSS already added.

## Remaining edits (all in index.html)
1. DONE: `.chag-icon` CSS.
2. DONE: core JS block (CHAG_SVG x15, chagIcon, Hebrew-cal fns, getChagState,
   getMinorTheme, getShaahZmanisMs, getShabbosState, banners,
   shouldShowWeatherDetails, renderWeatherDetails). NOTE: dropped the
   `<text>נ</text>` from chanukah_b (font-dependent rendering).
3. DONE: weekday banner (chag > Shabbos > minor).
4. DONE: grouped panel right column (header swap + branches).
5. DONE: separated yuMinyanim widget same restructure.
6. DONE: fetchWeather += apparent_temperature, relative_humidity_2m (current) +
   apparent_temperature, precipitation_probability, relative_humidity_2m (hourly);
   globals lastFeelsLike/lastHumidity.
7. DONE: Weather-tab setting `weatherDetailsBox` (default Off) + const/save/open/
   livePreview/listener wiring.
8. DONE: `node --check` OK; 23/23 node logic tests pass (/tmp/chagtest.js);
   dates verified vs yuzmanim.com (Sep12=1 Tishri, Sep28=17 Tishri CM, Oct3=22 Tishri SA).
9. TODO: report + local test instructions (in chat).

## (D) Live location-change detection — DONE
- Old code: one-shot getCurrentPosition, lat-only 0.01° check w/ stale closure,
  no user feedback, duplicated IP fallback.
- New: `startGeoWatch()` (one-shot + continuous `watchPosition`, started once via
  `geoWatchId`); `handleNewPosition()` compares with haversine km
  (`LOCATION_CHANGE_KM = 2`) against fresh localStorage, refetches
  zmanim+weather+location-name and fires `showToast("📍 Location changed
  (~X km) — times updated")`; deduped `ipLocationFallback()`; `#toast` div+CSS.
- Verified: node --check OK; haversine sanity (0.01°=1.11km, jitter 0.11km quiet,
  NYC→Teaneck 20.7km triggers); zero bare `updateLocation(` refs left.

## (E) YU Zmanim API reverse-engineering — DONE, verdict: NO public API exists
- Probed: frontend bundles (vendor-only, zero fetch calls), Accept:json (HTML back),
  ?format=json (HTML), /api/* x8 (all 404), api/app/m subdomains (DNS fail),
  embedded page data (none), robots.txt (nothing). Mobile apps wrap the same
  server-rendered Laravel pages. Doc: `YUZMANIM_API.md` (verdict, evidence, markup
  spec, scraper recommendations).
- Since no API: hardened scraper instead — proxy fallback chain
  (allorigins → corsproxy.io) + 15s AbortController timeout + retry loop;
  notes-column parsing (Haneitz/Ashkenaz/Edot HaMizrach) w/ dup-word collapse;
  notes in dedupe key; escHtml() on all scraped fields (XSS fix);
  15s timeout on fetchWeather too.
- Verified: node --check OK; real-page parser test 14/14 w/ notes
  (/tmp/domtest/parsetest.mjs + /tmp/yu_full.html via linkedom in /tmp only).

## (F) Follow-ups — DONE
- Manual weather box now wins over Shabbos/Chag banners too (`wxManual` branch at
  top of both panel columns). Auto mode unchanged: banners stay, weather fills
  minyan slots only. Rationale: explicit user override vs smart default.
- Rain-start messages now typed + ranged: `rainStartText(hour)` gives
  "Rain starting at 4:00 PM until 6:00 PM (Light rain)" using new
  `precipTypeDesc(code, mm)` (Light/Moderate/Heavy rain, drizzle, snow, showers,
  Thunderstorm) + new `hourlyCodes[]` global fed from fetchWeather.
- New default-ON setting `showRainUntil` ("Rain starting at... until...") in
  Weather tab + full wiring (const/save/open/livePreview/listener).
- Verified: node --check OK; harness test (/tmp/raintest.mjs) — spell w/ until,
  thunderstorm spell, until-off, snow/empty descs all correct.

## (G) Weather-box hours + auto-fit — DONE (defaults to Auto)
- New setting `weatherDetailsHours`: Auto (fit screen, DEFAULT) / 2 / 3 / 4 / 6 /
  8 / 10 / 12, in Weather tab + full wiring.
- `getWeatherDetailsHours()`: fixed values clamped 1..12; Auto picks from
  viewport height (<480→2, <650→3, <800→5, <1000→6, else 8), recomputed every
  1s refresh so resize/orientation adapts.
- Hard guarantee: hour rows wrapped in `.wx-hours` (max-height 34vh +
  overflow-y auto, thin scrollbar) so the box can never push widgets off-screen.
- Verified: node --check OK; band/clamp tests pass (/tmp/wxtest.mjs).

## (H) Cutoff fix — DONE and MEASURED in headless Chrome
- First two attempts (row cap, whole-box 42vh cap) insufficient: box got smaller
  but total #clock still exceeded short viewports; centered body clips top+bottom.
- Reproduced in headless Chrome (desktop binary, playwright-core, /tmp only):
  retro @1024x600 had weekday top=-5, nextZman bottom=605 with box ON (fits with
  box OFF) — exactly the reported symptom. Auto-mode note: on Chol HaMoed the
  box self-enables, so "auto" showed it even with the setting off.
- Real fix: (1) body + #clock use `safe center` (overflow top-aligns instead of
  clipping); (2) #clock max-height 100vh/100dvh + overflow-y auto w/ hidden
  scrollbars (phone layout already did this); (3) short-screen media query
  (max-height 650px: smaller retro digits, tighter paddings/margins).
- Measured after fix: 1024x600 retro fully fits (top=6, bottom=594); sweeps over
  800x600, 1366x768, 375x667, 1920x1080 x all layouts → ZERO top cutoffs;
  worst case 1920x1080 retro needs 19px hidden scroll (acceptable).
- Test harness: /tmp/domtest/measure*.mjs; server on :8123 (left running).
- If user still sees cutoff: need layout style + screen px — cannot reproduce
  further without it.

## (J) Box-vs-timeline overlap — DONE
- Only sidebar pins the hourly timeline absolute at bottom; other layouts have it
  in flow (no overlap possible). Headless showed it clear, but fixed structurally:
  `.sidebar #nextZman, .sidebar #yuMinyanim` capped at 58vh + own thin scroll;
  Auto hours now subtract 1-2 rows when the timeline bar is actually visible
  (measured offsetHeight).
- Measured sidebar 1024x600 / 1366x768 / 1920x1080, zoom 1.0 and 1.3 → all clear.
  (Custom layout excluded — user-arranged; extreme zoom can still visually scale
  things together.) node --check OK.

## (I) Humidity label confusion — DONE
- User saw "Mainly clear" + "💧89%" and read the droplet as rain. It was
  relative_humidity_2m from Open-Meteo. Renamed to text "Humidity 89%" (no
  droplet emoji); hourly "...° · 0%" (precipitation_probability) renamed to
  "...° · 0% rain". Only the SVG rain gauge still uses a droplet shape (with
  "% full" hover tooltip). Verified node --check.

## Test plan for new session
- `python3 -m http.server 8000` in repo; open localhost.
- Console: force `currentZmanimData.sunset` past/future on Fri/Sat; stub Hebrew date
  via temporary override to simulate RH/Sukkos/Pesach; check banners + icons.
- Toggle weather box in settings; check minyan column swap + 8-hr rows.
