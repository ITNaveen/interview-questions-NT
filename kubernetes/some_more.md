Kubernetes Advanced Concepts

1. Cluster Autoscaler & Cost Optimization

1.1 What is Cluster Autoscaler?

Cluster Autoscaler automatically adjusts the number of nodes in a Kubernetes cluster based on pending pod scheduling requests. It adds nodes when pods are unschedulable due to insufficient resources and removes underutilized nodes to optimize costs.

1.2 How Cluster Autoscaler Works
Scale Up: If pods are in Pending state due to resource constraints, Cluster Autoscaler adds new nodes to the cluster.
Scale Down: If nodes are underutilized and pods can be rescheduled onto other nodes, it removes the underutilized nodes.

1.3 Setting Up Cluster Autoscaler on AWS EKS
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/cluster-autoscaler-autodiscover.yaml
Ensure IAM roles allow Autoscaler to modify nodes.
Configure autoscaling groups properly in AWS.

1.4 What is Karpenter? (Cost-Optimized Scaling)
Karpenter is an alternative to Cluster Autoscaler that:
Provisions nodes faster using EC2 Fleet, Spot Instances, and ARM-based instances.
Optimizes cost by launching right-sized nodes dynamically.
Reduces waste by consolidating workloads on fewer nodes.

1.5 Key Differences: Cluster Autoscaler vs. Karpenter

Feature                                         Cluster Autoscaler                                            Karpenter

Scaling Speed                                   Slower (relies on ASG scaling)                                Faster (direct EC2 API calls)

Cost Optimization                               Basic                                                         Advanced (uses Spot, ARM, etc.)

Node Provisioning                               Fixed instance types                                          Flexible instance types

# Karpenter Explained

Karpenter is an open-source Kubernetes node provisioning system designed to dynamically create and terminate EC2 instances based on actual workload demands. It is built specifically for AWS and replaces Cluster Autoscaler by offering faster scaling, better cost optimization, and more flexible instance provisioning.

Unlike Cluster Autoscaler (CA), which depends on Auto Scaling Groups (ASGs), Karpenter provisions nodes directly using EC2 API calls, enabling faster and more efficient scaling.
Why Choose Karpenter Over Cluster Autoscaler?

Here’s why Karpenter is better in many AWS scenarios:
1. Faster Scaling

    Cluster Autoscaler: Works by adjusting Auto Scaling Group (ASG) sizes, which can be slow.
    Karpenter: Directly provisions EC2 nodes via API calls, making scaling much faster.

✅ Example: If a workload suddenly needs more resources, Karpenter can immediately provision new nodes, whereas CA takes time due to ASG-based scaling delays.
2. No Need for Auto Scaling Groups (ASGs)

    Cluster Autoscaler: Requires manually setting up ASGs with a fixed instance type and a minimum/maximum node count.
    Karpenter: Removes ASG dependency and provisions nodes dynamically based on actual workload demand.

✅ Example: If the cluster needs 5 nodes today and 50 tomorrow, Karpenter scales automatically—no need to predefine limits like in ASGs.
3. Cost Optimization

    Cluster Autoscaler: Can only select from a fixed list of ASG instance types, with limited Spot Instance support.
    Karpenter: Automatically picks the most cost-efficient instance types, including Spot, ARM-based, and Graviton instances.

✅ Example: If Spot Instances are cheaper, Karpenter will prioritize them over On-Demand instances, reducing AWS costs significantly.
4. Flexible Instance Provisioning

    Cluster Autoscaler: Requires manually defining the exact instance types inside ASGs (e.g., t3.medium).
    Karpenter: Can either automatically select the best instance type or allow specific instance types you define.

✅ Example: If a workload needs high memory, Karpenter will automatically provision an r5.large node instead of a generic instance.
Do You Need to Define Instance Types in Karpenter?

👉 No, if you want full automation.
Karpenter will dynamically choose the best instance type based on pod resource requirements (CPU, memory, GPU, etc.).

📌 Example Provisioner YAML (Fully Dynamic):
```yml
apiVersion: karpenter.k8s.aws/v1alpha5
kind: Provisioner
metadata:
  name: flexible
spec:
  provider:
    subnetSelector:
      karpenter.sh/discovery: "my-cluster"
    securityGroupSelector:
      karpenter.sh/discovery: "my-cluster"
  ttlSecondsAfterEmpty: 30  # Auto-terminates empty nodes
```
✅ Result: this provisioner has no limitations on CPU, memory, instance types, zones, or capacity type.

