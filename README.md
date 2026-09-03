# ഒന്നാണോ? — Same aano? AI parayatte.

**AI Sibling Fairness Judge** — point a camera at whatever two siblings are fighting over, and let computer vision settle it.

> "Sibling fight undo? AI decide cheyyatte."

---

## 1. Problem statement

Two siblings are sharing something — a chocolate bar, a pile of toys, a stack of cards — and someone always thinks they got less. Normally a parent eyeballs it and picks a side, which never fully convinces anyone. This project replaces that judgment call with a repeatable, camera-based measurement: point, scan, and get a specific, explainable division with a numeric fairness score.

## 2. Why sibling fairness?

It's a small, well-scoped problem that still touches real computer-vision fundamentals — segmentation, contour geometry, optimization, area estimation — and it's fun to demo: everyone in the room has an intuition for whether a 52/48 split "feels fair," so the AI's answer is instantly checkable.

## 3. Features

- **Mode A — Split One Object**: detects a single object (chocolate, cake, fruit, cookie…) and computes a straight cutting line that divides its visible area as close to 50:50 as possible, at any orientation.
- **Mode B — Divide Multiple Objects**: detects N separate objects, picks a division strategy (equal-count vs. equal-value) based on how similar the objects are, and assigns each object to Sibling A or Sibling B to balance total value.
- **Fairness score & classification**: one consistent 0–100 score with four tiers, from "Perfectly Fair 🏆" to "Sibling Fight Warning 🚨😂".
- **Fun Malayalam/Manglish verdicts**, randomized from configurable templates.
- **Verification pass**: re-scan after the physical cut / distribution and get a second, "did it actually work" score.
- **Friendly error handling**: too dark, blurry, no object, too many objects, camera denied, backend unreachable — all surfaced as plain messages, never a stack trace.
- **Honest limitation notice**: every result reminds the user this is a *visual-area* estimate, not a weight measurement.

## 4. Demo workflow

```
Open app → "ഒന്നാണോ?" → Choose mode → Camera opens → Place object(s) →
Scan → AI detects & measures → Fairest division shown → Cut / distribute →
Verify (scan again) → Final fairness score → Malayalam verdict →
"Fight Cancelled. 😂⚖️"
```

## 5. Architecture

```
React (Vite)  ──REST/JSON/multipart──►  FastAPI
   │                                        │
   camera capture (getUserMedia)            OpenCV pipeline:
   renders results & verdict                detection → segmentation →
                                             measurement → cutting /
                                             fair-split → scoring
```

No images are written to disk; each request is processed in memory and only an annotated preview (base64 JPEG) is sent back to the browser.

## 6. Technology stack

| Component        | Technology            |
|-------------------|------------------------|
| Frontend          | React + Vite (JavaScript) |
| Computer Vision    | Python + OpenCV        |
| Backend            | Python + FastAPI       |
| Communication      | REST API + JSON (multipart image upload) |
| Styling            | Plain CSS              |

## 7. Computer vision methodology

For every scan the backend runs:

1. **Resize** to a working resolution (long edge ≤ 900px) for consistent speed.
2. **Lighting/contrast/blur checks** on the grayscale frame — reject with a friendly error before wasting time on a bad shot.
3. **Gaussian blur** to suppress sensor noise.
4. **Otsu thresholding** (with an adaptive-threshold fallback for uneven lighting) to get a binary foreground mask.
5. **Morphological close + open** to fill small gaps and remove speckle noise.
6. **Contour detection** (`cv2.findContours`) to find individual object blobs, filtered by a minimum-area-ratio to drop noise.

This is deliberately generic — nothing in `object_detection.py` or `segmentation.py` assumes chocolate, or any other specific object class. It just finds visually-distinguishable blobs on a reasonably plain background.

## 8. Equal-area algorithm (Mode A)

Naively cutting through the bounding-box center (`width / 2`) fails for irregular or rotated objects. Instead:

1. Take the object's filled binary mask.
2. For a set of candidate orientations (every few degrees, 0–180°), project every foreground pixel onto that orientation's normal vector.
3. Sort the projections and pick the split index closest to exactly half the pixel count — this *directly* finds the optimal offset for that orientation (equivalent to trying every offset and keeping the best, but done in one sort instead of a nested loop).
4. Compare candidates across all orientations; keep the one with the smallest area difference, breaking near-ties by preferring the physically shortest cut.
5. Convert the winning line back into two on-image endpoints (clipped against the object's bounding box) for drawing.

This works uniformly for rectangles, rotated rectangles, squares, circles, and irregular polygons — verified in `tests/test_algorithms.py`.

## 9. Multi-object optimization (Mode B)

1. Detect all objects and compute each one's contour area.
2. Compute the coefficient of variation of the areas.
   - **Low variation** (objects are similar-sized) → **equal-count strategy**: sort by size, alternately assign the next-largest item to whichever pile currently has less total value.
   - **High variation** → **equal-value strategy**: solve a balanced 2-partition problem using area as value.
     - N ≤ 22 → **exact dynamic-programming subset-sum** (values scaled to integers to bound the table size).
     - N > 22 → **greedy largest-first assignment + local 1-item swap refinement**, to stay fast without brute-forcing a combinatorial explosion.
3. A configurable `max_objects` cap (default 20, hard limit 60) prevents runaway detection or DP cost on a cluttered frame.

## 10. Fairness scoring

```
balance_error   = |value1 - value2| / (value1 + value2)
fairness_score  = (1 - balance_error) × 100      (clamped to 0–100)
```

The same formula is used for single-object area splits and multi-object group values, so scores are directly comparable across modes.

**Classification tiers** (thresholds configurable in `algorithms/scoring.py`):

| Score range | Label |
|---|---|
| 98–100 | PERFECTLY FAIR 🏆 |
| 95–97.99 | ALMOST PERFECT 😎 |
| 90–94.99 | PRETTY FAIR 👍 |
| < 90 | SIBLING FIGHT WARNING 🚨😂 |

## 11. API endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | Liveness check |
| POST | `/analyze/single` | Mode A analysis — returns cut line, percentages, fairness score, verdict, annotated image |
| POST | `/analyze/multiple` | Mode B analysis — returns group assignments, values, fairness score, verdict, annotated image |
| POST | `/verify/single` | Re-scan the two cut pieces, score the actual result |
| POST | `/verify/multiple` | Re-scan the two distributed piles (side by side in one frame), compare to the plan |

All endpoints accept `multipart/form-data` with a `file` field (the image) and return JSON. Full interactive docs are available at `http://localhost:8000/docs` once the backend is running.

## 12. Installation

### Prerequisites
- Python 3.10+
- Node.js 18+

### Windows

```bat
git clone <this-repo>
cd sibling-fairness-ai
```

**Backend:**
```bat
cd backend
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
```

**Frontend** (in a second terminal):
```bat
cd frontend
npm install
```

### macOS / Linux

```bash
cd backend
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

```bash
cd frontend
npm install
```

## 13. Running instructions

**Backend** (from `backend/`, with the venv activated):

```bash
uvicorn app.main:app --reload
```
Runs on `http://localhost:8000`.

**Frontend** (from `frontend/`):

```bash
npm run dev
```
Runs on `http://localhost:5173`. Copy `.env.example` to `.env` if you need to point it at a different backend URL than the default `http://localhost:8000`.

Open `http://localhost:5173` in a browser that has camera access (Chrome/Edge/Firefox all work; camera permission will be requested on the Scan page).

## 14. Testing

The mathematical core is testable independently of FastAPI/React:

```bash
cd sibling-fairness-ai
python -m pytest tests/ -v
# or, without pytest:
python tests/test_algorithms.py
```

Covers: rectangle / square / rotated rectangle / irregular polygon / circle cutting, equal-sized / differently-sized / odd-count multi-object splits, classification tiers, score bounds, and error cases (empty image, no object, multiple-object detection).

## 15. Limitations

- **Visual area, not weight.** Two objects can look similarly sized but weigh differently (e.g. a dense fruit vs. a hollow toy). *This fairness score is based on visual measurements. For true weight-based fairness, connect a weighing sensor in a future version.*
- Requires a reasonably plain, contrasting background — very cluttered or textured surfaces will hurt detection quality.
- 2D projection only: a tall object photographed from an angle will report the visible silhouette's area, not its true 3D volume.
- The `/verify/multiple` endpoint scores a single "both piles in one frame, left vs. right" shot as a practical default; it doesn't (yet) track individual object identity between the analyze and verify scans.

## 16. Future improvements

- **Weight sensor integration**: pair the camera with a load cell so the app can combine visual fairness + weight fairness for true mass-based 50:50 division (the code is already structured so this would slot in as an additional value signal in `fair_split.py`/`scoring.py` without touching the CV pipeline).
- Object re-identification between the analyze and verify scans (so verification can confirm *which* specific objects ended up in which pile, not just left/right position).
- A user-facing "What matters?" toggle (equal quantity / equal size / equal visual area) to override the automatic strategy selection.
- On-device model for object classification (so fruit, toys, and stationery could get type-aware value estimates instead of pure pixel area).
