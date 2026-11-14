# Product Requirements Document (PRD): ASTRO-CARE (Autonomous Spaceflight Triage, Readiness & Operational Care)

| Document Metadata | Details |
| --- | --- |
| **Product Name** | ASTRO-CARE — Offline-First Crew Health & Mission Readiness Companion |
| **Challenge Alignment** | 2026 NASA Space Apps Challenge: *Health Monitoring Software for Astronauts on Space Missions*

|
| **Target Award Archetypes** | *Best Mission Concept* & *Best Use of Technology*

|
| **Architecture Pattern** | Local-First Monorepo Web Application (Zero-Internet / LAN Multi-User Sync) |
| **Primary Users** | Spaceflight Crew Members (Unified `astronaut` Role) |

---

## 1. Executive Summary & Mission Context

Deep-space missions beyond Low Earth Orbit (LEO)—such as Artemis lunar surface operations and future Mars transits—expose crews to NASA's five hazards of human spaceflight: **space radiation, isolation and confinement, distance from Earth, altered gravity fields, and hostile closed environments**. Because communication delays and blackouts prevent real-time tele-monitoring from Earth flight surgeons, astronauts—who are typically not physicians—must autonomously assess physiological deterioration, manage daily biological homeostasis, and evaluate crew readiness before high-risk operations.

**ASTRO-CARE** is a local-first, zero-internet monorepo web application designed to run on a spacecraft or planetary habitat Local Area Network (LAN). It combines an expressive, **Nintendogs-style interactive biometric avatar** driven by the clinical **NEWS2 (National Early Warning Score 2)** scale, rapid biological care logging, peer-to-peer crew surveillance, cognitive checks, and an environment-aware **GO / NO-GO Mission Readiness Engine** calibrated with real NASA Open Science Data Repository (OSDR) spaceflight datasets.

---

## 2. Recommended Monorepo Tech Stack

The stack is engineered for **100% offline execution** (zero external CDNs, zero cloud auth, zero internet dependencies) while supporting **real-time multi-device collaboration over a local habitat network (LAN)**.

| Layer | Technology Choice | Justification & Offline/LAN Strategy |
| --- | --- | --- |
| **Monorepo Orchestration** | **Turborepo + pnpm Workspaces** | Single repository housing frontend, local edge server, and shared clinical scoring logic with deterministic offline lockfiles (`pnpm-lock.yaml`). |
| **Frontend Framework** | **React 19 + TypeScript + Vite** | Fast bundle compilation with strict type safety across biometric schemas and clinical thresholds. |
| **Offline PWA & Local Storage** | **Vite PWA (Workbox) + Dexie.js (IndexedDB)** | Precaches all UI assets, character animations, and emergency SOPs locally. Queues offline mutations if a tablet loses habitat Wi-Fi in shielded modules. |
| **Interactive Avatar Engine** | **Rive (`@rive-app/react-canvas`)** *(Fallback: Layered SVG/CSS State Machine)* | Uses a visual **State Machine** to blend breathing speed, skin/visor oxygenation tint, posture, fatigue, and interactive care animations in real time. |
| **UI & Telemetry Charts** | **Tailwind CSS + Lucide Icons + Recharts** | High-contrast dark-mode aerospace HUD with clinical color-coding (Green/Amber/Red) and locally bundled time-series charts.

|
| **Local LAN Backend** | **Node.js + Fastify + WebSockets (`ws`)** | Lightweight local server hosted on the habitat's main terminal (`astro-care.local` via mDNS), broadcasting crew vitals and alerts across LAN clients in $<50\text{ ms}$. |
| **Embedded Database** | **SQLite (`better-sqlite3` in WAL mode)** | Zero-configuration, single-file relational database resilient to power cycles, storing crew vitals, care logs, and audit trails locally. |
| **Shared Clinical Engine** | **`@astro-care/core` (Pure TypeScript Package)** | Shared by client and server to compute NEWS2 scores, biological homeostasis decay, and environment-specific GO/NO-GO rules identically offline or on LAN. |

### Monorepo Directory Structure

```text
astro-care/
├── apps/
│   ├── web/                  # React 19 + Vite PWA (Nintendogs Avatar HUD, Crew Roster, GO/NO-GO Test)
│   └── habitat-server/       # Fastify + WebSocket + SQLite local LAN sync server & OSDR data seeder
├── packages/
│   ├── clinical-engine/      # NEWS2 calculator, homeostasis decay model, & Earth/Space/Moon GO/NO-GO rules
│   ├── shared-types/         # Zod schemas & TypeScript interfaces for vitals, care logs, & WS events
│   └── osdr-fixtures/        # Pre-processed NASA OSDR baseline profiles & anomaly simulation datasets
├── package.json
├── pnpm-workspace.yaml
└── turbo.json

```

