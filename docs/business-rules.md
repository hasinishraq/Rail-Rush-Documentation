# Business rules

The rules follow Bangladesh Railway's online sales. The dev seed (`migrations/seed/bd.sql`) loads
24 stations (English and Bangla names), 22 intercity services on five corridors from Dhaka
(Kamalapur), and journeys for 12 days, skipping each train's off day.

!!! info
    Some times, off days, distances and coach compositions are approximations.

## Classes

| Code | Name |
|---|---|
| `S_CHAIR` | Shovan Chair |
| `SHOVAN` | Shovan |
| `F_SEAT` | First Seat |
| `F_CHAIR` | First Chair |
| `SNIGDHA` | Snigdha (AC chair) |
| `AC_S` | AC Seat |
| `F_BERTH` | First Berth |
| `AC_B` | AC Berth |

Coaches use Bengali-letter codes (KA, KHA, GA, GHA, UMA, CHA, SCHA, JA, JHA, NEO); seats are named like
`UMA-17`. Seat types: `WINDOW`, `AISLE`, `MIDDLE`, `BERTH_LOWER`, `BERTH_UPPER`.

## Fares and charges

- Fares come from a fare chart per station pair and class (`fare`). Example, Dhaka–Chattogram:
  Tk 450 (Shovan Chair) / 855 / 1,025 / 1,540.
- Children pay **half fare**.
- An online **service charge of Tk 20 per ticket** is added (`charge_rule`).
- Amounts are stored in poisha (minor units) with the currency `BDT`.

## Sale window

| Rule | Value |
|---|---|
| Advance booking | 10 days ahead |
| Opening time | 08:00 for western-zone trains, 14:00 for eastern-zone trains |
| Online sales close | 30 minutes before departure (`journey.sales_close_at`) |
| Quota | One pool per class, `ONLINE` |

## Limits

| Rule | Value |
|---|---|
| Seats per user per journey | 4 (cart + held + bought) |
| Seats per user per travel day | 4, across all trains (`ORDER_LIMIT_EXCEEDED`) |
| Seat lock | 2 minutes idle, 5 minutes maximum |
| Hold (OTP + payment) | 10 minutes |
| OTP | 3 minutes, 5 attempts, 3 resends, 10 sends per user per hour |
| Accounts | One per phone number (`+8801XXXXXXXXX`) and one per NID |

There are no fallback seats: a user only pays for the seats they locked. The one exception is a
payment that arrives just after the hold expired, which gets the nearest free seats
(`seats_changed`) or, if none are left, a full refund.

## Refunds

| When cancelled | Refund |
|---|---|
| 24 hours or more before departure | Fare minus 25 % (minus 40 % for AC classes, 30 % for first class) |
| 12–24 hours before | 50 % |
| 6–12 hours before | 25 % |
| Less than 6 hours before | Nothing |
| Failed confirmation (`CONFIRM_FAILED`), amount mismatch, duplicate capture | Full amount |

Refund rules live in `internal/fares`. Refunds are paid asynchronously by the refunder and retried
until they succeed.
