# ArgoCD Interview Questions and Answers (Intermediate Level)

## 1. What is ArgoCD and how does it implement GitOps principles?

**Answer:** 
ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes. It implements GitOps principles by:

- Using Git repositories as the single source of truth for defining the desired application state
- Automating the deployment of applications to specified target environments
- Continuously monitoring deployed applications and comparing their live state against the desired state defined in Git
- Automatically identifying and reconciling any deviations between the desired and actual state

**Example:**
```yaml
# Application definition in ArgoCD
# Defines the API version for ArgoCD custom resources
apiVersion: argoproj.io/v1alpha1 
# Declares that this is an ArgoCD Application resource
kind: Application 
metadata:
  # Name of the ArgoCD Application
  name: guestbook  
  # Namespace where the ArgoCD instance is running
  namespace: argocd  
spec:
  # The project under which this application falls in ArgoCD
  project: default  
  source:
    # URL of the Git repository containing the Kubernetes manifests
    repoURL: https://github.com/argoproj/argocd-example-apps.git  
    # The branch, tag, or commit to sync from; "HEAD" means the latest commit on the default branch
    targetRevision: HEAD  
    # The path inside the Git repository where the application’s manifest files are located
    path: guestbook  
  destination:
    # The Kubernetes cluster where the application should be deployed
    server: https://kubernetes.default.svc  
    # The namespace in the target Kubernetes cluster where the application should be deployed
    namespace: guestbook  
  syncPolicy:
    automated:
      # If enabled, ArgoCD will delete any resources that are in the cluster but no longer in the Git repository
      prune: true  
      # Ensures that if any changes are made to the live cluster that do not match the Git repository, they are automatically reverted
      selfHeal: true  
```
In this example, ArgoCD continuously monitors the specified Git repository and automatically applies any changes to the Kubernetes cluster, ensuring the deployed application always matches what's defined in Git.

Defining the ArgoCD Application

This manifest defines an ArgoCD-managed application called guestbook.
It tells ArgoCD where to find the application's Kubernetes manifests and how to deploy them.
ArgoCD pulls the Kubernetes manifests from the GitHub repository:

Repo: https://github.com/argoproj/argocd-example-apps.git

Branch: Latest commit on HEAD

HEAD refers to the latest commit in the default branch (usually main or master).
If you want to find the latest commit for your repository:
Navigate to your GitHub repository.
Check the branch you want to track (e.g., main or develop).

The latest commit SHA (hash) can be found at the top of the repository page or by running:
git rev-parse HEAD
ArgoCD will automatically sync with this latest commit.
Path: guestbook (This is the folder inside the repository where Kubernetes YAMLs for the app are stored).

- ArgoCD deploys the application to a Kubernetes cluster:
Cluster: https://kubernetes.default.svc
This is a Kubernetes API server address, which is the default internal cluster address used within Kubernetes.
Since you are using EKS, your actual cluster API server address can be found by running:

aws eks describe-cluster --name <your-cluster-name> --query "cluster.endpoint" --output text
The output will be a URL similar to:
https://ABC1234567890.gr7.us-west-2.eks.amazonaws.com
In your case, replace https://kubernetes.default.svc with this EKS endpoint if managing multiple clusters.

- Namespace: guestbook
The application will be deployed into this Kubernetes namespace.
Sync Policy Ensures:

- Auto-syncing:
Whenever the Git repository is updated, changes are automatically applied to the Kubernetes cluster.

- Pruning:
If a resource exists in the cluster but is removed from Git, ArgoCD will delete it from Kubernetes to maintain Git as the single source of truth.

- Self-healing:
If someone manually modifies a deployed resource (e.g., updating a deployment using kubectl edit), ArgoCD will detect the drift and revert it to match the Git repository.

- Summary:
This ArgoCD Application manifest ensures that the guestbook app is continuously deployed and stays in sync with the Git repository.

