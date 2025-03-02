# ArgoCD: From Basics to Daily Usage - Complete Interview Guide

## Part 1: Understanding ArgoCD Simply

### What is ArgoCD?

Imagine you're building a LEGO castle. You have a blueprint of how it should look. Now, what if someone keeps changing parts of your castle when you're not looking? You'd have to keep checking and fixing it, right?

That's what ArgoCD does for computer programs running in Kubernetes:

**ArgoCD is like a watchful guardian that:**
1. Looks at the blueprint (your code in Git)
2. Checks your actual application (running in Kubernetes)
3. Fixes anything that doesn't match the blueprint

In technical terms: ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes that ensures your deployed applications match what's defined in your Git repository.

### Why is ArgoCD Important?

Before ArgoCD, developers had to manually update their applications, which was:
- Slow (imagine rebuilding everything by hand)
- Error-prone (people make mistakes)
- Hard to track (who changed what and when?)

With ArgoCD, everything is automated and always matches what's in your Git repository.

### How ArgoCD Works

1. **You store your app blueprints in Git**
   - YAML files that define your Kubernetes resources

2. **ArgoCD constantly watches your Git repository**
   - Detects any changes to your configuration

3. **ArgoCD also watches your running application**
   - Monitors the actual state in your Kubernetes cluster

4. **If they don't match, ArgoCD reconciles the differences**
   - Either automatically or after manual approval, depending on your settings

## Part 2: My Daily Experience with ArgoCD

### My Typical Day with ArgoCD

"Every morning, I start by checking our ArgoCD dashboard to get a quick overview of all our applications. We have about 15 microservices deployed across 3 Kubernetes clusters, and ArgoCD gives me instant visibility into their health and sync status."

"Just last week, I noticed one application was marked as 'OutOfSync' in the dashboard. Using the App Diff tool, I could immediately see that a ConfigMap had been manually changed in the cluster instead of updating it in Git. I reverted the manual change and reminded the team about our GitOps workflow - all changes must go through Git."

### Real Troubleshooting Scenarios I've Handled

"We recently had an incident where a deployment failed because a developer accidentally committed an invalid resource specification. ArgoCD immediately showed the application as 'Degraded' with a specific error message pointing to the problem. I used the App Diff tool to identify the exact issue - a typo in the container port definition. After fixing it in Git, ArgoCD automatically reconciled the application within minutes."

"Another time, we had a mysterious issue where an application would successfully sync but then report as unhealthy. By examining the Events tab in ArgoCD, I discovered that the pods were being evicted due to resource constraints. We adjusted the resource requests in our manifests, committed the change to Git, and ArgoCD automatically deployed the fixed version."

## Part 3: Key ArgoCD Features I Use Daily

### 1. Sync Policies: Automated vs. Manual

**In my daily work:**
"We use different sync policies based on environment. For development, we have fully automated sync with selfHeal enabled, so any changes to the Git repo or manual changes to the cluster are automatically reconciled. For production, we use manual sync to ensure changes are reviewed before being applied."

**How it works:**
- **Automated Sync:** Changes in Git are automatically applied to the cluster
- **Manual Sync:** Changes are detected but not applied until manually approved
- **SelfHeal:** Automatically reverts manual changes made directly to the cluster
- **Prune:** Removes resources that were deleted from Git

### 2. Multi-Cluster Management

**In my daily work:**
"We manage applications across three EKS clusters in different AWS regions using ApplicationSets. This allows us to define an application once and have ArgoCD automatically deploy it to all clusters, while still allowing for region-specific configuration when needed."

**How it works:**
- Register multiple clusters with ArgoCD
- Use ApplicationSet controller to template and generate Applications
- Define generators (list, cluster, Git) to create variations for different clusters

### 3. Secret Management

**In my daily work:**
"For secrets management, I've set up External Secrets Operator to work with AWS Secrets Manager. When our application needs access to database credentials, ESO fetches them securely from AWS and creates Kubernetes secrets, which ArgoCD then references in the application deployment."

