# DevOps Networking Interview Guide: 50 Essential Questions & Answers
Foundational Networking (15 Questions)

Basic Network Fundamentals
# Q1: Explain the OSI model and its layers. How do they interact?
Think of it like sending a letter through a post office! 📬

1️⃣ Physical (Layer 1) – Wires & Signals
    Deals with raw data transmission (bits: 1s & 0s).
    Example: Cables, Wi-Fi, fiber optics, switches, hubs.
    🔹 Like a road for sending letters.

2️⃣ Data Link (Layer 2) – MAC Address & Switching
    Ensures error-free data transfer between two directly connected devices.
    Example: MAC addresses, Ethernet, Switches.
    🔹 Like a mailman who delivers letters based on house numbers (MAC address).

3️⃣ Network (Layer 3) – IP Address & Routing
    Handles packet forwarding and routing between networks.
    Example: IP addresses, Routers.
    🔹 Like a post office that decides where to send your letter (IP routing).

4️⃣ Transport (Layer 4) – Reliable Delivery (TCP/UDP)
    Ensures end-to-end communication, error checking, and flow control.
    Example: TCP (reliable, like a delivery receipt), UDP (fast, no receipt).
    🔹 Like choosing Express Mail (TCP) vs. Postcard (UDP).

5️⃣ Session (Layer 5) – Keeping Connections Open
    Manages start, end, and maintenance of communication sessions.
    Example: Logging into a website, VoIP calls.
    🔹 Like a customer service call that stays open while you talk.

6️⃣ Presentation (Layer 6) – Data Translation & Encryption
    Converts data into a format the application understands (encoding, encryption).
    Example: SSL/TLS encryption, file formats (JPEG, MP3).
    🔹 Like translating a letter into another language before sending.

7️⃣ Application (Layer 7) – User Interaction
    The interface users interact with.
    Example: Web browsers, email apps, WhatsApp.
    🔹 Like writing a letter before sending it.

# "Please Do Not Throw Sausage Pizza Away"
(Physical, Data Link, Network, Transport, Session, Presentation, Application)

# Q2: What is the difference between TCP and UDP?
TCP (Transmission Control Protocol):
Connection-oriented protocol
Guarantees packet delivery
Provides error checking and recovery
Slower but reliable
Used for web browsing, email, file transfers
Here’s how it works in simple terms:
Connection Establishment (Handshake) – Before sending data, TCP creates a connection between two devices using a three-step process called the three-way handshake (SYN, SYN-ACK, ACK). This ensures both sides are ready to communicate.
Data Transmission – TCP breaks data into small packets, numbers them, and sends them one by one. The receiver checks for missing or corrupted packets and requests a resend if needed.
Acknowledgment – The receiver sends an acknowledgment (ACK) for each packet received correctly. If an ACK isn’t received, TCP resends the lost packet.
Connection Termination – Once data transfer is complete, TCP closes the connection properly to free up resources.

UDP (User Datagram Protocol):
Connectionless protocol
No guarantee of packet delivery
Faster, lower overhead
No error recovery
Used for streaming, gaming, DNS

# Q3: What is a subnet and how do you calculate subnet masks?
# subne mask - 
Binary  -       1       1       1       1       1       1       1       1   
Decimal -       128     64      32      16      8       4       2       1
Subnet_mask -   128     192     224     240     248     252     254     255

# classes - 
Class A: 1-126 (Subnet mask: 255.0.0.0) or /8.
Class B: 128-191 (Subnet mask: 255.255.0.0) /16.
Class C: 192-223 (Subnet mask: 255.255.255.0) /24.
A subnet divides a larger network into smaller, more manageable networks.
Subnet Calculation Example:

IP: 192.168.1.0/24
Total Hosts: 256 (254 usable)
Subnet Mask: 255.255.255.0

Subnetting Formula:
2^(32 - subnet bits) = Total Addresses
subnet = 2 to the power borrowed bits and here 24 has no borrowed bits so 1 subnet only.
/20 = 4 borrowed bit = 16 subnet
/27 = 3 so 8 subnets 

