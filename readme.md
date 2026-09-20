# Hospital Triage & Resource Simulation — README

## 1. Project Overview

This project is a browser-based hospital emergency/triage simulation built as a single HTML application.

The simulation models:

- Patient arrivals and triage.
- Patient severity levels from **Level 1 to Level 5**.
- Dynamic urgency/priority calculation.
- Hospital beds and ICU beds.
- Operating rooms.
- Doctors by specialty.
- Nurses.
- Ambulances.
- Automatic and manual patient admissions.
- Multiple queue-management strategies.
- Mortality risk while patients wait in triage.
- Staff shortages.
- Resource failures.
- Automatic peak-hour periods.
- Random walking-customer arrivals.
- Automatic ambulance arrivals.
- Mass-casualty/surge events.
- Treatment/discharge progression.
- Resource allocation and release.
- Simulation speed control.
- Charts, logs, alerts, sounds, and status indicators.

The application is designed as an interactive simulation rather than a real-world clinical decision system.

---

# 2. Main Simulation Concept

The hospital contains three important groups of objects:

1. **Waiting patients**
   - Patients waiting in the triage queue.
   - Their urgency score is continuously recalculated.
   - Their waiting time increases every simulation minute.
   - Their mortality risk can increase while they remain untreated.

2. **Admitted patients**
   - Patients who have successfully received the resources required for admission.
   - They consume doctors, nurses, beds, and sometimes operating rooms.
   - Their treatment progresses with simulation time.
   - Resources are released when treatment finishes.

3. **Hospital resources**
   - Doctors.
   - Nurses.
   - General beds.
   - ICU beds.
   - Operating rooms.
   - Ambulances.

The central simulation loop advances the hospital clock and updates all of these systems.

---

# 3. Simulation Clock

## Starting time

The simulation starts at:

**08:00**

Internally:

```text
simMinutes = 8 × 60
```

which is:

```text
480 simulation minutes
```

## Simulation tick

One simulation tick represents:

```text
1 simulation minute
```

Therefore:

```text
simMinutes += 1
```

on every simulation step.

## Simulation speed

The interface supports different simulation speeds.

The speed controls how frequently simulation ticks are executed in real time.

The simulation still advances by:

```text
1 simulation minute per tick
```

The important distinction is:

- **1x** → normal tick frequency.
- **2x** → more ticks per real second.
- **5x** → even more ticks per real second.

The Staff Shortage event does **not** accelerate simulation time.

---

# 4. Patient Severity Levels

Patients have a severity level:

```text
Level 1
Level 2
Level 3
Level 4
Level 5
```

The level is used by several systems:

- Urgency calculation.
- Staffing requirements.
- Specialist requirements.
- Nurse requirements.
- Mortality risk.
- Ambulance generation.
- Surge events.
- Admission eligibility.

## Staffing by severity

| Severity | Doctors | Nurses |
|---|---:|---:|
| Level 1 | 0 | 1 |
| Level 2 | 1 | 1 |
| Level 3 | 1 | 1 |
| Level 4 | 1 specialist | 2 |
| Level 5 | 1 specialist | 2 |

This is implemented by:

```js
getRequiredStaffForSeverity(severity)
```

The rules are:

```text
L1 → 0 doctors + 1 nurse
L2 → 1 doctor + 1 nurse
L3 → 1 doctor + 1 nurse
L4 → 1 specialist doctor + 2 nurses
L5 → 1 specialist doctor + 2 nurses
```

---

# 5. Urgency / Priority Formula

The simulation uses the following formula:

```text
Urgency =
    (Severity × 0.5)
  + (Disease × 0.35)
  + (Age × 0.5)
```

Implemented by:

```js
calculatePriorityScore(patient)
```

## Variables

### Severity

```text
Severity × 0.5
```

Higher severity produces a larger urgency contribution.

### Disease

```text
Disease × 0.35
```

The patient has a separate `disease` score.

For generated patients, the simulation currently defaults:

```text
Disease = Severity
```

when a separate disease value is not supplied.

### Age

```text
Age × 0.5
```

Older patients therefore receive a larger age contribution.

---

# 6. Example Urgency Calculation

Suppose a patient has:

```text
Severity = 5
Disease = 5
Age = 60
```

Then:

```text
Urgency
= (5 × 0.5)
+ (5 × 0.35)
+ (60 × 0.5)

= 2.5
+ 1.75
+ 30

= 34.25
```

The patient's urgency factor is therefore:

```text
34.25
```

The value is rounded to two decimal places in the JavaScript implementation.

---

# 7. Important Priority Behavior

Under **Priority Wise** mode:

```text
Higher urgency score = earlier position in the queue
```

Every simulation step recalculates the urgency score for all queued patients.

The queue is then sorted again.

This means the queue is dynamic rather than permanently fixed when a patient first arrives.

---

# 8. Queue Management Strategies

The simulation contains three queue strategies.

## 8.1 Priority Wise

Function:

```js
sortQueue()
```

Priority Wise sorts by:

```text
priorityScore
```

descending.

Therefore:

```text
Highest urgency → first
Lowest urgency → later
```

---

## 8.2 FCFS — First Come First Serve

FCFS uses waiting time.

The implementation sorts:

```js
b.waitTime - a.waitTime
```

