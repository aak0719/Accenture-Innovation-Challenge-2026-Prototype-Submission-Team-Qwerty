# PatientTriage.ai

An intelligent, real-time triage and resource allocation dashboard for Indian
emergency departments. It scores incoming patients by severity using an
age-aware, 4-step deterministic clinical scoring engine, surfaces its own
confidence, recommends bed allocation live, adapts behavior under surge
conditions, and keeps a full audit trail of clinician overrides.

Built for the Accenture Innovation Challenge 2026 — Problem Track 2
(PatientTriage.ai).


## Table of contents

- What it does
- Solution architecture
- Requirements
- Installation
- Configuration
- Sample data / CSV import format
- Known limitations
- Future work
- Maintainers


## What it does

- **Patient intake** — via a structured manual form, or "Magic Parse": a
    built-in natural-language pattern matcher that converts unstructured
    free-text clinical notes into structured demographics and vitals
    automatically.
- **4-step triage scoring engine:**
    1. **Immediate life-threat bypass** — keyword/AVPU-based check for
        catastrophic presentations (e.g. cardiac arrest, unresponsive
        state); if matched, instantly assigned Blue Tier at max score.
    1. **Age-split physiological scoring** — adult scoring (age ≥12) vs.
        pediatric scoring across 6 age brackets (age <12), each with
        distinct vital-sign thresholds.
    1. **Universal vitals scoring** — SpO2, temperature, and AVPU
        consciousness state scored independently of age.
    1. **Symptom-specific pathology overrides** — keyword detection on
        chief complaint that can force a tier (e.g. ACS, stroke,
        respiratory distress pathways) regardless of the raw numeric
        score.
- **Tier & confidence output** — final Red / Orange / Yellow / Green tier
    with an associated target wait time, plus a calculated confidence
    percentage. Confidence below 80% triggers a warning badge requiring
    clinician verification.
- **Resource allocation** — a downward-cascade protocol matches each tier
    to a prioritized sequence of bed types (Critical / Emergency / General
    / Chairs) based on live free-bed counts, with saturation handling when
    no beds are available.
- **Surge management** — a continuously calculated Surge Ratio (queue
    length + occupied beds ÷ active doctors) drives 4 operational surge
    levels. When beds are fully saturated during a surge, a Fast-Track
    Express Lane reroutes Green/Yellow patients to an ambulatory consult
    lane, preserving acute beds for higher-severity patients.
- **Clinician override & audit trail** — every patient supports three
    terminal actions: *Disagree & Choose Bed* (override with mandatory
    justification note), *Left / Checked Elsewhere*, and *Consulted
    Without Bed*. All overrides are logged with clinician reasoning,
    timestamp, and the original AI recommendation.
- **Zero-history vs. returning patients** — if a prior discharge summary
    exists for a patient, it's factored into the current assessment (e.g.
    flagged history of a cardiac event); if not, scoring is based purely
    on observed vitals and symptoms at intake.
- **CSV batch import** — allows bulk upload of multiple simulated
    patients at once, used in this prototype to simulate multiple
    simultaneous intake points feeding the live queue.
- **Live bed/chair status tracking** — Critical/Emergency beds are
    designed to auto-toggle free/occupied via network integration where
    available; General beds and chairs use a manual nurse-operated
    toggle. Both feed the same live count used by the resource-cascade
    logic.


## Solution architecture

PatientTriage.ai is built as a lightweight, client-side single-page
application — there is no backend server or database in this prototype.

- **Front end:** Semantic HTML5, styled with the Tailwind CSS utility
    framework for a fully responsive, color-coded clinical interface.
- **State management & reactivity:** Alpine.js handles the core scoring
    logic, live timers, and dynamic DOM updates, so bed changes and queue
    re-sorting update instantly without a page reload or server round
    trip.
- **Scoring logic:** A deterministic, hardcoded rule engine (see
    [Working_Mechanism_Of_Triage_Tool](Working_Mechanism_Of_Triage_Tool.docx)
    for the full mathematical specification) — not a trained machine
    learning model.
- **Magic Parse:** A built-in natural-language pattern matcher
    (regex-based) that extracts age, gender, and vitals from free-text
    clinical notes. This prototype was developed with the assistance of
    Google Gemini as a coding tool, but Magic Parse and the scoring engine
    do not call any AI model or external API at runtime.
- **Data handling:** Patient data and queue state are held entirely in
    the browser's memory for the duration of the session; nothing is
    persisted server-side. Bulk patient intake is handled via an
    integrated client-side CSV parser.

**Data flow (high level):**
```
Patient intake (form or Magic Parse, both client-side)
        ↓
Scoring Engine (4-step deterministic rule hierarchy) → Tier + Confidence
        ↓
Resource Cascade Logic → Bed Recommendation (live bed counts)
        ↓
Dashboard Queue (with Surge Ratio monitoring)
        ↓
Clinician Review → Accept or Override (logged)
```


## Requirements

This prototype requires no local build step, server, or package
installation. All dependencies are loaded automatically from a CDN when
`index.html` is opened:

