# Problem Set: University Course Management System

**Course:** CSCI 112 - Contemporary Databases  
**Topic:** MongoDB Data Modeling  
**Format:** Oral Examination

---

## Scenario

You are designing a MongoDB database for a university's course management system. The system needs to handle student enrollments, course offerings, and instructor assignments efficiently. The application frequently displays course rosters with basic student information, generates class lists for instructors, and creates enrollment reports.

---

## Dataset Categories

### Students
- Student ID, full name, email, phone number
- Academic program, year level, GPA
- Home address, emergency contact information
- Financial aid status, scholarship details

### Courses  
- Course code, title, description, credit units
- Department, prerequisites, course level
- Maximum enrollment capacity, room assignments

### Instructors
- Employee ID, full name, email, office location
- Department affiliation, academic rank, specializations
- Office hours, contact preferences

### Enrollments
- Enrollment records linking students to specific course sections
- Enrollment date, status (enrolled, waitlisted, dropped)
- Grade information, attendance tracking

---

## Sample Data

### Student Records
- Maria Santos (ID: 211234): BS Computer Science, 3rd year, GPA 3.75, Manila address
- John Rivera (ID: 221567): BS Management Information Systems, 2nd year, GPA 3.50, Quezon City address  
- Ana Dela Cruz (ID: 201890): BS Computer Science, 4th year, GPA 3.90, Makati address
- Carlos Mendoza (ID: 231122): BS Management Information Systems, 1st year, GPA 3.25, Pasig address

### Course Offerings
- CSCI 30: Data Structures and Algorithms (3 units), Computer Science Department
- CSCI 21: Introduction to Programming I (3 units), Computer Science Department
- MSYS 25: Introduction to Information Technology (3 units), Management Information Systems Department
- MSYS 40: Business Applications Development 1 (3 units), Management Information Systems Department

### Instructor Information
- Dr. Elena Rodriguez: Computer Science, Associate Professor, Database Systems specialist
- Prof. Miguel Torres: Management Information Systems, Assistant Professor, Software Engineering focus

---

## Task

Design a MongoDB database schema for this university course management system. Consider how the application will frequently need to display course rosters with student names and programs, generate instructor class lists, and create enrollment reports. Your design should optimize for these common read operations while maintaining data integrity.

---

## Guide Questions

1. How would you structure the collections to represent students, courses, instructors, and enrollments? What are the key relationships between these entities?

2. What information would be most frequently accessed together when displaying course rosters or generating class lists? How might this influence your schema design?

3. What are the potential challenges of storing all related information in completely separate collections when generating reports?

4. If the application frequently displays student names and programs alongside enrollment information, how would you optimize the database structure to reduce query complexity?

5. What trade-offs would you consider when deciding whether to store instructor details separately or include some instructor information with course data?

6. How would you balance the need for data consistency with query performance in your proposed schema?

7. What factors would influence your decision about which student or instructor fields to include in enrollment-related documents?

8. How would your design handle scenarios where student information (like program or contact details) might change after enrollment?