# Q4: Explain the difference between public and private IP addresses.
Public IP Addresses:
Globally unique
Routable on the internet
Assigned by ISPs
Unique across all networks

Private IP Addresses:
Used within local networks
Not routable on public internet
Address ranges:
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16

# Q5: What is NAT (Network Address Translation)?
NAT allows multiple devices in a local network to access the internet using a single public IP address.
NAT (Network Address Translation) is the process of modifying the IP address information in the IP header of packets as they pass through a router or firewall. NAT allows devices on a private network (using private IP addresses) to communicate with devices on the internet (using public IP addresses).
- Types of NAT:
Static NAT: One-to-one mapping of private to public IP
1 private IP ↔ 1 public IP.
Permanent mapping.
Used when a device needs to be consistently reachable via the same public IP (e.g., web servers).

Dynamic NAT: Multiple private IPs mapped to a pool of public IPs

Port Address Translation (PAT): Multiple private IPs use a single public IP with different ports
1 public IP for many private IPs.
Port numbers differentiate the devices.
Most commonly used for home and office networks.

# Q6: What is DHCP and how does it work?
Dynamic Host Configuration Protocol (DHCP) automatically assigns IP configurations to network devices.
DHCP Pool:
The router has a range of IP addresses (called a DHCP pool) that it can assign to devices. For example, it might assign IPs from 192.168.1.100 to 192.168.1.200.
Each time a new device joins the network, the router gives it an available IP from the pool, making sure each device gets a unique IP address to avoid conflicts.

Leasing IPs:
The router leases an IP address to the device for a specific period (called the lease time). Once the lease expires, the device may renew the lease to keep using the same IP.

DHCP process - 
DHCP Discover: Client broadcasts request
DHCP Offer: Server proposes IP configuration
DHCP Request: Client accepts offer
DHCP Acknowledgment: Server confirms assignment

# Q7: Explain the basic routing concepts.
Routing is the process of choosing the best path for sending data (like a message or webpage) from one device to another across a network.
Key Routing Concepts:
Routing Table: Map of network paths

What Happens When a Client Asks for an IP Address?
When a client device (like a computer) asks the router to reach an IP address, the router looks in its routing table to determine the best path for that destination.
If the router has a direct route (like a network it is directly connected to), it sends the traffic directly there.
If the router does not know how to reach the destination (like an unknown network), it will use the default route (if configured) or send the traffic to the next-hop router.

Routing Protocols:
Static Routing: Manually configured routes (LAN based)
 If a request (packet) comes to the router for a destination network that is not defined in the routing table, the router will not know how to forward that packet, and it will be dropped (rejected). This is because the router doesn't have a path to the destination network.
it is only configured for certain networks only.

Dynamic Routing: Automatically updated routes (OSPF, BGP)
Hop Count: Number of network devices between source and destination
Metric: Cost of sending data through a specific route
OSPF (or RIP) helps routers discover routes dynamically and automatically update their routing tables. If a client’s request can’t be fulfilled by a router’s existing route, the router will ask its neighbors (other routers) for the best path.

# Q8: What is a firewall and its types?
A firewall monitors and controls network traffic based on security rules.
Firewall Types:
1. Packet Filtering Firewall
What it does: A packet filtering firewall looks at individual packets that flow through the network. It compares each packet against a set of predefined rules (like source and destination IP, port, and protocol) to decide whether to allow or block it.
Where it's used: Typically placed at the network perimeter (the edge of your network, usually in front of your router or as the first line of defense).
Works at network layer

Configuring Rules: As an admin, you’ll create rules that specify which IP addresses or subnets are allowed or denied. For example, you might block incoming traffic to certain ports (like port 80 for HTTP) or allow traffic only from certain trusted IPs.
Example: You might configure it to only allow SSH (port 22) traffic from a specific range of IPs (e.g., only from internal offices or VPN clients).
Cons: No awareness of connection state, so it cannot detect if an attacker has hijacked a session or if a packet is part of an ongoing, legitimate communication.

