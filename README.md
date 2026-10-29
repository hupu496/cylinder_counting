# AI Based Cylinder Counting & Inspection System

> **Offline AI-powered cylinder counting, cap detection, and multi-angle inspection system.**
> Source basis: *AI Based Cylinder Counting & Inspection System* (ComSys IT, 9-page concept note).
> Implementation stack: **Python (Django) + PostgreSQL (pgvector) + industrial cameras**.

---

## 1. Introduction

The AI-Based Cylinder Counting, Cap Detection & Multi-Angle Inspection System is an **offline** solution that automates the cylinder unloading process using industrial cameras and Artificial Intelligence. The system counts cylinders, verifies the presence of safety caps, captures images from multiple angles, and generates digital reports.

**Core logic:**

- **Counting is scan-based** — a cylinder is counted only when it crosses the scan line in the counting bay (one crossing = one count, no duplicates).
- **Camera placement is fixed by function** — one camera on **top of the counting bay** (scan-based counting + cap detection) and one camera at the **back of the truck** (loaded-row detection for the expected baseline count).

**Why it is needed:**

- Eliminate manual counting errors.
- Improve unloading speed and operational efficiency.
- Ensure every cylinder has a safety cap.
- Maintain digital records with image evidence.
- Enhance safety, accuracy, and process transparency.

---

## 2. Project Flow (Operational Workflow)

### 2.1 Main unloading flow

When a truck enters the unloading bay, the system follows these steps:

1. Truck arrives at the unloading bay — the **back-of-truck camera** captures the loaded truck bed.
2. AI detects loaded **rows** from the back-of-truck view → `rows × cylinders_per_row` = **row-expected count** (baseline).
3. Application records truck information (registration, type, capacity).
4. Workers unload cylinders.
5. Cylinders move one by one through the **counting bay** under the **top-mounted counting-bay camera**.
6. AI **scan-counts** every cylinder as it crosses the scan line (one crossing = one count, no duplicates).
7. AI checks whether the safety cap is present.
8. Both cameras capture cylinder images (evidence frames linked to the session).
9. Scan count is updated automatically (live).
10. Scan count is **reconciled** against the rear-camera row-expected count → accuracy verdict.
11. Final inspection report is generated.

### 2.2 Application flow by stage

```text
TRUCK ARRIVAL          SCAN COUNTING BAY           ROW CHECK (REAR)         INSPECTION & OUTPUT
─────────────          ──────────────────           ────────────────         ───────────────────
Truck arrives    →     Cylinder crosses      →     Rear cam detects   →     Scan vs row estimate
Rear cam: rows   →     scan line             →     rows × per-row     →     reconciled count
× per-row =      →     AI scan-count +       →     count (baseline)   →     Count + cap status
baseline         →     cap check             →      Frame quality     →     Operator review
Create session   →     Live count updated    →      check              →     Inspection report
```

### 2.3 Count data rules

Each session stores these values **separately** (never a single ambiguous `count`):

| Field | Meaning |
|---|---|
| `declared_capacity` | Capacity from truck master / operator — not an AI count |
| `ai_proposed_count` | Reconciled estimate from the vision pipeline |
| `cap_detected_count` | Cylinders verified to have a safety cap |
| `cap_missing_count` | Cylinders missing a safety cap |
| `verified_count` | Count accepted by an operator after review |
| `count_variance` | `verified_count - declared_capacity` |
| `count_status` | `pending_review` / `verified` / `needs_recount` / `exception` |

Rules:

- Never silently overwrite the AI estimate when an operator corrects a count — store both plus who changed it.
- Low image quality or low confidence → flag for review, do not auto-accept.
- Reports must label AI-estimated and operator-verified values distinctly.

### 2.4 Scan-based counting rule

The counting part shall be **scan-based**:

- The **top-of-counting-bay camera** defines a virtual **scan line** over the bay exit path. A cylinder is counted **only when it crosses this line** — no crossing, no count.
- One crossing = one count; re-detections of the same cylinder on the same crossing are suppressed (track ID + frame embedding de-duplication).
- Each counted cylinder triggers the cap check and an evidence frame from both cameras in the same event.
- The **back-of-truck camera** scans the loaded truck bed **before unloading** to detect **rows**; `rows × cylinders_per_row` becomes the expected baseline used to reconcile the running scan count.
- If `|scan_count − row_baseline|` exceeds the allowed tolerance, the session is flagged `needs_recount` for operator review — the system never auto-adjusts either number.

