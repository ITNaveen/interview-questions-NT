
# 🌐 Complete Guide to API Gateway for DevOps (AKS Focused) - Interview & Hands-on Prep

## 📘 What is an API Gateway?

An **API Gateway** is a server that acts as an entry point into a microservices system. It receives requests from clients, routes them to the appropriate microservice, performs authentication and authorization, applies rate limiting and caching, and sends the response back to the client.

### 🔍 Analogy
Think of it as a **bouncer at a nightclub**. It checks who you are (authentication), what you're allowed to access (authorization), makes sure too many people don’t come in at once (rate limiting), and tells you where to go (routing).

---

## 🚀 Why Use an API Gateway?

- **Single Entry Point**: Simplifies access to a complex system of microservices.
- **Security**: Handles authentication and authorization with tools like Keycloak.
- **Routing & Load Balancing**: Sends requests to the correct microservice.
- **Rate Limiting**: Prevents abuse and controls traffic flow.
- **Logging & Monitoring**: Centralized logs and performance metrics.
- **Transformation**: Modify headers, paths, or payloads.
- **Protocol Translation**: Translate between protocols (e.g., HTTP to gRPC).

---

## ⚙️ How Does It Work?
1. Client sends a request to a public endpoint.
2. API Gateway intercepts the request.
3. It checks authentication & authorization.
4. Applies any policies (rate limits, caching).
5. Routes the request to the proper backend microservice.
6. Collects and forwards the response back to the client.

---

## 🧰 Popular API Gateways
- **NGINX Ingress Controller** (Open source, Kubernetes-native)
- **Azure API Management** (Managed PaaS, built into Azure)
- **Istio Gateway** (Service Mesh-focused)
- **Kong**, **Ambassador**, **Traefik** (Cloud-native alternatives)

---

## 🏗️ Setting Up API Gateway in AKS Using NGINX

### ✅ Prerequisites:
- Azure AKS Cluster setup
- `kubectl`, `helm`, `az` CLI installed

### 🛠️ Install NGINX Ingress Controller:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

### 🧭 Sample Ingress Resource:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

### 🔐 Securing with TLS:
```bash
kubectl create secret tls tls-secret \
  --cert=cert.crt \
  --key=cert.key
```

Add to Ingress:
```yaml
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
```

---

## 📈 Monitoring and Security Integration
- **Authentication**: Use Keycloak + OAuth2 Proxy
- **Authorization**: OPA Gatekeeper policies
- **Metrics**: Prometheus + Grafana
- **Tracing**: Instana

---


## 💬 15 Deep Real-World API Gateway Interview Questions & Answers

### 1. **How do you expose a secure internal service using API Gateway in AKS?**
**Answer:** I use NGINX Ingress Controller on AKS, configured to route HTTPS traffic via TLS certificates stored in Kubernetes secrets. The internal service is exposed through a Kubernetes `Service`, and an `Ingress` resource is created with TLS enabled. Authentication is enforced using OAuth2 with Keycloak, by integrating `oauth2-proxy` as a sidecar or middleware. Requests must have a valid JWT token before reaching the backend. I use OPA Gatekeeper to ensure policies like “no HTTP allowed” or “no wildca...

### 2. **Describe a real issue you solved involving Ingress not routing correctly.**
**Answer:** We had an issue where traffic hit the LoadBalancer, but the service wasn't responding. The Ingress path used `pathType: ImplementationSpecific`, which caused mismatch routing. The correct type should have been `Prefix`. Also, the backend service was listening on port 8080 while Ingress was configured for 80. Fixing the path type and aligning ports solved the issue. I also verified the DNS entry pointed to the correct LoadBalancer IP and tested using `curl` and browser dev tools.

### 3. **How do you implement authentication with Keycloak in NGINX Gateway?**
**Answer:** I deploy `oauth2-proxy` to work as an authentication middleware. It integrates with Keycloak via OIDC. The NGINX Ingress is annotated to send unauthenticated users to the proxy. Once a user logs in via Keycloak, `oauth2-proxy` issues a secure cookie or forwards a JWT. The ingress only allows traffic if a valid JWT or auth cookie is present. This offloads auth from backend services and centralizes security.

### 4. **How do you enforce API request limits per user in API Gateway?**
**Answer:** Using NGINX, I apply annotations such as `nginx.ingress.kubernetes.io/limit-rps` (requests per second) and `limit-connections`. These can be customized based on headers, IP, or auth tokens. For per-user limits, the token (from Keycloak) is parsed via Lua scripting or a custom plugin. In Azure APIM, I define policies based on subscriptions or identity claims, setting quota and rate limits with XML-based policy definitions.

