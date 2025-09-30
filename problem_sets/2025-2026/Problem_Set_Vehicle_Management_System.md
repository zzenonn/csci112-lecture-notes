# Problem Set: Vehicle Management System

**Course:** CSCI 112 / 212 - Contemporary Databases  
**Topic:** MongoDB Data Modeling Challenge  
**Format:** Oral Examination Discussion

---

## Scenario

You are tasked with designing a MongoDB database for **AutoHub**, a comprehensive vehicle management platform that handles different types of vehicles for various business purposes. The system needs to manage data for three distinct vehicle categories that the company deals with:

### Vehicle Categories

**1. Personal Cars**
- Vehicle identification number (VIN)
- Make and model
- Year of manufacture
- Owner information (name, contact)
- Registration details
- Number of seats
- Fuel type (gasoline, diesel, hybrid, electric)
- Mileage
- Insurance policy number
- Last service date

**2. Commercial Trucks**
- Vehicle identification number (VIN)
- Make and model
- Year of manufacture
- Company owner information
- Registration details
- Cargo capacity (in tons)
- Truck type (delivery, freight, refrigerated)
- Commercial license requirements
- DOT number
- Last inspection date
- Route restrictions

**3. Motorcycles**
- Vehicle identification number (VIN)
- Make and model
- Year of manufacture
- Owner information (name, contact)
- Registration details
- Engine displacement (cc)
- Motorcycle type (sport, cruiser, touring, dirt)
- License class required
- Helmet storage availability
- Last maintenance date

---

## Sample Data

### Personal Cars
1. **Honda Civic 2020**
   - VIN: 1HGBH41JXMN109186
   - Owner: Maria Santos, phone: 09171234567
   - Registration: ABC-1234, expires: 2024-12-15
   - 5 seats, gasoline, 45,000 km
   - Insurance: POL-789456123
   - Last service: 2024-08-15

2. **Tesla Model 3 2022**
   - VIN: 5YJ3E1EA4NF123456
   - Owner: John Dela Cruz, phone: 09189876543
   - Registration: XYZ-5678, expires: 2025-03-20
   - 5 seats, electric, 28,000 km
   - Insurance: POL-456789012
   - Last service: 2024-09-10

### Commercial Trucks
1. **Isuzu NPR 2019**
   - VIN: JALC4B16K97123456
   - Company: FastDelivery Corp, contact: 02-8123-4567
   - Registration: TRK-9012, expires: 2024-11-30
   - 3.5 tons capacity, delivery truck
   - Commercial license: Class 3 required
   - DOT: DOT-567890123
   - Last inspection: 2024-09-05
   - Route: Metro Manila only

2. **Volvo FH16 2021**
   - VIN: YV2A2A18XMA123456
   - Company: LongHaul Logistics, contact: 02-8765-4321
   - Registration: HVY-3456, expires: 2025-01-15
   - 25 tons capacity, freight truck
   - Commercial license: Class 1 required
   - DOT: DOT-234567890
   - Last inspection: 2024-08-28
   - Route: Nationwide

### Motorcycles
1. **Yamaha YZF-R3 2021**
   - VIN: JYARN37E0MA123456
   - Owner: Carlos Reyes, phone: 09123456789
   - Registration: MCY-7890, expires: 2024-10-25
   - 321cc, sport motorcycle
   - License: Class A1 required
   - Helmet storage: Yes
   - Last maintenance: 2024-09-12

2. **Honda PCX 160 2023**
   - VIN: MLHKE1810P5123456
   - Owner: Ana Garcia, phone: 09234567890
   - Registration: SCT-2345, expires: 2025-05-18
   - 157cc, scooter
   - License: Class A required
   - Helmet storage: Yes
   - Last maintenance: 2024-09-20

---

## Task

Design a MongoDB data model for this vehicle management system. Consider how you would structure the documents to efficiently store and query this diverse vehicle data.

---

## Guide Questions for Oral Examination

1. How would you structure the MongoDB collection(s) for this vehicle data? Explain your reasoning.

2. Identify which fields are common across all vehicle types and which are unique to specific categories. How does this influence your design?

3. Would you use one collection or multiple collections? Justify your choice with specific advantages and disadvantages.

4. Show how you would represent one document from each vehicle category in your proposed schema.

5. How would your design handle these common queries:
   - Find all vehicles owned by a specific person
   - Get all vehicles due for service/inspection in the next 30 days
   - List all commercial trucks with cargo capacity over 10 tons

6. Some fields have similar purposes but different names (e.g., "Last service date" vs "Last inspection date" vs "Last maintenance date"). How would you handle this in your schema?

7. How would your design handle adding new vehicle types (e.g., boats, aircraft) in the future?

8. How does your chosen approach affect storage space and document organization?