2. Stateful Inspection Firewall (transport layer)
What it does: A stateful inspection firewall tracks the state of connections. It remembers the state of each active connection (e.g., TCP connection). If an incoming packet is part of an existing session, it’s allowed; if it’s new or doesn’t match any active session, it’s blocked.
Decision Criteria: It checks whether the incoming packet is part of an existing connection and maintains a state table to track the status of connections.

Connection Tracking: As an admin, you'd configure rules that don’t just look at the individual packets but also whether a packet is part of an established connection. For example, if a packet is from a legitimate user accessing a web server, it will be allowed through based on session state.
Example: For an internal office network, you'd set up rules to allow TCP sessions to pass through based on the connection status. If the client has established an SSH session, further packets in the same session will be allowed.

3. Application Layer Firewall
Analyzes traffic at application layer
What it does: The application layer firewall analyzes traffic at the application layer (Layer 7). It understands and inspects application-specific protocols like HTTP, FTP, DNS, etc. It can inspect content (e.g., look for SQL injections or malware in HTTP requests) and is more thorough in filtering malicious traffic.

Decision Criteria: It checks the data being transmitted for malicious content, protocols, or unauthorized requests.
Example: If you have a web server handling HTTP traffic, you'd configure your application firewall to inspect HTTP requests, blocking any requests that contain malicious SQL queries or unauthorized access attempts to sensitive areas like /admin.
Cons: More resource-intensive than other firewalls, as it has to inspect the content of each packet.

4. Proxy Firewall:
How it works: A proxy firewall acts as an intermediary between the internal network and the external world. It filters traffic and can hide the internal network's IP addresses.
Layer: Typically operates at the Application Layer.
Decision Criteria: It may filter content based on specific protocols (HTTP, FTP, etc.) and can prevent direct connections to the network, requiring requests to go through the proxy first.
Pros: Provides extra security by hiding the real network structure from external entities.

Example: In a corporate network, you could use a proxy firewall to allow employees to browse the web but filter out malicious websites or monitor internet usage. You could also use it for traffic caching (saving bandwidth) and user authentication.
Cons: Latency is introduced because traffic has to pass through the proxy. Not suitable for all types of traffic (like real-time services).

# Q9: Explain network switching concepts.
Switching is the process of forwarding data packets between devices on a local area network (LAN). Unlike routers, which forward packets between different networks, switches are primarily concerned with forwarding data within the same network or subnet.

frame - So, when Device A sends data to Device B, the data is wrapped in a frame that includes:
Source MAC address (Device A’s MAC address)
Destination MAC address (Device B’s MAC address).

ARP - (Address Resolution Protocol) is typically used by devices (like Device A) to resolve IP addresses to MAC addresses. ARP is not directly used by switches — switches are concerned with MAC addresses, and ARP is used to map an IP address to a MAC address. The switch doesn't need to use ARP; it just forwards frames based on MAC addresses.

Example Scenario:
Device A sends a frame to Device B.
The frame has Device A's MAC as the source and Device B's MAC as the destination.
The switch receives the frame and looks at its MAC address table to find Device B's MAC address.
If Device B's MAC address is in the table, the switch knows the port where Device B is connected and forwards the frame to that port.
If Device B's MAC address is not in the table, the switch doesn't know where to send the frame, so it broadcasts (floods) the frame to all other ports except the one it came from.
The actual data being sent.

Switch Types:
Layer 2 Switch: Uses MAC addresses
Layer 3 Switch: Combines switching and routing
Layer 3 Switch is used when you need to handle routing between different networks/subnets in addition to regular switching. It combines the functionality of both Layer 2 switches (MAC address-based forwarding) and routers (IP address-based routing).

Managed vs. Unmanaged Switches

# Q10: What is a VLAN and its purpose?
A Virtual LAN (VLAN) is a technology that allows a network to be logically segmented into different sub-networks, despite all devices being connected to the same physical network. By grouping devices into VLANs, you can organize the network more efficiently and manage traffic more effectively.

