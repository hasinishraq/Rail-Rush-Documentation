# Web app

`web/` is a React 18 + TypeScript app built with Vite. In Docker it is built into static files and
served by nginx on port 8088, which also proxies `/v1/` and `/.well-known/` to Traefik so the app is
same-origin (no CORS).

## Pages

| Route | File | What it does |
|---|---|---|
| `/` | `pages/Search.tsx` | Station pickers, date, results with per-class fares and availability |
| `/login`, `/register` | `pages/Auth.tsx` | Sign in and registration |
| `/journey/:id?from=&to=&class=` | `pages/Journey.tsx` | Trip and class pickers, coach tabs, seat map, automatic waiting room, cart and passenger form |
| `/bookings/:pnr` | `pages/Booking.tsx` | OTP entry, payment method, status, e-ticket download, cancellation |
| `/pay/return` | `pages/PayReturn.tsx` | Landing page after the gateway; waits for the booking to confirm |
| `/trips` | `pages/Trips.tsx` | The user's bookings |
| `/account` | `pages/Account.tsx` | Profile and saved passengers |

Every page except search and auth requires a signed-in user (`RequireUser` in `App.tsx`). Render
errors are caught by `components/ErrorBoundary.tsx`, which shows the message instead of a blank page.

## Modules

| File | Purpose |
|---|---|
| `api.ts` | Typed API client: session in `localStorage` (`rail.session`), automatic token refresh, error type `ApiError`, idempotency keys, `202 → poll` helper (`settle`), SSE reader for seat deltas, admission token per journey in `sessionStorage` |
| `seatmap.ts` | Decodes the seat-map snapshot, applies deltas, builds trip masks |
| `components/CoachMap.tsx` | Draws a coach from the layout grid; each seat shows a strip with one cell per route segment |
| `components/StationBoard.tsx` | Station picker |
| `hooks.ts` | `useUser`, `useNow`, `usePoll` |
| `format.ts` | Money, times in `Asia/Dhaka`, class and status labels |

## Journey page behaviour

1. Load the journey, layout, saved passengers and any existing cart (so a refresh restores locked seats).
2. Load the seat-map snapshot, then open the **live SSE stream** right away, admitted or not; poll
   the snapshot every 2 s only while the stream is not live.
3. When the seat map shows the train on sale, **join the waiting room automatically**
   (`POST /v1/admission`), showing the queue position while waiting.
4. Once admitted, seats can be locked; re-join 30 s before the admission expires.
5. Each click on a free seat is a `PUT`; on a locked seat a `DELETE`. The cart timer counts down.
6. Passenger details → `POST /v1/holds` → navigate to the booking page for OTP and payment.

## Fonts

Plus Jakarta Sans for text, Barlow Condensed for signage, and **Hind Siliguri** as the Bangla
fallback, so Bangla names render consistently. All three come from Google Fonts.