👉 Yes, if you want control.
You can specify which instance types Karpenter is allowed to use.
📌 Example Provisioner YAML (Restricted to Certain Instances):
```yml
apiVersion: karpenter.k8s.aws/v1beta1
kind: Provisioner
metadata:
  name: controlled
spec:
  requirements:
    - key: "node.kubernetes.io/instance-type"
      operator: In
      values: ["t3.medium", "c5.large", "m5.large"]  # Allowed instance types

    - key: "topology.kubernetes.io/zone"
      operator: In
      values: ["us-east-1a", "us-east-1b", "us-east-1c"]  # Adding more availability zones for better distribution

    - key: "karpenter.k8s.aws/capacity-type"
      operator: In
      values: ["spot", "on-demand"]  # Supports both spot and on-demand

  limits:
    resources:
      cpu: "50"  # Limits total CPU usage to 50 cores
      memory: "250Gi"  # Limits total memory usage to 250Gi

  provider:
    subnetSelector:
      karpenter.sh/discovery: "my-cluster"
    securityGroupSelector:
      karpenter.sh/discovery: "my-cluster"

  ttlSecondsAfterEmpty: 60  # Increase time before termination to 60 seconds
```
# explanation - 
apiVersion: Defines the API version (karpenter.k8s.aws/v1beta1) being used for this resource.
kind: Specifies the resource type as Provisioner, which allows Karpenter to manage node provisioning dynamically.
metadata: Contains metadata like the provisioner name (controlled) for identification.
requirements: Defines constraints on which node types and properties can be used.
node.kubernetes.io/instance-type: Restricts provisioning to t3.medium, c5.large, and m5.large instance types.
topology.kubernetes.io/zone: Ensures nodes are provisioned only in us-east-1a, us-east-1b, and us-east-1c availability zones.
karpenter.k8s.aws/capacity-type: Restricts nodes to either spot or on-demand instances.
limits: Specifies maximum resource limits to prevent over-provisioning.
cpu: Caps the total CPU usage across provisioned nodes at 50 cores.
memory: Limits total memory allocation to 250Gi.
provider: Configures networking-related settings like subnets and security groups.
subnetSelector: Uses the karpenter.sh/discovery: my-cluster tag to automatically select subnets.
securityGroupSelector: Uses the same discovery tag to select security groups.
ttlSecondsAfterEmpty: Terminates an empty node automatically after 60 seconds of inactivity to optimize costs.

How to Set Up Karpenter in AWS? (Step-by-Step)

To prove hands-on experience, explain the real steps to set up Karpenter in AWS:
1. Prerequisites

✅ A running EKS Cluster
✅ AWS CLI and kubectl installed
✅ IAM permissions to create policies, roles, and EC2 instances
2. Install Karpenter Using Helm

helm repo add karpenter https://charts.karpenter.sh/
helm repo update
helm install karpenter karpenter/karpenter \
  --namespace karpenter \
  --create-namespace \
  --set clusterName=<YOUR-EKS-CLUSTER-NAME> \
  --set aws.defaultInstanceProfile=KarpenterNodeInstanceProfile

3. Create IAM Role for Karpenter

Create a role that allows Karpenter to provision EC2 nodes.

aws iam create-role --role-name KarpenterControllerRole \
  --assume-role-policy-document file://karpenter-trust-policy.json

Attach necessary policies:

aws iam attach-role-policy --role-name KarpenterControllerRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy --role-name KarpenterControllerRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

4. Create a Karpenter Provisioner

Define how Karpenter should provision nodes.

apiVersion: karpenter.k8s.aws/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: "node.kubernetes.io/instance-type"
      operator: In
      values: ["t3.medium", "m5.large", "c5.large"]
    - key: "topology.kubernetes.io/zone"
      operator: In
      values: ["us-west-2a", "us-west-2b"]
  provider:
    subnetSelector:
      karpenter.sh/discovery: "my-cluster"
    securityGroupSelector:
      karpenter.sh/discovery: "my-cluster"
  ttlSecondsAfterEmpty: 30

Apply it:

kubectl apply -f karpenter-provisioner.yaml

5. Deploy a Sample Application to Test Scaling

apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox
spec:
  replicas: 5
  selector:
    matchLabels:
      app: busybox
  template:
    metadata:
      labels:
        app: busybox
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["sleep", "3600"]
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"

Apply it:

kubectl apply -f busybox.yaml

6. Watch Karpenter Provision Nodes

kubectl get nodes -w

✅ Result: Karpenter automatically provisions nodes based on actual workload demands! 🚀
Final Takeaway (Why Karpenter Wins)

If your interviewer asks:

    "Why should we use Karpenter instead of Cluster Autoscaler?"

You can confidently say:
✅ Faster Scaling – No ASG dependency, direct EC2 API calls.
✅ Better Cost Optimization – Uses Spot, ARM, and optimized instances dynamically.
✅ More Flexibility – Supports multiple instance types and scales beyond predefined limits.

🔥 If the workload needs 5 nodes today and 50 tomorrow, Karpenter will handle it automatically—no manual limits required!

🚀 This is why Karpenter is the future of Kubernetes node autoscaling in AWS!

# 2. Storage & Persistent Volumes

2.1 Persistent Volumes (PV) & Persistent Volume Claims (PVC)

PV: Cluster-wide storage resource.

PVC: Request for storage by a pod.

Example PVC:

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi

2.2 Container Storage Interface (CSI)

CSI allows Kubernetes to work with external storage providers like AWS EBS, Azure Disk, and GCP Persistent Disk.

Example StorageClass for AWS EBS:

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3

# 4. Advanced Debugging & Troubleshooting

4.1 Debugging Pods

Check logs:

kubectl logs <pod-name>

Check detailed pod status:

kubectl describe pod <pod-name>

Exec into a running pod:

kubectl exec -it <pod-name> -- /bin/sh

4.2 Debugging Nodes & Networking

Check node status:

kubectl get nodes -o wide

Check networking issues:

kubectl get svc -A  (Check Services in All Namespaces)

kubectl get endpoints -A  (Check if Services Have Endpoints)
📌 Why use this?

    Checks if services are correctly routing traffic to pods.
    If a service has no endpoints, it means no pods are backing the service (which is a problem).

🔍 Example Output:

NAMESPACE   NAME            ENDPOINTS           AGE
default     my-service      192.168.1.10:8080   5d
default     my-service2     <none>              2d

📌 Things to check:
✔️ If an endpoint exists (e.g., 192.168.1.10:8080), the service is correctly routing traffic.
❌ If the endpoint is <none>, the service is not forwarding requests to a pod (likely an issue with pod labels or readiness probes).

