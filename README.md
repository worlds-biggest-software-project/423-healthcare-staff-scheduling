# Healthcare Staff Scheduling

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native platform for nurse and physician scheduling that unifies credentialing, constraint-based shift assignment, and overtime compliance into a single system.

Healthcare organizations face a uniquely punishing scheduling problem: 24/7 coverage requirements intersect with complex credentialing rules, scope-of-practice regulations, union contracts, fatigue mandates, and chronic staffing shortages. Getting a schedule wrong carries direct patient safety consequences and significant legal and financial liability. This project builds an open alternative to the proprietary platforms that dominate the space today, applying AI to demand forecasting, fatigue-optimized scheduling, and burnout prevention where incumbents rely on static rules.

---

## Why Healthcare Staff Scheduling?

- **Every incumbent is proprietary and expensive.** QGenda, symplr, NurseGrid, and Petal are all closed-source SaaS products. No open-source alternative exists for healthcare-specific scheduling with credentialing integration.
- **Incumbents are fragmented by role.** QGenda and symplr focus on physicians and advanced practice providers; NurseGrid serves only nurses; Shift MedStaff handles only agency/per-diem staff. No single platform manages the full workforce continuum from employed clinicians through float pool to agency staff.
- **Credentialing and scheduling remain siloed.** Most organizations run separate credentialing and scheduling systems, creating the credential-to-schedule mismatches that generate liability. Only symplr attempts unification, and its nursing coverage is limited.
- **AI adoption is shallow.** Petal claims AI-powered scheduling but most platforms still rely on rules-based engines. None offer predictive burnout detection, fatigue-optimized shift design, or AI-driven demand forecasting based on patient acuity and admission patterns.
- **Agency and overtime spend is enormous.** Health systems spend billions annually on premium labor at 2-3x the cost of employed staff per hour. Better scheduling and float-pool management can directly reduce this spend, but current tools lack the optimization depth to capture the savings.

---

## Key Features

### Constraint-Based Schedule Generation

- Automated schedule creation respecting minimum rest periods, maximum consecutive hours, shift-length regulations, FTE targets, and overtime thresholds
- Union contract rule enforcement built into the constraint engine
- Fair shift distribution algorithms balancing desirable and undesirable shifts across staff
- Scheduler decision support explaining trade-offs and feasibility of generated schedules

### Credentialing and Privilege Management

- End-to-end credentialing lifecycle: initial credentialing, primary source verification, privileging, re-credentialing, and expiration-driven renewals
- Credential-to-shift matching preventing scope-of-practice violations at assignment time
- Multi-facility privilege management ensuring credential validity per location and service line
- Automated alerts when assignments would exceed a clinician's credentialed scope

### Staffing Operations

- Acuity-based staffing ratios integrating with patient census data to generate dynamic staffing targets
- Open-shift marketplace with self-service pickup, rule-based eligibility filtering, and manager approval
- Float-pool and agency staff management with cost-tier tracking and automatic preference ordering (employed first, then float, then agency)
- Real-time rescheduling when staff call out, with automated reassignment recommendations

### Staff Experience and Retention

- Mobile-first interface for schedule viewing, availability submission, time-off requests, and shift swaps
- Nurse self-service schedule control and shift pickup
- Burnout risk metrics and staff satisfaction tracking by unit and role
- Equitable shift distribution tools with DEI auditing capabilities

### Analytics and Workforce Planning

- Historical demand modeling and predictive staffing for seasonal variation and surge events
- Turnover and retention metrics by unit, role, and shift pattern
- Overtime and compliance dashboards with proactive manager alerts before violations occur
- Time and attendance integration with variance reporting against scheduled hours

---

## AI-Native Advantage

This platform applies machine learning where incumbents rely on static rules and manual judgment. Demand forecasting models predict staffing needs by shift and unit based on seasonal patterns, predicted admissions, and patient acuity trends. Fatigue-optimized scheduling algorithms minimize circadian disruption and consecutive-shift burden beyond what regulatory minimums require. Burnout risk prediction identifies staff at high risk before they leave, enabling preventive schedule adjustments and retention interventions. Call-out prediction models estimate no-show likelihood, enabling proactive overstaffing of high-risk shifts rather than reactive scrambling.

---

## Tech Stack & Deployment

The scheduling engine solves an NP-hard combinatorial optimization problem. Real-world instances with hundreds of staff, dozens of shift types, and thousands of simultaneous constraints require heuristic or metaheuristic solvers (genetic algorithms, integer programming) producing near-optimal schedules within acceptable compute time.

Multi-system integration is essential: the platform connects to EHRs (Epic, Cerner, Athena) for census and acuity data, HRIS for employment and pay-rate data, payroll systems for hours export, and credentialing databases including licensing boards, the NPDB, DEA, and specialty certification bodies. Deployment must support both cloud-hosted and self-hosted configurations to meet the data residency and security requirements of healthcare organizations.

---

## Market Context

Healthcare workforce management software is a multi-billion-dollar market driven by persistent nursing and physician shortages, growing regulatory scrutiny of staffing ratios (California's nurse-to-patient ratio mandate has inspired similar legislation in several states), and the high cost of agency labor at 2-3x employed staff rates. Target customers include hospital systems with multiple facilities and complex float-pool requirements, large multispecialty physician groups, academic medical centers with GME scheduling needs, and healthcare staffing agencies. The consolidation of scheduling and credentialing into unified platforms is the dominant strategic direction, as demonstrated by symplr and QGenda acquisitions.

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
