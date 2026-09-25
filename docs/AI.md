# D-avocado AI Model Summary

> Architecture and evaluation summary for avocado ripeness stage classification models.

---

## 1. Evaluation Context

D-avocado has evaluated four model tracks for 5-stage avocado ripeness classification:

| Model Track | Training Setup | Evaluation Setup |
| --- | --- | --- |
| ResNet-18 Custom | PyTorch, ImageNet-pretrained | 5-fold cross-validation |
| AutoML Vision Raw | Vertex AI AutoML Vision | Single 70/15/15 train/validation/test split |
| AutoML Vision Balanced | Vertex AI AutoML Vision | Single 70/15/15 train/validation/test split |
| GMM Color Baseline | Color statistics + per-stage GaussianMixture (classical ML) | Single 70/15/15 split (same split as ResNet's fixed-split run) |

The models are not evaluated under the same protocol or data yet. ResNet-18 uses 5-fold cross-validation on a curated subset (392 of 478 samples with a complete stage 1→5 trajectory, 13,192 images), while the AutoML models use a single train/test split with different dataset sizes. Direct comparison should therefore be treated as directional rather than final.

> **Deployment status.** The live AI service currently routes classification to the Vertex AI AutoML Vision endpoint (`MODEL_BACKEND=automl`), because ResNet-18 collapsed to stage 1 on real phone photos (domain gap between light-box training images and phone photos). The production endpoint serves the **AutoML Vision Balanced** model. The ResNet-18 path is kept and can be restored with `MODEL_BACKEND=resnet`.

---

## 2. Shared Metrics

### AutoML-focused Metrics

| Metric | Meaning |
| --- | --- |
| Precision | Share of predicted positives that are correct |
| Recall | Share of actual positives that are detected |
| Average Precision (AP) | Area under the precision-recall curve |

### ResNet-focused Metrics

| Metric | Meaning |
| --- | --- |
| Exact Accuracy | Share of images where the predicted stage exactly matches the label |
| Within-1-Stage Accuracy | Share of predictions within one ordinal ripeness stage of the label |
| QWK | Quadratic Weighted Kappa; penalizes ordinal distance between true and predicted stage |
| Macro-F1 | Average F1 score across the five stages |
| Stage MAE | Mean absolute error measured in ripeness-stage distance |

---

## 3. Model Comparison Summary

| Model | Dataset | Evaluation | Main Result | Notes |
| --- | --- | --- | --- | --- |
| ResNet-18 Custom | 13,192 images | 5-fold CV | 79.4% exact accuracy, 99.5% within-1-stage accuracy, QWK 0.946 | Strong ordinal behavior; ResNet-18 deployment candidate is fold4/best.pt |
| AutoML Vision Raw | 14,570 images | 70/15/15 split | AP 0.904, precision 82.8%, recall 78.4% | Managed baseline with natural class distribution |
| AutoML Vision Balanced | 20,000 images | 70/15/15 split | AP 0.908, precision 84.3%, recall 79.9% | Balanced to 4,000 images per class through data augmentation; best AutoML aggregate result; production model |
| GMM Color Baseline | 13,192 images | 70/15/15 split | 65.0% exact accuracy, 96.4% within-1-stage, QWK 0.889 | Non-deep-learning color floor (12-D "rich" features); rgb-only variant 55.6% |

---

## 4. ResNet-18 Custom Model

### 4.1 Architecture

| Item | Value |
| --- | --- |
| Framework | PyTorch |
| Backbone | ResNet-18 |
| Pretraining | ImageNet-pretrained |
| Input size | 224 x 224 |
| Training | GCP Vertex AI Custom Job |
| Serving | Cloud Run AI service (`MODEL_BACKEND=resnet`; not the current production backend) |

### 4.2 Training Configuration

| Hyperparameter | Value |
| --- | --- |
| Optimizer | SGD |
| Momentum | 0.9 |
| Weight decay | 0.0001 |
| Seed | 42 |
| Epochs | 30 |
| Batch size | 128 |
| Learning rate | 0.01 (StepLR, ×0.1 every 10 epochs) |
| Early stopping patience | 150 validation checks (validation every 10 iterations) |

### 4.3 Validation Protocol

ResNet-18 uses 5-fold cross-validation. Folds are split on `(Storage Group, Sample)`, never at image level, so the same avocado never appears in both training and evaluation.

For round `r`:

- Held-out fold: `fold r`, used as the validation fold for checkpoint selection (best checkpoint)
- Training folds: the remaining four folds

The reported metrics are the held-out fold's metrics at the selected checkpoint. Because the same fold is used for checkpoint selection, there is no separate test fold, and the CV numbers may be slightly optimistic. The independent single-split run below reports a held-out test result.

### 4.4 Training-only Data Processing

Augmentation is applied only to the training fold:

- Reflection
- Rotation within ±10°
- Rescale from 0.95 to 1.05
- Translation within ±10 px
- Geometric augmentation only
- No color changes

Class balancing through oversampling is also applied only to the training fold.

### 4.5 Results

Results are reported as 5-fold mean ± standard deviation of the held-out (validation) fold metrics over 13,192 images.

| Metric | Result |
| --- | --- |
| Exact Accuracy | 0.794 ± 0.009 (79.4%) |
| Within-1-Stage Accuracy | 0.995 ± 0.003 (99.5%) |
| QWK | 0.946 ± 0.004 |
| Macro-F1 | 0.790 ± 0.010 |
| Stage MAE | 0.212 ± 0.010 |

> **Single-split reproduction (held-out test).** An independent fixed-split retrain (`P1_general_resnet18`, 15 epochs, lr 0.01 → 0.001 at epoch 10) reached **test exact accuracy 0.805, within-1 0.996, QWK 0.950, MAE 0.199** on the held-out test set (n = 1,962), with per-class recall `[0.94, 0.81, 0.75, 0.72, 0.79]`.

### 4.6 Fold Selection

- Fold 2 had the highest exact accuracy: `0.8071`.
- Fold 4 had the highest QWK: `0.9514`.
- `fold4/best.pt` was selected as the ResNet-18 deployment candidate.

### 4.7 Confusion Matrix Diagonal

Correct-prediction rate by stage:

| Stage | Correct-prediction Rate |
| --- | --- |
| Stage 1 | 90.2% |
| Stage 2 | 76.0% |
| Stage 3 | 76.6% |
| Stage 4 | 71.1% |
| Stage 5 | 81.7% |

Stage 4 is the weakest class by diagonal accuracy, while Stage 1 is the strongest.

---

## 5. AutoML Vision Raw Model

### 5.1 Architecture

| Item | Value |
| --- | --- |
| Framework | GCP Vertex AI AutoML Vision |
| Training mode | Managed training |
| Class distribution | Natural distribution |

### 5.2 Dataset

| Split | Images |
| --- | ---: |
| Train | 10,200 |
| Validation | 2,185 |
| Test | 2,185 |
| Total | 14,570 |

Split ratio: 70/15/15.

### 5.3 Results

| Metric | Result |
| --- | --- |
| Average Precision (AP) | 0.904 |
| Precision | 82.8% |
| Recall | 78.4% |

### 5.4 Per-class AP

| Stage | AP |
| --- | ---: |
| Stage 1 | 0.977 |
| Stage 2 | 0.863 |
| Stage 3 | 0.798 |
| Stage 4 | 0.865 |
| Stage 5 | 0.942 |

Stage 3 is the weakest class by AP in the raw AutoML model.

---

## 6. AutoML Vision Balanced Model

### 6.1 Architecture

| Item | Value |
| --- | --- |
| Framework | GCP Vertex AI AutoML Vision |
| Training mode | Managed training |
| Class distribution | Balanced to 4,000 images per class through data augmentation |

### 6.2 Dataset

| Split | Images |
| --- | ---: |
| Train | 14,000 |
| Validation | 3,000 |
| Test | 3,000 |
| Total | 20,000 |

Split ratio: 70/15/15.

### 6.3 Results

| Metric | Result |
| --- | --- |
| Average Precision (AP) | 0.908 |
| Precision | 84.3% |
| Recall | 79.9% |

### 6.4 Per-class AP

| Stage | AP |
| --- | ---: |
| Stage 1 | 0.974 |
| Stage 2 | 0.896 |
| Stage 3 | 0.823 |
| Stage 4 | 0.848 |
| Stage 5 | 0.949 |

Balancing improves aggregate AP, precision, recall, and Stage 3 AP compared with the raw AutoML model, while Stage 4 AP decreases slightly.

---

## 7. Key Takeaways

- ResNet-18 provides stronger control over training, validation, checkpoint selection, and future MLOps.
- AutoML Balanced performs slightly better than AutoML Raw on aggregate AP, precision, and recall.
- ResNet-18 shows very high within-1-stage accuracy, which is important because ripeness stages are ordinal.
- Exact accuracy remains the main operational target because QWK can look overly optimistic for near-miss predictions.
- Stage 4 remains an important risk area in the ResNet confusion matrix.
- Stage 3 and Stage 4 are the most important classes to monitor closely across future evaluation runs.

---

## 8. Open Evaluation Gaps

- Align model evaluation protocols so ResNet-18 and AutoML are compared under the same split or cross-validation setup.
- Report per-class precision, recall, and F1 for ResNet-18.
- Report full confusion matrices for all model tracks.
- Evaluate real-world phone images, not only studio-style dataset images.
- Track performance after inference-time background segmentation.
- Decide whether final model selection should prioritize exact accuracy, Macro-F1, QWK, or deployment constraints.