Purpose of VLAN:
Logical Segmentation: VLANs allow you to segment a network based on factors like department, project, or function, even if devices are physically connected to the same switch or router. This segmentation is independent of the physical layout.
Improved Network Performance: By reducing the size of broadcast domains, VLANs reduce unnecessary traffic across the network, improving overall network efficiency.
Enhanced Security: VLANs can isolate traffic between different segments of the network. For example, sensitive data traffic can be placed in a separate VLAN to prevent unauthorized access.
Reduced Broadcast Domain: Since broadcast traffic is confined to a specific VLAN, it helps reduce the impact of broadcast storms and unnecessary network load.
Flexible Network Design: VLANs allow for easier changes in network configurations without needing to rewire physical connections, making network management and growth more adaptable.

VLAN Types:
1. Port-based VLAN - 
In this scenario, you’re using the physical ports on your router (or switch) to define which devices belong to which VLAN.
Example: If your router has 20 ports, you could allocate:
Ports 1-5 for VLAN1
Ports 6-10 for VLAN2
Ports 11-15 for VLAN3
Ports 16-20 for VLAN4
This means that any device physically connected to ports 1-5 will be in VLAN1, any device connected to ports 6-10 will be in VLAN2, and so on.

2. Protocol-based VLAN - 
This type of VLAN assigns devices to VLANs based on the protocol they use, rather than their physical port or MAC address.
Example: You can group traffic by protocol:
VLAN1 for HTTP traffic (port 80)
VLAN2 for HTTPS traffic (port 443)
VLAN3 for SSH traffic (port 22)
VLAN4 for FTP traffic (port 21)
Devices using HTTP, regardless of the physical port they are plugged into, will be placed in VLAN1. Similarly, devices using HTTPS will go into VLAN2, and so on.

3. MAC Address-based VLAN
In this setup, the VLAN membership is determined by the MAC address of the devices. It doesn’t matter which physical port the device is plugged into; as long as the device’s MAC address matches a defined rule, it will be assigned to a specific VLAN.
Example: You could choose to group specific devices based on their MAC addresses, even if they are spread across different physical ports on the switch.
Devices with MAC addresses 00:1A:2B:3C:4D:5E, 00:1A:2B:3C:4D:5F, etc., might be placed in VLAN1.
Devices with MAC addresses 00:6A:7B:8C:9D:0F, 00:6A:7B:8C:9D:10, etc., could be placed in VLAN2.
This way, you can pick specific devices (e.g., 5 out of 20 devices) and group them together in the same VLAN, regardless of their physical location in the network.


# Q11: Explain DNS and its resolution process.
Domain Name System (DNS) translates domain names to IP addresses.
Step-by-Step DNS Resolution:
Recursive DNS Resolver:
1. When you type a domain name (e.g., example.edu) into your browser, the recursive DNS resolver (usually provided by your ISP or a third-party service like Google DNS or Cloudflare DNS) starts the resolution process.
The recursive DNS resolver is responsible for finding the IP address that corresponds to the domain name. If the address is not already cached (stored from previous queries), it starts the lookup process.

2. Query to Root DNS Servers:
The recursive DNS resolver first sends a query to one of the root DNS servers. There are 13 root DNS servers worldwide, and they don’t store actual IP addresses for specific domains.
Root DNS servers don't directly know where example.edu is hosted. Instead, **root DNS servers know where to find the authoritative servers for different Top-Level Domains (TLDs), like .edu, .com, .org, .net, etc.

3. Root DNS Server Response:
The root DNS server receives the query and does not look up the IP address directly. Instead, it points the recursive resolver to the appropriate TLD DNS server based on the extension of the domain you're querying.
For example, if you are looking up example.edu, the root DNS server knows that the .edu TLD is managed by a particular set of TLD DNS servers.
The root DNS server doesn’t provide the IP for example.edu. Instead, it sends the resolver to the TLD DNS servers for .edu (the .edu TLD servers).

4. Query to TLD DNS Server (e.g., .edu TLD):
The recursive DNS resolver then sends the query to the TLD DNS server for .edu.
The TLD DNS server doesn’t know the exact IP address for example.edu either, but it knows which authoritative name servers are responsible for this domain.

