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

Feature

Cluster Autoscaler

Karpenter

Scaling Speed

Slower (relies on ASG scaling)

Faster (direct EC2 API calls)

Cost Optimization

Basic

Advanced (uses Spot, ARM, etc.)

Node Provisioning

Fixed instance types

Flexible instance types

2. Storage & Persistent Volumes

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

3. Multi-Cluster & Hybrid Cloud

3.1 Kubernetes Federation (KubeFed)

KubeFed enables management of multiple Kubernetes clusters from a single control plane.

Commands to install KubeFed:

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/kubefed/main/charts/kubefed.yaml

3.2 Crossplane for Multi-Cloud

Crossplane extends Kubernetes to manage cloud infrastructure (AWS, Azure, GCP) using native Kubernetes APIs.

kubectl apply -f https://raw.githubusercontent.com/crossplane/crossplane/main/install.yaml

4. Advanced Debugging & Troubleshooting

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

kubectl get svc -A
kubectl get endpoints -A

Conclusion

By mastering autoscaling, storage, multi-cluster setups, and debugging strategies, you can optimize Kubernetes for high availability, cost efficiency, and scalability.