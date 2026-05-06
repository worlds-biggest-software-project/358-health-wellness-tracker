# Standards & API Reference

> Project: Health & Wellness Tracker · Candidate #358 · Generated: 2026-05-04

---

## Industry Standards & Specifications

### ISO / IEEE Standards

**ISO/IEEE 11073 — Personal Health Device Communication**
- URL: https://sagroups.ieee.org/11073/phd-wg/ and https://www.iso.org/standard/84781.html
- A family of standards enabling communication between personal health devices (wearables, blood pressure monitors, glucose monitors, scales) and external systems such as smartphones and health gateways. The architecture defines Device (agent) and Manager (smartphone/gateway) roles with a Domain Information Model, Service Model, and Communication Model. Part 20601 (2022) covers the Optimised Exchange Protocol for Bluetooth and USB transport. Relevant for any open-source health tracker that communicates directly with medical-grade or consumer health devices without going through a proprietary cloud API.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The international standard for information security management. Defines requirements for an ISMS covering risk assessment, access control, encryption, audit logging, and incident response. Health apps storing personal health data (PHI) should align their security programme with ISO 27001. The 2022 revision requires migration by October 2025. Commonly paired with ISO/IEC 27701 for privacy extensions.

**ISO/IEC 27701:2025 — Privacy Information Management Systems (PIMS)**
- URL: https://www.tekclarion.com/cyber-security/iso-iec-27701-2025-global-privacy-standard-guide/
- The 2025 edition is now a standalone standard (no longer requiring prior ISO 27001 certification). Extends the ISMS framework with privacy controls aligned to GDPR and other regional data protection regulations. Directly applicable to health apps processing personal health data under EU or UK jurisdiction. Provides a certifiable privacy framework complementing GDPR legal obligations.

---

### W3C & IETF Standards

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational authorization framework used by all major wearable and health platform APIs (Oura, Fitbit, WHOOP, Withings, Garmin). Defines the Authorization Code, Client Credentials, Implicit, and Resource Owner Password grant types. Health apps must implement the Authorization Code + PKCE flow (RFC 7636) for mobile clients accessing wearable data.

**RFC 7636 — Proof Key for Code Exchange (PKCE)**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- Extension to OAuth 2.0 that prevents authorization code interception attacks in native and mobile applications. Required by all major wearable APIs for mobile client implementations and mandated by the 2025 OAuth Security Best Current Practice.

**RFC 9700 — OAuth 2.0 Security Best Current Practice (2025)**
- URL: https://oauth.net/2/oauth-best-practice/
- Published by IETF in January 2025. Consolidates the current best practices for OAuth 2.0 implementations, including mandatory PKCE, DPoP token binding for enhanced security, rotating refresh tokens, and the deprecation of the Implicit grant. Health apps implementing OAuth flows for wearable API access should reference this document for security posture.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Authentication layer built on OAuth 2.0 that adds identity tokens (JWTs) and a UserInfo endpoint. Required for health apps implementing user account authentication alongside wearable data authorisation. The HEART (Health Relationship Trust) profile (below) extends OIDC specifically for healthcare contexts.

**HEART Profile for OAuth 2.0 and OpenID Connect**
- URL: https://openid.net/specs/openid-heart-oauth2-1_0.html
- A specialised profile of OAuth 2.0 and OpenID Connect for healthcare applications. Mandates asymmetric key authentication at token endpoints, restricts grant types to Authorization Code for user-facing clients, and aligns with HIPAA and GDPR requirements. Relevant for health apps seeking formal regulatory alignment in clinical or near-clinical contexts.

---

### Data Model & API Specifications

**HL7 FHIR R5 (Fast Healthcare Interoperability Resources)**
- URL: https://www.hl7.org/fhir/
- The primary international standard for electronic health information exchange. FHIR R5 (v5.0.0) defines RESTful APIs and JSON/XML/RDF resource types covering Patient, Observation, Condition, MedicationRequest, DiagnosticReport, and more. Mandated by the 2020 21st Century Cures Act for US health IT interoperability. Relevant for any health tracker that integrates with clinical EHR systems, medical records, or the Apple Health medical records feature. FHIR Observation resources can model wearable biometric readings in a clinically interoperable format.

**SMART on FHIR (SMART App Launch Framework)**
- URL: https://docs.smarthealthit.org/ and https://hl7.org/fhir/smart-app-launch/
- A security and launch framework built on FHIR + OAuth 2.0 + OpenID Connect that enables health apps to launch from and exchange data with EHR systems. Used by Apple Health to connect to hundreds of US healthcare systems for medical records access. Relevant for a health tracker wishing to pull clinical records (labs, medications, visit history) from EHR systems to contextualise wearable and self-reported data.

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry standard for describing RESTful HTTP APIs. Used by Oura (v2), Fitbit, WHOOP, and Withings to publish machine-readable API contracts. An AI-native health tracker should expose its own API as an OpenAPI 3.1 document to support third-party integrations and AI agent tool use.

