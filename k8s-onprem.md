# Control Plane Components and Where to Find Them
The Control Plane is responsible for managing and controlling the Kubernetes cluster, including scheduling, maintaining the cluster’s state, and ensuring that the system behaves according to the desired configuration.

1. Kubernetes API Server (kube-apiserver)
Role: The API server exposes the Kubernetes API and serves as the entry point for interacting with the cluster. It is the central control point for all management tasks.

Where to Find It:

Location: The API Server typically runs on the master node (or control plane node).

Pod Location: It runs as a pod in the kube-system namespace.

Command: You can check the API server pod using:

bash
Copy
Edit
kubectl get pods -n kube-system -l component=kube-apiserver
2. Scheduler (kube-scheduler)
Role: The scheduler is responsible for assigning newly created pods to worker nodes, based on available resources and constraints (e.g., CPU, memory, affinity).

Where to Find It:

Location: It runs on the master node (or control plane node).

Pod Location: It runs as a pod in the kube-system namespace.

Command: You can find the scheduler pod with:

bash
Copy
Edit
kubectl get pods -n kube-system -l component=kube-scheduler
3. Controller Manager (kube-controller-manager)
Role: The controller manager runs controllers that handle the state of the cluster (e.g., replicas, deployments). It is responsible for ensuring that the desired state of the cluster is met by continuously monitoring the cluster and taking necessary actions.

Where to Find It:

Location: It runs on the master node (or control plane node).

Pod Location: It runs as a pod in the kube-system namespace.

Command: You can find the controller manager pod with:

bash
Copy
Edit
kubectl get pods -n kube-system -l component=kube-controller-manager
4. etcd
Role: etcd is a distributed key-value store that holds the cluster’s state and configuration. It contains all the critical data for the cluster, including nodes, pods, services, and secrets.

Where to Find It:

Location: etcd usually runs on the master node (or control plane node).

Pod Location: It runs as a pod in the kube-system namespace.

Command: You can check the etcd pod with:

bash
Copy
Edit
kubectl get pods -n kube-system -l component=etcd
5. Cloud Controller Manager (cloud-controller-manager)
Role: The cloud controller manager is responsible for cloud-specific control logic (e.g., managing cloud load balancers, cloud storage volumes, and networking).

Where to Find It:

Location: Runs on the master node (or control plane node) if your cluster is running on a cloud platform (AWS, GCP, etc.).

Pod Location: It runs as a pod in the kube-system namespace.

Command: To find the cloud controller pod, use:

bash
Copy
Edit
kubectl get pods -n kube-system -l component=kube-cloud-controller-manager

Worker Node Components and Where to Find Them
The Worker Nodes are responsible for running the containers (pods) that host your applications. The components of the worker node help with container orchestration, networking, and managing pod lifecycles.

1. Kubelet (kubelet)
Role: The Kubelet is the primary agent on each worker node. It ensures that containers are running in the pods and that the node is in the desired state. It also reports the node’s status to the Kubernetes API server.

Where to Find It:

Location: It runs on every worker node in the cluster.

Pod Location: It is typically a system service on the node, not running as a pod.

Command: You can find it running as a systemd service or similar, using:

bash
Copy
Edit
systemctl status kubelet
2. Kube Proxy (kube-proxy)
Role: The Kube Proxy maintains network rules on each node and provides services like load balancing across pods. It ensures the network traffic is correctly routed to the right pods in the cluster.

Where to Find It:

Location: Runs on every worker node.

Pod Location: It runs as a pod in the kube-system namespace.

Command: You can find the kube-proxy pod with:

bash
Copy
Edit
kubectl get pods -n kube-system -l component=kube-proxy
3. Container Runtime (e.g., Docker, containerd, CRI-O)
Role: The container runtime is responsible for pulling container images and running the containers. It interacts with the Kubelet to start and stop containers within the pods.

Where to Find It:

Location: It runs on every worker node in the cluster.

Pod Location: It is a system service (e.g., docker or containerd) running on the worker node, not a pod.

Command: To check the status of the container runtime (e.g., Docker), you can use:

bash
Copy
Edit
docker info
Or, for containerd:

bash
Copy
Edit
systemctl status containerd
4. Pods
Role: A Pod is the smallest unit in Kubernetes and consists of one or more containers that share the same network namespace. Pods are where applications are deployed and run.

Where to Find It:

Location: Pods are scheduled to run on worker nodes by the scheduler.

Pod Location: You can see the pods running on a node using:

bash
Copy
Edit
kubectl get pods -o wide
This command will list the pods along with the node they're running on.


