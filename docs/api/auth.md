# Authentication

Served by the `auth` role, except `/v1/me*` (served by `api`).

## Register

`POST /v1/auth/register` · header `Idempotency-Key: <uuid>` · `201 Created`

```json
{
  "phone": "+8801711111111",
  "email": "ayesha@example.com",
  "name": "Ayesha Rahman",
  "nid": "1234567890",
  "password": "correct-horse-battery",
  "captcha_token": "0.Zm9vYmFy..."
}
```

| Field | Rules |
|---|---|
| `phone` | Required, `+8801[3-9]XXXXXXXX`, unique |
| `email` | Optional, unique (case-insensitive) |
| `name` | Required, 1–100 characters |
| `nid` | Optional, 10, 13 or 17 digits, one account per NID; stored only as an HMAC |
| `password` | Required, at least 10 characters, not in the built-in list of breached passwords |
| `captcha_token` | Required, Turnstile token (see [CAPTCHA](#captcha)) |

Response: `{ "user_id": 13 }`. Validation problems give `422 VALIDATION_FAILED` with
`details.fields`.

`captcha_token` is not part of the idempotency fingerprint, so a retry with the same
`Idempotency-Key` sends a fresh token.

## Login

`POST /v1/auth/login` · `200 OK`

```json
{
  "login": "+8801711111111",
  "password": "correct-horse-battery",
  "device_id": "web-7f3c",
  "captcha_token": "0.Zm9vYmFy..."
}
```

`login` is a phone number or an e-mail address. `device_id` ties the refresh-token family to a device.
`captcha_token` is a Turnstile token (see [CAPTCHA](#captcha)).

```json
{
  "access_token": "eyJhbGciOiJFUzI1NiIsImtpZCI6ImRldjEi...",
  "token_type": "Bearer",
  "expires_in": 900,
  "refresh_token": "b3J...",
  "refresh_expires_in": 2592000,
  "user": { "user_id": 13, "name": "Ayesha Rahman", "phone": "+8801711111111", "verified": false }
}
```

Wrong credentials, unknown users and locked accounts all give the same `401 INVALID_CREDENTIALS`.

## CAPTCHA

`GET /v1/auth/captcha` · `200 OK`

```json
{ "provider": "turnstile", "site_key": "0x4AAAAAAA..." }
```

Render a [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/) widget with `site_key`
and send its token as `captcha_token` on register and login. Tokens are single-use and expire after 300 s. A
missing or rejected token gives `403 CAPTCHA_INVALID` with `details.captcha_required: true`; if
Cloudflare cannot be reached, `503 UNAVAILABLE`.

## Refresh

`POST /v1/auth/refresh` · `200 OK`

```json
{ "refresh_token": "b3J...", "device_id": "web-7f3c" }
```

Returns a new token pair. **Refresh tokens are single-use.** Presenting one that was already used
revokes its whole family (every session descended from that login) and gives `401 REFRESH_REUSED`:
this is treated as a stolen token.

## Logout

`POST /v1/auth/logout` · bearer token · `204 No Content`

```json
{ "refresh_token": "b3J..." }
```

Revokes the refresh family and the current access token (denylisted on every pod until it expires).

## JWKS

`GET /.well-known/jwks.json` returns the public ES256 keys (`kid`) that verify access tokens. API pods
in other processes fetch it to verify tokens without calling the auth service.

## Access token claims

ES256-signed JWT, 15 minutes (`ACCESS_TTL`). Claims include the user ID (`sub`), role, verified flag,
device ID, `jti` (for revocation), issuer and audience.

## Profile

| Request | Response |
|---|---|
| `GET /v1/me` | `{ "user_id", "name", "phone", "verified", "email", "role" }` |
| `GET /v1/me/passengers` | `{ "items": [{ "id", "name", "gender", "type" }] }` |
| `POST /v1/me/passengers` + `Idempotency-Key` | `201` with the saved passenger |
| `DELETE /v1/me/passengers/{id}` | `204` |

Saved passenger body: `{ "name": "Karim Uddin", "gender": "MALE", "type": "ADULT" }`
(`gender`: `MALE`, `FEMALE`, `OTHER`; `type`: `ADULT`, `CHILD`; name 1–40 characters, Bangla allowed).
