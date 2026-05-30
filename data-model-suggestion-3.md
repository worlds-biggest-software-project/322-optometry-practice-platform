# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Optometry Practice Platform · Created: 2026-05-25

## Philosophy

An optometry practice is a compliance-heavy environment: HIPAA requires immutable audit trails for every access to PHI, clinical notes must track amendments with original content preserved, insurance claims go through a multi-step lifecycle (draft → submitted → accepted/denied → appealed → paid), and prescriptions have legal significance with expiration dates and refill tracking. An event-sourced architecture makes compliance the default rather than an afterthought: every state change — from a patient checking in, to a refraction measurement being recorded, to a claim being denied — is stored as an immutable event. The current state of any entity is derived by replaying its event stream.

This approach is particularly powerful for optometry because the clinical workflow is naturally event-driven: check-in → pre-testing → refraction → exam → diagnosis → prescription → dispensing → billing. Each step generates events that trigger downstream processing. A refraction measurement event can automatically populate the prescription draft. A prescription event triggers charge capture. A dispensing event triggers inventory adjustment and insurance claim generation. The event stream is the single source of truth; read models (materialised views) are rebuilt from events to serve specific query patterns.

The immutable event store also solves the amendment problem: HIPAA and malpractice requirements dictate that clinical notes cannot be deleted or silently modified. In an event-sourced model, amendments are new events that reference the original, preserving the complete clinical timeline. Insurance claim lifecycle events provide a clear audit trail for denial management and appeals.

**Best for:** Practices requiring bulletproof HIPAA audit compliance, insurance claim lifecycle tracking with full history, clinical note amendment trails for malpractice protection, and teams that want event-driven automation (auto-charge capture, eligibility checks, recall reminders) built into the data layer.

**Trade-offs:**
- **Pro:** Complete, immutable audit trail — HIPAA compliance by construction
- **Pro:** Clinical note amendments preserve original content (malpractice protection)
- **Pro:** Insurance claim lifecycle fully traceable (every status change with reason)
- **Pro:** Event-driven automation — charge capture, eligibility, recalls from event streams
- **Pro:** Temporal queries ("what was the patient's prescription on 2025-03-01?") are natural
- **Pro:** FHIR export as event projection — each encounter's events map to a FHIR Bundle
- **Con:** Read models must be maintained and rebuilt when projections change
- **Con:** Higher write amplification — every state change is an insert, never an update
- **Con:** Complex queries require well-designed read models rather than ad-hoc JOINs
- **Con:** Event schema evolution requires careful versioning
- **Con:** Team must understand CQRS pattern — steeper learning curve

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Read models project encounter events into FHIR VisionPrescription, Observation, Claim resources |
| FHIR Eye Care IG | Ophthalmic observation events map to FHIR Eye Care profiles on projection |
| HL7 v2.x ORU/MDM | Device result messages stored directly as events with raw payload preserved |
| DICOM | OCT/visual field events include DICOM Study UID; image archive lookup by event |
| ANSI Z80 | Lens specification events carry Z80-aligned measurements |
| NCPDP SCRIPT | E-prescribing events carry NCPDP message IDs and status |
| ANSI X12 837P/835/270/271 | Claim events reference EDI transaction IDs; ERA events parse 835 responses |
| ICD-10-CM / CPT | Diagnosis and procedure events carry standard codes |
| HIPAA Security Rule | Immutable event store satisfies audit trail requirement; access events logged |
| CloudEvents v1.0 | Event envelope follows CloudEvents spec (ce_source, ce_type, ce_specversion) |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

---

## Event Store

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    event_data JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    sequence_number BIGINT NOT NULL,
    ce_source TEXT NOT NULL DEFAULT '/optometry-platform',
    ce_specversion TEXT NOT NULL DEFAULT '1.0',
    created_by UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, sequence_number)
) PARTITION BY RANGE (created_at);