Therefore the patient who has waited the longest moves toward the front.

Conceptually:

```text
Longest waiting patient → first
```

---

## 8.3 LRF — Least Resources First

LRF means:

```text
Least Resources First
```

The simulation calculates a resource-demand score.

Function:

```js
calculateResourceDemand(patient)
```

The formula is:

```text
Resource Demand =
    Treatment Time
  + (Nurses × 5)
  + OR requirement penalty
  + ICU requirement penalty
```

The penalties are:

```text
Operating Room required → +20
ICU bed required        → +15
```

Therefore:

```text
Resource Demand =
Treatment Time
+ Nurses × 5
+ 20 if OR is required
+ 15 if ICU is required
```

LRF sorts from the smallest resource-demand score to the largest.

---

# 9. Base Hospital Configuration

The Base Configuration area allows hospital capacity to be changed.

The configurable resources include:

- General beds.
- ICU beds.
- Operating rooms.
- Nurses.
- Ambulances.
- ER Physicians.
- Cardiologists.
- General Surgeons.
- Intensivists.
- Neurologists.
- Orthopedic Surgeons.
- General Practitioners.

The configuration is updated by:

```js
updateBaseConfig()
```

---

# 10. Default Nurse Capacity

The default nurse pool is:

```text
15 nurses
```

Internal state:

```js
nurses: {
    total: 15,
    allocated: 0,
    outage: false
}
```

The nurse count can be changed from the Base Configuration section.

Available nurses are:

```text
Available Nurses =
Total Nurses − Allocated Nurses
```

If the nurse pool is marked as unavailable, available nurses become:

```text
0
```

---

# 11. Doctor System

The simulation has multiple doctor pools.

Default doctor pools:

| Specialty | Default Doctors |
|---|---:|
| ER Physicians | 6 |
| Cardiologists | 4 |
| General Surgeons | 5 |
| Intensivists | 4 |
| Neurologists | 3 |
| Orthopedic Surgeons | 3 |
| General Practitioners | 4 |

Each doctor pool tracks:

```text
total
allocated
outage
```

Available doctors are calculated using:

```text
Effective Doctor Capacity − Allocated Doctors
```

---

# 12. Normal Staffing Rules

## Level 1

Level 1 patients are:

```text
1 nurse
0 doctors
```

They are treated as nurse-led cases.

The admission log identifies them as:

```text
1 Nurse (L1)
```

---

## Levels 2–3

Levels 2 and 3 require:

```text
1 doctor
1 nurse
```

The simulation first attempts to use the patient's requested/specialist doctor pool.

If that doctor is unavailable and the patient is Level 2 or Level 3, the simulation may use an available doctor from another department.

This is called:

```text
Cross-Department Assist
```

---

## Levels 4–5

Levels 4 and 5 require:

```text
1 specialist doctor
2 nurses
```

The specialist requirement is strict.

Unlike Levels 2–3, a Level 4–5 patient cannot simply replace the specialist with any unrelated doctor.

---

# 13. Doctor Allocation Algorithm

Function:

```js
checkResourceAvailability(patient)
```

First, the simulation determines the required staffing.

For patients requiring a doctor:

1. Check the patient's required specialty.
2. If that specialist is available, assign it.
3. If the patient is Level 2 or 3 and the specialist is unavailable:
   - Search all doctor pools.
   - Assign another available doctor if one exists.
4. If the patient is Level 4 or 5:
   - Specialist must be available.
5. Level 1 skips doctor allocation.

---

# 14. Staff Shortage Event

Staff Shortage is a separate event from Resource Failure.

When enabled:

```text
20% of doctor capacity is removed
```

The effective doctor count is:

```text
floor(total doctors × 0.8)
```

Function:

```js
getEffectiveDoctorTotal(doc)
```

Example:

```text
10 doctors
× 0.8
= 8 effective doctors
```

Another example:

```text
5 doctors
× 0.8
= 4 effective doctors
```

because the implementation uses `Math.floor()`.

---

# 15. Staff Shortage Behavior

When Staff Shortage is active:

### Level 1

Still:

```text
1 nurse
0 doctors
```

### Levels 2–3

Still:

```text
1 doctor
1 nurse
```

but:

- Specialist doctor is preferred.
- Cross-department doctor assistance is allowed if the specialist is unavailable.

### Levels 4–5

Still:

```text
1 specialist
2 nurses
```

A non-specialist doctor cannot replace the specialist.

---

# 16. Staff Shortage Does NOT Increase Treatment Speed

Staff Shortage does not change the simulation speed.

It also does not directly add treatment time.

Its primary effect is:

```text
20% reduction in effective doctor capacity
```

This can indirectly increase queueing because fewer doctor slots are available.

---

# 17. Resource Failure Event

Resource Failure is different from Staff Shortage.

When Resource Failure is activated:

```text
ICU beds → unavailable
Operating rooms → unavailable
```

Doctor pools are **not** shut down by Resource Failure.

The event also increases treatment duration by:

```text
30%
```

---

# 18. Resource Failure Treatment-Time Mathematics

For a treatment with original duration:

```text
T
```

the new duration is:

```text
T_new = T × 1.30
```

The implementation rounds the result.

Example:

```text
20 minutes × 1.30 = 26 minutes
```

Example:

```text
40 minutes × 1.30 = 52 minutes
```

