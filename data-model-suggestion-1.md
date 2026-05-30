# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Optometry Practice Platform · Created: 2026-05-25

## Philosophy

An optometry practice platform manages a complex clinical and retail pipeline: practices have locations, locations have providers, patients come in for exams, exams produce vision prescriptions (eyeglasses and contact lenses), prescriptions drive dispensing orders from the optical shop, dispensing generates charges that flow into invoices, and insurance claims are submitted for reimbursement. Alongside the clinical pipeline, diagnostic instruments send structured results (autorefractor readings, OCT scans, visual field data) that must be stored with proper reference to the encounter. A normalized relational model gives each clinical, retail, and financial concept its own table.

This mirrors how an optometry practice actually operates: the front desk schedules appointments and verifies insurance eligibility, the technician pre-tests with instruments, the OD conducts the refraction and ocular health exam, prescriptions are written, the optician dispenses frames and lenses in the retail shop, and billing submits claims. Each step maps to a table. The FHIR R4 `VisionPrescription` resource maps directly to the prescription table's column structure, and ANSI Z80 lens tolerance parameters inform the dispensing order schema.

**Best for:** Teams building a HIPAA-compliant optometry platform where the exam → prescription → dispensing → billing pipeline needs strict relational integrity, where FHIR R4 interoperability is a certification requirement, and where insurance claim lifecycle tracking is essential.

**Trade-offs:**
- **Pro:** Database-enforced exam → prescription → dispensing → claim pipeline
- **Pro:** Vision prescriptions as typed columns align with FHIR VisionPrescription resource
- **Pro:** Insurance claims with line-level detail enable denial tracking and resubmission
- **Pro:** Diagnostic results as explicit rows enable result trending (IOP, visual acuity)
- **Pro:** Inventory with SKU-level tracking supports multi-location dispensary
- **Con:** 30 tables — high complexity
- **Con:** Exam findings vary by exam type but use shared columns
- **Con:** Diagnostic instrument data varies by device type
- **Con:** High join count for "full patient chart" view

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Patient, Encounter, VisionPrescription, Observation, Claim resources |
| HL7 FHIR US Core IG v6.1 | USCDI v3 data elements mapped to patient/encounter columns |
| HL7 v2.x ORU/MDM | Diagnostic device result messaging |
| DICOM | OCT, visual field, fundus image references |
| ANSI Z80.1/Z80.5/Z80.20 | Prescription tolerances, frame measurements, CL parameters |
| NCPDP SCRIPT v2023011 | E-prescribing message format |
| ANSI X12 837P/835/270/271 | Insurance claim submission, ERA processing, eligibility |
| ICD-10-CM | Diagnosis codes on encounters and claims |
| CPT/HCPCS | Procedure codes on claims |
| HIPAA Security Rule | Audit logging, encryption, access controls |
| HIPAA Privacy Rule | PHI handling, minimum necessary |
| ISO 8601 | All timestamps as TIMESTAMPTZ |
| ISO 4217 | Currency codes for billing |
| OAuth 2.0 / SMART on FHIR | API authentication |

---

## Practice, Locations & Providers

```sql
CREATE TABLE practices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    tax_id TEXT,
    npi TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL REFERENCES practices(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    address_line1 TEXT NOT NULL,
    address_line2 TEXT,
    city TEXT NOT NULL,
    state TEXT NOT NULL,
    postal_code TEXT NOT NULL,
    country TEXT NOT NULL DEFAULT 'US',
    phone TEXT,
    fax TEXT,
    place_of_service_code TEXT NOT NULL DEFAULT '11',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_locations_practice ON locations(practice_id);

CREATE TABLE providers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL REFERENCES practices(id) ON DELETE CASCADE,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    role TEXT NOT NULL CHECK (role IN (
        'optometrist', 'ophthalmologist', 'optician', 'technician',
        'receptionist', 'billing', 'admin'
    )),
    npi TEXT,
    license_number TEXT,
    license_state TEXT,
    dea_number TEXT,
    surescripts_spi TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_providers_practice ON providers(practice_id);

CREATE TABLE provider_locations (
    provider_id UUID NOT NULL REFERENCES providers(id) ON DELETE CASCADE,
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    PRIMARY KEY (provider_id, location_id)
);
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
    email TEXT,
    phone TEXT,
    phone_secondary TEXT,
    address_line1 TEXT,
    address_line2 TEXT,
    city TEXT,
    state TEXT,
    postal_code TEXT,
    country TEXT NOT NULL DEFAULT 'US',
    preferred_contact TEXT DEFAULT 'phone',
    preferred_language TEXT DEFAULT 'en',
    ssn_last4 TEXT,
    emergency_contact_name TEXT,
    emergency_contact_phone TEXT,
    allergies TEXT[],
    medications TEXT[],
    medical_history TEXT[],
    ocular_history TEXT[],
    family_ocular_history TEXT[],
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (practice_id, mrn)
);

CREATE INDEX idx_patients_practice ON patients(practice_id);
CREATE INDEX idx_patients_name ON patients(practice_id, last_name, first_name);
CREATE INDEX idx_patients_dob ON patients(practice_id, date_of_birth);
CREATE INDEX idx_patients_phone ON patients(phone);
```

