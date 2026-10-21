# Waiting room and seat selection

## Waiting room

Locking seats and holding them need an **admission token** in the `X-Admission-Token` header.
Watching does not: every signed-in user gets the [live seat updates](#live-seat-updates).
In production a waiting-room vendor issues the token; locally
(`ADMISSION_MODE=local`) the API does:

`POST /v1/admission` · user

```json
{ "journey_id": 74220261021 }
```

Admitted:

```json
{ "admitted": true, "admission_token": "eyJhbGciOiJFUzI1NiIs...", "expires_at": "2026-10-20T12:15:00Z" }
```

Still waiting (retry after `Retry-After` seconds, about 1.5 s in the web app):

```json
{ "admitted": false, "position": 37 }
```

Token rules:

- ES256-signed, 15 minutes.
- **Bound to the first user who presents it.** Anyone else gets `ADMISSION_INVALID`, so tokens cannot be shared or resold.
- **One admission, one booking.** After a successful hold the token is used (`ADMISSION_USED`, with the PNR in `details`).
- Calling `/v1/admission` again while holding a valid, unused token with at least 2 minutes left returns the same token, so reloading the page does not spend a new place in the queue.

The web app joins automatically when a train is on sale and re-joins shortly before expiry.

## Lock a seat

`PUT /v1/journeys/{id}/selection/{seatId}` · user + admission

```json
{ "from_idx": 1, "to_idx": 4, "class_code": "S_CHAIR", "quota": "ONLINE" }
```

`quota` defaults to `ONLINE`. Response `200 OK`, the whole cart after this click:

```json
{
  "journey_id": 74220261021,
  "from_idx": 1, "to_idx": 4,
  "class_code": "S_CHAIR",
  "seats": [{ "seat_id": "CHA-10", "seat_idx": 193, "coach": "CHA", "seat_no": 10 }],
  "max_seats": 4,
  "expires_at": "2026-10-20T09:28:00Z",
  "seatmap_version": 3
}
```

Clicking a seat already in your cart returns the cart unchanged (safe double click).

| Error | When |
|---|---|
| `409 SEAT_TAKEN` | Someone else holds the seat on part of your trip |
| `409 NOT_OPEN` / `409 SALES_CLOSED` | Outside the sale window |
| `409 SOLD_OUT` | No seat left in the class for your trip |
| `409 SELECTION_LIMIT` | You would exceed 4 seats on this train (cart + held + bought); `details.max` |
| `409 ORDER_LIMIT_EXCEEDED` | You would exceed 4 seats on this travel day; `details.limit`, `details.issued` |
| `409 CART_CONFLICT` | Your cart is for another class or stretch: release it first |
| `409 ACTIVE_HOLD_EXISTS` | You already have an unpaid hold on this train |
| `422 SEAT_NOT_IN_POOL` | The seat belongs to another class |
| `403 ADMISSION_*` | Missing, invalid or used admission token |
| `429 RATE_LIMITED` | Too many clicks |

## Release a seat

`DELETE /v1/journeys/{id}/selection/{seatId}` · user + admission → `200` with the cart.
Use `*` as the seat ID to release the whole cart. Only your own locks can be released.

## Current cart

`GET /v1/journeys/{id}/selection` · user → `200` with the cart, or `404` when you have none.
The web app calls it on load to restore locked seats after a refresh.

## Live seat updates

`GET /v1/journeys/{id}/seatmap/live` · user (no admission needed) · `text/event-stream`

```sh
curl -N -H "Authorization: Bearer $TOKEN" -H "Last-Event-ID: 36" \
  http://localhost:8088/v1/journeys/74220261021/seatmap/live
```

```text
event: delta
id: 37
data: {"from_v":36,"v":37,"changes":"…","at":"2026-10-20T14:37:51.208Z"}
```

`Last-Event-ID` (or `?last_event_id=`) is the version of the snapshot you hold; it is required, and
without it the stream answers `resync` at once. Events (`delta`, `resync`, `reconnect`) are described
in [Real-time seat map](../architecture/realtime.md#live-deltas-sse). One stream per user and journey:
opening a second one closes the first with `reconnect` (`reason: replaced`).

| Error | When |
|---|---|
| `401 UNAUTHENTICATED` / `TOKEN_EXPIRED` | Missing or expired bearer token |
| `404 NOT_FOUND` | Unknown journey |
| `429 OVERLOADED` | This live pod is at `LIVE_MAX_CONNS`; poll the seat map and retry |
| `503 UNAVAILABLE` | Redis unavailable; poll the seat map |

## Asynchronous results

When the allocator has not answered within 2 s, select, release, hold and cancel return
`202 Accepted` with `request_id`, `poll_url` and `stream_url`.

`GET /v1/requests/{requestId}` (only the user who made the request):

```json
{ "request_id": "01a1...", "state": "DONE", "status_code": 200, "body": { "...": "the original response" } }
```

`state` is `PENDING` or `DONE`; `status_code` and `body` are what the original call would have
returned (a cart, a hold, a cancellation, or an error envelope).

`GET /v1/requests/{requestId}/stream` is the same as SSE: one `event: result` with the body above, or
`event: timeout` if nothing arrives in time.
