# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Optometry Practice Platform · Created: 2026-05-25

## Philosophy

An optometry practice has a highly structured clinical pipeline — scheduling, exams, prescriptions, dispensing, billing — but the data captured at each stage varies by exam type, diagnostic instrument, lens configuration, and payer. A comprehensive eye exam captures refraction, ocular health, and contact lens fitting data. A medical follow-up for glaucoma captures IOP trending and visual field analysis. A contact lens recheck captures over-refraction and wear schedule compliance. Rather than creating separate tables for each instrument, exam type, and lens configuration, this model keeps the pipeline relational (patients, encounters, prescriptions, claims) while storing variable clinical and product data in JSONB columns.

The JSONB approach is particularly well-suited to optometry because diagnostic instruments evolve rapidly: an OCT device upgrade may add new measurement types, a wavefront aberrometer produces entirely different data structures than a keratometer, and payer-specific claim attachments vary by carrier. JSONB columns absorb this variation without schema migrations. Core query patterns — patient lookup, appointment scheduling, prescription history, claim status — operate on relational columns with standard indexes, while clinical detail queries use GIN-indexed JSONB containment operators.

HIPAA audit logging and insurance claim headers remain relational because they require strict integrity for compliance reporting. Patient demographics and the appointment → encounter → prescription → dispensing → claim pipeline stay in typed columns. Only the variable interior of each stage — exam findings, device results, lens configurations, payer-specific fields — moves into JSONB.

**Best for:** Teams prioritising rapid development and instrument adaptability, where the optical dispensary handles diverse lens types, diagnostic devices evolve frequently, and the team wants fewer tables without sacrificing the clinical pipeline's referential integrity.

**Trade-offs:**
- **Pro:** 9 core tables vs. 22 in normalized — dramatically simpler schema
- **Pro:** New diagnostic device types require no schema migration
- **Pro:** Exam templates (comprehensive, CL fitting, medical, pediatric) as JSONB blueprints
- **Pro:** Dispensing lens configurations (progressive, bifocal, specialty CL) as JSONB — infinite variety
- **Pro:** Payer-specific claim fields absorbed into JSONB without per-payer columns
- **Con:** JSONB clinical data cannot use database-level type constraints (sphere must be REAL)
- **Con:** Refraction trending queries require JSONB path extraction instead of column comparison
- **Con:** Application layer must validate JSONB structure against exam templates
- **Con:** Reporting across JSONB fields is slower than columnar aggregation

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Patient and encounter columns map to FHIR resources; JSONB exam data exported as FHIR Observation bundles |
| FHIR VisionPrescription | Prescription JSONB structure mirrors VisionPrescription lensSpecification |
| HL7 v2.x ORU | Device results stored as JSONB — raw HL7 segments preserved in `raw_message` field |
| DICOM | OCT/visual field JSONB includes DICOM Study UID and Series UID references |
| ANSI Z80.1/Z80.5/Z80.20 | Lens spec and CL parameter fields within dispensing JSONB |
| NCPDP SCRIPT v2023011 | E-prescribing fields within encounter JSONB prescriptions section |
| ANSI X12 837P/835 | Claim JSONB includes EDI-specific fields (payer ID, subscriber, service lines) |
| ICD-10-CM / CPT | Coded as typed fields on encounters and claims for indexing |
| HIPAA | Audit log kept relational; JSONB PHI fields encrypted at application layer |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

---

## Practice & Providers

```sql
CREATE TABLE practices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    tax_id TEXT,
    npi TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "locations": [
    --     {"id": "loc-uuid", "name": "Main Office", "address": {...}, "phone": "...", "fax": "...", "pos_code": "11"}
    --   ],
    --   "exam_templates": {
    --     "comprehensive": {"sections": ["chief_complaint","visual_acuity","refraction","ocular_health","assessment","plan"]},
    --     "contact_lens": {"sections": ["chief_complaint","over_refraction","cl_fitting","wear_schedule","assessment"]},
    --     "medical": {"sections": ["chief_complaint","iop","visual_field","oct","assessment","plan"]}
    --   },
    --   "fee_schedule": [
    --     {"cpt": "92004", "description": "Comprehensive new patient", "charge_cents": 25000},
    --     {"cpt": "92014", "description": "Comprehensive established", "charge_cents": 18000}
    --   ],
    --   "insurance_plans": [
    --     {"id": "plan-uuid", "payer_name": "VSP", "payer_id": "12345", "type": "vision", "edi_payer_id": "..."}
    --   ]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE providers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL REFERENCES practices(id) ON DELETE CASCADE,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    role TEXT NOT NULL CHECK (role IN (
        'optometrist', 'ophthalmologist', 'optician', 'technician',
        'receptionist', 'billing', 'admin'
    )),
    credentials JSONB NOT NULL DEFAULT '{}',
    -- credentials example:
    -- {
    --   "npi": "1234567890",
    --   "license_number": "OD12345",
    --   "license_state": "CA",
    --   "dea_number": "FA1234567",
    --   "surescripts_spi": "...",
    --   "location_ids": ["loc-uuid-1", "loc-uuid-2"],
    --   "specialties": ["pediatric", "contact_lens"]
    -- }
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_providers_practice ON providers(practice_id);
```

