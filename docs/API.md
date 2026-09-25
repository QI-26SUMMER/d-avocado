# D-avocado API Specification

> REST API documentation for the avocado ripeness stage classification and optimal D-day prediction app.
Companion document: [`Database.md`](Database.md) (**v1.0**)
Version: **v1.0 implementation baseline** — flat scan model (`/scans`), room-temperature only (`storage_condition` removed), global target stage (1-5), notifications via `push_enabled` + `advance_notice_days`, decimal `days_to_target`.

---

## 0. Common Conventions

| Item | Value |
| --- | --- |
| Base URL | `https://davocado-backend-950009771312.us-central1.run.app` (Cloud Run; no `/v1` path prefix) |
| Data format | Requests and responses use `application/json`; image upload uses `multipart/form-data` |
| Response envelope | Every successful response body is wrapped as `{ "data": ... }` (except `GET /health`). The examples below show the contents of `data`. |
| Authentication | `Authorization: Bearer {access_token}` (JWT, single token, no refresh token) |
| Timestamps | ISO 8601, UTC |
| Character encoding | UTF-8 |
| Pagination | `?limit=20&cursor={next_cursor}` (`limit` default 20, max 100); `next_cursor` is the numeric `id` of the last item, or `null` on the last page. Items are ordered by `id` descending |

### Common Error Response

```json
{ "error": { "code": "ERROR_CODE", "message": "Human-readable message" } }
```

**ErrorCode values, mapped 1:1 to the backend enum**

| HTTP | code | Situation |
| --- | --- | --- |
| 400 | `BAD_REQUEST` | Invalid parameter format (reserved; malformed JSON or parameter type errors currently return 500 `INTERNAL_ERROR`) |
| 401 | `UNAUTHORIZED` | Missing, malformed, or invalid token |
| 401 | `TOKEN_EXPIRED` | Expired token |
| 401 | `INVALID_CREDENTIALS` | Email/password mismatch |
| 403 | `FORBIDDEN` | Access to another user's resource |
| 404 | `NOT_FOUND` | Resource not found |
| 409 | `DUPLICATE_EMAIL` | Email already registered |
| 413 | `FILE_TOO_LARGE` | Uploaded image is too large (max 10 MB per file, 12 MB per request) |
| 422 | `VALIDATION_FAILED` | Value validation failed, such as settings range |
| 422 | `NO_AVOCADO_DETECTED` | The AI service returned an error for the image (no object detected during crop, or the image could not be processed) |
| 502 | `INFERENCE_SERVICE_UNAVAILABLE` | Cloud Run call failed or timed out |
| 500 | `INTERNAL_ERROR` | Server error |

---

## 1. Authentication (`/auth`)

### 1.1 Sign Up — `POST /auth/signup` (No Authentication Required)

```json
{ "email": "user@example.com", "password": "PlainPassword123!", "nickname": "Avocado Lover" }
```

- `email` is required, unique, a valid email address, and at most 255 characters.
- `password` is required, 8–100 characters, and hashed on the server.
- `nickname` is optional, at most 50 characters.
- New users are created with default settings: `preferred_stage=3`, `push_enabled=true`, `advance_notice_days=1`.

**Response 201**

```json
{ "id": 1, "email": "user@example.com", "nickname": "Avocado Lover", "created_at": "2026-07-20T09:30:00Z" }
```

Failure cases: 409 `DUPLICATE_EMAIL`, 422 `VALIDATION_FAILED`.

### 1.2 Log In — `POST /auth/login` (No Authentication Required)

```json
{ "email": "user@example.com", "password": "PlainPassword123!" }
```

**Response 200** — token plus user settings, so the Settings screen can render immediately without a separate fetch.

```json
{
  "access_token": "eyJhbGciOi...",
  "token_type": "Bearer",
  "expires_in": 1209600,
  "user": {
    "id": 1,
    "email": "user@example.com",
    "nickname": "Avocado Lover",
    "preferred_stage": 3,
    "push_enabled": true,
    "advance_notice_days": 1
  }
}
```

Failure case: 401 `INVALID_CREDENTIALS`.

### 1.3 Log Out — `POST /auth/logout` → **204**

- Because the app uses a single token with no refresh token, there is no server-side token invalidation target. The client deletes the local token.
- The user's `push_token` is cleared to prevent the device from receiving future notifications.

### 1.4 Password Reset — `POST /auth/password/reset` → **202** (No Authentication Required)