---

## Insurance

```sql
CREATE TABLE insurance_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL REFERENCES practices(id) ON DELETE CASCADE,
    payer_name TEXT NOT NULL,
    payer_id TEXT NOT NULL,
    plan_type TEXT NOT NULL CHECK (plan_type IN ('vision', 'medical')),
    edi_payer_id TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE patient_insurances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id UUID NOT NULL REFERENCES patients(id) ON DELETE CASCADE,
    plan_id UUID NOT NULL REFERENCES insurance_plans(id),
    priority TEXT NOT NULL DEFAULT 'primary' CHECK (priority IN (
        'primary', 'secondary', 'tertiary'
    )),
    subscriber_id TEXT NOT NULL,
    subscriber_name TEXT,
    group_number TEXT,
    effective_date DATE,
    termination_date DATE,
    copay_cents BIGINT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patient_insurance ON patient_insurances(patient_id);
```

---

## Scheduling

```sql
CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    provider_id UUID NOT NULL REFERENCES providers(id),
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

## Encounters & Exam Data

```sql
CREATE TABLE encounters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    appointment_id UUID REFERENCES appointments(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    provider_id UUID NOT NULL REFERENCES providers(id),
    location_id UUID NOT NULL REFERENCES locations(id),
    encounter_type TEXT NOT NULL DEFAULT 'comprehensive',
    status TEXT NOT NULL DEFAULT 'in_progress' CHECK (status IN (
        'in_progress', 'completed', 'addendum', 'locked'
    )),
    chief_complaint TEXT,
    subjective TEXT,
    objective TEXT,
    assessment TEXT,
    plan TEXT,
    signed_by UUID REFERENCES providers(id),
    signed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_encounters_patient ON encounters(patient_id, created_at DESC);
CREATE INDEX idx_encounters_provider ON encounters(provider_id, created_at DESC);

CREATE TABLE refraction_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID NOT NULL REFERENCES encounters(id) ON DELETE CASCADE,
    refraction_type TEXT NOT NULL CHECK (refraction_type IN (
        'autorefraction', 'manifest', 'cycloplegic', 'over_refraction'
    )),
    od_sphere REAL,
    od_cylinder REAL,
    od_axis INT,
    od_add REAL,
    od_prism REAL,
    od_prism_base TEXT,
    od_va TEXT,
    os_sphere REAL,
    os_cylinder REAL,
    os_axis INT,
    os_add REAL,
    os_prism REAL,
    os_prism_base TEXT,
    os_va TEXT,
    pd_distance REAL,
    pd_near REAL,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_refraction ON refraction_data(encounter_id);

CREATE TABLE ocular_measurements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID NOT NULL REFERENCES encounters(id) ON DELETE CASCADE,
    measurement_type TEXT NOT NULL CHECK (measurement_type IN (
        'iop', 'pachymetry', 'keratometry', 'pupil_size',
        'anterior_chamber_depth', 'axial_length', 'cup_disc_ratio'
    )),
    od_value TEXT,
    os_value TEXT,
    unit TEXT,
    method TEXT,
    device TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_measurements ON ocular_measurements(encounter_id);

CREATE TABLE diagnoses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID NOT NULL REFERENCES encounters(id) ON DELETE CASCADE,
    icd10_code TEXT NOT NULL,
    description TEXT NOT NULL,
    eye TEXT CHECK (eye IN ('od', 'os', 'ou', 'bilateral', 'not_applicable')),
    is_primary BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_diagnoses_encounter ON diagnoses(encounter_id);
CREATE INDEX idx_diagnoses_icd ON diagnoses(icd10_code);
```

---

## Vision Prescriptions

```sql
CREATE TABLE vision_prescriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID NOT NULL REFERENCES encounters(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    prescriber_id UUID NOT NULL REFERENCES providers(id),
    prescription_type TEXT NOT NULL CHECK (prescription_type IN (
        'eyeglasses', 'contact_lenses'
    )),
    od_sphere REAL,
    od_cylinder REAL,
    od_axis INT,
    od_add REAL,
    od_prism REAL,
    od_prism_base TEXT,
    os_sphere REAL,
    os_cylinder REAL,
    os_axis INT,
    os_add REAL,
    os_prism REAL,
    os_prism_base TEXT,
    pd_distance REAL,
    pd_near REAL,
    -- Contact lens specific
    od_base_curve REAL,
    od_diameter REAL,
    od_brand TEXT,
    od_color TEXT,
    os_base_curve REAL,
    os_diameter REAL,
    os_brand TEXT,
    os_color TEXT,
    expiration_date DATE NOT NULL,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rx_patient ON vision_prescriptions(patient_id, created_at DESC);
CREATE INDEX idx_rx_encounter ON vision_prescriptions(encounter_id);
```

---

## Optical Dispensary

```sql
CREATE TABLE frames (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL REFERENCES practices(id) ON DELETE CASCADE,
    brand TEXT NOT NULL,
    model TEXT NOT NULL,
    color TEXT,
    size TEXT,
    upc TEXT,
    wholesale_cents BIGINT,
    retail_cents BIGINT NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_frames_practice ON frames(practice_id);
CREATE INDEX idx_frames_upc ON frames(upc) WHERE upc IS NOT NULL;

CREATE TABLE frame_inventory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    frame_id UUID NOT NULL REFERENCES frames(id) ON DELETE CASCADE,
    location_id UUID NOT NULL REFERENCES locations(id),
    quantity INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (frame_id, location_id)
);

CREATE TABLE dispensing_orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id UUID NOT NULL REFERENCES patients(id),
    prescription_id UUID NOT NULL REFERENCES vision_prescriptions(id),
    location_id UUID NOT NULL REFERENCES locations(id),
    order_type TEXT NOT NULL CHECK (order_type IN (
        'eyeglasses', 'contact_lenses', 'sunglasses'
    )),
    status TEXT NOT NULL DEFAULT 'ordered' CHECK (status IN (
        'ordered', 'in_lab', 'received', 'notified', 'dispensed',
        'cancelled', 'returned'
    )),
    frame_id UUID REFERENCES frames(id),
    lens_type TEXT,
    lens_material TEXT,
    lens_coating TEXT[],
    tint TEXT,
    lab_name TEXT,
    lab_order_number TEXT,
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
    insurance_id UUID NOT NULL REFERENCES patient_insurances(id),
    location_id UUID NOT NULL REFERENCES locations(id),
    claim_type TEXT NOT NULL CHECK (claim_type IN ('vision', 'medical')),
    claim_number TEXT,
    status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'submitted', 'accepted', 'rejected', 'denied',
        'paid', 'partially_paid', 'appealed', 'voided'
    )),
    total_charge_cents BIGINT NOT NULL DEFAULT 0,
    allowed_cents BIGINT,
    paid_cents BIGINT,
    patient_responsibility_cents BIGINT,
    denial_reason TEXT,
    denial_code TEXT,
    submitted_at TIMESTAMPTZ,
    adjudicated_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claims_encounter ON claims(encounter_id);