---

## Patients

```sql
CREATE TABLE patients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL REFERENCES practices(id) ON DELETE CASCADE,
    mrn TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    date_of_birth DATE NOT NULL,
    sex TEXT CHECK (sex IN ('male', 'female', 'other', 'unknown')),
    contact JSONB NOT NULL DEFAULT '{}',
    -- contact example:
    -- {
    --   "email": "jane@example.com",
    --   "phone": "555-0100",
    --   "phone_secondary": "555-0101",
    --   "address": {"line1": "123 Main", "city": "Portland", "state": "OR", "postal": "97201", "country": "US"},
    --   "preferred_contact": "phone",
    --   "preferred_language": "en",
    --   "emergency": {"name": "John Doe", "phone": "555-0102"}
    -- }
    clinical_history JSONB NOT NULL DEFAULT '{}',
    -- clinical_history example:
    -- {
    --   "allergies": ["sulfa", "latex"],
    --   "medications": ["Latanoprost 0.005%", "Restasis"],
    --   "medical_history": ["diabetes_type_2", "hypertension"],
    --   "ocular_history": ["myopia", "dry_eye", "previous_lasik_2019"],
    --   "family_ocular_history": ["glaucoma_mother", "macular_degeneration_father"]
    -- }
    insurance JSONB NOT NULL DEFAULT '[]',
    -- insurance example:
    -- [
    --   {
    --     "plan_id": "plan-uuid", "priority": "primary", "subscriber_id": "VSP123",
    --     "subscriber_name": "Jane Doe", "group_number": "GRP456",
    --     "effective_date": "2026-01-01", "copay_cents": 1000
    --   },
    --   {
    --     "plan_id": "plan-uuid-2", "priority": "secondary", "subscriber_id": "BCBS789",
    --     "group_number": "MED012", "effective_date": "2026-01-01"
    --   }
    -- ]
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (practice_id, mrn)
);

CREATE INDEX idx_patients_practice ON patients(practice_id);
CREATE INDEX idx_patients_name ON patients(practice_id, last_name, first_name);
CREATE INDEX idx_patients_dob ON patients(practice_id, date_of_birth);
CREATE INDEX idx_patients_insurance ON patients USING GIN (insurance jsonb_path_ops);
```

---

## Appointments

```sql
CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL REFERENCES practices(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    provider_id UUID NOT NULL REFERENCES providers(id),
    location_id UUID NOT NULL,
    appointment_type TEXT NOT NULL DEFAULT 'comprehensive' CHECK (appointment_type IN (
        'comprehensive', 'follow_up', 'contact_lens', 'medical',
        'pediatric', 'emergency', 'dry_eye', 'pre_op', 'post_op'
    )),
    status TEXT NOT NULL DEFAULT 'scheduled' CHECK (status IN (
        'scheduled', 'confirmed', 'checked_in', 'in_exam',
        'in_dispensary', 'completed', 'cancelled', 'no_show'
    )),
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end TIMESTAMPTZ NOT NULL,
    actual_start TIMESTAMPTZ,
    actual_end TIMESTAMPTZ,
    reason TEXT,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appts_location ON appointments(location_id, scheduled_start);
CREATE INDEX idx_appts_provider ON appointments(provider_id, scheduled_start);
CREATE INDEX idx_appts_patient ON appointments(patient_id, scheduled_start DESC);
```

---

## Encounters