5. TLD DNS Server Response:
The TLD DNS server (for .edu in this case) responds with the address of the authoritative DNS servers for example.edu.
This response includes the names or IP addresses of the authoritative name servers that manage the DNS records for example.edu.

6. Query to Authoritative DNS Servers:
The recursive DNS resolver now sends a query to the authoritative DNS server for example.edu.
The authoritative DNS server is the one that actually stores the DNS records for example.edu, including the IP address (like 192.0.2.1).

7. Authoritative DNS Server Response:
The authoritative DNS server responds with the IP address for example.edu. This is the final step where the IP address of the domain is obtained.

- DNS Record Types:
A (IPv4) - 
Purpose: The A record (Address record) maps a domain name to an IPv4 address (e.g., 192.0.2.1). This is the most common DNS record used for associating a domain name with an IP address.
Use: When you type a domain name (like example.com) into your browser, the A record tells the browser the corresponding IPv4 address of the web server that should be contacted to load the website.

AAAA (IPv6) - Just like the A record, the AAAA record maps a domain to an IP address, but this time it's for the newer IPv6 format, which allows for a larger address space than IPv4.

CNAME - 
Purpose: The CNAME record (Canonical Name record) is used to alias one domain name to another. Essentially, it allows you to point a domain to another domain rather than directly to an IP address.
Use: CNAME records are typically used when you want multiple domain names to resolve to the same IP address but want to avoid the need to update the A or AAAA records for each domain if the IP changes.
For example, you might want www.example.com to point to example.com. This way, both www.example.com and example.com will resolve to the same IP address without needing to update them individually.

MX - 
Step-by-Step Email Sending (Simplified):
You send the email to nt@cloudsaver.com from your email client.
The MTA (Mail Transfer Agent) needs to find the mail server for cloudsaver.com. It does this by looking up the MX records for cloudsaver.com in DNS.
DNS Query:
Your DNS resolver (e.g., your ISP’s DNS or Google DNS) starts by querying the root DNS servers.
The root DNS server directs the resolver to the TLD DNS servers for .com.
The TLD DNS servers for .com direct the resolver to the authoritative DNS servers for cloudsaver.com.
Authoritative DNS Server: This server holds the MX records for cloudsaver.com and provides them to the DNS resolver.
Mail Server Selection:
The MTA checks the MX records and picks the mail server with the lowest priority number.
IP Address Resolution:
The MTA queries DNS again to find the IP address of the chosen mail server.
Email Sent: The MTA sends the email to the mail server using the IP address.
Recipient’s Mailbox: The email is stored in nt@cloudsaver.com’s mailbox for retrieval.

# Q13: Explain network load balancing.
Load balancing distributes network traffic across multiple servers.

Load Balancing Techniques:
1. Round Robin - 
Description: Distributes requests evenly across all available servers, one after another in a circular order.
Use Case: Good for situations where all servers are equally capable of handling the same type of traffic (e.g., simple web servers).
Example: If there are 3 servers, the first request goes to Server 1, the second to Server 2, the third to Server 3, and the fourth request goes back to Server 1.

2. Least Connections - 
Description: Directs traffic to the server with the fewest active connections at the moment. This helps balance the load when traffic patterns are uneven.
Use Case: Effective in scenarios where some servers may be more heavily loaded than others or in applications where processing time varies (e.g., databases or dynamic web apps).
Example: If Server 1 has 3 active connections and Server 2 has 5, the next request would be sent to Server 1.

3. IP Hash - 
Description: Uses a hash function to assign requests based on the client's IP address. This ensures that the same client always connects to the same server, which can be important for session persistence.
Use Case: Useful for applications that require session persistence or sticky sessions (e.g., e-commerce sites where user login is involved).
Example: A user's IP address could be hashed to determine which server they will interact with, ensuring consistent behavior across sessions.

