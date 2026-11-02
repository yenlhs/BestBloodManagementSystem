# intent: Blood bank management system (pitch prototype)
   - Status: draft
   - Originator: Adrian Li-Hung-Shun
   - Source: idea
   - ADO work item: TBD
   - Accepted by: —

## What we want
A blood bank management system that manages end-to-end processes: booking appointments, donor CRM, collection, staff access control, shipping collections for processing, testing and components, managing hospital orders, and issuing blood bags.

It is a pitch prototype for a specific Australian blood service. All areas are built to full depth on synthetic data. Two groups use it directly:
- donors (self-service)
- blood service staff

Hospitals use the National Blood Authority's BloodNet for everything: ordering, confirming receipt and reporting problems. The system exchanges all of these with BloodNet and issues against the orders.

Staff only see and do what their role allows, and every action can be traced to a person.

It also covers the control functions a blood establishment system needs:
- donor eligibility and deferral
- quarantine and release
- product quality control
- lookback and haemovigilance
- consumable tracking
- configuration of products, tests, rules and sites

## Why now
There's a known pain point with the blood service's current setup: poor user and donor experience, fragmented systems, and old legacy tech. No fixed date or tender is driving it.

## Constraints
- Synthetic data only. No real donor or patient data.
- The design must visibly respect:
  - the TGA Australian Code of GMP for blood and blood components
  - the Privacy Act 1988 and the Australian Privacy Principles
  - Australian data residency
- No time or effort constraint has been set.

## Out of scope
- Real integrations (lab analysers, BloodNet, hospital LIS/EMR, their legacy system). These are simulated if needed; all hospital traffic goes through a simulated BloodNet.
- Any hospital-facing screens. Hospitals use BloodNet for everything.
- Migrating data from the legacy system.
- Bedside transfusion: anything after the bag reaches the hospital.
- Certification: TGA licensing and formal validation.
- Staff rostering, competency tracking and HR records.

## How we'll know it worked
- **Full trace in a live demo:** one donation is traced from booking through collection, testing and components to a bag issued against a hospital order.
- **Better UX:** the blood service's staff and donors who see the demo say it's clearly better than what they use today.
- **Compliance holds up:** their quality and regulatory people can't find an obvious gap against TGA GMP, the Privacy Act/APPs or data residency.

## Open questions
- Which blood service is it, and who will see the demo?
- How will the UX verdict be gathered: who gives it, and against which tasks?
- Which TGA GMP expectations will their quality team check first?
- Which integrations need simulating for the full trace to work?
- Is full depth across all areas realistic with no date set? Revisit once the pitch timing firms up.
- Is the incumbent system built on eProgesa? If so, that's the benchmark the pitch will be judged against.
- Which of these are in scope?
  - plasma for fractionation
  - mobile donor sessions
  - HLA/RBC matching for special donors
  - reporting and statistics