CREATE TABLE event_store_2026_h1 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-07-01');
CREATE TABLE event_store_2026_h2 PARTITION OF event_store
    FOR VALUES FROM ('2026-07-01') TO ('2027-01-01');

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, sequence_number);
CREATE INDEX idx_events_type ON event_store(event_type, created_at DESC);
CREATE INDEX idx_events_creator ON event_store(created_by, created_at DESC);
CREATE INDEX idx_events_data ON event_store USING GIN (event_data jsonb_path_ops);
```

---

## Event Type Registry

```sql
CREATE TABLE event_types (
    event_type TEXT PRIMARY KEY,
    stream_type TEXT NOT NULL,
    description TEXT NOT NULL,
    schema_version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO event_types (event_type, stream_type, description) VALUES
-- Patient stream
('patient.registered', 'patient', 'New patient registration with demographics'),
('patient.demographics_updated', 'patient', 'Patient demographics changed'),
('patient.insurance_added', 'patient', 'Insurance coverage added'),
('patient.insurance_updated', 'patient', 'Insurance coverage modified'),
('patient.insurance_removed', 'patient', 'Insurance coverage terminated'),
('patient.history_updated', 'patient', 'Medical/ocular/family history updated'),
('patient.allergy_added', 'patient', 'Allergy recorded'),
('patient.medication_updated', 'patient', 'Current medications updated'),
('patient.deactivated', 'patient', 'Patient marked inactive'),

-- Appointment stream
('appointment.scheduled', 'appointment', 'Appointment created'),
('appointment.confirmed', 'appointment', 'Patient confirmed appointment'),
('appointment.checked_in', 'appointment', 'Patient arrived and checked in'),
('appointment.roomed', 'appointment', 'Patient taken to exam room'),
('appointment.completed', 'appointment', 'Appointment completed'),
('appointment.cancelled', 'appointment', 'Appointment cancelled'),
('appointment.no_show', 'appointment', 'Patient did not show'),
('appointment.rescheduled', 'appointment', 'Appointment moved to new time'),

-- Encounter stream
('encounter.started', 'encounter', 'Clinical encounter opened'),
('encounter.chief_complaint_recorded', 'encounter', 'Chief complaint documented'),
('encounter.visual_acuity_measured', 'encounter', 'Visual acuity testing completed'),
('encounter.autorefraction_recorded', 'encounter', 'Autorefractor results captured'),
('encounter.manifest_refraction_recorded', 'encounter', 'Manifest refraction performed'),
('encounter.cycloplegic_refraction_recorded', 'encounter', 'Cycloplegic refraction performed'),
('encounter.keratometry_recorded', 'encounter', 'Keratometry measurements taken'),
('encounter.iop_measured', 'encounter', 'Intraocular pressure measured'),
('encounter.pachymetry_recorded', 'encounter', 'Corneal thickness measured'),
('encounter.pupils_examined', 'encounter', 'Pupil evaluation completed'),
('encounter.slit_lamp_examined', 'encounter', 'Anterior segment examination'),
('encounter.fundus_examined', 'encounter', 'Posterior segment examination'),
('encounter.oct_resulted', 'encounter', 'OCT scan results received'),
('encounter.visual_field_resulted', 'encounter', 'Visual field test results received'),
('encounter.fundus_photo_captured', 'encounter', 'Fundus photograph taken'),
('encounter.topography_resulted', 'encounter', 'Corneal topography results'),
('encounter.cl_fitting_performed', 'encounter', 'Contact lens fitting evaluation'),
('encounter.cl_trial_dispensed', 'encounter', 'Trial contact lenses dispensed'),
('encounter.over_refraction_recorded', 'encounter', 'Over-refraction with CL performed'),
('encounter.diagnosis_added', 'encounter', 'Diagnosis recorded with ICD-10'),
('encounter.diagnosis_removed', 'encounter', 'Diagnosis removed from encounter'),
('encounter.assessment_documented', 'encounter', 'Clinical assessment written'),
('encounter.plan_documented', 'encounter', 'Treatment plan written'),
('encounter.note_amended', 'encounter', 'Clinical note amended with reason'),
('encounter.signed', 'encounter', 'Encounter signed by provider'),
('encounter.locked', 'encounter', 'Encounter locked — no further edits'),
('encounter.addendum_added', 'encounter', 'Addendum added to locked encounter'),

-- Prescription stream
('prescription.written', 'prescription', 'Vision prescription created'),
('prescription.eyeglasses_prescribed', 'prescription', 'Eyeglasses Rx with sphere/cyl/axis/add'),
('prescription.cl_prescribed', 'prescription', 'Contact lens Rx with brand/BC/diameter'),
('prescription.renewed', 'prescription', 'Existing prescription renewed'),
('prescription.expired', 'prescription', 'Prescription passed expiration date'),
('prescription.voided', 'prescription', 'Prescription voided with reason'),

-- E-prescribing stream
('erx.created', 'erx', 'Electronic prescription created'),
('erx.sent', 'erx', 'Prescription transmitted via Surescripts'),
('erx.acknowledged', 'erx', 'Pharmacy acknowledged receipt'),
('erx.dispensed', 'erx', 'Pharmacy dispensed medication'),
('erx.cancelled', 'erx', 'Prescription cancelled'),
('erx.denied', 'erx', 'Pharmacy denied fill'),

-- Dispensing stream
('dispensing.order_created', 'dispensing', 'Dispensing order placed'),
('dispensing.frame_selected', 'dispensing', 'Frame selected for order'),
('dispensing.lens_configured', 'dispensing', 'Lens type, material, coatings selected'),
('dispensing.sent_to_lab', 'dispensing', 'Order transmitted to optical lab'),
('dispensing.lab_acknowledged', 'dispensing', 'Lab confirmed receipt'),
('dispensing.received_from_lab', 'dispensing', 'Finished product received from lab'),
('dispensing.patient_notified', 'dispensing', 'Patient notified glasses ready'),
('dispensing.dispensed', 'dispensing', 'Product dispensed to patient'),
('dispensing.adjusted', 'dispensing', 'Frame adjustment performed'),
('dispensing.returned', 'dispensing', 'Product returned by patient'),
('dispensing.warranty_claimed', 'dispensing', 'Warranty claim filed'),

-- Claim stream
('claim.drafted', 'claim', 'Insurance claim drafted'),
('claim.line_added', 'claim', 'Service line added to claim'),
('claim.line_modified', 'claim', 'Service line modified'),
('claim.submitted', 'claim', 'Claim submitted to payer via EDI 837P'),
('claim.accepted', 'claim', 'Payer accepted claim for processing'),
('claim.rejected', 'claim', 'Claim rejected — EDI format error'),
('claim.adjudicated', 'claim', 'Payer adjudicated claim'),
('claim.paid', 'claim', 'Payment received from payer'),
('claim.partially_paid', 'claim', 'Partial payment received'),
('claim.denied', 'claim', 'Claim denied with reason'),
('claim.appealed', 'claim', 'Appeal submitted for denied claim'),
('claim.appeal_decided', 'claim', 'Appeal decision received'),
('claim.voided', 'claim', 'Claim voided'),
('claim.era_received', 'claim', 'ERA 835 remittance received and parsed'),

-- Payment stream
('payment.collected', 'payment', 'Patient payment collected'),
('payment.refunded', 'payment', 'Refund issued to patient'),
('payment.insurance_posted', 'payment', 'Insurance payment posted from ERA'),

-- Inventory stream
('inventory.frame_received', 'inventory', 'Frame received into inventory'),
('inventory.frame_sold', 'inventory', 'Frame sold to patient'),
('inventory.frame_transferred', 'inventory', 'Frame transferred between locations'),
('inventory.frame_returned', 'inventory', 'Frame returned to vendor'),
('inventory.count_adjusted', 'inventory', 'Physical count adjustment'),
('inventory.cl_stock_received', 'inventory', 'Contact lens inventory received'),
('inventory.cl_dispensed', 'inventory', 'Contact lenses dispensed to patient'),

-- Eligibility stream
('eligibility.checked', 'eligibility', 'Insurance eligibility verified via 270/271'),
('eligibility.active', 'eligibility', 'Patient eligible — benefits confirmed'),
('eligibility.inactive', 'eligibility', 'Patient not eligible — coverage issue'),
('eligibility.prior_auth_requested', 'eligibility', 'Prior authorisation submitted'),
('eligibility.prior_auth_approved', 'eligibility', 'Prior authorisation approved'),
('eligibility.prior_auth_denied', 'eligibility', 'Prior authorisation denied'),

-- Access stream (HIPAA)
('access.patient_chart_viewed', 'access', 'User viewed patient chart'),
('access.phi_exported', 'access', 'PHI data exported'),
('access.phi_printed', 'access', 'PHI data printed'),
('access.break_glass', 'access', 'Emergency access to restricted record');
```

---

## Stream Snapshots

```sql
CREATE TABLE stream_snapshots (
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    snapshot_data JSONB NOT NULL,
    last_sequence_number BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id)
);
```

---

## Projection Checkpoints

```sql
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_event_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Read Model: Patients

