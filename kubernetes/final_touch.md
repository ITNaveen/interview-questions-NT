# DR - 
✅ Explanation of Active-Passive EKS with Route 53 & Karpenter
"My architecture is based on an Active-Passive failover strategy using Route 53, ALBs, EKS Managed Node Groups, and Karpenter:

1️⃣ Client requests hit Route 53 (my domain is hosted here).
2️⃣ Route 53 checks health and decides which ALB to send traffic to:

✅ If ALB1 (Active) is healthy, Route 53 sends traffic to Cluster 1 (Active).
❌ If ALB1 is unhealthy, Route 53 redirects traffic to ALB2 (Passive).
3️⃣ ALB1 (Active Cluster) → Runs with 12 manually defined worker nodes (EKS Managed Node Group) for stability.
4️⃣ ALB2 (Passive Cluster) → Initially has no running nodes but is set up with Karpenter to dynamically scale when needed.
5️⃣ If traffic switches to ALB2 (Passive Cluster), Karpenter automatically provisions worker nodes to handle workloads.
6️⃣ Once Cluster 1 (Active) recovers, Route 53 redirects traffic back to ALB1, and Karpenter scales Cluster 2 back down to reduce cost."
🚀 Why Is This a Good Design?
✅ Ensures high availability – Route 53 automatically shifts traffic during failure.
✅ Optimized cost efficiency – Passive cluster runs only when needed (no pre-allocated nodes).
✅ Fast recovery – Karpenter rapidly provisions nodes when failover occurs.
✅ Stability in the active cluster – Manually defined EKS Managed Node Group ensures predictable performance.


# KEDA - 
KEDA (Kubernetes Event-Driven Autoscaling) – Scaling Based on Real-World Workloads
KEDA enables event-driven autoscaling in Kubernetes by reacting to external workloads instead of just CPU/memory. Below are key event sources KEDA can scale on, explained in detail:

✅ HTTP Request Rate (Prometheus Metrics)
KEDA can scale pods based on incoming HTTP requests per second.
It integrates with Prometheus to fetch request rate metrics (e.g., from an Ingress controller or API Gateway).
If the number of requests per second exceeds a threshold, KEDA triggers scaling to handle the increased load.
Example: If an API service starts receiving more than 50 requests per second, new pods are created to distribute traffic.

✅ Database Queries (PostgreSQL, MySQL)
KEDA can track database workloads and autoscale pods based on query volume.
It integrates with databases like PostgreSQL, MySQL, and MongoDB to monitor active connections or pending queries.
If database connections exceed a limit (e.g., more than 100 active queries), KEDA triggers more replicas to handle traffic.
Example: A read-heavy application might scale up additional read-replica pods if query load increases beyond 100 queries per second.

✅ Cloud Services Metrics (Azure Monitor, AWS CloudWatch)
KEDA can fetch cloud-native metrics from AWS CloudWatch, Azure Monitor, and Google Cloud Monitoring.
Based on custom-defined thresholds, KEDA dynamically scales workloads.
Example: If an AWS Lambda function starts experiencing high execution latency, KEDA can increase pods in a Kubernetes cluster to reduce processing delays.

This ensures that Kubernetes workloads react dynamically to cloud-based metrics.
🚀 Interview-Ready Answer:
"KEDA is a Kubernetes event-driven autoscaler that scales workloads based on real-world external events. Instead of relying only on CPU/memory like HPA, KEDA reacts to HTTP request rates, message queue depths, database queries, and cloud service metrics. For example, I use KEDA with Prometheus to autoscale my API based on HTTP traffic and with AWS SQS to scale worker pods based on pending messages. This ensures cost efficiency by scaling only when needed and even allowing pods to scale down to zero when there’s no workload." 🚀

# REDIS - 
🔥 AWS ElastiCache (Redis) is primarily used for:

1️⃣ Database Query Caching – Instead of hitting the database repeatedly, frequently accessed queries are stored in Redis for ultra-fast retrieval.
2️⃣ Session Storage for Authentication – User sessions (tokens, login states) are stored in Redis, so authentication doesn't need to query the database every time. If Redis recognizes the session, the request is validated instantly.
3️⃣ Scaling for High Traffic – When demand increases, ElastiCache can scale by adding read replicas, distributing the load across multiple nodes while maintaining low latency.
4️⃣ Reducing Latency in EKS Microservices – Microservices can fetch data from Redis much faster than querying an RDS (or any other DB), leading to faster API responses and better application performance.

✅ So, Redis eliminates unnecessary DB queries, speeds up authentication, and auto-scales when traffic spikes! 🚀

🔥 Real-World Example: How I Use AWS ElastiCache in My Infra
"In my EKS-based architecture, I use AWS ElastiCache for Redis to optimize database performance. My backend services use PostgreSQL (RDS) as the primary relational database. However, for frequently accessed data like user profiles, product catalogs, and session tokens, I cache responses in Redis. This significantly reduces the load on RDS, improves API response times, and ensures a smooth user experience."

🔹 Scenario: Optimizing Authentication & API Response Times

When a user logs in, the authentication service first checks Redis for an active session.
If found, the user is instantly authenticated without querying the database.
If not, the system fetches the credentials from PostgreSQL, validates them, and stores the session in Redis for future requests.
🔹 Scenario: High-Traffic E-Commerce Search API

My app has a product search feature that queries the PostgreSQL database.
Instead of running expensive queries every time, Redis caches the most searched products.
When traffic spikes, Redis read replicas scale automatically, ensuring consistent low-latency responses.

# CSI - 
CSI (Container Storage Interface) driver acts as the middle link between your pod and the EBS volume when handling PersistentVolumeClaims (PVCs) in Kubernetes.

🔹 How CSI Enables EBS for Persistent Storage in EKS
1️⃣ Pod Requests Storage → The pod makes a request for a PersistentVolumeClaim (PVC).
2️⃣ Kubernetes Calls the CSI Driver → The EBS CSI driver provisions an EBS volume dynamically (or attaches an existing one).
3️⃣ EBS Volume is Mounted to the Pod → The pod gets persistent storage, even if it's rescheduled.

🔹 Why CSI is Important for EBS in EKS?
✅ Dynamic Provisioning → Creates and attaches EBS volumes on-demand based on PVC requests.
✅ Persistent Storage → Even if a pod crashes or moves to another node, EBS remains and reattaches.
✅ Multi-AZ Support → Works with AWS EBS gp3, io1, io2 for high performance.
✅ Storage Management → Kubernetes can automatically resize, snapshot, and delete volumes using CSI.

🔹 Real-World EKS Example
🔥 "In my EKS cluster, I use the AWS EBS CSI driver for my PostgreSQL database. When my database pod requests a 50GB storage volume, Kubernetes automatically provisions an EBS volume and mounts it to my pod. If my pod moves to another node, the volume is detached and reattached without data loss."

port - 5432