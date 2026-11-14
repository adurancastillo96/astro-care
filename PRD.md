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
| **Offline PWA & Local Storage** | **Vite PWA (Workbox) + Dexie.js (IndexedDB)** | Precaches all UI assets, local `.glb` 3D avatar models, and emergency SOPs locally. Queues offline mutations if a tablet or phone loses habitat Wi-Fi in shielded modules. |
| **Interactive Avatar Engine** | **Three.js + React Three Fiber (`@react-three/fiber`, `@react-three/drei`) + `.glb`** *(Fallback: Lightweight Telemetry Card)* | Uses a local `.glb` astronaut character with facial **morph targets (blendshapes)**, dynamic skin/visor material tinting, breathing speed synced to respiratory rate, and interactive care animation clips driven by normalized clinical props. |
| **UI & Telemetry Charts** | **Tailwind CSS + Lucide Icons + Recharts** | High-contrast dark-mode aerospace HUD with clinical color-coding (Green/Amber/Red) and locally bundled time-series charts.

|
| **Local LAN Backend** | **Node.js + Fastify + WebSockets (`ws`)** | Lightweight local server hosted on the habitat's main terminal (`astro-care.local` via mDNS), broadcasting crew vitals and alerts across LAN clients in $<50\text{ ms}$. |
| **Embedded Database** | **SQLite (`better-sqlite3` in WAL mode)** | Zero-configuration, single-file relational database resilient to power cycles, storing crew vitals, care logs, and audit trails locally. |
| **Shared Clinical Engine** | **`@astro-care/clinical-engine` (Pure TypeScript Package)** | Shared by client and server to compute NEWS2 scores, biological homeostasis decay, normalized 3D avatar telemetry signals, and environment-specific GO/NO-GO rules identically offline or on LAN. |

### Monorepo Directory Structure

```text
astro-care/
├── apps/
│   ├── web/                  # Mobile-first React 19 + Vite PWA (3D .glb Avatar HUD, Thumb-Zone Care Dock, Crew Roster, GO/NO-GO Test)
│   │   └── public/avatars/   # Local precached .glb 3D astronaut character & care props (100% offline)
│   └── habitat-server/       # Fastify + WebSocket + SQLite local LAN sync server & OSDR data seeder
├── packages/
│   ├── clinical-engine/      # NEWS2 calculator, homeostasis decay, Avatar Telemetry Mapper, & Earth/Space/Moon GO/NO-GO rules
│   ├── shared-types/         # Zod schemas & TypeScript interfaces for vitals, care logs, & WS events
│   └── osdr-fixtures/        # Pre-processed NASA OSDR baseline profiles & anomaly simulation datasets
├── package.json
├── pnpm-workspace.yaml
└── turbo.json

```

---

## 3. Authentication & LAN Collaboration Architecture

### 3.1 Unified `astronaut` Role, Mobile Login UX & Peer Access

* **Zero-Cloud Authentication:** Crew members authenticate locally against the habitat server using a **Crew ID + 4-Digit Tactical PIN** (issuing a local `HttpOnly` JWT).
* **Mobile Login Flow (Zero OS Keyboard Friction):**
  * Displays a **tap-to-select Crew Card grid** (populated from pre-seeded NASA OSDR/crew profiles) paired with a large **4-digit tactile PIN keypad** (`>= 48×48px` keys) so mobile users never fight soft keyboards.
  * Includes an explicit **Arm Audio Alerts** toggle on the login action so the sign-in tap unlocks the browser Web Audio API for emergency alarms.
* **Single `astronaut` Role:** All authenticated crew members hold the `astronaut` role. This flat peer-to-peer privilege model allows any crew member to:
  * View real-time biometric dashboards and avatar states of all fellow crewmates.
  * Perform **Buddy-Check Consciousness (ACVPU)** and **Cognitive Checklist** evaluations on a teammate during suspected hypoxia, decompression sickness, or acute illness.
  * Trigger emergency countermeasure guides if a crewmate deteriorates.





### 3.2 Offline + LAN Sync Topology

1. **Nominal Habitat Operation (LAN Connected):** All crew tablets connect via local Wi-Fi/Ethernet to `habitat-server`. Every vital entry, care action, or peer evaluation writes to SQLite and immediately broadcasts via WebSocket (`CREW_STATE_UPDATED`) to all connected crew screens.
2. **Dead-Zone / RF-Shielded Operation (Isolated Offline):** If an astronaut enters a radiation shelter or rover outside LAN range, the PWA writes events to `Dexie.js` (IndexedDB) with monotonic timestamps and automatically reconciles with `habitat-server` upon reconnection.

---

## 4. Core Functional Requirements

### 4.1 Interactive 3D Biometric Avatar ("Nintendogs" Paradigm) & NEWS2 Engine