```sql
CREATE TABLE rm_patients (
    id UUID PRIMARY KEY,
    practice_id UUID NOT NULL,
    mrn TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    date_of_birth DATE NOT NULL,
    sex TEXT,
    email TEXT,
    phone TEXT,
    address JSONB,
    clinical_history JSONB NOT NULL DEFAULT '{}',
    active_insurance JSONB NOT NULL DEFAULT '[]',
    last_exam_date DATE,
    last_rx_expiry DATE,
    recall_due DATE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (practice_id, mrn)
);

CREATE INDEX idx_rm_patients_practice ON rm_patients(practice_id);
CREATE INDEX idx_rm_patients_name ON rm_patients(practice_id, last_name, first_name);
CREATE INDEX idx_rm_patients_recall ON rm_patients(practice_id, recall_due)
    WHERE recall_due IS NOT NULL AND is_active = TRUE;
```

---

## Read Model: Encounters

```sql
CREATE TABLE rm_encounters (
    id UUID PRIMARY KEY,
    appointment_id UUID,
    patient_id UUID NOT NULL,
    provider_id UUID NOT NULL,
    location_id UUID NOT NULL,
    encounter_type TEXT NOT NULL,
    status TEXT NOT NULL,
    chief_complaint TEXT,
    exam_summary JSONB NOT NULL DEFAULT '{}',
    diagnoses JSONB NOT NULL DEFAULT '[]',
    prescriptions JSONB NOT NULL DEFAULT '[]',
    device_results JSONB NOT NULL DEFAULT '[]',
    e_prescriptions JSONB NOT NULL DEFAULT '[]',
    amendments JSONB NOT NULL DEFAULT '[]',
    signed_by UUID,
    signed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_enc_patient ON rm_encounters(patient_id, created_at DESC);
CREATE INDEX idx_rm_enc_provider ON rm_encounters(provider_id, created_at DESC);
CREATE INDEX idx_rm_enc_diagnoses ON rm_encounters USING GIN (diagnoses jsonb_path_ops);
```

