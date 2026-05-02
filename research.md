# Optometry Practice Platform

> Candidate #322 · Researched: 2026-05-02

## Existing Products and Software Packages

| Tool | Description | Type | Pricing | Strengths / Weaknesses |
|------|-------------|------|---------|------------------------|
| RevolutionEHR | Cloud-based optometry-specific EHR and practice management platform | Cloud SaaS | Quote only | Strengths: purpose-built for ODs, strong compliance tools; Weaknesses: limited optical dispensing depth |
| Eyefinity Encompass | All-in-one platform covering EHR, practice management, and patient engagement; owned by VSP | Cloud SaaS | Quote only | Strengths: VSP integration, net-collection dashboards; Weaknesses: complex onboarding |
| Crystal Practice Management | Integrated optical dispensary management with point-of-sale, frame/contact lens inventory | Hybrid | Quote only | Strengths: deep dispensing tools; Weaknesses: UI dated compared with cloud rivals |
| MaximEyes | Combines EHR, practice management, and optical dispensing in one platform | Hybrid/Cloud | Quote only | Strengths: breadth of features; Weaknesses: learning curve |
| Compulink Advantage | Long-established optometry platform with clinical templates and billing | On-premise/Cloud | Quote only | Strengths: mature feature set; Weaknesses: slower innovation cadence |
| Barti | Modern cloud EHR and practice management built for independent eye care | Cloud SaaS | Quote only | Strengths: clean UX, fast development; Weaknesses: newer, smaller install base |
| Glasson | Cloud-based optical retail and practice management focusing on omnichannel dispensing | Cloud SaaS | Tiered | Strengths: omnichannel focus; Weaknesses: less clinical depth |
| OfficeMate / ExamWRITER | Henry Schein's optometry suite combining practice management and EHR | Desktop/Cloud | Quote only | Strengths: wide adoption; Weaknesses: legacy architecture |

## Relevant Industry Standards or Protocols

- **HIPAA (Health Insurance Portability and Accountability Act)** — governs all patient health data; optometry practices must encrypt retinal scans, OCT images, and prescription records in transit and at rest
- **HL7 FHIR (Fast Healthcare Interoperability Resources)** — emerging standard for exchanging clinical data between EHR systems, referral partners, and patient-facing apps
- **ANSI Z80 Standards** — American National Standards Institute standards for prescription formats and lens/frame tolerances relevant to the dispensing workflow
- **CMS MIPS / Promoting Interoperability** — federal quality reporting requirements that ODs participating in Medicare must satisfy, including attestations tied to EHR use
- **VSP / EyeMed / Davis Vision Insurance EDI Formats** — proprietary electronic claim submission formats for the dominant vision insurance carriers
- **ICD-10-CM / CPT Coding** — standard diagnostic and procedural code sets that billing modules must maintain and update annually

## Available Research Materials

1. Stolee, P., et al. (2019). *Electronic Health Records in Optometry: A Systematic Review of Implementation and Outcomes.* Optometry and Vision Science. https://doi.org/10.1097/OPX.0000000000001352 — peer-reviewed
2. American Optometric Association. (2024). *AOA Coding Optometry: CPT and ICD-10 Guidance.* https://www.aoa.org/practice/billing-coding — industry guidance (not peer-reviewed)
3. RevolutionEHR. (2026). *Optometry Practice Management Trends for 2026: AI, Software, KPIs.* https://www.revolutionehr.com/blogs/trends-optometry-practice-management — industry analysis (not peer-reviewed)
4. Grand View Research. (2024). *Ophthalmology EHR Market Size & Forecast to 2027.* (Referenced via clinikehr.com summary) — market report (not peer-reviewed)
5. Gitnux. (2026). *Optometry Industry Statistics: Market Data Report.* https://gitnux.org/optometry-industry-statistics/ — aggregated industry data (not peer-reviewed)
6. Patient Protect. (2026). *HIPAA Compliance for Optometry Practices: The Complete 2026 Guide.* https://patient-protect.com/post/hipaa-compliance-optometry-practices-2026 — compliance guide (not peer-reviewed)
7. ClinikalEHR. (2026). *Clinic Management Software for Eye Clinics: Complete Guide.* https://clinikehr.com/blog/clinic-management-software-for-eye-clinics — industry analysis (not peer-reviewed)

## Market Research

**Market Size:** The global ophthalmology EHR market is projected to reach USD 1.2 billion by 2027, growing at approximately 8.7% CAGR. Corporate optometry ownership now represents roughly 45% of practices, driving demand for multi-site management features.

**Funding:** VSP Global (owner of Eyefinity) is a dominant strategic investor. Henry Schein owns the OfficeMate/ExamWRITER suite. Newer cloud-native entrants such as Barti have raised early-stage funding targeting independent OD practices.

**Pricing Landscape:** Most enterprise platforms require direct sales engagement. HIPAA compliance overhead averages $8,000 per practice annually. Smaller cloud platforms tier pricing by number of providers; typical range is $300–$800/month for independent practices.

**Key Buyer Personas:** Independent OD practice owners (prioritise simplicity and insurance billing accuracy), group optometry practices (need multi-location visibility), optical retail chains (prioritise dispensing and POS), ophthalmology/optometry hybrid practices (need clinical depth and surgical referral workflows).

**Notable Trends:** AR-based virtual try-on and omnichannel dispensing (browse online, purchase in-store) are gaining traction. Patients increasingly expect self-scheduling, digital intake forms, and two-way messaging. Integration with diagnostic instruments (OCT, autorefractors) via DICOM is becoming a baseline expectation.

## AI-Native Opportunity

- **Automated refraction and exam documentation** by connecting directly to diagnostic instruments and pre-populating clinical notes, eliminating manual transcription errors
- **AI-driven optical recommendation engine** that combines prescription data, face-shape analysis, and past purchase history to suggest frames and lens options at the dispensing counter
- **Intelligent insurance eligibility and claim pre-screening** that predicts likely denials before submission and suggests corrective coding, reducing rework and improving first-pass claim acceptance rates
- **Predictive recall and population health management** using longitudinal vision data to flag patients at elevated risk for conditions such as glaucoma or macular degeneration and prioritise outreach
- **Conversational AI intake assistant** that collects chief complaint, ocular history, and medication details before the patient arrives, freeing exam time for clinical decision-making