The centerpiece of the primary dashboard is a friendly, interactive **3D `.glb` astronaut companion avatar** (rendered via Three.js + `@react-three/fiber` + `@react-three/drei`) whose facial morph targets, skin/visor material shaders, breathing animation, posture, and care reactions are **directly bound to the crew member's live physiological telemetry and NEWS2 (National Early Warning Score 2) state**—never hardcoded to static UI states.

Rather than binding raw clinical numbers inside React components, `@astro-care/clinical-engine` exposes a pure **Avatar Telemetry Mapper** that transforms vitals and homeostasis decay into normalized 3D driver props (`feverIntensity`, `hypoxiaIntensity`, `breathingHz`, `distressLevel`, `energyAura`, and `careTrigger`).

#### Biometric-to-Avatar Binding Contract

| Astronaut Metric / Event | Normalized 3D Driver | Visual & Interactive 3D Avatar Response |
| --- | --- | --- |
| **Temperature** ($\ge 38.1^\circ\text{C}$) | `feverIntensity` (`0..1`) | Flushed red facial skin shader, brow furrow morph target, sweat/heat particles; prompts hydration check. |
| **$\text{SpO}_2$ Oxygen Saturation** ($\le 93\%$) | `hypoxiaIntensity` (`0..1`) | Cool cyanotic skin/lip tint, visor rim shifts blue-amber, labored chest heave & chest-clutch posture. |
| **Respiratory Rate** (breaths/min) | `breathingHz` (`RR / 60`) | Ribcage/suit chest scale oscillation and HUD pulse cadence synced to actual respiratory rate. |
| **Pulse / BP / NEWS2 Total** | `distressLevel` (`0..3`) | Smooth skeletal blend from upright alert (`0`), fatigued slouch (`1`), kneeling brace (`2`), to distress collapse (`3`). |
| **Hydration & Nutrition Balance** | `energyAura` (`0..1`) | Suit chest-emblem & aura ring brightness dims as homeostasis decays; replenishes immediately when care is logged. |
| **5 Care Interactions** | `careTrigger` (`feed`, `drink`, `exercise`, `defecate`, `urinate`) | Tapping the avatar or Care Dock fires friendly 3D reaction clips (drinking from pouch, eating ration, ARED exercise, relief/happy nod) while persisting the log. |

#### Vitals Ingestion & Judge Simulation Mode

To avoid tedious manual form-filling on mobile while keeping hackathon demos effortless:
1. **Auto-Streaming OSDR Wearable Mock:** Streams realistic baseline HR, $\text{SpO}_2$, skin temperature, BP, and RR curves derived from `packages/osdr-fixtures` (`OSD-570`–`OSD-575`).
2. **1-Handed Manual Steppers & Zone Sliders:** Tapping any live vital chip opens a mobile Bottom Sheet with a thumb-friendly slider/stepper displaying **NEWS2 Green / Yellow / Amber / Red threshold bands** directly on the track for rapid manual override.
3. **1-Tap Judge Anomaly Simulator:** A collapsible **Mission Sim Drawer** in the top header triggers instant test scenarios across LAN clients (*Nominal $0g$ Baseline*, *Cabin Hypoxia $\text{SpO}_2$ 91%*, *Febrile Spike $39.2^\circ\text{C}$*, *Lunar Orthostatic BP Drop 92 mmHg*, *72h GI Alert*, and *RF Dead-Zone Offline Queue*).

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
2. **Optional Neuro-Cognitive Mini-Game (PVT-Lite):** A 15-second **Psychomotor Vigilance & Stroop Reaction Mini-Game** launched either via a quick-launch **`Neuro Check (15s)`** pill on the `Companion` avatar corner (self-check) or inside the **`Crew`** Buddy-Check flow. Severe latency spikes ($>500\text{ ms}$ above personal baseline) or repeated accuracy errors automatically flag potential `New-Onset Confusion (C)` (`+3` NEWS2 points) and prompt a peer buddy-check.

---

### 4.2 Interactive Care & Biological Homeostasis Action Dock

Directly beneath the 3D avatar in the mobile thumb zone, five tactile care buttons—with **circular homeostasis depletion rings**—let astronauts log daily biological inputs in 1–2 taps and trigger immediate 3D avatar feedback animations via non-blocking **Bottom Sheets** (no multi-field forms):

