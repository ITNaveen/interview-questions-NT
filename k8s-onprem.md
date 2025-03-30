# Control Plane Components and Where to Find Them
The Control Plane is responsible for managing and controlling the Kubernetes cluster, including scheduling, maintaining the cluster’s state, and ensuring that the system behaves according to the desired configuration.

1. Kubernetes API Server (kube-apiserver)
Role: The API server exposes the Kubernetes API and serves as the entry point for interacting with the cluster. It is the central control point for all management tasks.

Where to Find It:
Location: The API Server typically runs on the master node (or control plane node).
Pod Location: It runs as a pod in the kube-system namespace.
Command: You can check the API server pod using:
kubectl get pods -n kube-system -l component=kube-apiserver

2. Scheduler (kube-scheduler)
Role: The scheduler is responsible for assigning newly created pods to worker nodes, based on available resources and constraints (e.g., CPU, memory, affinity).
Where to Find It:
Location: It runs on the master node (or control plane node).
Pod Location: It runs as a pod in the kube-system namespace.
Command: You can find the scheduler pod with:
kubectl get pods -n kube-system -l component=kube-scheduler

3. Controller Manager (kube-controller-manager)
Role: The controller manager runs controllers that handle the state of the cluster (e.g., replicas, deployments). It is responsible for ensuring that the desired state of the cluster is met by continuously monitoring the cluster and taking necessary actions.

Where to Find It:
Location: It runs on the master node (or control plane node).
Pod Location: It runs as a pod in the kube-system namespace.
Command: You can find the controller manager pod with:
kubectl get pods -n kube-system -l component=kube-controller-manager

4. etcd
Role: etcd is a distributed key-value store that holds the cluster’s state and configuration. It contains all the critical data for the cluster, including nodes, pods, services, and secrets.

Where to Find It:
Location: etcd usually runs on the master node (or control plane node).
Pod Location: It runs as a pod in the kube-system namespace.
Command: You can check the etcd pod with:
kubectl get pods -n kube-system -l component=etcd

5. Cloud Controller Manager (cloud-controller-manager)
Role: The cloud controller manager is responsible for cloud-specific control logic (e.g., managing cloud load balancers, cloud storage volumes, and networking).
Where to Find It:
Location: Runs on the master node (or control plane node) if your cluster is running on a cloud platform (AWS, GCP, etc.).
Pod Location: It runs as a pod in the kube-system namespace.
Command: To find the cloud controller pod, use:
kubectl get pods -n kube-system -l component=kube-cloud-controller-manager

# Worker Node Components and Where to Find Them
The Worker Nodes are responsible for running the containers (pods) that host your applications. The components of the worker node help with container orchestration, networking, and managing pod lifecycles.

1. Kubelet (kubelet)
Role: the kubelet plays a vital role in ensuring that the containers specified by the Kubernetes control plane (through the API server) are running and healthy on each node in the cluster. It listens for instructions from the API server and then takes actions like starting and stopping containers to match the desired state defined in the Kubernetes system.

Where to Find It:
Location: It runs on every worker node in the cluster.
Pod Location: It is typically a system service on the node, not running as a pod.
Command: You can find it running as a systemd service or similar, using:
systemctl status kubelet

2. Kube Proxy (kube-proxy)
Role: The Kube Proxy maintains network rules on each node and provides services like load balancing across pods. It ensures the network traffic is correctly routed to the right pods in the cluster.
Where to Find It:
Location: Runs on every worker node.
Pod Location: It runs as a pod in the kube-system namespace.
Command: You can find the kube-proxy pod with:
kubectl get pods -n kube-system -l component=kube-proxy

3. Container Runtime (e.g., Docker, containerd, CRI-O)
Role: The container runtime is responsible for pulling container images and running the containers. It interacts with the Kubelet to start and stop containers within the pods.
Where to Find It:
Location: It runs on every worker node in the cluster.
Pod Location: It is a system service (e.g., docker or containerd) running on the worker node, not a pod.
Command: To check the status of the container runtime (e.g., Docker), you can use:
docker info
Or, for containerd:
systemctl status containerd

