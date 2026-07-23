# D-avocado Database Specification

> Database design document for the avocado ripeness stage classification and optimal D-day prediction app.
Target DBMS: **PostgreSQL**
Version: **v1.0 implementation baseline** — aligned with the frontend, flat scan model finalized, ready for entity and migration implementation.
Companion document: `D-avocado_API_Specification.md` (**v1.0**)

---

## 0. Design Principles

1. **One scan equals one record.** The app does not register or track individual avocado entities. Taking a photo produces one result, and each result appears as an independent row in History. The core table is `scans`.
2. **Room-temperature only.** There is no `storage_condition` distinction. Refrigerated storage is outside the app scope because ripening prediction is not meaningful under chilling conditions.
3. **Temperature is optional, and the AI service owns D-day calculation.** Spring stores the user-entered temperature as-is. The AI service calculates remaining days using temperature; if the value is missing, the AI service uses a default. Spring has no beta coefficient or refitting logic. Remaining days may be decimal, such as `4.5`.
4. **Target stage is a global user setting.** Settings has a single target value: `users.preferred_stage`, from 1 to 5. Each scan snapshots the value into `scans.target_stage` so historical D-day interpretation does not change later.
5. **Capture uses a single image.** One scan has one image.
6. **Each scan has at most one notification.** The notification fires once, `advance_notice_days` days before the target date. It uses the Settings values `push_enabled` and `advance_notice_days`.

---

## 1. Table Relationships (ERD)

```text
users (1) ──< (N) scans (1) ──< (1) images
  │                 │
  │                 └──< (0..1) notifications
  └───────────────────< (N) notifications
```

- One user has many scans.
- One scan has one image.
- One scan has zero or one notification.
- `notifications` references both `user_id` as the recipient and `scan_id` as the scan target.

**Four tables:** `users`, `scans`, `images`, `notifications`.

---

## 2. Table Specifications

### 2.1 `users` — Users

Stores email login data and the three Settings screen values: `preferred_stage`, `push_enabled`, and `advance_notice_days`.

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | BIGSERIAL | PK | User ID |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL | Login email |
| `password_hash` | VARCHAR(255) | NOT NULL | Hashed password, using bcrypt or argon2. Plaintext is forbidden |
| `nickname` | VARCHAR(50) | NULL | Display name for the Settings profile card |
| `preferred_stage` | SMALLINT | NOT NULL, DEFAULT 3, CHECK (1-5) | Global target stage, shown as Preferred Ripeness |
| `push_enabled` | BOOLEAN | NOT NULL, DEFAULT true | Push Notifications master toggle |
| `advance_notice_days` | SMALLINT | NOT NULL, DEFAULT 1, CHECK (0-3) | Advance Notice. 0 means same day, 1-3 means N days before |
| `push_token` | VARCHAR(255) | NULL | FCM/APNs push token, registered or refreshed automatically at app launch and cleared on logout |
| `last_login_at` | TIMESTAMPTZ | NULL | Last login timestamp, internal use |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Sign-up timestamp |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Last update timestamp |

- `email` is unique.
- `preferred_stage` allows the full 1-5 range, so users can target less-ripe or more-ripe states.
- Push delivery requires both `push_enabled=true` and a registered `push_token`. If `push_enabled=false`, notifications are not scheduled. If the token is missing, delivery cannot occur.
- `push_token` is not edited on the profile screen. It is registered automatically after OS notification permission and token issuance, then cleared on logout.

---

### 2.2 `scans` — Scan Records

This is the app's main output table. **One photo creates one row.** The current state and prediction result live together in this table. Prediction fields are stored exactly as returned by the AI service.

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | BIGSERIAL | PK | Scan ID |
| `user_id` | BIGINT | FK → users(id), NOT NULL | Owner |
| `target_stage` | SMALLINT | NOT NULL, CHECK (1-5) | Target stage for this scan, snapshotted from `preferred_stage` |
| `temp_celsius` | NUMERIC(4,1) | NULL | Room temperature entered by the user. NULL means the AI service used its default |
| `predicted_stage` | SMALLINT | NOT NULL, CHECK (1-5) | Predicted stage from the model. Used for Result and History badges |
| `confidence` | NUMERIC(5,4) | NULL | Confidence of the top predicted class, from 0 to 1 |
| `stage_probs` | JSONB | NULL | Probability distribution for the five stages, `[p1..p5]` |
| `days_to_target` | NUMERIC(4,1) | NULL | Remaining days until the target stage. Decimal and negative values are allowed; negative means overripe |
| `estimated_peak_date` | DATE | NULL | Estimated target eating date and notification scheduling basis |
| `model_version` | VARCHAR(50) | NOT NULL | Model version used for prediction, such as `resnet18_v3` |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Scan and prediction timestamp |