4. Weighted Algorithms - 
Description: Assigns different weights to different servers based on their capacity or other factors. Servers with higher weights will receive more traffic.
Use Case: Useful when servers have unequal capacities (e.g., one powerful server and several less powerful ones).
Example: If Server 1 has a weight of 3 and Server 2 has a weight of 1, Server 1 would handle three times as many requests as Server 2.

Types:
Layer 4 (Transport Layer) - 
Description: Operates at the transport layer (Layer 4) of the OSI model, which deals with TCP/UDP traffic. It makes load balancing decisions based on IP addresses, ports, and protocols (like HTTP, HTTPS, FTP).
Use Case: Typically used for faster load balancing where the application does not need to inspect the data being transferred, such as in simple web traffic or database connections.
Example: A load balancer might route traffic based on the IP address and port number alone, without understanding the application data.

Layer 7 (Application Layer) - 
Description: Operates at the application layer (Layer 7) of the OSI model, which is the layer that deals with application data, like HTTP headers, cookies, or URL paths. It makes more complex decisions based on the content of the request.
Use Case: Suitable for applications that require more intelligent routing based on specific content or context, such as different types of web traffic (e.g., HTTP, HTTPS), user-agent strings, or URL paths.
Example: A Layer 7 load balancer could route traffic based on the requested URL (/images might go to one server, while /videos might go to another) or cookies (redirecting traffic for a specific user session to the same server).

# Q14: What is a proxy server?
A proxy server acts as an intermediary between client and destination server.

Proxy Types: - 
1. Forward Proxy - 
Description: A forward proxy sits between a client and the internet. It forwards requests from clients (like web browsers) to the destination server.
Use Case: Typically used for client-side applications where the client is hidden from the internet. It is often used in corporate networks to control and monitor employee internet access.
Example: An employee in a company might use a forward proxy to access websites on the internet, but the proxy server forwards the requests to the destination website.

2. Reverse Proxy -
Description: A reverse proxy sits between the internet and a web server (or a group of servers). It handles incoming traffic from external clients (such as users accessing a website) and forwards it to one or more backend servers.
Use Case: Often used for load balancing, security (hiding the backend servers), or caching.
Example: A company’s website may use a reverse proxy to distribute incoming web traffic across several web servers to optimize performance and ensure high availability.

# Advanced DevOps Networking Interview Guide
Kubernetes and Container Networking Advanced Concepts
# Q16: Explain the Kubernetes Networking Model in Comprehensive Detail
Kubernetes introduces a unique networking model that fundamentally differs from traditional network architectures. In this model, every pod receives a unique IP address, effectively treating each pod as a complete virtual machine with its own network stack. This approach eliminates the need for port mapping and enables direct communication between pods across different nodes without network address translation (NAT).
The core philosophy is flat network space where containers can communicate seamlessly. Each pod gets its own IP, and containers within the same pod share the network namespace, meaning they can communicate via localhost. This design simplifies inter-pod communication and provides a consistent networking experience regardless of the underlying infrastructure.
Pods do not get IP addresses directly from the underlying host network. Instead, the Container Network Interface (CNI) plugin assigns IPs. Kubernetes supports different CNI plugins, each handling pod IP assignment differently. 

# Q17: Describe Service Networking in Kubernetes in Depth
What is a Kubernetes Service?
A Service in Kubernetes provides a stable IP and DNS name for a set of dynamically changing pods.
Since pods are ephemeral (they come and go), a Service ensures that network traffic always reaches the correct pods without worrying about pod IP changes.
It acts as a load balancer, distributing requests across multiple pod instances

Kube-Proxy Assigns Static IPs to Services
When you create a Service, Kubernetes assigns it a static internal IP (ClusterIP) from the service range (default: 10.x.x.x or 172.x.x.x).
This IP does not belong to any specific node—it is virtual, managed by kube-proxy.

1️⃣ Does Kube-Proxy Have a Collection of IPs Like a Switch?
✅ Yes, but only for Services, not Pods.
Kube-Proxy does not assign IPs to pods.
Pod IPs are assigned by the Container Network Interface (CNI) plugin (Flannel, Calico, Cilium, etc.).
However, Kube-Proxy assigns a stable ClusterIP to each Service from a predefined Service IP range (10.x.x.x or 172.x.x.x).

