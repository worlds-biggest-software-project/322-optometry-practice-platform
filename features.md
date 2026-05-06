# Optometry Practice Platform — Feature & Functionality Survey

> Candidate #322 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| RevolutionEHR | Cloud SaaS | Commercial, quote-only | https://www.revolutionehr.com/ |
| Eyefinity Encompass | Cloud SaaS | Commercial, quote-only | https://www.eyefinity.com/products/solutions/eyefinity-encompass.html |
| Crystal Practice Management | Hybrid (on-premise + cloud) | Commercial, quote-only | https://www.crystalpm.com/ |
| MaximEyes | Hybrid/Cloud SaaS | Commercial, quote-only | https://www.maximeyes.com/ |
| Compulink Advantage | On-premise/Cloud | Commercial, quote-only | https://compulinkadvantage.com/optometry/ |
| OfficeMate / ExamWRITER | Desktop/Cloud | Commercial (Eyefinity/VSP-owned) | https://www.eyefinity.com/products/solutions/officemate-examwriter.html |
| Barti | Cloud SaaS | Commercial, tiered | https://barti.com/ |
| Glasson | Cloud SaaS | Commercial, tiered | https://www.glasson.app/ |
| OpenEMR (Eye Module) | Self-hosted | Open source (GPL) | https://github.com/openemr/openemr |

---

## Feature Analysis by Solution

### RevolutionEHR

**Core features**
- Cloud-based EHR with optometry-specific clinical exam templates
- Appointment scheduling with calendar views, reminders, and waitlist management
- Diagnostic equipment integrations for auto-population of exam findings
- Optical dispensary management (frames, lenses, contact lenses inventory and sales)
- Integrated insurance billing and electronic claims submission
- Patient engagement via RevEngage (SMS/email campaigns, automated recall messages)
- RevIntake (online digital patient intake forms before the appointment)
- Built-in reporting and KPI dashboards

**Differentiating features**
- Native SMART on FHIR® API certified under ONC 2015 Edition Cures Update
- RevEngage reportedly increases patient retention by 35% and generates 60% more online reviews
- Strong diagnostic device integration ecosystem covering major instrument manufacturers
- Optometry-specific workflows without need for generic add-ons

**UX patterns**
- Single-platform onboarding; all workflows from scheduling to dispensing in one login
- Modular UI with role-based views (front desk, clinician, optician, admin)
- Progressive disclosure: exam template expands sections as the clinician documents

**Integration points**
- SMART on FHIR R4 API (read-only patient data for third-party apps)
- HL7 ORU/MDM device messaging for diagnostic instrument data
- Electronic lab ordering for optical laboratories
- Keragon and other middleware connectors support 300+ third-party tools
- Surescripts for e-prescribing

**Known gaps**
- Limited multi-location management depth (compared to enterprise competitors)
- No native VoIP / phone system integration
- Optical retail / omnichannel e-commerce features are basic relative to retail-focused rivals

**Licence / IP notes**
- Proprietary commercial SaaS. No open-source components exposed.

---

### Eyefinity Encompass

**Core features**
- Integrated practice management + EHR + patient engagement + analytics ("core four")
- EncompassScribe: AI-assisted exam documentation built into the platform
- EncompassPay: integrated payment processing with in-app refund capability
- Two-way patient messaging (text, voice, email)
- Claims Status Inquiry (CSI) via TriZetto eDI partnership for real-time claim tracking
- Streamlined lab ordering supporting multiple optical laboratories
- KPI dashboards for production, capture rate, and accounts receivable
- VSP insurance portal direct integration (as VSP-owned platform)

**Differentiating features**
- First SOC 2 Type 2 certified optometry software with 150+ cybersecurity controls
- Deep VSP Vision Care integration gives it native claim and eligibility advantages for VSP practices
- EncompassScribe is the first enterprise AI scribe built natively into an optometry EHR

**UX patterns**
- All-in-one layout designed to reduce tab-switching between PM and EHR views
- Onboarding designed for staff of varying technical skill; modular training pathways
- Real-time claim status within the billing workflow rather than requiring portal visits

