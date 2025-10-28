# Lab 4: DynamoDB Table Design with Local Secondary Indexes

## Objective
Design and implement a DynamoDB table with Local Secondary Indexes (LSIs) to support multiple query patterns for an e-commerce order tracking system. Practice table creation, data insertion, and query operations using Python and boto3.

## Prerequisites
- Python 3.x installed
- AWS Learner Lab access
- Basic understanding of DynamoDB concepts (partition key, sort key, LSI)

## Setup Instructions

### 1. Install boto3
```bash
pip install boto3
```

### 2. Configure AWS Credentials
**IMPORTANT**: Update your AWS credentials file with your Learner Lab credentials:

1. Get your credentials from AWS Learner Lab:
   - Click "AWS Details" in your Learner Lab
   - Copy the AWS CLI credentials

2. Edit your credentials file:
   - **Windows**: `C:\Users\[username]\.aws\credentials`
   - **Mac/Linux**: `~/.aws/credentials`

3. Add your Learner Lab credentials:
```ini
[default]
aws_access_key_id = YOUR_ACCESS_KEY_HERE
aws_secret_access_key = YOUR_SECRET_KEY_HERE
aws_session_token = YOUR_SESSION_TOKEN_HERE
region = us-east-1
```

## Use Case: E-commerce Order Tracking

### Application Requirements
Your e-commerce platform needs to support these query patterns:

1. **Find all orders for a customer** (primary access pattern)
2. **Find customer orders sorted by order date** (newest first)
3. **Find customer orders sorted by total amount** (highest first)
4. **Get specific order details** by customer and order ID

### Business Context
- Customers place multiple orders over time
- Customer service needs to view orders by date or amount
- The app needs fast lookups for individual order details
- All queries are scoped to a specific customer (no cross-customer queries needed)

## DynamoDB Concepts Review

### Key Concepts
- **Partition Key**: Distributes data across multiple partitions
- **Sort Key**: Orders items within a partition
- **Local Secondary Index (LSI)**: Alternative sort key for same partition key
- **Item**: Individual record in the table
- **Attribute**: Field within an item

### LSI Limitations
- Must share the same partition key as the base table
- Maximum 10 LSIs per table
- Must be created at table creation time
- All attributes are projected by default

## Tasks

### Task 1: Table Creation (25 points)
**Requirement**: Write Python code to create a DynamoDB table that supports all four query patterns using LSIs.

**Deliverable**: Complete the `create_table()` function:

```python
import boto3
from decimal import Decimal

def create_table():
    dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
    
    # Your table creation code here
    # Include table name, key schema, attribute definitions, and LSIs
    
    return table
```

### Task 2: Data Insertion (15 points)
**Requirement**: Write Python code to insert the following order data.

**Sample Data**:
```
Customer: "CUST001"
- OrderID: "ORD-2024-001", OrderDate: "2024-01-15", Status: "DELIVERED", TotalAmount: 299.99, Items: 3
- OrderID: "ORD-2024-045", OrderDate: "2024-03-22", Status: "SHIPPED", TotalAmount: 149.50, Items: 2
- OrderID: "ORD-2024-078", OrderDate: "2024-05-10", Status: "DELIVERED", TotalAmount: 599.00, Items: 1

Customer: "CUST002"
- OrderID: "ORD-2024-012", OrderDate: "2024-02-08", Status: "DELIVERED", TotalAmount: 89.99, Items: 4
- OrderID: "ORD-2024-056", OrderDate: "2024-04-18", Status: "PROCESSING", TotalAmount: 449.75, Items: 2
- OrderID: "ORD-2024-089", OrderDate: "2024-06-05", Status: "SHIPPED", TotalAmount: 199.25, Items: 3
```

**Deliverable**: Complete the `insert_data()` function:

```python
def insert_data(table):
    # Your data insertion code here
    # Use individual table.put_item() calls for each order
    pass
```

### Task 3: Query Operations (20 points)
**Requirement**: Write Python functions for each access pattern:

**Query 1** (5 points): Get all orders for a specific customer
```python
def get_customer_orders(table, customer_id):
    # Your query code here
    pass
```

**Query 2** (5 points): Get customer orders sorted by order date (newest first)
```python
def get_orders_by_date(table, customer_id):
    # Your query code here using LSI
    pass
```

**Query 3** (5 points): Get customer orders sorted by total amount (highest first)
```python
def get_orders_by_amount(table, customer_id):
    # Your query code here using LSI
    pass
```

**Query 4** (5 points): Get specific order details
```python
def get_specific_order(table, customer_id, order_id):
    # Your query code here
    pass
```

## Deliverables

Submit **only**:
- **CSCI112-[StudentID1]-[LastName1]-[StudentID2]-[LastName2]-DynamoDB.zip** The zip file must only contain the requirements.txt and an optional readme. **DO NOT include dependencies in the submission**

**File format example**:
```python
"""
Certificate of Authorship:
I have not discussed the Python language code in my program with anyone 
other than my instructor or the teaching assistants assigned to this course.
I have not used Python language code obtained from another student, 
or any other unauthorized source, either modified or unmodified.
If any Python language code or documentation used in my program 
was obtained from another source, such as a textbook or course notes, 
that has been clearly noted with a proper citation in the comments of my program.
"""

# Lab 4 Solution - DynamoDB Table Design with LSIs
# Students: [Student1 Name], [Student2 Name]

import boto3
from decimal import Decimal

def create_table():
    """Task 1: Create DynamoDB table with LSIs (25 points)"""
    # Your implementation here
    pass

def insert_data(table):
    """Task 2: Insert sample order data (15 points)"""
    # Your implementation here
    pass

def get_customer_orders(table, customer_id):
    """Task 3.1: Get all orders for customer (5 points)"""
    # Your implementation here
    pass

def get_orders_by_date(table, customer_id):
    """Task 3.2: Get orders sorted by date (5 points)"""
    # Your implementation here
    pass

def get_orders_by_amount(table, customer_id):
    """Task 3.3: Get orders sorted by amount (5 points)"""
    # Your implementation here
    pass

def get_specific_order(table, customer_id, order_id):
    """Task 3.4: Get specific order (5 points)"""
    # Your implementation here
    pass

def main():
    """Main function to test all operations"""
    # Create table
    table = create_table()
    
    # Insert data
    insert_data(table)
    
    # Test queries
    print("All CUST001 orders:")
    print(get_customer_orders(table, "CUST001"))
    
    print("\nCUST001 orders by date:")
    print(get_orders_by_date(table, "CUST001"))
    
    print("\nCUST002 orders by amount:")
    print(get_orders_by_amount(table, "CUST002"))
    
    print("\nSpecific order:")
    print(get_specific_order(table, "CUST002", "ORD-2024-056"))

if __name__ == "__main__":
    main()
```

## Grading

**Total: 60 points**

- Task 1: Table creation with appropriate LSIs (25 points)
- Task 2: Data insertion functions (15 points)
- Task 3: Query operations for all access patterns (20 points)

## Hints

- Use `Decimal` from decimal module for monetary amounts
- Consider what makes a good partition key for this use case
- Think about which attributes need to be sort keys for different access patterns
- LSI names should be descriptive of their purpose
- Use ISO 8601 date format (YYYY-MM-DD) for consistent sorting
- Remember that DynamoDB is case-sensitive for attribute names
- Use `ScanIndexForward=False` for descending order

## Notes
- Test your code before submission
- Ensure your table design supports all required query patterns
- LSIs cannot be added after table creation
- Remember to update your AWS credentials from Learner Lab before running