control plane = apiserver, etcd, schedular, cloud-controller, control-manager.
worker = kube-proxy, kubelet, container runtime.

# main asnwer - 
1. Architecture Overview
We have 8 nodes:
    3 Master Nodes → To ensure High Availability (HA)
    5 Worker Nodes → To run workloads
    Calico → For Networking (CNI)
    NFS → For Persistent Storage
    HAProxy → For Load Balancing

```yml 
                    ┌───────────────────┐
                    │  External Clients │
                    └────────▲──────────┘
                             │
                        HAProxy Load Balancer
                             │
       ┌──────────┬──────────┬──────────┐
       │ Master 1 │ Master 2 │ Master 3 │
       └──────────┴──────────┴──────────┘
                │   │        │   │
        ───────────────────────────
         Calico Network (CNI Plugin)
        ───────────────────────────
                │   │        │   │
   ┌──────────┬──────────┬──────────┬──────────┬──────────┐
   │ Worker 1 │ Worker 2 │ Worker 3 │ Worker 4 │ Worker 5 │
   └──────────┴──────────┴──────────┴──────────┴──────────┘
                   NFS Storage Backend

# pre k8s installation - 
Step 1: Configure hosts file on all 8 nodes
Run this on each node (adjust IPs and hostnames to match your environment):
sudo bash -c 'cat >> /etc/hosts << EOF
192.168.1.101 master1
192.168.1.102 master2
192.168.1.103 master3
192.168.1.104 worker1
192.168.1.105 worker2
192.168.1.106 worker3
192.168.1.107 worker4
192.168.1.108 worker5
EOF'

Step2 - set up ssh - 
# Generate SSH key (press Enter for all prompts)
ssh-keygen -t rsa

# Copy the key to all other nodes (you'll need to enter the password for each)
ssh-copy-id user@master2
ssh-copy-id user@master3
ssh-copy-id user@worker1
ssh-copy-id user@worker2
ssh-copy-id user@worker3
ssh-copy-id user@worker4
ssh-copy-id user@worker5

# Test connection (should connect without password)
ssh user@master2 exit


# 2. Preparing Infrastructure
Each master and worker node is a bare-metal server or VM running Ubuntu 22.04 (or RHEL).
a. Setting up nodes
    Set Hostnames & Update System
sudo hostnamectl set-hostname master1
sudo apt update && sudo apt upgrade -y

Disable Swap (Required for K8s)
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

Set Host Entries for Cluster Communication (/etc/hosts)
192.168.1.10 master1
192.168.1.11 master2
192.168.1.12 master3
192.168.1.20 worker1
192.168.1.21 worker2
192.168.1.22 worker3
192.168.1.23 worker4
192.168.1.24 worker5

Install Container Runtime (containerd)  # in all nodes.
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd

Install Kubernetes Packages # in all nodes as well (kubeadm, kubelet, kubectl).
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
sudo apt install -y kubelet kubeadm kubectl
sudo systemctl enable kubelet && sudo systemctl start kubelet

# 3. Installing Kubernetes Cluster with HA.
We are using HAProxy for high availability of the control plane.
a. HAProxy Load Balancer (On a Separate VM) # 9th node.
Install HAProxy
sudo apt install -y haproxy

Configure HAProxy (/etc/haproxy/haproxy.cfg)
frontend k8s-api
    bind *:6443
    default_backend k8s-masters

backend k8s-masters
    balance roundrobin
    server master1 192.168.1.10:6443 check
    server master2 192.168.1.11:6443 check
    server master3 192.168.1.12:6443 check

Restart HAProxy
    sudo systemctl restart haproxy && sudo systemctl enable haproxy

b. Initializing the Kubernetes Control Plane
    Run on Master1
kubeadm init --control-plane-endpoint "haproxy-ip:6443" --upload-certs

Join Master2 & Master3
kubeadm join haproxy-ip:6443 --token <token> --discovery-token-ca-cert-hash <hash> --control-plane --certificate-key <cert-key>

Join Worker Nodes
kubeadm join haproxy-ip:6443 --token <token> --discovery-token-ca-cert-hash <hash>

Setup kubectl on Master
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 4. 4. Networking with Calico
Install Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
Verify
kubectl get pods -n kube-system

# 5.Persistent Storage with NFS
Install NFS Server (On a Separate Storage Node)
sudo apt install -y nfs-kernel-server
sudo mkdir -p /var/nfs/k8s-storage
sudo chmod 777 /var/nfs/k8s-storage
echo "/var/nfs/k8s-storage *(rw,sync,no_root_squash)" | sudo tee -a /etc/exports
sudo exportfs -a
sudo systemctl restart nfs-kernel-server

Mount NFS on Worker Nodes
sudo apt install -y nfs-common
sudo mount 192.168.1.100:/var/nfs/k8s-storage /mnt    # mount on each worker node.

Deploy NFS StorageClass
The StorageClass definition you provided tells Kubernetes:
✅ “I have an NFS server at 192.168.1.100 providing shared storage at /var/nfs/k8s-storage.”
✅ “Whenever a pod requests persistent storage, dynamically use this NFS storage.”

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: example.com/nfs
parameters:
  server: 192.168.1.100
  path: "/var/nfs/k8s-storage"

# 6. Securing the Cluster - 
Enable Role-Based Access Control (RBAC) - 
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-only-binding
subjects:
- kind: User
  name: dev-user
roleRef:
  kind: Role
  name: read-only
  apiGroup: rbac.authorization.k8s.io

Use Network Policies - 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress

Audit Logging - 
kubectl logs --previous <pod-name>

```
# questions - 
2. What is Swap? Why Disable It?
    What is Swap?
        Swap is a portion of the disk used as fake RAM when physical RAM is full.
        If a system runs out of RAM, it moves less-used data from RAM to swap space on disk.
        But disk is much slower than RAM, so swap makes performance very slow.

    Why Disable It in Kubernetes?
        Kubernetes assumes RAM is always real and does not understand swap.
        If swap is enabled, Kubernetes might schedule more pods than the node can handle.
        This can lead to random pod crashes, slow response times, and instability.

