# Messaging (Kafka)

Redpanda provides the Kafka API locally. Messages are Protocol Buffers defined in
`proto/booking/v1/` (generate Go code with `buf generate`).

## Topics

| Topic | Partitions (local) | Key | Producers | Consumers |
|---|---|---|---|---|
| `booking-requests` | 256 | `journey_id` | API (Select, Release, Hold, Cancel), relay (Confirm from the outbox), scheduler (Warm, Open, Close) | Allocator |
| `inventory-events` | 256 | `journey_id` | Allocator (transactional) | Projector, invariant checker, allocator (replay on load) |
| `notifications` | 32 | `user_id` | Outbox relay (OTP SMS), projector (ticket SMS) | Relay role's SMS notifier |
| `payment-events` | 32 | — | — | Created by `redpanda-init`; reserved, not used yet |

Keying by `journey_id` puts every command of a journey on one partition, so one allocator worker sees
them in order. That is what makes the allocator a single writer without any locking.

## Commands (`commands.proto`)

Every command is wrapped in `Command { request_id, idem_key, journey_id, reply_to, sent_at, body }`.
`reply_to` is the Redis channel of the API pod waiting for the answer (`results:{pod}`); `idem_key`
is `{user_id}:{Idempotency-Key}` (empty for system commands).

| Command | Sent when | Fields |
|---|---|---|
| `Select` | A user clicks a free seat | user, seat, `from_idx`, `to_idx`, class, quota, channel |
| `Release` | A user clicks a locked seat (`seat_id = ""` releases the whole cart) | user, seat |
| `Hold` | Passenger details are submitted | PNR, user, passengers (one per seat, cart order), channel |
| `Confirm` | A payment is captured | PNR, payment ID, amount, capture time |
| `Cancel` | A booking is cancelled | PNR, user, cancel ID, passenger seqs, channel (`COUNTER` skips the owner check) |
| `Warm` / `Open` / `Close` | Scheduler | `Open` lists pools; empty means every pool whose `opens_at` has passed |

## Events (`events.proto`)

Every event is wrapped in `Event { event_id, journey_id, at, request_id, idem_key, version, body }`.
`version` is the journey version after the event; seat-map snapshots and deltas use it.

| Event | Meaning |
|---|---|
| `SeatSelected` | Seat locked; carries the whole cart and its expiry |
| `SelectRejected` | `NOT_OPEN`, `SALES_CLOSED`, `SEAT_TAKEN`, `SEAT_NOT_IN_POOL`, `SELECTION_LIMIT`, `CART_CONFLICT`, `ACTIVE_HOLD_EXISTS`, `INVALID_SEGMENT` |
| `SeatReleased` | Seats released by the owner |
| `CartExpired` | Locks released by the idle/maximum timer |
| `SeatHeld` | Cart turned into a hold: PNR, passengers, seats, expiry, amount |
| `HoldRejected` | `EMPTY_CART`, `PASSENGER_COUNT_MISMATCH`, `SALES_CLOSED`, `ACTIVE_HOLD_EXISTS` |
| `Confirmed` | Hold turned into a booking; `late` and `seats_changed` flag a late payment |
| `ConfirmFailed` | Payment arrived too late and no seats were left; triggers a full refund |
| `HoldExpired` | Hold ran out unpaid; seats return to sale |
| `DeadlinesExtended` | All live carts and holds extended after an outage |
| `Cancelled` / `CancelRejected` | Cancellation applied / refused (`NOT_FOUND`, `FORBIDDEN`, `NOT_CANCELLABLE`) |
| `SoldOut` / `Available` | A pool's availability crossed zero |
| `JourneyWarmed` / `Opened` / `SalesClosed` | Sale-window lifecycle |

## Snapshots (`snapshot.proto`)

`JourneySnapshot` holds the full journey state: per-seat claims, pool states, carts, holds and the
recently seen idempotency keys with their events. Snapshots are stored in `journey_snapshot`
(Postgres) and bound the replay needed after a restart.

## Outbox

Anything that must reach Kafka as a consequence of a database change is written to the `outbox`
table **in the same transaction** (`internal/outbox`). The relay role publishes unsent rows and
marks them sent. This is how a captured payment becomes a `Confirm` command and how OTP SMS are
queued, with no window where the database and Kafka disagree. `outbox_oldest_unsent_seconds`
alerts when the relay falls behind.
