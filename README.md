# InPost Mobile API — Unofficial Documentation

Practical, build-ready documentation for the **private HTTP API** behind InPost
parcel flows (Paczkomaty / parcel lockers in Poland). It exists so you can build
your own client — a script, a widget, a home-automation hook, a nicer app.

> ⚠️ **Unofficial.** This project was built on publicly available information
> and data. It is **not** published, supported, or endorsed by InPost. There is
> no SLA; endpoints, fields, and auth can change or be blocked without notice.
> Use it for personal/interoperability purposes, be a good citizen (no hammering,
> no abuse), and check InPost's Terms of Service before doing anything at scale.

---

## What's in this folder

| File | What it is | Use it when… |
|------|-----------|--------------|
| **`README.md`** (this file) | Human orientation: concepts, the happy path, and what's easy to get wrong. | Start here. |
| **`API_REFERENCE.md`** | The deep reference. Every endpoint, every field, status enums, auth internals, error codes, and a full endpoint inventory (incl. loyalty/shopping). | You need the exact field name / status value / detail. |
| **`openapi.yaml`** | OpenAPI 3.0 spec for the **core** flows (auth, parcels, compartment, points, notifications). Valid; loads in Swagger UI / Redoc / codegen. | You want generated client code, a Postman collection, or an interactive explorer. |

Scope of the OpenAPI file is the **consumer core**. The long tail (loyalty/
"InCoins", shopping "izi", Nimbus AI, sending, returns, payments) is *inventoried*
in `API_REFERENCE.md` Appendix A but not modeled field-by-field.

---

## What you can do with it

- **List your inbound parcels** and their shipping status.
- **Show a parcel's pickup QR code** (the one a Paczkomat scanner reads).
- **Open a locker compartment remotely** to collect a parcel.
- **Find Paczkomaty / pickup points** near a location.
- Log in, refresh tokens, manage tracked parcels, read notifications.

The parcel API is normal JSON-over-HTTPS with Bearer tokens. OAuth token
requests use form-encoded bodies. If you've consumed any REST API, you already
know how this works.

---

## 60-second mental model

1. **Mobile base URL** (note the `/global`):
   `https://api-inmobile-pl.easypack24.net/global`. Browser login uses
   `https://account.inpost-group.com/oauth2/authorize`; token exchange/refresh
   use `https://api-inmobile-pl.easypack24.net/global/oauth2/token`.
2. **Log in** → get an **access token** (`access_token`) + a **refresh token**.
3. Put `Authorization: Bearer <accessToken>` on every call.
4. `GET /v4/parcels/tracked` → your parcels, each with a `status` and (when
   waiting in a locker) an `openCode` + `qrCode`.
5. To collect: either **show the `qrCode`** at the machine, **or** run the
   **remote-open** sequence (`validate → open → status → closed → claim`).

That's the entire happy path. Everything else is detail.

---

## Quick start — the happy path

### 1. Authenticate with Log InPost (OAuth + PKCE)

Generate a random `code_verifier` (43–128 URL-safe characters), retain it locally,
and compute `code_challenge = BASE64URL(SHA256(code_verifier))` without padding.
Also generate and retain a random `state`. Open this URL in a browser, with query
parameter values properly URL-encoded:

```text
https://account.inpost-group.com/oauth2/authorize
  ?client_id=inpost-mobile
  &redirect_uri=https%3A%2F%2Faccount.inpost-group.com%2Fcallback
  &response_type=code
  &response_mode=query
  &scope=openid
  &code_challenge=<challenge>
  &code_challenge_method=S256
  &state=<random-state>
  &nonce=<random-nonce>
  &prompt=login
  &lang=pl
  &theme=light
  &supported_markets=PL%2CIT%2CES%2CPT
```

Complete the official login page's SMS and, when requested, e-mail verification.
The callback is `https://account.inpost-group.com/callback?code=...&state=...`.
Check the callback origin/path and `state` before exchanging the code. A custom
client can intercept navigation to this registered callback in a WebView, or
have the user copy its full address into the client. The callback page itself
can be blank; the query parameters are what the client needs.

Exchange the single-use code on the **mobile API host**:

```http
POST https://api-inmobile-pl.easypack24.net/global/oauth2/token
Content-Type: application/x-www-form-urlencoded

client_id=inpost-mobile&grant_type=authorization_code&redirect_uri=https%3A%2F%2Faccount.inpost-group.com%2Fcallback&code_verifier=<verifier>&code=<code>
```

