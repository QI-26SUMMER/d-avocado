# D-avocado Deployment Guide

> Deployment guide for the D-avocado iOS, Spring Boot backend, AI inference service, and Google Cloud infrastructure.

---

## 1. Target Environment

| Layer | Target |
| --- | --- |
| Mobile | iOS app distributed through Xcode/TestFlight/App Store |
| Backend API | Spring Boot container on Cloud Run |
| AI inference | FastAPI/Python container on Cloud Run |
| Database | Cloud SQL for PostgreSQL |
| Image storage | Cloud Storage private bucket |
| Container registry | Artifact Registry |
| CI/CD | Cloud Build |
| Push delivery | FCM or APNs, final provider pending |

---

## 2. Google Cloud Resources

### 2.1 Required Services

Enable these GCP APIs:

```text
run.googleapis.com
sqladmin.googleapis.com
storage.googleapis.com
artifactregistry.googleapis.com
cloudbuild.googleapis.com
secretmanager.googleapis.com
iamcredentials.googleapis.com
```

Add `firebase.googleapis.com` if FCM is selected for push notifications.

### 2.2 Resource Names

Suggested baseline names:

| Resource | Name |
| --- | --- |
| Project ID | `d-avocado` or team-owned equivalent |
| Region | `asia-northeast3` or selected primary region |
| Artifact Registry repo | `d-avocado-containers` |
| Backend Cloud Run service | `davocado-backend` |
| AI Cloud Run service | `davocado-ai-service` |
| Cloud SQL instance | `davocado-postgres` |
| Database | `davocado` |
| GCS bucket | `d-avocado-images` |

---

## 3. Secrets and Environment Variables

Store sensitive values in Secret Manager and inject them into Cloud Run.

### 3.1 Backend API

| Variable | Description |
| --- | --- |
| `SPRING_PROFILES_ACTIVE` | Runtime profile, such as `prod` |
| `DATABASE_URL` | JDBC URL for Cloud SQL PostgreSQL |
| `DATABASE_USERNAME` | Database user |
| `DATABASE_PASSWORD` | Database password from Secret Manager |
| `JWT_SECRET` | JWT signing secret from Secret Manager |
| `GCS_BUCKET_NAME` | Image bucket name, such as `d-avocado-images` |
| `AI_SERVICE_URL` | Internal HTTPS URL of the AI Cloud Run service |
| `GOOGLE_CLOUD_PROJECT` | GCP project ID |
| `PUSH_PROVIDER` | `fcm` or `apns`, pending final decision |
| `FCM_CREDENTIALS` | FCM credentials or Secret Manager reference, if FCM is used |

### 3.2 AI Service

| Variable | Description |
| --- | --- |
| `MODEL_VERSION` | Model version string, such as `resnet18_v3` |
| `MODEL_PATH` | Local container path or GCS path for model weights |
| `DEFAULT_TEMP_CELSIUS` | Default room temperature when the client does not provide one |
| `VALID_TEMP_MIN_CELSIUS` | Lower valid model range, currently around `10` |
| `VALID_TEMP_MAX_CELSIUS` | Upper valid model range, currently around `25` |

---

## 4. Database Deployment

1. Create a Cloud SQL PostgreSQL instance.
2. Create the `davocado` database.
3. Create a least-privilege application user.
4. Configure Cloud Run database connectivity through the Cloud SQL connector or private networking.
5. Run schema migrations using Flyway, Liquibase, or the backend migration tool.

The v1.0 schema is defined in [Database.md](Database.md).

---

## 5. Storage Deployment

1. Create the private Cloud Storage bucket.
2. Disable public access.
3. Grant the backend Cloud Run service account object read/write access.
4. Use these object prefixes:

```text
raw/{user_id}/{scan_id}.jpg
cropped/{user_id}/{scan_id}.jpg
```

5. Return only short-lived signed URLs to clients.

---

## 6. Backend API Deployment

### 6.1 Build Container