Any changes made in Git automatically reflect in EKS.
If someone modifies the running application manually, ArgoCD restores the Git-defined state.
Deleted resources in Git are also removed from the cluster to maintain consistency.


## 2. Explain the difference between ArgoCD's "automated" and "manual" sync policies.

**Answer:**
ArgoCD offers two primary sync policies:

**Manual Sync:**
- Changes detected in the Git repository are not automatically applied
- Administrators must manually trigger a sync operation through the UI or CLI
- Provides more control over when updates are applied to the cluster
- Useful for environments requiring explicit approval before deployment

**Automated Sync:**
- Changes in the Git repository are automatically detected and applied to the cluster
- Can be configured with options like `prune` (remove resources that no longer exist in Git) and `selfHeal` (revert manual changes made to the cluster)
- Provides a fully automated GitOps workflow
- Better suited for development environments or when continuous deployment is desired

**Example Configuration:**
```yaml
# Manual sync policy (default)
syncPolicy: {}

# Automated sync policy with pruning and self-healing
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

## 3. How does ArgoCD handle secrets management?

ArgoCD does not store or manage secrets directly but integrates with external secret management solutions. In an AWS-based infrastructure, the best practice for handling secrets securely is by using AWS Secrets Manager.

How ArgoCD Uses AWS Secrets Manager?
1️⃣ Install External Secrets Operator (ESO)
ArgoCD cannot fetch secrets from AWS Secrets Manager directly. Instead, we need External Secrets Operator (ESO) to act as a bridge between AWS Secrets Manager and Kubernetes Secrets.

Install ESO using Helm:
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets --namespace external-secrets --create-namespace
👉 This allows Kubernetes to fetch secrets from AWS Secrets Manager.

2️⃣ Enable Kubernetes to Authenticate with AWS Secrets Manager
🔹 Step 1: Create an IAM Role for ESO to Access AWS Secrets Manager
First, we create an IAM Role that will be assumed by ESO when it needs to fetch secrets from AWS Secrets Manager.

🔹 Step 2: Create an IAM Policy (Attach to Role)
Define an IAM Policy that grants read access to AWS Secrets Manager.
Attach this policy to the IAM Role from Step 1.
This ensures that any service assuming the role has permission to fetch secrets.

--- The IAM Role itself doesn't define permissions—it must be attached to a policy that grants the required permissions.

🔹 Step 3: Create a Kubernetes Service Account
Since Kubernetes pods can’t directly assume AWS IAM Roles, we need to create a Kubernetes Service Account.
The Service Account is linked to the IAM Role via an annotation (eks.amazonaws.com/role-arn).

- AWS IAM policies and roles cannot be attached directly to Kubernetes pods because pods are not AWS resources.

🔹 Step 4: Associate the Service Account with ESO
Update ESO’s ClusterSecretStore configuration to reference the Kubernetes Service Account created in Step 3.
This tells ESO to use the IAM Role for authentication when fetching secrets.

🔹 Now ESO Can Fetch Secrets from AWS Secrets Manager!
✅ ESO authenticates using the IAM Role.
✅ It fetches secrets from AWS Secrets Manager.
✅ Secrets are stored inside Kubernetes as native Kubernetes Secrets.
✅ ArgoCD can now use these secrets during deployments.

3️⃣ ESO Fetches Secrets from AWS Secrets Manager
Now that Kubernetes can authenticate with AWS, we create an ExternalSecret to define which secrets should be pulled from AWS Secrets Manager and stored as Kubernetes Secrets.

yaml
Copy
Edit
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-secret
  namespace: my-app
spec:
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: my-secret-k8s  # The name of the Kubernetes Secret to create
  data:
    - secretKey: db-password  # Key in Kubernetes Secret
      remoteRef:
        key: my-database-secret  # AWS Secrets Manager secret name
        property: password  # Field to fetch
👉 This means the password field from the my-database-secret secret in AWS will be copied into a Kubernetes Secret called my-secret-k8s.

4️⃣ ArgoCD Fetches Secrets from Kubernetes Secrets
Once the secret is available inside Kubernetes, ArgoCD can use it inside applications. ArgoCD does not fetch from AWS Secrets Manager directly—it only interacts with Kubernetes Secrets.

Modify your ArgoCD Application manifest:

yaml
Copy
Edit
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/my-app.git
    path: my-app
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
  kustomize:
    patches:
      - target:
          kind: Deployment
        patch: |-
          - op: add
            path: /spec/template/spec/containers/0/env
            value:
              - name: DB_PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: my-secret-k8s
                    key: db-password
👉 This ensures that the DB_PASSWORD environment variable inside the container is populated with the secret value from Kubernetes Secret my-secret-k8s, which ESO fetched from AWS Secrets Manager.

🔄 What’s Happening Here?
1️⃣ ArgoCD deploys an application in Kubernetes, which needs a secret.
2️⃣ ESO fetches secrets from AWS Secrets Manager and creates a Kubernetes Secret.
3️⃣ ArgoCD references the Kubernetes Secret inside application manifests.
4️⃣ If the AWS Secret is updated, ESO automatically syncs the Kubernetes Secret.
5️⃣ If someone manually changes the secret in Kubernetes, ArgoCD self-heals and restores it from Git.

❓ Frequently Asked Questions
Where can I find the latest commit on HEAD?
In an AWS-based deployment (EKS), the latest commit on HEAD refers to the most recent commit in your Git repository. You can find it using:

🔹 Git CLI:
git log -1 --format=%H
🔹 GitHub UI: Navigate to your repository, and the latest commit hash is shown at the top.

ArgoCD automatically pulls the latest commit from the specified branch and syncs changes accordingly.

# What is "https://kubernetes.default.svc" in the manifest?
This is the Kubernetes API server's internal cluster address.
ArgoCD connects to this endpoint to deploy applications within the EKS cluster.
In AWS EKS, this resolves automatically inside the cluster without requiring external access.

If you need to check the actual EKS API endpoint, run:
aws eks describe-cluster --name my-cluster --query "cluster.endpoint" --output text
✨ Why This Approach is Secure and Scalable?
✅ Secrets Never Stored in Git – Avoids security risks of storing plaintext secrets in a Git repository.
✅ IAM-Based Access Control – Only specific Kubernetes workloads can access AWS Secrets Manager.
✅ Automatic Rotation – AWS can automatically rotate secrets (e.g., DB passwords) without breaking applications.
✅ Self-Healing & Syncing – If a secret is changed manually in Kubernetes, ESO ensures it always matches AWS Secrets Manager.
✅ Scalability – Works across multiple clusters in AWS without storing sensitive information inside Kubernetes manifests.

Conclusion
Yes, the correct flow is:
🔹 Step 1: Install External Secrets Operator (ESO) to enable Kubernetes to fetch secrets.
🔹 Step 2: Configure Kubernetes Authentication with AWS (IAM role via IRSA).
🔹 Step 3: ESO pulls secrets from AWS Secrets Manager and creates Kubernetes Secrets.
🔹 Step 4: ArgoCD reads Kubernetes Secrets and injects them into application workloads.

## 5. How would you implement a multi-cluster deployment strategy using ArgoCD?

**Answer:**
To implement a multi-cluster deployment strategy with ArgoCD:

1. **Register Multiple Clusters:**
   - Add target clusters to ArgoCD using `argocd cluster add` command or through the UI
   - Ensure ArgoCD has the necessary credentials and permissions for each cluster

2. **Application of Applications Pattern:**
   - Use the "App of Apps" pattern to manage applications across clusters
   - Create a parent Application that points to a Git repository containing multiple child Application definitions

3. **ApplicationSet Controller:**
   - Use the ApplicationSet controller to template and generate multiple Application resources
   - Create templates that can target different clusters based on labels or other criteria

4. **Environment-Specific Configurations:**
   - Use Kustomize overlays or Helm values for environment-specific configurations
   - Maintain a single source code base with environment variations

**Example using ApplicationSet:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: production
        url: https://kubernetes.example.com
        values:
          replicas: 5
      - cluster: staging
        url: https://staging.example.com
        values:
          replicas: 2
  template:
    metadata:
      name: '{{cluster}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps.git
        targetRevision: HEAD
        path: guestbook
        helm:
          parameters:
          - name: replicaCount
            value: '{{values.replicas}}'
      destination:
        server: '{{url}}'
        namespace: guestbook
```

