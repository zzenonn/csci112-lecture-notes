# Problem Set: Hospital Patient Management System

**Course:** CSCI 112 - Contemporary Databases  
**Topic:** MongoDB Data Modeling  
**Format:** Oral Examination

---

## Scenario

You are designing a database for a hospital's patient management system. The system needs to efficiently handle queries that frequently involve multiple related entities together, such as retrieving a patient's complete medical history (appointments, diagnoses, prescriptions, lab results) or finding all information related to a specific treatment plan.

The current approach uses separate collections, but performance issues arise due to frequent cross-collection queries needed to assemble comprehensive patient records and treatment summaries.

---

## Dataset Categories

### Patients
- Patient ID, name, date of birth, contact information
- Insurance details, emergency contacts, medical alerts

### Doctors  
- Doctor ID, name, specialization, department
- License number, contact information, schedule

### Appointments
- Appointment ID, date and time, duration, status
- Patient, doctor, appointment type, notes

### Diagnoses
- Diagnosis ID, condition name, ICD code, severity
- Patient, diagnosing doctor, date, treatment plan

### Prescriptions
- Prescription ID, medication name, dosage, frequency
- Patient, prescribing doctor, start date, duration

### Lab Results
- Lab Result ID, test name, values, reference ranges
- Patient, ordering doctor, collection date, status

---

## Sample Data

**Patient:**
- Patient ID: P001, Name: "Sarah Johnson", DOB: "1985-03-15", Phone: "555-0123", Insurance: "HealthCare Plus"

**Doctor:**
- Doctor ID: D001, Name: "Dr. Michael Rodriguez", Specialization: "Cardiology", Department: "Internal Medicine"

**Appointment:**
- Appointment ID: A001, Date: "2024-10-05", Time: "14:00", Patient: P001, Doctor: D001, Type: "Follow-up"

**Diagnosis:**
- Diagnosis ID: DX001, Condition: "Hypertension", ICD Code: "I10", Patient: P001, Doctor: D001, Date: "2024-10-05"

**Prescription:**
- Prescription ID: RX001, Medication: "Lisinopril", Dosage: "10mg", Frequency: "Once daily", Patient: P001, Doctor: D001

**Lab Result:**
- Lab Result ID: LR001, Test: "Blood Pressure", Value: "140/90 mmHg", Patient: P001, Date: "2024-10-05"

---

## Task

Design an efficient MongoDB data model that can handle the hospital's frequent queries involving multiple related medical entities while minimizing the need for expensive join operations across different data types.

---

## Guide Questions

1. What are the primary relationships between patients, doctors, appointments, diagnoses, prescriptions, and lab results, and how do these relationships affect typical medical query patterns?

2. What are the most common query scenarios in a hospital system, and which medical entities are frequently accessed together when building patient records or treatment summaries?

3. How would you organize these related medical entities to optimize for the identified query patterns, and what field structures would support efficient retrieval of comprehensive patient information?

4. How would you distinguish between different types of medical documents while maintaining relationships, and what metadata would be necessary to support medical workflows?

5. How would you model the connections between different medical entity types to enable efficient traversal and filtering when building patient histories or treatment plans?

6. What are the trade-offs of your chosen approach compared to maintaining separate medical collections, and how does this impact query complexity for typical hospital operations?

7. How would you ensure data integrity and consistency in your proposed medical data model, and what challenges might arise with patient privacy and medical record accuracy?

8. How would your design handle growth in the number of patients, medical records, and related data over multiple years of hospital operations?
