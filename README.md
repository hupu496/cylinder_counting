# cylinder_counting
The system uses CCTV cameras to observe cylinders in a truck-loading/counting area, analyze the top cylinder view and rear row view, collect processed data in software, and generate reports.
# Camera-Based Cylinder Counting System

> **Project planning document**\
> Source basis: *Concept Note -- Cylinder Counting through Camera* (3
> pages).\
> Status: Proposed architecture for implementation; details not
> specified in the concept note are marked **TBD** and must be confirmed
> before production.

## 1. Project Overview

The system uses CCTV cameras to observe cylinders in a
truck-loading/counting area, analyze the top cylinder view and rear row
view, collect processed data in software, and generate reports.

The concept note describes two operational workflows:

1.  **Truck arrival / loading workflow**
2.  **Truck departure workflow**

The concept specifies cameras at the top of the counting area and on one
rear side. The top cameras capture the cylinder view; the rear camera
captures the row view. The system then processes the data, stores it in
software, and generates a report.

### Important source limitation

The concept note does **not** provide a confirmed cylinder quantity,
truck-capacity table, camera specifications, AI model, database design,
or required counting accuracy. Do not hard-code or present any of these
as confirmed facts. Obtain them from the plant/site team and validate
them during commissioning.

## 2. Goals

-   Detect and count visible cylinders in camera frames.
-   Analyze cylinder rows using top and rear camera views.
-   Associate each counting session with a truck and its declared
    capacity/type.
-   Keep arrival/loading and departure counts as separate events.
-   Show the detected count, reviewed/verified count, declared capacity,
    and any discrepancy separately.
-   Preserve evidence and processing history so a count can be audited.
-   Generate searchable and exportable reports.
-   Flag low-confidence, obscured, duplicate, or inconsistent detections
    for human review.

## 3. Scope

### In scope for the proposed first release

-   Truck/session registration.
-   Truck capacity and truck-type lookup.
-   Camera capture or ingestion of recorded images/video.
-   Cylinder detection and count estimation.
-   Row analysis from the rear view.
-   Combining evidence from multiple views without blindly adding
    overlapping detections.
-   Manual review and count correction with an audit trail.
-   Arrival/departure reports and count discrepancy alerts.
-   User roles, logs, and configurable site/camera settings.

### Out of scope until confirmed

-   Automatic truck barrier control or gate integration.
-   Weighbridge, RFID, ANPR, ERP, or other external integrations.
-   A guaranteed counting accuracy percentage.
-   A fixed number of cylinders per truck.
-   Automatic decisions based solely on an unreviewed AI count.

## 4. Operational Workflows

### 4.1 Arrival / loading

1.  Create or identify the truck visit.
2.  Fetch/enter truck capacity and truck type.
3.  Move the truck into the camera zone.
4.  Capture top-camera cylinder views.
5.  Capture the rear-camera row view.
6.  Run image quality checks and cylinder/row detection.
7.  Reconcile observations across camera views.
8.  Present the proposed count and confidence to an operator.
9.  Operator verifies or corrects the count when needed.
10. Save the event, evidence, and audit history.
11. Generate the arrival/loading report.

### 4.2 Departure

1.  Identify the same truck visit or create a departure session.
2.  Fetch/confirm truck capacity and truck type.
3.  Move the truck into the camera zone.
4.  Capture top-camera cylinder views.
5.  Capture the rear-camera row view.
6.  Process the views and estimate the departure count.
7.  Compare the verified departure count with the relevant loading count
    and declared capacity.
8.  Flag discrepancies for review; do not silently overwrite prior
    counts.
9.  Save evidence and audit history.
10. Generate the departure report.

## 5. Proposed System Architecture