---

## 3. Camera Placement

Placement of the cameras is function-driven. Two cameras are **required** for the scan-based counting logic:

| Camera | Placement | Purpose |
|---|---|---|
| **Camera 1 (required)** | **Top of the counting bay** (overhead, facing down across the scan line) | **Scan-based counting** — counts each cylinder as it crosses the scan line — plus safety-cap detection |
| **Camera 2 (required)** | **Back of the truck** (fixed at the bay, viewing the loaded truck bed rows) | **Row detection** → `rows × cylinders_per_row` = row-expected baseline count |
| Camera 3/4 (optional) | Left / Right side | Complete cylinder documentation, image verification |
| Camera 5 (optional) | Front (based on site requirement) | Full documentation, damage inspection (future enhancement) |

Placement rules:

- **Camera 1 (top of counting bay):** mounted directly above the counting bay path so every unloaded cylinder passes through its field of view and crosses the scan line — full coverage of the bay width, no blind spots at the crossing point; the scan line is defined in calibration.
- **Camera 2 (back of the truck):** mounted at bay level behind the truck so the entire loaded bed and all **rows** are visible before unloading starts; must see full width/depth of the rows, unobstructed by tarps/gates.
- Required cameras must be online for a session to start; optional cameras are best-effort.
- All cameras are time-synchronized; each captured frame is linked to a truck visit + counting session.
- Camera credentials are stored encrypted (never in source control).

---

## 4. Report Generation

After the unloading process is completed, the system automatically generates a detailed inspection report containing:

- Truck details (registration, type, declared capacity)
- Total cylinder count (AI-proposed vs operator-verified)
- Cap detection results (present / missing counts)
- Captured multi-angle images as evidence
- Date, time, and inspection status
- Discrepancy flags and review history

Reports are searchable, storable, and exportable (PDF / CSV / XLSX) for future reference and auditing.

---

## 5. System Architecture

```text
 ┌────────────────────────────────────────────────────────────────────────────┐
 │                          INDUSTRIAL CAMERAS (ON-PREM)                      │
 │  Cam1 top of bay: scan-count   Cam2 back of truck: rows   Cam3/4 sides     │
 └───────────────┬────────────────────────┬───────────────────────────────────┘
                 │ RTSP/ONVIF frames      │
        ┌────────▼────────────────────────▼────────┐
        │  Capture Service (Django + Celery worker) │
        │  - frame grabber / stream ingestion       │
        │  - timestamp sync, quality check          │
        │  - evidence file storage (local/NAS)      │
        └────────────────┬─────────────────────────┘
                         │
        ┌────────────────▼─────────────────────────┐
        │        AI VISION PIPELINE (offline)       │
        │  1. Scan-line cylinder counting (YOLO)    │
        │  2. Cap detection       (presence/absent) │
        │  3. Multi-angle capture reconciliation    │
        │  4. Quality / confidence scoring          │
        │  5. Image embedding (pgvector)            │
        └────────────────┬─────────────────────────┘
                         │
     ┌───────────────────▼───────────────────────────┐
     │              DJANGO APPLICATION               │
     │  truck_master │ visit_management │ cameras    │
     │  sessions     │ detection/caps   │ review     │
     │  reports      │ audit_logs       │ users/RBAC │
     └───────┬───────────────────┬───────────────────┘
             │                   │
   ┌─────────▼─────────┐  ┌──────▼──────────────────────────┐
   │   PostgreSQL 16   │  │ Evidence storage (frames/clips) │
   │  + pgvector       │  │  local disk / NAS / S3-minio    │
   │  - sessions       │  └─────────────────────────────────┘
   │  - counts         │
   │  - detections     │  ┌─────────────────────────────────┐
   │  - embeddings     │  │ Reports: PDF / CSV / XLSX       │
   │  - audit          │  │ + review console (Django views) │
   └───────────────────┘  └─────────────────────────────────┘
```

### Architecture layers