```sql
CREATE TABLE encounters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    appointment_id UUID REFERENCES appointments(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    provider_id UUID NOT NULL REFERENCES providers(id),
    location_id UUID NOT NULL,
    encounter_type TEXT NOT NULL DEFAULT 'comprehensive',
    status TEXT NOT NULL DEFAULT 'in_progress' CHECK (status IN (
        'in_progress', 'completed', 'addendum', 'locked'
    )),
    chief_complaint TEXT,
    exam_data JSONB NOT NULL DEFAULT '{}',
    -- exam_data for comprehensive exam:
    -- {
    --   "visual_acuity": {"od_sc_dist": "20/40", "od_cc_dist": "20/20", "os_sc_dist": "20/30", "os_cc_dist": "20/20"},
    --   "autorefraction": {"od": {"sphere": -2.25, "cylinder": -0.75, "axis": 180}, "os": {"sphere": -1.75, "cylinder": -0.50, "axis": 175}},
    --   "manifest_refraction": {"od": {"sphere": -2.00, "cylinder": -0.75, "axis": 178, "va": "20/20"}, "os": {"sphere": -1.50, "cylinder": -0.50, "axis": 170, "va": "20/20"}},
    --   "keratometry": {"od": {"k1": 43.25, "k1_axis": 180, "k2": 44.00, "k2_axis": 90}, "os": {"k1": 43.50, "k1_axis": 175, "k2": 44.25, "k2_axis": 85}},
    --   "iop": {"od": 16, "os": 15, "method": "goldmann", "time": "10:30"},
    --   "pachymetry": {"od": 545, "os": 550},
    --   "pupils": {"od": {"size_mm": 4, "rapd": false}, "os": {"size_mm": 4, "rapd": false}},
    --   "slit_lamp": {"lids": "normal", "conjunctiva": "clear", "cornea": "clear", "anterior_chamber": "deep_quiet", "iris": "flat_intact", "lens": "clear"},
    --   "fundus": {"od": {"disc": "pink_flat_distinct", "cd_ratio": 0.3, "macula": "flat_foveal_reflex", "vessels": "normal", "periphery": "normal"}, "os": {"disc": "pink_flat_distinct", "cd_ratio": 0.3, "macula": "flat_foveal_reflex", "vessels": "normal", "periphery": "normal"}},
    --   "assessment": "Myopia, stable. No glaucomatous changes.",
    --   "plan": "Continue current correction. RTC 1 year."
    -- }
    --
    -- exam_data for contact lens fitting:
    -- {
    --   "over_refraction": {"od": {"sphere": 0.00, "va": "20/20"}, "os": {"sphere": -0.25, "va": "20/20"}},
    --   "cl_fitting": {"od": {"brand": "Acuvue Oasys", "bc": 8.4, "diameter": 14.0, "power": -2.25, "centration": "good", "movement": "adequate", "coverage": "full"}, "os": {"brand": "Acuvue Oasys", "bc": 8.4, "diameter": 14.0, "power": -1.75, "centration": "good", "movement": "adequate", "coverage": "full"}},
    --   "wear_schedule": {"modality": "daily_wear", "replacement": "biweekly", "hours_per_day": 12},
    --   "assessment": "Good fit, comfortable, adequate vision."
    -- }
    diagnoses JSONB NOT NULL DEFAULT '[]',
    -- diagnoses example:
    -- [
    --   {"icd10": "H52.13", "description": "Myastigmatism, bilateral", "eye": "ou", "primary": true},
    --   {"icd10": "H52.4", "description": "Presbyopia", "eye": "ou", "primary": false}
    -- ]
    prescriptions JSONB NOT NULL DEFAULT '[]',
    -- prescriptions example:
    -- [
    --   {
    --     "id": "rx-uuid", "type": "eyeglasses", "prescribed_at": "2026-05-25T14:00:00Z",
    --     "od": {"sphere": -2.00, "cylinder": -0.75, "axis": 178, "add": 1.50},
    --     "os": {"sphere": -1.50, "cylinder": -0.50, "axis": 170, "add": 1.50},
    --     "pd": {"distance": 63, "near": 60},
    --     "expiration_date": "2028-05-25"
    --   },
    --   {
    --     "id": "rx-uuid-2", "type": "contact_lenses", "prescribed_at": "2026-05-25T14:00:00Z",
    --     "od": {"sphere": -2.25, "bc": 8.4, "diameter": 14.0, "brand": "Acuvue Oasys"},
    --     "os": {"sphere": -1.75, "bc": 8.4, "diameter": 14.0, "brand": "Acuvue Oasys"},
    --     "modality": "biweekly", "expiration_date": "2027-05-25"
    --   }
    -- ]
    device_results JSONB NOT NULL DEFAULT '[]',
    -- device_results example:
    -- [
    --   {"type": "oct", "device": "Zeiss Cirrus 6000", "performed_at": "2026-05-25T09:45:00Z",
    --    "od": {"rnfl_avg": 95, "rnfl_superior": 110, "rnfl_inferior": 105, "rnfl_temporal": 70, "rnfl_nasal": 80, "cd_ratio": 0.32},
    --    "os": {"rnfl_avg": 92, "rnfl_superior": 108, "rnfl_inferior": 100, "rnfl_temporal": 68, "rnfl_nasal": 78, "cd_ratio": 0.30},
    --    "dicom_study_uid": "1.2.840.113619.2.55.3..."},
    --   {"type": "visual_field", "device": "Humphrey HFA3", "performed_at": "2026-05-25T09:30:00Z",
    --    "od": {"md": -1.2, "psd": 1.5, "vfi": 98, "ght": "within_normal_limits"},
    --    "os": {"md": -0.8, "psd": 1.3, "vfi": 99, "ght": "within_normal_limits"},
    --    "dicom_study_uid": "1.2.840.113619.2.55.4..."},
    --   {"type": "autorefraction", "device": "Nidek ARK-1", "performed_at": "2026-05-25T09:20:00Z",
    --    "od": {"sphere": -2.25, "cylinder": -0.75, "axis": 180},
    --    "os": {"sphere": -1.75, "cylinder": -0.50, "axis": 175},
    --    "raw_message": "MSH|^~\\&|ARK1|NIDEK|EHR|..."}
    -- ]
    e_prescriptions JSONB NOT NULL DEFAULT '[]',
    -- e_prescriptions example:
    -- [
    --   {"medication": "Latanoprost 0.005% ophthalmic solution", "sig": "1 drop OU QHS",
    --    "quantity": "2.5 mL", "refills": 3, "pharmacy_ncpdp": "1234567",
    --    "pharmacy_name": "CVS #1234", "surescripts_id": "MSG-...", "status": "sent"}
    -- ]
    signed_by UUID REFERENCES providers(id),
    signed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_encounters_patient ON encounters(patient_id, created_at DESC);
CREATE INDEX idx_encounters_provider ON encounters(provider_id, created_at DESC);
CREATE INDEX idx_encounters_diagnoses ON encounters USING GIN (diagnoses jsonb_path_ops);
CREATE INDEX idx_encounters_devices ON encounters USING GIN (device_results jsonb_path_ops);
```