``` text
                 ┌────────────────────────────┐
                 │ Truck / Visit Registration │
                 │ truck type + capacity      │
                 └──────────────┬─────────────┘
                                │
                 ┌──────────────▼─────────────┐
                 │ Camera Capture / Ingestion │
                 └──────────────┬─────────────┘
                                │
               ┌────────────────┴────────────────┐
               │                                 │
    ┌──────────▼──────────┐           ┌──────────▼──────────┐
    │ Top CCTV Camera(s)  │           │ Rear CCTV Camera    │
    │ cylinder/top view   │           │ row/side view       │
    └──────────┬──────────┘           └──────────┬──────────┘
               │                                 │
    ┌──────────▼──────────┐           ┌──────────▼──────────┐
    │ Quality checks      │           │ Quality checks      │
    │ + cylinder detect   │           │ + row/count detect  │
    └──────────┬──────────┘           └──────────┬──────────┘
               └────────────────┬────────────────┘
                                │
                 ┌──────────────▼─────────────┐
                 │ Multi-view reconciliation  │
                 │ avoid duplicate counting   │
                 └──────────────┬─────────────┘
                                │
                 ┌──────────────▼─────────────┐
                 │ Count + confidence + flags │
                 └──────────────┬─────────────┘
                                │
                 ┌──────────────▼─────────────┐
                 │ Operator verification      │
                 │ correction + audit trail   │
                 └──────────────┬─────────────┘
                                │
        ┌───────────────────────┼────────────────────────┐
        │                       │                        │
┌───────▼────────┐    ┌─────────▼────────┐    ┌──────────▼────────┐
│ Database       │    │ Evidence storage │    │ Reports / exports │
│ sessions/counts│    │ frames/clips     │    │ PDF/CSV/XLSX TBD  │
└────────────────┘    └──────────────────┘    └───────────────────┘
```

This is a proposed logical architecture, not a claim that these
components already exist.

## 6. Exact Cylinder Count: Data and Rules

The application should never use one ambiguous field called `count` for
every stage. Store the following values separately:

  ---------------------------------------------------------------------------
  Field                               Meaning
  ----------------------------------- ---------------------------------------
  `declared_capacity`                 Capacity supplied by the truck
                                      master/site operator; not an AI count.

  `top_view_count`                    Count estimated from the top camera
                                      view(s).

  `rear_view_count`                   Count/row estimate from the rear camera
                                      view.

  `ai_proposed_count`                 Reconciled estimate produced by the
                                      counting pipeline.

  `verified_count`                    Count accepted by an authorized
                                      operator after review.

  `count_variance`                    `verified_count - declared_capacity`,
                                      when capacity is known.

  `loading_count`                     Verified count recorded for the loading
                                      event.

  `departure_count`                   Verified count recorded for the
                                      departure event.

  `loading_departure_variance`        `departure_count - loading_count`, when
                                      both are available.

  `count_status`                      For example: `pending_review`,
                                      `verified`, `needs_recount`,
                                      `exception`.
  ---------------------------------------------------------------------------

**Do not add `top_view_count + rear_view_count` by default.** Both views
may see the same cylinders. Use camera geometry, calibrated zones,
tracking, and reconciliation logic to avoid duplicate counts.

### Count decision rules

1.  If image quality is inadequate, mark the session `needs_recount` or
    `pending_review`.
2.  If camera views disagree beyond a configurable tolerance, require
    review.
3.  If capacity is unknown, store it as unknown; do not assume a default
    capacity.
4.  If the AI estimate has low confidence, require human verification.
5.  Never replace the original AI estimate when an operator corrects the
    count. Store both values and who changed the verified count.
6.  A report must label AI-estimated and operator-verified values
    distinctly.
7.  Define the discrepancy sign convention explicitly: this document
    uses **actual/verified count minus comparison value**.

## 7. Proposed Data Model

These are suggested entities; exact schema should be finalized after
requirements discovery.

### `Truck`

-   `id`
-   `truck_registration` (unique, if applicable)
-   `truck_type`
-   `declared_capacity` (nullable until confirmed)
-   `active`
-   `created_at`, `updated_at`

### `TruckVisit`

-   `id`
-   `truck_id`
-   `visit_reference`
-   `arrival_time`
-   `departure_time` (nullable)
-   `visit_status`
-   `created_by`

### `CountingSession`

-   `id`
-   `truck_visit_id`
-   `event_type` (`arrival_loading` or `departure`)
-   `started_at`, `completed_at`
-   `top_view_count` (nullable)
-   `rear_view_count` (nullable)
-   `ai_proposed_count` (nullable)
-   `verified_count` (nullable)
-   `declared_capacity_snapshot` (nullable)
-   `count_variance` (nullable)
-   `count_status`
-   `model_version` (nullable)
-   `reviewed_by` (nullable)
-   `reviewed_at` (nullable)
-   `notes`

### `Camera`

-   `id`
-   `camera_code`
-   `view_type` (`top` or `rear`)
-   `location_description`
-   `stream_reference` (secret credentials must not be stored in plain
    text)
