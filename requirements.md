# Requirements Document

## Introduction

EquiSurg is an AI-Driven Surgical Triage & Accessibility Engine designed for public healthcare systems to address the critical "Waiting Room" crisis in government hospitals. The system tackles the societal problem where demand for life-saving surgeries vastly outweighs the supply of Operating Theatres (OTs) and surgeons, leading to inequitable access, information blackouts for rural patients, and resource inefficiency.

### The Problem

In public district and government hospitals:

- **Inequitable Access**: Waitlists are often chaotic or vulnerable to manipulation, pushing aside high-risk, low-income patients
- **Information Blackout**: Rural patients travel long distances and wait in crowded corridors for days without knowing when or if their surgery will happen
- **Resource Inefficiency**: Static scheduling leads to underutilized rooms and burned-out public health staff

### The Solution: Three Core Pillars

EquiSurg operates on three core pillars that align with improving access to resources for public systems and providing low-bandwidth civic information:

#### A. Equitable AI Triage (Access to Public Resources)

Instead of a "first-come, first-served" system that disadvantages rural patients who arrive late, the engine uses an **XGBoost Machine Learning model** combined with a **Constraint Programming Solver** (Google OR-Tools CP-SAT). The system evaluates patient physiology (Age, BMI) and Clinical Risk (ASA Score), then mathematically optimizes the schedule to ensure high-risk patients are prioritized without causing resource overlaps (surgeons, rooms, or equipment). This provides mathematical equity in accessing public healthcare resources.

#### B. Low-Bandwidth, Local-Language Assistant (Civic Information)

To bridge the digital divide, patients do not need to download an app. EquiSurg features an automated, low-bandwidth notification pipeline. Once the AI finalizes the schedule, it triggers automated **SMS/WhatsApp voice-note alerts** in the patient's local language (e.g., Hindi, Kannada). This eliminates the anxiety of the "waiting room" and respects the time and dignity of daily-wage workers.

#### C. Real-Time Emergency Resilience (Community Impact)

Public hospitals deal with sudden community traumas (accidents, mass casualties). The system includes a **"Code Red" dynamic scheduler** that instantly bypasses the standard queue when emergencies arrive, allocates a reserved "Green Corridor" trauma bay, and automatically sends SMS updates to elective patients whose surgeries are delayed. This prevents the collapse of the hospital's daily operations during local emergencies.

### Technical Architecture

- **Predictive Layer**: XGBoost models predict exact surgery durations based on community health data, preventing schedule overruns
- **Optimization Layer**: Google OR-Tools (CP-SAT) handles the NP-Hard logic of assigning Doctors, Rooms, and Equipment to maximize daily public output
- **Accessibility Layer**: Twilio/Fast2SMS API integration for real-time, low-bandwidth text and voice updates to non-smartphone users
- **Dashboard**: A Streamlit-based interface for public hospital administrators to visualize the schedule and trigger emergency protocols instantly

### Real-World Impact

EquiSurg transforms the public healthcare experience from a system of stress and uncertainty into one of predictability and transparency. By optimizing the backend logistics of government hospitals, we increase the net capacity of surgeries performed daily. By adding a low-bandwidth communication layer, we ensure that the most vulnerable populations are kept informed in their native language, drastically improving community trust in public health systems.

## Glossary

- **EquiSurg_System**: The complete AI-driven surgical triage and accessibility platform
- **Triage_Engine**: The XGBoost ML model that predicts surgery duration and patient risk
- **Scheduler**: The Google OR-Tools CP-SAT constraint programming solver that optimizes surgical schedules
- **Notification_Service**: The SMS/WhatsApp communication module for patient alerts
- **Emergency_Handler**: The dynamic rescheduling component for Code Red scenarios
- **Dashboard**: The Streamlit-based administrative interface
- **Patient**: An individual requiring surgical intervention
- **Surgery**: A scheduled medical procedure requiring operating room resources
- **ASA_Score**: American Society of Anesthesiologists physical status classification (1-5)
- **Green_Corridor**: Reserved trauma bay capacity for emergency cases
- **Code_Red**: Emergency scenario requiring immediate resource reallocation
- **Elective_Surgery**: Pre-planned, non-emergency surgical procedure
- **Resource**: Operating room, surgeon, equipment, or staff required for surgery

## Requirements

### Requirement 1: Patient Data Management

**User Story:** As a hospital administrator, I want to register and manage patient information, so that the system can make informed triage decisions.

#### Acceptance Criteria

1. WHEN a patient is registered, THE EquiSurg_System SHALL store patient demographics (age, gender), physiological data (BMI, weight), and clinical risk indicators (ASA Score)
2. WHEN patient data is entered, THE EquiSurg_System SHALL validate that age is a positive integer, BMI is between 10 and 60, and ASA Score is between 1 and 5
3. WHEN patient data is updated, THE EquiSurg_System SHALL maintain an audit trail of all changes with timestamps and user identifiers
4. THE EquiSurg_System SHALL persist patient data to storage immediately upon successful validation