---

# 19. Resource Failure and Already-Admitted Patients

When Resource Failure is switched ON, currently admitted patients are also affected.

For every admitted patient:

```text
oldTotal = existing treatment duration
added = round(oldTotal × 0.30)
newTotal = oldTotal + added
```

The additional time is also added to remaining treatment time.

Therefore Resource Failure affects:

1. Patients already receiving treatment.
2. Patients admitted after the failure begins.

---

# 20. Resource Failure and New Admissions

If Resource Failure is already active when a new patient is admitted:

```text
totalEstimated =
round(originalTreatmentTime × 1.3)
```

with a minimum treatment duration of 3 minutes.

---

# 21. Resource Availability

Admission requires all relevant resources to be available.

The check includes:

### Doctor

Required for Levels 2–5.

### Nurse

Required for every level.

### General Bed

Required when:

```text
bedType = General
```

### ICU Bed

Required when:

```text
bedType = ICU
```

### Operating Room

Required when:

```text
orRequired = yes
```

A patient can only be normally admitted when every required resource is available.

---

# 22. Automatic Admission

Function:

```js
attemptAutoAdmission()
```

When Auto Admit is ON:

1. Look through the queue.
2. Start with the highest-ranked patient according to the current strategy.
3. Check resource availability.
4. If resources are available:
   - Remove the patient from the queue.
   - Allocate resources.
   - Move the patient into the admitted list.
5. The system then continues simulation.

---

# 23. Manual Admission

A user can manually select a patient for admission.

Function:

```js
manualAdmit(patientId)
```

The system first checks normal resource availability.

If resources are available:

```text
Normal admission
```

If resources are not available:

```text
Force Admission confirmation modal
```

---

# 24. Force Admission

The Force Admission option allows an operator to override the resource bottleneck.

Function:

```js
confirmForceAdmit()
```

The patient is admitted despite the failed availability check.

This is an intentional simulation override and can therefore create a resource-demand situation that would not be allowed by normal admission logic.

---

# 25. Patient Admission Resource Allocation

Function:

```js
admitPatientDirectly(patient)
```

When admitted, the simulation can allocate:

- 1 doctor for Levels 2–5.
- 1 or 2 nurses depending on severity.
- 1 General bed or ICU bed.
- 1 Operating Room if required.

The patient's record stores:

```text
assignedDoctorSpec
treatedByNurseOnly
nursesAllocated
doctorsAllocated
totalTime
elapsedTime
remainingTime
admittedAt
```

This is important because resources can later be returned to the correct pools.

---

# 26. Discharge

Function:

```js
dischargePatient(index)
```

When treatment is complete:

1. Doctor allocation is released.
2. Bed allocation is released.
3. Operating room allocation is released if used.
4. Nurse allocation is released.
5. Discharged-patient statistics increase.
6. A discharge event is logged.
7. A discharge sound can be played.

---

# 27. Treatment Progress

Each simulation tick:

```text
elapsedTime += 1
remainingTime -= 1
```

A patient is discharged when:

```text
elapsedTime / totalTime >= 100%
```

or:

```text
remainingTime <= 0
```

---

# 28. Mortality Risk System

Waiting patients have a calculated mortality risk.

Function:

```js
calculateMortalityRisk(patient)
```

The calculation depends on:

- Severity.
- Waiting time.

There is an initial:

```text
2-minute grace period
```

After that, mortality risk increases in discrete:

```text
10-minute intervals
```

---

# 29. Mortality Base Risk

The current base risk values are:

| Severity | Base Risk | Increase per completed 10 min |
|---|---:|---:|
| Level 5 | 25% | 5% |
| Level 4 | 15% | 5% |
| Level 3 | 10% | 5% |
| Level 2 | 2% | 1% |
| Level 1 | 0% | 0% |

The formula is effectively:

```text
Wait Over = max(0, Wait Time − 2)

Completed Intervals =
floor(Wait Over / 10)

Risk =
Base Risk
+ Completed Intervals × Risk Increase
```

The final displayed risk is capped at:

```text
100%
```

---

# 30. Mortality Probability Per Simulation Minute

The displayed risk percentage is not directly used as a 100% chance every minute.

Instead:

```text
Per-Minute Chance =
Risk Percentage / 12000
```

with a maximum of:

```text
0.99
```

A random number is generated.

If:

```text
random number < per-minute chance
```

the patient dies in the triage queue.

This means mortality is evaluated every simulation minute after the relevant waiting period.

---

# 31. Random Walking-Patient System

Random Patient Generation can be turned ON/OFF.

When enabled, patients are automatically created and added to the triage queue.

Function:

```js
generateRandomPatientArrival()
```

The random patient receives:

- Random severity from 1–5.
- A matching condition from the condition database.
- Random age.
- Department.
- Doctor specialty.
- Bed requirement.
- OR requirement.
- Treatment duration.
- Disease score defaulted to severity.
- Calculated urgency score.

---

# 32. Normal Random Arrival Rate

Outside peak hours:

```text
1 arrival every 2–4 simulation minutes
```

The next interval is randomly selected using:

```js
randomBetween(2, 4)
```

---

# 33. Peak Hour System

Peak Hours are automatic by default.

State:

```text
peakHourEnabled = true
```

Peak periods occur every:

```text
3 simulation hours
```

Each peak period lasts:

```text
15 simulation minutes
```

The schedule is anchored to the starting time of:

```text
08:00
```

Therefore the first peak begins at:

```text
11:00
```

and lasts until:

```text
11:15
```

The next examples are:

```text
14:00–14:15
17:00–17:15
20:00–20:15
...
```

---

# 34. Peak Hour Mathematics

The function:

```js
isPeakHour()
```

calculates:

```text
elapsedFromStart =
max(0, simMinutes − 8 × 60)
```

A peak is active when:

```text
elapsedFromStart >= 180
```

and:

```text
(elapsedFromStart % 180) < 15
```

So:

```text
180 simulation minutes = 3 hours
15 simulation minutes = peak duration
```

---

# 35. Peak Hour Arrival Rate

Normal:

```text
2–4 simulation minutes between arrivals
```

Peak:

```text
1–2 simulation minutes between arrivals
```

The system therefore creates random walking customers more frequently during peak periods.

The peak status is shown next to the simulation clock.

Possible display states include:

```text
NORMAL ARRIVALS
PEAK HOUR • HIGH ARRIVALS
PEAK SYSTEM OFF
```

---

# 36. Peak Hour Toggle

The Peak Hours button can disable the automatic peak system.

When OFF:

```text
No automatic peak periods
```

Random arrivals use the normal arrival rate.

When turned back ON:

```text
Automatic peak scheduling resumes
```

The next random-arrival interval is rescheduled immediately.

---

# 37. Important Peak-Hour Interaction

Peak Hours change the arrival rate of the **Random Patient Generation** system.

Therefore the user must have:

```text
Random Patient Generation = ON
```

for the increased walking-customer arrival rate to produce new random patients.

Peak Hours do not directly increase ambulance frequency.

---

# 38. Ambulance System

The hospital has an ambulance pool.

Default:

```text
8 ambulances
```

Automatic ambulance events occur every:

```text
30–40 simulation minutes
```

The next interval is randomly chosen.

---

# 39. Automatic Ambulance Composition

Each ambulance carries:

```text
1–3 patients
```

The patients are generated as:

```text
Level 3–5
```

Therefore automatic ambulance arrivals represent higher-acuity cases.

---

# 40. Ambulance ETA Warning

The ambulance system displays a warning when the projected ETA reaches:

```text
10 real seconds
```

The ambulance itself still arrives according to its scheduled simulation time.

This creates an alert period before the actual simulated arrival.

---

# 41. Ambulance Resource Allocation

When an ambulance is approaching:

1. The simulation checks whether an ambulance resource is available.
2. If available, one ambulance resource is allocated.
3. The arrival warning is shown.
4. When the simulation reaches the target time, the ambulance arrives.
5. Its patients are inserted into the triage queue.
6. The ambulance resource is released.

If no ambulance is available:

```text
The scheduled event remains visible,
but the ambulance does not successfully transfer patients.
```

---

# 42. Ambulance Patient Priority

Ambulance patients receive an urgency score using the same:

```text
Severity + Disease + Age
```

formula.

They are then placed into the queue and sorted according to the currently selected queue strategy.

---

# 43. Mass-Casualty / Surge Event

The simulation includes a manual surge event.

Function:

```js
triggerMassCasualty()
```

The event generates:

```text
1–5 patients
```

These patients are:

```text
Level 3–5
```

The event reports how many patients of each severity were created.

Example:

```text
L3: 2
L4: 1
L5: 2
```

The surge patients are added directly to the triage queue.

---

# 44. Condition Database

The simulation contains predefined clinical conditions.

Examples include:

- Cardiac Arrest / Acute MI.
- Acute Ischemic Stroke.
- Major Polytrauma / Gunshot.
- Complex Open Fracture.
- Acute Appendicitis.
- Severe Respiratory Distress.
- Deep Laceration / Traumatic Cut.
- Severe 3rd Degree Thermal Burn.
- Sepsis / Septic Shock.
- Acute Heart Failure.
- Food Poisoning / GI.
- Anaphylactic Shock.
- General Checkup / Minor Consultation.

Each condition can define:

```text
Name
Department
Doctor specialty
Default severity
Bed type
OR requirement
Nurse value
Ambulance eligibility
Treatment time
```

The staffing system uses severity-based staffing rules when a patient is actually admitted.

---

# 45. Manual Patient Creation

The Add Patient interface allows a patient to be created manually.

Inputs include:

- Name.
- Age.
- Condition.
- Severity.
- Doctor specialty.
- Department.
- Bed type.
- OR requirement.
- Nurse input.
- Ambulance status.
- Treatment time.

The patient is added to the queue and receives an urgency score.

## Important implementation detail

The admission engine ultimately uses:

```js
getRequiredStaffForSeverity(severity)
```

for actual staffing allocation.

Therefore the nurse value shown/entered for a manually created patient does not override the severity-based admission staffing rules.

For example:

```text
Level 1 → 1 nurse
Level 2/3 → 1 nurse
Level 4/5 → 2 nurses
```

---

# 46. Patient Identification

Automatically created patients receive generated IDs.

The simulation maintains:

```js
patientIdCounter
```

The counter starts at:

```text
101
```

Different patient sources use identifiers such as:

```text
AUTO-...
AMB-...
SURGE-...
PAT-...
```

depending on their source.

---

# 47. Patient Waiting Time

Each queued patient has:

```text
waitTime
waitTimeSeconds
```

Every simulation minute:

```text
waitTime += 1
waitTimeSeconds += 60
```

Waiting time is used by:

- FCFS queue sorting.
- Mortality risk.
- Queue display.
- Analytics.

---

# 48. Queue Clearing

The queue can be cleared using:

```js
clearQueue()
```

This removes all waiting patients.

It does not represent successful treatment.

The simulation logs the number of removed waiting patients.

Already-admitted patients are not removed by queue clearing.

---

# 49. Doctor Outage Controls

Individual doctor pools can be manually toggled into outage.

Function:

```js
toggleDoctorOutage(spec)
```

When a doctor pool is in outage:

```text
Available doctors = 0
```

for that pool.

This can create specialist shortages and trigger cross-department assistance for eligible Level 2–3 cases.

Level 4–5 patients remain dependent on their required specialist.

---

# 50. Infrastructure Outage Controls

Individual infrastructure resources can also be toggled.

Function:

```js
toggleResourceOutage(resKey)
```

A resource in outage has:

```text
Available resource count = 0
```

This mechanism is separate from the global Resource Failure event.

---

# 51. Resource Accounting

Every resource tracks:

```text
total
allocated
outage
```

The standard availability equation is:

```text
Available =
max(0, Total − Allocated)
```

If:

```text
outage = true
```

then:

```text
Available = 0
```

This is handled by:

```js
getAvailableResourceCount()
```

---

# 52. Hospital Beds

Default bed capacity:

| Resource | Default |
|---|---:|
| General Beds | 25 |
| ICU Beds | 10 |

The hospital also has department-level bed figures used by the simulation's hospital/department views:

```text
Cardiology: 6
Emergency: 8
ICU: 8
Surgery: 6
Neurology: 4
Orthopedics: 5
General Care: 6
```

---

# 53. Operating Rooms

Default:

```text
6 operating rooms
```

A patient requiring an OR consumes one operating-room resource during admission.

The OR is released after discharge.

Resource Failure makes all OR capacity unavailable.

---

# 54. ICU

Default:

```text
10 ICU beds
```

Patients with:

```text
bedType = ICU
```

consume one ICU bed while admitted.

Resource Failure makes ICU beds unavailable.

---

# 55. General Beds

Default:

```text
25 general beds
```

Patients requiring:

```text
bedType = General
```

consume one general bed.

General beds are not automatically disabled by Resource Failure.

---

# 56. Auto Admit

Auto Admit can be toggled.

When ON:

```text
Eligible patients are automatically admitted
```

when resources are available.

When OFF:

```text
Manual admission is required
```

unless another simulation event explicitly moves a patient.

---

# 57. Event Logging

The simulation maintains an event log.

Events include categories such as:

```text
CONFIG
ADMIT
DISCHARGE
ARRIVE
QUEUE
FATALITY
AMBULANCE
FAILURE
OVERRIDE
OUTAGE
SURGE
PATIENT
```

The log provides a timeline of important simulation events.

---

# 58. Audio Feedback

The application uses audio tones for important events.

Examples include:

- Admission.
- Discharge.
- Surge.
- Ambulance.
- Deterioration/fatality.
- Resource failure.

The implementation uses:

```js
playAudioTone(type)
```

and a Tone.js synthesizer.

---

# 59. Simulation Tabs

The application contains multiple dashboard views/tabs.

The main areas provide information such as:

- Live triage queue.
- Resource status.
- Hospital utilization.
- Acuity information.
- Strategy comparison.
- Department information.
- Wait versus treatment analysis.
- Doctor and patient distributions.

The active tab is stored in:

```js
state.activeTab
```

and changed through:

```js
switchTab(tabNum)
```

---

# 60. Charts and Analytics

The application maintains chart objects including:

```text
chartUtilization
chartAcuity
chartStrategy
chartDeptBeds
chartWaitVsTreatment
chartDeptDoctors
chartDeptPatients
```

These are updated as the simulation progresses.

The charts are intended to make the effect of:

- Patient volume.
- Resource utilization.
- Acuity.
- Waiting time.
- Treatment time.
- Department capacity.
- Staffing.
- Queue strategy

visible during simulation.

---

# 61. Strategy Metrics

The application maintains metrics for:

```text
Priority
FCFS
LRF
```

Each strategy has:

```text
count
totalWait
```

This supports comparison of queue behavior during the simulation.

The metrics are stored in:

```js
state.strategyMetrics
```

---

# 62. Simulation State Object

The central state object is:

```js
state
```

It stores the simulation's current condition.

Major fields include:

```text
simTick
simMinutes
isRunning
simSpeed
simIntervalId
rafId
lastTickTimestamp
randomArrivalCountdown
ambulanceCallCounter
nextAmbulanceSimMinute
incomingAmbulances
strategy
priorityWeights
autoAdmitEnabled
staffShortageEnabled
resourceFailureEnabled
randomPatientGenEnabled
peakHourEnabled
stats
departmentBeds
doctors
resources
queue
admitted
patientIdCounter
logs
strategyMetrics
pendingForcePatientId
activeTab
```

---

# 63. Main Simulation Loop

