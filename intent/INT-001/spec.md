# spec: Blood bank management system (pitch prototype)
   - Status: draft
   - Intent: [INT-001](./intent.md)
   - Originator: Adrian Li-Hung-Shun
   - ADO work item: TBD
   - Reviewed by: —
   - Last updated: 2026-11-02

## 0. Read this first

- **Greenfield.** There is no existing codebase to integrate with. This repo holds only the README and INT-001. This spec defines the architecture from scratch (§9).
- **No organisational brand, security or UX policy exists.** Nothing was found in the repo, the skills, or the account's design-system and docs artifacts. Every baseline in §6–§8 is marked **PROPOSED**: it is a recommendation, not org policy. Someone has to adopt or replace them before anyone can claim conformance (C-1).
- **Regulatory constraints come from the intent:** TGA GMP, the Privacy Act 1988 and the APPs, and Australian data residency. They are mapped at the instrument level. A QA/RA reviewer has to confirm the clause-level mapping (C-14).
- **Concerns and policy conflicts are in §11.** Read that section before you estimate any work.
- **Hospitals use BloodNet for everything.** Hospital orders, receipt confirmations and problem reports all come through a simulated version of the National Blood Authority's BloodNet. Hospitals have no accounts or screens in this system (§4.6).
- **Some scope is still pending.** The intent hasn't decided on plasma for fractionation, mobile sessions, HLA/RBC matching or reporting, so the spec doesn't specify them yet (§5).

