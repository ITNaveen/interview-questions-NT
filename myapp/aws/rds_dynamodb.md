# What is RDS?
Amazon RDS (Relational Database Service) is a managed relational database service that automates database setup, patching, backups, and scaling.

# Understanding Relational Databases (RDS Basics)
- RDS stores data in tables (just like an Excel sheet but much more powerful).
- Each table has columns & rows (like a structured database).
- Data relationships are defined using primary & foreign keys.

# advantages of RDS - 
ACID stands for:
🔹 Atomicity → All-or-Nothing Transactions
If a patient books an appointment, the entire transaction must succeed or fail.
Example: If a payment is processed but appointment confirmation fails, the payment is rolled back.

🔹 Consistency → Data must always be valid.
If a drug dosage is updated in the system, it should always have valid values (e.g., 500mg, not "abc").

🔹 Isolation → Multiple transactions should not interfere.
If two users try to book the last available appointment at the same time, only one succeeds, preventing double booking.

🔹 Durability → Data must not be lost, even in a failure.
If a power failure occurs, data remains safe and recoverable from RDS backups.

Why We Use RDS in Our App?
✅ Best for patient records, billing, prescriptions, and clinical trials (structured & relational data).
✅ Supports complex SQL queries, transactions, and joins.
✅ Ensures data consistency and security (HIPAA compliance).

#### ###### ####### ###### ######## #########
### ##### ####### ####### ####### ##########

# What is DynamoDB?
DynamoDB is a fully managed NoSQL database designed for high-speed, low-latency queries at scale. DynamoDB is great for storing fast-changing, short-lived, or frequently accessed data.
1. When a user logs in, Cognito generates a JWT token → we store this in DynamoDB with a TTL of 1 hour.
2. Every API request checks the token in DynamoDB.
3. If the token is expired, the user is logged out automatically.

This saves RDS from handling unnecessary authentication queries.

Understanding NoSQL Databases (DynamoDB Basics)
- Unlike RDS, DynamoDB does not use tables with fixed schemas (it’s flexible).
- Instead, it uses key-value pairs and document-based storage.
- It does not require complex joins and scales automatically.

Why is DynamoDB So Fast?
🔹 Key-Value Access → Instead of scanning tables like RDS, DynamoDB instantly finds records using primary keys.
🔹 Auto-Scaling → Handles millions of requests per second, ideal for high-traffic apps.
🔹 NoSQL Flexibility → No need for predefined schemas or relationships.

Why We Use DynamoDB in Our App?
✅ Best for real-time session management, user authentication, and activity logs.
✅ Super-fast lookups for appointment availability, notifications, and caching.
✅ Scales automatically, perfect for high-traffic mobile apps.

# answer - 
We use RDS for structured, transactional data like patient records and billing because it ensures ACID compliance. But for high-speed, short-lived, or frequently accessed data (like JWT tokens, session data, and real-time lookups), DynamoDB is the better choice. This hybrid approach gives us both reliability and speed, ensuring our system scales efficiently while maintaining data integrity.