---

## Read Model: Dispensing Pipeline

```sql
CREATE TABLE rm_dispensing_pipeline (
    id UUID PRIMARY KEY,
    encounter_id UUID NOT NULL,
    patient_id UUID NOT NULL,
    patient_name TEXT NOT NULL,
    location_id UUID NOT NULL,
    order_type TEXT NOT NULL,
    status TEXT NOT NULL,
    prescription_summary JSONB NOT NULL DEFAULT '{}',
    frame_summary JSONB,
    lens_summary JSONB,
    lab_name TEXT,
    lab_order_number TEXT,
    total_cents BIGINT NOT NULL DEFAULT 0,
    ordered_at TIMESTAMPTZ NOT NULL,
    dispensed_at TIMESTAMPTZ,
    events JSONB NOT NULL DEFAULT '[]',
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_disp_location ON rm_dispensing_pipeline(location_id, status)
    WHERE status NOT IN ('dispensed', 'cancelled');
CREATE INDEX idx_rm_disp_patient ON rm_dispensing_pipeline(patient_id, ordered_at DESC);
```

---

## Read Model: Claims Lifecycle

```sql
CREATE TABLE rm_claims_lifecycle (
    id UUID PRIMARY KEY,
    encounter_id UUID NOT NULL,
    patient_id UUID NOT NULL,
    patient_name TEXT NOT NULL,
    location_id UUID NOT NULL,
    claim_type TEXT NOT NULL,
    claim_number TEXT,
    payer_name TEXT NOT NULL,
    status TEXT NOT NULL,
    service_lines JSONB NOT NULL DEFAULT '[]',
    total_charge_cents BIGINT NOT NULL DEFAULT 0,
    allowed_cents BIGINT,
    paid_cents BIGINT,
    patient_responsibility_cents BIGINT,
    denial JSONB,
    timeline JSONB NOT NULL DEFAULT '[]',
    -- timeline example:
    -- [
    --   {"event": "claim.drafted", "at": "2026-05-25T15:00:00Z", "by": "user-uuid"},
    --   {"event": "claim.submitted", "at": "2026-05-25T16:30:00Z", "by": "user-uuid", "edi_ref": "CLM-2026-001"},
    --   {"event": "claim.denied", "at": "2026-06-01T10:00:00Z", "reason": "Service not covered", "code": "CO-96"},
    --   {"event": "claim.appealed", "at": "2026-06-05T09:00:00Z", "by": "user-uuid", "notes": "Added medical necessity documentation"},
    --   {"event": "claim.appeal_decided", "at": "2026-06-20T14:00:00Z", "result": "overturned", "paid_cents": 12000}
    -- ]
    submitted_at TIMESTAMPTZ,
    adjudicated_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_claims_patient ON rm_claims_lifecycle(patient_id, created_at DESC);
CREATE INDEX idx_rm_claims_status ON rm_claims_lifecycle(status)
    WHERE status NOT IN ('paid', 'voided');
CREATE INDEX idx_rm_claims_payer ON rm_claims_lifecycle(payer_name, status);
```

