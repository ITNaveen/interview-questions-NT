# 🚀 My Role as a DevOps Engineer at Merck/MSD – Managing Healthcare Applications
I work as a DevOps Engineer managing critical healthcare applications at Merck/MSD, one of the world's largest pharmaceutical companies. The applications I manage are used for patient data management, oncology research studies, clinical trials, and drug development tracking. These apps are deployed on AWS and run on Kubernetes (EKS) with a full DevOps CI/CD automation pipeline using Jenkins & ArgoCD.

# first request - 
✔ First request = Credentials → Cognito → JWT Token Issued.
✔ Subsequent requests = JWT Token in headers → Microservices validate → Process request. 

# 📌 Application Overview
The applications I manage fall into three major categories:

1️⃣ Patient Data Management Platform
Stores Electronic Health Records (EHR) securely.
- Used by doctors, researchers, and clinical trial teams to access patient data.
- Features appointment scheduling, medical history tracking, drug prescription management, and billing.
- Must comply with HIPAA and GDPR regulations for security & data protection.
Microservices - 
1. Auth Service → Handles user authentication (OAuth, JWT, SSO).
2. Appointment Service → Manages doctor-patient appointments.
3. Medical Records Service → Stores EHR (Electronic Health Records) securely in AWS RDS (PostgreSQL).
4. Billing Service → Processes medical billing & insurance claims.

3️⃣ Drug Development & FDA Compliance Tracking
- Tracks new drug discoveries, clinical trials, and regulatory approvals.
- Automates submission of reports to FDA & EMA using DevOps pipelines.
- Uses Grafana dashboards for real-time compliance monitoring.
Microservices - 
1. Drug Lifecycle Service → Tracks each phase of drug development.
2. Regulatory Compliance Service → Ensures all FDA & EMA regulations are met.
3. Audit Logging Service → Stores all compliance logs in Elasticsearch (EFK Stack).

# We run Patient Data & Drug Development in one AWS EKS cluster but isolate them using namespaces, IAM policies, network policies, and separate databases/S3 buckets.
# How Are The Two Apps Isolated?
We run both apps in one AWS EKS cluster but ensure strict isolation:

✅ Namespaces for Logical Isolation
patient-data → For patient services.
drug-development → For drug research services.

✅ IAM Policies & Service Accounts
Patient services cannot access drug development databases or resources.
Drug development teams cannot access patient health records.

✅ Separate Databases & S3 Buckets
rds-patient-data (RDS for patient records).
rds-drug-development (RDS for clinical research).

Different S3 buckets for storing sensitive data (E.g., s3://patient-records/ vs. s3://drug-trials/).

✅ Network Policies
Block cross-namespace traffic.
Only authorized services can talk to their respective databases.

🔹 What Should a User Type?
🔹 For Patients & Doctors: https://healthcareapp.com/login
🔹 For Drug Research Teams: https://healthcareapp.com/research

👉 Based on user authentication (AWS Cognito + IAM), they are redirected to the correct application.

📌 DevOps Responsibilities & Infrastructure
I ensure these applications are highly available, secure, and scalable using AWS and Kubernetes.

1️⃣ AWS Infrastructure Management
✔ Kubernetes (EKS) – All apps run on AWS EKS for auto-scaling & high availability.
✔ AWS ALB + Route 53 – Traffic routing with WAF protection against DDoS & SQL injection.
✔ AWS RDS (PostgreSQL) – Used as the primary database (highly available, encrypted).
✔ AWS ElastiCache (Redis) – Used for caching patient data to improve performance.
✔ AWS S3 – Stores large datasets like MRI scans, genomic sequencing data, and medical images.
✔ AWS Secrets Manager – Stores API keys, database credentials, and encryption keys securely.
✔ AWS KMS – Encrypts patient data to meet HIPAA compliance.

2️⃣ CI/CD Pipeline with Jenkins & ArgoCD
✔ Jenkins – Used for build, test, and security scanning of containerized applications.
✔ ArgoCD – Handles GitOps-based deployments to Kubernetes.
✔ Helm Charts – Used for Kubernetes manifest templating.
✔ Blue-Green Deployments – Ensures zero downtime updates for the application.
✔ Automated Rollbacks – If a deployment fails, ArgoCD automatically rolls back to the last stable version.

3️⃣ Monitoring, Logging & Security
✔ Prometheus + Grafana – Monitors Kubernetes cluster health, API response times, and DB performance.
✔ EFK Stack (Elasticsearch, Fluentd, Kibana) – Centralized logging for debugging and auditing.
✔ AWS GuardDuty + AWS WAF – Protects against cyberattacks, DDoS, and unauthorized access.
✔ AWS IAM Policies & Least Privilege Access – Ensures only authorized users can access systems.

4️⃣ Disaster Recovery & High Availability
✔ Multi-Region DR Strategy – Route 53 directs traffic between Primary (Cluster A) and DR (Cluster B).
✔ AWS Karpenter – Auto-scales nodes in DR Cluster B when traffic shifts.
✔ AWS Backup & S3 Versioning – Ensures patient data is always recoverable.

📌 Final Summary
I manage critical healthcare applications that handle patient data, oncology research, and drug development tracking. Using AWS, Kubernetes, Jenkins, ArgoCD, and observability tools (Grafana, EFK), I ensure high availability, security, and compliance. My work directly impacts doctors, researchers, and clinical trial teams who rely on these applications for life-saving medical decisions.