3. Why Use Containerd Specifically?
    Lightweight & Optimized → Designed only for running containers (unlike Docker, which has extra overhead).
    t means this runtime (likely referring to containerd or CRI-O) is lightweight and purpose-built for managing containers without extra features like Docker CLI, image building, or networking—it only creates, runs, and stops containers efficiently.

    Directly Integrated with Kubernetes → Used natively by Kubelet without extra dependencies.
    More Secure & Stable → Less attack surface than Docker.
    Faster Startup Times → Containers launch quicker than in Docker.

4. Why Calico? What is BGP?
Calico is not just a CNI plugin—it’s a full networking and security solution. Unlike traditional overlay networks, Calico can operate in pure Layer 3 (L3) mode using BGP, which provides:
✅ Better Performance → No VXLAN/GRE encapsulation overhead.
✅ Easier Troubleshooting → Since it uses normal IP routing, debugging is simpler than encapsulated networks.
✅ Scalability → Works efficiently for clusters with thousands of nodes.
2. Calico Architecture: How It Works in K8s On-Prem
🔹 Data Plane Modes

1️⃣ eBPF Mode (Best for Performance)
    Uses the Linux kernel's eBPF to replace iptables.
    Faster packet processing with low latency.
    Directly integrates with the kernel networking stack.

2️⃣ iptables Mode (Traditional)
    Uses iptables rules for policy enforcement.
    Slightly slower than eBPF but still widely used.