---

## Dispensing Orders

```sql
CREATE TABLE dispensing_orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID NOT NULL REFERENCES encounters(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    location_id UUID NOT NULL,
    order_type TEXT NOT NULL CHECK (order_type IN (
        'eyeglasses', 'contact_lenses', 'sunglasses'
    )),
    status TEXT NOT NULL DEFAULT 'ordered' CHECK (status IN (
        'ordered', 'in_lab', 'received', 'notified', 'dispensed',
        'cancelled', 'returned'
    )),
    prescription_id TEXT NOT NULL,
    specs JSONB NOT NULL DEFAULT '{}',
    -- specs for eyeglasses:
    -- {
    --   "frame": {"brand": "Ray-Ban", "model": "RB5154", "color": "Black Gold", "size": "51-21-145", "upc": "805289126591"},
    --   "lenses": {"type": "progressive", "material": "polycarbonate", "coatings": ["anti_reflective", "blue_light"], "tint": "none"},
    --   "measurements": {"seg_height": 18, "oc_height_od": 22, "oc_height_os": 22, "panto": 10, "wrap": 5},
    --   "lab": {"name": "Essilor Lab", "order_number": "LAB-2026-1234"}
    -- }
    --
    -- specs for contact lenses:
    -- {
    --   "od": {"brand": "Acuvue Oasys", "bc": 8.4, "diameter": 14.0, "power": -2.25, "boxes": 4},
    --   "os": {"brand": "Acuvue Oasys", "bc": 8.4, "diameter": 14.0, "power": -1.75, "boxes": 4},
    --   "modality": "biweekly", "annual_supply": true
    -- }
    total_cents BIGINT NOT NULL DEFAULT 0,
    warranty_expiry DATE,
    ordered_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    dispensed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dispensing_patient ON dispensing_orders(patient_id, created_at DESC);
CREATE INDEX idx_dispensing_status ON dispensing_orders(location_id, status)
    WHERE status NOT IN ('dispensed', 'cancelled');
```

