# 🥑 D-avocado

> AI-powered avocado ripeness tracking platform.

D-avocado is an end-to-end platform that predicts avocado ripeness from a single photo and estimates the remaining days until the user's preferred ripeness stage (D-day).

The project is built with a native iOS application, a Spring Boot backend, and an AI inference service deployed on Google Cloud Platform.

---

## 📖 Overview

D-avocado provides a complete workflow for avocado ripeness management.

Users can photograph an avocado, receive an AI-based ripeness prediction, estimate the remaining days until their preferred ripeness stage, and track their scans over time.

### Key Features

- 📷 Photo-based avocado ripeness prediction
- 🥑 Five-stage ripeness classification
- 📅 Personalized D-day estimation
- 📊 Scan history management
- ☁️ Cloud-native AI inference service

---

## 🏗️ System Architecture

<p align="center">
<img src="docs/images/architecture.png" alt="D-avocado system architecture" width="900">
</p>

---

## 📂 Project Structure

```text
d-avocado/
│
└── docs/
    ├── PRD.md
    ├── Architecture.md
    ├── API.md
    ├── Deployment.md
    ├── Database.md
    ├── AI.md
    └── images/
```

This repository holds the project documentation. The source code lives in three separate repositories:

| Repository | Description |
|------------|-------------|
| [davocado-frontend](https://github.com/QI-26SUMMER/davocado-frontend) | iOS app (Swift, SwiftUI) |
| [davocado-backend](https://github.com/QI-26SUMMER/davocado-backend) | Backend API (Spring Boot) |
| [d-avocado-ripeness-mlops](https://github.com/QI-26SUMMER/d-avocado-ripeness-mlops) | Model training, evaluation, and AI inference service (FastAPI) |

---

## ⚙️ Technology Stack

### Mobile

- Swift, SwiftUI (iOS 18+)
- Observation framework (`@Observable` app state)

### Backend

- Java 21
- Spring Boot
- PostgreSQL
- JWT Authentication

### AI / Machine Learning

- Python, FastAPI (inference service on Cloud Run)
- Google Vertex AI AutoML Vision (production classifier)
- PyTorch, ResNet-18 (in-house model, trained and evaluated with 5-fold cross-validation)
- InSPyReNet (background removal before classification)

### Cloud

- Google Cloud Platform
- Cloud Run
- Cloud SQL
- Cloud Storage
- Artifact Registry
- Cloud Build
- Vertex AI (AutoML endpoint, Custom Job training)

---

## 🚀 End-to-End Workflow

```text
Take Photo (iOS)
      │
      ▼
Upload Image → Backend API (Spring Boot)
      │
      ▼
AI Inference Service (FastAPI, Cloud Run)
  ├─ Background removal & crop (InSPyReNet)
  ├─ Ripeness classification (Vertex AI AutoML)
  └─ Temperature-adjusted D-day calculation
      │
      ▼
Backend: save scan · upload images to GCS · schedule notification
      │
      ▼
Return D-Day Result
```

---

## 📚 Documentation

```
docs/
├── PRD.md
├── Architecture.md
├── API.md
├── Deployment.md
├── Database.md
└── AI.md
```

| Document | Description |
|----------|-------------|
| **PRD.md** | Product Requirements Document |
| **Architecture.md** | System architecture |
| **API.md** | Backend API specification |
| **Deployment.md** | Google Cloud deployment guide |
| **Database.md** | Database schema and design |
| **AI.md** | AI model architecture and evaluation summary |

---

## ✨ Features

- User authentication
- Personalized target ripeness settings
- AI-powered ripeness prediction
- Scan history management
- Image storage with Google Cloud Storage
- Cloud-native deployment
- Modular service architecture

---

## 🔮 Roadmap

- [x] iOS Application
- [x] Backend API
- [x] AI Inference Service
- [x] Google Cloud Deployment
- [ ] Push notification scheduling
- [ ] Model monitoring
- [ ] CI/CD automation
- [ ] Continuous model retraining
- [ ] Explainable AI visualization

---

## 👥 Team

| Role | Repository |
|------|------------|
| Mobile | `davocado-frontend` |
| Backend | `davocado-backend` |
| AI / MLOps | `d-avocado-ripeness-mlops` |

---