The response contains `access_token`, `refresh_token`, `id_token` and
`expires_in` (seconds). No client secret or Bearer header is needed for this
exchange. The access token can be used **directly** for the parcel API; no
additional token exchange is required for listing parcels.

Refresh using the same endpoint:

```http
POST https://api-inmobile-pl.easypack24.net/global/oauth2/token
Content-Type: application/x-www-form-urlencoded

client_id=inpost-mobile&grant_type=refresh_token&refresh_token=<refresh_token>
```

Persist **both** returned tokens before making subsequent calls: the refresh
response supplies a replacement refresh token as well as an access token.
Serialize refreshes per account to avoid races with token rotation. A failed
refresh with OAuth `invalid_grant` requires a new browser login.

### 2. List inbound parcels

```http
GET /v4/parcels/tracked
Authorization: Bearer <access_token>
```
→ `{ "parcels": [ ... ], "removedParcelList": [...], "more": false }`

If `more` is `true`, call the **same** endpoint again and append — pagination is
server-stateful, there is no cursor.

### 3. Show the QR / read the codes

Each parcel waiting in a locker has:
- `openCode` — the **numeric** code (locker keypad / remote open),
- `qrCode` — an **opaque string** you render as a QR for the locker's scanner.

### 4. Open the compartment remotely

```
POST /v2/collect/validate          { parcel:{shipmentNumber, openCode}, geoPoint:{lat,lng,accuracy} }
   → { sessionUuid }
POST /v1/collect/compartment/open  { sessionUuid }   → which door, timers
POST /v1/collect/compartment/status{ sessionUuid, expectedStatus:"OPENED" }   (poll)
   … user takes parcel, shuts door …
POST /v1/collect/compartment/status{ sessionUuid, expectedStatus:"CLOSED" }   (poll)
POST /v1/collect/compartment/closed{ sessionUuid }
POST /v1/collect/compartment/claim { sessionUuid, shipmentNumbers:[...] }
```

`sessionUuid` from step 1 threads through every later call.

> **Two backends behind this flow.** There is both a **legacy** implementation
> (bare `/v1|v2/collect/…`, single-market PL) and a newer **global backend**
> (`/remote-opening/v1/collect/…`, country-aware, extra `xCountryCode` header). A
> remote-config flag (`isParcelLockerGlobalBackendEnabled`, **default `false`**)
> picks one per call. The **legacy** paths are the practical default target; the
> two share identical `status`/`closed` semantics. Details in
> `API_REFERENCE.md §7.0`.

---

## Things that are easy to get wrong (read this)

1. **Do not blindly concatenate `/global` with every path.** The default base is
   `https://api-inmobile-pl.easypack24.net/global`, but several documented
   endpoints are root-relative (`/v4/parcels/tracked`). In
   standard HTTP clients, a leading `/` drops the base path. Verify the final
   URL your client sends; tracked-parcel calls work at the host-root URL
   `https://api-inmobile-pl.easypack24.net/v4/parcels/tracked`, while OAuth token
   calls use `/global/oauth2/token` on that same mobile host.

2. **A parcel has TWO different codes.** `openCode` ≠ `qrCode`. `openCode` is the
   short numeric code; `qrCode` is a long opaque token. The QR screen encodes
   `qrCode` **verbatim** — don't QR-encode the `openCode` and expect it to work,
   and don't try to derive one from the other.

3. **OAuth token requests are form-encoded, not JSON.** Retain the PKCE
   verifier until code exchange and check the callback `state`. Store both
   tokens returned by every refresh, and serialize refreshes per account.

4. **`status` is a free-form string, not an enum** in the payload (e.g.
   `ready_to_pickup`, `stack_in_box_machine`, `delivered`). Match on the strings
   listed in `API_REFERENCE.md §5.4`. Use `statusGroup` for a coarse bucket. To
   decide "can I open it now?", check `operations.collect == true`. (A parcel
   field `mobileCollectPossible` used to exist in the v1 API and is easy to
   assume is still here — but the v4 `/parcels/tracked` response does **not**
   return it. Do not gate on it; `operations.collect` is the signal.)

5. **Remote open is GPS-gated, but the GPS is whatever you send.** `validate`
   takes a `geoPoint` from the client. The proximity check is effectively a
   client-side guardrail, not a hard server gate — which is why CLI clients can
   open lockers without "being close". Don't build on the assumption that the
   server strictly enforces your location; equally, don't abuse it.

6. **The locker timers are durations, not timestamps.** `actionTime`,
   `confirmActionTime`, `openCompartmentWaitingTime` are **milliseconds to add to
   "now"**, not absolute epoch times.