### 5. **How do you manage multiple environments (dev/stage/prod) with one API Gateway?**
**Answer:** I separate environments at the namespace level. Each environment has its own Ingress controller with distinct IngressClass names. Domains like `dev.api.example.com`, `stage.api.example.com` point to the respective Ingress. I use Helm to templatize the configurations and Azure DevOps to deploy per environment using different variable groups. This ensures isolation while maintaining similar configs.

### 6. **How did you troubleshoot a broken ingress endpoint?**
**Answer:** I used `kubectl describe ingress` to check for misconfigured paths or missing services. I checked logs of the NGINX controller using `kubectl logs`. Then, I used `nslookup` and `curl` from inside and outside the cluster. Found that the DNS was correct, but the service selector labels didn’t match the pod labels, so traffic wasn’t reaching pods. Fixing the selector solved the issue.

### 7. **How do you use OPA Gatekeeper with API Gateway?**
**Answer:** I deploy Gatekeeper in the cluster and define `ConstraintTemplates` that define rules like "ingress must have TLS", "no wildcard domains", etc. These constraints are applied at the admission controller level. When someone tries to deploy a misconfigured Ingress, it is denied with an error. This helps enforce compliance automatically.

### 8. **What’s your process to onboard a new microservice to the API Gateway?**
**Answer:** First, I deploy the microservice and expose it via a Kubernetes `Service`. Then I create an `Ingress` rule that maps a path or subdomain to that service. I configure Keycloak to add a new client for the service and integrate `oauth2-proxy` if needed. Finally, I add DNS records and use Helm to deploy the Ingress rule through our CI/CD pipeline.

### 9. **How do you integrate Azure DevOps with Ingress updates?**
**Answer:** I use a Helm chart to template Ingress resources. Azure DevOps pipelines render these templates using environment-specific variables. The Kubernetes service connection in Azure DevOps is used to apply the configuration using `kubectl apply`. I also add steps to validate syntax with `kubeval` and policy compliance using `conftest` before deployment.

### 10. **Explain how you handled CORS issues in production.**
**Answer:** We had a frontend hosted on a different domain, and the API was rejecting requests due to CORS. I updated the Ingress annotations to enable CORS using `nginx.ingress.kubernetes.io/enable-cors: "true"` and set headers for `Access-Control-Allow-Origin`, `Methods`, and `Headers`. I also ensured the backend handled preflight OPTIONS requests correctly. Validated with browser dev tools.

### 11. **How do you handle rolling out API versioning?**
**Answer:** I use path-based versioning such as `/v1`, `/v2`, where each version points to a different service deployment. In some cases, we use header-based routing using `nginx.ingress.kubernetes.io/configuration-snippet` for advanced matching. Azure APIM supports header-based versioning natively. For gradual rollout, I use canary deployments to test new versions with limited traffic.

### 12. **How do you monitor API usage and performance?**
**Answer:** NGINX Ingress exposes Prometheus metrics, which I scrape and visualize in Grafana. I monitor metrics like request rate, error rates, latency, and status codes. I also use Loki for log aggregation. In Azure, API Management provides built-in analytics dashboards that track usage per API, per user, and latency trends.

### 13. **What’s your approach to zero-downtime deployments with API Gateway?**
**Answer:** I ensure deployments have readiness and liveness probes. Kubernetes won’t send traffic until the pod is healthy. The Ingress controller respects this and avoids routing traffic to non-ready pods. I use rolling updates or canary deployments to shift traffic gradually. If something goes wrong, rollback is immediate via Helm or pipeline scripts.

### 14. **How do you handle TLS certificates rotation in production?**
**Answer:** I use `cert-manager` to manage TLS certificates automatically with Let’s Encrypt. Certificates are renewed before expiry and updated in Kubernetes secrets. When a secret changes, the Ingress controller detects it and reloads automatically. This ensures zero downtime during cert renewal.

### 15. **How do you test an API Gateway setup before going live?**
**Answer:** I start by validating Ingress rules using `kubectl describe`. Then I test routing with `curl`, Postman, and browser. I simulate user traffic using tools like `k6`. I also check logs and metrics to ensure no errors are surfacing. In staging, I verify OAuth flows with Keycloak and test rate limits using automated scripts.

---