### Requirement 2: Surgery Duration Prediction

**User Story:** As a surgical scheduler, I want accurate predictions of surgery duration, so that I can optimize operating room utilization.

#### Acceptance Criteria

1. WHEN a surgery is scheduled, THE Triage_Engine SHALL predict surgery duration based on patient age, BMI, ASA Score, and surgery type using the XGBoost model
2. WHEN the prediction model is invoked, THE Triage_Engine SHALL return a duration estimate in minutes with a confidence interval
3. THE Triage_Engine SHALL achieve a mean absolute error of less than 20 minutes on historical surgery data
4. WHEN the model encounters missing patient data, THE Triage_Engine SHALL return an error indicating which required fields are absent

### Requirement 3: Risk-Based Prioritization

**User Story:** As a medical director, I want high-risk patients to be prioritized, so that we minimize adverse outcomes and mortality.

#### Acceptance Criteria

1. WHEN multiple patients are awaiting surgery, THE Scheduler SHALL prioritize patients with higher ASA Scores (4-5) over lower scores (1-3)
2. WHEN two patients have identical ASA Scores, THE Scheduler SHALL prioritize the patient who has been waiting longer
3. WHEN a patient's ASA Score is updated, THE Scheduler SHALL re-evaluate the surgical schedule and adjust priorities accordingly
4. THE Scheduler SHALL ensure that no patient with ASA Score 4 or 5 waits more than 7 days for elective surgery

### Requirement 4: Resource Constraint Management

**User Story:** As an operations manager, I want to prevent resource conflicts, so that surgeries proceed without delays or cancellations.

#### Acceptance Criteria

1. WHEN scheduling a surgery, THE Scheduler SHALL verify availability of required surgeon, operating room, and equipment for the entire predicted duration
2. WHEN a resource conflict is detected, THE Scheduler SHALL not assign the surgery to that time slot
3. THE Scheduler SHALL maintain a minimum 30-minute buffer between consecutive surgeries in the same operating room for cleaning and preparation
4. WHEN all resources are fully booked, THE Scheduler SHALL identify the earliest available time slot that satisfies all constraints

### Requirement 5: Optimal Schedule Generation

**User Story:** As a hospital administrator, I want an optimized surgical schedule, so that we maximize patient throughput while respecting clinical priorities.

#### Acceptance Criteria

1. WHEN generating a schedule, THE Scheduler SHALL use Google OR-Tools CP-SAT solver to find a feasible solution
2. THE Scheduler SHALL maximize the number of high-priority (ASA 4-5) surgeries scheduled within the planning horizon
3. THE Scheduler SHALL minimize total patient wait time as a secondary optimization objective
4. WHEN no feasible schedule exists, THE Scheduler SHALL return a detailed report of constraint violations and resource bottlenecks

### Requirement 6: Low-Bandwidth Patient Notifications

**User Story:** As a patient in a rural area, I want to receive surgery notifications via SMS or voice call, so that I can prepare without needing a smartphone or internet.

#### Acceptance Criteria

1. WHEN a surgery is scheduled, THE Notification_Service SHALL send an SMS to the patient's registered mobile number within 5 minutes
2. WHERE the patient has selected voice notification preference, THE Notification_Service SHALL deliver an automated voice call in the patient's preferred language (Hindi, Kannada, Tamil, Telugu, Bengali)
3. WHEN a notification fails to deliver, THE Notification_Service SHALL retry up to 3 times with exponential backoff (1 min, 5 min, 15 min)
4. THE Notification_Service SHALL include surgery date, time, location, and pre-operative instructions in all notifications
5. WHEN a notification is successfully delivered, THE Notification_Service SHALL log the delivery timestamp and confirmation status

### Requirement 7: Emergency Code Red Handling

**User Story:** As an emergency department coordinator, I want to dynamically reallocate resources during mass casualty events, so that critical patients receive immediate care.

#### Acceptance Criteria

1. WHEN a Code Red is declared, THE Emergency_Handler SHALL immediately allocate all Green Corridor trauma bays to emergency cases
2. WHEN emergency resources are insufficient, THE Emergency_Handler SHALL identify elective surgeries that can be postponed with minimal clinical risk
3. WHEN an elective surgery is postponed, THE Emergency_Handler SHALL trigger the Notification_Service to inform the affected patient within 10 minutes
4. WHEN the Code Red is resolved, THE Emergency_Handler SHALL regenerate the surgical schedule incorporating postponed cases with elevated priority
5. THE Emergency_Handler SHALL maintain a log of all Code Red events, including trigger time, affected surgeries, and resolution time

### Requirement 8: Green Corridor Capacity Management

**User Story:** As a trauma surgeon, I want dedicated emergency capacity, so that critical patients are not delayed by elective procedures.

#### Acceptance Criteria