The endpoint exists but is a no-op stub in v1.0 because there is no corresponding frontend screen.

---

## 2. User and Settings (`users`)

The Settings screen uses these endpoints for the profile card, preferred ripeness, and notification settings. Push token registration is called automatically by the app at launch, not from a visible settings screen.

### 2.1 Get My Profile and Settings — `GET /users/me`

Used to hydrate the Settings screen after app relaunch when the login response is not cached.

**Response 200**

```json
{
  "id": 1,
  "email": "user@example.com",
  "nickname": "Avocado Lover",
  "preferred_stage": 3,
  "push_enabled": true,
  "advance_notice_days": 1,
  "created_at": "2026-07-01T00:00:00Z"
}
```

### 2.2 Update Profile — `PATCH /users/me`

Updates the nickname (at most 50 characters). Returns the same shape as `GET /users/me`.

```json
{ "nickname": "New Nickname" }
```

### 2.3 Update Settings — `PATCH /users/me/settings`

Updates the three Settings screen values. Partial updates are allowed.

```json
{ "preferred_stage": 4, "push_enabled": true, "advance_notice_days": 2 }
```

| Field | Validation | Description |
| --- | --- | --- |
| `preferred_stage` | 1-5 | Global target stage, shown as Preferred Ripeness |
| `push_enabled` | boolean | Push Notifications master toggle |
| `advance_notice_days` | 0-3 | Advance Notice; 0 means same day, 1-3 means N days before |

- Out-of-range values return 422 `VALIDATION_FAILED`.
- Changing `preferred_stage` does not apply retroactively to old scans. Each scan stores the target stage at capture time in `target_stage`, so the new value applies only to future scans.

**Response 200**

```json
{ "preferred_stage": 4, "push_enabled": true, "advance_notice_days": 2 }
```

### 2.4 Register or Update Push Token — `PUT /users/me/push-token`

Called automatically at app launch, not from the profile screen. The app requests OS notification permission, receives an FCM/APNs token, and stores it on the server. The call is idempotent and overwrites the previous token when it changes.

```json
{ "push_token": "fcm_or_apns_token_xyz" }
```

**Response 200**

```json
{ "push_token_registered": true }
```

- If `push_enabled=true` but no token is registered, notifications may be scheduled but delivery is skipped.
- On logout, `POST /auth/logout` clears the user's `push_token`. A future `DELETE /users/me/push-token` endpoint may also be used for explicit removal.

---

## 3. Scans (`scans`)

The previous `avocados` + `predictions` model is consolidated into a single `scans` resource. There is no separate fruit registration step: one photo creates one scan record.

### 3.1 Run Scan (Image Upload) — `POST /scans`

**Content-Type:** `multipart/form-data`

| Part | Type | Required | Description |
| --- | --- | --- | --- |
| `image` | file | Yes | One image from a single capture |
| `source` | string | Yes | `camera` or `gallery` |
| `temp_celsius` | number | No | Room temperature set by the user in Settings (the iOS app always sends it; slider 10–25 °C, default 24 °C). If omitted, the AI service uses 20 °C |

**Processing Flow on GCP**

1. Client uploads the image to Spring.
2. Spring calls the AI service on Cloud Run, authenticated with a GCP service account ID token. The request body uses the Vertex AI prediction format:

```json
{ "instances": [ { "b64": "<base64 image>" } ], "parameters": { "target_stage": 3, "temp_celsius": 22.0 } }
```

- `target_stage` is read by the server from `users.preferred_stage`; the client does not send it. The value is snapshotted onto the scan row.
- `storage_condition` is no longer sent because v1.0 is room-temperature only.
- `temp_celsius` is included only when provided.

3. The AI service removes the background and crops the image (InSPyReNet), classifies the ripeness stage (Vertex AI AutoML in production), calculates the temperature-adjusted `days_to_target`, and returns `{ predicted_stage, label, hint, confidence, stage_probs, model_version, days_to_target, cropped_b64 }`.
4. Spring calculates `estimated_peak_date` as today (UTC) + `round(days_to_target)`.
5. Spring inserts the `scans` row, uploads the original image to GCS at `raw/{user_id}/{scan_id}.jpg` and the cropped image at `cropped/{user_id}/{scan_id}.jpg`, then inserts the `images` row.
6. If `push_enabled=true`, Spring schedules one `notifications` row for `estimated_peak_date - advance_notice_days`.

