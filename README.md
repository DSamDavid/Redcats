# Redcats
Our solution is a continuous care-coordination system that unifies patient monitoring, medication adherence, family communication, caregivers, and telehealth. Rather than simply detecting abnormal values, it intelligently converts health data into timely, appropriate care actions.
from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
from typing import List, Dict, Optional, Any
import random
import uuid


# =========================
# ENUMS
# =========================

class AlertSeverity(str, Enum):
    INFO = "info"
    WARNING = "warning"
    CRITICAL = "critical"


class ContactType(str, Enum):
    FAMILY = "family"
    CAREGIVER = "caregiver"
    DOCTOR = "doctor"
    NURSE = "nurse"


class VisitStatus(str, Enum):
    SCHEDULED = "scheduled"
    COMPLETED = "completed"
    MISSED = "missed"
    CANCELLED = "cancelled"


# =========================
# DATA MODELS
# =========================

@dataclass
class VitalSigns:
    timestamp: datetime
    heart_rate: int
    systolic_bp: int
    diastolic_bp: int
    oxygen_saturation: int
    blood_glucose: Optional[int] = None
    temperature_c: Optional[float] = None
    steps: int = 0

    def is_critical(self) -> bool:
        return (
            self.heart_rate > 120
            or self.systolic_bp > 160
            or self.diastolic_bp > 100
            or self.oxygen_saturation < 92
            or (self.blood_glucose is not None and (self.blood_glucose < 70 or self.blood_glucose > 250))
            or self.temperature_c is not None and (self.temperature_c < 35.5 or self.temperature_c > 38.5)
        )

    def to_dict(self) -> Dict[str, Any]:
        return {
            "timestamp": self.timestamp.isoformat(),
            "heart_rate": self.heart_rate,
            "systolic_bp": self.systolic_bp,
            "diastolic_bp": self.diastolic_bp,
            "oxygen_saturation": self.oxygen_saturation,
            "blood_glucose": self.blood_glucose,
            "temperature_c": self.temperature_c,
            "steps": self.steps,
        }


@dataclass
class Medication:
    name: str
    dose: str
    schedule_time: str  # e.g., "08:00"
    purpose: str
    taken_today: bool = False

    def mark_taken(self):
        self.taken_today = True


@dataclass
class Contact:
    contact_id: str
    name: str
    relationship: str
    phone: str
    email: str
    type: ContactType
    is_primary: bool = False

    def to_dict(self) -> Dict[str, Any]:
        return {
            "contact_id": self.contact_id,
            "name": self.name,
            "relationship": self.relationship,
            "phone": self.phone,
            "email": self.email,
            "type": self.type.value,
            "is_primary": self.is_primary,
        }


@dataclass
class TelehealthSession:
    session_id: str
    patient_id: str
    doctor_name: str
    scheduled_time: datetime
    status: VisitStatus = VisitStatus.SCHEDULED
    notes: str = ""
    follow_up_required: bool = False

    def complete(self, notes: str, follow_up_required: bool = False):
        self.status = VisitStatus.COMPLETED
        self.notes = notes
        self.follow_up_required = follow_up_required


@dataclass
class Alert:
    alert_id: str
    patient_id: str
    severity: AlertSeverity
    category: str
    message: str
    created_at: datetime
    acknowledged: bool = False
    acknowledged_by: Optional[str] = None

    def acknowledge(self, person_name: str):
        self.acknowledged = True
        self.acknowledged_by = person_name


@dataclass
class CarePlan:
    patient_id: str
    chronic_conditions: List[str]
    medications: List[Medication] = field(default_factory=list)
    goals: List[str] = field(default_factory=list)
    monitoring_frequency_min: int = 30
    emergency_contacts: List[Contact] = field(default_factory=list)
    care_team: List[Contact] = field(default_factory=list)

    def add_medication(self, medication: Medication):
        self.medications.append(medication)

    def add_goal(self, goal: str):
        self.goals.append(goal)


@dataclass
class Patient:
    patient_id: str
    name: str
    age: int
    residence: str
    primary_diagnosis: str
    care_plan: CarePlan
    vitals_history: List[VitalSigns] = field(default_factory=list)
    alerts: List[Alert] = field(default_factory=list)
    telehealth_sessions: List[TelehealthSession] = field(default_factory=list)

    def add_vitals(self, vital_signs: VitalSigns):
        self.vitals_history.append(vital_signs)

    def add_alert(self, alert: Alert):
        self.alerts.append(alert)

    def add_session(self, session: TelehealthSession):
        self.telehealth_sessions.append(session)

    def latest_vitals(self) -> Optional[VitalSigns]:
        return self.vitals_history[-1] if self.vitals_history else None

    def medication_status(self) -> Dict[str, bool]:
        return {m.name: m.taken_today for m in self.care_plan.medications}


