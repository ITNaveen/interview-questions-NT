Detailed Explanation of HAProxy and Related Concepts

1. Access Control List (ACL)

What is ACL?

ACL (Access Control List) in HAProxy is a mechanism that allows conditional traffic control based on parameters like IP addresses, headers, or request methods. It is used to define rules that determine how requests are handled before reaching the backend servers.

Why Use ACL?

ACLs are crucial for:

Restricting access based on source IP.

Routing traffic based on URL path, domain, or request method.

Blocking or allowing traffic dynamically.

Example Configuration:

acl block_ip src 192.168.1.100
http-request deny if block_ip

Explanation: Blocks traffic from IP 192.168.1.100.

acl secure_page path_beg -i /admin
use_backend admin_servers if secure_page

Explanation: If the request path starts with /admin, the request is routed to the admin_servers backend.

2. Backend

What is a Backend?

A backend in HAProxy is a pool of servers that handle incoming requests from the frontend.

Why Use a Backend?

Ensures high availability and fault tolerance.

Distributes traffic across multiple servers.

Helps in load balancing.

Example Configuration:

backend web_servers
    balance roundrobin
    server web1 192.168.1.101:80 check
    server web2 192.168.1.102:80 check

Explanation:

balance roundrobin: Distributes traffic evenly among backend servers.

server web1 192.168.1.101:80 check: Defines a backend server with health checks enabled.

3. Types of Load Balancer (LB)

1. No Load Balancer (Direct Connection)

Clients directly connect to backend servers.

No failover or redundancy.

Not recommended for production.

2. Layer 4 Load Balancer (Transport Layer)

Works at the TCP/UDP level (Transport Layer of the OSI Model).

Distributes traffic based on source and destination IPs/ports.

Cannot inspect HTTP headers or content.

Example: TCP Load Balancing.

Example:

frontend tcp_front
    bind *:443
    default_backend web_servers

Explanation: This forwards all traffic coming to port 443 (HTTPS) to the backend servers.

3. Layer 7 Load Balancer (Application Layer)

Works at the HTTP level (Application Layer of the OSI Model).

Can inspect URLs, headers, and cookies.

Provides advanced routing capabilities.

Example: Routing API traffic separately from website traffic.

Example:

frontend http_front
    bind *:80
    acl is_api path_beg /api
    use_backend api_servers if is_api

Explanation:

Routes requests to /api URLs to api_servers backend.

4. Load Balancing Algorithms

1. Least Connections (leastconn)

Sends requests to the backend server with the fewest active connections.

Example:

backend app_servers
    balance leastconn
    server app1 192.168.1.103:80 check
    server app2 192.168.1.104:80 check

2. Source (source)

Ensures the same client always reaches the same backend server using the client’s IP.

Example:

backend session_persistent
    balance source
    server web1 192.168.1.101:80 check
    server web2 192.168.1.102:80 check

5. /etc/hosts in HAProxy Node

What is it?

The /etc/hosts file is used to map IP addresses to hostnames locally, without needing DNS.

Why Use It?

Helps HAProxy resolve backend server names without relying on an external DNS server.

Should be updated on each HAProxy and backend node.

Example:

192.168.1.101 web1
192.168.1.102 web2

6. Installing HAProxy on RedHat

dnf install haproxy -y

Explanation:

dnf install haproxy: Installs HAProxy.

-y: Automatically confirms the installation.

7. HAProxy Configuration Files

What are these?

/etc/haproxy/haproxy.cfg: Main configuration file where HAProxy is set up.

/etc/haproxy/conf.d/: Directory for additional configuration files.

8. Configuring haproxy.cfg

Example 1: Basic HTTP Load Balancing

global
    log /dev/log local0
    maxconn 5000

defaults
    log global
    mode http
    timeout connect 5s
    timeout client 50s
    timeout server 50s

frontend http_front
    bind *:80
    default_backend web_servers

backend web_servers
    balance roundrobin
    server web1 192.168.1.101:80 check
    server web2 192.168.1.102:80 check

9. /etc/rsyslog.conf and imudp

What is rsyslog?

A service that manages system logs.

Why Modify It?

HAProxy logs are not recorded by default, so we enable logging.

How?

Uncomment these lines in /etc/rsyslog.conf:

module(load="imudp")
input(type="imudp" port="514")

Restart rsyslog:

systemctl restart rsyslog

10. Defining Logs for HAProxy

Why?

By default, HAProxy does not store logs.

We define logs to /var/log/haproxy.log.

Steps:

Open /etc/rsyslog.conf.

Add this line:

local2.*    /var/log/haproxy.log

Restart rsyslog:

systemctl restart rsyslog

11. getsebool ha_proxy_connect_any

What is SELinux?

Security Enhanced Linux (SELinux) restricts service access.

Why Enable This?

By default, HAProxy cannot connect to backend servers on any port. This command removes the restriction.

setsebool -P ha_proxy_connect_any on

Explanation:

setsebool: Sets SELinux boolean.

-P: Makes changes permanent.

ha_proxy_connect_any on: Allows HAProxy to connect anywhere.