---

## Insurance Claims

```sql
CREATE TABLE claims (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID NOT NULL REFERENCES encounters(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    location_id UUID NOT NULL,
    claim_type TEXT NOT NULL CHECK (claim_type IN ('vision', 'medical')),
    claim_number TEXT,
    status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'submitted', 'accepted', 'rejected', 'denied',
        'paid', 'partially_paid', 'appealed', 'voided'
    )),
    payer JSONB NOT NULL DEFAULT '{}',
    -- payer example:
    -- {
    --   "plan_id": "plan-uuid", "payer_name": "VSP", "edi_payer_id": "84146",
    --   "subscriber_id": "VSP123456", "subscriber_name": "Jane Doe",
    --   "group_number": "GRP789"
    -- }
    service_lines JSONB NOT NULL DEFAULT '[]',
    -- service_lines example:
    -- [
    --   {"line": 1, "cpt": "92014", "modifiers": ["25"], "icd10": ["H52.13","H52.4"],
    --    "description": "Comprehensive established", "units": 1, "charge_cents": 18000,
    --    "allowed_cents": 15000, "paid_cents": 12000},
    --   {"line": 2, "cpt": "92015", "modifiers": [], "icd10": ["H52.13"],
    --    "description": "Refraction", "units": 1, "charge_cents": 5000,
    --    "allowed_cents": null, "paid_cents": null}
    -- ]
    total_charge_cents BIGINT NOT NULL DEFAULT 0,
    allowed_cents BIGINT,
    paid_cents BIGINT,
    patient_responsibility_cents BIGINT,
    denial JSONB,
    -- denial example:
    -- {"reason": "Service not covered", "code": "CO-96", "remark_codes": ["N130"],
    --  "appeal_deadline": "2026-08-25"}
    submitted_at TIMESTAMPTZ,
    adjudicated_at TIMESTAMPTZ,
    era_835 JSONB,
    -- era_835: parsed ERA data for reconciliation
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claims_encounter ON claims(encounter_id);
CREATE INDEX idx_claims_patient ON claims(patient_id, created_at DESC);
CREATE INDEX idx_claims_status ON claims(status) WHERE status NOT IN ('paid', 'voided');
CREATE INDEX idx_claims_payer ON claims USING GIN (payer jsonb_path_ops);
```

---

## Frame Inventory

```sql
CREATE TABLE frame_inventory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL REFERENCES practices(id) ON DELETE CASCADE,
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
    -- specs example:
    -- {
    --   "bridge": 21, "temple": 145, "lens_width": 51, "lens_height": 38,
    --   "material": "acetate", "rim_type": "full_rim",
    --   "gender": "unisex", "age_group": "adult"
    -- }
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_frames_practice ON frame_inventory(practice_id, location_id);
CREATE INDEX idx_frames_upc ON frame_inventory(upc) WHERE upc IS NOT NULL;
CREATE INDEX idx_frames_brand ON frame_inventory(practice_id, brand, model);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL,
    user_id UUID NOT NULL,
    action TEXT NOT NULL,
    resource_type TEXT NOT NULL,
    resource_id UUID NOT NULL,
    details JSONB,
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_log_2026_h1 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-07-01');
CREATE TABLE audit_log_2026_h2 PARTITION OF audit_log
    FOR VALUES FROM ('2026-07-01') TO ('2027-01-01');

CREATE INDEX idx_audit_practice ON audit_log(practice_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Example Queries

### Full patient chart with latest exam data

```sql
SELECT p.first_name, p.last_name, p.date_of_birth,
       p.clinical_history,
       e.encounter_type, e.exam_data, e.diagnoses, e.prescriptions,
       e.device_results
FROM patients p
LEFT JOIN LATERAL (
    SELECT * FROM encounters
    WHERE patient_id = p.id AND status IN ('completed', 'locked')
    ORDER BY created_at DESC LIMIT 1
) e ON TRUE
WHERE p.id = 'patient-uuid';
```

### IOP trending from JSONB exam data

```sql
SELECT e.created_at::DATE AS visit_date,
       (e.exam_data->'iop'->>'od')::INT AS od_iop,
       (e.exam_data->'iop'->>'os')::INT AS os_iop,
       e.exam_data->'iop'->>'method' AS method