### 1. **How do you expose a secure internal service using API Gateway in AKS?**
**Answer:** (See previous detailed version — retained for brevity.)

### 2. **Describe a real issue you solved involving Ingress not routing correctly.**
**Answer:** (See previous detailed version.)

### 3. **How do you implement authentication with Keycloak in NGINX Gateway?**
**Answer:** (See previous detailed version.)

### 4. **How do you enforce API request limits per user in API Gateway?**
**Answer:** (See previous detailed version.)

### 5. **How do you manage multiple environments (dev/stage/prod) with one API Gateway?**
**Answer:** (See previous detailed version.)

### 6. **How did you troubleshoot a broken ingress endpoint?**
**Answer:** (See previous detailed version.)

### 7. **How do you use OPA Gatekeeper with API Gateway?**
**Answer:** (See previous detailed version.)

### 8. **What’s your process to onboard a new microservice to the API Gateway?**
**Answer:** (See previous detailed version.)

### 9. **How do you integrate Azure DevOps with Ingress updates?**
**Answer:** (See previous detailed version.)

### 10. **Explain how you handled CORS issues in production.**
**Answer:** (See previous detailed version.)

### 11. **How do you handle rolling out API versioning?**
**Answer:** (See previous detailed version.)

### 12. **How do you monitor API usage and performance?**
**Answer:** (See previous detailed version.)

### 13. **What’s your approach to zero-downtime deployments with API Gateway?**
**Answer:** (See previous detailed version.)

### 14. **How do you handle TLS certificates rotation in production?**
**Answer:** (See previous detailed version.)

### 15. **How do you test an API Gateway setup before going live?**
**Answer:** (See previous detailed version.)

---

## 🧠 Final Interview Tips
- Practice explaining authentication with Keycloak.
- Highlight security enforcement via Gatekeeper.
- Emphasize observability (Grafana, Instana).
- Show YAML + Helm confidence.
- Don’t just say “I know it”—walk them through real debugging.

---

Need a CI/CD pipeline example or diagram added? Just say the word! 🚀


# FLOW - 
```yml

Your Answer (Refined):
Client Request:

The client types your application's URL in the browser, which sends a request to Route 53.

Route 53 Health Check:

Route 53 performs a health check and sees that Cluster A is healthy. Since Cluster A is healthy, it routes the request to Cluster A’s endpoint.

It's important to clarify here: Route 53 is responsible for directing the traffic to the right endpoint (which could be Cluster A or Cluster B, depending on health). At this point, the request is on its way to the ALB in Cluster A.

API Gateway Intercepts the Request:

The request first hits the API Gateway before reaching the ALB.

The API Gateway is in place to handle several important tasks:

Authentication: It checks if the incoming request contains a valid JWT token. If not, it redirects the user to Keycloak for authentication.

Authorization: After the user is authenticated, API Gateway checks the user's permissions to ensure they can access the requested resources.

Keycloak Authentication:

If this is the first request or the request doesn't have a valid JWT token, the API Gateway redirects the user to Keycloak for authentication.

Keycloak validates the user's credentials and issues a JWT token upon successful authentication.

API Gateway Stores the JWT Token:

The API Gateway now stores the JWT token issued by Keycloak.

This token is essentially a "session token" for this user/request. For any subsequent requests from the same client, the API Gateway will check if the request includes a valid token.

Forwarding Request with JWT Token:

Once authenticated, the API Gateway forwards the request along to Cluster A's ALB with the JWT token included in the request headers. This allows the ALB to route the request to the Ingress Controller for further processing.

Request to the Ingress Controller:

The request then passes from the ALB to the Ingress Controller within Cluster A.

The Ingress Controller forwards the request to the appropriate backend service running inside the Kubernetes cluster.

Subsequent Requests:

For any subsequent requests from the same client, the API Gateway will already have the JWT token stored. It will automatically attach this token to the request and forward it to the ALB.

The ALB and Ingress Controller will continue processing the requests based on the token, ensuring the user is authenticated for each interaction.

Key Points to Emphasize:
API Gateway is the security and traffic management layer that sits in front of your ALB and handles all the authentication and authorization tasks.

Keycloak handles user authentication and provides JWT tokens that are then passed by the API Gateway to ensure that each subsequent request is properly authenticated.

The JWT token is not just for the first request, but is carried along with every request from that client, so the API Gateway doesn't need to re-authenticate the user every time.

The API Gateway maintains the user's session through the JWT token and ensures that only authenticated requests are forwarded to the appropriate backend services.
```