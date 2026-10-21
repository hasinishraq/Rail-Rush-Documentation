# Search and journeys

Served by the `search` role. Everything here except the seat map is public and cacheable.

## Stations

`GET /v1/stations?q=dha&limit=10`

```json
{ "items": [{ "code": "DHK", "name": "Dhaka (Kamalapur)", "name_bn": "ঢাকা (কমলাপুর)", "city": "Dhaka" }] }
```

`q` matches the start of the name; Bangla names are included where known.

## Search trains

`GET /v1/search?from=DHK&to=CTG&date=2026-10-21[&class=SNIGDHA]` · `Cache-Control: public, max-age=15`

```json
{
  "from": "DHK", "to": "CTG", "date": "2026-10-21",
  "generated_at": "2026-10-20T09:25:59Z",
  "trains": [{
    "journey_id": 74220261021,
    "train_no": "742",
    "train_name": "Turna",
    "from": { "idx": 0, "station_code": "DHK", "station_name": "Dhaka (Kamalapur)", "time": "2026-10-21T17:15:00Z" },
    "to":   { "idx": 6, "station_code": "CTG", "station_name": "Chattogram", "time": "2026-10-21T23:15:00Z" },
    "duration_min": 360,
    "classes": [{
      "class_code": "S_CHAIR",
      "fare": { "amount_minor": 45000, "currency": "BDT" },
      "status": "AVAILABLE",
      "count": 256
    }]
  }]
}
```

| Class `status` | Meaning |
|---|---|
| `NOT_OPEN` | Sale window not open yet; `opens_at` says when |
| `AVAILABLE` | `count` seats are free for this exact trip |
| `SOLD_OUT` | No seat free for this trip |
| `CLOSED` | Online sales closed (30 minutes before departure) |

`fare` is per seat, before the service charge. `from`, `to` and `date` are required, and `from` must
differ from `to` (`422 VALIDATION_FAILED` otherwise).

## Journey

`GET /v1/journeys/{id}`

```json
{
  "journey_id": 74220261021,
  "train_no": "742", "train_name": "Turna", "run_date": "2026-10-21",
  "status": "SCHEDULED",
  "layout_id": 11,
  "sales_close_at": "2026-10-21T16:45:00Z",
  "route": [{ "idx": 0, "station_code": "DHK", "station_name": "Dhaka (Kamalapur)", "time": "2026-10-21T17:15:00Z" }],
  "pools": [{ "class_code": "S_CHAIR", "quota": "ONLINE", "opens_at": "2026-10-11T08:00:00Z", "seats": 256, "max_seats_per_user": 4 }]
}
```

`status`: `SCHEDULED`, `CLOSED`, `DEPARTED`, `CANCELLED`. A trip is a pair of route indexes
(`from_idx < to_idx`).

## Layout

`GET /v1/layouts/{layoutId}`

```json
{
  "layout_id": 11,
  "coaches": [{
    "coach": "KA", "class_code": "AC_B", "width": 17, "height": 2,
    "seats": [{ "seat_idx": 0, "seat_id": "KA-1", "seat_no": 1, "seat_type": "BERTH_LOWER", "x": 0, "y": 0 }]
  }]
}
```

`seat_idx` indexes the seat-map arrays. `x`/`y` place the seat on the coach grid; empty columns and
rows are aisles or gaps. `seat_type`: `WINDOW`, `AISLE`, `MIDDLE`, `BERTH_LOWER`, `BERTH_UPPER`.

## Seat map

`GET /v1/journeys/{id}/seatmap` · any signed-in user, no admission needed

Returns the snapshot described in [Real-time seat map](../architecture/realtime.md#snapshot). Before
the sale window opens it gives `409 NOT_OPEN` with `details.opens_at`.