**Integration points**
- SMART on FHIR R4 API (read-only USCDI data for third-party apps)
- FHIR API portal: https://fhirapi.eyefinity.com/eyefinity/base/r4/Home/ApiDocumentation
- TriZetto / Cognizant for EDI claim submission and real-time CSI
- HL7 interfaces for lab and device data
- Partner ecosystem via Eyefinity Certified Partners programme

**Known gaps**
- Onboarding is complex and reported as time-consuming by users
- Historically tied to VSP ecosystem, giving non-VSP practices less EDI advantage
- Desktop OfficeMate/ExamWRITER legacy products still in active use with limited API access

**Licence / IP notes**
- Proprietary commercial SaaS. Owned by VSP Global. API access gated by vendor approval.

---

### Crystal Practice Management

**Core features**
- Appointment scheduling and patient registration
- Clinical documentation (EHR) with optometry-specific templates
- Optical dispensary management with frame, lens, and contact lens inventory tracking
- POS with integrated payment processing and automated ledger posting
- Revenue cycle management and insurance billing
- Patient engagement (reminders, follow-ups)
- Reporting and business analytics

**Differentiating features**
- Deep optical retail POS integration — inventory, packaging, and optical ordering tools tightly coupled with clinical records
- Optical warranty and return tracking linked directly to patient and dispensing records
- Synchronises dispensing information, warranties, and payments in a single ledger automatically

**UX patterns**
- Integrated single-screen design connecting clinical exam records to optical sales workflow
- Designed to reduce re-entry when handing off from exam room to optical dispensary
- Inventory views accessible without leaving the patient chart

**Integration points**
- Insurance billing clearinghouse integrations for EDI claim submission
- Third-party payment processing integrations
- Optical lab ordering interfaces

**Known gaps**
- UI perceived as dated compared to cloud-native rivals
- Limited patient-facing digital engagement (no robust online booking or intake forms)
- API access not publicly documented; limited third-party extensibility

**Licence / IP notes**
- Proprietary commercial software. No public API or SDK.

---

### MaximEyes

**Core features**
- Cloud-based and server-based EHR with optometry and ophthalmology workflows
- AI Scribe (EVAA Scribe) for ambient voice documentation generating SOAP notes
- Automated ICD-10/CPT coding triggers based on exam notes to prevent claim denials
- Contact lens fitting workflows, drawing tools, and auto-letters
- Lab interfacing for optical and pharmaceutical orders
- Physician dashboard and automated patient recall reminders
- E-prescribing to pharmacies (first ophthalmic e-prescribing solution to market)
- AOA MORE Registry and IRIS Registry (AAO) direct integration for quality reporting
- CMS PQRS/MIPS reporting support

**Differentiating features**
- First EHR to integrate with the AOA MORE Registry for quality measure reporting
- First to offer ophthalmic e-prescribing; sub-10-second pharmacy transmission
- Automated coding triggers are specialty-specific, designed to prevent optometry-specific denials
- Compatible with Android and iOS mobile devices for mobile-first exam documentation

**UX patterns**
- Structured specialty exam workflows that mirror real OD practitioner flows (refraction, CL fitting)
- Diagnostic tools structured around typical exam sequence to minimise out-of-workflow clicks
- Server-based option available for practices with limited broadband

**Integration points**
- Diagnostic device integrations (autorefractors, OCT, field analysers)
- Surescripts / e-prescribing network
- AOA MORE Registry API
- AAO IRIS Registry API
- CMS PQRS direct reporting

**Known gaps**
- No public API documented for third-party developers
- Multi-site management considered less polished than enterprise competitors
- Learning curve reported as steep for practices migrating from simpler platforms

**Licence / IP notes**
- Proprietary commercial software. No public open API.

---

### Compulink Advantage

**Core features**
- All-in-one optometry EHR and practice management with OneTab single-screen layout
- Extensive optometry-specific clinical template library (diagnoses, ocular histories, exam structures)
- Optical POS and inventory management integrated with clinical records
- SMART Practice AI module for near-fully automated billing (claims generation and working)
- Workflow automation tools: billing triggers, eligibility verification, appointment reminders
- Patient engagement suite (recall, online booking, intake)
- Multi-location management for group and chain practices