3️⃣ VXLAN Mode (If BGP Is Not Available)
    Uses encapsulation if direct BGP is not possible.
    Helps when dealing with cloud networks that block BGP.

3. Calico and BGP: Why It Matters in On-Prem K8s
🔹 Why Use BGP Instead of VXLAN?
    On-Prem Data Centers Already Use BGP
        In a traditional data center, routers use BGP to exchange routes.
        Calico allows Kubernetes nodes to announce pod network routes to physical routers.

    No Encapsulation = Better Performance
        VXLAN/GRE overlays introduce extra overhead.
        With BGP, pods are directly routable like real servers.

🔹 How BGP Works in Calico?

    Each Kubernetes node runs a BGP agent (called Felix) that:
    ✅ Advertises its pod CIDR to other nodes.
    ✅ Uses BIRD (BGP Daemon) to manage routes.
    ✅ Avoids the need for NAT or overlays (if BGP is fully enabled).

💡 Example Route Table (If BGP is enabled)
Destination CIDR	Next Hop (Node IP)	Interface
192.168.1.0/24	10.0.0.1 (Node 1)	eth0
192.168.2.0/24	10.0.0.2 (Node 2)	eth0

    If a pod on Node 1 wants to talk to a pod on Node 2, it simply follows normal IP routing via BGP-advertised routes.

4. Calico Network Policy: How It Secures On-Prem K8s

Calico provides fine-grained security by allowing NetworkPolicies that control:
✅ Which pods can communicate (zero-trust networking).
✅ Ingress and egress rules for workloads.
✅ Denies all traffic by default (if configured properly).
Example: Deny All Traffic Except for Specific Pods
```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-except-web
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 80
```
🔹 This policy ensures:
    Only pods labeled backend can talk to web pods on port 80.
    No other communication is allowed.

5. Calico in Real On-Prem Deployments

🔹 Scenario: Hybrid Kubernetes Cluster (Bare Metal + Cloud)
    If running on-prem K8s with external cloud services, BGP lets on-prem pods talk to cloud workloads using direct routing.
    Without BGP, you’d need VPNs, tunnels, or NAT, adding complexity.

🔹 Scenario: Multi-Datacenter K8s Cluster
    With BGP Peering, you can connect multiple K8s clusters across data centers without extra overlay networks.

6. Debugging Calico Like a Pro
🔹 Check BGP Peering Status

calicoctl node status
    Shows if BGP peers (routers, nodes) are connected.

🔹 View Network Policies in Effect
calicoctl get policy -o wide

    Lists all security policies applied to the cluster.

🔹 Inspect Pod Routes (BGP Mode)
ip route show
    Shows BGP-learned routes on each node.

5. Why NFS Only? What is NFS and Why Is It Used?
    What is NFS?
        Network File System (NFS) allows multiple computers to access a shared storage over the network as if it were a local disk.

    Why Use NFS in Kubernetes?
        Simple & Reliable → No complex setup like CephFS or GlusterFS.
        Centralized Storage → All worker nodes access the same data.
        Persistent Volumes (PV) Support → Works well with Kubernetes PV/PVC.
        Low Resource Usage → Does not consume CPU/memory like distributed storage solutions.

6. Does sudo mount 192.168.1.100:/var/nfs/k8s-storage /mnt Mount on Each Worker Node?

✅ Yes! This command must be run on each worker node so they can access the shared NFS storage.
💡 In Kubernetes, this is usually automated using a Persistent Volume (PV) and Persistent Volume Claim (PVC).

