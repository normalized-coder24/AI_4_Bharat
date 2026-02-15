# Design Document: EquiSurg

## Overview

EquiSurg is a multi-layered system that combines machine learning prediction, constraint programming optimization, and low-bandwidth communication to provide equitable surgical scheduling for public healthcare systems. The architecture is designed to handle the complexity of resource allocation while maintaining accessibility for marginalized communities and resilience during emergencies.

The system processes patient data through a predictive layer (XGBoost), optimizes schedules using constraint programming (Google OR-Tools CP-SAT), communicates with patients via low-bandwidth channels (SMS/WhatsApp), and provides administrative oversight through a web dashboard (Streamlit).

## Architecture

### System Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    Dashboard Layer (Streamlit)               │
│  - Schedule Visualization  - Analytics  - Code Red Controls  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Application Core Layer                    │
│  - Patient Management  - Schedule Coordinator                │
│  - Emergency Handler   - Notification Orchestrator           │
└─────────────────────────────────────────────────────────────┘
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
┌──────────────────┐  ┌──────────────┐  ┌──────────────────┐
│  Predictive      │  │ Optimization │  │  Notification    │
│  Layer           │  │ Layer        │  │  Layer           │
│  (XGBoost)       │  │ (OR-Tools)   │  │  (Twilio/SMS)    │
└──────────────────┘  └──────────────┘  └──────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Data Persistence Layer                    │
│  - Patient Records  - Schedules  - Audit Logs  - Analytics  │
└─────────────────────────────────────────────────────────────┘
```

### Component Interaction Flow

1. **Patient Registration**: Dashboard → Patient Management → Data Persistence
2. **Schedule Generation**: Patient Management → Triage Engine (Prediction) → Scheduler (Optimization) → Data Persistence
3. **Notification**: Schedule Coordinator → Notification Service → External SMS/Voice API
4. **Emergency Handling**: Dashboard (Code Red Trigger) → Emergency Handler → Scheduler (Re-optimization) → Notification Service

## Components and Interfaces

### 1. Patient Management Component

**Responsibility**: Manage patient data lifecycle including registration, validation, updates, and retrieval.

**Interface**:
```python
class PatientManager:
    def register_patient(patient_data: PatientData) -> PatientID:
        """
        Register a new patient with validation.
        
        Args:
            patient_data: PatientData containing demographics, physiology, clinical risk
        
        Returns:
            PatientID: Unique identifier for the registered patient
        
        Raises:
            ValidationError: If patient data fails validation rules
        """
        pass
    
    def update_patient(patient_id: PatientID, updates: PatientData) -> bool:
        """
        Update existing patient information with audit trail.
        
        Args:
            patient_id: Unique patient identifier
            updates: Partial or complete patient data updates
        
        Returns:
            bool: True if update successful
        
        Raises:
            PatientNotFoundError: If patient_id does not exist
            ValidationError: If updates fail validation
        """
        pass
    
    def get_patient(patient_id: PatientID) -> PatientData:
        """Retrieve patient data by ID."""
        pass
    
    def get_waitlist() -> List[PatientData]:
        """Retrieve all patients awaiting surgery, sorted by priority."""
        pass
```

**Validation Rules**:
- Age: positive integer (1-120)
- BMI: float between 10.0 and 60.0
- ASA Score: integer between 1 and 5
- Phone number: valid format for SMS delivery
- Language preference: one of supported languages (Hindi, Kannada, Tamil, Telugu, Bengali, English)

### 2. Triage Engine (Predictive Layer)

**Responsibility**: Predict surgery duration using XGBoost model trained on historical surgical data.

**Interface**:
```python
class TriageEngine:
    def __init__(model_path: str):
        """Load pre-trained XGBoost model from disk."""
        pass
    
    def predict_duration(patient: PatientData, surgery_type: SurgeryType) -> PredictionResult:
        """
        Predict surgery duration with confidence interval.
        
        Args:
            patient: Patient physiological and clinical data
            surgery_type: Type of surgical procedure
        
        Returns:
            PredictionResult containing:
                - predicted_minutes: int
                - confidence_lower: int (minutes)
                - confidence_upper: int (minutes)
                - confidence_level: float (0.0-1.0)
        
        Raises:
            MissingDataError: If required patient fields are absent
        """
        pass
    
    def retrain_model(historical_data: List[SurgeryRecord]) -> ModelMetrics:
        """
        Retrain XGBoost model on updated historical data.
        
        Returns:
            ModelMetrics containing MAE, RMSE, R² score
        """
        pass
