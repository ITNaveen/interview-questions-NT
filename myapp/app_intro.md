🚀 My Role as a DevOps Engineer at Merck/MSD – Managing Healthcare Applications
I work as a DevOps Engineer managing critical healthcare applications at Merck/MSD, one of the world's largest pharmaceutical companies. The applications I manage are used for patient data management, oncology research studies, clinical trials, and drug development tracking. These apps are deployed on AWS and run on Kubernetes (EKS) with a full DevOps CI/CD automation pipeline using Jenkins & ArgoCD.

📌 Application Overview
The applications I manage fall into three major categories:

1️⃣ Patient Data Management Platform
Stores Electronic Health Records (EHR) securely.

Used by doctors, researchers, and clinical trial teams to access patient data.

Features appointment scheduling, medical history tracking, drug prescription management, and billing.

Must comply with HIPAA and GDPR regulations for security & data protection.

2️⃣ Oncology Research & Clinical Trial Management
Used by oncologists and clinical researchers to track patient responses to drugs.

Stores real-time genomic sequencing data and tumor progression reports.

Integrated with AI/ML models to predict patient outcomes.

Needs 99.99% uptime to ensure researchers always have access to data.

3️⃣ Drug Development & FDA Compliance Tracking
Tracks new drug discoveries, clinical trials, and regulatory approvals.

Automates submission of reports to FDA & EMA using DevOps pipelines.

Uses Grafana dashboards for real-time compliance monitoring.

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

