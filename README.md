# Statistical Process Control

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for real-time SPC charting, control plan management, and corrective actions across manufacturing operations.

Statistical Process Control (SPC) is the discipline manufacturers use to monitor production variability, demonstrate process capability, and prove conformance to standards such as IATF 16949, ISO 9001, AS9100, and FDA 21 CFR Part 820. This project aims to deliver a modern, real-time, AI-augmented SPC platform for quality engineers, plant managers, and shop-floor operators in automotive, aerospace, medical device, semiconductor, and food manufacturing settings.

---

## Why Statistical Process Control?

- Industry-reference incumbents like Minitab carry steep learning curves and are not focused on real-time production-floor monitoring, while enterprise platforms such as InfinityQS ProFicient and WinSPC use opaque custom pricing that often reaches tens of thousands annually for multi-line deployments.
- Affordable tools (QI Macros at ~$279, SPC XL, SQCpack from ~$329) live inside Excel or single-user desktops, with no real-time data feeds, automated monitoring, or multi-user collaboration.
- Existing platforms detect rule violations only after they occur; predictive, ML-based early warning before Western Electric or Nelson rules fire is not mainstream in any incumbent product.
- Most vendors provide proprietary, closed integrations; an open, well-documented API with SDKs for popular languages is absent from incumbents such as WinSPC (COM/ActiveX-based) and Minitab Real-Time SPC (data-streaming REST only).
- No open-source SPC platforms at enterprise scale exist today, leaving small and mid-size manufacturers with one or two production lines no affordable real-time option.

---

## Key Features

### Control Charting and Capability Analysis

- Variables charts: Xbar-R, Xbar-S, I-MR, I-MR-R/S, EWMA
- Attributes charts: P, NP, C, U
- Western Electric and Nelson rules with automatic alarm on violation
- Process capability indices: Cp, Cpk, Pp, Ppk with histograms and normal probability plots
- Phase I (limits derived from data) and Phase II (data vs established standard) studies

### Real-Time Data Collection and Alerting

- Keyboard entry, gage USB/RS-232, and REST API ingestion at MVP
- IoT ingestion via MQTT and OPC-UA connectors in v1.1
- Configurable in-app, email, and webhook alerts on rule violation
- Assignable cause capture at point of alert with corrective action workflow

### Compliance and Reporting

- Audit trail with time-stamped record of all data entries and changes, suitable for FDA 21 CFR Part 11 and IATF 16949
- Multi-user web interface with role-based access (operator, engineer, manager, admin)
- Export to PDF and Excel for customer-submission reports
- Terminology and reporting alignment with IATF 16949, ISO 9001, AS9100, and the AIAG-VDA SPC Yellow Volume 2026

### Quality Planning Integration

- Control Plan management linked to SPC chart configuration
- Gage R&R (MSA) module with AIAG-compliant calculations
- Multi-plant enterprise dashboard with cross-line comparison
- Mobile-optimised operator data entry and alert acknowledgement

### AI-Augmented Analytics (backlog)

- ML-based predictive drift detection (LSTM / EWMA ensemble) with early warning before rule violations
- LLM-powered chart interpretation with root-cause hypotheses and management recommendations
- Automated control plan generation from FMEA risk priorities and historical Cpk data
- Lead-Lag Correlation Analysis across upstream process parameters and downstream quality results
- Cross-plant benchmarking and best-practice propagation recommendations

---

## AI-Native Advantage

Unlike incumbents that detect special-cause variation only after rules trigger, this project applies ML models to process data to detect pre-failure patterns before Western Electric or Westgard rules fire, enabling proactive adjustment instead of reactive containment. LLM-driven conversational analytics let plant managers query defect trends, Cpk degradation, and out-of-control events in natural language without navigating BI dashboards. Automatic special-cause classification proposes corrective actions from historical patterns, and AI-driven control plan generation recommends monitoring frequencies, control limits, and reaction plans grounded in process capability data and FMEA risk priorities.

---

## Tech Stack & Deployment

The platform targets a multi-user web interface with role-based access for operators, engineers, managers, and administrators. Real-time data ingestion follows the patterns established by NEXSPC and InfinityQS: REST API for ERP/SAP integration, MQTT for millisecond-level machine bursts, and TCP/RS-232/USB for direct gage and scale connectivity. OPC-UA support is planned for v1.1. Architecture aligns with the AIAG-VDA SPC Yellow Volume 2026, IATF 16949, ISO 9001, AS9100, FDA 21 CFR Part 820 / ISO 13485, and the AIAG MSA Manual. Both cloud and private-intranet deployment are envisaged so regulated industries can maintain full data isolation.

---

## Market Context

The SPC software market is projected at approximately USD 1.5 billion by 2026, growing at roughly 10% CAGR from 2021, driven by automotive, aerospace, and medical device manufacturing and the global spread of Six Sigma (Verified Market Research, 2026; Spherical Insights, 2026). Pricing across incumbents ranges from one-time licences around $279–$329 (QI Macros, SQCpack single-user) to SaaS subscriptions at $99–$329 per user per month, with enterprise real-time platforms (InfinityQS, WinSPC) reaching tens of thousands annually under custom pricing. Primary buyers are quality engineers and managers, Six Sigma Black Belts, plant managers, and supply quality engineers in automotive, aerospace, semiconductor, pharmaceutical, and food manufacturing.

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
