# PRD — Avocado Ripeness Detection & D-day Prediction Service (D-avocado)

## Problem & Goal

**Problem.** Avocados are a climacteric fruit that continues to ripen after harvest, and
it's hard to tell the internal state from the outside. Consumers typically rely on
squeezing the fruit by hand, but this judgment varies widely by experience and isn't
reproducible. As a result, people either eat avocados too early when they're still hard,
or miss the window entirely and end up throwing away overripe fruit. There's no
consistent basis for knowing when an avocado is actually ready to eat.

**Goal.** From a single photo, determine the current ripening stage (1–5) and tell the
user **how many days remain until their target eating stage (D-day)**.

**Success Criteria**

- Ripeness stage classification (5 classes): target accuracy — *pending team discussion*
(Note: QWK was initially chosen as the primary metric to penalize ordinal
misclassification proportionally to distance, but was dropped in favor of accuracy
because QWK scores came out unrealistically high and were not discriminating enough.)
- Reducing confusion between the Breaking and Overripe stages specifically, tracked as a
separate goal (confirmed in the literature as the hardest pair to distinguish)
- A working iOS app: photo capture → result (stage + D-day) displayed
- Notifications fire and display correctly when the target stage is approaching

---

## Target User (Persona)

**Minjun, early 30s, lives alone, grocery shops 1–2 times a week**

- Buys 2–3 avocados at once at the store and keeps them in the fridge or at room
temperature
- Wants to make avocado toast on the weekend, but bought the avocados on Tuesday and
doesn't know if they'll be ready by Saturday
- Has heard "squeeze it gently" as advice but doesn't know what "just right" actually
feels like
- Ends up with one avocado that's still hard and inedible, and another that's turned
black and gone bad

Key moment: **not at the point of purchase, but a few days later when the fridge is
opened again.**

---

## Value — Why Ours

1. **We answer "when," not just "what."**
Instead of just saying "this is stage 3," we calculate and report the number of days
remaining until the user's target stage (`preferred_stage`, 1–5). The target is set once
per user (not per fruit) and applies automatically to every subsequent scan.
2. **We account for storage temperature.**
The same ripeness stage can mean different remaining time depending on storage
temperature. We use the Q10 coefficient to interpolate across temperature ranges and
adjust the β coefficient accordingly. The valid range is approximately 10–25°C;
anything outside that range is explicitly out of scope for now (see Out of Scope below).
3. **We notify before the window closes.**
As the target stage approaches, we send a notification in advance by the number of days
the user configured (`advance_notice_days`). Past scans are also viewable in the History
screen.
4. **We're upfront about our limitations.**
The training data was photographed under white-background studio conditions, while real
users will photograph avocados against arbitrary backgrounds. The model predicts
*visual* ripening stage only — it does not measure actual taste. Refrigerated storage
is excluded from the model entirely, since chilling injury (below 10°C) makes the D-day
prediction meaningless. These limitations are disclosed in the result screen and in
documentation.
5. Above ~20°C, further temperature increases have little additional effect on ripening
speed — per *"Hass" avocado quality as influenced by temperature and ethylene prior to
and during final ripening* (Arpaia et al.).

---

## Must-have Features

| # | Feature | Description |
| --- | --- | --- |
| F1 | **Ripeness stage classification** | Photo upload → 5-class prediction + per-class probabilities (`stage_probs`) |
| F2 | **Target stage setting** | Global per-user setting (`preferred_stage`, 1–5). Changed in Settings; snapshotted onto each scan |
| F3 | **D-day prediction** | Stage + temperature (optional) → days remaining until target stage |
| F4 | **Result card** | Predicted stage and days remaining (`days_to_target`) |
| F5 | **Notifications** | Push notification when target is approaching + in-app notification inbox; toggle per scan |
| F6 | **History** | List of past scans (Total / Notified / Pending) |

Priority: F1 > F3 > F2 > F4 > F5 > F6

---

## User Stories

- I want to photograph an avocado and know **if it'll be ready to eat tomorrow.**
- I have guests coming this weekend — I want to know **if the avocado I just bought will
be ready by then.**
- I want to set my target stage once and **just scan from then on**, without re-entering
it every time.
- I want to **get a heads-up notification** before the target date arrives.
- I want to see **what happened to the avocado I scanned last week** in my history.

---

## Out of Scope

**Not doing in the initial build**

- Per-fruit registration/tracking — simplified so that one photo = one scan record
- Refrigerated-storage ripening prediction (≤10°C) — excluded entirely since chilling
injury makes prediction meaningless; room temperature only for now
- β interpolation above 25°C — outside the valid Q10 interpolation range (e.g. un-air-
conditioned indoor environments in hot climates are a future consideration)
- Expansion to fruits other than avocado

**Deliberately not doing**

- Predicting actual taste/sweetness — the dataset has no sensory or sugar-content labels.
What we predict is **visual ripening stage**, and we don't market that as "taste."

---

## Known Risks

| Risk | Mitigation |
| --- | --- |
| Training images are shot on white backgrounds under studio lighting → performance may degrade on real photos with arbitrary backgrounds (domain gap) | Inference-time preprocessing: background segmentation via **rembg**. (SAM3 was considered but a lighter-weight model was used instead due to GPU constraints.) |
| Breaking and Overripe stages are visually easy to confuse | QWK is tracked as a secondary metric, giving lower penalty to misclassifications that are close in ordinal distance |
| GPU training environment is currently CPU-only due to GCP free-trial quota limits | Plan to move to a GPU-backed Vertex AI Custom Job after upgrading; training-time delays are factored into the schedule |

---

## References

- Xavier et al. (2024), *Foods* — Hass Avocado Ripening Photographic Dataset, α ripening coefficients
- Perez et al. (2004) — basis for Q10 value
- Arpaia et al. (2018) — basis for the temperature–ripening-speed plateau finding