- Index: `user_id`, `created_at DESC` for newest-first History queries.

#### Design Notes

- **Append-only:** scan results are not modified after completion. "Re-scan" creates a new row rather than updating the old one. Mutable notification state lives in `notifications`.
- **Target snapshot:** `users.preferred_stage` is copied into `scans.target_stage` at scan time. Later user setting changes do not alter historical D-day interpretation.
- **Decimal `days_to_target`:** the AI service may return values such as `4.5`. UI labels such as `D-N` are derived from this value or from `estimated_peak_date`.
- **Optional `temp_celsius`:** until the frontend temperature input is added, the value remains NULL. Once added, Spring stores the original input and lets the AI service interpret or clamp it.

---

### 2.3 `images` — Captured Images

Each scan has one image. The original and cropped images are stored in **GCS**, and the database stores only paths.

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | BIGSERIAL | PK | Image ID |
| `scan_id` | BIGINT | FK → scans(id), NOT NULL, UNIQUE | Parent scan, one-to-one |
| `image_url` | VARCHAR(500) | NOT NULL | Original image path: `gs://d-avocado-images/raw/{user_id}/{scan_id}.jpg` |
| `cropped_url` | VARCHAR(500) | NULL | Cropped and normalized image path: `gs://d-avocado-images/cropped/{user_id}/{scan_id}.jpg` |
| `source` | VARCHAR(10) | NOT NULL, CHECK (`camera`/`gallery`) | Capture source |
| `width` | INT | NULL | Original width in pixels |
| `height` | INT | NULL | Original height in pixels |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Upload timestamp |

- `scan_id` is unique because each scan has a single image.
- The bucket is private, and clients receive TTL-signed URLs.
- `cropped_url` is produced by the AI service. It starts as NULL after original upload, then is filled after inference succeeds. It is useful for debugging, re-evaluation, and UI messaging such as "this region was analyzed."

---

### 2.4 `notifications` — Notifications

Each scan has at most one notification, scheduled once before the target date according to `advance_notice_days`.

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | BIGSERIAL | PK | Notification ID |
| `user_id` | BIGINT | FK → users(id), NOT NULL | Recipient |
| `scan_id` | BIGINT | FK → scans(id), NOT NULL | Target scan |
| `scheduled_at` | TIMESTAMPTZ | NOT NULL | Send time: `estimated_peak_date - advance_notice_days` |
| `sent_at` | TIMESTAMPTZ | NULL | Actual send timestamp |
| `status` | VARCHAR(15) | NOT NULL, DEFAULT 'scheduled', CHECK (`scheduled`/`sent`) | Notification status |
| `payload` | JSONB | NULL | Pre-rendered notification title and body |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Creation timestamp |

- Index: `status`, `scheduled_at` for scheduler queries.
- Regeneration rule: deleting a scan cascades to scheduled notifications. Although sent notifications are historical records in principle, the current policy cascades all notification rows when the scan is deleted.

#### History Mapping

- Badges such as **"Eat Now"**, **"Overripe"**, and **"D-2"** are derived from `scans.days_to_target`, not from notifications.
- The row bell icon means an active notification exists for the scan.
- Summary counts map to notification status: **Notified** = `sent`, **Pending** = `scheduled`.

---

## 3. Status Values

| Column | Allowed values |
| --- | --- |
| `images.source` | `camera`, `gallery` |
| `notifications.status` | `scheduled`, `sent` |

---

## 4. D-day Calculation Flow

1. The client uploads a photo.
2. Spring stores the original image in GCS and creates an `images` row with `cropped_url=NULL`.
3. Spring calls the AI service with the image, `target_stage` copied from `preferred_stage`, and `temp_celsius` when present.

```json
{ "target_stage": 3, "temp_celsius": 22.0 }
```