Each requirement is tagged with the source that justifies it:
`[I]` intent · `[GMP]` TGA Code of GMP for blood (2013) · `[TGO108]` donor selection and testing order · `[APP n]` Australian Privacy Principle · `[RES]` data residency · `[P]` proposed baseline (not policy, removable).
Priority is **Must** (needed for the intent's success criteria or its stated constraints) or **Should** (strengthens the demo; can be cut).

## 1. Goal

Build a blood bank management system that runs end to end: booking, donor CRM, collection, staff access control, shipping for processing, testing and components, hospital orders, and issuing blood bags. It is a pitch prototype for a specific Australian blood service. Every area is built to full depth, on synthetic data only. The aim is to show that this system fixes the service's pain points: poor user and donor experience, fragmented systems, and old legacy technology. [I]

It also covers the control functions a blood establishment system needs: donor eligibility and deferral, quarantine and release, product quality control, lookback and haemovigilance, consumable tracking, and configuration. [I]

**Success** means three things are true: a full trace works in a live demo, the people who see it give a "clearly better UX" verdict, and their QA/RA staff find no obvious compliance gap (§13).

**Out of scope** (from the intent):
- real integrations (simulated instead)
- hospital-facing screens (hospitals use BloodNet for everything)
- legacy data migration
- bedside transfusion
- TGA licensing, certification and formal validation
- staff rostering, competency tracking and HR records

## 2. Users and roles

| Surface | Users | Roles (initial set; to be confirmed with the target service) |
|---|---|---|
| Donor portal | Donors | Donor |
| Staff console | Blood service staff | Booking agent · Donor screening nurse · Medical officer · Collection staff · Logistics officer · Processing technician · Laboratory scientist · Release officer (authorised person) · Distribution officer · Quality reviewer (audit review, QC and haemovigilance decisions, safety-rule approval) · Configuration manager (master data and rules) · System administrator (users and roles only, no operational rights) |

One person can hold several roles. Segregation-of-duty rules still apply (ACC-04).

Hospitals have no accounts in this system; they work entirely through BloodNet (§4.6).

## 3. Domain language

These are the canonical terms the code and UI must use. Each entry lists the words to avoid. When the glossary stabilises, move this section into a root `GLOSSARY.md`.

- **Donor:** a person registered to give blood. _Avoid:_ patient, customer, user.
- **Session:** a scheduled period at a donor centre or mobile site, with slot capacity per donation type. _Avoid:_ clinic, shift, event.
- **Appointment:** a donor's booking into a slot of a session. _Avoid:_ booking (as a noun), reservation.
- **Screening:** the pre-donation health questionnaire, interview and physical checks that end in an eligibility outcome. _Avoid:_ triage, assessment.
- **Deferral:** a decision that a donor can't donate until a date (temporary) or ever (permanent), with a coded reason. _Avoid:_ ban, block, rejection.
- **Donation:** the act and record of one collection from one donor. It is identified by a **DIN**. _Avoid:_ collection (as a noun for the record), unit.
- **DIN (donation identification number):** the unique ID assigned at collection. It links the pack, the sample tubes and every component made from that donation. _Avoid:_ bag number, barcode.
- **Pack:** the physical container the blood is collected into. _Avoid:_ bag (in code; "blood bag" is fine in donor-facing copy).
- **Sample:** a tube taken with the donation for testing.
- **Shipment:** a tracked transport of packs and samples, or of components, between sites. Its contents are listed on a **manifest**. _Avoid:_ delivery, consignment.
- **Component:** a product made from a donation (e.g. red cells, plasma, platelets). It is identified by DIN plus product code. _Avoid:_ product (alone), unit (alone).
- **Quarantine:** the default state of a component until it is released. _Avoid:_ hold, pending.
- **Release:** an authorised person's signed decision that a component may be supplied. _Avoid:_ approve, publish.
- **BloodNet:** the National Blood Authority's national system that hospitals use to order and receipt blood and report problems. It is simulated in the prototype. _Avoid:_ hospital portal.
- **Order:** a hospital's request for components, received from BloodNet and made up of **order lines**. _Avoid:_ requisition, request.
- **Allocation:** assigning a specific released component to an order line. _Avoid:_ reservation, picking.
- **Issue:** dispatching allocated components to a hospital. Custody ends when the hospital confirms **receipt** in BloodNet. _Avoid:_ sale, delivery.
- **Recall:** withdrawing components already released or issued because of new information. _Avoid:_ return.
- **Post-donation information:** new information about a donation that arrives after it was collected, such as a donor reporting illness. It can trigger a recall.
- **Storage location:** a registered fridge, freezer or platelet agitator with a required temperature range. _Avoid:_ store, fridge (as a generic term).
- **QC sample:** a component selected under a sampling plan and tested against its specification. _Avoid:_ spot check, audit sample.
- **Consumable lot:** a supplier's batch of packs, tubes, reagents or solutions, tracked by lot number and expiry. _Avoid:_ batch (alone), stock.
- **Haemovigilance report:** a report of an adverse event in a donor or a transfusion recipient, linked to the donation or component involved. _Avoid:_ incident, complaint.
- **Configuration version:** a dated, approved version of master data or rules. Every record keeps the version it was made under.

### State models

- **Appointment:** Booked → Checked-in → Screened → (Donated | Did not donate) · plus Cancelled and No-show.
- **Donor:** Active · Temporarily deferred (until a date) · Permanently deferred · Inactive.
- **Donation:** Collected → In transit → Received → Processed · plus Incomplete and Discarded.
- **Component:** Quarantined → Released → Allocated → Issued → Received-by-hospital · plus Discarded, Expired and Recalled (from any state after Quarantined).
- **Order:** Submitted → Acknowledged → (Partially) allocated → Dispatched → Received · plus Cancelled.
- **Consumable lot:** Received → Available → Used up · plus Quarantined, Expired and Recalled.
- **Haemovigilance report:** Reported → Under investigation → Closed (with an outcome).

The system enforces every transition server-side. A transition that skips a state or fails its precondition is refused, with a reason.

## 4. Functional requirements

### 4.1 Booking appointments (BOOK)
- **BOOK-01 Must [I]:** Donors find sessions by location and date, and see available slots per donation type.
- **BOOK-02 Must [I]:** Donors book, reschedule and cancel their own appointments without staff help.
- **BOOK-03 Must [TGO108]:** Booking never offers a slot before the donor's next eligible date. That date comes from the configured minimum donation intervals and any active deferrals. Booking does not decide eligibility; screening does (COL-03).
- **BOOK-04 Must [I]:** Staff book, change and cancel appointments on a donor's behalf, and register walk-ins.
- **BOOK-05 Must [I]:** Staff define sessions: site, date, opening times, and slot capacity per donation type. Capacity is a number; staff rostering is out of scope.
- **BOOK-06 Must [I]:** Attendance outcomes (checked-in, no-show) are recorded and appear in the donor timeline.
- **BOOK-07 Should [P]:** Donors receive confirmations and reminders. In the prototype these go to the demo outbox (DEMO-03), not to real email or SMS.

### 4.2 Donor CRM (DON)
- **DON-01 Must [I]:** The donor record holds identity, contact details, the demographics eligibility needs, blood group once known, donation history, deferrals and communication preferences.
- **DON-02 Must [APP 3, APP 5]:** Registration shows a collection notice and records consent to collect health information, with the notice version and a timestamp.
- **DON-03 Must [I]:** Donors view and update their contact details and preferences. They can see their own donation history, upcoming appointments and next eligible date.
- **DON-04 Must [APP 12, APP 13]:** Donors can request access to, or correction of, their information. Corrections to screening or clinical records are made by staff with a reason, as an amendment. The original is never overwritten (C-5).
- **DON-05 Must [I]:** Staff search donors by name, date of birth or donor ID. They see one timeline of appointments, screenings, donations, deferrals and communications.
- **DON-06 Must [TGO108, GMP]:** Potential duplicate donors are flagged at registration. A merge needs an authorised role and is audited, so a donor can't escape a deferral by re-registering.
- **DON-07 Must [I]:** Deferrals can be temporary or permanent, each with a coded reason. Booking and screening enforce them. Reason detail is visible only to clinical roles (ACC-07).
- **DON-08 Should [I, APP 7]:** Staff build donor segments (e.g. by blood group or lapsed status) and send recruitment messages to the demo outbox. Marketing opt-outs are honoured.

### 4.3 Collection (COL)
- **COL-01 Must [I, GMP]:** At check-in, staff confirm the donor's identity and link the visit to the appointment or register a walk-in.
- **COL-02 Must [I, TGO108]:** The donor health questionnaire (medical and social history) is a versioned, configurable question set. Donors may pre-fill it online (C-6). It is finalised in a staff interview and closes with a donor declaration. Declared answers are immutable.
- **COL-03 Must [I, TGO108]:** Staff record pre-donation physical checks. Configured rules propose an outcome: eligible, or deferred with a reason. A staff override needs an authorised role and a reason.
- **COL-04 Must [GMP]:** On acceptance, a DIN is assigned. Pack and sample labels are linked by scanning, and the system checks that pack and samples match before collection starts.
- **COL-05 Must [I, GMP]:** The collection record holds start and end times, volume or weight, pack type and lot, the collecting staff member, and any incomplete-donation reason.
- **COL-06 Must [I, GMP]:** Donor adverse reactions are recorded against the donor and the donation as a haemovigilance report (TRC-06).

### 4.4 Shipping collections for processing (SHIP)
- **SHIP-01 Must [I]:** Packs and samples are scanned into a shipment container, and a manifest is generated.
- **SHIP-02 Must [GMP]:** Dispatch and receipt each record a time and a person. On receipt, contents are reconciled against the manifest, and missing, extra or damaged items are flagged.
- **SHIP-03 Must [GMP]:** Transport temperature and time come from a simulated data logger (DEMO-02). An excursion automatically quarantines the affected donations until QA decides.
- **SHIP-04 Must [I]:** Staff see every shipment in transit and its status.

### 4.5 Testing and components (LAB)
- **LAB-01 Must [I, GMP]:** Staff record component production from a donation: method, equipment and time since collection. Configured time limits are enforced.
- **LAB-02 Must [GMP]:** Each component has its own identity (DIN plus product code) linked to its parent donation. Splits and pools stay traceable in both directions.
- **LAB-03 Must [TGO108, GMP]:** The required tests are configured per donation type. Results arrive through the simulated analyser interface (DEMO-02). Manual result entry is a fallback for an authorised role only, and needs second-person verification.
- **LAB-04 Must [I]:** Blood group (ABO/RhD) and antibody screen results are recorded. A group that disagrees with the donor's historical group blocks release.
- **LAB-05 Must [TGO108]:** A reactive infectious-marker result blocks every component from that donation, flags the donor for clinical follow-up, and opens a lookback on earlier donations (TRC-03).
- **LAB-06 Must [I, GMP]:** Components start quarantined. Release is possible only when three things are true: all required results are complete and acceptable, the label is verified, and a release officer applies an e-signature (ACC-05). The system enforces this; it doesn't just record it.
- **LAB-07 Must [GMP]:** The final label is printed only at release and verified by scan. Labelling follows ISBT 128 (PROPOSED; confirm under D-5).
- **LAB-08 Must [I]:** Inventory shows stock by component type, blood group, site, status and expiry. Expired components are blocked automatically. Discards record a reason.
- **LAB-09 Must [GMP]:** Storage locations are registered with their temperature ranges, and readings come from the simulated logger (DEMO-02). An excursion quarantines the components stored there until QA decides.

### 4.6 Hospital orders (ORD)
Hospitals use BloodNet for everything, so this section describes no hospital-facing screens.
- **ORD-01 Must [I]:** Orders arrive through the simulated BloodNet interface (DEMO-02). Each line has a component type, ABO/RhD, quantity, attributes from the configured catalogue, and a required-by time with an urgency.
- **ORD-02 Must [I]:** Every status change (acknowledged, allocated, dispatched) is sent back to BloodNet, so hospitals follow their orders there.
- **ORD-03 Must [I]:** Blood service staff triage orders against live stock. They accept, part-fill, or propose a substitution, which goes to the hospital through BloodNet for confirmation.
- **ORD-04 Must [APP 6]:** Messages to BloodNet carry DIN and component details only, never donor identity.
- **ORD-05 Must [I]:** Amendments and cancellations from BloodNet are accepted until the order is allocated. After that they are refused, and the reason is sent back.
- **ORD-06 Must [GMP]:** Every BloodNet message, in or out, is stored with its time and content and linked to its order, so the order history can be audited end to end.

### 4.7 Issuing blood bags (ISS)
- **ISS-01 Must [I, GMP]:** Staff allocate components to order lines by scanning. The system blocks any component that is unreleased, expired, quarantined, recalled, or the wrong product or group.
- **ISS-02 Should [P]:** The system suggests components first-expiry-first-out.
- **ISS-03 Must [GMP]:** Staff record packing, the shipping container, transport conditions and dispatch, and produce a delivery docket. An electronic delivery record goes to BloodNet.
- **ISS-04 Must [I, GMP]:** Hospital receipt confirmations, and any discrepancies, arrive from BloodNet. Receipt ends the blood service's custody. A discrepancy opens a task for the Distribution officer.

### 4.8 Staff access control (ACC), cross-cutting
- **ACC-01 Must [I]:** Every screen and action checks permissions on the server, deny by default. Hiding something in the UI doesn't count as control.
- **ACC-02 Must [I, GMP]:** Every action is tied to a named individual. Shared accounts are not allowed.
- **ACC-03 Must [P]:** Staff must use MFA.
- **ACC-04 Must [GMP]:** For designated steps (manual result verification, release), the person who performs the step can't also verify it.
- **ACC-05 Must [GMP]:** Critical steps (release, issue, discard, deferral override, record amendment, safety-rule approval) need an e-signature. The user re-authenticates, and the signature records who signed, when, and its meaning (e.g. "Released by").
- **ACC-06 Must [I]:** System administrators manage users and role assignments but hold no operational permissions. Every role change is audited.
- **ACC-07 Must [APP 11]:** Only clinical roles can see sensitive health information: questionnaire answers, reactive results and deferral reasons. Every lookup from a DIN to a donor's identity is logged.
- **ACC-08 Should [P]:** Staff permissions can be scoped to one or more sites.

### 4.9 Traceability and audit (TRC), cross-cutting
- **TRC-01 Must [I]:** Starting from any identifier (donor, appointment, DIN, component, shipment, order), the trace view shows the full chain in both directions, filtered to what the viewer's role may see. This is success criterion 1.
- **TRC-02 Must [GMP]:** The audit trail records who, what, when (server time) and why, with old and new values, for every create, change and state transition on GMP records. It is append-only, can't be switched off, and can be reviewed and exported by the Quality reviewer.
- **TRC-03 Must [GMP]:** A recall or lookback lists every affected component and where it is now, including the hospitals it was issued to. It quarantines anything still in stock and sends recall notices to those hospitals through BloodNet.
- **TRC-04 Must [I, GMP]:** Post-donation information can be recorded against a donation and starts TRC-03.
- **TRC-05 Must [I]:** Hospital reports of suspected transfusion reactions, or problems with a component, arrive from BloodNet, identified by DIN. The report links to the component, its donation and donor, and can quarantine the other components from the same donation and start TRC-03. Reports carry no patient name or record number (APP 3). Bedside transfusion itself stays out of scope.
- **TRC-06 Must [I, GMP]:** The Quality reviewer tracks every haemovigilance report (COL-06, TRC-05) to closure and records the outcome.

### 4.10 Synthetic data and demo (DEMO)
- **DEMO-01 Must [I]:** A deterministic, seeded generator creates donors, histories, sessions, stock, hospitals and orders. No real person's data is used, and there is no import path for real data.
- **DEMO-02 Must [I]:** Simulators for analyser results, temperature loggers, courier events and BloodNet (hospital orders, receipts and reports) reach the system through the same interfaces a real integration would use. Their output is labelled "Simulated" (C-12).
- **DEMO-03 Must [P]:** All donor and staff notifications go to an in-app demo outbox. No email or SMS provider is used (C-10).
- **DEMO-04 Must [P]:** Every surface shows a persistent "Demo environment: synthetic data" banner.
- **DEMO-05 Must [P]:** The demo can be reset to a known state. Scripted scenarios cover six cases: the full happy-path trace, a reactive test result, a temperature excursion with quarantine, a part-filled urgent order, a supplier lot recall, and a hospital-reported transfusion reaction.
- **DEMO-06 Must [P]:** Each role has its own seeded demo account that signs in through the real authentication flow. There is no "switch persona" backdoor. A demo sign-in helper may list those accounts, but only when the environment flag is on.

### 4.11 Product quality control (QC)
- **QC-01 Must [I, GMP]:** Sampling plans are configured per component type (CFG-01). The system selects components for QC testing and records each result against the component's specification.
- **QC-02 Must [GMP]:** A component used up by destructive QC testing leaves inventory with that reason.
- **QC-03 Must [GMP]:** An out-of-specification result opens a QC failure. The Quality reviewer decides the outcome and can quarantine components from the same production run or equipment.
- **QC-04 Should [GMP]:** Where platelets are produced, bacterial screening results arrive through the analyser simulator. A positive screen recalls that component and the others from the same donation (TRC-03).

### 4.12 Consumables (CON)
- **CON-01 Must [I, GMP]:** Consumables (packs, sample tubes, reagents, additive solutions) are received into the system by type, supplier, lot and expiry.
- **CON-02 Must [I, GMP]:** Each use is recorded by scanning the lot: packs and tubes at collection, materials at processing, and reagent lots for each test run. Expired, quarantined and recalled lots are blocked.
- **CON-03 Must [GMP]:** Quarantining or recalling a lot (e.g. after a supplier recall) lists every donation, component and test run that used it, and passes the affected components to TRC-03.

### 4.13 Configuration (CFG)
- **CFG-01 Must [I]:** A configuration manager maintains master data and rules:
  - sites and donation types
  - the component catalogue (product codes, shelf life, storage conditions)
  - test panels
  - eligibility rules and questionnaire versions
  - QC sampling plans
  - label templates
- **CFG-02 Must [GMP]:** Configuration is versioned. A change needs a reason, takes effect from a set date, and never alters existing records. Each record keeps the configuration version it was made under.
- **CFG-03 Must [GMP]:** A change to a safety rule (eligibility, test panels, release rules, QC plans) needs the Quality reviewer's approval, by e-signature (ACC-05), before it takes effect.

## 5. Out of scope and pending decisions

Everything excluded by the intent (§1), plus:
- real messaging providers
- production hosting or an SLA
- downtime and business continuity procedures (a real system needs them; the prototype doesn't)
- break-glass or emergency access (C-9)
- returns of unused components from hospitals (open question Q-7)

**Pending scope decisions.** The intent leaves these open, so the spec doesn't specify them yet. Each one adds an area once it's decided:
- plasma for fractionation (Q-14; this also settles Q-6)
- mobile donor sessions (Q-14)
- HLA/RBC matching for special donors (Q-14)
- reporting and statistics (Q-14)

## 6. Security baseline (PROPOSED: no org policy exists)

- **Standard:** OWASP ASVS v5.0 Level 2 is the verification checklist. The Australian Government ISM/IRAP was not selected in the intent and is not applied.
- **Authentication:** OIDC. MFA (TOTP or passkey) for staff. Donors sign in with a passkey or an email one-time code. Re-authentication for e-signatures. Staff idle sessions time out after 15 minutes.
- **Authorisation:**
  - Server-side, deny by default (ACC-01).
  - The BloodNet interface authenticates as a system client with its own narrowly scoped permissions.
  - A role × action permission matrix is kept as code and covered by tests.
- **Donor privacy at the edge:**
  - Sign-in, registration and password-reset responses never reveal whether an email belongs to a donor, because being a blood donor is itself personal information.
  - Booking and sign-in are rate-limited.
- **Data protection:**
  - TLS 1.2 or later.
  - Managed encryption at rest.
  - Should: field-level encryption for questionnaire answers and reactive results.
  - Secrets live in the cloud key vault, never in the repo.
- **Input:** Every trust boundary (API, simulator interfaces, file inputs) is validated against the shared schema contracts (§9.1).
- **Logging:** No personal or health data in application logs, traces or error reports. The audit trail (TRC-02) is the record and is stored separately.
- **Supply chain:**
  - Lockfile is committed.
  - Dependency and SAST scanning runs in CI.
  - New dependencies need review.
- **Backups:** Encrypted, AU-only, and restore tested once before the demo.
- **Threat model:** Before the donor portal is built, write a short STRIDE pass covering account takeover, enumeration, booking abuse, and a forged or replayed BloodNet message.

## 7. Privacy, residency and GMP mapping

### 7.1 Privacy Act 1988 / APPs (designed for; synthetic data means the Act doesn't strictly apply, see C-4)

| APP | How the design responds | Req. |
|---|---|---|
| 1 Open management | A demo privacy policy is linked from every surface | DON-02 |
| 3 Collection; sensitive information | Only the fields eligibility needs; health information collected with consent | DON-01, DON-02 |
| 5 Notification | Versioned collection notice at registration | DON-02 |
| 6 Use and disclosure | Messages to hospitals through BloodNet carry DIN and component only, never donor identity | ORD-04, ACC-07 |
| 7 Direct marketing | Opt-out honoured for recruitment messages | DON-08 |
| 8 Cross-border | Nothing disclosed overseas (§7.2) | RES |
| 10–11 Quality, security | Donors keep their details current; §6 controls; retention rules (C-5) | DON-03, §6 |
| 12–13 Access, correction | Self-service view; corrections made as amendments | DON-03, DON-04 |

The Notifiable Data Breaches scheme needs a response runbook. That runbook is out of scope for the prototype. The audit and access logs (TRC-02, ACC-07) support an investigation.

### 7.2 Data residency [RES]

All of the following stay in Australian regions: the database, file and blob storage, backups, logs, metrics, traces, the identity provider's user store, the demo outbox, and any CI environment that runs against seeded data.

Recommended region: Azure Australia East, with backups in Australia Southeast (D-3).

A global CDN may serve static assets only. Authenticated responses are never cached at the edge.

Third-party SaaS that would receive personal data is not allowed unless its AU hosting is confirmed in writing.

### 7.3 TGA GMP and TGOs (designed for compliance, not validated; see C-3)

| GMP area | Design response | Req. |
|---|---|---|
| Donor selection (TGO 108) | Versioned questionnaire, rule-proposed eligibility, enforced deferrals, duplicate-donor control | COL-02/03, DON-06/07 |
| Testing (TGO 108) | Required tests set per donation type, interface-delivered results, reactive result blocks components | LAB-03/04/05 |
| Quarantine and release | Quarantine by default; release is system-enforced and e-signed | LAB-06, ACC-05 |
| Labelling | Label printed at release and verified by scan | LAB-07 |
| Storage and transport | Temperature and time records, excursion quarantine, expiry blocking | SHIP-03, LAB-08/09, ISS-03 |
| Quality control | Sampling plans, results checked against specification, QC failure decisions | QC-01 to QC-04 |
| Materials | Consumable lots received, used by scan, blocked when expired or recalled | CON-01 to CON-03 |
| Change control | Versioned configuration; safety-rule changes approved by Quality | CFG-02/03 |
| Records and data integrity | Append-only audit trail, amendments instead of overwrites, attributable actions | TRC-02, DON-04, ACC-02 |
| Personnel | Role-based authorisation and segregation of duties (training records out of scope, C-9) | ACC-01/04 |
| Traceability, recall and haemovigilance | Bidirectional trace, lookback, recall, donor and recipient adverse-event reports | TRC-01, TRC-03 to TRC-06, COL-06 |
| Computerised systems | Requirement IDs traced to automated tests (validation-ready; validation itself out of scope) | §13 |

## 8. UX standards and brand (PROPOSED)

Derived from the Impeccable skill's guidance for "Operate" mode, the mode for surfaces where users complete tasks. Its full process starts with `/impeccable init`, which writes PRODUCT.md, and then a design direction that produces DESIGN.md. That hasn't been run, because this task was a spec only. Both must be done before any UI is built.

| Surface | Mode | Primary device |
|---|---|---|
| Donor portal | Operate (book and manage) | Phone first |
| Staff console | Operate (dense) | Desktop; tablet at collection; keyboard-wedge barcode scanner (one that types like a keyboard) |

**General**
- Use familiar patterns over novelty: a top bar with side navigation, tables and standard form controls. Someone fluent in this kind of software should trust the interface on first sight.
- Type: one sans-serif family and a fixed rem type scale with a ratio of 1.125–1.2.
- Colour: restrained. The accent colour is used only for primary actions, selection and state.
- Define one semantic state vocabulary as design tokens: hover, focus, active, disabled, selected, loading, error, warning, success and info.
- Every interactive component ships all of its states: default, hover, focus, active, disabled, loading and error.
- Show loading with skeleton screens. Empty states tell the user what to do next.
- Motion lasts 150–250 ms, is used only to show a change of state, and respects `prefers-reduced-motion`.
- Prefer inline and progressive disclosure. Modals are reserved for e-signatures and irreversible confirmations.

**Safety-critical interaction**
- **Scan first:** wherever someone handles a physical item, scanning is the primary input. Manual entry is a fallback that needs a reason.
- **Blocking errors:** say what is wrong and what to do next, e.g. "Component … is quarantined: results pending. Scan a different component."
- **Status:** never shown by colour alone. Always text plus an icon (WCAG 1.4.1).
- **Irreversible steps:** summarise what will happen, then ask for the e-signature.
- **Identifiers:** DIN and product code always appear in one consistent, copyable format.

**Donor-specific**
- Use plain language with no clinical jargon. Keep booking steps to a minimum.
- Deferral messages are respectful and never disclose test results (C-7).

**Accessibility:** WCAG 2.2 AA on both surfaces. Every staff flow works by keyboard alone.

**Brand:** no brand guidelines exist.
- Use a neutral working name and identity until DESIGN.md exists.
- Do not use the target blood service's name, logo, colours or wording without written permission (C-11).
- Do not use the red cross emblem or anything that resembles it. It is protected under the *Geneva Conventions Act 1957* (Cth).

## 9. Architecture (greenfield)

### 9.1 Fixed by the requirements
These hold whatever language the team picks.
- **One deployable API, built as a modular monolith (D-2).** Release and allocation need single-transaction integrity across modules. There's one team and no scale requirement, and one deployable is simpler to make validation-ready.
- **PostgreSQL.** It provides:
  - transactions for release and allocation
  - database grants that make the audit tables append-only (TRC-02)
  - managed hosting in Australia East and Australia Southeast
- **Web only, no native apps.** The donor portal is responsive and phone-first. Keyboard-wedge scanners work in a browser (§8).
- **One schema contract** is shared by web, API and simulators, and validated at every trust boundary (§6). The simulators use the same contract a real integration would (DEMO-02).
- **Hosting runs code and holds data only in Australia** (§7.2). That rules out serverless and edge platforms that execute functions or cache responses outside Australia.
- **Module boundaries are checked by a lint or build rule**, not left to convention.

### 9.2 Team choice (D-1)
Nothing in the intent favours one language. Choose on the team's skills and record the choice as an ADR at planning.

| Option | Fits because | Watch for |
|---|---|---|
| **TypeScript:** React (Vite) web, Node API (NestJS or Fastify), Drizzle or Kysely, Zod | One language across web, API, simulators and seed; contracts shared directly | No surface needs server-side rendering, so a full-stack framework adds little. If one is used, self-host it in AU. |
| **.NET:** ASP.NET Core, EF Core, React or Blazor web | Fits Azure and ADO well; mature for regulated systems | Contracts shared through OpenAPI code generation rather than directly |

Default if the team has no preference: the TypeScript option, using pnpm workspaces. A build orchestrator isn't needed for five packages.

### 9.3 Layout and modules
```
web/          donor and staff surfaces, each with its own auth policy
api/          modular monolith (modules below)
simulators/   analyser, temperature-logger, courier and BloodNet simulators, using the same interfaces as the real systems
contracts/    shared schemas used by web, api and simulators
seed/         deterministic synthetic data generator (DEMO-01)
infra/        local containers (Postgres, identity provider); IaC for the AU demo environment
```

**API modules:** booking · donors · collection · logistics · processing · testing · quality-control · consumables · inventory · orders · bloodnet (interface adapter) · distribution · haemovigilance · configuration · access · audit · trace · outbox.

**Module rules**
- Each module owns its own Postgres schema and exposes a service interface. Other modules never reach into its tables.
- In-process domain events feed the trace view and the outbox.
- Steps that cross modules and must be atomic run in one database transaction: release (testing → inventory) and allocation (orders → inventory → distribution).
- Audit rows are written in the same transaction as the change they describe. The application's database role has no UPDATE or DELETE on the audit tables. A hash chain over the audit rows is a Should.

**Identity:** use an OIDC provider whose user store is in Australia.
- Recommended: Keycloak, self-hosted in the same AU region.
- Entra ID is acceptable once its data location is confirmed (D-4).

**Environments**
- Local: docker-compose.
- One shared AU demo environment.
- No production environment.

## 10. Delivery sequence

The intent sets no time box. Delivery is sequenced so that success criterion 1 is met early and each area then gains depth.

| Milestone | Delivers |
|---|---|
| M0 Foundations | Repo, identity, permission matrix, audit trail, e-signature, configuration and master data (CFG), seed generator, AU demo environment |
| M1 Thin trace | One whole-blood donation from booking to hospital receipt at minimal depth, plus the trace view. Proves criterion 1. |
| M2 Donor depth | BOOK and DON in full, COL-01 to COL-03, the donor portal |
| M3 Lab depth | COL-04 onwards, SHIP, LAB, QC, CON, recall and lookback |
| M4 Hospital supply | ORD and ISS in full through the BloodNet simulator, plus hospital haemovigilance reports (TRC-05/06) |
| M5 Demo readiness | Scripted scenarios, a UX walkthrough with representative users, and a mock QA audit against §7.3 |

## 11. Concerns and policy conflicts

| # | Conflict | Why it can't be fully satisfied | Proposed handling | Needs |
|---|---|---|---|---|
| C-1 | "Conform to our brand, security and UX policies" vs none existing | You can't claim conformance to a policy that doesn't exist | §6 and §8 baselines are marked PROPOSED; an owner adopts or replaces them | Policy owner named |
| C-2 | "Full depth, all areas" vs "pitch prototype" vs no time box | Full depth across eight areas is close to a product build, and nothing forces a cut | Sequence by M0 to M5 so criterion 1 lands at M1; revisit depth once a pitch date exists | Originator |
| C-3 | TGA GMP vs "certification and validation out of scope" | GMP expects computerised systems to be validated, so an unvalidated system can't be GMP-compliant and a QA reviewer will say so straight away. That threatens success criterion 3. | Say it up front in the demo; show validation-readiness (requirement → test traceability, audit trail, change control in ADO) | Originator accepts the framing |
| C-4 | "Respect the Privacy Act" vs synthetic data only | The Act covers personal information; synthetic data isn't, so the controls are demonstrations. Real data can still leak in, e.g. demo attendees typing their own details. | Build the controls for real, not as mock-ups; generator-only data; demo banner; treat anything typed in during a demo as real | — |
| C-5 | Immutable GMP records vs APP 11.2 (destroy or de-identify when no longer needed) and APP 13 (correction) | Traceability records must outlive the donor's relationship with the service; corrections can't overwrite | Corrections are amendments; "delete my account" closes portal access and marketing, but donation records are retained as required by law; the collection notice says so | QA/RA to confirm retention periods (Q-8) |
| C-6 | Low-friction donor self-service vs TGO 108 screening rigour | Fewer steps for donors conflicts with a full questionnaire and identity checks | The portal pre-fills the questionnaire only; eligibility is decided in person; the portal never tells a donor they are eligible | — |
| C-7 | Donor access to their own data (APP 12) vs clinical disclosure of reactive results | A reactive infectious-marker result needs clinical counselling, not a portal notification | The portal shows "please contact us" only; disclosure is a staff-led clinical task outside the system | The service's disclosure policy (Q-9) |
| C-8 | Full donor-to-hospital trace vs data minimisation (APP 6) | Hospitals need the chain from the component onwards, not the donor | BloodNet messages carry DIN and component only; only blood service roles can resolve a DIN to a donor, and each lookup is logged | — |
| C-9 | "Only do what your role allows" and "competency tracking out of scope" vs GMP personnel training and urgent operational needs | GMP expects tasks to be done by trained people, which role assignment can only approximate; there's no break-glass for emergencies | Role assignment stands in for competency (stated in the demo); emergency issue is out of scope | Originator: is emergency issue needed? (Q-10) |
| C-10 | AU residency vs typical SaaS (email/SMS, error tracking, analytics, identity provider, AI tools) | Many default to US regions | No third party receives personal data: demo outbox instead of real messaging, self-hosted or AU-region telemetry, identity provider residency confirmed (D-4). Seed data in AI coding tools is synthetic only; never paste real data into them. | — |
| C-11 | Pitching to a specific service vs using their brand | Their branding would help the pitch but risks trademark and impersonation problems; the red cross emblem is legally protected | Neutral brand; ask permission if co-branding is wanted | Originator |
| C-12 | "Real integrations out of scope" vs "full trace" and "compliance holds up" | Keying results in by hand would itself be a GMP red flag | Simulators use the real interface paths; their output is labelled "Simulated" | — |
| C-13 | "Clearly better UX" vs no measure | No acceptance threshold | Walkthrough of named tasks with representative users; the verdict is recorded; the measure stays an open question | Q-2 |
| C-14 | Regulatory references verified only at instrument level | The current TGA page lists the 2013 Code of GMP; TGO 108 replaced TGO 88 and TGO 109 replaced TGO 83–86; whether TGO 102 is still current is unconfirmed | Map clauses with a QA/RA reviewer before M3 | QA/RA reviewer |
| C-15 | "BloodNet covers everything" vs what's publicly confirmed | Public National Blood Authority material confirms ordering and receipting through BloodNet. I couldn't confirm that transfusion-reaction reports or recall notices go through it. The real interface specification belongs to the National Blood Authority and hasn't been reviewed, so the simulator's message format is our own design. | Simulate all hospital traffic through BloodNet, as the intent states. Keep the format behind the bloodnet adapter so it can be replaced. Confirm the real message set before any real integration. | Originator; National Blood Authority |
| C-16 | Prototype breadth vs a mature incumbent | If the incumbent is built on eProgesa, the pitch will be compared with a product whose 2023 FDA filing lists 18 modules, including the pending scope items in §5 | Differentiate on experience and integration, not breadth; say in the demo which modules are deliberately left out | Originator (Q-14, Q-15) |

## 12. Decisions and open questions

**Decisions (recommendation in bold)**
- **D-1 Language and frameworks:** the team chooses, in an ADR at planning, against §9.1. Default: **TypeScript (React + Vite, NestJS, Drizzle, Zod)**. .NET is an equal fit if that's the team's skill set.
- **D-2 Shape:** **modular monolith on PostgreSQL**, rather than microservices (§9.1).
- **D-3 Cloud:** **Azure Australia East, with backups in Australia Southeast** (the team already uses ADO). AWS Sydney/Melbourne is the alternative.
- **D-4 Identity:** **Keycloak in the AU region**, or Entra ID once its data location is confirmed.
- **D-5 Labelling:** **ISBT 128**, to be confirmed against TGO 102 and the target service's practice.
- **D-6 Product name and brand:** TBD (C-11).

**Open questions** (carried from the intent, plus new ones)
- Q-1 Which blood service is the target, and who will see the demo?
- Q-2 How will the UX verdict be gathered, and against which tasks?
- Q-3 Which TGA GMP expectations will their quality team check first?
- Q-4 Which integrations must be simulated? (This spec assumes analyser, temperature logger, courier and BloodNet.)
- Q-5 Is full depth across all areas realistic without a date? (C-2)
- Q-6 Which donation types are in scope: whole blood only, or plasma and platelet apheresis too? This drives COL, LAB and the component catalogue, and is linked to plasma for fractionation (Q-14).
- Q-7 Are returns of unused components from hospitals in scope?
- Q-8 What record retention periods apply? (C-5)
- Q-9 What is the target service's policy for disclosing reactive results to donors? (C-7)
- Q-10 Is emergency or urgent issue, or break-glass access, needed? (C-9)
- Q-11 Which staff roles does the target service actually have? (§2 is an initial set.)
- Q-12 Who owns and adopts the PROPOSED security, UX and brand baselines? (C-1)
- Q-13 ~~Should hospital orders come from a simulated BloodNet instead of the hospital portal?~~ **Answered 2026-11-02:** hospitals use BloodNet for everything (C-15).
- Q-14 Which of these are in scope: plasma for fractionation, mobile donor sessions, HLA/RBC matching for special donors, reporting and statistics? (§5)
- Q-15 Is the incumbent system built on eProgesa? (C-16)

## 13. Acceptance and verification

| Intent success criterion | Verified by |
|---|---|
| Full trace in live demo | DEMO-05 happy-path script, plus an automated Playwright end-to-end test that follows one DIN from booking to hospital receipt and checks the trace view (TRC-01) |
| Better UX (their verdict) | M5 walkthrough with representative donors and staff on named tasks; verdicts recorded (Q-2) |
| Compliance holds up | M5 mock audit by a QA/RA reviewer against §7; the only accepted gaps are the ones declared in §11 (C-3, C-9) |

**Engineering checks** (required for every milestone)
- Unit tests for rules: donation intervals, eligibility, release gating, allocation blocks, segregation of duties.
- A permission-matrix test covering every role × action.
- A test proving the audit tables reject UPDATE and DELETE.
- Contract tests for the BloodNet interface, including rejected, duplicate and out-of-order messages.
- Every Must requirement ID appears in at least one test name, so a requirement → test matrix can be generated (C-3).
- An accessibility check (axe) on every surface.
- The Impeccable detector runs on finished UI.
- **Should:** keep a hazard log that links each safety hazard to the requirements and tests that control it (validation-readiness, C-3).

## References

- TGA, Manufacturing blood and blood components: https://www.tga.gov.au/products/biologicals/human-cell-and-tissue-products/manufacturing/manufacturing-blood-and-blood-components
- Australian Code of GMP: human blood and blood components, human tissues and human cellular therapy products (2013): https://www.legislation.gov.au/Details/F2013L00854
- TGO 108 explanatory statement (replaces TGO 88): https://www.legislation.gov.au/F2021L01331/asmade/2021-09-28/es/1/pdf
- TGO 108 clause-by-clause tracking: https://www.tga.gov.au/products/regulations-all-products/legislation-and-legislative-instruments/therapeutic-goods-orders/clause-clause-tracking-tgo108-new-old
- ePROGESA v5.0.3 510(k) summary (FDA, 2023): https://www.fda.gov/media/173672/download
- National Blood Authority, BloodNet: https://www.blood.gov.au/bloodnet-blood-products-lifeblood
- Privacy Act 1988 (Cth) and the Australian Privacy Principles (OAIC)
- OWASP ASVS v5.0; WCAG 2.2