**Differentiating features**
- SMART billing AI reportedly reduces time on claims by approximately 90%
- OneTab layout eliminates context switching between PM and EHR screens
- Long-established with a mature code base covering edge cases built up over decades
- Strong multi-location configuration for optical chains and group practices

**UX patterns**
- OneTab single-screen layout presents the full exam workflow without navigating between modules
- Procedure opportunity reminders surface relevant add-on services during the exam
- Designed for high-volume practices; automation prioritised over customisation depth

**Integration points**
- EDI clearinghouse integrations for medical and vision insurance claims
- Diagnostic device interfaces (OCT, autorefractor, visual field)
- Optical lab ordering
- Patient communication platform integrations

**Known gaps**
- Innovation cadence perceived as slower compared with cloud-native startups
- Some users report difficulty switching between locations in multi-site deployments
- UI feel described as dated by users transitioning from modern consumer apps

**Licence / IP notes**
- Proprietary commercial software. No public API documentation found.

---

### OfficeMate / ExamWRITER (Eyefinity)

**Core features**
- Appointment scheduling with day/week calendar views and patient history lookup
- ExamWRITER EHR for automated documentation of patient history, exams, test results, and prescriptions
- Insurance billing (patient and insurance), ERA processing, general ledger integration
- Lab ordering integrations
- HL7 interfaces for integration with medical software systems
- ONC-ATCB 2014 certified compliance for Meaningful Use / Promoting Interoperability

**Differentiating features**
- Dominant legacy install base — industry standard since 1985
- Practitioner familiarity advantage: staff trained on OfficeMate rarely require retraining
- FHIR API available for ExamWRITER (separate from Encompass) for legacy practice interoperability

**UX patterns**
- Desktop-first UI with server-based architecture; familiar Windows interface conventions
- New engine in version 15 adds modernised backend while preserving familiar UI
- Primarily designed for single-location independent and small-group practices

**Integration points**
- ExamWRITER FHIR R4 API: https://fhirapi.eyefinity.com/examwriter/
- HL7 interfaces for clinical and lab data
- VSP/EyeMed EDI claim submission
- Lab ordering integrations

**Known gaps**
- Legacy architecture; limited mobile and remote access compared with cloud rivals
- Shrinking active development investment as Eyefinity pushes migration to Encompass
- Online patient engagement features absent or limited without third-party add-ons

**Licence / IP notes**
- Proprietary commercial software. Legacy product; Eyefinity is actively migrating customers to Encompass.

---

### Barti

**Core features**
- Cloud EHR with flexible, paper-like single-tab exam charting
- AI Scribe: first optometry EHR to deploy AI-driven voice documentation
- AI Office Copilot: AI phone agent that answers calls, schedules appointments, and enters data into the calendar
- Integrated VoIP phone system within the EHR platform
- Optical inventory management: frames, lenses, contact lenses, vendor orders, POS
- Patient engagement (automated reminders, recall messaging, intake forms)
- Insurance billing and claims submission
- AOAExcel endorsed and invested (first EHR with formal AOA endorsement)

