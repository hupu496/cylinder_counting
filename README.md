# AI Based Cylinder Counting & Inspection System

> **Offline AI-powered cylinder counting, cap detection, and multi-angle inspection system.**
> Source basis: *AI Based Cylinder Counting & Inspection System* (ComSys IT, 9-page concept note).
> Implementation stack: **Python (Django) + PostgreSQL (pgvector) + industrial cameras**.

---

## 1. Introduction

The AI-Based Cylinder Counting, Cap Detection & Multi-Angle Inspection System is an **offline** solution that automates the cylinder unloading process using industrial cameras and Artificial Intelligence. The system counts cylinders, verifies the presence of safety caps, captures images from multiple angles, and generates digital reports.

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

1. Truck arrives at the unloading bay.
2. Application records truck information (registration, type, capacity).
3. Workers unload cylinders.
4. Cylinders move one by one on the conveyor.
5. AI detects every cylinder.
6. AI checks whether the safety cap is present.
7. Multiple cameras capture cylinder images.
8. Count is updated automatically.
9. Final inspection report is generated.

### 2.2 Application flow by stage

```text
TRUCK ARRIVAL                UNLOADING / CONVEYOR              INSPECTION & OUTPUT
─────────────                ────────────────────              ───────────────────
Truck arrives        →       Cylinders on conveyor      →      AI cylinder detection
Record truck info    →       Cap presence check         →      Multi-angle capture
Create visit/session →       Live count updated         →      Count + cap status stored
                             Frame quality check        →      Operator review (if flagged)
                                                            →   Inspection report (PDF/CSV)
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

---

## 3. Camera Placement

| Camera | View | Purpose |
|---|---|---|
| Camera 1 | Top | Cylinder detection, counting, cap detection |
| Camera 2 | Left side | Complete cylinder documentation, image verification |
| Camera 3 | Right side | Complete cylinder documentation, image verification |
| Camera 4/5 | Front / Rear (based on site requirement) | Full documentation, damage inspection (future enhancement) |

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
 │  Cam1 Top (detect/count/cap)   Cam2 Left   Cam3 Right   Cam4/5 Front/Rear  │
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
        │  1. Cylinder detection  (YOLO/segment.)   │
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

1. **Edge/Camera layer** — industrial cameras on the unloading bay; RTSP/ONVIF streams or frame polling.
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
| `Camera` | `camera_code`, `view_type` (top/left/right/front/rear), `stream_ref` (encrypted), `calibration_config`, `active` |
| `CountingSession` | `visit`, `event_type`, `ai_proposed_count`, `verified_count`, `cap_detected_count`, `cap_missing_count`, `count_variance`, `count_status`, `model_version`, `reviewed_by` |
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
| **Phase 1: Hardware Installation** | Mount HD cameras at the inspection bay; cabling and networking infrastructure for camera connectivity and data transmission. | 1–2 weeks |
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
| Camera 4/5 (front/rear) site requirement | TBD |
| Counting live vs. after truck stops in bay | TBD |
| Expected count accuracy and allowable error | TBD |
| Evidence retention period | TBD |
| Report fields/export formats required | TBD |
| Integration with gate, weighbridge, RFID, ERP | Not specified; confirm if required |
| Damage inspection scope (future enhancement) | TBD |

---

## 14. Source Note

Derived from the concept note *AI Based Cylinder Counting & Inspection System* (ComSys IT): truck arrival → unloading bay → conveyor → AI cylinder detection → cap detection → multi-angle camera capture → automatic count → inspection report. All modules, schema, and technology choices above are implementation recommendations to be confirmed with stakeholders during the site survey.
