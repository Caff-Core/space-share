# Protocol v1 (outline)

Status: **draft outline**. Required by plan §6 before implementation. Field-level schemas become normative when `packages/protocol` (Zod) and `crates/protocol-types` (serde) land in Phase 3; both are tested against `packages/test-fixtures`. Any change after Phase 2 is recorded here and in an ADR before multiple implementations are built.

## 1. Transport

- HTTPS over TLS 1.3 only, on an application-owned listener. Receiver listens; sender connects.
- Mutual TLS. Each peer presents its self-signed installation certificate. Peers are verified by exact SHA-256 SPKI pin (see `architecture.md` §5). No CA chain, hostname or SNI checks; validation is never disabled.
- Listener binds to the LAN interface address(es) on an OS-assigned port (or a configured fixed port for firewall rules). Never binds publicly by default.
- JSON request/response bodies: `Content-Type: application/json; charset=utf-8`. File bodies: `application/octet-stream`, streamed.
- All paths are prefixed with the major version: `/v1/...`.

## 2. Encoding conventions

| Item | Rule |
| --- | --- |
| Byte lengths | Decimal strings matching `^(0|[1-9][0-9]{0,15})$`, ≤ `MAX_FILE_BYTES`. Avoids JS integer precision loss |
| Identifiers | `offerId`, `fileId`, `sessionId`: 128-bit random, base64url without padding (22 chars) |
| Digests | SHA-256, lowercase hex (64 chars) |
| Fingerprints | SHA-256 of SPKI DER, lowercase hex (64 chars); displayed grouped |
| Timestamps | RFC 3339 UTC strings |
| Display names / filenames | UTF-8, NFC-normalised, length limits in §7 |
| Unknown JSON fields | Rejected on security-relevant objects (manifest, pair); ignored elsewhere only if marked extensible |

## 3. QR / manual connection payload

Encoded as a URI so it is human-inspectable and versioned:

```text
spaceshare://v1/pair?h=<host>&p=<port>&fp=<receiverFingerprint>&s=<pairingSecret>&n=<displayName>&e=<expiresAt>
```

- `h` may repeat for multiple interface addresses (IPv4 and IPv6).
- `s`: 128-bit random, base64url, single use, expires (default 5 minutes).
- Manual connection = the user enters host, port, fingerprint (or compares a short fingerprint code) and pairing code. Address or name alone never authenticates.

## 4. Version negotiation

- Every JSON body carries `"protocol": 1`.
- `GET /v1/hello` (unauthenticated, TLS required) returns `{ protocol: { min, max }, app: "x.y.z" }`. The sender picks the highest common major or fails with a clear "incompatible version" error.
- Unknown major versions are rejected with `426` and `{ code: "protocol_unsupported", supported: { min, max } }`.

## 5. Operations

| # | Method & path | Auth | Purpose | Success | Notable errors |
| --- | --- | --- | --- | --- | --- |
| 1 | `GET /v1/hello` | TLS pin only | Version + receiver identity info | 200 | 426 |
| 2 | `POST /v1/pair` | Pairing secret in body | Exchange identities; issue session token bound to sender SPKI | 201 `{ sessionId, token, expiresAt, receiver: {name, fingerprint} }` | 401 bad/expired/used secret, 429 |
| 3 | `POST /v1/offers` | Session token | Submit manifest for review | 201 `{ offerId, status: "pending_acceptance" }` | 400 invalid manifest, 413 limits, 409 offer already pending |
| 4 | `GET /v1/offers/{offerId}` | Session token | Offer status (bounded polling, long-poll ≤ 25 s) | 200 `{ status, files[] }` | 404 |
| 5 | (local only) accept / reject | Receiver UI action | Changes offer status; never exposed over the network | n/a | n/a |
| 6 | `PUT /v1/offers/{offerId}/files/{fileId}` | Session token + accepted offer | Stream one file body; `Content-Length` must equal manifest length | 200 `{ received, digest }` once body ends | 403 not accepted, 409 already uploading/committed, 413/400 length mismatch, 507 storage |
| 7 | `POST /v1/offers/{offerId}/files/{fileId}/commit` | Session token | Sender's length + digest; receiver verifies and atomically commits | 200 `{ state: "completed", storedName }` | 422 digest/length mismatch → file failed, temp deleted |
| 8 | `POST /v1/offers/{offerId}/cancel` | Session token (sender) or local UI (receiver) | Stop active stream; delete incomplete temps | 200 | 404 |
| 9 | `GET /v1/offers/{offerId}/result` | Session token | Authoritative per-file and batch outcomes | 200 | 404 |

Authorization rules:

- Token is bound to sender SPKI fingerprint; a token presented on a TLS connection with a different client certificate is rejected (`401`).
- Every file request is bound to `(session, offerId, fileId)`. A file can be uploaded only after acceptance, only once concurrently, and never after `completed`.
- Sequential MVP: at most **one active upload per offer**; global receive concurrency limited by `MAX_ACTIVE_UPLOADS`.