4. Inside the AI service: crop image, run ResNet-18 inference, output `predicted_stage` and `stage_probs`, then calculate temperature-based `days_to_target`. If temperature is missing, the service uses a default. If the target stage is reached or passed, the value may be zero or negative.
5. The AI service returns `{ predicted_stage, stage_probs, days_to_target, estimated_peak_date, model_version }` plus the cropped image.
6. Spring stores the cropped image in GCS and updates `images.cropped_url`.
7. Spring inserts the `scans` row with the snapshotted `target_stage` and AI results.
8. If `push_enabled=true`, Spring schedules one `notifications` row for `estimated_peak_date - advance_notice_days`; the scheduler sends via FCM/APNs using `push_token`, then records `status='sent'` and `sent_at`.

---

## 5. DDL (PostgreSQL Implementation Baseline)

```sql
-- users
CREATE TABLE users (
    id                  BIGSERIAL PRIMARY KEY,
    email               VARCHAR(255) NOT NULL UNIQUE,
    password_hash       VARCHAR(255) NOT NULL,
    nickname            VARCHAR(50),
    preferred_stage     SMALLINT NOT NULL DEFAULT 3 CHECK (preferred_stage BETWEEN 1 AND 5),
    push_enabled        BOOLEAN  NOT NULL DEFAULT TRUE,
    advance_notice_days SMALLINT NOT NULL DEFAULT 1 CHECK (advance_notice_days BETWEEN 0 AND 3),
    push_token          VARCHAR(255),
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- scans
CREATE TABLE scans (
    id                  BIGSERIAL PRIMARY KEY,
    user_id             BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    target_stage        SMALLINT NOT NULL CHECK (target_stage BETWEEN 1 AND 5),
    temp_celsius        NUMERIC(4,1),
    predicted_stage     SMALLINT NOT NULL CHECK (predicted_stage BETWEEN 1 AND 5),
    confidence          NUMERIC(5,4),
    stage_probs         JSONB,
    days_to_target      NUMERIC(4,1),
    estimated_peak_date DATE,
    model_version       VARCHAR(50) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_scans_user_created ON scans (user_id, created_at DESC);

-- images
CREATE TABLE images (
    id          BIGSERIAL PRIMARY KEY,
    scan_id     BIGINT NOT NULL UNIQUE REFERENCES scans(id) ON DELETE CASCADE,
    image_url   VARCHAR(500) NOT NULL,
    cropped_url VARCHAR(500),
    source      VARCHAR(10) NOT NULL CHECK (source IN ('camera','gallery')),
    width       INT,
    height      INT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- notifications
CREATE TABLE notifications (
    id           BIGSERIAL PRIMARY KEY,
    user_id      BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    scan_id      BIGINT NOT NULL REFERENCES scans(id) ON DELETE CASCADE,
    scheduled_at TIMESTAMPTZ NOT NULL,
    sent_at      TIMESTAMPTZ,
    status       VARCHAR(15) NOT NULL DEFAULT 'scheduled' CHECK (status IN ('scheduled','sent')),
    payload      JSONB,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_notifications_status_scheduled ON notifications (status, scheduled_at);
```

**Deletion policy:** deleting a user cascades to scans, then to images and notifications. Deleting a scan cascades to images and notifications. Migrations should be managed through Flyway, Liquibase, or an equivalent migration tool.

---

## 6. Change History

| Version | Summary |
| --- | --- |
| v0.2 | Seven-table design including stickers and beta coefficient tables |
| v0.3 | Removed stickers and `ripening_coefficients`; five-table design; introduced `is_active`; confirmed single-image capture |
| v0.4 | Added `room_temp_celsius`; removed `push_token` and used in-app polling; moved notification settings to `users` columns |
| **v1.0** | Flat scan model replacing `avocados` + `predictions` with `scans`; room-temperature only with `storage_condition` removed; global target stage 1-5; notifications via `push_enabled` + `advance_notice_days`; restored `push_token` for real FCM/APNs push; decimal `days_to_target`; finalized DDL |

---

## 7. Future Extensions (Out of Scope for v1)

- **Temperature refinement:** after the frontend temperature input is added, define handling for high-temperature ranges above 25°C and finalize how broadly `temp_celsius` is used.
- **Merge `images` into `scans`:** because capture is one-to-one, the image table could be collapsed into `scans`. It remains separate for now because the GCS original/cropped two-step flow is clearer.
- **Image retention policy:** decide whether original images are retained indefinitely or deleted after N days to reduce GCS cost.
- **Notification send-time policy:** extract send time, such as 09:00 local time, into a configurable setting.
