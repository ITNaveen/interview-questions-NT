# Step-by-Step Request Flow - 

# 🔷 Step 1: User types healthcareapp.com
A user (doctor, patient, or admin) opens a web browser or a mobile app and enters https://healthcareapp.com.
The browser needs to resolve the domain name (healthcareapp.com) to an IP address to send the request.


# 🔷 Step 2: Route 53 Resolves DNS & Performs Health Checks
.........................................
How Does DNS Resolution Work?
1️⃣ The browser checks its cache or the system’s DNS resolver (set by the OS or network).

2️⃣ If not cached, it sends a query to the root DNS servers, which then direct it to the .com TLD (Top-Level Domain) servers.
- The browser first checks its own cache - if healthcareapp.com was visited recently, the browser already knows the IP and connects directly (skipping DNS lookup).
If not found, it moves to the next step.
- When a browser cannot find a cached DNS record, it asks a Recursive DNS Resolver (set by netwrok - like Google DNS or Cloudflare). If the resolver does not have the IP cached either, it begins the DNS resolution process.
- Next Step:
✅ The Recursive DNS Resolver sends a query to a Root DNS Server. there are 13 sets of Root DNS Servers, but each set consists of multiple physical servers distributed worldwide for redundancy and performance.
✅ The Root DNS Server is the top-most level in the DNS hierarchy.
- How Root DNS Directs Requests to TLDs:
If the domain is example.com, the Root DNS says:
"Ask the .com TLD Name Server."
If the domain is university.edu, it says:
"Ask the .edu TLD Name Server."
If the domain is techstartup.io, it says:
"Ask the .io TLD Name Server."
If the domain is mywebsite.net, it says:
"Ask the .net TLD Name Server."

3️⃣ The TLD servers check where the DNS is hosted (AWS Route 53, GoDaddy, Cloudflare, etc.) and return the Route 53 name server details.
What the TLD Name Server Does
✅ It checks its records and finds out which Authoritative DNS Name Server is responsible for healthcareapp.com.
✅ Since healthcareapp.com is hosted on AWS Route 53, the .com TLD replies:
📢 "I don’t have the IP, but ask AWS Route 53 at ns-123.awsdns.com."
✅ The Recursive DNS Resolver now knows where to go next → AWS Route 53.

4️⃣ The browser queries Route 53’s authoritative name servers, which provide the IP address of the Application Load Balancer (ALB).

# shorter flow - 
📌 Step-by-Step DNS Resolution Flow
1️⃣ User types healthcareapp.com in the browser
The browser first checks its local cache for an IP address.
If not found, it queries the Recursive DNS Resolver (e.g., Google DNS 8.8.8.8, Cloudflare 1.1.1.1).

2️⃣ Recursive DNS Resolver is contacted
It checks its cache for the IP.
If not found, it sends a request to a Root DNS Server.

3️⃣ Root DNS Server (one of 13 sets worldwide) is contacted
The Root Server does not have the IP, but it sees that this is a .com domain.
It directs the resolver to the .com TLD Name Server.

4️⃣ TLD Name Server for .com is contacted
It does not have the IP but knows which Authoritative DNS Server is responsible.
Since healthcareapp.com is managed by AWS Route 53, it tells the resolver:
"Ask AWS Route 53 at ns-123.awsdns.com."

5️⃣ Recursive DNS Resolver queries AWS Route 53 (Authoritative DNS Server)
Route 53 has the exact IP address of healthcareapp.com.
It returns the IP (e.g., 34.201.10.50) to the resolver.

6️⃣ Recursive DNS Resolver sends the IP back to the browser
The browser connects to 34.201.10.50 (which is the Load Balancer in AWS).
The request is forwarded to the web server (or Kubernetes cluster) hosting the app.
.........................................................................

How Does Route 53 Perform Health Checks?
Route 53 continuously pings the ALB of Cluster A using an HTTP(S) health check on a specific endpoint (e.g., /healthz).

If Cluster A is healthy, it returns a 200 OK response, and traffic continues to ALB A.
If Cluster A is unhealthy, Route 53 automatically switches traffic to Cluster B’s ALB.

🔹 Why Not Use Godaddy DNS Instead of Route 53?
✔️ Route 53 supports health checks & automatic failover to Cluster B.
✔️ Tightly integrates with AWS services like ALB, S3, CloudFront, etc.
✔️ Latency-based routing & geo-routing options optimize performance.

# 🔷 Step 3: Route 53 Routes Traffic to ALB of Cluster A - 
Route 53 returns the IP of Cluster A’s ALB to the browser, which then sends the request directly to the ALB.