---

## 3. Authentication & LAN Collaboration Architecture

### 3.1 Unified `astronaut` Role & Peer Access

* **Zero-Cloud Authentication:** Crew members authenticate locally against the habitat server using a **Crew ID + 4-Digit Tactical PIN** (issuing a local `HttpOnly` JWT).
* **Single `astronaut` Role:** All authenticated crew members hold the `astronaut` role. This flat peer-to-peer privilege model allows any crew member to:
* View real-time biometric dashboards and avatar states of all fellow crewmates.
* Perform **Buddy-Check Consciousness (ACVPU)** and **Cognitive Checklist** evaluations on a teammate during suspected hypoxia, decompression sickness, or acute illness.


* Trigger emergency countermeasure guides if a crewmate deteriorates.





### 3.2 Offline + LAN Sync Topology

1. **Nominal Habitat Operation (LAN Connected):** All crew tablets connect via local Wi-Fi/Ethernet to `habitat-server`. Every vital entry, care action, or peer evaluation writes to SQLite and immediately broadcasts via WebSocket (`CREW_STATE_UPDATED`) to all connected crew screens.
2. **Dead-Zone / RF-Shielded Operation (Isolated Offline):** If an astronaut enters a radiation shelter or rover outside LAN range, the PWA writes events to `Dexie.js` (IndexedDB) with monotonic timestamps and automatically reconciles with `habitat-server` upon reconnection.

---

## 4. Core Functional Requirements

### 4.1 Interactive Biometric Avatar ("Nintendogs" Paradigm) & NEWS2 Engine

The centerpiece of the primary dashboard is an interactive full-body astronaut companion avatar whose posture, facial expression, breathing animation, suit/vital aura, and micro-animations react dynamically to the crew member's **NEWS2 (National Early Warning Score 2)** physiological state.

#### NEWS2 Scoring Matrix (`@astro-care/clinical-engine`)

Each parameter is scored from `0` to `3` based on Royal College of Physicians clinical thresholds:

| Physiological Parameter | Score 3 (Red) | Score 2 (Amber) | Score 1 (Yellow) | Score 0 (Normal) | Score 1 (Yellow) | Score 2 (Amber) | Score 3 (Red) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Respiratory Rate** (breaths/min) | $\le 8$ | — | $9\text{--}11$ | $12\text{--}20$ | — | $21\text{--}24$ | $\ge 25$ |
| **$\text{SpO}_2$ Oxygen Saturation** (%) | $\le 91$ | $92\text{--}93$ | $94\text{--}95$ | $\ge 96$ | — | — | — |
| **Systolic Blood Pressure** (mmHg) | $\le 90$ | $91\text{--}100$ | $101\text{--}110$ | $111\text{--}219$ | — | — | $\ge 220$ |
| **Pulse / Heart Rate** (bpm) | $\le 40$ | — | $41\text{--}50$ | $51\text{--}90$ | $91\text{--}110$ | $111\text{--}130$ | $\ge 131$ |
| **Temperature** ($^\circ\text{C}$) | $\le 35.0$ | — | $35.1\text{--}36.0$ | $36.1\text{--}38.0$ | $38.1\text{--}39.0$ | $\ge 39.1$ | — |
| **Consciousness (ACVPU)** | **C, V, P, or U** | — | — | **Alert (A)** | — | — | — |

#### Avatar State Mapping & Clinical Triage Guidance

* **NEWS2 `0` (Optimal / Green):** Avatar stands upright with steady breathing animation, alert gaze, and calm posture.
* **NEWS2 `1–4` (Low Risk / Yellow):** Avatar exhibits subtle fatigue, heavier chest excursion (synced to actual respiratory rate), and prompts self-assessment or hydration/rest.
* **Single Parameter = `3` OR NEWS2 `5–6` (Medium Risk / Amber):** Avatar kneels or holds chest/head, ambient visor shifts amber, and an **Actionable Countermeasure Drawer** opens explaining *why* the metric is abnormal and step-by-step stabilization procedures (e.g., supplemental $\text{O}_2$ check, cabin $\text{CO}_2$ scrubber verification, fluid bolus).


