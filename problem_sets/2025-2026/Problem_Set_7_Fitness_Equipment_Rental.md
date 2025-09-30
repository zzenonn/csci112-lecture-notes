# Problem Set 8: Fitness Equipment Rental System

**Course:** CSCI 112 - Contemporary Databases  
**Topic:** MongoDB Data Modeling  
**Format:** Oral Examination

---

## Scenario

You are working with a fitness equipment rental company that has been storing their inventory data in separate systems. They have provided you with existing JSON data from three different equipment tracking systems that need to be consolidated into a single MongoDB collection for better management and querying.

The company rents out various types of fitness equipment: cardio machines, strength training equipment, and yoga/flexibility accessories. Each type has common rental information but also unique specifications that customers need to know before renting.

---

## Existing JSON Data

The company has provided the following JSON data from their current systems:

### Cardio Equipment Data
```json
[
  {
    "_id": 1,
    "equipment_id": "CARD001",
    "name": "ProForm Treadmill X22i",
    "brand": "ProForm",
    "rental_price_daily": 45.00,
    "availability_status": "available",
    "location_warehouse": "North Branch",
    "max_speed_mph": 12,
    "incline_range": "0-40%",
    "display_type": "22-inch HD touchscreen",
    "heart_rate_monitor": true,
    "power_requirements": "110V"
  },
  {
    "_id": 2,
    "equipment_id": "CARD002", 
    "name": "Schwinn IC4 Indoor Bike",
    "brand": "Schwinn",
    "rental_price_daily": 35.00,
    "availability_status": "rented",
    "location_warehouse": "South Branch",
    "resistance_levels": 100,
    "flywheel_weight_lbs": 40,
    "display_type": "LCD console",
    "bluetooth_enabled": true,
    "max_user_weight_lbs": 330
  }
]
```

### Strength Equipment Data
```json
[
  {
    "_id": 3,
    "item_id": "STR001",
    "product_name": "Bowflex SelectTech 552 Dumbbells",
    "manufacturer": "Bowflex",
    "daily_rate": 25.00,
    "status": "available",
    "warehouse": "North Branch",
    "weight_range_lbs": "5-52.5 per dumbbell",
    "adjustment_mechanism": "dial system",
    "space_required_sqft": 4,
    "warranty_months": 24
  },
  {
    "_id": 4,
    "item_id": "STR002",
    "product_name": "Total Gym XLS",
    "manufacturer": "Total Gym",
    "daily_rate": 40.00,
    "status": "maintenance",
    "warehouse": "East Branch", 
    "resistance_type": "body weight percentage",
    "exercise_count": 80,
    "folded_dimensions": "19W x 43L x 9H inches",
    "max_user_height": "6'8\"",
    "included_accessories": ["wing attachment", "squat stand", "leg pull accessory"]
  }
]
```

### Yoga/Flexibility Equipment Data
```json
[
  {
    "_id": 5,
    "yoga_id": "YOGA001",
    "title": "Manduka PRO Yoga Mat",
    "brand_name": "Manduka",
    "price_per_day": 8.00,
    "current_status": "available",
    "storage_location": "South Branch",
    "mat_thickness_mm": 6,
    "material": "PVC",
    "dimensions": "71L x 24W inches",
    "grip_level": "superior",
    "eco_friendly": false
  },
  {
    "_id": 6,
    "yoga_id": "YOGA002", 
    "title": "Gaiam Yoga Block Set",
    "brand_name": "Gaiam",
    "price_per_day": 5.00,
    "current_status": "available",
    "storage_location": "North Branch",
    "block_count": 2,
    "material": "EVA foam",
    "dimensions_per_block": "9L x 6W x 4H inches",
    "density": "firm",
    "color_options": ["purple", "blue", "pink"]
  }
]
```

---

## Task 1: Data Model Design

Design a MongoDB data model that consolidates this diverse fitness equipment data into a single collection while preserving all the unique characteristics of each equipment type. Your solution should handle the inconsistent field naming and structure across the different equipment categories.

---

## Task 2: Data Pipeline Implementation

Create MongoDB aggregation pipelines that transform the existing JSON data from the three separate collections (cardio_equipment, strength_equipment, yoga_equipment) into your unified equipment_inventory collection following your proposed data model structure.

---

## Guide Questions

1. What are the common fields across all three equipment types, and how do the field names and structures differ between the current systems?

2. What unique attributes does each equipment category possess that distinguish it from the others, and how should these be preserved in your unified model?

3. How would you design a single collection structure that accommodates all three equipment types while maintaining their distinct characteristics?

4. What field would you use to distinguish between different equipment types, and what values would you assign to each category?

5. How would you standardize the inconsistent field naming (e.g., "rental_price_daily" vs "daily_rate" vs "price_per_day") across all equipment types?

6. What aggregation pipeline stages would you use to transform each equipment type's data into your unified structure, and how would you handle the field mapping and standardization?

7. How would your unified model support common rental system queries like finding all available equipment, filtering by price range, or searching by equipment type?

8. What are the advantages of consolidating these three equipment types into a single collection versus maintaining separate collections for each type?
