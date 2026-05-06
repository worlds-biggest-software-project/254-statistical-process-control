# Standards & API Reference

> Project: Statistical Process Control · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 7870 Series — Control Charts**
- URL: https://www.iso.org/standard/40174.html (Part 2) and related parts
- The primary ISO standard series for control charts. Parts cover: Part 1 (general guidelines), Part 2 (Shewhart control charts; replaces ISO 8258:1991), Part 3 (acceptance control charts), Part 9 (control charts for stationary processes). Mandatory reference for any SPC software implementing standard charting methods.

**ISO 8258:1991 — Shewhart Control Charts (withdrawn)**
- URL: https://www.iso.org/standard/15366.html
- The original Shewhart chart standard, now superseded by ISO 7870-2:2013. Relevant for understanding legacy implementations and compliance claims in older software.

**ISO 22514 Series — Statistical Methods in Process Management: Capability and Performance**
- URL: https://www.iso.org/standard/69643.html (Part 9 example)
- Defines Cp, Cpk, Pp, Ppk and related process capability indices. Multiple parts cover: variable characteristics (Part 2), attribute characteristics (Part 5), multivariate normal distributions (Part 6), measurement process capability (Part 7; also known as MSA/Gage R&R standard), and geometrical product specification characteristics (Part 9). Essential for any SPC platform computing capability indices.

**ISO 11462-1 / ISO/FDIS 11462-1 — Guidelines for Implementation of SPC**
- URL: https://www.iso.org/standard/85264.html (FDIS, replacing 2001 edition)
- Part 1 defines the elements, tools, and techniques of SPC. A draft revision (FDIS) is in the approval phase as of 2026. Part 3 (ISO/TR 11462-3:2020) provides reference data sets for SPC software validation; Part 4 (ISO/TR 11462-4:2022) covers measurement process analysis software validation; Part 5 (ISO/TR 11462-5:2023) defines a quality data exchange format for SPC software interoperability.

**ISO 9001:2015 / ISO 9001:2026 (revision in progress) — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- Clause 9.1 requires monitoring, measurement, analysis, and evaluation; SPC is the primary method of evidence. A revision cycle is underway with publication anticipated around 2026; changes expected to align risk-based thinking and measurement requirements with IATF 16949 and AS9100.

**ISO 13485:2016 — Medical Devices Quality Management Systems**
- URL: https://www.iso.org/standard/59752.html
- Requires documented process control and statistical methods for medical device manufacturing. SPC software used in medical device facilities must support 21 CFR Part 11 audit trails and traceable records.

### Automotive & Aerospace Quality Standards

**AIAG-VDA SPC Yellow Volume (2026 Draft)**
- URL: https://www.aiag.org/training-and-resources/manuals/details/SPCDRAFT
- Harmonised AIAG and VDA SPC manual eliminating conflicting terminology between US and European automotive quality practices. Expands the SPC toolkit beyond Shewhart charts, distinguishes performance from capability, and aligns SPC within risk-based IATF 16949 quality planning. Stakeholder review open through May 3, 2026; expected to become the reference standard for automotive supplier SPC. SPC software vendors are planning updates (DataLyzer: Summer 2026).

**IATF 16949:2016 — Automotive Quality Management Systems**
- URL: https://www.iatfglobaloversight.org/
- Mandates documented SPC plans, control charts, and capability studies (minimum Cpk ≥ 1.33) for special characteristics. Requires AIAG core tools: APQP, PPAP, FMEA, MSA, and SPC. A second edition (tentatively IATF 16949:2027) is expected to follow ISO 9001:2026 by 12–18 months.

**AIAG SPC Manual (4th Edition / Yellow Volume)**
- URL: https://www.aiag.org/expertise-areas/quality/quality-core-tools
- The practitioner's reference for automotive SPC implementation, control chart selection, capability study requirements, and reaction plan documentation. Software must align terminology (Cp, Cpk, Pp, Ppk, control limits, process capability) with this manual for customer acceptance.

**AIAG MSA Manual — Measurement Systems Analysis**
- URL: https://www.aiag.org/expertise-areas/quality/quality-core-tools
- Defines Gage R&R study methods (Average & Range, ANOVA), bias studies, linearity, and stability analyses required before SPC implementation. SPC platforms must implement MSA methods per this manual for automotive compliance.

**AS9100 Rev D — Aerospace Quality Management Systems**
- URL: https://www.sae.org/standards/content/as9100/
- Aerospace standard requiring statistical methods and process capability demonstration for critical characteristics. Aligns with ISO 9001 but adds aerospace-specific requirements for risk management and traceability.

### Regulatory Compliance Standards