* **NEWS2 $\ge 7$ (High Risk / Red Alert):** Avatar collapses/enters distress state, triggers an audible/visual LAN alert across all crew devices, and launches an emergency triage checklist for fellow astronauts.



#### Consciousness (ACVPU) Evaluation Modes

Because an impaired astronaut cannot reliably self-diagnose acute confusion (`C`) or lethargy (`V/P/U`), the Consciousness parameter supports two input flows:

1. **Peer Crewmate Verification:** Another authenticated astronaut selects the teammate's profile and logs their observed ACVPU status (`Alert`, `New Confusion`, `Responds to Voice`, `Responds to Pain`, `Unresponsive`).
2. **Optional Neuro-Cognitive Mini-Game (PVT-Lite):** A 15-second **Psychomotor Vigilance & Stroop Reaction Mini-Game** built into the avatar screen. Severe latency spikes ($>500\text{ ms}$ above personal baseline) or repeated accuracy errors automatically flag potential `New-Onset Confusion (C)` (`+3` NEWS2 points) and prompt a peer buddy-check.

---

### 4.2 Interactive Care & Biological Homeostasis Action Dock

Directly beneath the avatar, five tactile care buttons—modeled on pet-care interaction loops—let astronauts log daily biological inputs and trigger immediate avatar feedback animations:

| Action Button | Logged Data & Quick Inputs | Spaceflight Physiological Rationale (HRP Risk) | Avatar Interaction & Visual Feedback |
| --- | --- | --- | --- |
| **Food** | Meal type, caloric intake ($\text{kcal}$), electrolytes | Mitigates *Risk of Inadequate Food & Nutrition* and spaceflight-induced appetite suppression.

| Avatar consumes ration pack; nutrition meter replenishes; happy gesture. |
| **Water** | Volume ($\text{mL}$) in $100\text{ mL}$ / $250\text{ mL}$ increments | Counters microgravity diuresis, plasma volume loss, and orthostatic intolerance.

| Avatar drinks from pouch; hydration bar fills; skin turgor/aura brightens. |
| **Physical Activity** | Duration ($\text{min}$), modality (ARED resistive, cycle, treadmill) | Counters *Spaceflight-Induced Bone Changes* and *Reduced Cardiorespiratory Fitness*.

| Avatar performs floating resistance/cardio animation; cardiovascular resilience boosts. |
| **Defecation** | Bristol Stool Scale (`1–7`), timestamp | Altered GI motility, fluid shifts, and host-microbiome changes in space can cause severe constipation or GI distress.

| Relieves abdominal discomfort animation; resets GI transit timer; flags $>72\text{ h}$ absence. |
| **Urination** | Volume estimate / color scale (`1–8` hydration chart) | Tracks renal clearance, dehydration, and hypercalciuria-driven renal stone risk in microgravity. | Relieves bladder indicator; cross-checks output against water intake balance. |

---

### 4.3 Crew Status Matrix & Cognitive Evaluation Checklist

* **Crew Roster Button (`CREW STATUS`):** Displays live compact avatar cards for every crew member on the LAN, sorted by NEWS2 severity, showing last vital timestamp, hydration/nutrition bars, and active alerts.
* **Cognitive & Behavioral Evaluation Checklist:** Accessible inside any crew member's detail view (for self-check or peer assessment) to address *Risk of Behavioral Changes* and *Inadequate Sleep*:


* Sleep duration & quality in the last $24\text{ h}$ ($<6\text{ h}$ flags circadian fatigue).


* Subjective mood, irritability, and isolation stress scale (`1–5`).


* Fine motor & vestibular stability check (dizziness, hand-eye coordination, visual/SANS symptoms such as near-vision blur or headaches).





---

### 4.4 Environment-Aware "GO / NO-GO" Readiness Test Engine

A prominent **`MISSION READINESS TEST`** button evaluates whether an astronaut is medically and cognitively cleared (`GO` or `NO-GO`) before performing a critical task (e.g., Extravehicular Activity / spacewalk, rover driving, docking, or habitat maintenance) in one of three selectable operational environments:

| Selectable Environment | Gravity & Operational Hazards | Specific GO / NO-GO Gating Rules |
| --- | --- | --- |
| **1. Earth ($1g$)** | Baseline terrestrial gravity; analog training or post-landing recovery.

| • **GO:** NEWS2 $= 0\text{--}2$ (no single score of $3$), Cognitive Checklist passed, hydration $\ge 70\%$.<br>