**USDA FoodData Central API**
- URL: https://fdc.nal.usda.gov/api-guide/
- A free, publicly accessible REST API providing nutritional data for 380,000+ foods validated to US federal standards. The primary open-licensed data source for building a nutrition tracking feature without dependency on proprietary food databases. Free for unrestricted use; no licence fee.

**Open Food Facts API**
- URL: https://openfoodfacts.github.io/openfoodfacts-server/api/
- An open, community-maintained database of food products from 180+ countries covering barcoded packaged goods. Available under the Open Database Licence (ODbL). REST API provides product lookup by barcode and full nutrient profiles. Complementary to USDA FoodData Central for global branded food coverage.

---

### Security & Authentication Standards

**HIPAA (Health Insurance Portability and Accountability Act) — Security Rule**
- URL: https://www.hhs.gov/hipaa/for-professionals/security/
- US federal regulation requiring covered entities and business associates handling Protected Health Information (PHI) to implement administrative, physical, and technical safeguards. Technical requirements include AES-256 encryption at rest, TLS 1.2+ in transit, access control, audit logging retained for 6+ years, and formal incident response plans. While consumer wellness apps are generally not covered entities, HIPAA-aligned security practices are the de facto standard for health data. The 2025 proposed HIPAA Security Rule update mandates end-to-end encryption, expanded risk assessments, and stricter MFA.

**GDPR (General Data Protection Regulation) — Article 9: Special Categories**
- URL: https://gdpr-info.eu/art-9-gdpr/
- EU regulation classifying health data as a "special category" of personal data requiring explicit consent before processing, mandatory data protection impact assessments (DPIA), appointment of a Data Protection Officer for systematic health data processing, and the right to erasure. Any health tracker serving EU users must comply. ISO/IEC 27701:2025 provides a certifiable framework aligned to GDPR.

**OWASP Mobile Application Security Verification Standard (MASVS) 2.1**
- URL: https://mas.owasp.org/MASVS/
- Industry baseline for mobile app security covering data storage, network communication, cryptography, authentication, and platform-specific security. Health apps should target MASVS-L2 (Defence in Depth) given the sensitivity of health data. Provides specific test cases for secure local data storage, certificate pinning, and biometric authentication.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- URL: https://modelcontextprotocol.io/
- An open protocol for connecting AI language models to external tools and data sources. Open Wearables v0.3 (February 2026) introduced an MCP server connecting wearable data to Claude and ChatGPT, enabling natural-language health queries over personal biometric history. An AI-native health tracker could expose nutrition logs, symptom records, and wearable data via an MCP server, allowing users to query their health data through any MCP-compatible AI assistant without building a custom chat interface.

---

## Similar Products — Developer Documentation & APIs

### Apple HealthKit

- **Description:** The iOS and watchOS framework providing a central, permission-controlled repository for health and fitness data. Apps can read and write data across 100+ data types including activity, biometrics, sleep, nutrition, and clinical records.
- **API Documentation:** https://developer.apple.com/documentation/healthkit
- **SDKs/Libraries:** Swift (native), React Native via `react-native-health`, Flutter via `health` package
- **Developer Guide:** https://developer.apple.com/health-fitness/
- **Standards:** Proprietary HealthKit data model; FHIR R4 for medical records (ClinicalDocument resources); CDA/FHIR for EHR record import
- **Authentication:** Per-data-type user permission prompt; no OAuth flow — entitlement granted by App Store category

### Google Health Connect

- **Description:** Android's unified health data platform providing read/write access to fitness, activity, sleep, nutrition, and biometric data from Android apps and Wear OS devices.
- **API Documentation:** https://developer.android.com/health-and-fitness/guides/health-connect
- **SDKs/Libraries:** Android Health Connect SDK (Kotlin/Java); Flutter via `health` package; React Native via `react-native-health-connect`
- **Developer Guide:** https://developer.android.com/health-and-fitness/guides/health-connect/plan/data-types
- **Standards:** Proprietary Health Connect data schema; aligns with FHIR Observation resource types conceptually
- **Authentication:** Android permission model with per-data-type user consent

### Oura Ring API

- **Description:** REST API providing access to sleep, readiness, activity, heart rate, HRV, and resilience data generated by the Oura Ring. OAuth 2.0-based with a public developer portal.
- **API Documentation:** https://cloud.ouraring.com/docs/
- **SDKs/Libraries:** Python: `oura-ring` (community); no official SDK
- **Developer Guide:** https://support.ouraring.com/hc/en-us/articles/4415266939155-The-Oura-API
- **Standards:** REST/JSON; OpenAPI documented
- **Authentication:** OAuth 2.0 (Authorization Code flow)

### Fitbit Web API (Google)

