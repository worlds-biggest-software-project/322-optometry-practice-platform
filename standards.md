# Standards & API Reference

> Project: Optometry Practice Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

No single ISO standard governs optometry practice management end-to-end, but several ISO standards are directly relevant to data security, clinical imaging, and device interoperability:

- **ISO/IEC 27001 (Information Security Management Systems)** — https://www.iso.org/standard/27001
  The baseline security management framework for any healthcare SaaS platform. Optometry EHRs storing PHI (Protected Health Information) should align with ISO 27001 controls in conjunction with HIPAA technical safeguards.

- **ISO 13485 (Medical Devices Quality Management)** — https://www.iso.org/standard/59752.html
  Relevant to any software component classified as a Software as a Medical Device (SaMD), such as AI-driven clinical decision support tools that interpret OCT or visual field data.

- **ISO 14971 (Risk Management for Medical Devices)** — https://www.iso.org/standard/72704.html
  Required for risk management processes covering any SaMD functionality (e.g., AI scribe features used for clinical documentation, diagnostic AI).

- **ISO 80001 (Application of Risk Management for IT Networks Incorporating Medical Devices)** — https://www.iso.org/standard/44863.html
  Relevant to integration of diagnostic equipment (autorefractors, OCT devices, phoropters) into the practice's network and EHR system.

---

### W3C & IETF Standards

- **RFC 7231 — HTTP/1.1 Semantics and Content** — https://datatracker.ietf.org/doc/html/rfc7231
  Foundation for all REST-based API communication in the platform, including FHIR REST APIs and integration endpoints.

- **RFC 6749 — The OAuth 2.0 Authorization Framework** — https://datatracker.ietf.org/doc/html/rfc6749
  Required for SMART on FHIR authentication and for any third-party application access to patient data. All reviewed EHR FHIR APIs use OAuth 2.0 with PKCE.

- **RFC 7519 — JSON Web Tokens (JWT)** — https://datatracker.ietf.org/doc/html/rfc7519
  Used as the token format in OAuth 2.0 / SMART on FHIR flows; relevant to secure API access token handling.

- **RFC 8288 — Web Linking** — https://datatracker.ietf.org/doc/html/rfc8288
  Used in FHIR paging (Bundle next/prev links) for paginating large patient data sets returned by the FHIR server.

- **W3C WebAuthn (Web Authentication)** — https://www.w3.org/TR/webauthn-3/
  Relevant for implementing phishing-resistant MFA across clinician, staff, and patient-facing portals; increasingly expected by HIPAA-aligned security frameworks.

---

### Data Model & API Specifications

- **HL7 FHIR R4 (Fast Healthcare Interoperability Resources, Release 4)** — https://hl7.org/fhir/R4/
  The primary interoperability standard for clinical data exchange. Key FHIR resources for optometry include:
  - `VisionPrescription` — structured representation of eyeglass and contact lens prescriptions: https://hl7.org/fhir/R4/visionprescription.html
  - `Patient`, `Practitioner`, `Organization`, `Appointment`, `Encounter`, `Condition`, `Observation`, `DiagnosticReport`, `Coverage`, `Claim`, `ClaimResponse`
  - ONC requires FHIR R4 (US Core Implementation Guide) for certified EHR systems as of 2026.

- **HL7 FHIR R4 — US Core Implementation Guide (v6.1)** — https://www.hl7.org/fhir/us/core/
  Constrains FHIR R4 resources to US-specific requirements including USCDI v3 data elements. Mandatory for ONC-certified EHR systems from January 1, 2026.

- **HL7 FHIR Eye Care Implementation Guide** — https://build.fhir.org/ig/HL7/fhir-eyecare-ig/
  Under active development by the HL7 Patient Care Working Group ("Eyes on FHIR" project). Specifies FHIR profiles for ophthalmic observations, retinal imaging findings, and visual field results. GitHub: https://github.com/HL7/fhir-eyecare-ig

- **HL7 v2.x Messaging (ADT, ORU, MDM)** — https://www.hl7.org/implement/standards/product_brief.cfm?product_id=185
  Still widely used for device-to-EHR messaging (autorefractors, OCT devices sending results via ORU messages), lab order/result flows, and legacy system integrations.

- **DICOM (Digital Imaging and Communications in Medicine)** — https://www.dicomstandard.org/
  The universal standard for ophthalmic image storage and transfer. Key supplements for optometry/ophthalmology:
  - **DICOM Supplement 110** — OCT for ophthalmology
  - **DICOM Supplement 130** — Keratometry and autorefractor measurements
  - Ophthalmic imaging DICOM profile defines storage of fundus photographs, OCT scans, and visual field data
  - Despite the standard existing, incomplete DICOM adoption across ophthalmic device manufacturers remains a documented interoperability challenge.