---

## Read Model: Clinical Trends

```sql
CREATE TABLE rm_clinical_trends (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id UUID NOT NULL,
    metric_type TEXT NOT NULL CHECK (metric_type IN (
        'iop', 'pachymetry', 'refraction', 'visual_acuity',
        'rnfl_thickness', 'cup_disc_ratio', 'visual_field_md'
    )),
    measurements JSONB NOT NULL DEFAULT '[]',
    -- measurements example for IOP:
    -- [
    --   {"date": "2024-05-20", "od": 16, "os": 15, "method": "goldmann"},
    --   {"date": "2025-05-18", "od": 17, "os": 16, "method": "goldmann"},
    --   {"date": "2026-05-25", "od": 18, "os": 17, "method": "goldmann"}
    -- ]
    --
    -- measurements example for refraction:
    -- [
    --   {"date": "2024-05-20", "od": {"sph": -1.75, "cyl": -0.50, "axis": 180}, "os": {"sph": -1.50, "cyl": -0.25, "axis": 175}},
    --   {"date": "2026-05-25", "od": {"sph": -2.00, "cyl": -0.75, "axis": 178}, "os": {"sph": -1.50, "cyl": -0.50, "axis": 170}}
    -- ]
    --
    -- measurements example for RNFL:
    -- [
    --   {"date": "2025-05-18", "od": {"avg": 95, "sup": 110, "inf": 105, "temp": 70, "nasal": 80}, "os": {"avg": 92, "sup": 108, "inf": 100, "temp": 68, "nasal": 78}},
    --   {"date": "2026-05-25", "od": {"avg": 94, "sup": 108, "inf": 103, "temp": 70, "nasal": 79}, "os": {"avg": 91, "sup": 107, "inf": 99, "temp": 67, "nasal": 77}}
    -- ]
    alert_threshold JSONB,
    last_value JSONB,
    trend_direction TEXT CHECK (trend_direction IN ('stable', 'improving', 'worsening', 'insufficient_data')),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (patient_id, metric_type)
);

CREATE INDEX idx_rm_trends_patient ON rm_clinical_trends(patient_id);
CREATE INDEX idx_rm_trends_alert ON rm_clinical_trends(metric_type, trend_direction)
    WHERE trend_direction = 'worsening';
```