**How it works:**
1. Store secrets in a secure external vault (AWS Secrets Manager)
2. Install External Secrets Operator (ESO) in your cluster
3. Configure ESO to fetch secrets and create Kubernetes Secrets
4. ArgoCD deploys applications that reference these Kubernetes Secrets

### 4. Progressive Delivery (Canary/Blue-Green)

**In my daily work:**
"We use Argo Rollouts integrated with ArgoCD for our critical services. Last month, we deployed a major update to our payment service using a canary strategy that gradually shifted traffic from 10% to 100% over 2 hours, with automatic analysis of error rates at each step."

**How it works:**
- Install Argo Rollouts controller
- Define Rollout resources instead of Deployments
- Configure the desired strategy (canary or blue/green)
- Let ArgoCD manage the Rollout resource

## Part 4: Practical ArgoCD Customizations

### Custom Health Checks

**In my daily work:**
"We've implemented custom health checks for our MongoDB StatefulSets since the default Kubernetes readiness probes don't fully capture database health. I added a Lua script to our ArgoCD ConfigMap that checks specific MongoDB metrics through our monitoring system."

```yaml
resource.customizations.health.mongodb.com/MongoDB: |
  hs = {}
  if obj.status ~= nil then
    if obj.status.phase == "Running" and obj.status.mongoUri ~= nil then
      hs.status = "Healthy"
    else
      hs.status = "Progressing"
    end
  else
    hs.status = "Progressing"
  end
  return hs
```

### Integration with CI Pipeline

**In my daily work:**
"Our CI pipeline builds container images, runs tests, and then updates the image tag in our Helm values repository. ArgoCD detects this change and updates our applications accordingly. This creates a fully automated pipeline from code commit to deployment."

### Notifications and Alerting

**In my daily work:**
"We've set up ArgoCD notifications to alert our team's Slack channel whenever an application goes out of sync or fails to sync. This has drastically reduced our time to respond to issues from hours to minutes."

## Part 5: Common Interview Questions and My Experience-Based Answers

### "What is ArgoCD and how does it implement GitOps principles?"

"ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes. In our organization, we use it to manage all our Kubernetes deployments across development, staging, and production environments. It implements GitOps by using our Git repositories as the single source of truth. For example, when a developer wants to update a service configuration, they submit a pull request to our config repository. Once approved and merged, ArgoCD automatically detects the change and updates the application in Kubernetes. This gives us full audit trail, versioning, and rollback capabilities for all our infrastructure changes."

### "Explain the difference between ArgoCD's automated and manual sync policies."

Development Environment: Automated Sync with SelfHeal & Prune
Automated Sync: Any change made in the Git repository is automatically applied to the Kubernetes cluster.
SelfHeal Enabled: If someone manually changes something directly in the cluster (instead of updating Git), ArgoCD automatically reverts the change to match Git.
Prune Enabled: If something in the cluster is not present in the Git repo (e.g., a deleted resource), it gets removed automatically.
Example: A developer manually edited a Kubernetes deployment to debug an issue. However, because of SelfHeal, ArgoCD reverted the change to match what was in Git, preventing inconsistencies and confusion.

In production, ArgoCD continuously monitors the Git repository. When a developer pushes new code or changes to the repo, ArgoCD detects that the cluster no longer matches Git and marks the application as "OutOfSync." However, because it's set to manual sync, ArgoCD does not automatically apply the changes.
Instead, someone with the right permissions (like a DevOps engineer or an SRE) has to manually approve and trigger the sync in ArgoCD. This extra step ensures that changes are reviewed before they go live in production.

### "How does ArgoCD handle secrets management?"

"In our AWS-based infrastructure, we've integrated ArgoCD with External Secrets Operator and AWS Secrets Manager. For example, our database credentials are stored in AWS Secrets Manager, and ESO creates Kubernetes secrets that our applications can use. ArgoCD then references these secrets in deployments without ever storing sensitive information in Git. When we need to rotate credentials, we update them in AWS Secrets Manager, and the change propagates through ESO to our applications without any changes to our GitOps workflow."