1. **Edge/Camera layer** — industrial cameras: the **top-of-counting-bay camera** (scan-based counting, cap check) and the **back-of-truck camera** (row detection) are required; optional side/front cameras as needed. RTSP/ONVIF streams or frame polling.
2. **Ingestion layer** — Celery workers grab frames, run quality checks, and write evidence files.
3. **AI layer** — fully offline detection (cylinder + cap), multi-angle reconciliation, confidence flags, and embedding generation.
4. **Application layer** — Django / Django REST Framework apps for visits, sessions, review, and reporting.
5. **Data layer** — PostgreSQL 16 + **pgvector** for relational data and image/region embeddings; filesystem/object store for evidence.
6. **Presentation layer** — Django templates or DRF + frontend for dashboard, live monitoring, verification, and reports.

---

## 6. Technology Stack

| Layer | Choice |
|---|---|
| Language | Python 3.11+ |
| Web framework | Django + Django REST Framework |
| Task queue | Celery + Redis (frame capture, report jobs) |
| Database | PostgreSQL 16 with **pgvector** extension |
| Computer vision | OpenCV + YOLO-class detector (offline inference) |
| Embeddings | Image embedding model → `vector` column in PostgreSQL |
| Frontend | Django templates / HTML + JS (dashboard, review console) |
| Reporting | WeasyPrint (PDF) + CSV/XLSX export |
| Server | Gunicorn + Nginx, on-prem edge server (fully offline) |
| Auth | Django built-in auth + role-based permissions |

### Why pgvector

- **Near-duplicate frame detection** — compare frame embeddings to avoid double-counting the same cylinder pass.
- **Evidence retrieval** — semantic/image similarity search ("show all damaged-cap images for this truck type").
- **Cluster defect/cap-missing examples** — group similar exception images for faster operator review.
- **Audit-friendly** — embeddings stored beside relational count data in one transactional database.

Example schema:

```sql
CREATE EXTENSION IF NOT EXISTS vector;

ALTER TABLE capture_frame
  ADD COLUMN embedding vector(512);

CREATE INDEX ON capture_frame
  USING hnsw (embedding vector_cosine_ops);
```

---

## 7. Proposed Data Model (Django apps / models)

| Model | Key fields |
|---|---|
| `Truck` | `registration`, `truck_type`, `declared_capacity`, `active` |
| `TruckVisit` | `truck`, `visit_reference`, `arrival_time`, `departure_time`, `status` |
| `Camera` | `camera_code`, `view_type` (`top_counting_bay` / `rear_truck` / `left` / `right` / `front`), `required` (mandatory for scan counting), `stream_ref` (encrypted), `calibration_config` (incl. scan line), `active` |
| `CountingSession` | `visit`, `event_type`, `detected_rows`, `cylinders_per_row`, `row_baseline_count` (rows × per-row), `scan_count`, `ai_proposed_count`, `verified_count`, `cap_detected_count`, `cap_missing_count`, `count_variance`, `count_status`, `model_version`, `reviewed_by` |
| `Capture` | `session`, `camera`, `captured_at`, `image_ref`, `quality_status`, `embedding` |
| `Detection` | `capture`, `class_name` (cylinder/cap), `bbox`, `confidence`, `track_ref` |
| `CapResult` | `detection`, `cap_present`, `confidence` |
| `CountRevision` | `session`, `old_count`, `new_count`, `reason`, `changed_by` |
| `InspectionReport` | `session`, `status`, `generated_at`, `pdf_ref` |
| `AuditLog` | `user`, `action`, `entity_type`, `entity_id`, `timestamp`, `details` |

---

## 8. Project Structure

```text
cylinder_counting/
├── README.md
├── manage.py
├── requirements.txt
├── config/                     # Django project settings (settings/urls/wsgi)
├── apps/
│   ├── truck_master/           # truck identity, type, capacity
│   ├── visit_management/       # arrival/departure lifecycle
│   ├── camera_management/      # camera config, health, calibration
│   ├── capture_service/        # frame grabber (Celery), quality checks
│   ├── cylinder_detection/     # AI pipeline: cylinder + cap detection
│   ├── count_reconciliation/   # multi-view merge, confidence, flags
│   ├── review_console/         # operator verification & corrections
│   ├── reports/                # PDF/CSV/XLSX inspection reports
│   ├── audit_logs/             # audit trail & monitoring
│   └── users/                  # auth, roles, permissions
├── ai_models/
│   ├── detection/              # trained cylinder detector
│   ├── cap_detection/          # cap presence model
│   ├── embeddings/             # image embedding generator (pgvector)
│   └── evaluation/             # accuracy metrics vs ground truth
├── database/
│   └── schema/                 # migrations, pgvector DDL, seeds
├── evidence/                   # captured frames/clips (git-ignored)
├── templates/                  # dashboard, live monitoring, reports
├── tests/
│   ├── unit/
│   ├── integration/
│   └── accuracy/
└── deployment/
    ├── configuration/          # env, gunicorn, nginx, celery
    └── backup/
```