---

## Read Model: Frame Inventory

```sql
CREATE TABLE rm_frame_inventory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL,
    location_id UUID NOT NULL,
    brand TEXT NOT NULL,
    model TEXT NOT NULL,
    color TEXT,
    size TEXT,
    upc TEXT,
    wholesale_cents BIGINT,
    retail_cents BIGINT NOT NULL,
    quantity INT NOT NULL DEFAULT 0,
    specs JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_inv_practice ON rm_frame_inventory(practice_id, location_id);
CREATE INDEX idx_rm_inv_brand ON rm_frame_inventory(practice_id, brand, model);
```

---

## Read Model: AI Analyses

```sql
CREATE TABLE rm_ai_analyses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID,
    patient_id UUID,
    analysis_type TEXT NOT NULL CHECK (analysis_type IN (
        'soap_draft', 'coding_suggestion', 'denial_risk',
        'frame_recommendation', 'wellness_prediction', 'discharge_draft'
    )),
    content TEXT NOT NULL,
    score REAL,
    details JSONB,
    model_version TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_ai_encounter ON rm_ai_analyses(encounter_id);
```

---

## Example Queries

### Replay encounter from events

```sql
SELECT event_type, event_data, created_by, created_at
FROM event_store
WHERE stream_type = 'encounter'
  AND stream_id = 'encounter-uuid'
ORDER BY sequence_number;
```

### Full clinical timeline for patient

```sql
SELECT es.event_type, es.event_data, es.created_at, es.created_by
FROM event_store es
WHERE es.stream_type IN ('encounter', 'prescription', 'dispensing')
  AND es.stream_id IN (
      SELECT id FROM rm_encounters WHERE patient_id = 'patient-uuid'
      UNION
      SELECT (jsonb_array_elements(prescriptions)->>'id')::UUID
      FROM rm_encounters WHERE patient_id = 'patient-uuid'
  )
ORDER BY es.created_at;
```

### IOP trending from read model

```sql
SELECT metric_type, measurements, trend_direction
FROM rm_clinical_trends
WHERE patient_id = 'patient-uuid'
  AND metric_type = 'iop';
```

### Claim denial rate by payer from read model

```sql
SELECT payer_name,
       COUNT(*) AS total,
       COUNT(*) FILTER (WHERE status = 'denied') AS denied,
       ROUND(COUNT(*) FILTER (WHERE status = 'denied') * 100.0 / COUNT(*), 1) AS denial_pct
FROM rm_claims_lifecycle
WHERE location_id = 'location-uuid'
  AND created_at >= CURRENT_DATE - 90
GROUP BY payer_name
ORDER BY denial_pct DESC;
```

### Amendment history for encounter

```sql
SELECT event_data->>'original_text' AS original,
       event_data->>'amended_text' AS amended,
       event_data->>'reason' AS reason,
       created_by, created_at
FROM event_store
WHERE stream_type = 'encounter'
  AND stream_id = 'encounter-uuid'
  AND event_type = 'encounter.note_amended'
ORDER BY created_at;
```

### HIPAA access audit

```sql
SELECT event_data->>'user_name' AS who,
       event_type,
       event_data->>'patient_name' AS patient,
       event_data->>'ip_address' AS ip,
       created_at
FROM event_store
WHERE stream_type = 'access'
  AND created_at >= CURRENT_DATE - 30
ORDER BY created_at DESC;
```

---

## Event-Driven Automation Patterns

### Auto-Charge Capture

When `encounter.diagnosis_added` or `encounter.signed` events fire, the charge capture projection:
1. Maps diagnosis + procedure codes to fee schedule
2. Emits `claim.drafted` with pre-populated service lines
3. Updates `rm_claims_lifecycle` in draft status