```

**Model Features**:
- Patient age
- Patient BMI
- ASA Score
- Surgery type (categorical encoding)
- Historical surgeon efficiency (optional)

**Model Output**: Duration in minutes with 95% confidence interval

### 3. Scheduler (Optimization Layer)

**Responsibility**: Generate optimal surgical schedules using constraint programming while respecting resource constraints and clinical priorities.

**Interface**:
```python
class Scheduler:
    def generate_schedule(
        patients: List[PatientData],
        resources: ResourceAvailability,
        constraints: SchedulingConstraints,
        planning_horizon_days: int
    ) -> ScheduleResult:
        """
        Generate optimized surgical schedule using CP-SAT solver.
        
        Args:
            patients: List of patients awaiting surgery with predictions
            resources: Available surgeons, rooms, equipment by time slot
            constraints: Hard constraints (conflicts, buffers) and soft constraints (priorities)
            planning_horizon_days: Number of days to schedule ahead
        
        Returns:
            ScheduleResult containing:
                - schedule: List[ScheduledSurgery] if feasible
                - status: "OPTIMAL" | "FEASIBLE" | "INFEASIBLE"
                - objective_value: float (optimization score)
                - constraint_violations: List[ConstraintViolation] if infeasible
        """
        pass
    
    def reschedule_emergency(
        current_schedule: Schedule,
        emergency_case: PatientData,
        code_red_resources: ResourceAvailability
    ) -> RescheduleResult:
        """
        Dynamically reschedule to accommodate emergency case.
        
        Returns:
            RescheduleResult containing:
                - updated_schedule: Schedule
                - postponed_surgeries: List[ScheduledSurgery]
                - emergency_slot: TimeSlot
        """
        pass
```

**Constraint Programming Model**:

**Decision Variables**:
- `surgery_start[i]`: Start time of surgery i (integer variable in minutes from planning start)
- `room_assigned[i, r]`: Binary variable, 1 if surgery i assigned to room r
- `surgeon_assigned[i, s]`: Binary variable, 1 if surgery i assigned to surgeon s

**Hard Constraints**:
1. **No Resource Overlap**: For any two surgeries i, j assigned to same room r:
   - `surgery_start[i] + duration[i] + buffer <= surgery_start[j]` OR `surgery_start[j] + duration[j] + buffer <= surgery_start[i]`
2. **Surgeon Availability**: Surgeon s can only be assigned to one surgery at a time
3. **Room Capacity**: Each surgery assigned to exactly one room
4. **Green Corridor Reservation**: 20% of rooms reserved for emergency use (not assignable to elective surgeries)
5. **Operating Hours**: All surgeries must start and end within daily operating hours (8 AM - 8 PM)

**Soft Constraints (Optimization Objectives)**:
1. **Primary**: Maximize number of high-priority surgeries (ASA 4-5) scheduled
2. **Secondary**: Minimize total patient wait time (weighted by ASA score)
3. **Tertiary**: Maximize overall room utilization

**Objective Function**:
```
maximize: 
  1000 * (count of ASA 4-5 surgeries scheduled) 
  - 10 * (sum of wait_days * ASA_score for all patients)
  + 1 * (total room utilization percentage)