**Differentiating features**
- First eye care EHR endorsed and invested in by AOAExcel (AOA's financial products arm)
- First optometry EHR with a deployed AI Office Copilot handling inbound phone calls end-to-end
- Integrated VoIP eliminates a separate phone system subscription
- ACQUIOS partnership for AI-powered business analytics and benchmarking for independent ODs
- Raised $12M Series A specifically to accelerate AI-powered EHR capabilities

**UX patterns**
- Single-tab charting mimics paper chart layout to reduce learning curve for clinicians accustomed to paper
- AI-handled call intake reduces front desk interruptions during exam flow
- Clean, modern SaaS UI targeting independent OD practices that find legacy platforms overwhelming

**Integration points**
- Diagnostic instrument integrations (machine integration for auto-population of exam findings)
- Insurance EDI clearinghouse connections
- VoIP telephony integrated natively
- ACQUIOS analytics platform

**Known gaps**
- Newer entrant with smaller install base and less edge-case coverage than legacy competitors
- Depth of multi-location management features still evolving
- Limited public API documentation for third-party developers

**Licence / IP notes**
- Proprietary commercial SaaS. No public open API.

---

### Glasson

**Core features**
- Cloud-based optical practice management with patient appointment booking and intake
- Full patient history including past prescriptions, frame choices, and service notes
- Intelligent lens search: database of 5M+ lens variants with AI-powered matching across 22,000+ vision defect combinations
- Multi-location support with shared client databases and cross-location inventory visibility
- Inventory management with real-time stock visibility and reorder alerts
- Automated personalised recall reminders and follow-up messaging
- Optical retail omnichannel support for online + in-store operations

**Differentiating features**
- Intelligent lens search engine covering over 5 million lens variants — the deepest lens catalogue of any platform reviewed
- Omnichannel optical retail focus designed for optical salons as much as clinical practices
- Tiered transparent pricing (published, unlike most competitors)

**UX patterns**
- Retail-centric UI designed to serve optical salon staff with low clinical IT backgrounds
- Lens finder tool surfaces results in seconds, reducing dispensing consultation time
- Multi-location chain view allows centralised management without per-location logins

**Integration points**
- Online booking integrations
- Inventory supplier connections
- Patient communication integrations (SMS, email)

**Known gaps**
- Clinical EHR depth is limited; primarily a retail/dispensing and PM tool rather than full clinical EHR
- HIPAA compliance posture less prominent (targeted at European optical market primarily)
- No AI documentation or scribe capabilities

**Licence / IP notes**
- Proprietary commercial SaaS. Tiered pricing published at https://www.glasson.app/price-list/.

---

### OpenEMR (Eye Module)

**Core features**
- Open-source EHR with ophthalmology/optometry module (Eye Exam module)
- Impression/Plan Builder extracting diagnoses from ocular history and exam findings with ICD-10/CPT coding
- General practice management (scheduling, billing, electronic claims)
- FHIR R4 API support (ONC-certified)
- Active GitHub repository with community contributions

**Differentiating features**
- Only fully open-source option; self-hostable with no per-provider licensing fees
- Community-contributed ophthalmology/optometry module with dedicated eye exam documentation forms
- Extendable by developers via public GitHub repository and documented REST API

**UX patterns**
- General-purpose EHR with optometry bolted on via specialty module
- UI is functional but reflects generic EHR design conventions rather than OD-specific workflows
- Suitable for technically capable practices or non-profit/international deployment scenarios

**Integration points**
- OpenEMR REST API and FHIR R4 API: https://github.com/openemr/openemr
- HL7 and DICOM interfaces
- Surescripts e-prescribing integration available
- Community-built integrations for various labs and devices

**Known gaps**
- Optometry-specific workflows are basic compared with purpose-built commercial platforms
- No AI scribe or AI-powered features in core build
- Requires significant technical resources to deploy and maintain; not suitable for non-technical practices
- Optical dispensary and POS features are minimal

**Licence / IP notes**
- GPL v2 or later. Free and open source. See https://github.com/openemr/openemr.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Appointment scheduling with automated reminders (text, email, voice)
- Optometry-specific clinical exam documentation templates (refraction, CL fitting, ocular health)
- Patient demographics, medical history, and prescription management
- Insurance eligibility verification (270/271 EDI transactions)
- Electronic claims submission (837P for medical and vision insurance)
- ERA (835) processing and automated payment posting
- Optical dispensary inventory management (frames, lenses, contact lenses)
- E-prescribing integration with pharmacy networks (Surescripts/NCPDP SCRIPT)
- Automated patient recall and recall compliance tracking
- ONC certification and HIPAA-compliant data storage

### Differentiating Features
- AI ambient scribe for voice-driven SOAP note generation (Barti, MaximEyes, Eyefinity Encompass)
- Automated ICD-10/CPT coding triggers that predict and prevent claim denials (MaximEyes, Compulink)
- AI-powered billing automation claiming up to 90% reduction in claims processing time (Compulink SMART Practice)
- AI phone copilot handling inbound call intake and scheduling (Barti)
- Deep diagnostic instrument integrations with auto-population of exam findings (RevolutionEHR, MaximEyes)
- Native VoIP integration eliminating a separate phone subscription (Barti)
- SOC 2 Type 2 security certification (Eyefinity Encompass)
- Quality registry direct reporting: AOA MORE, AAO IRIS Registry (MaximEyes)
- Intelligent lens search across millions of variants (Glasson)

### Underserved Areas / Opportunities
- **True omnichannel dispensing:** Allowing patients to browse frames online and complete the purchase in-store with seamless optical record linkage remains partially solved
- **DICOM instrument integration:** Despite standards existing, complete adoption of DICOM for OCT, autorefractors, and other devices remains inconsistent across platforms
- **Predictive population health management:** Flagging patients at risk for glaucoma, macular degeneration, or diabetic retinopathy using longitudinal exam data is absent from all reviewed platforms
- **Transparent, affordable pricing for independent ODs:** Nearly all enterprise platforms require direct sales quotes; no self-serve pricing model exists for small practices
- **Cross-platform patient data portability:** Practices switching EHR vendors face poor data migration tooling; no neutral data export standard is enforced
- **AI-driven optical recommendations:** Combining prescription data, face-shape analysis, and purchase history for frame and lens recommendations at the dispensing counter is not offered
- **Integrated virtual try-on:** AR/AI virtual try-on is referenced as a market trend but is not natively embedded in any reviewed EHR-adjacent platform
- **Coordinated care and specialist referral workflows:** Structured referral workflows to ophthalmology, retina specialists, or primary care are shallow in all platforms

### AI-Augmentation Candidates
- **Exam documentation:** All reviewed AI scribe implementations use voice-to-text → structured SOAP notes; higher-order AI could suggest differential diagnoses and risk stratification from exam data
- **Claim denial prediction and prevention:** Automated coding triggers exist; AI could further pre-screen claims against payer-specific rule sets before submission
- **Recall and appointment scheduling optimisation:** AI could predict optimal recall windows per patient based on condition risk profile rather than blanket 12-month cycles
- **Lens and frame recommendations:** Rule-based dispensing assistance could be replaced with a personalised AI recommendation engine
- **Diagnostic image analysis:** AI interpretation of OCT, fundus photography, and visual field data is an emerging capability not embedded in any reviewed PM platform natively

---

## Legal & IP Summary

All commercial platforms reviewed (RevolutionEHR, Eyefinity Encompass, Crystal PM, MaximEyes, Compulink Advantage, OfficeMate/ExamWRITER, Barti, Glasson) are proprietary commercial software with no open-source components. API access is typically gated behind vendor approval processes. The FHIR R4 APIs provided under ONC certification are read-only and scoped to USCDI data, which limits the buildable surface for third-party applications. OpenEMR is licensed under GPL v2 or later and can be freely used, modified, and redistributed under those terms. No patents on specific optometry workflow features were identified in public records during this research; however, AI scribe and coding-trigger implementations may be the subject of pending or unidentified patents. An independent AI-native optometry platform built from scratch would not infringe on any reviewed open-source licence provided it does not incorporate GPL code without complying with that licence.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Appointment scheduling with automated SMS/email reminders and online self-booking
- Optometry-specific EHR with refraction, contact lens fitting, and ocular health exam templates
- Optical dispensary module: inventory, POS, order tracking for frames, lenses, and contact lenses
- Insurance billing: 837P claim generation, 270/271 eligibility verification, ERA 835 posting for vision and medical plans
- HIPAA-compliant data storage with role-based access control and audit logging
- Patient demographic and prescription history management

**Should-have (v1.1)**
- AI ambient scribe for voice-driven exam documentation
- Automated ICD-10/CPT coding suggestions with denial-risk scoring
- Patient recall engine with predictive scheduling (not just fixed-interval recall)
- Diagnostic instrument integration (autorefractor, OCT, visual field auto-population)
- E-prescribing via Surescripts / NCPDP SCRIPT
- Multi-location support with consolidated reporting

**Nice-to-have (backlog)**
- AI phone copilot for inbound call handling and scheduling
- AI-driven frame and lens recommendation engine at the dispensing counter
- AR virtual try-on integration
- Population health dashboard flagging patients at risk for glaucoma, AMD, or diabetic eye disease
- FHIR R4 API for patient-facing app access and care coordination
- Quality registry reporting (AOA MORE Registry, AAO IRIS Registry, CMS MIPS)
