# Glossary

Admission token
:   A short-lived signed token from the waiting room, required to lock seats and hold them (not to
    watch the seat map). Bound to one user, good for one booking.

Allocator
:   The stateful service that decides every seat change for the journeys of the Kafka partitions it owns.

Cart
:   A user's locked seats on one journey, for one trip and class, before passenger details.

Claim
:   An occupied stretch of a seat: `{mask, kind, owner}`, where kind is lock, hold or booked.

Delta
:   A small update to the seat map covering versions `(from_v, v]`, pushed over SSE.

Hold
:   Locked seats with passenger details and a PNR, waiting for OTP and payment (10 minutes).

Idempotency key
:   A client-chosen UUID that makes a retried request return the original result instead of acting twice.

Journey
:   One run of a train on one date. ID = train number × 10⁸ + YYYYMMDD.

Mask
:   A bitmap of the route segments a trip uses (bits `from_idx … to_idx − 1`).

Outbox
:   A Postgres table written in the same transaction as business data, relayed to Kafka afterwards.

PNR
:   Passenger Name Record: the 10-digit booking reference.

Pool
:   A sellable group of seats on a journey: class × quota (for example `S_CHAIR/ONLINE`).

Projection
:   Postgres tables (`booking`, `ticket`) derived from the `inventory-events` log by the projector.

Segment
:   The stretch of track between two consecutive stops of a journey.

Seat map
:   The snapshot of every seat's occupancy bitmap and pool for a journey, published to Redis.

SSE
:   Server-Sent Events: a one-way HTTP stream used for live seat deltas and request results.

Trip
:   A passenger's stretch of a journey: `from_idx` to `to_idx`.

Waiting room
:   The queue that admits users to the booking flow a few at a time during sales rushes.
