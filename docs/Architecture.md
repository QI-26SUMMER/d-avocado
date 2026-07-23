# D-avocado Architecture

> System architecture for the avocado ripeness stage classification and D-day prediction service.

---

## 1. High-level Architecture

<p align="center">
<img src="images/architecture.png" alt="D-avocado system architecture" width="900">
</p>

D-avocado is built as a mobile-first system with three main runtime components:

| Component | Runtime | Responsibility |
| --- | --- | --- |
| iOS App | Xcode / Swift / SwiftUI | Captures avocado photos, shows scan results, manages settings, history, and notifications |
| Backend API | Spring Boot on Cloud Run | Handles authentication, user settings, scan records, image storage, notification scheduling, and calls to the AI service |
| AI Service | Cloud Run inference service | Crops/preprocesses images, predicts ripeness stage, calculates D-day, and returns model metadata |

Google Cloud Platform provides the managed infrastructure: Cloud Run for compute, Cloud Storage for images, Cloud SQL for PostgreSQL data, Artifact Registry for container images, and Cloud Build for CI/CD.

---

## 2. Runtime Components

### 2.1 iOS App

The iOS app is the only client in the initial build.

- Built with SwiftUI and MVVM.
- Sends authenticated API requests with a JWT access token.
- Uploads one avocado image per scan.
- Displays the result card: predicted stage, D-day text, and analyzed image.
- Provides Settings for `preferred_stage`, `push_enabled`, and `advance_notice_days`.
- Shows scan History and an in-app notification inbox.
- Registers the FCM/APNs push token automatically after notification permission is granted.

### 2.2 Spring Boot Backend API

The backend is the central orchestration layer.

- Exposes REST API endpoints under `/v1`.
- Owns users, settings, scan history, image metadata, and notifications.
- Stores original and cropped image paths in the database.
- Uploads image objects to Cloud Storage.
- Reads `users.preferred_stage` at scan time and snapshots it into `scans.target_stage`.
- Calls the AI Cloud Run service using service-to-service authentication.
- Schedules one notification per scan when push notifications are enabled.

### 2.3 AI Inference Service

The AI service owns all model-related work.

- Receives an image plus `{ target_stage, temp_celsius? }`.
- Performs preprocessing and avocado/background segmentation.
- Runs ResNet-18 based ripeness classification.
- Returns `predicted_stage`, `stage_probs`, `days_to_target`, `estimated_peak_date`, and `model_version`.
- Produces a cropped image for storage and optional UI/debug use.

The backend stores the AI response as scan output. It does not recalculate model predictions or D-day values.

---

## 3. Data Flow

### 3.1 Scan Flow

```text
iOS App
  │
  │ POST /scans
  │ image + source + optional temp_celsius
  ▼
Spring Boot API
  │
  ├─ Store original image in Cloud Storage
  │
  ├─ Read users.preferred_stage
  │
  ├─ Call AI Service on Cloud Run
  │    image + target_stage + optional temp_celsius
  │
  ├─ Store cropped image in Cloud Storage
  │
  ├─ Insert scans + images rows in Cloud SQL
  │
  ├─ Schedule notification if enabled
  │
  ▼
iOS App receives result card response
```

### 3.2 History Flow

```text
iOS App
  │
  ├─ GET /scans
  ├─ GET /scans/stats
  └─ GET /scans/{id}
       │
       ▼
Spring Boot API
  │
  ├─ Read scans, images, notifications from Cloud SQL
  ├─ Generate signed URLs for private GCS image objects
  └─ Return display-ready scan records
```

### 3.3 Notification Flow

```text
Scan created
  │
  ├─ estimated_peak_date returned by AI service
  ├─ scheduled_at = estimated_peak_date - advance_notice_days
  └─ notifications row inserted
       │
       ▼
Scheduler
  │
  ├─ Finds due scheduled notifications
  ├─ Sends push through FCM/APNs using users.push_token
  └─ Marks notification as sent
```

---

## 4. Storage Design

### 4.1 Cloud SQL

Cloud SQL for PostgreSQL stores normalized application data:

- `users`
- `scans`
- `images`
- `notifications`

See [Database.md](Database.md) for table definitions and DDL.

### 4.2 Cloud Storage

Cloud Storage stores private image objects.

```text
gs://d-avocado-images/
├── raw/{user_id}/{scan_id}.jpg
└── cropped/{user_id}/{scan_id}.jpg
```

- `raw/` stores the original uploaded image.
- `cropped/` stores the AI-produced crop.
- The bucket remains private.
- The API returns temporary signed URLs to the iOS app.

---

## 5. Security Boundaries

- The iOS app authenticates to the backend using JWT.
- The iOS app does not call the AI service directly.
- The backend calls the AI service with a GCP service account identity token.
- Cloud Storage objects are private and exposed only through signed URLs.
- User-owned resources must be checked by `user_id` on every read, update, and delete.
- Passwords are stored only as secure hashes.

---

## 6. Main Design Decisions

| Decision | Reason |
| --- | --- |
| One photo equals one scan | Simplifies the first build and removes per-fruit lifecycle complexity |
| Global `preferred_stage` | Users set their target once, then scan without repeated setup |
| Snapshot `target_stage` per scan | Historical D-day results stay stable even if the user changes settings later |
| Room-temperature only | Refrigerated storage introduces chilling injury and invalidates the current D-day model |
| AI service owns `days_to_target` | Keeps model and temperature logic in one place |
| Private GCS bucket + signed URLs | Protects user images while still allowing mobile display |

---

## 7. Open Architecture Questions

- Whether `temp_celsius` becomes required after the iOS temperature input is added.
- Whether push delivery uses FCM for iOS or direct APNs.
- Whether image upload should remain Spring-mediated or move to direct signed URL upload.
- Whether the notification scheduler runs inside the backend service, Cloud Scheduler, or a separate worker.