**FDA 21 CFR Part 11 — Electronic Records; Electronic Signatures**
- URL: https://www.ecfr.gov/current/title-21/chapter-I/subchapter-A/part-11
- Requires computer-generated, time-stamped, tamper-evident audit trails for all electronic records; prohibits shared user accounts; requires electronic signatures with full name, date, and meaning. SPC software used in FDA-regulated manufacturing (pharma, medical devices, food) must meet Part 11 requirements. Enforcement has intensified: 327 FDA warning letters in H2 2025, a 73% increase year-over-year.

**FDA 21 CFR Part 820 — Quality System Regulation (Medical Devices)**
- URL: https://www.ecfr.gov/current/title-21/chapter-I/subchapter-H/part-820
- Requires manufacturers to establish and maintain process controls, including statistical methods, to ensure device conformance. SPC is a primary compliance mechanism.

### Manufacturing Integration Standards

**ISA-95 / IEC 62264 — Enterprise-Control System Integration**
- URL: https://www.isa.org/standards-and-publications/isa-95-standard
- Defines the five-level manufacturing hierarchy and data models for integrating ERP and MES with shop-floor control. Relevant because SPC fits within ISA-95's "Quality Operations Management" functional area (inspection planning, SPC, traceability, deviation handling). B2MML (Business To Manufacturing Markup Language) is the open-source XML implementation of ISA-95/IEC 62264.

**OPC UA (OPC Unified Architecture)**
- URL: https://opcfoundation.org/about/opc-technologies/opc-ua/
- The dominant machine-to-system communication standard for industrial IoT. Provides secure, reliable, manufacturer-neutral data transport from PLCs, CNCs, and sensors to MES and quality systems. Combined with MQTT for PubSub messaging, OPC UA is the preferred data ingestion path for real-time SPC platforms. Relevant for SPC software receiving live machine measurements.

**MQTT Protocol (ISO/IEC 20922:2016)**
- URL: https://mqtt.org/
- Lightweight publish/subscribe messaging protocol optimised for constrained IoT environments; widely used for millisecond-level machine data streaming to SPC platforms. ISO/IEC 20922:2016 is the formal standard.

### Data Model & API Specifications

**ISO/TR 11462-5:2023 — Quality Data Exchange Format for SPC Software**
- URL: https://www.iso.org/standard/74098.html
- Defines a standard data exchange format to enable interoperability between SPC software packages. Relevant for import/export, inter-system data sharing, and SPC software validation.

**OpenAPI Specification 3.x**
- URL: https://spec.openapis.org/oas/latest.html
- De-facto standard for describing REST APIs; relevant for any SPC platform exposing a REST API for data ingestion or third-party integration. All modern SPC platform APIs should publish an OpenAPI specification.

**JSON Schema**
- URL: https://json-schema.org/
- Standard for defining and validating JSON payloads; relevant for SPC data ingestion API request/response contracts and the ISO/TR 11462-5 quality data exchange format.

### Security & Authentication Standards

**OAuth 2.0 / OpenID Connect**
- URL: https://oauth.net/2/ | https://openid.net/connect/
- Industry-standard protocols for API authentication and authorisation. SPC platforms with REST APIs should support OAuth 2.0 bearer tokens for system-to-system integration and OIDC for user identity federation with enterprise identity providers (Active Directory, Okta, etc.).

**NIST SP 800-53 / NIST Cybersecurity Framework**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- US federal security controls framework increasingly referenced in industrial cybersecurity postures. SPC platforms handling process data in defence or government supply chains may encounter CMMC requirements which build on NIST SP 800-171.

**OWASP Top 10**
- URL: https://owasp.org/www-project-top-ten/
- Reference for web application security; applicable to SPC platforms delivered as web applications, particularly around data input validation, injection vulnerabilities, and broken access control.

---

## Similar Products — Developer Documentation & APIs

### Minitab Real-Time SPC
- **Description:** Cloud SaaS platform for real-time SPC charting, capability analysis, and process dashboards; strong statistical pedigree; SAP Digital Manufacturing certified.
- **API Documentation:** https://support.minitab.com/en-us/real-time-spc/api-access/the-real-time-spc-api/
- **Station Streaming Endpoint:** https://support.minitab.com/en-us/real-time-spc/api-access/station-endpoint-for-data-streaming/
- **SDKs/Libraries:** Not publicly published; REST API with API key authentication only
- **Developer Guide:** https://support.minitab.com/en-us/real-time-spc/ (support hub); Getting Started PDF available from minitab.com
- **Standards:** REST/JSON; API key authentication
- **Authentication:** API Key per request