# =========================
# MONITORING & ALERT SYSTEM
# =========================

class AlertManager:
    def __init__(self):
        self.alert_log: List[Alert] = []

    def create_alert(self, patient: Patient, category: str, message: str, severity: AlertSeverity) -> Alert:
        alert = Alert(
            alert_id=str(uuid.uuid4()),
            patient_id=patient.patient_id,
            severity=severity,
            category=category,
            message=message,
            created_at=datetime.now(),
        )
        patient.add_alert(alert)
        self.alert_log.append(alert)
        return alert

    def summarize_alerts(self, patient: Patient) -> List[str]:
        summaries = []
        for alert in patient.alerts:
            summaries.append(f"[{alert.severity.value}] {alert.category}: {alert.message}")
        return summaries


class RemoteMonitoringEngine:
    def __init__(self, alert_manager: AlertManager):
        self.alert_manager = alert_manager

    def evaluate_patient(self, patient: Patient) -> List[Alert]:
        latest = patient.latest_vitals()
        if not latest:
            return []

        generated_alerts: List[Alert] = []

        if latest.is_critical():
            alert = self.alert_manager.create_alert(
                patient,
                category="Vital Sign Critical",
                message=(
                    f"Critical care warning for {patient.name}: HR={latest.heart_rate}, "
                    f"BP={latest.systolic_bp}/{latest.diastolic_bp}, "
                    f"O2={latest.oxygen_saturation}%, glucose={latest.blood_glucose}"
                ),
                severity=AlertSeverity.CRITICAL,
            )
            generated_alerts.append(alert)

        elif latest.heart_rate > 95 or latest.systolic_bp > 140 or latest.oxygen_saturation < 95:
            alert = self.alert_manager.create_alert(
                patient,
                category="Monitoring Warning",
                message=(
                    f"Elevated risk indicators for {patient.name}: "
                    f"HR={latest.heart_rate}, BP={latest.systolic_bp}/{latest.diastolic_bp}, "
                    f"O2={latest.oxygen_saturation}%"
                ),
                severity=AlertSeverity.WARNING,
            )
            generated_alerts.append(alert)

        # medication adherence check
        non_adherent = [m.name for m in patient.care_plan.medications if not m.taken_today]
        if non_adherent:
            alert = self.alert_manager.create_alert(
                patient,
                category="Medication Adherence",
                message=f"Medication reminder: {', '.join(non_adherent)} not yet recorded as taken.",
                severity=AlertSeverity.INFO,
            )
            generated_alerts.append(alert)

        return generated_alerts


class FamilyCommunicationHub:
    def __init__(self):
        self.message_log: List[str] = []

    def notify_family(self, patient: Patient, message: str):
        self.message_log.append(f"{datetime.now().isoformat()} | {patient.name}: {message}")
        print(f"[Family Update] {patient.name}: {message}")

    def distribute_alerts(self, patient: Patient, alerts: List[Alert]):
        for alert in alerts:
            for contact in patient.care_plan.emergency_contacts:
                self.notify_family(
                    patient,
                    f"Alert: {alert.category} - {alert.message} | Contact: {contact.name} ({contact.relationship})"
                )


# =========================
# CARE COORDINATION
# =========================