##### ALB ingress and ALB - 
1. install alb ingress controller as deployment in kube-system ns - 
```yml 
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<your-cluster-name>
```
✔ kube-system is reserved for cluster-wide critical components (like CoreDNS, kube-proxy, and the AWS ALB Ingress Controller).
✔ Since the ALB Ingress Controller monitors and manages ALBs for the entire cluster, it makes sense to run it in kube-system where other cluster-wide controllers run.

2. define ingress object (ingress rule).
```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing   # ALB is public-facing
    alb.ingress.kubernetes.io/target-type: ip  
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]' # ALB listens on HTTP/HTTPS
spec:
  ingressClassName: alb    # Uses AWS ALB Ingress Controller
  rules:
  - host: healthcareapp.com  # Traffic matching this hostname
    http:
      paths:
      - path: /               # Any request to "/"
        pathType: Prefix      # Matches any path that starts with "/"
        backend:
          service:
            name: my-service  # Traffic is forwarded to this Kubernetes Service
            port:
              number: 80      # Service must be listening on port 80

```
3.  As soon as you apply the Ingress resource, the ALB Ingress Controller detects it and:
Automatically provisions an AWS ALB.
Creates Target Groups to route traffic to Kubernetes services.
Handles path-based or host-based routing.

Why Do We Use an ALB Instead of an NLB?
✅ ALB is a Layer 7 Load Balancer – It understands HTTP/HTTPS traffic and allows path-based routing (/auth to auth service, /api to backend).
✅ ALB can perform SSL termination – This offloads encryption processing from backend services.
✅ ALB supports AWS WAF for security filtering.

# 🔷 Step 4: AWS WAF Inspects Traffic for Security Threats - 
What is SQL Injection?
SQL Injection (SQLi) is an attack where hackers inject malicious SQL queries into a web application’s input fields.
Example Attack:
SELECT * FROM users WHERE username = 'admin' OR 1=1 --';
If an app doesn't sanitize inputs, this would return all users instead of just "admin".
WAF mitigates this by blocking SQL-like patterns in request bodies and headers.

What is a DDoS Attack?
Distributed Denial of Service (DDoS) is when an attacker floods the server with fake requests to overwhelm it.

AWS WAF blocks requests based on rate limits, IP reputation lists, and anomaly detection.

# 🔷 Step 5: ALB Performs SSL Termination & Sends Traffic to ALB Ingress Controller- 
What is SSL Termination & Why Do It at ALB?
SSL Termination means decrypting HTTPS traffic at the ALB level, then sending unencrypted traffic inside the cluster.
This is done because:
✅ Reduces CPU load on Kubernetes worker nodes (encryption/decryption is computationally expensive).
✅ ALB manages SSL certificates via AWS ACM (AWS Certificate Manager).

How Does ALB Know Where to Send Traffic?
ALB is attached to an AWS ALB Ingress Controller running in Kubernetes.
ALB routes traffic to the NodePort service on worker nodes where the Ingress Controller is running.

# 🔷 Step 6: ALB Ingress Routes Requests to the Correct NodePort Service - 
🔹 If target-type: ip is used:
1️⃣ When you install the ALB Ingress Controller, it watches for Ingress objects and automatically registers the pods running the Ingress Controller as ALB targets.
2️⃣ ALB then creates a Target Group in AWS, which contains only the pods’ IPs (not worker nodes).

🔥 Key Takeaways
✅ ALB knows where to send traffic because it registers Ingress Controller pods as Target Group members.
✅ Traffic goes directly to Ingress Controller pods, NOT to worker nodes.
✅ No extra hop is needed (faster and more efficient than target-type: instance).

# 🔷 Step 7: Application Processes the Request - 
The backend application queries the database (AWS RDS PostgreSQL) or caches results in AWS ElastiCache (Redis) for fast performance.

Authentication requests go to Cognito/Auth service.
Secure credentials are retrieved from AWS Secrets Manager.

# Step 8 & 9: Response Sent Back to the User
The application generates a response and sends it back through:

Pod → Kubernetes Service → Ingress Controller → ALB → Route 53 → User’s Browser

The user sees their appointment list, patient record, or any requested data.

🔹 Full Traffic Flow Summary (End-to-End)
✅ 1️⃣ User enters healthcareapp.com.
✅ 2️⃣ Route 53 resolves DNS & checks health of clusters.
✅ 3️⃣ Route 53 sends traffic to ALB of Cluster A.
✅ 4️⃣ ALB sends traffic to AWS WAF, which blocks threats like SQL Injection & DDoS.
✅ 5️⃣ ALB performs SSL termination & forwards traffic to Kubernetes via ALB Ingress Controller.
✅ 6️⃣ ALB Ingress routes traffic to a NodePort service on worker nodes.
✅ 7️⃣ NodePort forwards traffic to the correct backend service.
✅ 8️⃣ Application processes the request (DB queries, authentication).
✅ 9️⃣ Response is sent back through the same path.
✅ 🔟 User sees the requested data in their browser/app.