- **ANSI Z80 Standards (American National Standards Institute)** — https://www.ansi.org/
  Covers ophthalmic prescription formats, lens tolerances, and frame measurements relevant to the dispensing workflow. Key standards include:
  - ANSI Z80.1: Prescription ophthalmic lenses — tolerances
  - ANSI Z80.5: Requirements for ophthalmic frames
  - ANSI Z80.20: Contact lens standards
  These define the data model for dispensing prescriptions and optical lab orders.

- **OpenAPI Specification 3.1** — https://spec.openapis.org/oas/v3.1.0
  Recommended specification format for documenting the platform's own REST API, enabling third-party integrations and developer tooling.

- **NCPDP SCRIPT Standard v2023011** — https://www.ncpdp.org/
  The electronic prescribing messaging standard for transmitting prescriptions from prescribers (including optometrists) to pharmacies. Surescripts routes NCPDP SCRIPT messages. CMS mandated v2023011 upgrade for certified EHR systems from 2024 onward.

---

### Security & Authentication Standards

- **HIPAA Security Rule (45 CFR Part 164, Subpart C)** — https://www.hhs.gov/hipaa/for-professionals/security/index.html
  Federal standard governing administrative, physical, and technical safeguards for electronic PHI. All optometry EHR platforms operating in the US must comply. Technical safeguards include encryption at rest and in transit, access controls, audit logging, and automatic logoff.

- **HIPAA Privacy Rule (45 CFR Part 164, Subpart E)** — https://www.hhs.gov/hipaa/for-professionals/privacy/index.html
  Governs permissible uses and disclosures of patient health information. Updated in 2024 (42 CFR Part 2) with enforcement deadline February 16, 2026 to align substance use disorder privacy protections with HIPAA.

- **ONC Health IT Certification (21st Century Cures Act, HTI-1 Final Rule)** — https://www.healthit.gov/topic/certification-ehrs/2015-edition-test-method
  EHR certification programme administered by ONC. Certified systems must support USCDI v3 (mandatory from January 1, 2026), SMART on FHIR API access, and prohibition on information blocking. Optometry EHRs participating in CMS incentive programmes must maintain certification.

- **CMS Interoperability and Prior Authorization Final Rule (CMS-0057-F)** — https://www.cms.gov/newsroom/fact-sheets/cms-interoperability-prior-authorization-final-rule-cms-0057-f
  Requires payers to expose patient data via FHIR Provider Access API (effective January 1, 2026) and implement electronic prior authorisation workflows (effective January 1, 2027). Impacts EHR systems integrating with payer APIs.

- **SMART on FHIR (Substitutable Medical Applications and Reusable Technologies)** — https://smarthealthit.org/
  An open authorisation framework built on OAuth 2.0 + FHIR that allows third-party apps to securely access EHR data. All major optometry EHR FHIR APIs (RevolutionEHR, Eyefinity) implement SMART on FHIR.

- **NIST SP 800-66 (Guide to HIPAA Security Rule)** — https://csrc.nist.gov/publications/detail/sp/800-66/rev-2/final
  NIST guidance that provides a structured approach to implementing the HIPAA Security Rule; the de-facto reference for healthcare security risk assessments.

- **SOC 2 Type 2 (AICPA)** — https://www.aicpa.org/resources/article/soc-2
  Third-party audited security, availability, and confidentiality controls. Eyefinity Encompass is the first optometry platform to achieve SOC 2 Type 2; increasingly expected by enterprise buyers and group practices.

- **OWASP Application Security Verification Standard (ASVS)** — https://owasp.org/www-project-application-security-verification-standard/
  Best-practice application security checklist relevant to web-based EHR portals and patient-facing applications. ASVS Level 2 is appropriate for healthcare applications handling PHI.

---

### EDI & Billing Standards

- **ANSI X12 837P (Professional Claims)** — https://x12.org/products/transaction-sets
  Standard EDI transaction for submitting professional insurance claims (vision and medical) from optometry practices to payers and clearinghouses. Used for VSP, EyeMed, Davis Vision, medical carriers.

- **ANSI X12 270/271 (Eligibility Inquiry/Response)** — https://x12.org/products/transaction-sets
  EDI transactions for real-time insurance eligibility verification. Used by all reviewed platforms for pre-visit eligibility checks to confirm patient vision benefit availability and deductible status.

- **ANSI X12 835 (Electronic Remittance Advice)** — https://x12.org/products/transaction-sets
  Standard transaction carrying payment and adjudication details from payers back to the practice. Used for automated ERA posting into patient accounts receivable.

- **USCDI v3 (United States Core Data for Interoperability, Version 3)** — https://www.healthit.gov/isa/united-states-core-data-interoperability-uscdi
  Mandatory from January 1, 2026 for certified EHR systems. Defines the minimum data elements that must be electronically shareable: patient demographics (including sexual orientation/gender identity), clinical notes, lab results, medications, allergies, and more. Expands scope compared with USCDI v1/v2.

