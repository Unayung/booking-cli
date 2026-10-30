# booking-cli

Agent-native CLI for Booking.com hotels in the Taiwan locale (`zh-tw`, TWD). Zero dependencies (Python 3.8+ stdlib).
Search, hotel rooms/rates, price checks, login, trips (read-only) and the wishlist (saved hotels).
**No booking, no checkout, no payment.**

Every command takes `--json` and returns `{ok, data|error}`. Exit codes: 0 ok, 1 api, 2 usage, 3 auth, 4 net/blocked.

## Transport

Booking.com puts AWS WAF in front of its HTML pages: plain HTTPS without cookies gets HTTP 202 and a JS challenge.
With the `aws-waf-token` cookie that Edge already holds, the pages answer 200. The site's GraphQL endpoint
(`/dml/graphql`) answered without any cookie. `booking login` takes the `Cookie` request header of your own signed-in
browser session (see Login) and checks it against the live profile service.

After that, every command uses plain HTTPS. Edge on CDP 9222 is not touched except by `login --from-browser`, `--via browser` and the WAF
refresh: when a page answers with the challenge, the CLI first takes Edge's current `aws-waf-token`. If that token
is the same one, it opens one background tab on the home page so Edge passes the challenge, then takes the token
again. Background tabs are created with `Target.createTarget(background=true)`, never focused, and closed afterwards.
If it is the last tab, it is blanked instead, so Edge does not quit. Run one command at a time.

Browser state: Booking keeps no locale/currency cookie. Currency is set per request (`selected_currency=TWD`).
Edge's own booking.com pages show HKD by default, and they still did after this CLI's TWD tabs. A background tab sets
`bk_nav_search` (the last search, 30 min); the CLI puts it back the way it was before the tab opened.

## Login

Booking.com signs in with a password (+ verification), so the CLI never takes credentials. It logs in with the
`Cookie` request header of a browser session you already have:

1. In your browser: DevTools > Network > click any request to `www.booking.com` > Request Headers.
2. Copy the value of `Cookie:` (with or without the leading `Cookie: `).
3. Run `booking login`. It reads the clipboard, or stdin when piped.

```bash
booking login                   # Cookie header from stdin (piped) or the clipboard;
                                #   nothing usable there = exit 2 with these steps
booking login --from-clipboard  # clipboard only
booking login --cookie-stdin    # paste (hidden) or pipe it
booking login --keyring         # store it in the OS keyring instead of the file
booking login --from-browser    # opt-in: copy the session from Edge on CDP 9222; if signed out, opens
                                #   the sign-in page in a background tab (--wait 300)
booking whoami | logout
```

The header must contain `bkng_sso_auth` (or `bkng_sso_session`); copy it from a page request so it also holds
`aws-waf-token`, which the HTML pages need (`booking doctor` warns when it is missing; the WAF refresh can
still fetch one from Edge). It is checked against the live profile service before saving to
`~/.config/booking/state.json` (0600, `BOOKING_HOME` moves it) or the OS keyring with `--keyring`.
`BOOKING_COOKIE` overrides both. Exit 3 means there is no valid session (or it expired): log in again.