7. **`expectedStatus` only has two values:** `OPENED` and `CLOSED`. Poll for
   `OPENED` after opening, then `CLOSED` after the door shuts.

8. **`/v4/parcels/tracked` takes no query parameters** and returns the full set.
   There's no `updatedAfter`/delta sync. `updatedUntil` is returned, but a
   simple client should re-fetch and reconcile (honor `removedParcelList[]`).

9. **Dates are ISO-8601 with offset** (`2024-06-30T14:30:00.000+02:00`). Parse as
   offset date-times, not naive local times.

10. **Tokens: refresh reactively on `401`.** There's no documented proactive
    skew. Simplest correct behavior: call → on 401, refresh once → save the
    returned access/refresh token pair → retry.

11. **TLS certificate pinning** in native clients does **not** affect your
    own client calling the API; it only affects traffic inspection of those
    clients.

---

## Headers: what you actually need

Minimal set used by the older v1 client
[`alufers/inpost-cli`](https://github.com/alufers/inpost-cli):

```
Authorization: Bearer <accessToken>     # on authenticated calls
Content-Type: application/json            # on JSON requests with a body
Accept: application/json
```

Common clients may also send `User-Agent`, `Device-ID`, `device-uid`,
`Session-ID`, `X-App-Id`, `Accept-Language`. The older client omits these;
that does not establish their requirements for the OAuth/v4 flow. An
app-style header set includes `X-App-Id: pl.inpost.inmobile`, `Protocol-Version: 1.1`,
`Accept-Language: pl`, persisted UUIDs for `Device-ID` and `device-uid`, a
`Session-ID` UUID, and the app-style User-Agent
`InPost-Mobile/4.21.1(42101000) (Android 16; Pixel 9; Google tokay; pl)`.
The code exchange and refresh work without those device headers and use
`Content-Type: application/x-www-form-urlencoded`. See
`API_REFERENCE.md §3, §11b`.

---

## Errors

Parcel/locker JSON errors use this envelope:

```jsonc
{
  "error": "sessionExpired",   // machine-readable, camelCase — switch on this
  "description": "…",          // human text (nullable)
  "status": 409,               // echoes HTTP status (nullable)
  "diagnosticId": "…"          // quote this if you contact InPost
}
```

OAuth token errors use a separate format, e.g. HTTP 400
`{"error":"invalid_grant"}`. Other failures may have an empty or non-JSON body;
check the HTTP status before parsing JSON.

Codes you'll meet on the locker flow: `invalidSession`, `sessionExpired`,
`invalidSessionState`, `invalidCompartmentState`, `invalidParcelState`,
`cannotFindCompartment`, `boxMachineNotFound`. On `sessionExpired` /
`invalidSessionState`, start over from `validate`. Full list in
`API_REFERENCE.md §11b`.

---

## Using the OpenAPI file

```bash
# Interactive docs
npx @redocly/cli preview-docs openapi.yaml
# or
npx @redocly/cli lint openapi.yaml        # validates (warnings are cosmetic)

# Generate a client (examples)
npx @openapitools/openapi-generator-cli generate -i openapi.yaml -g typescript-fetch -o ./client-ts
npx @openapitools/openapi-generator-cli generate -i openapi.yaml -g swift5 -o ./client-swift
npx @openapitools/openapi-generator-cli generate -i openapi.yaml -g python -o ./client-py
```

Import `openapi.yaml` into Postman/Insomnia to get a ready request collection.
Set the base server, drop your Bearer token into the auth, and go.

Notes for codegen consumers:
- Some long-tail response bodies (sent parcels, notifications) are modeled as
  open objects (`additionalProperties: true`) — fields not fully extracted.
- A few nested objects are referenced via `allOf`; treat them as optional.

---

## Notes & Keeping This Current

- This project was built on publicly available information and data.
- Cross-check useful behavior against
  [`alufers/inpost-cli`](https://github.com/alufers/inpost-cli), especially the
  auth model, the `openCode`/`qrCode` split, and the collect state machine.
- Verify exact HTTP statuses, optional headers, and edge-case responses against
  live behavior before relying on them.
- When behavior changes, update `openapi.yaml` and note the change here.

---

## A note on responsible use

This documentation is for interoperability and personal use. Don't use it to
hammer InPost's servers, scrape other people's data, or anything you wouldn't be
comfortable explaining. The remote-open flow in particular is tied to real
physical lockers — only open compartments for parcels that are actually yours.