---

### MCP Server Specifications

The Model Context Protocol (MCP) is relevant to AI-native deployments where optometry EHR data is surfaced to LLM-powered agents (e.g., AI scribe, AI copilot, clinical decision support):

- **MCP Protocol Specification** — https://modelcontextprotocol.io/
  Defines how AI agents access structured context (patient records, exam templates, insurance rules) from server-side resources. An optometry EHR could expose MCP servers for:
  - Patient chart context (demographics, history, prescriptions)
  - Insurance benefit eligibility data
  - Clinical decision support rules (diagnostic criteria, recall schedules)
  - Optical inventory and dispensing data

  This would allow an AI scribe or copilot agent to ground its outputs in real practice data without bespoke prompt engineering.

---

## Similar Products — Developer Documentation & APIs

### RevolutionEHR

- **Description:** Cloud-based optometry EHR and practice management platform purpose-built for optometrists, covering scheduling, clinical documentation, optical dispensing, billing, and patient engagement.
- **API Documentation:** https://revolutionehrdev.dynamicfhir.com/dhit/basepractice/r4/Home/ApiDocumentation
- **FHIR Server Base URL:** https://revolutionehrdev.dynamicfhir.com/ (dev); production endpoints per practice
- **Developer Guide:** https://help.revehr.com/hc/en-us/articles/11465192746903-Standardized-API-for-Patient-and-Population-Services
- **Integration Partners:** https://www.revolutionehr.com/integrations (Keragon, IntakeQ, 300+ tools via middleware)
- **Standards:** FHIR R4, SMART on FHIR, ONC 2015 Edition Cures Update certified, USCDI
- **Authentication:** OAuth 2.0 (SMART on FHIR); third-party apps must request access at partner-requests@revolutionehr.com

---

### Eyefinity Encompass / ExamWRITER

- **Description:** All-in-one optometry platform from VSP Global covering EHR, practice management, patient engagement, and analytics. ExamWRITER is the legacy server-based EHR; Encompass is the current cloud platform.
- **API Documentation (Encompass):** https://fhirapi.eyefinity.com/eyefinity/base/r4/Home/ApiDocumentation
- **API Documentation (ExamWRITER):** https://fhirapi.eyefinity.com/examwriter/154562/r4/Home/ApiDocumentation
- **FHIR Server Base URL:** https://fhirapi.eyefinity.com/
- **Certified Partners:** https://www.eyefinity.com/products/partners/certified-partners/integration.html
- **Developer Guide:** https://www.eyefinity.com/resource/regulatory/fhir.html
- **Standards:** FHIR R4, SMART on FHIR, ONC certified, SOC 2 Type 2, USCDI
- **Authentication:** OAuth 2.0 (SMART on FHIR); access gated by VSP/Eyefinity approval — contact Meaningfuluse@VSP.com

---

### OpenEMR

- **Description:** Open-source EHR and practice management system with an optometry/ophthalmology module. Free to self-host; GPL v2 licence.
- **API Documentation:** https://www.open-emr.org/wiki/index.php/OpenEMR_API_REST
- **GitHub Repository:** https://github.com/openemr/openemr
- **Eye Exam Module:** https://www.open-emr.org/wiki/index.php/Eye_Exam
- **Developer Guide:** https://github.com/openemr/openemr/blob/master/API_README.md
- **Standards:** FHIR R4 (ONC certified build available), HL7 v2, REST/JSON, OpenAPI
- **Authentication:** OAuth 2.0 with PKCE; API keys for trusted internal integrations

---

### Metriport (Open-Source Healthcare API)

- **Description:** Open-source healthcare data API for aggregating and normalising patient data across EHR systems using FHIR and CDA standards. Relevant as infrastructure layer for an AI-native optometry platform needing cross-EHR patient data access.
- **API Documentation:** https://docs.metriport.com/
- **GitHub Repository:** https://github.com/metriport/metriport
- **Developer Guide:** https://docs.metriport.com/getting-started/introduction
- **Standards:** FHIR R4, CDA R2, REST/JSON, OpenAPI 3.1
- **Authentication:** API Key (server-to-server); OAuth 2.0 for patient-facing access

---

### Surescripts (E-Prescribing Network)

- **Description:** National e-prescribing network connecting prescribers (including optometrists) to pharmacies. All commercial optometry EHRs integrate with Surescripts for NCPDP SCRIPT-based electronic prescription routing.
- **Developer Documentation:** https://surescripts.com/developers (requires vendor registration)
- **NCPDP SCRIPT Standard Reference:** https://www.ncpdp.org/resources.aspx
- **Standards:** NCPDP SCRIPT v2023011, RESTful APIs for eligibility and formulary queries
- **Authentication:** Surescripts vendor credentialing required; mTLS for production connections