<br>• **NO-GO:** NEWS2 $\ge 3$ or acute fatigue/vestibular impairment. |
| **2. Space ($0g$ Microgravity)** | Orbital/deep-space transit; cephalad fluid shift, decompression sickness (DCS) risk during EVA suit depressurization, $\text{CO}_2$ buildup, hypoxia.

| • **GO:** NEWS2 $= 0$ (strict baseline required), $\text{SpO}_2 \ge 97\%$, resting HR $45\text{--}95\text{ bpm}$, zero DCS/vestibular flags, cognitive checklist $100\%$ clear, hydration & nutrition $\ge 80\%$ in past $12\text{ h}$.<br>

<br>• **NO-GO:** Any NEWS2 $\ge 1$, $\text{SpO}_2 \le 96\%$ (elevated EVA hypoxia/DCS risk), or missed hydration/elimination thresholds.

|
| **3. Moon ($1/6g$ Lunar Surface)** | Partial gravity; high metabolic exertion in pressurized surface suits, lunar dust exposure, cardiovascular orthostatic stress, solar radiation.

| • **GO:** NEWS2 $\le 1$ (no respiratory or BP deviations), Systolic BP $105\text{--}140\text{ mmHg}$ (orthostatic tolerance), recent physical countermeasure compliance, cognitive reaction score within $15\%$ of baseline.<br>

<br>• **NO-GO:** Systolic BP $\le 104\text{ mmHg}$ (syncope risk under $1/6g$ load), elevated respiratory rate ($>20\text{ bpm}$), or unlogged bowel/bladder clearance prior to $6\text{--}8\text{ h}$ suit lock-in. |

**Output:** Generates a timestamped **Flight Readiness Certificate (`GO` / `NO-GO`)** displaying the exact pass/fail breakdown by parameter and recommended corrective actions if `NO-GO` is issued.

---

## 5. NASA Open Science Data Repository (OSDR) Integration

To anchor the clinical thresholds, avatar state transitions, and demo simulation mode in real aerospace biology rather than arbitrary numbers, ASTRO-CARE integrates pre-bundled, offline-accessible reference profiles derived from **NASA OSDR** (including Ames Life Sciences Data Archive human spaceflight and analog datasets):

| OSDR Dataset / Study | Data Type Available on OSDR | How ASTRO-CARE Uses It |
| --- | --- | --- |
| **Inspiration4 Multi-Omics & Biometric Datasets (`OSD-570` through `OSD-575`)** | Continuous wearable telemetry (heart rate, $\text{SpO}_2$, skin temperature, activity), Cognition test battery performance, and clinical blood/urine biomarkers across pre-flight, in-flight ($0g$), and post-flight ($1g$). | • Calibrates normal in-flight resting heart rate and $\text{SpO}_2$ shifts between **Earth ($1g$)** and **Space ($0g$)** modes.<br>

<br>• Provides reference reaction-time variance for the **Cognitive Mini-Game**.<br>

<br>• Seeds realistic multi-day crew demo profiles for hackathon judging. |
| **NASA/ESA/DLR Head-Down Tilt (HDT) Bed Rest Studies (e.g., `AGBRESA` / `VaPER` in OSDR/ALSDA)** | Longitudinal cardiovascular metrics (systolic/diastolic blood pressure, orthostatic tolerance), fluid intake/urine output balance, and neuro-ocular (SANS) assessments under simulated microgravity and artificial gravity. | • Defines the **Moon ($1/6g$) & Space ($0g$) Systolic BP** thresholds in the GO/NO-GO engine to prevent orthostatic syncope.<br>

<br>• Models the **Water Intake vs. Urination** homeostasis decay curve. |
| **ISS Spaceflight Microbiome, Nutrition & Metabolic Studies (e.g., `OSD-252`, `OSD-389`, `OSD-530`)** | Crew dietary intake logs, fecal/urine sample metadata, GI transit shifts, and immune/inflammatory markers during long-duration ISS missions. | • Calibrates expected caloric/hydration targets and flags abnormal **Defecation (Bristol scale)** or **Urination** intervals linked to spaceflight GI/renal risks.

|
| **Spaceflight Radiation & Dosimetry Datasets (OSDR / RadLab)** | Cabin and EVA radiation dose rates across LEO and deep-space missions.

| • Adds a passive environmental radiation exposure modifier to the **Space** and **Moon** GO/NO-GO risk briefing. |