Build the Spring Boot application into a container image and push it to Artifact Registry.

```text
artifact-registry/d-avocado-containers/davocado-backend:{version}
```

### 6.2 Deploy to Cloud Run

Deploy with:

- HTTP ingress enabled for the iOS app.
- Cloud SQL connection configured.
- Secret Manager values mounted or injected as environment variables.
- Service account with access to Cloud SQL, Cloud Storage, Secret Manager, and AI service invocation.

### 6.3 Backend Health Checks

Recommended endpoints:

```text
GET /actuator/health
GET /actuator/info
```

The health endpoint should verify application readiness without performing expensive AI calls.

---

## 7. AI Service Deployment

### 7.1 Build Container

Build the Python/FastAPI inference service with:

- Runtime dependencies.
- Model weights or model download logic.
- Preprocessing dependencies.
- Startup validation that confirms the model can be loaded.

Push the image to Artifact Registry:

```text
artifact-registry/d-avocado-containers/davocado-ai-service:{version}
```

### 7.2 Deploy to Cloud Run

Deploy with:

- Authentication required.
- Invocation allowed only from the backend service account.
- CPU and memory sized for image preprocessing and ResNet-18 inference.
- Timeout long enough for cold starts and first inference.
- Concurrency tuned conservatively if model memory usage is high.

### 7.3 AI Health Checks

Recommended endpoints:

```text
GET /health
GET /model/version
```

`/health` should confirm the service is alive. `/model/version` should return the active `MODEL_VERSION`.

---

## 8. Notification Deployment

The notification system needs two pieces:

1. A scheduler that finds due `scheduled` notifications.
2. A push provider integration that sends through FCM or APNs.

Possible scheduler options:

| Option | Description |
| --- | --- |
| Cloud Scheduler + backend endpoint | Simple scheduled HTTP call into the backend |
| Backend internal scheduler | Easy for the first build, but less isolated |
| Separate worker service | Cleaner long-term separation for retries and scaling |

For v1.0, Cloud Scheduler calling a protected backend endpoint is the recommended baseline.

---

## 9. CI/CD Flow

Recommended Cloud Build pipeline:

```text
Push to main
  │
  ├─ Run tests
  ├─ Build backend container
  ├─ Build AI service container
  ├─ Push images to Artifact Registry
  ├─ Run database migrations
  ├─ Deploy backend to Cloud Run
  └─ Deploy AI service to Cloud Run
```

Use immutable image tags such as Git commit SHA values. Keep `latest` only as a convenience tag, not as the deployment source of truth.

---

## 10. Deployment Order

For a clean environment, deploy in this order:

1. Enable GCP APIs.
2. Create service accounts and IAM roles.
3. Create Artifact Registry repository.
4. Create Cloud SQL instance and database.
5. Create Cloud Storage bucket.
6. Add secrets to Secret Manager.
7. Build and deploy the AI service.
8. Build and deploy the backend API.
9. Run database migrations.
10. Configure scheduler and push provider.
11. Configure the iOS app with the production API base URL.
12. Run end-to-end smoke tests.

---

## 11. Smoke Test Checklist

- Sign up and log in.
- Fetch `GET /users/me`.
- Register a push token with `PUT /users/me/push-token`.
- Upload an image with `POST /scans`.
- Confirm the original and cropped images exist in Cloud Storage.
- Confirm scan, image, and notification rows exist in Cloud SQL.
- Load History with `GET /scans`.
- Load stats with `GET /scans/stats`.
- Open a scan detail with `GET /scans/{id}`.
- Confirm signed image URLs render in the iOS app.
- Trigger a due notification in a test environment and verify it becomes `sent`.

---

## 12. Rollback Notes

- Cloud Run revisions allow rollback to the previous backend or AI service image.
- Database migrations should be backward-compatible whenever possible.
- Avoid deploying backend code that requires a migration before the migration has completed.
- Keep model versions explicit so scan records can be traced to the exact inference model used.
