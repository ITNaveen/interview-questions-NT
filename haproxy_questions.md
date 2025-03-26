Basic Questions

What is HAProxy?HAProxy (High Availability Proxy) is an open-source load balancer and proxy server used to distribute traffic across multiple servers, improving availability and reliability.

What are the key features of HAProxy?Features include load balancing, health checks, SSL termination, logging, rate limiting, and ACL-based request handling.

How does HAProxy improve application performance?By distributing traffic efficiently, reducing server load, and ensuring high availability through failover mechanisms.

What are the different modes of HAProxy?

TCP mode (Layer 4) for raw traffic routing.

HTTP mode (Layer 7) for intelligent routing based on HTTP headers.

Explain the basic HAProxy configuration structure.

global (global settings)

defaults (common configurations)

frontend (handles incoming connections)

backend (defines server pools)

listen (combines frontend and backend)

How do you install HAProxy on a Linux server?

sudo apt update
sudo apt install haproxy -y

What is a frontend in HAProxy?A frontend listens for incoming client connections and directs them to the appropriate backend based on rules.

What is a backend in HAProxy?A backend is a pool of servers that process requests forwarded by the frontend.

How does HAProxy handle health checks?It regularly probes backend servers using TCP or HTTP checks to ensure they are alive and functioning.

What are ACLs in HAProxy?Access Control Lists (ACLs) help filter and route traffic based on conditions like source IP, headers, or URLs.

Intermediate Questions

How does HAProxy handle SSL termination?By using bind with an SSL certificate:

frontend https_frontend
   bind *:443 ssl crt /etc/haproxy/certs/mycert.pem

What is sticky session and how is it implemented in HAProxy?Sticky sessions ensure requests from the same client are routed to the same backend server using cookie or source hashing.

How does HAProxy support high availability?It integrates with Keepalived for failover between multiple HAProxy instances.

Explain load balancing algorithms in HAProxy.

roundrobin (default)

leastconn (least connections)

source (hashing based on source IP)

How do you monitor HAProxy performance?Through the HAProxy Stats page or tools like Prometheus and Grafana.

How does HAProxy integrate with Kubernetes?It can be deployed as an ingress controller for Kubernetes clusters.

Explain rate limiting in HAProxy.Using stick-table to limit request rates per IP:

frontend http_frontend
   stick-table type ip size 100k expire 30s store http_req_rate(10s)
   acl too_many_requests src_http_req_rate(http_frontend) gt 10
   http-request deny if too_many_requests

How do you enable logging in HAProxy?Modify /etc/haproxy/haproxy.cfg to include:

global
    log /dev/log local0

What is a TCP load balancer in HAProxy?HAProxy can operate at Layer 4 to load balance raw TCP connections without inspecting application data.

How does HAProxy integrate with Prometheus and Grafana?By exposing metrics via the HAProxy Exporter.

Advanced Questions

How do you perform blue-green deployments using HAProxy?By dynamically switching traffic between two backend versions.

How does HAProxy handle connection draining?It marks backend servers as DRAIN to allow existing connections to finish while stopping new ones.

How do you secure HAProxy against DDoS attacks?

Use stick-table to rate-limit traffic.

Enable TCP connection limits.

Deploy WAF alongside HAProxy.

Explain the http-reuse feature in HAProxy.It optimizes HTTP connection reuse to backend servers for better performance.

How can you implement canary deployments with HAProxy?Using weighted load balancing with ACLs:

backend app_backend
   server app_v1 192.168.1.10:80 weight 90
   server app_v2 192.168.1.11:80 weight 10

How do you debug HAProxy issues?

Check logs (/var/log/haproxy.log)

Use haproxy -c -f /etc/haproxy/haproxy.cfg to verify config

Enable stats page

How does HAProxy handle WebSockets?By configuring a TCP mode listener:

frontend websocket_frontend
   bind *:8080
   use_backend websocket_backend

How do you reload HAProxy configuration without downtime?

sudo systemctl reload haproxy

How do you implement authentication in HAProxy?Using HTTP basic authentication with http-request auth.

How do you manage HAProxy in a Terraform setup?By defining HAProxy configurations in Terraform scripts and managing instances dynamically.