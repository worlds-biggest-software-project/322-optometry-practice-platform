# Optometry Practice Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source practice platform for optometry covering patient records, exam workflows, eyewear dispensing, and billing.

The Optometry Practice Platform is a candidate project to build a unified EHR, practice management, and optical dispensary system purpose-built for independent and group optometry practices. It targets the gap between legacy quote-only enterprise suites and generic open-source EHRs, combining specialty-specific clinical workflows with AI-driven documentation, coding, and dispensing assistance.

---

## Why an Optometry Practice Platform?

- Every reviewed enterprise platform (RevolutionEHR, Eyefinity Encompass, Crystal PM, MaximEyes, Compulink, OfficeMate/ExamWRITER) is proprietary and quote-only, with no self-serve pricing for independent ODs.
- HIPAA compliance overhead averages roughly USD 8,000 per practice annually, and typical cloud platforms cost USD 300–800/month per provider, pricing many independent practices out of modern tooling.
- The only fully open-source option, OpenEMR, treats optometry as a bolt-on module with basic OD-specific workflows, no AI features, and minimal optical dispensary or POS support.
- Legacy suites such as OfficeMate/ExamWRITER and Compulink are widely deployed but innovate slowly, with dated UIs and limited mobile or remote access.
- API access on commercial platforms is gated by vendor approval and usually limited to read-only USCDI data via SMART on FHIR R4, restricting third-party extensibility.

---

## Key Features

### Clinical EHR and Exam Workflow

- Optometry-specific exam templates for refraction, contact lens fitting, and ocular health
- Patient demographics, medical history, and prescription management
- Diagnostic instrument integration with auto-population of exam findings (autorefractor, OCT, visual field)
- E-prescribing via Surescripts / NCPDP SCRIPT
- Role-based exam views and progressive disclosure during documentation

### Optical Dispensary and Retail

- Frame, lens, and contact lens inventory management
- Integrated POS with payment processing and ledger posting
- Order tracking for optical labs
- Warranty and return tracking linked to patient and dispensing records
- Multi-location inventory visibility

### Scheduling and Patient Engagement

- Appointment scheduling with calendar views, reminders, and waitlist management
- Online self-booking and digital intake forms
- Automated SMS, email, and voice recall messaging
- Two-way patient messaging

### Billing and Insurance

- 837P claim generation for medical and vision insurance
- 270/271 eligibility verification
- ERA (835) processing and automated payment posting
- ICD-10/CPT coding support with denial-risk scoring
- VSP, EyeMed, and Davis Vision EDI submission

### Compliance and Multi-Site Operations

- HIPAA-compliant storage with role-based access control and audit logging
- Multi-location support with consolidated reporting
- Quality registry reporting hooks (AOA MORE Registry, AAO IRIS Registry, CMS MIPS)
- ONC certification path for Promoting Interoperability requirements

---

## AI-Native Advantage

AI capabilities are designed in from the start rather than retrofitted: an ambient scribe converts exam dialogue to structured SOAP notes, an intelligent insurance engine pre-screens claims against payer rules to predict denials before submission, and an AI-driven optical recommender combines prescription data, face-shape analysis, and purchase history to suggest frames and lenses at the dispensing counter. Predictive recall uses longitudinal vision data to flag patients at elevated risk for glaucoma, macular degeneration, or diabetic eye disease, and a conversational intake assistant collects chief complaint and ocular history before the patient arrives.

---

## Tech Stack & Deployment

The platform targets cloud-hosted SaaS as the primary deployment mode, with a self-hostable option for practices that need on-premise control. Interoperability is built around HL7 FHIR R4 (SMART on FHIR), HL7 ORU/MDM messaging for diagnostic devices, DICOM for imaging instruments, and Surescripts / NCPDP SCRIPT for e-prescribing. EDI 837P/835/270/271 transactions cover insurance billing for medical and vision payers. Frame and lens dispensing follows ANSI Z80 prescription and tolerance standards.

---

## Market Context

The global ophthalmology EHR market is projected to reach USD 1.2 billion by 2027 at roughly 8.7% CAGR (Grand View Research, 2024), with corporate ownership of around 45% of practices driving multi-site demand. Incumbent enterprise platforms are quote-only and dominated by VSP-owned Eyefinity and Henry Schein's OfficeMate/ExamWRITER; cloud-native entrants such as Barti have raised early-stage funding (USD 12M Series A) targeting independent ODs. Primary buyers are independent OD owners, group practices, optical retail chains, and ophthalmology/optometry hybrid practices.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