# Linux - 
```yml
🔹 1. Process & System Monitoring Commands
These help monitor system health, find bottlenecks, and debug issues.
Command	Usage
top / htop	= Shows real-time CPU, memory usage, and running processes. htop is more user-friendly.
ps aux	= Lists all running processes, useful for debugging.
kill -9 <PID>	= Forcefully stops a process by its Process ID (PID).
uptime	= Shows how long the system has been running.
free -m	= Displays available and used memory (RAM) in MB.
df -h	= Shows disk space usage in human-readable format.
du -sh *	= Finds large files or directories inside the current folder.
vmstat 1 5	= Displays CPU, memory, I/O, and swap usage over 5 seconds.

🔹 2. Disk & File Management Commands
These help manage storage, files, and backups.
Command	Usage
ls -lah	Lists files with details, including hidden ones.
find /var/log -type f -name "*.log"	Finds all .log files inside /var/log.
grep "error" /var/log/syslog	Searches for “error” inside syslog logs.
tail -f /var/log/syslog	Live monitoring of logs (real-time updates).
tar -czvf backup.tar.gz /var/www	Compresses /var/www into a .tar.gz file.
scp file.txt user@192.168.1.10:/tmp/	Securely copies file.txt to another server.
rsync -avz /data/ backup:/mnt/storage/	Syncs directories between two systems efficiently.

🔹 3. User & Permission Management
For security and access control.
Command	Usage
whoami	Shows current logged-in user.
id	Displays user ID (UID) and group ID (GID).
chmod 755 file.sh	Changes file permissions (Owner: rwx, Group: r-x, Others: r-x).
chown ubuntu:ubuntu file.sh	Changes file ownership to user ubuntu.
sudo su -	Switches to the root user.
usermod -aG docker myuser	- Adds myuser to the docker group.
passwd	Changes the user password.

🔹 4. Networking Commands
Essential for debugging network issues and securing servers.
Command	Usage
ip a / ifconfig	Shows IP addresses and network interfaces.
ip route show =	Displays routing table (useful for Calico/BGP debugging).
ping 8.8.8.8	Checks if the server can reach Google’s DNS.
nslookup google.com / dig google.com	Checks DNS resolution.
netstat -tulnp / ss -tulnp	Shows open ports and listening services.
curl -I https://example.com	Fetches HTTP headers (useful for API debugging).
wget https://example.com/file.tar.gz	Downloads a file from the internet.

🔹 5. Package Management
For installing, updating, and managing software.
Command	Usage
apt update && apt upgrade -y	Updates all packages (Debian/Ubuntu).
yum update -y / dnf update -y	Updates packages (CentOS/RHEL).
`dpkg -l	grep nginx`
`rpm -qa	grep httpd`
systemctl start nginx / systemctl status nginx	Starts or checks the status of Nginx service.

🔹 6. Kubernetes & Docker Commands
These help manage containers and Kubernetes clusters.
Command	Usage
kubectl get nodes	Lists all Kubernetes nodes.
kubectl get pods -A	Shows all pods in all namespaces.
kubectl logs -f pod-name	Live logs of a running pod.
kubectl exec -it pod-name -- bash	Opens a shell inside a pod.
kubectl describe pod pod-name	Detailed info about a pod (events, IP, status).
docker ps / containerd list	Lists all running containers.
docker logs -f container-id	Live logs of a Docker container.

🔹 7. Security & SSH Commands
For securing servers and remote access.
Command	Usage
ssh user@192.168.1.10	Logs into a remote server.
ssh-keygen -t rsa -b 4096	Generates SSH key pair for authentication.
fail2ban-client status sshd	Checks if Fail2Ban is blocking SSH attacks.
iptables -L -n -v	Shows current firewall rules.
ufw allow 22/tcp	Opens port 22 for SSH (UFW firewall).
`sudo journalctl -u sshd --no-pager	tail -20`

🔹 8. Advanced Linux Debugging
For troubleshooting deep system issues.
Command	Usage
strace -p <PID>	Traces system calls made by a running process.
lsof -i :8080	Shows which process is using port 8080.
`dmesg	tail -20`
iotop	Shows real-time disk usage by processes.
tcpdump -i eth0 port 443	Captures HTTPS traffic on eth0 (for debugging).

# more - 
1️⃣ What is a Hypervisor?
✅ Why Use Type 1 Hypervisor?

A Type 1 hypervisor (bare-metal hypervisor) runs directly on the physical hardware without needing a host operating system. This makes it faster, more efficient, and more secure compared to a Type 2 hypervisor.
🚀 Why Choose Type 1 Over Type 2?
Feature	Type 1 (Bare-Metal)	Type 2 (Hosted)
Performance	Better (direct access to hardware)	Slower (relies on host OS)
Latency	Lower	Higher due to extra OS layer
Security	More Secure (Minimal attack surface)	Less secure (host OS can be exploited)
Use Case	Data centers, cloud, enterprise	Local testing, development
🔹 Real-World Example: Type 1 in Kubernetes On-Prem

If you are setting up an on-prem Kubernetes cluster, you want to maximize performance and security. Type 1 hypervisors like KVM, VMware ESXi, or Xen allow you to:

✅ Create virtualized Kubernetes nodes without OS overhead
✅ Efficiently allocate CPU & RAM to worker nodes
✅ Reduce latency in networking (important for Calico+BGP setups)
✅ Ensure better resource isolation between nodes

💡 Example Setup for On-Prem K8s Cluster:

    Use KVM on physical servers to create 8 virtual machines (VMs) for your Kubernetes cluster.

    Each VM runs a Linux OS (Ubuntu/RHEL) with Kubeadm-installed Kubernetes.

    Networking is handled by Calico BGP, and storage is managed using NFS.



A hypervisor is software or firmware that creates and manages virtual machines (VMs) by allowing multiple operating systems to run on a single physical machine.
Types of Hypervisors:

✅ Type 1 (Bare Metal): Runs directly on hardware, no host OS required.

    Examples: VMware ESXi, Microsoft Hyper-V, KVM, Xen

    Used in: Data centers, cloud environments like AWS

✅ Type 2 (Hosted): Runs on top of a host OS (like an application).

    Examples: VirtualBox, VMware Workstation

    Used in: Development, testing, and local VM setups

💡 In Kubernetes On-Prem, Type 1 Hypervisors (KVM/Xen) are used to create worker nodes in a virtualized environment.

2️⃣ What is SIGKILL (kill -9)?
🔹 SIGKILL (Signal 9) is a forceful termination signal sent to a process.
🔹 It immediately kills the process without allowing cleanup.
🔹 Unlike SIGTERM (kill -15), the process CANNOT catch or ignore SIGKILL.
Common Use Cases for kill -9

✅ When a process is unresponsive or stuck in an infinite loop.
✅ When a process is ignoring normal termination signals (kill -15).
✅ When a process is consuming too much CPU/memory and must be killed immediately.
Example Usage:

ps aux | grep nginx   # Find the PID of the nginx process
kill -9 <PID>        # Force kill the process

🚀 Real-World DevOps Example:
If a Kubernetes pod is stuck and kubectl delete pod <pod> --force --grace-period=0 doesn’t work, you might SSH into the node and kill -9 the container process manually.

..........

🚀 Correct Understanding of Your Setup

1️⃣ You have a physical machine (bare-metal server).
2️⃣ KVM (Type 1 Hypervisor) is installed on the physical machine.
3️⃣ KVM creates 8 Virtual Machines (VMs).

    3 VMs for Kubernetes master nodes

    5 VMs for Kubernetes worker nodes
    4️⃣ Inside each VM, you install Ubuntu/RHEL and Kubernetes using kubeadm.
    5️⃣ Calico is installed for networking (using BGP).
    6️⃣ NFS is set up for persistent storage, mounted on worker nodes.
    7️⃣ HAProxy is installed as a Load Balancer for the Kubernetes API server.
🔹 What This Means for Your Kubernetes Cluster

✅ The 8 VMs are where Kubernetes runs.
✅ They communicate using Calico networking.
✅ They store persistent data on an external NFS server.
✅ HAProxy distributes traffic across master nodes.