FROM encounters e
WHERE e.patient_id = 'patient-uuid'
  AND e.exam_data ? 'iop'
ORDER BY e.created_at;
```

### Find patients with specific diagnosis

```sql
SELECT DISTINCT p.id, p.first_name, p.last_name
FROM patients p
JOIN encounters e ON e.patient_id = p.id
WHERE p.practice_id = 'practice-uuid'
  AND e.diagnoses @> '[{"icd10": "H40.11"}]';
```

### Claim denial rate by payer

```sql
SELECT c.payer->>'payer_name' AS payer_name,
       COUNT(*) AS total_claims,
       COUNT(*) FILTER (WHERE c.status = 'denied') AS denied,
       ROUND(COUNT(*) FILTER (WHERE c.status = 'denied') * 100.0 / COUNT(*), 1) AS denial_pct
FROM claims c
WHERE c.location_id = 'location-uuid'
  AND c.created_at >= CURRENT_DATE - 90
GROUP BY c.payer->>'payer_name'
ORDER BY denial_pct DESC;
```

### Frame inventory search

```sql
SELECT brand, model, color, retail_cents, quantity
FROM frame_inventory
WHERE practice_id = 'practice-uuid'
  AND location_id = 'location-uuid'
  AND is_active = TRUE
  AND brand ILIKE '%ray%ban%'
ORDER BY model;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Practice & Providers | 2 | practices (locations + plans + templates in JSONB), providers (credentials in JSONB) |
| Patients | 1 | patients (contact, clinical history, insurance in JSONB) |
| Scheduling | 1 | appointments |
| Encounters | 1 | encounters (exam data, diagnoses, prescriptions, device results, e-rx all in JSONB) |
| Dispensary | 2 | dispensing_orders (specs in JSONB), frame_inventory (specs in JSONB) |
| Claims | 1 | claims (payer, service lines, denial, ERA in JSONB) |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **9** | |

---

## Key Design Decisions

1. **Encounters as self-contained clinical documents** — `exam_data`, `diagnoses`, `prescriptions`, `device_results`, and `e_prescriptions` live as JSONB on the encounter. This mirrors how clinicians think: one visit, one document. The encounter is the unit of clinical work and the unit of FHIR export.

2. **Exam templates drive JSONB structure** — Practice-level `exam_templates` in `practices.settings` define which sections appear for each encounter type. The application renders the template; the JSONB stores whatever sections the template produced. New exam types (dry eye evaluation, myopia management) require only a template addition, not a schema migration.

3. **Device results as JSONB array** — Diagnostic instruments produce wildly different data: OCT has RNFL thickness and cup-disc ratios, visual fields have MD/PSD/VFI, autorefractors have sphere/cylinder/axis. Each result is a typed JSONB object in the `device_results` array. DICOM Study UIDs link to the image archive. Raw HL7 messages can be preserved for audit.

4. **Prescriptions inline on encounters** — Both eyeglasses and contact lens prescriptions live in the encounter's `prescriptions` JSONB array. Each prescription has a generated UUID for cross-referencing from dispensing orders. This eliminates the eyeglasses-vs-CL column nullability problem of the normalized model.

5. **Dispensing specs as JSONB** — Frame selection, lens type, coatings, measurements (seg height, OC height), and lab order details live in `specs`. This handles the combinatorial explosion of progressive/bifocal/single-vision × polycarbonate/trivex/hi-index × coating combinations without per-configuration columns.

6. **Insurance on patients, payer on claims** — Patient insurance coverage lives as a JSONB array on the patient record (coverage changes by swapping array elements). Each claim snapshots the payer details at submission time into its own `payer` JSONB, ensuring claim integrity even if the patient's coverage changes later.

7. **Claim service lines as JSONB** — CPT-coded line items with ICD-10 pointers, modifiers, and adjudication amounts live in `service_lines` JSONB. This avoids a join table for a structure that's always read and written as a unit with the claim header.

8. **Audit log kept relational** — Despite the JSONB-heavy design, the audit log remains a separate partitioned table with typed columns for `action`, `resource_type`, and `resource_id`. HIPAA compliance demands reliable, queryable audit trails that don't depend on JSONB structure consistency.