```

### 4. Notification Service

**Responsibility**: Deliver low-bandwidth notifications to patients via SMS and voice calls in local languages.

**Interface**:
```python
class NotificationService:
    def __init__(api_key: str, provider: str):
        """Initialize SMS/voice provider (Twilio or Fast2SMS)."""
        pass
    
    def send_notification(
        patient: PatientData,
        notification_type: NotificationType,
        content: NotificationContent
    ) -> DeliveryResult:
        """
        Send notification via patient's preferred channel and language.
        
        Args:
            patient: Patient data including phone and language preference
            notification_type: "SCHEDULE_CONFIRMED" | "SCHEDULE_CHANGED" | "REMINDER" | "POSTPONED"
            content: Structured notification data (surgery time, location, instructions)
        
        Returns:
            DeliveryResult containing:
                - status: "SENT" | "DELIVERED" | "FAILED"
                - timestamp: datetime
                - delivery_id: str (provider tracking ID)
                - retry_count: int
        """
        pass
    
    def retry_failed_notifications() -> List[DeliveryResult]:
        """Retry all failed notifications with exponential backoff."""
        pass
    
    def get_delivery_status(delivery_id: str) -> DeliveryStatus:
        """Query provider for delivery confirmation."""
        pass
```

**Notification Templates** (per language):
- **Schedule Confirmed**: "Your surgery is scheduled for {date} at {time}. Please report to {location} by {arrival_time}. Fasting required from {fasting_start}."
- **Schedule Changed**: "Your surgery has been rescheduled to {new_date} at {new_time}. Previous date was {old_date}. Please call {contact_number} if you have questions."
- **Postponed (Emergency)**: "Due to an emergency, your surgery on {date} has been postponed. We will contact you within 24 hours with a new date. We apologize for the inconvenience."
- **Reminder**: "Reminder: Your surgery is tomorrow at {time}. Please arrive by {arrival_time} and follow fasting instructions."

**Retry Logic**:
- Attempt 1: Immediate
- Attempt 2: 1 minute after failure
- Attempt 3: 5 minutes after failure
- Attempt 4: 15 minutes after failure
- After 4 failures: Flag for manual follow-up

### 5. Emergency Handler

**Responsibility**: Manage Code Red scenarios by reallocating resources and rescheduling elective surgeries.

**Interface**:
```python
class EmergencyHandler:
    def declare_code_red(emergency_details: EmergencyDetails) -> CodeRedSession:
        """
        Initiate Code Red protocol.
        
        Args:
            emergency_details: Type, severity, estimated patient count
        
        Returns:
            CodeRedSession with session ID and allocated resources
        """
        pass
    
    def allocate_emergency_resources(
        session: CodeRedSession,
        emergency_patient: PatientData
    ) -> EmergencyAllocation:
        """
        Allocate Green Corridor resources to emergency patient.
        
        Returns:
            EmergencyAllocation containing assigned room, surgeon, time slot
        """
        pass
    
    def identify_postponable_surgeries(
        current_schedule: Schedule,
        required_resources: ResourceRequirement
    ) -> List[ScheduledSurgery]:
        """
        Identify elective surgeries that can be safely postponed.
        
        Prioritizes postponing:
        - Lower ASA scores (1-2)
        - Shorter wait times
        - Non-urgent procedures
        """
        pass
    
    def resolve_code_red(session: CodeRedSession) -> ResolutionReport:
        """
        End Code Red and trigger schedule regeneration.
        
        Returns:
            ResolutionReport with statistics and postponed surgery list
        """
        pass