**Response 201**

```json
{
  "id": 123,
  "predicted_stage": 2,
  "confidence": 0.87,
  "stage_probs": [0.05, 0.87, 0.06, 0.01, 0.01],
  "target_stage": 3,
  "temp_celsius": 22.0,
  "days_to_target": 3.0,
  "estimated_peak_date": "2026-07-23",
  "model_version": "automl:qiautoml1:{endpoint_id}",
  "image": {
    "original_url": "https://storage.googleapis.com/.../raw/1/123.jpg?X-Goog-Signature=...",
    "cropped_url": "https://storage.googleapis.com/.../cropped/1/123.jpg?X-Goog-Signature=..."
  },
  "created_at": "2026-07-20T09:31:00Z",
  "display": {
    "dday_text": "D-3",
    "status": "ripening"
  }
}
```

- `days_to_target` can be decimal or negative. `display.dday_text` is rounded half-up to an integer: `D-N` / `ripening` when positive, `D-Day` / `eat_now` when it rounds to 0, and `D+N` / `overripe` when negative. `display` is omitted when `days_to_target` is null. The stage label is not returned; the client derives it from `predicted_stage`.
- `model_version` is `automl:{project}:{endpoint_id}` for the AutoML backend, or the experiment id (e.g. `P1_general_resnet18_paper_aug_oversample`) for the ResNet-18 backend.
- Image URLs are V4-signed URLs (15-minute TTL by default) because the bucket is private. `image` is `null` when image storage is disabled.
- "Re-scan" calls this endpoint again and creates a new row.
- Failure cases: 422 `VALIDATION_FAILED` (invalid `source`, empty image, or non-numeric temperature), 422 `NO_AVOCADO_DETECTED`, 413 `FILE_TOO_LARGE`, 502 `INFERENCE_SERVICE_UNAVAILABLE`.

### 3.2 List Scans (History) — `GET /scans?limit=20&cursor=...`

Returns the user's scans in newest-first order for the History screen.

**Response 200**

```json
{
  "items": [
    {
      "id": 123,
      "predicted_stage": 2,
      "target_stage": 3,
      "temp_celsius": 22.0,
      "days_to_target": 2.0,
      "estimated_peak_date": "2026-07-22",
      "created_at": "2026-07-14T09:31:00Z",
      "display": { "dday_text": "D-2", "status": "ripening" },
      "notification": { "status": "scheduled" },
      "thumbnail_url": "https://storage.googleapis.com/.../raw/1/123.jpg?X-Goog-Signature=..."
    }
  ],
  "next_cursor": null
}
```

- `notification.status` is `scheduled`, `sent`, or `none`, and is used for the bell icon in each History row.
- `thumbnail_url` is the signed URL of the original photo (not the crop).

### 3.3 Scan Stats — `GET /scans/stats`

Used by the History screen summary cards: Total, Notified, Pending.

**Response 200**

```json
{ "total": 5, "notified": 3, "pending": 2 }
```

- `total` is the number of all scans.
- `notified` is the number of sent notifications.
- `pending` is the number of scheduled notifications.

### 3.4 Get Scan Detail — `GET /scans/{id}`

Returns scan detail (same shape as the `POST /scans` response), including image URLs, so the Result screen can be reconstructed from History. Accessing another user's scan returns 403; an unknown id returns 404.

### 3.5 Delete Scan — `DELETE /scans/{id}` → **204**

Deletes the scan and cascades to its image and notification rows. The image objects in GCS are not deleted.

### 3.6 Toggle Scan Notification — `PATCH /scans/{id}/notification`

Used when the user toggles the bell icon in a History row.

```json
{ "enabled": false }
```

- `true` creates or keeps a `scheduled` notification if the target date is still in the future.
- `false` deletes the `scheduled` notification for that scan. `sent` notifications are preserved.
- If the target date has already passed, scheduling is ignored or returns 422, depending on final policy.

**Response 200**

```json
{ "id": 123, "notification": { "status": "none" } }
```

---

## 4. Notifications (`notifications`)

The server schedules notifications, sends push messages through the user's `push_token` at the scheduled time, then marks them as `sent`. The client also reads this list for the in-app notification inbox opened from the header bell icon.

### 4.1 List Notifications — `GET /notifications?status=sent&limit=20`

**Response 200**