The central simulation function is:

```js
stepSimulation()
```

Each tick approximately performs this sequence:

1. Record whether the simulation was in peak hour.
2. Increase the simulation tick.
3. Advance simulation time by one minute.
4. Update admitted patient treatment.
5. Discharge completed patients.
6. Increase waiting time for queued patients.
7. Calculate mortality risk.
8. Perform mortality checks.
9. Update peak-hour status.
10. Process random walking-patient arrivals.
11. Process automatic ambulance events.
12. Sort the queue.
13. Attempt automatic admission if enabled.
14. Update ambulance alerts.
15. Update charts.
16. Render the interface.

This function is the core engine connecting the different systems.

---

# 64. Important Core Functions

## `calculatePriorityScore(patient)`

Calculates:

```text
Severity × 0.5
+ Disease × 0.35
+ Age × 0.5
```

---

## `calculateMortalityRisk(patient)`

Calculates waiting-related mortality risk from:

```text
Severity
Waiting time
```

---

## `getRequiredStaffForSeverity(severity)`

Returns:

```text
Doctor requirement
Nurse requirement
```

based on severity.

---

## `getEffectiveDoctorTotal(doc)`

Applies Staff Shortage:

```text
total × 0.8
```

when enabled.

---

## `getAvailableDoctorCount(spec)`

Returns effective available doctors for a specialty.

---

## `getAnyAvailableDoctorSpec()`

Searches the doctor pools for an available doctor.

Used for Level 2–3 cross-department assistance.

---

## `getAvailableResourceCount(resKey)`

Calculates usable resource capacity.

---

## `checkResourceAvailability(patient)`

Determines whether a patient can normally be admitted.

Checks:

- Doctor.
- Nurse.
- General bed.
- ICU bed.
- Operating room.

---

## `attemptAutoAdmission()`

Attempts to admit an eligible queue patient automatically.

---

## `admitPatientDirectly(patient)`

Allocates the required resources and creates an admitted-patient record.

---

## `dischargePatient(index)`

Releases all resources used by an admitted patient.

---

## `scheduleNextRandomArrival()`

Sets the next random-patient arrival interval.

Normal:

```text
2–4 minutes
```

Peak:

```text
1–2 minutes
```

---

## `isPeakHour()`

Determines whether the current simulation time falls inside an automatic peak window.

---

## `updatePeakHourState()`

Updates the visual peak-hour badge.

---

## `processAutomaticAmbulanceCycle()`

Creates, warns about, and processes automatic ambulance arrivals.

---

## `triggerMassCasualty()`

Creates a random surge of Level 3–5 patients.

---

## `sortQueue()`

Recalculates urgency and orders the queue according to the selected strategy.

---

## `updateBaseConfig()`

Reads capacity settings from the UI and updates hospital resources.

---

# 65. Random Number Generation

The simulation uses:

```js
randomBetween(min, max)
```

which returns an integer between the two limits, inclusive.

Formula:

```text
floor(random × (max − min + 1)) + min
```

This is used for:

- Patient ages.
- Random severity.
- Arrival intervals.
- Ambulance intervals.
- Ambulance patient counts.
- Surge counts.
- Condition selection.

Because the simulation uses random values, different runs can produce different outcomes.

---

# 66. Rendering

The simulation separates its internal state from the visual interface.

After major state changes, functions call:

```js
renderAll()
```

This updates the displayed:

- Queue.
- Resources.
- Patients.
- Status indicators.
- Logs.
- Charts.
- Counters.

This keeps the interface synchronized with the simulation state.

---

# 67. Resource Failure vs Staff Shortage

These two events should not be confused.

| Feature | Staff Shortage | Resource Failure |
|---|---|---|
| Doctor capacity | −20% effective capacity | Unchanged |
| Nurses | Normal | Normal |
| ICU | Normal | Unavailable |
| OR | Normal | Unavailable |
| Treatment time | Normal | +30% |
| L1 staffing | 1 nurse | 1 nurse |
| L2–L3 cross-department doctor | Allowed | Depends on doctor availability |
| L4–L5 specialist requirement | Required | Required |

---

# 68. Peak Hours vs Resource Failure

Peak Hours and Resource Failure are independent systems.

Peak Hours:

```text
Increase random walking-patient arrival frequency
```

Resource Failure:

```text
Disable ICU/OR
Increase treatment duration by 30%
```

They can be active simultaneously.

This allows combined stress testing such as:

```text
Peak arrivals
+
ICU/OR failure
+
Longer treatment
```

---

# 69. Peak Hours vs Staff Shortage

These are also independent.

For example, the simulation can simultaneously have:

```text
Peak Hour = ON
Staff Shortage = ON
```

which creates:

- More random arrivals.
- 20% lower effective doctor capacity.
- Normal nurse rules.
- Normal bed rules unless another outage occurs.

---

# 70. Combined Failure Scenarios

The simulation can be used to test combinations such as:

### Scenario A — Normal operation

```text
Staff Shortage: OFF
Resource Failure: OFF
Peak Hour: OFF/currently normal period
```

### Scenario B — Doctor shortage

```text
Staff Shortage: ON
```

### Scenario C — Infrastructure failure

```text
Resource Failure: ON
```

### Scenario D — Patient surge

```text
Mass Casualty Event
```