CREATE INDEX idx_claims_patient ON claims(patient_id, created_at DESC);
CREATE INDEX idx_claims_status ON claims(status) WHERE status NOT IN ('paid', 'voided');

CREATE TABLE claim_lines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id UUID NOT NULL REFERENCES claims(id) ON DELETE CASCADE,
    line_number INT NOT NULL,
    cpt_code TEXT NOT NULL,
    modifier TEXT[],
    icd10_codes TEXT[] NOT NULL,
    description TEXT NOT NULL,
    units INT NOT NULL DEFAULT 1,
    charge_cents BIGINT NOT NULL,
    allowed_cents BIGINT,
    paid_cents BIGINT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claim_lines ON claim_lines(claim_id);
```

---

## E-Prescribing & Payments

```sql
CREATE TABLE e_prescriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID NOT NULL REFERENCES encounters(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    prescriber_id UUID NOT NULL REFERENCES providers(id),
    medication_name TEXT NOT NULL,
    sig TEXT NOT NULL,
    quantity TEXT NOT NULL,
    refills INT NOT NULL DEFAULT 0,
    pharmacy_ncpdp TEXT,
    pharmacy_name TEXT,
    surescripts_message_id TEXT,
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'sent', 'dispensed', 'cancelled', 'denied'
    )),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_erx_patient ON e_prescriptions(patient_id, created_at DESC);

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id UUID NOT NULL REFERENCES patients(id),
    location_id UUID NOT NULL REFERENCES locations(id),
    amount_cents BIGINT NOT NULL,
    payment_method TEXT NOT NULL CHECK (payment_method IN (
        'cash', 'credit_card', 'debit_card', 'check', 'insurance_payment',
        'account_credit', 'care_credit'
    )),
    reference_number TEXT,
    claim_id UUID REFERENCES claims(id),
    dispensing_order_id UUID REFERENCES dispensing_orders(id),
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_patient ON payments(patient_id, created_at DESC);
```

---

## Audit Log & AI

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

CREATE TABLE ai_analyses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID REFERENCES encounters(id),
    patient_id UUID REFERENCES patients(id),
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

CREATE INDEX idx_ai_encounter ON ai_analyses(encounter_id);
```