### InfinityQS Enact
- **Description:** Cloud-native SPC platform by InfinityQS/Advantive; modern REST API; real-time enterprise quality dashboards.
- **API Documentation:** https://enacthelp.infinityqs.com/ (online help including API sections)
- **SDKs/Libraries:** Not publicly published; REST API with refresh key
- **Developer Guide:** https://enacthelp.infinityqs.com/en-us/Configuration/Documentation.htm
- **Standards:** REST/JSON; refresh-key authentication
- **Authentication:** Refresh Key (OAuth-style token exchange)

### WinSPC
- **Description:** On-prem/cloud SPC platform with 500+ method API library for bidirectional integration with MES, ERP, HMI, and LIMS systems.
- **API Documentation:** Available to licenced customers via vendor; not publicly indexed
- **SDKs/Libraries:** COM/ActiveX API library; 500+ methods and properties
- **Developer Guide:** Contact vendor (winspc.com); documentation provided post-sale
- **Standards:** COM/ActiveX architecture; specific REST API support not confirmed publicly
- **Authentication:** Proprietary per-installation configuration

### DataLyzer SPC
- **Description:** Web-based SPC, FMEA, MSA, APQP, and CAPA integrated suite; unique closed-loop AIAG core-tools data model.
- **API Documentation:** Not publicly published; vendor contact required
- **SDKs/Libraries:** Not published
- **Developer Guide:** https://datalyzer.com/products/spc-software/
- **Standards:** Web-based; REST/API capabilities not prominently documented
- **Authentication:** Web session; ERP integration details available on request

### NEXSPC 4.0
- **Description:** Web-based SPC platform with LLM-powered chart analysis, MQTT/TCP/REST IoT ingestion, and multi-language support; strong in Asian manufacturing markets.
- **API Documentation:** https://nexspc.com/blog/117 (Automated Data Collection: MQTT, TCP, REST APIs)
- **SDKs/Libraries:** Not published; native MQTT broker and REST API for ERP connectivity
- **Developer Guide:** https://nexspc.com/overview
- **Standards:** REST/JSON for ERP; MQTT (ISO/IEC 20922) for machine data; TCP Server for direct device connections
- **Authentication:** API key / session-based; private intranet deployment option

### QI Macros
- **Description:** Excel add-in for SPC charts, Gage R&R, and Six Sigma templates; offline, no API.
- **API Documentation:** N/A (Excel-only)
- **SDKs/Libraries:** N/A
- **Developer Guide:** https://www.qimacros.com/qi-macros/
- **Standards:** Microsoft Office COM add-in
- **Authentication:** N/A

### Statgraphics Centurion
- **Description:** Desktop/cloud statistical analysis suite with extensive SPC, regression, ANOVA, DOE, and AI-aided (StatAdvisor) interpretation.
- **API Documentation:** Not publicly published; REST API for cloud version not prominently documented
- **SDKs/Libraries:** Not published
- **Developer Guide:** https://www.statgraphics.com/
- **Standards:** Windows desktop (Centurion); cloud version available
- **Authentication:** Subscription-based; API details not public

### SQCpack (PQ Systems)
- **Description:** Affordable SPC platform for small and mid-size manufacturers; FDA 21 CFR Part 11 and IATF 16949 compliant; CMM and handheld gage integration.
- **API Documentation:** Not publicly published
- **SDKs/Libraries:** RS-232/USB gage connectivity; data import via Excel/Oracle
- **Developer Guide:** https://www.pqsystems.com/quality-solutions/statistical-process-control/SQCpack/
- **Standards:** SQL Server database; file-based import/export
- **Authentication:** User accounts with role-based access

---

## Notes

**Emerging areas:**
- ISO/TR 11462-5:2023 (quality data exchange format for SPC software) is a new standard that could enable genuine interoperability between SPC platforms; few vendors have announced support. An open-source implementation of this format would be a meaningful differentiator.
- The AIAG-VDA SPC Yellow Volume 2026 is in its final stakeholder review period (deadline May 3, 2026); SPC software will need to update capability metrics, terminology, and chart selection guidance to align with the final published version. This creates a near-term update cycle across all major vendors.
- OPC UA + MQTT as the standard IoT data ingestion path for SPC is well-established in theory but sparsely implemented in mainstream SPC tools; only NEXSPC prominently documents native MQTT connectivity.
- REST API coverage across SPC tools is inconsistent: Minitab Real-Time SPC and InfinityQS Enact publish REST API documentation; WinSPC and DataLyzer keep API details behind sales/support walls. An SPC platform with fully open, OpenAPI-documented REST and webhook APIs would address a clear gap.
