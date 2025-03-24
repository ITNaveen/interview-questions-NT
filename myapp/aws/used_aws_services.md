# Here are the AWS services used in your healthcare app:

1. Networking & Traffic Management
Route 53 → DNS resolution.
Application Load Balancer (ALB) → Distributes traffic to EKS.
AWS Load Balancer Controller (Ingress Controller) → Manages ingress routing in EKS.

2. Compute & Container Orchestration
Amazon EKS → Kubernetes cluster management.
EC2 (Worker Nodes) → Runs the Kubernetes workloads.

3. Authentication & Security
AWS Cognito → User authentication & JWT management.
IAM (Identity and Access Management) → Access control for AWS resources.
AWS WAF → Web firewall protection.
AWS Secrets Manager → Stores database credentials securely.

4. Data Storage & Databases
Amazon RDS (PostgreSQL/MySQL) → Structured database for patient & drug data.
Amazon DynamoDB → NoSQL database for fast lookups.
Amazon S3 → Stores medical records, reports, images.
Amazon EFS → Shared file storage for workloads.

5. APIs & Messaging
Amazon API Gateway → Manages API requests (used for mobile apps).
Amazon SNS → Sends notifications (SMS, email alerts).
Amazon SQS → Message queue for asynchronous tasks.
Amazon EventBridge → Event-driven integration between services.

6. Monitoring & Logging
Amazon CloudWatch → Logs, metrics, alerts.
AWS X-Ray → Traces API requests end-to-end.
AWS OpenSearch (Elasticsearch) → Centralized logging & search.

7. Security & Compliance
AWS KMS → Encrypts sensitive data.
AWS Audit Manager → Compliance tracking (HIPAA, GDPR).
AWS Config → Monitors AWS resource configurations.