✅ Where Do These IPs Come From?
When the Kubernetes cluster starts, the API server reserves a block of IPs for services.
Example:
10.96.0.0/12 is reserved for Services.
kube-proxy picks an available IP from this pool when a new Service is created.

Scenario: Pod-to-Pod Communication on the Same Node - Each worker node in your Kubernetes cluster has a CNI plugin installed. This is responsible for managing the network interfaces for all the Pods running on that node.
If Pod A (192.168.1.10) wants to talk to Pod B (192.168.1.11) (on the same node):
Pod A sends the request directly to Pod B’s IP.
Since both pods share the same node’s network, they communicate without kube-proxy.

How Does Pod-to-Pod Communication Work Across Nodes?
Now, if Pod A (on Node 1) wants to talk to Pod B (on Node 2):
Pod A sends traffic to Pod B’s IP (192.168.2.15).
Since Pod B is on a different node, the CNI plugin ensures routing between the nodes.
No need for kube-proxy unless the request is going through a Service.

Example: Pod Communicating via a Service
🔹 Scenario:
Pod A wants to communicate with backend-service (10.96.0.5) instead of a specific pod.
The backend-service is backed by multiple pods (backend-pod-1, backend-pod-2).
🔹 What Happens Step-by-Step?
1️⃣ Pod A sends a request to backend-service (10.96.0.5).
2️⃣ kube-proxy intercepts the request and checks iptables/IPVS rules.
3️⃣ It sees that backend-service is mapped to backend-pod-1 (192.168.1.12) and backend-pod-2 (192.168.1.13).
4️⃣ kube-proxy load balances the request to one of the backend pods.
5️⃣ Traffic reaches the selected pod, and the response is sent back to Pod A.

# Q18: Elaborate on Container Network Interface (CNI) Architecture
🔹 CNI’s Role (Container Network Interface):
Pod IP Assignment:
CNI is responsible for assigning IP addresses to Pods. When a Pod is created on a node, the CNI plugin allocates a unique IP address for the Pod (from the CNI network range).
It also configures the network interface within the Pod, so that the Pod can communicate with other Pods within the cluster and potentially outside the cluster.

🔑 Key Point: CNI handles Pod networking, including assigning IP addresses and managing network interfaces for the Pods themselves.
Network Setup:
The CNI plugin configures the network interface within each Pod and ensures the Pod can communicate with other Pods either on the same node or across nodes.
Routing rules and network policies (e.g., using Calico, Flannel, or Cilium) are configured by CNI to manage how traffic flows between Pods.

kube-proxy ensures that a Service IP (e.g., 10.96.0.10) maps to one or more backend Pods and directs traffic to the appropriate Pod. It is responsible for service discovery and traffic routing.

# Q19: Network Policies in Kubernetes: A Comprehensive Exploration
Kubernetes Network Policies represent a powerful mechanism for implementing network-level security and segmentation within a cluster. They function as firewall rules specifically designed for container environments, allowing administrators to define precise communication rules between pods.
These policies operate at the network layer, enabling granular control over ingress and egress traffic. An administrator can specify which pods can communicate with each other based on labels, namespaces, or IP address ranges. This approach provides a declarative way to implement micro-segmentation, ensuring that only authorized network communications are permitted.

Q20: Service Discovery Mechanisms in Kubernetes
Service discovery in Kubernetes is a complex yet elegant system that enables dynamic and reliable network communication. The primary mechanism involves CoreDNS, which automatically creates DNS records for services within the cluster. Each service gets a DNS name that resolves to its corresponding cluster IP, allowing pods to discover and communicate with services using human-readable names.
Simultaneously, Kubernetes uses environment variable injection and the kube-proxy component to facilitate service discovery. When a pod is created, Kubernetes automatically injects environment variables containing service connection details. The kube-proxy runs on every node, maintaining network rules that enable transparent routing to the appropriate service endpoints.