- [Tailwind CSS](https://tailwindcss.com/) — responsive UI styling and
    custom triage color themes
- [Alpine.js](https://alpinejs.dev/) — reactive state management, data
    binding, and the triage algorithm's execution
- [Font Awesome 6.4](https://fontawesome.com/) — clinical and operational
    iconography

A modern web browser (Chrome, Edge, Safari, or Firefox) is the only local
requirement.


## Installation

1. Clone this repository, or download the source as a `.zip` file and
    extract it.
1. Locate the `index.html` file in the root directory.
1. Open it in any modern web browser — double-clicking the file works,
    since no local server is required.


## Configuration

The dashboard has no external config files or environment variables to
set up. All configuration happens in the running app:

1. On load, click the **Resources** button (top right).
1. Set the hospital's available Critical, Emergency, and General beds,
    Chairs, and active Staff count.
1. These values drive the resource-cascade recommendations and the Surge
    Ratio calculation for the rest of the session.

To test the engine:

- Click **New Intake** to manually add a patient, or paste a free-text
    clinical note (e.g. *"34 yr old male, chest pain, HR 115"*) to test
    Magic Parse.
- Click **Import CSV** and upload the sample patient dataset (see below)
    to simulate a mass-casualty surge and watch the cascade logic in
    action.
- Use the `Allot Bed`, `Override`, and `Fast-Track` buttons to manage the
    queue and watch the Surge Level banner update live.


## Sample data / CSV import format

The dashboard supports bulk patient import via CSV. Column headers are
case-insensitive and accept common synonyms.

| Column | Accepted synonyms | Description |
|---|---|---|
| `id` | `patient_id`, `uhid` | Unique patient identifier |
| `name` | `patient_name`, `fullname` | Patient full name |
| `age` | `patient_age` | Age in years (drives pediatric/adult scoring) |
| `gender` | `sex` | Male / Female / Other |
| `chief_complaint` | `complaint`, `symptoms`, `reason` | Free-text presenting complaint |
| `heart_rate` | `hr`, `pulse` | Beats per minute (optional) |
| `resp_rate` | `rr`, `respiratory_rate` | Breaths per minute (optional) |
| `spo2` | `oxygen`, `saturation` | Oxygen saturation % (optional) |
| `systolic_bp` | `sbp`, `bp`, `blood_pressure` | Systolic BP in mmHg (optional) |
| `temp` | `temperature` | Body temperature in °C (optional) |
| `avpu` | `consciousness` | Alert / Voice / Pain / Unresponsive |
| `additional_flags` | `history`, `flags` | Optional history/hazard keywords (e.g. `cardiac`, `stroke`) |

**Example row:**
```
id,name,age,gender,chief_complaint,heart_rate,resp_rate,spo2,systolic_bp,temp,avpu,additional_flags
P2000,Rajesh Kumar,34,Male,Cardiac arrest,38,6,68,62,35.0,U,known hypertensive
```

A full sample dataset of 20 simulated patient records is included at
`Simulant_Patient_Record_Dataset.csv`.


## Known limitations

This is a proof-of-concept built for Round 2 of the Accenture Innovation
Challenge, on illustrative/simulated data. It is not production-grade.
Notably:

- **Magic Parse and the scoring engine are rule-based, not AI-driven at
    runtime.** Both run entirely on hardcoded regex patterns and
    deterministic clinical thresholds in the browser — there is no live
    call to Gemini or any other model. Google Gemini was used as a coding
    assistant during development, not as a runtime dependency. Because
    Magic Parse relies on fixed patterns, it may miss vitals or symptoms
    phrased outside its expected formats.
- **Intake is single-device in this build.** In production, each ED entry
    point would have its own connected device with real-time sync to a
    shared queue; here, single-patient manual entry and CSV batch upload
    simulate that distributed workflow.
- **Bed status auto-toggling is conceptual for Critical/Emergency beds**
    — this prototype assumes integration with hospital bed-monitoring
    hardware where available; General beds/chairs use a manual toggle,
    which is implemented.
- **No integration with real hospital records systems** — patient history
    (discharge summaries) is simulated for demonstration rather than
    pulled from a live EHR.
- **Regulatory compliance (DPDP Act) is designed for, not enforced in
    code** — consent handling, data retention windows, and deletion
    policies are architectural decisions documented in the business
    proposal, not yet implemented as automated system behavior in this
    prototype.
- **Surge handling is currently limited to Fast-Track rerouting.** The
    dashboard is already built to handle a worst-case 3x surge scenario
    without breaking, but the queue mechanism itself doesn't otherwise
    change behavior under surge — the only surge-specific adaptation is
    offering the Fast-Track lane to Green/Yellow patients once beds
    saturate. Further surge-specific logic (e.g. dynamic
    re-prioritization, staff reallocation suggestions) is a planned area
    of improvement.
- **No server-side persistence.** All patient and queue data lives in
    browser memory for the session and is lost on page reload — there is
    no database in this prototype.


## Future work

- **Train a custom ML model** on available clinical/triage data to power
    more nuanced scoring and free-text understanding, moving beyond the
    current fixed rule-based/regex logic.
- **Strengthen surge-specific behavior** beyond the current Fast-Track
    mechanism — e.g. dynamic queue re-prioritization or staff/resource
    reallocation suggestions during high-surge periods.
- **Live multi-device intake sync** replacing the current single-device
    /CSV-batch simulation.
- **Real hospital system integration** for bed sensors and patient
    records (EHR), including server-side persistence.


## Maintainers

Team Qwerty:

- Alfia
- Eeshaa
- Geetanshi