## 6. Manifest

```json
{
  "protocol": 1,
  "offerId": "base64url-22",
  "sender": { "displayName": "Ada's laptop" },
  "files": [
    { "fileId": "base64url-22", "name": "report.pdf", "size": "1048576", "mediaType": "application/pdf" }
  ]
}
```

- `name`: display filename only, never a path. Receiver sanitises it before use (reject separators, `..`, absolute paths, control chars, reserved names; truncate to limit preserving extension).
- `size`: decimal string; total = sum, computed with 64-bit integers natively and BigInt in TS.
- `mediaType`: optional, descriptive only; never used for trust or opening behaviour.
- `fileId` values are unique within the offer; duplicates → `400`.

## 7. Limits (central defaults)

Stored in `packages/protocol/src/limits.ts` and mirrored in `crates/protocol-types`; changing one without the other fails fixture tests.

| Limit | Default | Notes |
| --- | --- | --- |
| `MAX_FILE_BYTES` | 64 GiB (`68719476736`) | Per file |
| `MAX_FILES_PER_OFFER` | 10 000 | |
| `MAX_TOTAL_OFFER_BYTES` | 1 TiB | Also bounded by free storage check |
| `MAX_MANIFEST_BYTES` | 2 MiB | JSON body size |
| `MAX_JSON_BODY_BYTES` | 64 KiB | Non-manifest requests |
| `MAX_FILENAME_BYTES` | 255 (UTF-8) | Linux `NAME_MAX` |
| `MAX_DISPLAY_NAME_CHARS` | 64 | |
| `PAIRING_SECRET_TTL` | 5 min | Single use |
| `SESSION_TOKEN_TTL` | 30 min idle, 12 h absolute | Revoked when receive session stops |
| `OFFER_ACCEPT_TIMEOUT` | 5 min | Then `expired` |
| `UNAUTHENTICATED_CONN_LIFETIME` | 10 s | Time to complete TLS + `hello`/`pair` |
| `STREAM_IDLE_TIMEOUT` | 30 s | No body bytes → upload fails |
| `PAIR_ATTEMPTS_PER_MINUTE` | 5 per source address | 429 afterwards |
| `REQUESTS_PER_SECOND` | 50 per session | Excluding streamed body bytes |
| `MAX_ACTIVE_UPLOADS` | 1 per offer, 4 global | |
| `PARTIAL_RETENTION` | 24 h | Stale temps deleted at startup |
| `HISTORY_RETENTION` | 90 days / 500 entries | User-configurable, clearable |

## 8. Integrity and commit

- Sender computes SHA-256 while reading; receiver computes it while writing. File bytes are never buffered whole.
- Success requires: received length == manifest size == commit length, **and** receiver digest == sender digest, **and** a durable write (fsync temp file, then atomic rename into the destination; on Android SAF exports, a verified copy then delete temp).
- Digest proves integrity, TLS pin proves identity; neither replaces the other.
- Source changes: sender re-checks size/mtime (desktop) or uses a stable handle; mismatch → file `failed` with `source_changed`.

## 9. Failure, retry and idempotency

- Retry restarts a failed file **from zero** with the same `fileId`; allowed only from `failed`. The receiver discards any temp for that file first.
- `commit` is idempotent: repeating it for an already-completed file with the same digest returns the same `200` response; a different digest returns `409`.
- Lost acknowledgement: sender must call `GET .../result` (or repeat `commit`) before concluding failure. Never assume the receiver did not store the file.
- Reconnect: a new TLS connection with the same client certificate and a valid token resumes control operations. Byte-offset resume is out of scope for v1.
- Cancel: idempotent; completed files are kept and reported.

## 10. Error body

```json
{ "protocol": 1, "error": { "code": "digest_mismatch", "message": "Human readable, no secrets", "retryable": true } }
```

Initial codes: `protocol_unsupported`, `invalid_request`, `limit_exceeded`, `unauthorized`, `pairing_expired`, `pairing_used`, `rate_limited`, `offer_not_found`, `offer_not_accepted`, `offer_expired`, `offer_canceled`, `file_not_found`, `upload_in_progress`, `already_committed`, `length_mismatch`, `digest_mismatch`, `storage_full`, `write_failed`, `source_changed`, `internal`.

## 11. Open questions for the Phase 2 spike

- Key type (P-256 vs Ed25519) supported by rustls verifier **and** the Android engine path.
- Whether Android needs a foreground service to keep the listener alive while the app is foregrounded but the screen dims.
- Long-poll vs short polling interval for offer status on Android battery/network behaviour.
- Fixed default port vs OS-assigned port, given firewall guidance.