---

## 9. Implementation Roadmap (4 Phases)

| Phase | Activity | Estimated Time |
|---|---|---|
| **Phase 1: Hardware Installation** | Mount the **top-of-counting-bay camera** (over the scan line) and the **back-of-truck camera** (row detection); optional side cameras; cabling and networking infrastructure for camera connectivity and data transmission; scan-line calibration. | 1–2 weeks |
| **Phase 2: Data Collection & AI Model Training** | Capture and label **1,500–2,000 images per truck type** based on location; train and validate the counting + cap detection models. | 3–4 weeks |
| **Phase 3: Server Setup & Deployment** | Install the AI processing application on-site; integrate with existing plant operations and reporting systems; provision PostgreSQL + pgvector. | 1–2 weeks |
| **Phase 4: Testing & Go-Live** | Pilot runs, iterative refinements, full-scale production deployment. | 2–3 weeks |

**Total estimated duration: 7–11 weeks.**

---

## 10. Advantages

- Fully offline — data stays inside the plant environment
- High accuracy, low human error
- AI-based image analysis (cylinder + cap)
- Faster than manual counting
- Automatic report generation with image proof
- Digital records with searchable history
- Detects missing safety caps
- Scalable across bays/truck types

---

## 11. Getting Started

```bash
# 1. Clone and create virtual environment
python -m venv .venv
.venv\Scripts\activate          # Windows
# source .venv/bin/activate     # Linux

# 2. Install dependencies
pip install -r requirements.txt

# 3. Database (PostgreSQL + pgvector)
createdb cylinder_counting
psql -d cylinder_counting -c "CREATE EXTENSION IF NOT EXISTS vector;"

# 4. Environment variables (never commit secrets)
#    DATABASE_URL, SECRET_KEY, EVIDENCE_ROOT, CAMERA_STREAM_URLS...

# 5. Migrate and run
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver

# 6. Celery worker for capture/AI jobs
celery -A config worker --loglevel=info
```

**Minimum `requirements.txt`:** `django`, `djangorestframework`, `psycopg[binary]`, `pgvector`, `celery`, `redis`, `opencv-python`, `ultralytics`, `pillow`, `weasyprint`, `gunicorn`.

---

## 12. Security, Offline Operation & Audit

- Fully offline deployment — no external API calls during inference.
- Authenticated access with role-based permissions (operator / supervisor / admin).
- Camera passwords, DB credentials, and API keys kept in environment variables — never in source control.
- Original AI outputs and every operator revision preserved with audit history.
- Processing failures are logged and retryable without duplicate records.
- Evidence retention, backup, and restore policies defined with the site owner.

---

## 13. Open Questions / Inputs Required

| Item | Status |
|---|---|
| Exact cylinder capacity per truck type | **Confirm with site owner** |
| Cylinder types/sizes and mixed loads | TBD |
| Camera make, resolution, FPS, lens, stream protocol | TBD |
| Optional cameras (left/right/front) site requirement | TBD |
| Scan-line definition & trigger method (virtual line vs. hardware trigger) | TBD |
| Row-detection tolerance — allowed scan vs. row-baseline mismatch | TBD |
| Counting live vs. after truck stops in bay | TBD |
| Expected count accuracy and allowable error | TBD |
| Evidence retention period | TBD |
| Report fields/export formats required | TBD |
| Integration with gate, weighbridge, RFID, ERP | Not specified; confirm if required |
| Damage inspection scope (future enhancement) | TBD |

---

## 14. Source Note

Derived from the concept note *AI Based Cylinder Counting & Inspection System* (ComSys IT): truck arrival → unloading bay → conveyor → AI cylinder detection → cap detection → multi-angle camera capture → automatic count → inspection report. All modules, schema, and technology choices above are implementation recommendations to be confirmed with stakeholders during the site survey.