### Scenario E — Peak demand

```text
Peak Hour
+
Random Patient Generation ON
```

### Scenario F — Extreme stress test

```text
Peak Hour
+
Staff Shortage
+
Resource Failure
+
Random Patient Generation
+
Ambulance arrivals
+
Surge event
```

This can create a high-load hospital scenario.

---

# 71. Patient Flow

A typical patient lifecycle is:

```text
Arrival
   ↓
Triage Queue
   ↓
Urgency Calculation
   ↓
Queue Sorting
   ↓
Resource Check
   ↓
Admission
   ↓
Resource Allocation
   ↓
Treatment
   ↓
Discharge
   ↓
Resource Release
```

If the patient remains in the queue too long:

```text
Waiting
   ↓
Mortality Risk Increase
   ↓
Possible Fatality
```

---

# 72. Automatic Patient Flow

For an automatically generated patient:

```text
Random event
↓
Condition selected
↓
Severity assigned
↓
Patient object created
↓
Disease score assigned
↓
Urgency calculated
↓
Queue insertion
↓
Queue sorting
↓
Resource check
↓
Auto admission if possible
```

---

# 73. Ambulance Patient Flow

For an ambulance patient:

```text
Ambulance scheduled
↓
ETA warning
↓
Ambulance resource check
↓
Ambulance arrives
↓
1–3 Level 3–5 patients enter queue
↓
Urgency calculated
↓
Queue sorted
↓
Admission process
```

---

# 74. Resource Allocation Example

Suppose a Level 5 patient requires:

```text
1 specialist doctor
2 nurses
1 ICU bed
1 OR
```

Admission consumes:

```text
Specialist doctors: +1
Nurses: +2
ICU beds: +1
Operating rooms: +1
```

When treatment completes, those same allocations are subtracted.

---

# 75. Level 1 Example

A Level 1 patient requires:

```text
0 doctors
1 nurse
```

If a nurse is available and the required bed is available:

```text
Admission succeeds
```

No doctor is allocated.

This makes Level 1 cases nurse-led.

---

# 76. Level 3 Cross-Department Example

Suppose:

```text
Level = 3
Required specialist = Cardiologist
```

but all cardiologists are busy.

If another doctor pool has an available doctor:

```text
Cross-Department Assist
```

may be used.

The admission record stores the actual assigned doctor.

This prevents the system from assuming the requested specialist was used when another doctor actually handled the admission.

---

# 77. Level 5 Specialist Example

Suppose:

```text
Level = 5
Required specialist = Intensivist
```

and all intensivists are busy.

Even if another department has available doctors:

```text
Admission fails under normal rules
```

because Level 5 requires the specialist.

---

# 78. Resource Failure Example

Suppose:

```text
Original treatment = 30 minutes
```

Resource Failure starts.

Then:

```text
30 × 1.3 = 39 minutes
```

The patient now has approximately:

```text
39-minute total treatment duration
```

instead of 30.

ICU and OR availability simultaneously become zero.

---

# 79. Peak Hour Example

Starting time:

```text
08:00
```

First peak:

```text
11:00–11:15
```

Normal random arrivals:

```text
2–4 minutes apart
```

Peak random arrivals:

```text
1–2 minutes apart
```

Therefore a 15-minute peak window can generate substantially more random arrivals than an equivalent normal period.

---

# 80. Base Configuration Defaults

Current defaults in the source include:

```text
General Beds: 25
ICU Beds: 10
Operating Rooms: 6
Nurses: 15
Ambulances: 8
```

Doctors:

```text
ER Physicians: 6
Cardiologists: 4
General Surgeons: 5
Intensivists: 4
Neurologists: 3
Orthopedic Surgeons: 3
General Practitioners: 4
```

---

# 81. Important Implementation Notes

## Disease score fallback

When a patient has no separate disease score:

```text
Disease = Severity
```

This is handled by:

```js
patient.disease ?? severity
```

inside the priority calculation.

---

## Priority weights object

The state still contains an older `priorityWeights` object:

```js
priorityWeights: {
    severity: 25,
    waitTime: 1.5,
    doctorAvailBonus: 15
}
```

However, the active Priority Wise urgency calculation uses the fixed formula:

```text
Severity × 0.5
+ Disease × 0.35
+ Age × 0.5
```

The older object is not the active mathematical formula.

---

# 82. Technology

The simulation is delivered as a single HTML file.

The main technologies used include:

- HTML.
- CSS utility classes.
- JavaScript.
- DOM manipulation.
- `requestAnimationFrame`.
- Chart rendering.
- Tone.js for audio feedback.
- Font Awesome icons.

The application can run directly in a modern browser.

---

# 83. No Backend Requirement

The simulation is designed to run locally in the browser.

Its main simulation state exists in JavaScript memory:

```js
const state = {...}
```

There is no requirement for a hospital database or server to perform the simulation logic.

Refreshing the page resets the in-memory simulation state.

---

# 84. Simulation Reset Behavior

Because the simulation state is held in JavaScript memory, a browser refresh starts a new simulation state using the default configuration.

Runtime changes such as:

- Queue contents.
- Admissions.
- Fatalities.
- Discharges.
- Current clock.
- Active failures.

are therefore simulation-session data.

---

# 85. What the Simulation Is Designed to Demonstrate