---

### TriZetto / Cognizant (EDI Clearinghouse)

- **Description:** Major healthcare EDI clearinghouse used by Eyefinity (via its TriZetto partnership) and other optometry platforms for claim submission, eligibility verification, and remittance processing. Provider of the Claims Status Inquiry (CSI) used in Encompass.
- **Developer Documentation:** https://www.trizettogroup.com/provider-solutions/
- **Standards:** ANSI X12 837P, 270/271, 835; HIPAA-compliant EDI processing
- **Authentication:** Trading partner agreement required; SFTP and API connections available

---

### HL7 Da Vinci FHIR Prior Authorization (CRD/DTR/PAS)

- **Description:** HL7 Da Vinci project specifying FHIR-based workflows for Coverage Requirements Discovery (CRD), Documentation Templates and Rules (DTR), and Prior Authorization Support (PAS). Directly relevant to optometry billing workflows under the CMS-0057-F prior authorisation mandate (effective 2027).
- **Implementation Guide:** https://build.fhir.org/ig/HL7/davinci-crd/
- **PAS IG:** https://hl7.org/fhir/us/davinci-pas/
- **Standards:** FHIR R4, SMART on FHIR, CDS Hooks
- **Authentication:** OAuth 2.0 / SMART on FHIR

---

### IHE Eye Care Technical Framework

- **Description:** Integration framework published by IHE International (sponsored by the American Academy of Ophthalmology) specifying how eye care devices, EHR systems, and image management systems communicate using DICOM and HL7 standards.
- **Technical Framework Vol 1 (Profiles):** https://www.ihe.net/Technical_Framework/upload/ihe_eyecare_tf_rev3-7_vol1_Final_Text_2010-02-15.pdf
- **Unified Eye Care Workflow Supplement:** https://www.ihe.net/uploadedFiles/Documents/Eye_Care/IHE_EyeCare_Suppl_U-EYECARE_Refractive_Rev1.1_TI_2016-06-14.pdf
- **IHE Eye Care Domain:** https://www.ihe.net/ihe_domains/eye_care/
- **Key Profiles:** Unified Eye Care Workflow (U-EYECARE), Eye Care Evidence Documents (ECED), Eye Care Charge Posting (EC-CHG)
- **Standards:** DICOM, HL7 v2, IHE integration profiles

---

### DICOM Standard (Ophthalmology Supplements)

- **Description:** The universal medical imaging standard. Key supplements define how OCT, autorefractor, keratometer, fundus camera, and visual field device data are structured and transferred to EHR/PACS systems.
- **DICOM Standard Official Site:** https://www.dicomstandard.org/
- **Supplement 110 (OCT for Ophthalmology):** https://www.dicomstandard.org/News-dir/ftsup/docs/sups/sup110.pdf
- **Ophthalmology Image Recommendations:** https://pmc.ncbi.nlm.nih.gov/articles/PMC8335850/
- **Standards:** DICOM PS 3.x, IHE Eye Care profiles

---

## Notes

- **DICOM adoption gap:** Despite comprehensive DICOM supplements for ophthalmic devices, the AAO has documented that full native DICOM implementation on imaging devices and EHR systems remains incomplete across the industry. An AI-native platform should anticipate needing proprietary device adapter layers in addition to standard DICOM interfaces.

- **FHIR Eye Care IG maturity:** The HL7 FHIR Eye Care Implementation Guide (https://github.com/HL7/fhir-eyecare-ig) is still under active development and has not yet reached balloted status. Optometry-specific FHIR profiles are not yet standardised; platforms currently rely on generic FHIR resources (Observation, DiagnosticReport) with custom extensions.

- **USCDI v3 enforcement from 2026:** Starting January 1, 2026, USCDI v3 compliance is mandatory for ONC-certified systems. New data element categories include sexual orientation/gender identity and social determinants of health — areas that historically have not been captured in optometry practice workflows and will require new intake and documentation fields.

- **CMS Prior Authorisation FHIR APIs (2027):** The CMS-0057-F rule requires payers to implement FHIR-based Prior Authorisation APIs by January 1, 2027. An optometry EHR should plan to integrate with payer FHIR APIs for real-time prior authorisation status, reducing the manual phone-and-fax workflows currently prevalent for vision plan medical-necessity cases.

- **MCP as emerging AI interface standard:** The Model Context Protocol is not yet a healthcare-domain standard but is gaining traction as a framework for connecting AI agents to structured data sources. An AI-native optometry platform deploying LLM-based scribe or copilot features should evaluate MCP server implementations to expose clinical context to AI agents in a structured and auditable way.