- **Description:** REST API providing access to activity, sleep, heart rate, body measurements, food logging, and GPS data from Fitbit devices. Supports personal and production application tiers.
- **API Documentation:** https://dev.fitbit.com/build/reference/web-api/
- **SDKs/Libraries:** Python: `python-fitbit` (community); JavaScript: `fitbit-node` (community); no official SDK
- **Developer Guide:** https://dev.fitbit.com/
- **Standards:** REST/JSON; OAuth 2.0
- **Authentication:** OAuth 2.0 (Authorization Code + PKCE)

### WHOOP API

- **Description:** REST API providing access to recovery, strain, sleep, heart rate, HRV, and activity data from WHOOP wearables. v2 API released with improved data models and webhook support.
- **API Documentation:** https://developer.whoop.com/api/
- **SDKs/Libraries:** No official SDK; community Python and Node.js clients available
- **Developer Guide:** https://developer.whoop.com/
- **Standards:** REST/JSON; webhook-based event delivery
- **Authentication:** OAuth 2.0 (Authorization Code flow)

### Garmin Connect API

- **Description:** API platform providing access to activity, wellness, sleep, and biometric data from Garmin devices. Access requires formal application and approval; not fully open like Oura or Fitbit.
- **API Documentation:** https://developer.garmin.com/gc-developer-program/overview/
- **SDKs/Libraries:** Connect IQ SDK for on-device app development; no REST SDK for server-side integration
- **Developer Guide:** https://developer.garmin.com/gc-developer-program/
- **Standards:** REST/JSON; OAuth 2.0
- **Authentication:** OAuth 2.0 (application review required for production access)

### Withings Health API

- **Description:** REST API for accessing data from Withings connected health devices: scales, blood pressure monitors, sleep analysers, and smartwatches. Well-documented with a public developer portal.
- **API Documentation:** https://developer.withings.com/api-reference/
- **SDKs/Libraries:** Python: `nokia-health` (community); no official SDK
- **Developer Guide:** https://developer.withings.com/developer-guide/v3/withings-solutions/app-to-app-solution/
- **Standards:** REST/JSON; OAuth 2.0 Web flow
- **Authentication:** OAuth 2.0 (Authorization Code flow)

### USDA FoodData Central API

- **Description:** Free US government REST API providing nutritional data for 380,000+ foods including SR Legacy, FNDDS, Foundation Foods, and Branded Food datasets. Public domain data, no licence required.
- **API Documentation:** https://fdc.nal.usda.gov/api-guide/
- **SDKs/Libraries:** No official SDK; straightforward REST API usable with any HTTP client
- **Developer Guide:** https://fdc.nal.usda.gov/api-guide/
- **Standards:** REST/JSON; API key authentication (free registration)
- **Authentication:** API key (free, no approval process)

### Open Food Facts API

- **Description:** Open, community-maintained REST API covering barcoded food products from 180+ countries with nutritional data, ingredients, allergens, and labels. Data available under ODbL licence.
- **API Documentation:** https://openfoodfacts.github.io/openfoodfacts-server/api/
- **SDKs/Libraries:** Python: `openfoodfacts`; JavaScript: `openfoodfacts-nodejs`; Swift: `OFFApi`; Dart/Flutter: `openfoodfacts`
- **Developer Guide:** https://openfoodfacts.github.io/openfoodfacts-server/api/
- **Standards:** REST/JSON; no authentication required for read access
- **Authentication:** None for read access; account required for write/contribution

---

## Notes

**Wearable API Access Asymmetry:** Oura, Fitbit, and Withings offer relatively open developer portals with self-service registration. WHOOP offers open API access at v2. Garmin requires a formal application and business case review before granting production API access — this is a meaningful barrier for open-source projects. Polar similarly requires a partnership process for production data volumes.

**FHIR and Consumer Wellness Gap:** The FHIR standard is well-established for clinical data exchange but adoption in consumer wellness apps remains limited. The primary consumer integration points (HealthKit, Health Connect) use proprietary data schemas that conceptually map to FHIR Observation resources but are not natively FHIR-compliant. A health tracker bridging consumer wearable data and clinical records (e.g. for the medical appointment export feature) would need to implement FHIR mapping from HealthKit/Health Connect data types.

**Food Database Licensing:** USDA FoodData Central is public domain (no licence constraints). Open Food Facts is ODbL — derivative databases must also be published under ODbL, but applications using the data are not affected. These two sources together provide sufficient coverage for a global open-source nutrition tracking feature without depending on proprietary databases like MyFitnessPal's or Cronometer's.

**MCP for Health:** Open Wearables v0.3 (February 2026) demonstrated that MCP server integration for wearable data is technically feasible and enables natural-language health querying via Claude and ChatGPT without building a custom LLM interface. This is an emerging pattern with significant potential for AI-native health trackers.

**Regulatory Scope Clarity:** Consumer wellness apps that do not store, transmit, or process data on behalf of covered healthcare entities are generally not directly subject to HIPAA in the US. However, HIPAA-aligned security practices (AES-256, TLS 1.2+, access control, audit logs) are the accepted baseline for health data. GDPR Article 9 applies to any app processing health data for EU residents regardless of whether the app has clinical intent.