Edge on CDP 9222 is needed only for `login --from-browser`, `--via browser` and the `aws-waf-token` refresh. Setup: [Browser on CDP 9222](#browser-on-cdp-9222-setup).

## Browser on CDP 9222 (setup)

The commands listed above as needing Edge drive a background tab in your own browser through the Chrome DevTools
Protocol, so their requests carry a real browser's fingerprint and sign-in. The CLI never launches a browser;
start one yourself with remote debugging on:

```bash
# Linux (Edge; google-chrome / chromium take the same flags)
microsoft-edge-stable --remote-debugging-port=9222 --user-data-dir="$HOME/.config/microsoft-edge-debug"

# macOS
"/Applications/Microsoft Edge.app/Contents/MacOS/Microsoft Edge" --remote-debugging-port=9222 \
  --user-data-dir="$HOME/Library/Application Support/Microsoft Edge Debug"
```

- `--user-data-dir` must point at a separate profile: Chromium-based browsers ignore `--remote-debugging-port` on
  the default profile (Chrome 136+). Sign in to booking.com once inside this profile; it stays signed in across
  restarts. If that profile is already open without the flag, quit the browser first: launching again only opens
  a window in the running instance.
- Check it: `curl -s http://127.0.0.1:9222/json/version` prints the browser version, and `booking doctor` shows
  `cdp ok`. Another port or host: `--cdp http://127.0.0.1:9333` or `BOOKING_CDP=http://127.0.0.1:9333`.
- `--remote-allow-origins` is not needed: the CLI connects without an `Origin` header (verified 2026-10-30), and
  leaving it out keeps web pages from connecting. The port listens on 127.0.0.1 only; keep it that way, because
  anything that can reach it controls the browser and every session in it.
- The CLI only creates background tabs (`Target.createTarget(background=true)`), never focuses or touches yours,
  closes its own when done, and blanks rather than closes the last tab so the browser does not quit. Run one
  command at a time.

## Commands

```bash
booking doctor
booking login | whoami | logout          # whoami: name, masked email, Genius level
booking suggest 心齋橋                   # district:2883, landmark:257204, hotel:...
booking search 心齋橋 --checkin 2026-12-04 --checkout 2026-12-07 --adults 3 --children 7 \
     [--rooms 1] [--sort recommended|price|price-desc|rating|distance] [--page 1] [--limit 20] \
     [--pool 100] [--verify] [--stars 3,4] [--free-cancel] [--breakfast] [--min N --max N] \
     [--include-nearby] [--fresh] --json
booking hotel <id|url> --checkin ... --checkout ... --adults 3 --children 7 [--rooms 2]   # alias: rooms
booking price <id> <id> ... --checkin ... --checkout ... --adults 3 --children 7
booking trips [--status all|upcoming|past|cancelled] [--no-details]                      # read-only
booking saved list
booking saved add <hotel id> [--yes]     # dry run without --yes
booking saved rm  <hotel id> [--yes]
booking raw /robots.txt                  # GET only; refuses booking/checkout/payment/confirmation paths
```

`--children` takes ages (`7,4`). A destination is free text resolved through the typeahead (the match is printed,
e.g. 心齋橋 = `district:2883` in 大阪) or `<type>:<id>` such as `city:-240905`.

## Prices: read this

- `price_total` is the total Booking shows (總金額) for the whole stay and party: all rooms, all nights.
  `taxes_included: true` when Booking labels it 含稅費及其他費用. `extra_charges_possible: true` when it says
  可能產生額外費用: a charge such as the Osaka accommodation tax is excluded and paid on top (`excluded_charges` is
  the amount when Booking knows it). `price_per_night` = `price_total / nights`. Sorting uses `price_total` only.
- **The search card is not always the cheapest rate.** In a district/city search Booking often shows the rate it
  pre-selects, which is frequently the free-cancellation one. Measured for 3 adults + child 7 in 心齋橋: hotel 13963456
  card NT$17,275 (free cancellation), cheapest fitting rate NT$15,548 (non-refundable); 5 of the 10 cheapest cards had a
  cheaper rate. `search --verify` adds `verified.lowest_total` for every shown row. `hotel` and `price` always report
  the cheapest fitting rate, plus `search_card` / `card_total` and Booking's own suggestion for comparison.
- `hotel`, 1 room: only rates the page marks as fitting your party (`data-fltrs fit=1`) are used. The page also
  lists rates for 1..3 guests; these are counted in `rates_hidden_not_fitting`.
- `hotel`, 2+ rooms: N of the same room with the party split evenly (children count as guests), or one unit that
  fits everyone when Booking's own suggestion does that (it does for apartments). `fit_basis` says which.
- Booking's price order is not monotonic (measured 11,313 then 11,283). Price sorts read a pool of the first `--pool`
  hotels (default 100) in Booking's price order, re-sort it by total and serve `--page` windows from it. For Taipei
  (417 hotels) the top 20 of a 100-hotel pool and a 300-hotel pool were identical.
- The pool is cached for 15 minutes (`~/.cache/booking`), so `--page 2` continues the same list. Booking's order
  moves between calls: without the cache, 1 of 20 Taipei hotels showed up again on page 2. `--fresh` re-reads.
- With filters, Booking adds hotels from outside the destination ("auto-extended"; 心齋橋 + 4-star + breakfast +
  free cancellation = 4 in the district, 82 up to 21 km away). These are dropped and counted (`dropped.outside_area`);
  `--include-nearby` keeps them with `outside_area: true`.
- Multi-room searches: Booking (and the site itself) also lists one apartment that fits everyone for `--rooms 2`.
  `room`/`units` on each row show what the price buys (`2× 雙床房`, `20床混合宿舍間的雙人床位` = dorm beds).
- Genius: signed-in prices include your Genius discount (`genius: true`). Same search, anonymous vs signed in:
  hotel 11322422 NT$8,361 vs NT$7,051.

## Verified live (2026-10-30)

- `search` 心齋橋 (3 nights), 3 adults + child 7, 1 room: `--sort price` pages 1/2 ascending
  (5,574..21,161), `--sort price-desc` descending (84,320..16,595), recommended order; 0 overlap, 0 in-page
  duplicates, 0 nulls each time. 2 rooms: same checks pass (5,574..21,559 / 84,320..16,607).
- Child changes the result: 3 adults 77 hotels, cheapest 9616998 NT$5,017; with child 7: 64 hotels, NT$5,574.
- `hotel 13963456` cheapest NT$15,548 = its single-hotel card (district card NT$17,275, free cancellation).
  `price` 9616998 / 13963456 / 11859964 = 5,017 / 15,548 / 8,726, equal to their single-hotel cards. 2 rooms:
  `price` 15687243 / 10612423 / 13935295 = 16,479 / 17,793 / 17,303 (search cards 17,268 / 17,793 / 17,303).
- Taipei 2026-11-20..21, 2 adults: 417 hotels, cheapest 1511262 NT$1,380 in search and in `hotel`.
- `--stars 4 --free-cancel --breakfast`, `--sort rating`, `--min/--max`, `--verify`, `--via browser` (same prices
  as http, TWD), `suggest`, `raw`, `doctor`, the WAF-token refresh from Edge (a session without the token got 202,
  then 200).
- `whoami`: name, email and Genius level. `trips`: upcoming and past trips (status, refundability, dates)
  matched the 訂單和旅程 page.
- `saved`: start 1 list (我的旅行口袋) with 0 hotels -> add 11859964 (verified, 1) -> rm (verified, 0) -> same
  as the start. The list's `lastChanged` timestamp moved to today.

## Limitations

- Needs a signed-in session for Genius prices, `trips` and `saved` (`booking login`). Search works anonymously but
  without member prices. `login --from-browser` with a signed-out Edge (opens the sign-in page and waits) and
  `--keyring` were not run live.
- The second WAF step (letting a background tab pass the challenge) was never needed today, so it is unverified.
- No sponsored cards were seen in any search; detection relies on `showAdLabel` / `sponsoredListingData`, which the
  site's own JS uses as its ad flag.
- `--sort distance` runs (Booking's distance_from_search), but its order was not checked; `--currency` other than
  TWD and child ages under 2 were not verified live. `--min/--max` filter
  on `price_per_night` client-side.
- Multi-room `hotel` only combines N of the same room (or Booking's own suggested set); mixed room types that Booking's
  booking flow could offer are not explored.
- Undocumented private API and page markup; field names and the page layout can change.