1. THE EquiSurg_System SHALL reserve a minimum of 20% of operating room capacity as Green Corridor trauma bays
2. WHEN Green Corridor capacity falls below 15%, THE EquiSurg_System SHALL alert administrators and recommend reducing elective surgery bookings
3. THE EquiSurg_System SHALL prevent elective surgeries from being scheduled in Green Corridor trauma bays
4. WHEN no emergency cases are present for 24 hours, THE EquiSurg_System SHALL allow temporary use of Green Corridor bays for urgent elective cases with automatic reversion upon emergency arrival

### Requirement 9: Administrative Dashboard

**User Story:** As a hospital administrator, I want a visual dashboard, so that I can monitor surgical operations and system performance in real-time.

#### Acceptance Criteria

1. THE Dashboard SHALL display current day's surgical schedule with patient names, surgery types, assigned resources, and status (scheduled, in-progress, completed, cancelled)
2. THE Dashboard SHALL show real-time operating room utilization as a percentage for each room
3. THE Dashboard SHALL display a waitlist view sorted by priority score showing patient ASA Score, wait time, and predicted surgery duration
4. WHEN a Code Red is active, THE Dashboard SHALL display a prominent alert banner with emergency status and affected surgeries
5. THE Dashboard SHALL provide filters for viewing schedules by date range, surgeon, operating room, and surgery type

### Requirement 10: Multi-Language Support

**User Story:** As a patient who speaks a regional language, I want notifications in my native language, so that I can understand critical medical information.

#### Acceptance Criteria

1. THE EquiSurg_System SHALL support patient language preferences for Hindi, Kannada, Tamil, Telugu, Bengali, and English
2. WHEN a patient registers, THE EquiSurg_System SHALL allow selection of preferred notification language
3. WHEN generating notifications, THE Notification_Service SHALL use the patient's preferred language for all text and voice content
4. THE EquiSurg_System SHALL store notification templates in all supported languages with equivalent medical terminology

### Requirement 11: Schedule Persistence and Recovery

**User Story:** As a system administrator, I want schedules to be saved reliably, so that we can recover from system failures without data loss.

#### Acceptance Criteria

1. WHEN a schedule is generated, THE EquiSurg_System SHALL persist it to storage immediately
2. WHEN the system restarts, THE EquiSurg_System SHALL load the most recent schedule and resume operations
3. THE EquiSurg_System SHALL maintain backups of schedules for the past 90 days
4. WHEN a schedule is modified, THE EquiSurg_System SHALL create a versioned snapshot before applying changes

### Requirement 12: Notification Delivery Tracking

**User Story:** As a patient coordinator, I want to track notification delivery status, so that I can follow up with patients who didn't receive alerts.

#### Acceptance Criteria

1. WHEN a notification is sent, THE Notification_Service SHALL record the delivery attempt with timestamp, recipient, channel (SMS/voice), and status (sent, delivered, failed)
2. THE Dashboard SHALL display notification delivery statistics including success rate, failure rate, and average delivery time
3. WHEN a notification fails after all retries, THE Dashboard SHALL flag the patient record for manual follow-up
4. THE EquiSurg_System SHALL generate a daily report of failed notifications for administrative review

### Requirement 13: Constraint Violation Reporting

**User Story:** As a surgical scheduler, I want detailed explanations when scheduling fails, so that I can understand and resolve resource bottlenecks.

#### Acceptance Criteria

1. WHEN the Scheduler cannot find a feasible solution, THE Scheduler SHALL return a report listing all violated constraints
2. THE Scheduler SHALL identify specific resources (surgeons, rooms, equipment) that are causing bottlenecks
3. THE Scheduler SHALL suggest potential solutions such as extending operating hours, adding resources, or postponing lower-priority cases
4. THE Dashboard SHALL display constraint violation reports in a human-readable format with actionable recommendations

### Requirement 14: Historical Data Analytics

**User Story:** As a hospital administrator, I want to analyze historical surgical data, so that I can identify trends and improve resource planning.

#### Acceptance Criteria

1. THE EquiSurg_System SHALL store historical data for all completed surgeries including actual duration, scheduled duration, patient outcomes, and resource utilization
2. THE Dashboard SHALL provide analytics views showing average surgery duration by type, surgeon efficiency metrics, and operating room utilization trends
3. THE Dashboard SHALL display prediction accuracy metrics comparing predicted vs actual surgery durations
4. THE EquiSurg_System SHALL allow export of historical data in CSV format for external analysis

### Requirement 15: API Integration for External Systems

**User Story:** As a hospital IT manager, I want to integrate EquiSurg with existing hospital information systems, so that data flows seamlessly without manual entry.

#### Acceptance Criteria

1. THE EquiSurg_System SHALL provide a REST API for patient registration, surgery scheduling, and schedule retrieval
2. THE EquiSurg_System SHALL authenticate API requests using token-based authentication
3. WHEN an API request is malformed or unauthorized, THE EquiSurg_System SHALL return appropriate HTTP status codes (400, 401, 403) with error descriptions
4. THE EquiSurg_System SHALL rate-limit API requests to 100 requests per minute per client to prevent abuse