### Recall Reminders

When `encounter.signed` fires, the recall projection:
1. Reads the encounter plan for return interval
2. Calculates `recall_due` date
3. Updates `rm_patients.recall_due`
4. Schedules recall notification event

### Eligibility Auto-Check

When `appointment.scheduled` fires, the eligibility projection:
1. Looks up patient's active insurance from `rm_patients`
2. Triggers X12 270 eligibility inquiry
3. Stores result as `eligibility.active` or `eligibility.inactive` event
4. Alerts front desk if coverage is inactive

### Inventory Auto-Adjust

When `dispensing.dispensed` fires, the inventory projection:
1. Decrements `rm_frame_inventory.quantity` for the dispensed frame
2. Emits `inventory.frame_sold` event
3. Checks reorder threshold and flags low stock

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), event_types, stream_snapshots |
| Projection Infrastructure | 1 | projection_checkpoints |
| Read Model: Patients | 1 | rm_patients (demographics, insurance, recall) |
| Read Model: Encounters | 1 | rm_encounters (exam data, diagnoses, prescriptions, amendments) |
| Read Model: Dispensing | 1 | rm_dispensing_pipeline (order lifecycle) |
| Read Model: Claims | 1 | rm_claims_lifecycle (full claim timeline) |
| Read Model: Clinical Trends | 1 | rm_clinical_trends (IOP, refraction, RNFL trending) |
| Read Model: Inventory | 1 | rm_frame_inventory (per-location stock) |
| Read Model: AI | 1 | rm_ai_analyses (AI outputs) |
| **Total** | **12** | 4 infrastructure + 8 read models |

---

## Key Design Decisions

1. **Granular clinical events** — Each step of the exam (visual acuity, autorefraction, manifest refraction, IOP, slit lamp, fundus) is a separate event type rather than a single "exam completed" event. This enables real-time encounter building (the optician sees test results appear as the tech records them) and precise audit trails (which measurement was recorded at what time by whom).

2. **Amendment events preserve originals** — `encounter.note_amended` events carry both `original_text` and `amended_text` with a mandatory `reason`. The original note is never overwritten in the event store. The `rm_encounters` read model shows the current version with an `amendments` array for history. This satisfies HIPAA and malpractice documentation requirements.

3. **Claim lifecycle as event timeline** — Every claim status transition is an event with context (denial reason, appeal notes, ERA reference). The `rm_claims_lifecycle` read model includes a `timeline` JSONB array showing the complete history. This enables denial pattern analysis and appeal success tracking.

4. **Clinical trends as a dedicated read model** — `rm_clinical_trends` pre-computes longitudinal data for IOP, RNFL thickness, refraction, and visual field MD. Each patient × metric combination stores a time-series array of measurements. This avoids expensive event replays for the glaucoma monitoring and myopia progression use cases.

5. **Diagnostic device results as encounter events** — OCT, visual field, autorefractor, and topography results arrive as events on the encounter stream. Each event carries device-specific data plus DICOM/HL7 references. The projection merges them into `rm_encounters.device_results`. Raw device messages (HL7 ORU segments) are preserved in event metadata.

6. **HIPAA access events as a separate stream** — Chart views, PHI exports, prints, and break-glass access are events on the `access` stream type. These are never projected into clinical read models — they're queried directly from the event store for compliance audits. The immutable event store guarantees the audit trail cannot be tampered with.

7. **Prescription streams with legal lifecycle** — Prescriptions have their own stream tracking creation, renewal, expiration, and voiding. The `prescription.expired` event is system-generated based on the expiration date. This ensures the dispensary cannot fill an expired prescription without an explicit renewal event.

8. **Event-driven automation patterns** — Auto-charge capture, eligibility checks, recall scheduling, and inventory adjustments are implemented as event projections rather than application-layer triggers. This means the automation is auditable (you can see which event triggered which action) and recoverable (replay events to fix a missed projection).
