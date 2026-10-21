# Real-time seat map

The seat map serves thousands of concurrent viewers per train at low cost and delivers changes to
all of them within a second.

## Snapshot

The allocator publishes the whole seat map of a journey to Redis (`seatmap:{journey}`) at most once
per second. `GET /v1/journeys/{id}/seatmap` serves it unchanged:

```json
{
  "journey_id": 74220261021,
  "version": 42,
  "generated_at": "2026-10-20T09:25:59Z",
  "segments": 6,
  "encoding": "u64le-base64",
  "seats": "…",
  "seat_pool": "…",
  "pools": [{ "class_code": "S_CHAIR", "quota": "ONLINE", "open": true, "max_seats_per_user": 4 }],
  "segfree": { "S_CHAIR/ONLINE": [256, 256, 255, 255, 256, 256] },
  "live_url": "/v1/journeys/74220261021/seatmap/live"
}
```

| Field | Meaning |
|---|---|
| `seats` | base64 of one little-endian `uint64` per seat (by `seat_idx`): the occupancy bitmap, bit *k* = segment *k* taken |
| `seat_pool` | base64 of one byte per seat: index into `pools`, `255` = not sellable |
| `segfree` | free seats per segment for each `class/quota`; the minimum over a trip's segments is the trip's availability |
| `version` | increases with every decision; deltas are ordered by it |

The client decodes it once (`web/src/seatmap.ts`) and checks a seat with
`(bits[seat_idx] & mask(from, to)) == 0`.

## Live deltas (SSE)

**Every signed-in viewer** opens `GET /v1/journeys/{id}/seatmap/live`, whether already admitted or
still in the waiting room. Only a bearer token is needed; admission is required to *lock* seats,
not to watch them. Users in the queue therefore see seats disappear in real time and do not pick
seats that are already gone the moment they are let in.

After each commit the allocator gathers the seats that changed into a delta, at most once per
250 ms per journey (`LIVE_DELTA_INTERVAL`), publishes it to `smd:{journey}` and appends it to the
capped log `smlog:{journey}` (`SMLOG_MAXLEN` = 2,000). The 250 ms is a minimum gap, not a fixed
delay: on a quiet train a change is sent right after its commit; on a busy one, changes within 250 ms
are batched into one delta.

```text
event: delta
id: 43
data: {"from_v":42,"v":43,"changes":"…","at":"2026-10-20T09:26:00.250Z"}
```

`changes` is base64 of 12-byte records: `uint32` seat index followed by the seat's new `uint64`
bitmap. A delta covers versions `(from_v, v]` and carries final bits, so it applies to a map at
version *h* when `from_v ≤ h < v`; older deltas are ignored; a gap (`from_v > h`) means "refetch the
snapshot".

Other events on the stream:

| Event | Meaning |
|---|---|
| `resync` | The server cannot continue from your version; fetch the snapshot (`snapshot_url`) and reconnect |
| `reconnect` | Close and reconnect. `data.reason` is `max_age` (connection reached `LIVE_CONN_MAX_AGE`), `replaced` (the same user opened a stream for this train elsewhere), or empty (pod shutting down, or the client was too slow) |

On reconnect the client sends `Last-Event-ID`; the live pod subscribes first and then replays missed
deltas from `smlog`, so nothing falls in between. A ping comment every 15 s keeps proxies from closing
idle streams.

There is **one stream per user and train**. The web app treats `reason: replaced` as "another tab
has the stream" and polls the snapshot for 60 s before retrying.

### Fan-out

Each live pod holds **one** Redis subscription per hot journey and writes every delta to all its local
connections. A client whose buffer is full is disconnected (`live_slow_disconnects_total`) and resumes
from `smlog`. When a pod reaches `LIVE_MAX_CONNS` it refuses new streams so the client tries another pod.

## Seat mirror in the API

Each API pod keeps the bitmaps of hot journeys in memory, fed by the same `smd:{journey}` deltas
(`internal/seatmirror`). A click on a seat that was taken a moment ago is refused with `SEAT_TAKEN`
in about 1 ms without touching Kafka. The mirror is conservative: it only ever says "taken"; a seat
that looks free always goes to the allocator, which decides.

## Web app behaviour

| State | Feed |
|---|---|
| Signed in (admitted or in the queue) | Snapshot once, then the SSE stream; refetch the snapshot on a gap |
| Stream connecting, failing or overloaded | Poll the snapshot every 2 s; retry the stream with back-off (2 s × failures, at most 30 s) |
| Another tab has the stream (`replaced`) | Poll the snapshot every 2 s for 60 s, then try the stream again |
| Before the sale window opens | The snapshot answers `NOT_OPEN`; the page shows a countdown and keeps checking |

## How fresh is the seat map?

The **coach layout** never changes during a sale and is served from cache: every user sees it
instantly. **Seat occupancy** changes with every click and reaches users like this:

| Viewer | Path | Typical delay | Worst case |
|---|---|---|---|
| Any signed-in viewer | Live delta over SSE | Under 250 ms | ~1 s, or a `resync` on a gap |
| Viewer whose stream is down | Snapshot polling | 1–3 s | ~4 s: snapshot up to 1 s old + up to 2 s until the next poll + up to 1 s of cache |
| Search page | Per-class counts in search results | — | 15 s (search is cached for 15 s) |

`live_delta_age_seconds` measures the real delta age in production.

**Batching:** grouping changes per 250 ms turns a burst of clicks into one message per viewer instead
of one per click.

**Consistency:** the seat map is advisory; the allocator decides. A click on a seat taken a moment
ago gets `SEAT_TAKEN` from the [seat mirror](#seat-mirror-in-the-api) or the allocator.

**Capacity:** each viewer holds one connection on a live pod (`LIVE_MAX_CONNS` = 50,000 per pod).
Each live pod has one Redis subscription per train, independent of viewer count. Scale by adding live
pods; a full pod answers `429 OVERLOADED` and the browser falls back to polling.