The project is useful for demonstrating concepts such as:

- Triage prioritization.
- Queue management.
- Resource allocation.
- Capacity planning.
- Staff shortages.
- Infrastructure failures.
- Demand spikes.
- Mortality risk under waiting.
- Cross-department staffing.
- Treatment-time changes.
- Hospital utilization.
- Simulation-based decision analysis.

It is primarily an educational/simulation model.

---

# 86. Key Mathematical Summary

## Urgency

```text
U = 0.5S + 0.35D + 0.5A
```

where:

```text
S = Severity
D = Disease
A = Age
```

---

## Effective doctor capacity during Staff Shortage

```text
Effective Doctors = floor(Total Doctors × 0.8)
```

---

## Available resource capacity

```text
Available = max(0, Total − Allocated)
```

unless the resource is in outage:

```text
Available = 0
```

---

## Resource Failure treatment duration

```text
New Treatment Time = round(Original Time × 1.30)
```

---

## LRF resource-demand score

```text
Demand =
Treatment Time
+ Nurses × 5
+ 20 if OR required
+ 15 if ICU required
```

---

## Mortality risk

```text
Wait Over = max(0, Wait Time − 2)

Intervals = floor(Wait Over / 10)

Risk =
Base Risk
+ Intervals × Risk Increase
```

capped at:

```text
100%
```

---

## Peak-hour detection

```text
Elapsed = max(0, Simulation Time − 08:00)

Peak if:
Elapsed >= 180
AND
Elapsed % 180 < 15
```

---

## Normal random arrival interval

```text
2–4 simulation minutes
```

## Peak random arrival interval

```text
1–2 simulation minutes
```

---

# 87. Main Controls Summary

| Control | Purpose |
|---|---|
| Play/Pause | Start or stop simulation |
| 1x / 2x / 5x | Change simulation speed |
| Auto Admit | Automatically admit eligible patients |
| Random Patients | Generate walking customers |
| Staff Shortage | Remove 20% effective doctor capacity |
| Resource Failure | Disable ICU/OR and add 30% treatment time |
| Peak Hours | Enable/disable automatic demand spikes |
| Strategy | Select Priority, FCFS, or LRF |
| Add Patient | Create a custom patient |
| Surge/Mass Casualty | Generate 1–5 Level 3–5 cases |
| Doctor Outage | Disable an individual doctor pool |
| Resource Outage | Disable an individual infrastructure resource |
| Clear Queue | Remove all waiting patients |

---

# 88. Recommended Testing Sequence

For demonstrating the simulation, a useful test sequence is:

### Test 1 — Normal operation

1. Start simulation.
2. Add several patients.
3. Observe Priority Wise ordering.
4. Observe automatic admissions.
5. Observe resource utilization.
6. Wait for treatment completion.
7. Observe discharge.

### Test 2 — Staff Shortage

1. Turn Staff Shortage ON.
2. Observe doctor capacity.
3. Add Level 2–3 patients.
4. Observe cross-department assistance.
5. Add Level 4–5 patients.
6. Observe specialist dependency.

### Test 3 — Resource Failure

1. Admit several patients.
2. Turn Resource Failure ON.
3. Observe ICU/OR status.
4. Observe treatment times increase by 30%.
5. Attempt ICU/OR-dependent admissions.

### Test 4 — Peak Hours

1. Enable Random Patient Generation.
2. Start around the simulation's peak period.
3. Observe the clock badge.
4. Compare normal arrival intervals with peak intervals.

### Test 5 — Mortality

1. Disable or reduce admission capacity.
2. Allow high-severity patients to remain in queue.
3. Observe waiting time.
4. Observe mortality-risk changes.
5. Observe possible fatality events.

### Test 6 — Surge

1. Trigger a Mass Casualty event.
2. Observe 1–5 Level 3–5 patients.
3. Watch resource utilization.
4. Observe queue pressure and admission bottlenecks.

---

# 89. Final System Architecture

At a high level, the application works as:

```text
                   ┌──────────────────┐
                   │ Simulation Clock  │
                   └────────┬─────────┘
                            │
                            ▼
                   ┌──────────────────┐
                   │ stepSimulation() │
                   └────────┬─────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
   Patient Flow        Resource Flow       Event Systems
        │                   │                   │
        ▼                   ▼                   ▼
 Queue / Admission     Doctors / Nurses    Peak Hours
 Treatment             Beds / OR / ICU     Ambulances
 Discharge             Capacity            Surge Events
 Mortality             Outages             Random Arrivals
        │                   │                   │
        └───────────────────┼───────────────────┘
                            ▼
                     ┌───────────────┐
                     │  renderAll()  │
                     └───────┬───────┘
                             ▼
                     Browser Dashboard
```

---

# 90. Important Disclaimer

This simulation is a software model for demonstration, education, experimentation, and hospital-resource simulation.

Its formulas and staffing rules are programmed simulation assumptions.

They should **not** be interpreted as real clinical triage guidelines, medical advice, staffing regulations, or a substitute for qualified medical decision-making.

---

# 91. Source File

The simulation is contained in:

```text
aditya_priority_staff_nurse_peak.html
```

The README documents the behavior implemented in that source file, including the current staffing, priority, resource-failure, peak-hour, ambulance, mortality, and queue-management systems.