## 6. Explain how to implement a progressive delivery (canary or blue/green) deployment with ArgoCD.

**Answer:**
ArgoCD itself doesn't provide built-in progressive delivery capabilities, but it can be integrated with tools like Argo Rollouts to implement canary or blue/green deployments:

**Blue/Green with Argo Rollouts:**
1. Install Argo Rollouts controller in your cluster
2. Define a Rollout resource instead of a Deployment
3. Configure the Rollout to use the blue/green strategy
4. Let ArgoCD manage the Rollout resource

**Canary with Argo Rollouts:**
1. Define a Rollout resource with a canary strategy
2. Configure analysis templates for validation
3. Specify the desired traffic weight steps
4. Use ArgoCD to sync and manage the Rollout

**Example Rollout Resource for Canary:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: example-rollout
spec:
  replicas: 5
  selector:
    matchLabels:
      app: example
  template:
    metadata:
      labels:
        app: example
    spec:
      containers:
      - name: example
        image: nginx:1.19
        ports:
        - containerPort: 8080
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {duration: 10m}
      - setWeight: 40
      - pause: {duration: 10m}
      - setWeight: 60
      - pause: {duration: 10m}
      - setWeight: 80
      - pause: {duration: 10m}
```

**Integration with ArgoCD:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: rollout-example
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/repo.git
    targetRevision: HEAD
    path: rollouts
  destination:
    server: https://kubernetes.default.svc
    namespace: rollouts
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 7. How does ArgoCD's resource health assessment work, and how can you customize it?

**Answer:**
ArgoCD performs health checks on Kubernetes resources to determine the overall health of an application:

**Default Health Assessment:**
- ArgoCD has built-in health check criteria for common Kubernetes resources (Deployments, StatefulSets, Services, etc.)
- For Deployments, it checks if desired replicas match available replicas
- For StatefulSets, it verifies the readiness of pods
- For Services, it confirms the service has endpoints

**Custom Health Checks:**
1. **Lua Scripts:**
   - Define custom health assessment logic using Lua scripts
   - Configure these in the ArgoCD ConfigMap

2. **Resource Hooks:**
   - Use annotations to define custom health check behavior
   - Specify custom health check commands or URLs

3. **Health Check CRD:**
   - For completely custom resources, define a health.argocd.io/hook annotation
   - Implement a separate service that ArgoCD can call to check health

**Example Custom Health Check:**
```yaml
# ConfigMap with custom health check
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  resource.customizations.health.my-group/my-kind: |
    hs = {}
    hs.status = "Progressing"
    hs.message = ""
    if obj.status ~= nil then
      if obj.status.phase == "Running" then
        hs.status = "Healthy"
      end
      if obj.status.phase == "Failed" then
        hs.status = "Degraded"
        hs.message = obj.status.reason
      end
    end
    return hs
