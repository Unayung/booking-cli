---
name: booking
description: Use when searching Booking.com hotels (zh-tw, TWD), checking room rates, stay totals incl. taxes and cancellation terms for given dates and party size (including children's ages), comparing prices of several hotels with Genius member prices, or listing the user's Booking.com trips and wishlist (no booking).
---

# booking - Booking.com hotels from a shell (no booking)

`~/.local/bin/booking` -> `~/Projects/booking-cli/booking`. Python 3.8+ stdlib. Plain HTTPS with the cookies copied
from the user's Edge (`booking login`; HTML pages need its `aws-waf-token`, GraphQL does not). Edge on CDP
127.0.0.1:9222 is only used by `login`, the WAF-token refresh and `--via browser`; never launch Edge. Run ONE
command at a time. Start with `booking doctor`; exit 3 = run `booking login`, exit 4 = WAF/blocked.

- `booking suggest <text> --json` -> dest (`district:2883`), name, area, country.
- `booking search <text|type:id> --checkin YYYY-MM-DD --checkout YYYY-MM-DD [--adults 2] [--children 7,4] [--rooms 1]
  [--sort recommended|price|price-desc|rating|distance] [--page N] [--limit N] [--verify] [--include-nearby]
  [--stars 4] [--free-cancel] [--breakfast] --json` -> items[] with id, name, stars, rating, reviews, price_total,
  price_per_night, taxes_included, extra_charges_possible, room, units, free_cancellation, breakfast, genius, url.
  Ads and out-of-area (auto-extended) hotels dropped; price sorts monotonic, 0 nulls, 0 page overlap (measured).
- `booking hotel <id|url> --checkin --checkout --adults --children [--rooms]` -> address, stars, rating, check-in/out,
  amenities, rooms[].rates[] (price_total, cancellation, free_cancel_before, payment, breakfast, rooms_left,
  fit_basis), `cheapest`, `search_card`, `booking_recommendation`.
- `booking price <id>... --checkin --checkout ...` -> cheapest fitting total per hotel (hotel page) + card_total.
- `booking trips [--status upcoming]` (read-only trips + reservations), `booking saved list|add|rm <id> [--yes]`
  (wishlist; dry run without --yes; verified by re-reading after every write).
- `booking login|whoami|logout` (whoami = name, masked email, Genius level), `booking raw <path>` (GET only).

Price rules: compare by `price_total` = stay total for the whole party and all rooms (含稅費及其他費用 when
taxes_included). `extra_charges_possible` = a tax such as the Osaka accommodation tax is paid on top.
A search card can be a pricier free-cancellation rate while a cheaper non-refundable one exists (5 of the 10 cheapest
心齋橋 cards): before recommending, run `booking price <ids>` or `search --verify`. Price sorts rank a pool of
`--pool` (default 100) hotels re-sorted by total; the pool is cached 15 min so `--page 2` continues it (`--fresh`
re-reads). Multi-room searches can list one apartment that fits everyone; check `room`/`units`.

Unverified: second WAF step (background tab), `--sort distance` order, non-TWD currency, signed-out login flow,
keyring/clipboard login. NEVER book, hold, or pay; this CLI has no such command.