-   `calibration_config`
-   `active`

### `Capture`

-   `id`
-   `session_id`
-   `camera_id`
-   `captured_at`
-   `image_or_clip_reference`
-   `quality_status`
-   `processing_status`

### `Detection`

-   `id`
-   `capture_id`
-   `class_name`
-   `bounding_box_or_mask`
-   `confidence`
-   `track_or_object_reference` (nullable)
-   `review_status`

### `CountRevision`

-   `id`
-   `session_id`
-   `old_verified_count` (nullable)
-   `new_verified_count`
-   `reason`
-   `changed_by`
-   `changed_at`

### `AuditLog`

-   `id`
-   `user_id`
-   `action`
-   `entity_type`
-   `entity_id`
-   `timestamp`
-   `details`

## 8. Proposed Software Modules

1.  **Truck Master** --- truck identity, type, and capacity.
2.  **Visit Management** --- arrival/departure lifecycle.
3.  **Camera Management** --- camera configuration, health, and
    calibration.
4.  **Capture Service** --- timestamped images/clips and capture
    metadata.
5.  **Vision Pipeline** --- image-quality checks, cylinder detection,
    row analysis.
6.  **Count Reconciliation** --- multi-view de-duplication and count
    confidence.
7.  **Review Console** --- image evidence, count verification,
    correction reasons.
8.  **Reports** --- visit-wise, date-wise, truck-wise, loading/departure
    and exception reports.
9.  **User and Permissions** --- authentication and role-based access.
10. **Audit and Monitoring** --- error logs, processing status, camera
    health, and count changes.
11. **Configuration** --- site settings, thresholds, model version, and
    retention policy.

## 9. Suggested Technology Choices (Not specified by the concept note)

Choose these only after checking site constraints and the existing
environment.

-   **Backend:** Django / Django REST Framework, or another agreed
    backend.
-   **Database:** PostgreSQL or the site's approved database.
-   **Computer vision:** OpenCV plus a validated
    object-detection/segmentation model.
-   **Frontend:** Django templates or a separate web frontend.
-   **Evidence storage:** controlled local/NAS/object storage with
    retention rules.
-   **Reporting:** PDF and CSV/XLSX if requested by users.
-   **Deployment:** on-premises edge computer/server if camera-network
    latency or plant-network restrictions require it.

These are recommendations, not source-confirmed project choices. The AI
model must be evaluated on real images from the target site before
selection.

## 10. Development Phases and Deliverables

### Phase 1 --- Requirements and site survey

-   Confirm cylinder types and physical appearance.
-   Confirm maximum/typical cylinders per truck and the authoritative
    capacity source.
-   Confirm camera count, placement, resolution, frame rate, lighting,
    and network access.
-   Collect representative footage for empty, partial, full, occluded,
    overlapping, and difficult-lighting cases.
-   Agree accuracy, throughput, review, retention, and reporting
    targets.

**Exit condition:** approved requirements and a labelled sample dataset.

### Phase 2 --- Capture proof of concept

-   Connect to or ingest footage from each camera.
-   Save synchronized timestamps and truck/session references.
-   Check image quality and camera coverage.
-   Demonstrate a repeatable capture process.

**Exit condition:** usable top and rear views are available for test
sessions.

### Phase 3 --- Counting prototype

-   Label cylinder instances and row patterns.
-   Train/evaluate a suitable detector or segmentation model.
-   Implement view-specific counting and duplicate-count prevention.
-   Produce a proposed count and confidence/exception flags.

**Exit condition:** results measured against manually labelled ground
truth on a held-out dataset.

### Phase 4 --- Application and database

-   Build truck master, visits, sessions, camera configuration, and
    evidence records.
-   Implement operator review and count correction.
-   Add authentication, permissions, and audit logging.

**Exit condition:** a complete session can be created, processed,
reviewed, and saved.

### Phase 5 --- Reports and reconciliation

-   Build arrival/loading and departure reports.
-   Compare verified count with declared capacity and, where applicable,
    loading count.
-   Add exception lists and export formats confirmed by stakeholders.

**Exit condition:** report values match stored session data and approved
calculation rules.

### Phase 6 --- Site pilot and acceptance

-   Run in shadow mode alongside manual counting.
-   Compare machine counts with independently verified ground truth.
-   Analyze errors by camera, lighting, cylinder arrangement, and
    occlusion.
