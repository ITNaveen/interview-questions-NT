Amazon EKS (Elastic Kubernetes Service) - Complete Guide

This guide covers everything about AWS EKS, from setting up a cluster to deploying applications. By the end, you will have a strong understanding of EKS and be able to answer interview questions confidently.

# 1. Introduction to Amazon EKS

📌 What is Amazon EKS?

Amazon Elastic Kubernetes Service (EKS) is a managed Kubernetes service that makes it easy to run Kubernetes clusters on AWS.

🔹 Why Use EKS?

Fully Managed: AWS handles control plane maintenance.

Highly Available: Multi-AZ deployment support.

Secure: Integrated with AWS IAM and security services.

Scalable: Supports auto-scaling of worker nodes.

Networking Flexibility: Works with AWS VPC, ALB, and Service Mesh.

🔹 EKS vs. Self-Managed Kubernetes

Feature

Amazon EKS

Self-Managed Kubernetes

Control Plane

Managed by AWS

You manage everything

Scalability

AWS Auto Scaling

Manual configuration

Security

IAM integration

Needs manual setup

Maintenance

AWS handles updates

You must update manually

# 2. Setting Up an EKS Cluster

📌 Steps to Set Up an EKS Cluster

Create an IAM Role for EKS.

Create a VPC (or use an existing one).

Deploy an EKS Cluster.

Configure kubectl to connect to the cluster.

Deploy worker nodes (EC2 or Fargate).

🔹 Step 1: Install AWS CLI & kubectl

Before setting up EKS, install the required tools:

# Install AWS CLI
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /

# Install kubectl
curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.21.2/2021-07-05/bin/darwin/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/

🔹 Step 2: Create an IAM Role for EKS

aws iam create-role --role-name eksClusterRole --assume-role-policy-document file://eks-trust-policy.json
aws iam attach-role-policy --role-name eksClusterRole --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

🔹 Step 3: Create a VPC for EKS

Use AWS CloudFormation or eksctl:

eksctl create cluster --name my-eks-cluster --region us-west-2 --nodegroup-name my-nodes --nodes 2 --nodes-min 1 --nodes-max 3 --managed

🔹 Step 4: Update kubectl to Connect to the Cluster

aws eks update-kubeconfig --region us-west-2 --name my-eks-cluster
kubectl get svc

🔹 Step 5: Deploy Worker Nodes (EC2-based or Fargate)

For EC2-based nodes:

eksctl create nodegroup --cluster my-eks-cluster --name my-workers --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 5

For AWS Fargate (Serverless Kubernetes Nodes):

eksctl create fargateprofile --cluster my-eks-cluster --name fargate-profile --namespace default

# 3. Deploying Applications on EKS

🔹 Step 1: Deploy a Sample Application

Create a simple Nginx Deployment.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80

Apply the deployment:

kubectl apply -f nginx-deployment.yaml
kubectl get pods

🔹 Step 2: Expose the Application Using a Load Balancer

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

Apply the service:

kubectl apply -f nginx-service.yaml
kubectl get svc

# 4. Scaling EKS Cluster (Auto Scaling)

🔹 Scale Pods Automatically (HPA)

kubectl autoscale deployment nginx-deployment --cpu-percent=50 --min=1 --max=10

🔹 Scale Worker Nodes Automatically

eksctl scale nodegroup --cluster my-eks-cluster --name my-workers --nodes 5

# 5. Monitoring & Logging (CloudWatch, Prometheus, Grafana)

🔹 Enable CloudWatch Logging for EKS

aws eks update-cluster-config --region us-west-2 --name my-eks-cluster --logging '{"clusterLogging":[{"types":["api","audit","authenticator"],"enabled":true}]}'

🔹 Install Prometheus & Grafana

kubectl apply -f https://github.com/prometheus-operator/kube-prometheus/releases/latest/download/kube-prometheus.yaml

# 6. Security Best Practices

Use IAM Roles for Pods (Least privilege access)

Enable Pod Security Policies (Restrict container privileges)

Encrypt Data at Rest (EBS encryption for worker nodes)

Use AWS Secrets Manager (For sensitive information like database credentials)

# 7. Deleting an EKS Cluster

eksctl delete cluster --name my-eks-cluster --region us-west-2

Final Thoughts: Why Use EKS in DevOps?

Simplifies Kubernetes Management with AWS-managed control plane.

Integrates with AWS Services (IAM, ALB, CloudWatch, Fargate).

Supports Auto-Scaling for cost-effective infrastructure.

This document covers everything about EKS so you can confidently discuss and implement it in your DevOps workflows. 🚀