---

## Example Queries

### Full patient chart with latest prescription

```sql
SELECT p.*, vp.prescription_type,
       vp.od_sphere, vp.od_cylinder, vp.od_axis, vp.od_add,
       vp.os_sphere, vp.os_cylinder, vp.os_axis, vp.os_add,
       vp.expiration_date
FROM patients p
LEFT JOIN LATERAL (
    SELECT * FROM vision_prescriptions
    WHERE patient_id = p.id
    ORDER BY created_at DESC LIMIT 1
) vp ON TRUE
WHERE p.id = 'patient-uuid';
```

### IOP trending for glaucoma monitoring

```sql
SELECT e.created_at::DATE AS visit_date,
       om.od_value AS od_iop, om.os_value AS os_iop,
       om.method
FROM ocular_measurements om
JOIN encounters e ON e.id = om.encounter_id
WHERE om.measurement_type = 'iop'
  AND e.patient_id = 'patient-uuid'
ORDER BY e.created_at;
```

### Claim denial rate by payer

```sql
SELECT ip.payer_name,
       COUNT(*) AS total_claims,
       COUNT(*) FILTER (WHERE c.status = 'denied') AS denied,
       COUNT(*) FILTER (WHERE c.status = 'denied') * 100.0 / COUNT(*) AS denial_pct
FROM claims c
JOIN patient_insurances pi2 ON pi2.id = c.insurance_id
JOIN insurance_plans ip ON ip.id = pi2.plan_id
WHERE c.location_id = 'location-uuid'
  AND c.created_at >= CURRENT_DATE - 90
GROUP BY ip.payer_name
ORDER BY denial_pct DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Practice & Providers | 4 | practices, locations, providers, provider_locations |
| Patients | 1 | patients |
| Insurance | 2 | insurance_plans, patient_insurances |
| Scheduling | 1 | appointments |
| Encounters & Exams | 4 | encounters, refraction_data, ocular_measurements, diagnoses |
| Prescriptions | 1 | vision_prescriptions (eyeglasses + CL) |
| Dispensary | 3 | frames, frame_inventory, dispensing_orders |
| Claims | 2 | claims, claim_lines |
| E-Prescribing & Payments | 2 | e_prescriptions, payments |
| Audit & AI | 2 | audit_log (partitioned), ai_analyses |
| **Total** | **22** | |

---

## Key Design Decisions

1. **Refraction data as typed columns** — `refraction_data` stores sphere, cylinder, axis, add, prism for each eye as real/integer columns. This aligns with the FHIR VisionPrescription resource and enables precision queries ("find all patients with > -6.00 myopia").

2. **Ocular measurements as typed rows** — `ocular_measurements` stores each measurement type (IOP, pachymetry, keratometry) as its own row with OD/OS values. This enables trending ("show IOP over 5 visits") and device-specific data without per-device columns.

3. **Vision prescriptions with CL fields** — `vision_prescriptions` combines eyeglasses and contact lens prescription fields in one table with `prescription_type` discriminator. CL-specific fields (base curve, diameter, brand) are nullable for eyeglasses prescriptions. This keeps the prescription entity unified while supporting both types.

4. **Insurance claims with line items** — `claims` stores the claim header while `claim_lines` stores CPT-coded line items with ICD-10 pointers. This supports both vision and medical claim types and enables line-level denial tracking.

5. **Frame inventory by location** — `frame_inventory` stores per-location quantities for each frame. This supports multi-location practices where inventory visibility across locations is essential for the optical dispensary.

6. **Dispensing orders linked to prescriptions** — `dispensing_orders` references the vision prescription, frame, and lens configuration. The `status` lifecycle tracks the order from lab submission through patient notification to dispensing.

7. **HIPAA audit log** — `audit_log` stores every access to PHI with user, action, resource, and IP address. Partitioned by half-year for retention management. This satisfies HIPAA Security Rule technical safeguard requirements.

8. **Diagnoses with eye laterality** — `diagnoses` includes an `eye` column (OD, OS, OU) because optometric diagnoses are frequently unilateral. This aligns with ICD-10-CM optometry-specific codes that distinguish laterality.