```

## 8. How would you troubleshoot synchronization issues in ArgoCD?
Since you don’t have CLI access and ArgoCD is exposed via a LoadBalancer, all troubleshooting will be done through the ArgoCD Web UI. Below is a structured approach explaining where to find each troubleshooting option in the UI.

1️⃣ Check Application Sync Status
📍 Where to find it?

ArgoCD UI → Applications → Select your application
The Sync Status appears as a colored label next to the application name.
✅ What it means?

🟢 Synced – Everything matches the Git repository.
🟡 OutOfSync – Changes in Git are not yet applied to the cluster.
🔴 Failed – An error occurred during synchronization.
🔍 Next steps:

If OutOfSync, check what changes exist using the App Diff tool (see Step 6).
If Failed, go to the Sync Operation Logs (see Step 3).
2️⃣ Inspect Individual Resources for Errors
📍 Where to find it?

ArgoCD UI → Applications → Select application → Click the "Resources" tab
This shows all Kubernetes resources (Deployments, Services, ConfigMaps, etc.).
Click on a resource to see its current state and any errors.
✅ What to check?

If a pod is failing, check the error message.
If a resource is missing, it might be a misconfiguration in Git.
3️⃣ View Sync Operation Logs
📍 Where to find it?

ArgoCD UI → Applications → Select application → Click the "App Details" tab
Scroll down to the "History and Rollback" section.
Click on the latest sync event to view logs.
✅ What to check?

Look for error messages indicating authentication, RBAC, or Git issues.
4️⃣ Verify Git Repository Configuration
📍 Where to find it?

ArgoCD UI → Settings → Repositories
Check the repository URL, branch, and path settings.
✅ What to check?

Ensure the Git URL is correct.
Confirm the branch (e.g., main or develop) exists.
Verify repository credentials are valid.
5️⃣ Check Kubernetes Permissions (RBAC Issues)
📍 Where to find it?

ArgoCD UI → Applications → Select application → Click the "Events" tab
This will show any permission errors.
✅ Common errors:

"forbidden: User cannot create resource" → ArgoCD doesn’t have RBAC permissions.
"failed to update resource" → The RoleBinding or ClusterRole might be missing.
🔍 Next steps:

If you don’t have Kubernetes access, ask the admin to check ArgoCD’s ServiceAccount permissions.
6️⃣ Compare Live vs. Desired State (Diff Tool in UI)
📍 Where to find it?

ArgoCD UI → Applications → Select application → Click "App Diff" button
✅ What it does?

Shows the differences between the Git repo (desired state) and the live Kubernetes state.
Helps detect manual changes in Kubernetes that ArgoCD didn’t apply.
🔍 Next steps:

If changes are unintentional, manually sync the application (see Step 7).
7️⃣ Manually Resync the Application
📍 Where to find it?

ArgoCD UI → Applications → Select application → Click "SYNC" → Click "Synchronize"
✅ What it does?

If auto-sync is disabled, this forces ArgoCD to apply changes from Git.
If sync fails, check Sync Operation Logs (Step 3).
8️⃣ Check ArgoCD Controller Logs (Requires Kubernetes Access)
📍 Where to find it?

If deeper debugging is needed, someone with Kubernetes access can check logs:
bash
Copy
Edit
kubectl logs -n argocd deploy/argocd-application-controller
✅ Why check this?

Useful if there are no errors in the UI but sync still fails.
Why Expose ArgoCD via LoadBalancer?
✅ External Access – No need for VPN or SSH.
✅ Better Authentication – Can integrate with AWS ALB + Cognito/IAM for login.
✅ Recommended for Production – Easier for teams to access securely.

Final Answer (Interview-Ready)
"To troubleshoot synchronization issues in ArgoCD, I would use the ArgoCD Web UI. First, I’d check the sync status in the Applications tab. If OutOfSync or Failed, I’d inspect the Resources tab for errors. Next, I’d check sync operation logs in the History and Rollback tab. If needed, I’d verify the Git repository settings under Repositories and check RBAC permissions in the Events tab. I’d also compare the live vs. desired state using the App Diff tool. If required, I’d trigger a manual sync using the Sync button. Since ArgoCD is exposed via a LoadBalancer, this setup allows easy and secure access without CLI dependency."