# D-avocado API Specification

> REST API documentation for the avocado ripeness stage classification and optimal D-day prediction app.
Companion document: `D-avocado_DB_Specification.md` (**v1.0**)
Version: **v1.0 implementation baseline** — flat scan model (`/scans`), room-temperature only (`storage_condition` removed), global target stage (1-5), notifications via `push_enabled` + `advance_notice_days`, decimal `days_to_target`.

---

## 0. Common Conventions

| Item | Value |
| --- | --- |
| Base URL | `https://api.d-avocado.app/v1` |
| Data format | Requests and responses use `application/json`; image upload uses `multipart/form-data` |
| Authentication | `Authorization: Bearer {access_token}` (JWT, single token, no refresh token) |
| Timestamps | ISO 8601, UTC |
| Character encoding | UTF-8 |
| Pagination | `?limit=20&cursor={next_cursor}`; response includes `next_cursor` or `null` |

### Common Error Response

```json
{ "error": { "code": "ERROR_CODE", "message": "Human-readable message" } }
```

**ErrorCode values, mapped 1:1 to the backend enum**

| HTTP | code | Situation |
| --- | --- | --- |
| 400 | `BAD_REQUEST` | Invalid parameter format |
| 401 | `UNAUTHORIZED` | Missing token |
| 401 | `TOKEN_EXPIRED` | Expired token |
| 401 | `INVALID_CREDENTIALS` | Email/password mismatch |
| 403 | `FORBIDDEN` | Access to another user's resource |
| 404 | `NOT_FOUND` | Resource not found |
| 409 | `DUPLICATE_EMAIL` | Email already registered |
| 413 | `FILE_TOO_LARGE` | Uploaded image is too large |
| 422 | `VALIDATION_FAILED` | Value validation failed, such as settings range |
| 422 | `NO_AVOCADO_DETECTED` | No avocado detected during crop/preprocessing |
| 502 | `INFERENCE_SERVICE_UNAVAILABLE` | Cloud Run call failed or timed out |
| 500 | `INTERNAL_ERROR` | Server error |

---

## 1. Authentication (`/auth`)

### 1.1 Sign Up — `POST /auth/signup` (No Authentication Required)

```json
{ "email": "user@example.com", "password": "PlainPassword123!", "nickname": "Avocado Lover" }
```

- `email` is required and unique.
- `password` is required and hashed on the server.
- `nickname` is optional.
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

Password reset is excluded from v1.0 because there is no corresponding frontend screen.

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

### 2.2 Update Settings — `PATCH /users/me/settings`

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

**Response 200** — updated settings object.

### 2.3 Register or Update Push Token — `PUT /users/me/push-token`

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
| `temp_celsius` | number | No | Room temperature entered by the user. Sent only after the frontend adds temperature input; otherwise the AI service uses a default value |

**Processing Flow on GCP**

1. Client uploads image to Spring.
2. Spring stores the original image in GCS at `raw/{user_id}/{scan_id}.jpg`.
3. Spring calls Cloud Run with the image and payload below, authenticated with a GCP service account ID token.

```json
{ "target_stage": 3, "temp_celsius": 22.0 }
```

- `target_stage` is read by the server from `users.preferred_stage`; the client does not send it. The value is snapshotted onto the scan row.
- `storage_condition` is no longer sent because v1.0 is room-temperature only.
- `temp_celsius` is included only when provided.

4. Cloud Run crops the image, runs ResNet-18 inference, calculates temperature-adjusted `days_to_target`, and returns `{ predicted_stage, stage_probs, days_to_target, estimated_peak_date, model_version }` plus the cropped image.
5. Spring stores the cropped image in GCS and updates `images.cropped_url`.
6. Spring inserts the `scans` row in the same transaction.
7. If `push_enabled=true`, Spring schedules one `notifications` row for `estimated_peak_date - advance_notice_days`.

**Response 201**

```json
{
  "id": 123,
  "predicted_stage": 2,
  "confidence": 0.87,
  "stage_probs": [0.05, 0.87, 0.06, 0.01, 0.01],
  "target_stage": 3,
  "days_to_target": 3.0,
  "estimated_peak_date": "2026-07-23",
  "model_version": "resnet18_v3",
  "image": {
    "cropped_url": "https://storage.googleapis.com/.../123_crop.jpg?X-Goog-Signature=..."
  },
  "created_at": "2026-07-20T09:31:00Z",
  "display": {
    "stage_label": "Unripe",
    "dday_text": "D-3",
    "status": "ripening"
  }
}
```

- `days_to_target` can be decimal or negative. `display.dday_text` is rounded for display, such as `D-3`; `status` is `ripening` for positive values, `eat_now` near zero, and `overripe` for negative values.
- `cropped_url` is a TTL-signed URL because the bucket is private.
- "Re-scan" calls this endpoint again and creates a new row.
- Failure cases: 422 `NO_AVOCADO_DETECTED`, 413 `FILE_TOO_LARGE`, 502 `INFERENCE_SERVICE_UNAVAILABLE`.

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
      "days_to_target": 2.0,
      "estimated_peak_date": "2026-07-22",
      "created_at": "2026-07-14T09:31:00Z",
      "display": { "stage_label": "Unripe", "dday_text": "D-2", "status": "ripening" },
      "notification": { "status": "scheduled" },
      "thumbnail_url": "https://storage.googleapis.com/.../123_crop.jpg?X-Goog-Signature=..."
    }
  ],
  "next_cursor": null
}
```

- `notification.status` is `scheduled`, `sent`, or `none`, and is used for the bell icon in each History row.

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

Returns scan detail, including image URLs, so the Result screen can be reconstructed from History. Accessing another user's scan returns 403.

### 3.5 Delete Scan — `DELETE /scans/{id}` → **204**

Deletes the scan and cascades to its image and notification rows.

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

- **AI service:** Cloud Run + FastAPI + ResNet-18. Owns classification and `days_to_target` calculation. Payload: `{ target_stage, temp_celsius? }`. Response: `{ predicted_stage, stage_probs, days_to_target, estimated_peak_date, model_version }` plus cropped image. There is no `storage_condition`, no beta refit logic in Spring. Scale-to-zero and cold starts are acceptable. Spring authenticates with a GCP service account ID token.
- **Image storage:** GCS bucket `d-avocado-images` with `raw/` and `cropped/` prefixes. The bucket is private and clients receive TTL-signed URLs. Paths include `{user_id}/{scan_id}`.
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
| 11 | Password reset / profile edit | Present | Excluded because there is no frontend screen |
| 12 | Sticker API | Already removed | Not present |

---

## 8. Open Decisions

- Whether `temp_celsius` should become required after the frontend temperature input is finalized.
- Push infrastructure: FCM for both Android and iOS versus direct APNs. For iOS-first, one token field is enough either way; Android expansion may require a `platform` column.
- Bell toggle behavior: automatic scheduling as the target approaches versus explicit user-controlled scheduling.
- Image upload path: Spring-mediated upload, as currently specified, versus direct GCS upload through signed URLs.
- `dday_text` rounding rule: decide whether decimal `days_to_target` is rounded or ceiled, and keep it aligned with the database specification.