-   Tune calibration/model and document known limitations.
-   Obtain site acceptance before relying on automated counts
    operationally.

**Exit condition:** stakeholder-approved acceptance tests pass.

## 11. Testing and Acceptance Plan

Track at least these metrics:

-   **Exact-count accuracy:** percentage of sessions where predicted
    count exactly equals verified ground truth.
-   **Mean absolute count error:** average of
    `abs(predicted_count - ground_truth_count)`.
-   **Overcount rate:** frequency and magnitude of counts above ground
    truth.
-   **Undercount rate:** frequency and magnitude of counts below ground
    truth.
-   **Review rate:** percentage of sessions requiring operator
    intervention.
-   **Processing latency:** time from capture completion to proposed
    count.
-   **Evidence coverage:** percentage of sessions with usable
    images/clips from required cameras.
-   **Report correctness:** database values and report totals agree.

Test cases should include: - Empty truck/zone and no-cylinder frames. -
Partial and full loads. - Cylinders touching, overlapping, or
occluded. - Different cylinder orientations and row patterns. - Shadows,
glare, night lighting, rain/dust if relevant at the site. - Missing or
disconnected camera. - Repeated frames and duplicate processing. -
Unknown truck capacity. - Arrival and departure for the same visit. -
Operator correction and audit-history verification.

**Acceptance thresholds are TBD.** Set numerical thresholds only after
the site owner approves them and a representative test dataset is
available.

## 12. Security, Reliability, and Audit

-   Use authenticated access and role-based permissions.
-   Restrict access to camera streams and stored evidence.
-   Do not place camera passwords, database credentials, or API keys in
    source control.
-   Keep timestamps consistent across cameras and application services.
-   Log processing failures and make failed sessions retryable without
    duplicate records.
-   Preserve original AI outputs and all operator revisions.
-   Define evidence retention, deletion, backup, and restore policies
    with the site owner.
-   Monitor camera connectivity, disk space, queue depth, and processing
    errors.

## 13. Open Questions / Inputs Required

  -----------------------------------------------------------------------
  Item                                Status
  ----------------------------------- -----------------------------------
  Exact cylinder count or capacity    **Not provided --- confirm with
  for each truck type                 site owner**

  Cylinder types/sizes and whether    TBD
  mixed types occur                   

  Number and exact placement of top   Concept says top CCTV cameras;
  cameras                             exact number/layout TBD

  Rear camera placement and field of  One rear-side view is described;
  view                                detailed placement TBD

  Camera make, resolution, FPS, lens, TBD
  and stream protocol                 

  Whether counting is live or after   TBD
  truck stops in zone                 

  Authoritative truck                 TBD
  registration/capacity data source   

  Expected count accuracy and         TBD
  allowable error                     

  Human verification requirements     Proposed for uncertain cases; final
                                      policy TBD

  Database, hosting, and              TBD
  plant-network constraints           

  Required report fields and export   TBD
  formats                             

  Retention period for images/videos  TBD

  Integration with gate, weighbridge, Not specified; confirm whether
  RFID, ANPR, or ERP                  required
  -----------------------------------------------------------------------

## 14. Initial Release Definition of Done

The first release is ready for pilot when:

-   A truck visit can be registered and assigned a truck type/capacity.
-   Arrival/loading and departure are distinct event types.
-   Required camera evidence is captured and linked to the correct
    session.
-   The system produces a proposed cylinder count and records its
    source/model version.
-   The operator can verify or correct the count with a reason.
-   AI count, verified count, capacity, and variances are shown
    separately.
-   Low-quality or inconsistent sessions are flagged rather than
    silently accepted.
-   Reports can be generated from saved data and reconciled with the
    database.
-   Access control, error logging, backup, and audit history are tested.
-   Site-approved accuracy and performance acceptance tests pass.

## 15. Source Note

The uploaded concept note describes: - Cylinder counting through cameras
using top and rear-side views. - Truck-arrival workflow: truck arrival,
truck data/capacity/type, camera zone, top view, rear row view, data
processing, software data collection, report generation. -
Truck-departure workflow with a similar capture and reporting sequence.

The source does not provide implementation-level specifications. All
proposed modules, schema, validation rules, technology choices, and
development phases above are planning recommendations to be confirmed
with stakeholders.