| Action Button | Logged Data & Quick Bottom-Sheet Inputs | Spaceflight Physiological Rationale (HRP Risk) | 3D Avatar Interaction & Visual Feedback |
| --- | --- | --- | --- |
| **Food** | 1-tap meal/ration presets + caloric ($\text{kcal}$) & electrolyte chips | Mitigates *Risk of Inadequate Food & Nutrition* and spaceflight-induced appetite suppression. | Avatar consumes ration pack; nutrition ring replenishes; happy gesture. |
| **Water** | 1-tap volume chips (`+100 mL`, `+250 mL`, `+500 mL`) | Counters microgravity diuresis, plasma volume loss, and orthostatic intolerance. | Avatar drinks from pouch; hydration ring fills; skin/aura brightens. |
| **Physical Activity** | Duration slider (`min`) + modality chips (`ARED`, `Cycle`, `Treadmill`) | Counters *Spaceflight-Induced Bone Changes* and *Reduced Cardiorespiratory Fitness*. | Avatar performs floating resistance/cardio clip; resilience ring boosts. |
| **Defecation** | Visual 7-card **Bristol Stool Scale (`1–7`)** selector + auto-timestamp | Altered GI motility, fluid shifts, and host-microbiome changes in space can cause severe constipation or GI distress. | Relieves abdominal discomfort posture; resets `72 h` GI transit timer; flags $>72\text{ h}$ absence. |
| **Urination** | 8-swatch **Hydration Color Bar (`1–8`)** + volume chips (`S / M / L`) | Tracks renal clearance, dehydration, and hypercalciuria-driven renal stone risk in microgravity. | Relieves bladder indicator; cross-checks output against water intake balance. |

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

### 4.5 Mobile-First UX/UI, Ergonomics & Clinical Accessibility Architecture

`apps/web` is designed **mobile-first for portrait viewports (`360–430px`)** and scales responsively into a **2-column split-screen HUD** on habitat tablets and desktops:

1. **Sticky Telemetry Header (`56px`):** Houses the active Crew ID + NEWS2 badge, a live **Environment Switcher Pill (`Earth 1g` | `Space 0g` | `Moon 1/6g`)**, a **Sync Status Pill** (`LAN <50ms` vs. `Offline Queue: N pending`), and the collapsible **Mission Sim Drawer** trigger.
2. **Primary `Companion` Screen Hierarchy (Top-to-Bottom, Zero-Scroll):**
   * **3D Avatar Stage (~45–50% viewport height):** Locked **3/4 portrait camera** (subtle micro-parallax only, preventing touch-scroll conflicts) with **tap-on-body-zone hotspots**: tapping **Helmet** inspects $\text{SpO}_2$ / Temp / ACVPU, **Chest** inspects HR / RR, and **Torso/Abdomen** inspects Food / Hydration / GI status.
   * **Corner Overlays on 3D Stage:** Top-left **Live `GO / NO-GO` Status Pill** (tapping jumps straight to the `MISSION READINESS TEST` in `Readiness`) and top-right **`Neuro Check (15s)`** quick-launch pill for PVT-Lite.
   * **Overlaid Vitals Strip:** Live OSDR vital chips (`HR`, `SpO2`, `BP`, `Temp`, `RR`) docked cleanly along the bottom edge of the 3D stage; tapping any chip opens its 1-handed NEWS2 zone-slider Bottom Sheet.
   * **5-Button Care Dock (Thumb Zone):** Arranged in a compact 2-row / grid layout directly above the bottom tab bar with `>= 48×48px` tap targets and homeostasis depletion rings.
3. **3-Tab Bottom Navigation (`>= 48×48px` targets):**
   * **`Companion`**: 3D `.glb` Avatar Stage + Vitals Strip + Care Dock.
   * **`Crew`**: Sorted `CREW STATUS` matrix, Buddy-Check ACVPU, `Neuro Check (15s)` peer trigger, and Cognitive Checklist.
   * **`Readiness`**: `MISSION READINESS TEST`, timestamped `GO / NO-GO` Flight Readiness Certificate, and Actionable Countermeasure SOPs.
   * On tablet/desktop (`>= 768px`), the 3D avatar stage pins left while Care, Crew, and Readiness panels render alongside it without forcing mobile compromises.
4. **High-Stress Alerts, Accessibility & Resilience:**
   * **LAN Emergency Alert UX (`NEWS2 >= 7`):** Displays a **persistent pulsing red screen border + sticky Red Alert banner** at the top with a 1-tap **`Open Triage`** CTA, escalating to a **full-screen emergency takeover** on LAN when a crewmate crosses `NEWS2 >= 7` (with a clear dismiss/respond action).
   * **Color-Blind Safe NEWS2 Badges:** Because NEWS2 uses both Yellow (Score `1`) and Amber (Score `2`), all dark-mode severity indicators pair color with **distinct iconography, border weights, and explicit numeric tags** (e.g., `NEWS2: 5 • MAX: 3`).
   * **Crew Card Privacy Summary:** Shared `CREW STATUS` roster cards summarize elimination/mood status as an operational badge (`Homeostasis OK / Attention`), keeping granular Bristol Stool or mood logs inside the peer detail drill-down.
   * **WebGL Context Resilience:** The `<Canvas>` wrapper handles WebGL context loss gracefully on mobile browsers and provides a compact telemetry fallback card if WebGL is unavailable.

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
