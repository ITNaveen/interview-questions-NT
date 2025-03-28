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

# secret - 
Step-by-Step Process:
Create a Role to Deny Access (Optional): While Kubernetes doesn’t directly support "deny" rules in the traditional sense, you can restrict access to secrets by not granting the necessary permissions (like get, list, or edit). So, the first step is to ensure that default roles or service accounts don't have access to the secret. This is typically the default behavior unless specifically overridden by roles.

To deny access to a secret for most users, you can simply not assign any permissions to secrets (via roles). This step is generally implicit, as if no roles are assigned, access to secrets is implicitly denied.

If you want explicit control over the denied permissions (though not a true "deny"), you would simply avoid granting the relevant access at the global role level.

Note: Kubernetes doesn’t have a built-in “deny” mechanism, so this step is more about not granting access to most users by default.

Create the Service Account: This is where you create a service account that will be used by the pods you want to grant access to secrets. A service account essentially allows you to associate specific permissions with certain pods or applications.
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secret-access-sa
  namespace: default
This creates a service account called secret-access-sa in the default namespace.

Create a Role to Allow Access: Now, create a Role that allows access to the secret for specific actions like get, list, and read. You will specify what actions can be performed on the secret (e.g., get, list, read).
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: allow-secret-access
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
    resourceNames: ["secret-x"]  # This is the specific secret name you want to grant access to
The Role grants get and list permissions for the secret secret-x in the default namespace.

Create a RoleBinding to Bind the Role to the Service Account: The RoleBinding is what connects the Role to the Service Account (thus giving that Service Account the permissions defined in the Role). This makes sure that only pods using the specific service account will be allowed to access the secret.
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-access-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: secret-access-sa
    namespace: default  # Refers to the service account
roleRef:
  kind: Role
  name: allow-secret-access
  apiGroup: rbac.authorization.k8s.io
The RoleBinding links the allow-secret-access Role to the secret-access-sa Service Account.

This means that only pods that use this service account will have access to the secret.

Use the Service Account in Your Deployment or StatefulSet: Now, in your Deployment or StatefulSet, you need to specify the service account (secret-access-sa) that you created earlier. This tells Kubernetes to use this service account for the pod, thereby granting it the permissions defined in the Role (i.e., access to the secret secret-x).

Example of a Deployment using the service account:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: myapp
    spec:
      serviceAccountName: secret-access-sa  # This service account is tied to the role with secret access
      containers:
      - name: myapp-container
        image: nginx
        env:
        - name: SECRET_USERNAME
          valueFrom:
            secretKeyRef:
              name: secret-x
              key: username
        - name: SECRET_PASSWORD
          valueFrom:
            secretKeyRef:
              name: secret-x
              key: password
In this Deployment, the pod will use the secret-access-sa service account, which has been granted access to the secret secret-x through the RoleBinding.

Now, only this Deployment (or StatefulSet) with this specific service account will be able to access the secret-x.