class CareCoordinator:
    def __init__(self):
        self.patients: Dict[str, Patient] = {}
        self.monitoring_engine = RemoteMonitoringEngine(AlertManager())
        self.family_hub = FamilyCommunicationHub()
        self.telehealth_schedule: List[TelehealthSession] = []

    def add_patient(self, patient: Patient):
        self.patients[patient.patient_id] = patient

    def schedule_telehealth(self, patient: Patient, doctor_name: str, scheduled_time: datetime):
        session = TelehealthSession(
            session_id=str(uuid.uuid4()),
            patient_id=patient.patient_id,
            doctor_name=doctor_name,
            scheduled_time=scheduled_time,
        )
        patient.add_session(session)
        self.telehealth_schedule.append(session)

    def mark_session_complete(self, session_id: str, notes: str, follow_up_required: bool = False):
        for session in self.telehealth_schedule:
            if session.session_id == session_id:
                session.complete(notes, follow_up_required)
                return True
        return False

    def run_monitoring_cycle(self):
        for patient in self.patients.values():
            alerts = self.monitoring_engine.evaluate_patient(patient)
            if alerts:
                self.family_hub.distribute_alerts(patient, alerts)
                print(f"Generated {len(alerts)} alerts for {patient.name}")

    def create_daily_summary(self, patient: Patient) -> Dict[str, Any]:
        latest = patient.latest_vitals()
        summary = {
            "patient_id": patient.patient_id,
            "patient_name": patient.name,
            "latest_vitals": latest.to_dict() if latest else None,
            "medication_status": patient.medication_status(),
            "alerts": [a.message for a in patient.alerts[-5:]],
            "next_session": None,
        }

        upcoming = sorted(
            [s for s in patient.telehealth_sessions if s.status == VisitStatus.SCHEDULED],
            key=lambda s: s.scheduled_time
        )
        if upcoming:
            next_session = upcoming[0]
            summary["next_session"] = {
                "doctor_name": next_session.doctor_name,
                "scheduled_time": next_session.scheduled_time.isoformat(),
                "session_id": next_session.session_id,
            }

        return summary


# =========================
# DEMO DATA / SIMULATION
# =========================

def generate_sample_patient() -> Patient:
    care_plan = CarePlan(
        patient_id="P-1001",
        chronic_conditions=["Hypertension", "Type 2 Diabetes", "Arthritis"],
        goals=[
            "Maintain blood pressure under control",
            "Stay hydrated and mobile",
            "Medication compliance",
            "Track symptoms daily"
        ],
        monitoring_frequency_min=30,
        emergency_contacts=[
            Contact("C-1", "Maria Gomez", "Daughter", "+1-555-0101", "maria@gmail.com", ContactType.FAMILY, True),
            Contact("C-2", "Daniel Gomez", "Son", "+1-555-0102", "daniel@gmail.com", ContactType.FAMILY, False)
        ],
        care_team=[
            Contact("D-1", "Dr. Anita Reyes", "Primary Physician", "+1-555-0201", "dr.reyes@clinic.org", ContactType.DOCTOR, True),
            Contact("N-1", "Nurse Sara Lin", "Home Health Nurse", "+1-555-0202", "sara@care.org", ContactType.NURSE, False)
        ]
    )
    care_plan.add_medication(Medication("Amlodipine", "5mg", "08:00", "Blood pressure"))
    care_plan.add_medication(Medication("Metformin", "500mg", "12:00", "Blood sugar"))
    care_plan.add_medication(Medication("Vitamin D", "1000IU", "18:00", "Bone health"))

    patient = Patient(
        patient_id="P-1001",
        name="Edward Collins",
        age=79,
        residence="Oakwood Senior Living",
        primary_diagnosis="Hypertension and diabetes management for elderly patient",
        care_plan=care_plan,
    )

    # Simulate vital signs from recent monitoring
    now = datetime.now()
    patient.add_vitals(VitalSigns(now - timedelta(minutes=45), 88, 138, 86, 94, 128, 36.9, 4230))
    patient.add_vitals(VitalSigns(now - timedelta(minutes=15), 102, 152, 89, 91, 162, 37.5, 2180))

    return patient


def simulate_remotely_detected_risk():
    patient = generate_sample_patient()
    coordinator = CareCoordinator()
    coordinator.add_patient(patient)

    # Schedule telehealth session
    coordinator.schedule_telehealth(patient, "Dr. Anita Reyes", datetime.now() + timedelta(hours=2))

    # Simulate medication adherence issue and monitoring cycle
    patient.care_plan.medications[0].mark_taken()
    patient.care_plan.medications[1].taken_today = False
    patient.care_plan.medications[2].taken_today = False

    # Run monitoring engine
    coordinator.run_monitoring_cycle()

    # Print summary
    print("\n" + "=" * 80)
    print("DAILY CARE SUMMARY")
    print("=" * 80)
    print(coordinator.create_daily_summary(patient))

    # Example of completing telehealth session
    session = patient.telehealth_sessions[0]
    coordinator.mark_session_complete(
        session.session_id,
        "Patient reports dizziness in morning. Recommends hydration check and blood pressure re-test before evening medications.",
        follow_up_required=True
    )

    print("\nUpdated Telehealth Session:")
    print(session)


if __name__ == "__main__":
    simulate_remotely_detected_risk()