```

**Code Red Protocol**:
1. Immediately allocate all Green Corridor trauma bays
2. If insufficient, identify postponable elective surgeries (ASA 1-2, shortest wait times)
3. Notify affected patients within 10 minutes via Notification Service
4. Log all actions with timestamps for audit trail
5. Upon resolution, regenerate schedule with postponed surgeries at elevated priority

### 6. Dashboard (Streamlit Interface)

**Responsibility**: Provide administrative interface for monitoring, control, and analytics.

**Pages**:

**A. Today's Schedule View**:
- Table showing: Patient Name, Surgery Type, Room, Surgeon, Start Time, Duration, Status
- Status indicators: Scheduled (blue), In Progress (yellow), Completed (green), Cancelled (red)
- Real-time updates via polling (30-second refresh)
- Code Red alert banner (if active)

**B. Waitlist View**:
- Sortable table: Patient Name, ASA Score, Wait Days, Surgery Type, Predicted Duration
- Priority score calculation: `ASA_Score * 100 + Wait_Days`
- Filter by ASA score, surgery type
- "Generate Schedule" button to trigger optimization

**C. Resource Utilization View**:
- Operating room utilization chart (percentage per room)
- Surgeon workload distribution
- Green Corridor capacity gauge (current emergency capacity vs threshold)
- Historical trends (7-day, 30-day views)

**D. Analytics Dashboard**:
- Prediction accuracy: Predicted vs Actual duration scatter plot
- Average wait time by ASA score
- Surgery completion rate (scheduled vs completed)
- Notification delivery success rate

**E. Code Red Control Panel**:
- "Declare Code Red" button with emergency type selection
- Active Code Red status display
- List of postponed surgeries
- "Resolve Code Red" button

**F. Settings**:
- Configure operating hours
- Set Green Corridor capacity percentage
- Manage surgeon and room availability
- Configure notification retry parameters

## Data Models

### PatientData
```python
@dataclass
class PatientData:
    patient_id: str  # Unique identifier (UUID)
    name: str
    age: int  # 1-120
    gender: str  # "M" | "F" | "Other"
    bmi: float  # 10.0-60.0
    weight_kg: float
    asa_score: int  # 1-5
    phone_number: str
    language_preference: str  # "Hindi" | "Kannada" | "Tamil" | "Telugu" | "Bengali" | "English"
    notification_channel: str  # "SMS" | "Voice" | "Both"
    surgery_type: str
    registration_date: datetime
    last_updated: datetime
    updated_by: str  # User ID for audit trail
```

### ScheduledSurgery
```python
@dataclass
class ScheduledSurgery:
    surgery_id: str  # Unique identifier
    patient_id: str
    surgery_type: str
    predicted_duration_minutes: int
    scheduled_start: datetime
    scheduled_end: datetime
    assigned_room: str
    assigned_surgeon: str
    required_equipment: List[str]
    status: str  # "Scheduled" | "InProgress" | "Completed" | "Cancelled" | "Postponed"
    actual_start: Optional[datetime]
    actual_end: Optional[datetime]
    actual_duration_minutes: Optional[int]
    notes: str
```

### ResourceAvailability
```python
@dataclass
class ResourceAvailability:
    date: date
    operating_rooms: List[OperatingRoom]
    surgeons: List[Surgeon]
    equipment: Dict[str, int]  # Equipment type -> available count
    
@dataclass
class OperatingRoom:
    room_id: str
    room_type: str  # "Standard" | "GreenCorridor"
    available_slots: List[TimeSlot]
    equipment_installed: List[str]
    
@dataclass
class Surgeon:
    surgeon_id: str
    name: str
    specializations: List[str]
    available_slots: List[TimeSlot]
    efficiency_rating: float  # Historical metric for duration adjustment
```

### NotificationContent
```python
@dataclass
class NotificationContent:
    surgery_date: date
    surgery_time: time
    location: str  # Ward/Room number
    arrival_time: time
    fasting_start: time
    contact_number: str
    additional_instructions: str
```

### CodeRedSession
```python
@dataclass
class CodeRedSession:
    session_id: str
    declared_at: datetime
    declared_by: str  # User ID
    emergency_type: str  # "Accident" | "MassCasualty" | "Trauma" | "Other"
    severity: str  # "Critical" | "High" | "Moderate"
    estimated_patient_count: int
    allocated_resources: List[str]  # Room IDs
    postponed_surgeries: List[str]  # Surgery IDs
    resolved_at: Optional[datetime]
    resolution_notes: str
```

### ConstraintViolation
```python
@dataclass
class ConstraintViolation:
    constraint_type: str  # "ResourceConflict" | "InsufficientCapacity" | "TimeWindowViolation"
    description: str
    affected_resources: List[str]
    suggested_resolution: str
```

