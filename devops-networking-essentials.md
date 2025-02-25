# Essential Networking Concepts for DevOps Engineers

## 1. Network Fundamentals

### OSI Model Layers
- **Application (Layer 7)**: HTTP, HTTPS, FTP, DNS, SSH
- **Transport (Layer 4)**: TCP, UDP
- **Network (Layer 3)**: IP, ICMP, routing
- **Data Link (Layer 2)**: Ethernet, MAC addresses, switches
- **Physical (Layer 1)**: Cables, physical connections

### IP Addressing & Subnetting
- **IPv4 vs IPv6**: Addressing formats and differences
- **CIDR notation**: Understanding /24, /16, etc.
- **Private IP ranges**: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
- **Subnet masks**: How to calculate subnets and address ranges

### Common Ports & Protocols
| Service | Port | Protocol |
|---------|------|----------|
| HTTP | 80 | TCP |
| HTTPS | 443 | TCP |
| SSH | 22 | TCP |
| DNS | 53 | TCP/UDP |
| SMTP | 25 | TCP |
| PostgreSQL | 5432 | TCP |
| MongoDB | 27017 | TCP |
| Redis | 6379 | TCP |
| Elasticsearch | 9200, 9300 | TCP |

## 2. Network Tools for Troubleshooting

### Command-Line Tools
- **ping**: Test basic connectivity to a host
  ```bash
  ping google.com
  ```
- **traceroute/tracert**: Show network path to destination
  ```bash
  traceroute google.com
  ```
- **nslookup/dig**: DNS lookup
  ```bash
  nslookup google.com
  dig google.com
  ```
- **netstat**: Display network connections
  ```bash
  netstat -tuln
  ```
- **tcpdump**: Capture and analyze network packets
  ```bash
  tcpdump -i eth0 port 80
  ```
- **curl/wget**: Test HTTP endpoints
  ```bash
  curl -v https://api.example.com
  ```
- **nmap**: Network scanning
  ```bash
  nmap -sS 192.168.1.0/24
  ```
- **telnet/nc**: Test specific port connectivity
  ```bash
  telnet example.com 443
  nc -zv example.com 443
  ```

### Common Troubleshooting Scenarios
1. **Connectivity Issues**:
   - Check DNS resolution
   - Verify routing with traceroute
   - Test port connectivity with nc/telnet
   - Validate firewall rules

2. **Performance Issues**:
   - Monitor bandwidth with iperf
   - Check for packet loss with ping
   - Use tcpdump to analyze traffic patterns
   - Review MTU settings

## 3. Network Security

### Firewalls & Network Policies
- **Iptables**: Linux firewall basics
  ```bash
  iptables -L
  iptables -A INPUT -p tcp --dport 22 -j ACCEPT
  ```
- **Security Groups**: Cloud provider firewall rules
- **Network Policies**: Kubernetes network security

### TLS/SSL Concepts
- Certificate management
- Public/private key infrastructure
- Common issues with SSL certificates
- Testing with OpenSSL
  ```bash
  openssl s_client -connect example.com:443 -servername example.com
  ```

## 4. Cloud & Container Networking

### Virtual Private Cloud (VPC)
- **Subnets**: Public vs. private
- **Internet Gateway**: Providing internet access
- **NAT Gateway**: Allowing outbound connections
- **VPC Peering**: Connecting different VPCs
- **Transit Gateway**: Hub for multiple VPC connections

### Kubernetes Networking
- **Pod Network**: How pods communicate
- **Service Types**: ClusterIP, NodePort, LoadBalancer
- **Ingress**: HTTP/HTTPS routing
- **Network Policies**: Pod-level firewall rules
- **CNI Plugins**: Calico, Flannel, Weave, Cilium

### Service Mesh
- **Basics**: Istio, Linkerd, Consul
- **Features**: Traffic management, security, observability
- **Sidecar Pattern**: How proxies intercept traffic

## 5. Load Balancing & Proxies

### Load Balancer Types
- **Layer 4 (Transport)**: TCP/UDP load balancing
- **Layer 7 (Application)**: HTTP/HTTPS content-based routing
- **Global vs. Regional**: Geographic distribution

### Reverse Proxies
- **Nginx**: Common configurations
- **HAProxy**: TCP/HTTP load balancing
- **Envoy**: Modern service proxy

## 6. DNS Configuration & Troubleshooting

### DNS Components
- **Records**: A, AAAA, CNAME, MX, TXT, SRV
- **Nameservers**: Authoritative vs. recursive
- **TTL**: Time-to-live implications

### Common DNS Issues
- Propagation delays
- Split-horizon DNS
- DNS caching
- Troubleshooting with dig/nslookup

## 7. Network Automation

### Infrastructure as Code
- Terraform for network resources
- CloudFormation network stacks
- Ansible for network configuration

### API-driven Networking
- REST APIs for network management
- Programmatic network configuration
- Software-defined networking concepts

## 8. Practical Interview Questions

### Basic Level
1. **What is the difference between TCP and UDP?**
   - TCP: Connection-oriented, reliable, ordered delivery, flow control
   - UDP: Connectionless, faster, no guarantee of delivery, simpler

2. **Explain the difference between a private and public IP address.**
   - Private IPs (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) are for internal networks
   - Public IPs are globally routable and unique on the internet
   - NAT translates between private and public addresses

3. **How would you troubleshoot a connection timeout?**
   - Check DNS resolution with nslookup/dig
   - Verify network path with traceroute
   - Test port connectivity with nc/telnet
   - Check firewall rules
   - Verify service is running on target host

4. **What is a subnet mask and how does it work?**
   - Defines which portion of an IP address is network vs. host
   - Example: 255.255.255.0 (/24) means first 24 bits are network
   - Used to determine if traffic is local or needs routing

### Intermediate Level
1. **How do you set up a Kubernetes Ingress to route traffic to multiple services?**
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: example-ingress
   spec:
     rules:
     - host: app.example.com
       http:
         paths:
         - path: /api
           pathType: Prefix
           backend:
             service:
               name: api-service
               port:
                 number: 80
         - path: /
           pathType: Prefix
           backend:
             service:
               name: frontend-service
               port:
                 number: 80
   ```

2. **Explain how a service mesh works and its benefits.**
   - Service mesh uses sidecars (proxies) alongside each service
   - Intercepts all network traffic to/from services
   - Provides traffic management, security, and observability
   - Benefits: mTLS encryption, fine-grained traffic control, detailed metrics
   - Examples: Istio, Linkerd, Consul

3. **How would you design a multi-AZ VPC in AWS?**
   - Create a VPC with a large CIDR block (e.g., 10.0.0.0/16)
   - Create multiple subnets across AZs:
     - Public subnets with route to internet gateway
     - Private subnets with route to NAT gateway
   - Implement security groups for instance-level security
   - Set up network ACLs for subnet-level security
   - Configure routing tables appropriately for each subnet

4. **What is the difference between an L4 and L7 load balancer?**
   - L4 (Network/Transport): Routes based on IP and port
     - Simpler, faster, handles any TCP/UDP protocol
     - Can't see HTTP content
   - L7 (Application): Routes based on HTTP content
     - Can route based on URL paths, headers, cookies
     - TLS termination, content-based routing
     - More CPU intensive but smarter routing