### "How would you implement a multi-cluster deployment strategy using ArgoCD?"

What is the ApplicationSet Controller?
The ApplicationSet controller is an ArgoCD feature that automates the creation and management of multiple ArgoCD applications.
It allows you to define a single template and generate multiple applications dynamically based on different parameters (like cluster names, environments, or Git branches).
This helps manage multi-cluster deployments efficiently.
How the ApplicationSet Works in This Case
ApplicationSet Uses a List Generator

The List Generator inside the ApplicationSet defines multiple environments (development, staging, production).
It creates a separate ArgoCD application instance for each environment.
Example:
dev-cluster → Deploys app with dev settings.
staging-cluster → Deploys app with staging settings.
prod-cluster → Deploys app with production settings.
Single Git Repository with Kustomize Overlays

The team stores all configurations in a single Git repo.
Uses Kustomize overlays to apply different environment-specific settings.
Example:
base/ contains common configs (same for all environments).
overlays/dev/ has dev-specific configs.
overlays/staging/ has staging-specific configs.
overlays/prod/ has prod-specific configs.
Promotion of Changes

Changes start in development, then are promoted to staging, and finally to production by merging Git branches.
ArgoCD automatically detects changes and deploys them based on the environment.

### "Explain how to implement a progressive delivery deployment with ArgoCD."

"We use ArgoCD with Argo Rollouts for our user-facing services. For example, our authentication service uses a canary deployment strategy. We define a Rollout resource that specifies a series of steps: first deploy to 10% of users, wait for 10 minutes while analyzing error rates and response times, then increase to 30%, and so on. ArgoCD manages this Rollout resource just like any other Kubernetes resource, ensuring it matches what's defined in Git. This has been extremely valuable for detecting issues early before they affect all users."

### "How does ArgoCD's resource health assessment work, and how can you customize it?"

"ArgoCD has built-in health checks for standard Kubernetes resources, but we've extended these for our custom resources. For instance, we have a custom resource for our message queue service that ArgoCD wouldn't know how to check by default. I added a custom health assessment using a Lua script in the ArgoCD ConfigMap that checks if the queue service is properly connected to its broker and processing messages. This gives us more accurate health reporting in the ArgoCD dashboard."

### "How would you troubleshoot synchronization issues in ArgoCD?"

"I do this regularly. Just last week, we had an issue where an application wouldn't sync. My troubleshooting process was:

1. I checked the app status in the ArgoCD UI and saw it was 'Failed'
2. I looked at the sync operation logs to find specific error messages
3. The error showed a permission issue with a new resource type we were trying to deploy
4. I verified our ArgoCD service account permissions and found it was missing RBAC rules for the new resource
5. After updating the ClusterRole with the required permissions, the sync succeeded

This methodical approach helps quickly identify and resolve sync issues, whether they're related to Git access, RBAC, or manifest errors."

## Part 6: ArgoCD Application Example I Use Daily

Here's a typical ArgoCD Application definition I work with:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/our-org/payment-service-config.git
    targetRevision: HEAD
    path: kubernetes/overlays/production
    # Using Kustomize to handle environment differences
    kustomize:
      images:
        - name=payment-service:latest=payment-service:v1.2.3
  destination:
    server: https://kubernetes.default.svc
    namespace: payment-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  # Health checking for our application
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

## Conclusion

ArgoCD has transformed how I manage Kubernetes applications by ensuring they always match what's defined in Git. The key benefits I experience daily include:

1. **Complete visibility** into application state across all clusters
2. **Automated reconciliation** that catches and fixes drift
3. **Consistent deployments** that follow the same pattern every time
4. **Audit trail and versioning** for all infrastructure changes
5. **Simplified rollbacks** when issues are detected

When explaining ArgoCD in your interview, combine technical understanding with practical examples from daily use. This will demonstrate not only that you know how it works, but that you have real experience using it to solve problems.