```json
{
  "items": [
    {
      "id": 5,
      "scan_id": 45,
      "scheduled_at": "2026-07-17T09:00:00Z",
      "sent_at": "2026-07-17T09:00:01Z",
      "status": "sent",
      "payload": { "title": "Almost ready 🥑", "body": "Tomorrow is the target eating day!" }
    }
  ],
  "next_cursor": null
}
```

- `status` filter values: `scheduled`, `sent`.
- There is no notification type field. Each scan has at most one notification: N days before the target stage.
- Delivery channel: real FCM/APNs push. Delivery requires both `push_enabled=true` and a registered `push_token`. If the token is missing, the notification may remain scheduled but delivery is skipped. The list also acts as delivery history and the in-app inbox.

---

## 5. Server Internals (Not Exposed to Clients)

- **AI service:** Cloud Run + FastAPI (`avocado-serving`). Classification runs on a Vertex AI AutoML endpoint in production (`MODEL_BACKEND=automl`); the in-house ResNet-18 remains available as `MODEL_BACKEND=resnet`. Owns background removal, classification, and `days_to_target` calculation. Parameters: `{ target_stage, temp_celsius? }`. Response: `{ predicted_stage, label, hint, confidence, stage_probs, model_version, days_to_target, cropped_b64 }`. `estimated_peak_date` is calculated in Spring. There is no `storage_condition` and no ripening-coefficient logic in Spring. Spring authenticates with a GCP service account ID token.
- **Image storage:** GCS bucket `qi-2026summer-avocado-images` with `raw/` and `cropped/` prefixes. The bucket is private and clients receive TTL-signed URLs. Paths include `{user_id}/{scan_id}`.
- **Target snapshot:** On `POST /scans`, Spring reads `users.preferred_stage`, sends it to the AI service, and stores it in `scans.target_stage`.

---

## 6. Representative User Flow

1. `POST /auth/signup` → `POST /auth/login` to receive token and settings.
2. At app launch, request notification permission and call `PUT /users/me/push-token` automatically.
3. Optionally update Settings with `PATCH /users/me/settings`.
4. Run a scan with `POST /scans`; show the Result screen from `display`.
5. Load History with `GET /scans` and `GET /scans/stats`.
6. Open an item with `GET /scans/{id}` to reconstruct the Result screen.
7. Toggle the bell with `PATCH /scans/{id}/notification`.
8. Open the inbox with `GET /notifications`.
9. Delete a scan with `DELETE /scans/{id}`.

---

## 7. v0.3 to v1.0 Change Summary

| # | Area | v0.3 | v1.0 |
| --- | --- | --- | --- |
| 1 | Tracking model | `POST /avocados` + `POST /avocados/{id}/predictions` | Single `POST /scans` |
| 2 | List view | `GET /avocados` for fruit entities | `GET /scans` plus `GET /scans/stats` |
| 3 | Storage environment | Required `storage_condition` | Removed; room-temperature only |
| 4 | Temperature | Required `room_temp_celsius` for room storage | Optional `temp_celsius` |
| 5 | Target stage | Per-fruit `target_stage`, 3-4 | Global `users.preferred_stage`, 1-5, snapshotted per scan |
| 6 | Notification settings | `PATCH .../notification-settings` with type toggles | `PATCH /users/me/settings` with `push_enabled` + `advance_notice_days` |
| 7 | Notification types | `peak_soon`, `peak_today`, `overripe` | Removed; one notification per scan |
| 8 | Cloud Run payload | `target_stage` + `storage_condition` + optional temperature | `target_stage` + optional `temp_celsius` |
| 9 | `days_to_target` | Integer | Decimal allowed |
| 10 | Push token registration | Included in `PATCH /users/me` profile update | Dedicated `PUT /users/me/push-token`, called automatically at app launch |
| 11 | Password reset / profile edit | Present | Password reset kept as a no-op stub (no frontend screen); `PATCH /users/me` updates the nickname only |
| 12 | Sticker API | Already removed | Not present |

---

## 8. Open Decisions

- Whether `temp_celsius` should become required (the iOS app already always sends it).
- Push infrastructure: FCM for both Android and iOS versus direct APNs. For iOS-first, one token field is enough either way; Android expansion may require a `platform` column.
- Bell toggle behavior: automatic scheduling as the target approaches versus explicit user-controlled scheduling.
- Image upload path: Spring-mediated upload, as currently specified, versus direct GCS upload through signed URLs.
